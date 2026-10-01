# 06 · Browser Stack

**Зона:** Chrome / X11 / CDP
**Компоненты:** Xvfb `:0` · Chromium · CDP `:9222` · chromiumoxide · redirect.html · Chrome Policies
**Статус:** ⭕ OFF по умолчанию (включается `CUA_DD_ENABLE_CHROME=true`)

## Содержание

1. Что это такое
2. Полная цепочка
3. Компоненты
4. Xvfb — виртуальный X-сервер
5. Chromium и его параметры запуска
6. Chrome DevTools Protocol — `:9222`
7. chromiumoxide — мост внутри daemon
8. `/browser/*` endpoints
9. GUI endpoints
10. `/screenshot`
11. redirect.html — sentinel-механизм
12. Chrome Enterprise Policy — первый слой блокировки
13. URLBlocklist в действии
14. Chrome profile "ChatGPT Agent"
15. Chrome proxy configuration
16. Env-флаги
17. Как включить вручную
18. Что не работает
19. Health check
20. Связи
21. Открытые вопросы

## 1. Что это такое

Chrome Browser Stack — это полный графический стек внутри VM, позволяющий модели управлять браузером: открывать страницы, кликать, вводить текст, делать скриншоты. Он включает несколько компонентов:

- **Xvfb** — виртуальный X-сервер (графический дисплей без физического монитора)
- **Chromium** — сам браузер
- **CDP** — Chrome DevTools Protocol для управления браузером
- **chromiumoxide** — Rust-библиотека, через которую container_daemon общается с CDP
- **container_daemon** — HTTP-обёртка над всем этим

Стек полностью отключён по умолчанию. Включается флагом `CUA_DD_ENABLE_CHROME=true` и требует запуска Xvfb + Chromium + container_daemon одновременно.

Даже во включённом состоянии Chrome не может выйти в интернет: политики Managed Policy блокируют любую навигацию наружу. Это независимый первый слой защиты, поверх сетевой блокировки на уровне Azure.

## 2. Полная цепочка

```
Пользователь (модель)
        │
        │ HTTP
        ▼
container_daemon :8085
        │
        │ chromiumoxide
        ▼
Chromium CDP ws://127.0.0.1:9222
        │
        │ DevTools Protocol
        ▼
Chromium process
        │
        │ X11
        ▼
Xvfb :0
        │
        │ virtual framebuffer
        ▼
PNG screenshots
```

То есть модель обращается к container_daemon, тот через библиотеку chromiumoxide открывает WebSocket-соединение к CDP-порту Chromium, Chromium рисует в виртуальный X-дисплей Xvfb, а результат снимается как PNG или как HTML.

Каждое звено зависит от предыдущего:

- Нет Xvfb → Chromium не может стартовать
- Нет Chromium → CDP недоступен
- Нет CDP → container_daemon не может управлять браузером
- Нет container_daemon → HTTP-API недоступен

## 3. Компоненты

| Компонент | Роль | Порт / путь |
|---|---|---|
| Xvfb | виртуальный X-сервер | `:0`, socket `/tmp/.X11-unix/X0` |
| Chromium | браузер | — |
| CDP | Chrome DevTools Protocol | `127.0.0.1:9222` |
| chromiumoxide | Rust CDP-клиент | внутри container_daemon |
| container_daemon | HTTP-API управления | `:8085` |
| `/browser/*` | endpoints управления браузером | — |
| redirect.html | sentinel-редирект | `/home/oai/redirect.html` |
| `chrome-error://chromewebdata/` | страница ошибки Chrome | внутренний ресурс |
| Chrome Policies | managed policy | `/etc/chromium/policies/managed/` |

## 4. Xvfb — виртуальный X-сервер

### 4.1 Что это

Xvfb (X virtual framebuffer) — это X-сервер, который вместо физического монитора рисует в память. Программы «думают», что работают с реальным дисплеем, но на деле результат идёт в буфер, откуда его можно снять как PNG.

Это стандартный способ запускать графические приложения в headless-окружении.

### 4.2 Параметры запуска

В наших наблюдениях Xvfb стартовал с:

```
DISPLAY=:0
Screen: 1920x1080x24
Socket: /tmp/.X11-unix/X0
```

`1920x1080x24` — размер экрана и глубина цвета (24 бита = true color).

### 4.3 Команда запуска

```bash
mkdir -p /tmp/.X11-unix
chmod 1777 /tmp/.X11-unix

DISPLAY=:0 XAUTHORITY=/home/oai/.Xauthority \
    nohup Xvfb :0 -screen 0 1920x1080x24 -ac &
```

Флаг `-ac` отключает access control (не требует авторизации по `XAUTHORITY`). Это упрощает взаимодействие между процессами внутри VM.

### 4.4 Проверка

```bash
# Процесс жив
ps aux | grep '[X]vfb'

# Socket существует
ls -la /tmp/.X11-unix/X0

# Переменная окружения
echo $DISPLAY   # должно быть :0
```

### 4.5 Что без него не работает

Если Xvfb не запущен:

- Chromium не может стартовать
- `/screenshot` в container_daemon возвращает ошибку
- GUI-команды (`/click`, `/type`) работать не будут

## 5. Chromium и его параметры запуска

### 5.1 Команда

Chromium запускается с примерно такими параметрами:

```
chromium \
    --remote-debugging-port=9222 \
    --user-data-dir=/home/oai/.chromium \
    --display=:0 \
    ... (прочие флаги из CUA_DD_CHROME_CAAS_ARGS)
```

Ключевые флаги:

| Флаг | Назначение |
|---|---|
| `--remote-debugging-port=9222` | CDP-порт |
| `--user-data-dir=/home/oai/.chromium` | директория профиля |
| `--display=:0` | использовать Xvfb `:0` |

### 5.2 Профиль

Директория профиля: `/home/oai/.chromium/`

Содержимое:

```
/home/oai/.chromium/
├── Local State          общее состояние Chrome
└── Default/
    └── Preferences      настройки профиля
```

Что не хранится:

- History (истории навигации нет)
- Cookies
- Login Data (сохранённые пароли)

Это минимальный профиль, нужный только для запуска Chrome.

### 5.3 Процесс

После старта в процессах видно:

```
PID ~32371  chromium --remote-debugging-port=9222 ...
```

PID'ы меняются. В наблюдениях встречалось:

- container_daemon → PID ~32364
- Chromium → PID ~32371

Оба процесса связаны: container_daemon запускает Chromium.

### 5.4 X11-соединение

Chromium подключается к Xvfb через socket `/tmp/.X11-unix/X0`. Если Xvfb не запущен, Chromium завершится с ошибкой вида `cannot open display :0`.

## 6. Chrome DevTools Protocol — `:9222`

### 6.1 Что это

CDP — это стандартный протокол Chrome, использующийся для отладки и управления браузером. Работает по WebSocket + HTTP.

Через CDP можно:

- открывать вкладки
- навигировать по URL
- выполнять JavaScript
- снимать скриншоты
- получать DOM
- работать с сетью
- перехватывать события

### 6.2 Endpoint

```
http://127.0.0.1:9222
ws://127.0.0.1:9222
```

Основные HTTP-эндпоинты:

| Endpoint | Назначение |
|---|---|
| `/json/version` | версия Chrome, версия CDP |
| `/json/list` | список доступных target'ов (вкладок) |
| `/json/new` | создать новую вкладку |
| `/json/close/{id}` | закрыть вкладку |

### 6.3 Проверка

```bash
curl http://127.0.0.1:9222/json/version
```

Ожидаемый ответ (пример):

```json
{
  "Browser": "Chrome/1xx.0.0.0",
  "Protocol-Version": "1.3",
  "User-Agent": "...",
  "V8-Version": "...",
  "WebKit-Version": "...",
  "webSocketDebuggerUrl": "ws://127.0.0.1:9222/devtools/browser/..."
}
```

Если Chromium не запущен — `connection refused`.

### 6.4 Важно: CDP ≠ container_daemon

`:9222` и `:8085` — это два разных API:

- `:9222` — низкоуровневый протокол Chrome (общается с самим браузером)
- `:8085` — высокоуровневый API управления (общается с Chromium через chromiumoxide)

Можно работать напрямую с CDP, но обычно используется `:8085`.

## 7. chromiumoxide — мост внутри daemon

### 7.1 Что это

`chromiumoxide` — публичный Rust-крейт для работы с CDP. Внутри container_daemon он используется как низкоуровневый клиент.

container_daemon не открывает соединение с Chromium через какой-то свой протокол. Он использует библиотеку, которая говорит на CDP.

### 7.2 Что это даёт

- Асинхронное управление через tokio
- Типизированные вызовы CDP-методов
- Обработка событий браузера
- Интеграция с X11/Xvfb

### 7.3 Пример потока

Когда приходит `POST /browser/new_page`:

1. Axum принимает HTTP-запрос
2. Парсится JSON, извлекается `url`
3. Вызывается метод chromiumoxide
4. chromiumoxide отправляет CDP-команду через WebSocket `:9222`
5. Chromium выполняет навигацию
6. Результат возвращается обратно через тот же путь
7. Axum формирует HTTP-ответ

Это всё асинхронно, на tokio runtime.

## 8. `/browser/*` endpoints

### 8.1 Список

| Endpoint | Method | Request | Статус |
|---|---|---|---|
| `/browser/new_page` | POST | `{"url": "..."}` | ✅ runtime |
| `/browser/state` | POST | `{}` | ✅ runtime |
| `/browser/page_list` | POST | `{}` | ✅ runtime |
| `/browser/get_state` | POST | — | ❌ 404 |
| `/browser/js` | POST | — | ❌ 404 |

### 8.2 `/browser/new_page`

Создаёт новую вкладку и навигирует на URL.

Запрос:

```bash
curl -X POST http://127.0.0.1:8085/browser/new_page \
    -H 'Content-Type: application/json' \
    -d '{"url":"https://example.com"}'
```

Проверенные статусы:

| Запрос | Ответ |
|---|---|
| POST + корректный JSON | 200 |
| POST `{}` | 422 |
| GET | 405 |
| Chromium недоступен | 500 `Received no response from the chromium instance` |

Статус 500 здесь означает не проблему HTTP, а недоступность browser backend. HTTP-сервер при этом жив.

### 8.3 `/browser/state`

Текущее состояние browser subsystem. Поддерживает GET и POST. Точный формат ответа не восстановлен.

Проверка:

```bash
curl -i http://127.0.0.1:8085/browser/state
curl -i -X POST http://127.0.0.1:8085/browser/state \
    -H 'Content-Type: application/json' -d '{}'
```

### 8.4 `/browser/page_list`

Список открытых вкладок. Поддерживает GET и POST.

```bash
curl -i http://127.0.0.1:8085/browser/page_list
```

### 8.5 `/browser/get_state` и `/browser/js`

Возвращают 404. В реализации есть функции с такими именами (в файлах `get_state.rs`, `js.rs`), но HTTP-route не зарегистрирован. Имена файлов не совпадают с URL.

Правильный путь для state — `/browser/state`, не `/browser/get_state`.

### 8.6 Имена файлов vs HTTP-пути

Это важный паттерн reverse-инжиниринга. В Rust-проектах часто имена файлов означают модуль, а route регистрируется отдельно, и они могут не совпадать. Например:

- файл `get_state.rs` → route `/browser/state`
- файл `js.rs` → route может отсутствовать

Нельзя предполагать, что `/browser/<filename>` — рабочий endpoint.

## 9. GUI endpoints

Это endpoints для управления мышью и клавиатурой внутри браузера.

| Endpoint | Method | Назначение |
|---|---|---|
| `/click` | POST | один клик |
| `/double_click` | POST | двойной клик |
| `/scroll` | POST | прокрутка |
| `/move_mouse` | POST | перемещение курсора |
| `/type` | POST | ввод текста |
| `/paste` | POST | вставка из буфера |
| `/drag` | POST | drag-операция |
| `/compound` | POST | составное действие |
| `/multi_key_press_and_hold` | POST | нажатие нескольких клавиш |
| `/clipboard` | GET | работа с буфером обмена |
| `/screenshot` | GET | скриншот |
| `/user_context` | GET | контекст пользователя |
| `/resume` | POST | возобновить |

### 9.1 Схемы (где известны)

`/click` и `/double_click`:

```json
{
  "x": 100,
  "y": 200,
  "button": 0
}
```

`button` — `u8`: 0 = left, 1 = middle, 2 = right.

`/move_mouse`:

```json
{
  "x": 100,
  "y": 200
}
```

`/scroll`:

```json
{
  "x": 100,
  "y": 200,
  "delta_x": 0,
  "delta_y": -100
}
```

`/type`:

```json
{
  "text": "hello"
}
```

### 9.2 `/clipboard` GET → 500

В наших наблюдениях `/clipboard` возвращал 500, даже когда daemon был жив. Вероятно, требует активного X-селектора буфера обмена.

### 9.3 Полные схемы не зафиксированы

Для `/drag`, `/compound`, `/multi_key_press_and_hold`, `/user_context`, `/resume` известны только имена. Полные JSON-схемы не восстановлены.

Использовать их без проверки схемы не стоит — это может дать 422.

## 10. `/screenshot`

### 10.1 Назначение

Снять текущий экран Xvfb или активной вкладки Chromium.

### 10.2 Формат

```
GET /screenshot
Response: image/png
```

Пример:

```bash
curl http://127.0.0.1:8085/screenshot -o /tmp/screenshot.png
file /tmp/screenshot.png
```

Ожидаемо: `PNG image data`.

### 10.3 Размер

В одном из экспериментов размер PNG был ~68 383 байта. Это соответствует скриншоту 1920×1080 с относительно простым содержимым (браузер без открытой страницы).

### 10.4 Зависимости

`/screenshot` требует:

- работающий container_daemon
- работающий Chromium
- работающий Xvfb

Без Xvfb endpoint падает с ошибкой. Это подтверждено runtime.

### 10.5 Что на скриншоте

Скриншот берётся с виртуального дисплея Xvfb `:0`, на который рисует Chromium. То есть это визуальное состояние браузера целиком, включая:

- панель вкладок (если она есть)
- содержимое активной вкладки
- ошибки Chrome (если навигация заблокирована)

При попытке навигации на внешний URL на скриншоте будет видна страница ошибки Chrome.

## 11. redirect.html — sentinel-механизм

### 11.1 Что это

`/home/oai/redirect.html` — специальный локальный файл, который используется как промежуточная страница при навигации.

Дубликат лежит в `/etc/skel/redirect.html` — это значит, что файл создаётся автоматически для новых пользователей.

### 11.2 Содержимое

```javascript
const t = new URLSearchParams(location.search).get("target");
if (t) setTimeout(() => location.replace(t), 0);
```

Sentinel-комментарий:

```
This is a sentinel value detected in code, and should not be changed
```

### 11.3 Как работает

Двухступенчатая навигация:

1. Chromium открывает `file:///home/oai/redirect.html?target=<URL>`
2. Локальная страница читает query-параметр `target`
3. Через `location.replace()` переходит на указанный URL

### 11.4 Зачем это нужно

Несколько причин:

- **Обход CSP и sandbox для `file://`** — прямая навигация может быть заблокирована, а через локальный файл — нет
- **Логирование** — платформа видит целевой URL до того, как Chrome попробует пойти наружу
- **Точка контроля** — платформа может перехватить и проверить URL до фактической навигации
- **Sentinel** — если файл переписан, платформа может это заметить

### 11.5 Что это не даёт

- Не обходит сетевую блокировку
- Не даёт доступ к внешним ресурсам
- Не является «лазейкой»

Это просто промежуточная страница.

## 12. Chrome Enterprise Policy — первый слой блокировки

### 12.1 Расположение

```
/etc/chromium/policies/managed/
```

Содержит набор JSON-файлов, которые Chrome читает при запуске.

### 12.2 Список файлов

| Файл | Назначение |
|---|---|
| `000_policy_merge.json` | основной файл с ограничениями |
| `001_base_extensions.json` | blocklist расширений |
| `001_base_miscellaneous.json` | общие ограничения |
| `001_base_url_blocklist.json` | `URLBlocklist: ["*"]` |
| `010_dev_*.json` | dev-настройки (в dev-режиме) |
| `020_operator_*.json` | настройки Operator |
| `020_research_*.json` | настройки Research |
| `040_caas_*.json` | настройки CaaS |
| `100_cua_dd_chrome_devtools.json` | блокировка `devtools://` |

### 12.3 `000_policy_merge.json`

Ключевые ограничения:

```json
{
  "DefaultDirectSocketsSetting": 2,
  "DeveloperToolsAvailability": 0,
  "DefaultFileSystemReadGuardSetting": 2,
  "DefaultFileSystemWriteGuardSetting": 2,
  "URLBlocklist": ["*"],
  "ExtensionInstallBlocklist": ["*"],
  "DefaultGeolocationSetting": 2,
  "DefaultNotificationsSetting": 2,
  "DownloadDirectory": "/home/oai/share/",
  "DownloadRestrictions": 1
}
```

### 12.4 Расшифровка кодов

Chrome использует числовые коды для политик:

| Код | Значение |
|---|---|
| 0 | разрешено (allowed) |
| 1 | заблокировано (blocked) |
| 2 | запрещено (disallowed), жёстче чем blocked |

Плюс отдельные политики:

| Политика | Значение | Что значит |
|---|---|---|
| `DefaultDirectSocketsSetting` | 2 | Direct Sockets запрещены |
| `DeveloperToolsAvailability` | 0 | DevTools недоступны |
| `DefaultFileSystemReadGuardSetting` | 2 | File System API read запрещён |
| `DefaultFileSystemWriteGuardSetting` | 2 | File System API write запрещён |
| `URLBlocklist` | `["*"]` | все URL заблокированы |
| `ExtensionInstallBlocklist` | `["*"]` | все расширения запрещены |
| `DefaultGeolocationSetting` | 2 | геолокация запрещена |
| `DefaultNotificationsSetting` | 2 | уведомления запрещены |
| `DownloadRestrictions` | 1 | опасные загрузки блокируются |

### 12.5 Итог

Chrome запускается в максимально ограниченном режиме:

- никаких URL по умолчанию
- никаких расширений
- никаких DevTools
- никакого файлового доступа
- никакой геолокации и уведомлений

Единственное, что разрешено — работа с локальным `file://` (для redirect.html) и отрисовка в Xvfb.

## 13. URLBlocklist в действии

### 13.1 Что происходит при навигации

Попытка открыть любой URL проходит через NetworkService Chrome:

```
GET https://example.com
    │
    ▼
Chrome NetworkService
    │
    ▼
проверка Managed Policy
    │
    ▼
URLBlocklist: ["*"] — совпадение
    │
    ▼
ERR_BLOCKED_BY_ADMINISTRATOR
    │
    ▼
chrome-error://chromewebdata/
```

Важное: TCP-соединение не открывается. Chrome блокирует навигацию до сетевого слоя. Это значит, что даже если бы egress был открыт на уровне Azure, Chrome всё равно бы не смог выйти в интернет.

### 13.2 Что видно через API

При вызове:

```bash
curl -X POST http://127.0.0.1:8085/browser/new_page \
    -H 'Content-Type: application/json' \
    -d '{"url":"https://example.com"}'
```

Возвращается:

```json
{
  "page_html": "<html>...chrome-error://chromewebdata/...</html>",
  ...
}
```

HTTP-статус — 200 OK, но в HTML содержится страница ошибки Chrome, не содержимое example.com.

### 13.3 Лог daemon

```
navigate_and_wait.url = https://example.com
navigate_and_wait.page_html length = 249800
navigate_and_wait error
error = net::ERR_BLOCKED_BY_ADMINISTRATOR
```

Размер 249800 — это размер HTML error-page Chrome. Для примера: реальная страница example.com занимает ~1200 байт.

### 13.4 Что это доказывает

Это первый слой блокировки — уровень Chrome Managed Policy. Он работает независимо от сетевых ограничений Azure NSG.

Даже если бы URLBlocklist был пуст, Chrome упал бы с `Connection refused` на уровне TCP, потому что Azure NSG отрезает egress.

## 14. Chrome profile "ChatGPT Agent"

### 14.1 Preferences

Профиль в `/home/oai/.chromium/Default/Preferences` содержит:

```json
{
  "profile": {
    "name": "ChatGPT Agent"
  },
  "extensions": {
    "commands": {
      "linux:Ctrl+Shift+W": {
        "command_name": "disable0",
        "extension": "kcdongibgcplmaagnmgpjhpjgmmaaaaa"
      },
      "linux:Ctrl+S": {
        "command_name": "disable1",
        "extension": "kcdongibgcplmaagnmgpjhpjgmmaaaaa"
      },
      "linux:Ctrl+N": {
        "command_name": "disable2",
        "extension": "kcdongibgcplmaagnmgpjhpjgmmaaaaa"
      }
    },
    "settings": {
      "kcdongibgcplmaagnmgpjhpjgmmaaaaa": {
        "newAllowFileAccess": true
      }
    }
  }
}
```

### 14.2 Что это значит

- Имя профиля — "ChatGPT Agent"
- Расширение `kcdongibgcplmaagnmgpjhpjgmmaaaaa` — внутреннее расширение ChatGPT
- Отключены горячие клавиши:
    - Ctrl+Shift+W — закрыть все вкладки
    - Ctrl+S — сохранить
    - Ctrl+N — новое окно
- Расширению разрешён доступ к файлам

### 14.3 Отключение горячих клавиш

Отключены три клавиши, которые могли бы помешать автоматизированному управлению:

- Ctrl+Shift+W — случайное закрытие всех вкладок
- Ctrl+S — сохранение страницы в неизвестное место
- Ctrl+N — открытие нового окна, сбивающее фокус

Это часть дизайна: чтобы автоматизация не сломалась от случайных нажатий.

### 14.4 Расширение

Расширение `kcdongibgcplmaagnmgpjhpjgmmaaaaa` — внутренний компонент ChatGPT. В открытом Chrome Web Store его нет.

Возможные функции:

- Связь с container_daemon
- Передача навигационных команд
- Возможно, WebSocket-соединение с hive

Точная роль не установлена окончательно.

## 15. Chrome proxy configuration

### 15.1 Где настраивается

В `/usr/local/init_scripts/chrome.sh` есть логика прокси-конфигурации:

```bash
if [ -n "$OPERATOR_PROXY_URL" ]; then
    CHROME_FLAGS="$CHROME_FLAGS --proxy-pac-url=\"$OPERATOR_PROXY_URL/proxy.pac\""
elif [ -n "$HTTP_PROXY" ]; then
    CHROME_FLAGS="$CHROME_FLAGS --proxy-server=\"$HTTP_PROXY\""
fi
```

### 15.2 Возможные режимы

1. **PAC-файл** — если задан `OPERATOR_PROXY_URL`, Chrome использует URL `$OPERATOR_PROXY_URL/proxy.pac`
2. **HTTP proxy** — если задан `HTTP_PROXY`, Chrome использует его как прокси

### 15.3 Текущее состояние

В нашей сборке:

- `OPERATOR_PROXY_URL` — не задан
- `HTTP_PROXY`/`HTTPS_PROXY` — присутствуют в supervisor env, но не передаются в kernel env

Даже если бы Chrome запустился с proxy — сетевой egress всё равно закрыт на уровне Azure NSG. Прокси помог бы только для фильтрации и логирования внутри OpenAI, но не дал бы доступа в интернет.

## 16. Env-флаги

### 16.1 Управление Chrome Stack

| Флаг | Назначение |
|---|---|
| `CUA_DD_ENABLE_CHROME=true` | включить Chrome |
| `CUA_DD_INIT_CONTAINER_DAEMON=true` | включить daemon (нужен для управления Chrome) |
| `CDP_PORT=9222` | порт CDP |
| `DISPLAY=:0` | дисплей Xvfb |
| `CUA_DD_CHROME_USER=oai` | пользователь Chrome |
| `CUA_DD_CHROME_AUTO_RESTART` | авторестарт |
| `CUA_DD_CHROME_DEVTOOLS` | включить DevTools |
| `CUA_DD_CHROME_KIOSK_PRINTING` | режим печати |
| `CUA_DD_CHROME_CAAS_ARGS` | дополнительные аргументы |

### 16.2 Browser/CUA окружение (все по умолчанию не заданы)

```
CDP_PORT=9222
NEKO_PORT=8081
NEKO_PROXY_PORT=8082
NOVNC_PROXY_PORT=6902
```

### 16.3 Связанные флаги

- `CUA_DD_INIT_XVFB=true` — стартовать Xvfb
- `CUA_DD_INIT_NEKO=true` — стартовать Neko (веб-VNC)
- `CUA_DD_ENABLE_VNC=true` — включить VNC-стек
- `CUA_DD_MITMPROXY=true` — включить MITM-прокси

## 17. Как включить вручную

### 17.1 Минимальный сценарий

```bash
# 1. Xvfb
mkdir -p /tmp/.X11-unix
chmod 1777 /tmp/.X11-unix

DISPLAY=:0 XAUTHORITY=/home/oai/.Xauthority \
    nohup Xvfb :0 -screen 0 1920x1080x24 -ac > /tmp/xvfb.log 2>&1 &

sleep 2

# 2. container_daemon
export CUA_DD_INIT_CONTAINER_DAEMON=true
export DISPLAY=:0
export CDP_PORT=9222

nohup /usr/local/bin/container_daemon > /tmp/cd.log 2>&1 &

sleep 3

# 3. Проверка
ss -ltnp | grep ':8085'
ps aux | grep '[c]hromium'
curl http://127.0.0.1:9222/json/version
curl http://127.0.0.1:8085/screenshot -o /tmp/smoke.png
file /tmp/smoke.png
```

### 17.2 Через supervisorctl

Если сервисы описаны в supervisord:

```bash
export CUA_DD_ENABLE_CHROME=true
export CUA_DD_INIT_CONTAINER_DAEMON=true

supervisorctl start xvfb
supervisorctl start chromium
supervisorctl start container_daemon

supervisorctl status
```

## 18. Что не работает

| Что | Результат | Причина |
|---|---|---|
| Навигация на внешний URL | `ERR_BLOCKED_BY_ADMINISTRATOR` | `URLBlocklist ["*"]` |
| Навигация даже на локальный HTTP из Chrome | зависит от политики | ограничение политик |
| `/browser/get_state` | 404 | route не зарегистрирован |
| `/browser/js` | 404 | route не зарегистрирован |
| `/clipboard` GET | 500 | нет активного X-селектора |
| `/browser/new_page` без Chromium | 500 | browser backend недоступен |
| `/screenshot` без Xvfb | ошибка | нет X-дисплея |
| TCP-соединение наружу при блокировке | не открывается | Chrome блокирует до сети |

## 19. Health check

### 19.1 Полная проверка стека

```bash
# 1. Xvfb
ps aux | grep '[X]vfb'
ls -la /tmp/.X11-unix/X0

# 2. Chromium
ps aux | grep '[c]hromium'

# 3. CDP
curl -s http://127.0.0.1:9222/json/version | head

# 4. container_daemon
ss -ltnp | grep ':8085'

# 5. Browser subsystem
curl -i http://127.0.0.1:8085/browser/page_list

# 6. Screenshot
curl http://127.0.0.1:8085/screenshot -o /tmp/smoke.png
file /tmp/smoke.png

# 7. Проверка URLBlocklist
cat /etc/chromium/policies/managed/001_base_url_blocklist.json
```

### 19.2 Диагностика проблем

| Симптом | Что проверить |
|---|---|
| `/screenshot` возвращает ошибку | Xvfb запущен? `DISPLAY` установлен? |
| `/browser/new_page` → 500 | Chromium жив? CDP доступен? |
| CDP `connection refused` | Chromium запущен с `--remote-debugging-port`? |
| Chromium не стартует | Xvfb работает? `DISPLAY=:0`? |
| Навигация блокируется | это ожидаемое поведение, `URLBlocklist ["*"]` |

## 20. Связи

- `02-container-daemon.md` — `/browser/*` и GUI endpoints — часть container_daemon API
- `08-network-boundary.md` — Chrome Managed Policy — первый слой блокировки egress
- `01-infrastructure.md` — Chrome profile, `CUA_DD` флаги, env
- `09-hive.md` — возможная связь через Chrome Extension (не подтверждено)

## 21. Открытые вопросы

- Точный формат `/browser/state` и `/browser/page_list`
- Полные схемы `/drag`, `/compound`, `/multi_key_press_and_hold`
- Назначение и поведение `/user_context` и `/resume`
- Роль Chrome Extension `kcdongibgcplmaagnmgpjhpjgmmaaaaa` — какой именно канал использует
- Есть ли дополнительная политика для `file://` (redirect.html работает, значит что-то разрешено)
- Как платформа интерпретирует `chrome-error://chromewebdata/` — считает ли это ошибкой
- Возможно ли управление размером PTY/X11 screen через API
- Что происходит с Chrome при смене политик на лету (перечитывает ли он их)
