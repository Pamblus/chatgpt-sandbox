# 02 · container_daemon (`:8085`)

Этот документ описывает `container_daemon` — внутренний HTTP-сервис на порту `:8085`, написанный на Rust (Axum + Tokio). По богатству возможностей это самый мощный компонент песочницы: он объединяет управление браузером, файловую систему, выполнение команд, диагностику и сетевой слой offline-прокси.

Однако по умолчанию он выключен. И это ключевой operational-факт, который нужно понять до всего остального.

## 1. Что это такое

`container_daemon` — это отдельный бинарник `/usr/local/bin/container_daemon`, написанный на Rust. Внутри него собрано несколько подсистем:

- HTTP-сервер на Axum, слушающий `0.0.0.0:8085`
- **Browser subsystem:** обёртка над Chromium через крейт `chromiumoxide` (управление через CDP)
- **Filesystem subsystem:** чтение и запись файлов в пределах `/home/oai/**`
- **Exec subsystem:** выполнение команд внутри VM
- **Network/config:** получение внешнего IP, конфигурация, таблица offline-sites
- **Offline proxy:** forward HTTP-прокси для локальных upstream-сервисов
- **Diagnostics:** тестовый endpoint

Внутри бинарника найдены крейты:

- `axum` — HTTP-сервер
- `tokio` — асинхронный runtime
- `hyper` — низкоуровневый HTTP
- `reqwest` — HTTP-клиент
- `tungstenite` — WebSocket
- `chromiumoxide` — управление Chromium через CDP

Всё это — production-инфраструктура OpenAI, а не учебная обёртка.

## 2. Как daemon появляется

Штатная цепочка запуска:

```
supervisord (PID 1)
    │
    ▼
/usr/local/scripts/oai_service.sh
    │
    ▼
/usr/local/init_scripts/container_daemon.sh
    │
    ▼
/usr/local/bin/container_daemon
```

Внутри `container_daemon.sh` есть блок:

```bash
if [ "$CUA_DD_INIT_CONTAINER_DAEMON" = "false" ]; then
    exit 42
fi
```

По умолчанию в этой сборке:

```
CUA_DD_INIT_CONTAINER_DAEMON=false
CUA_DD_TERMINAL_MODE=false
START_PROXY=false
```

Поэтому daemon спит. Его нет в процессах, порт `:8085` не слушается.

### Три способа запустить

**Способ 1 — через supervisorctl** (если сервис описан):

```bash
export CUA_DD_INIT_CONTAINER_DAEMON=true
supervisorctl start container_daemon
```

Работает только если сервис не отключён в конфиге supervisord'а. Проверять через `supervisorctl status`.

**Способ 2 — ручной запуск в обход враппера:**

```bash
export CUA_DD_INIT_CONTAINER_DAEMON=true
export DISPLAY=:0
export CDP_PORT=9222

nohup /usr/local/bin/container_daemon > /tmp/cd.log 2>&1 &
```

Обходит `exit 42`, потому что мы не запускаем shell-скрипт, а вызываем бинарник напрямую.

**Способ 3 — с включённым offline proxy:**

```bash
export START_PROXY=true
export PROXY_PORT=9999
export CUA_DD_INIT_CONTAINER_DAEMON=true

nohup /usr/local/bin/container_daemon > /tmp/cd.log 2>&1 &
```

В этом случае поднимается ещё и forward-прокси на порту `9999`.

### Что обязательно должно быть задано

Минимальный набор env для успешного старта:

- `CUA_DD_INIT_CONTAINER_DAEMON` не равен `false`
- `DISPLAY` должен существовать (иначе browser subsystem упадёт)
- `CDP_PORT` должен быть задан (по умолчанию `9222`)

Плюс, если предполагается работа browser subsystem, должен быть запущен Xvfb на нужном `DISPLAY`.

## 3. Важное про auth middleware

При ручном запуске daemon пишет в лог:

```
skipping auth middleware
```

Это означает, что в тестовой конфигурации авторизация отключена. Все запросы к API проходят без Bearer-токена.

Это не баг, а особенность тестового режима. В production-конфигурации middleware может проверять токен, но в наших наблюдениях этого не было.

**Операционное следствие:** любой процесс внутри VM может обратиться к `:8085` без предъявления токена. Снаружи порт не проброшен, поэтому снаружи достучаться всё равно нельзя.

## 4. Проверка, что daemon работает

Три уровня проверки.

**Уровень 1 — порт слушается:**

```bash
ss -ltnp | grep ':8085'
```

Ожидаемый вывод:

```
LISTEN 0 128 0.0.0.0:8085 0.0.0.0:* users:(("container_daemon",pid=...,fd=...))
```

**Уровень 2 — процесс жив:**

```bash
ps aux | grep '[c]ontainer_daemon'
```

Ищем один процесс с бинарником `/usr/local/bin/container_daemon`.

**Уровень 3 — HTTP отвечает:**

```bash
curl -i http://127.0.0.1:8085/
```

Корневой route возвращает JSON со статусом. Подробнее — в следующем разделе.

**Проверка бинарника:**

```bash
ls -l /usr/local/bin/container_daemon
```

Должен быть ELF-бинарник. Если его нет — образ повреждён или это другая сборка.

## 5. Route: `GET /` — статус

Корневой endpoint. Возвращает JSON с информацией о сборке:

```json
{
  "error": false,
  "data": {
    "agi_status": "almost_there",
    "build": "openaiappliedcaasprod.azurecr.io/chrome-chatgpt-prod-5p4:20260803182207-770cc6d08333-linux-amd64",
    "commit_sha": "770cc6d08333...",
    "ip": "unknown"
  }
}
```

Поле `agi_status` со значением `almost_there` — курьёзная деталь, но полезная для идентификации сборки.

Поле `ip` обычно `"unknown"` — оно заполняется после успешного вызова `get_ip` при startup. Если IP так и не получен (а в нашей среде он не получается), значение остаётся `unknown`.

Этот endpoint — не API-метод в строгом смысле, а health-чек. Его задача — показать, что HTTP-сервер жив.

## 6. Полная карта routes

Ниже — все известные endpoints, разбитые по подсистемам. Для каждого указан метод, ожидаемый request и статус.

### 6.1 Browser: управление вкладками

| Route | Method | Request | Статус | Примечание |
|---|---|---|---|---|
| `/browser/new_page` | POST | `{"url": "..."}` | ✅ runtime | Создать вкладку + навигация |
| `/browser/state` | POST | `{}` | ✅ runtime | Текущее состояние браузера |
| `/browser/page_list` | POST | `{}` | ✅ runtime | Список открытых вкладок |
| `/browser/get_state` | POST | — | ❌ disproved | 404, не зарегистрирован |
| `/browser/js` | POST | — | ❌ disproved | 404, не зарегистрирован |

**Важно:** имя файла-исходника и HTTP-путь могут не совпадать. Например, в бинарнике есть функция из `get_state.rs`, но HTTP-путь `/browser/get_state` не зарегистрирован. Правильный route — `/browser/state`. Это типовая ошибка reverse-инжиниринга по именам файлов.

Пример запроса к `new_page`:

```bash
curl -X POST http://127.0.0.1:8085/browser/new_page \
    -H 'Content-Type: application/json' \
    -d '{"url":"https://example.com"}'
```

Проверенные статусы:

- POST + корректный JSON → `200`
- POST `{}` → `422` (route есть, schema не подходит)
- GET → `405` (route есть, метод не тот)
- Browser backend недоступен → `500` с сообщением `Received no response from the chromium instance`

Статус `500` здесь означает не проблему HTTP-сервера, а проблему browser subsystem. То есть HTTP жив, а Chrome не отвечает.

### 6.2 Browser: GUI actions

| Route | Method | Request | Статус |
|---|---|---|---|
| `/click` | POST | `{"x": int, "y": int, "button": u8}` | 🟢 static + частично runtime |
| `/double_click` | POST | аналогично `/click` | 🟢 static |
| `/scroll` | POST | `{"x": int, "y": int, "delta_*": ...}` | 🟢 static |
| `/move_mouse` | POST | `{"x": int, "y": int}` | 🟢 static |
| `/type` | POST | `{"text": str, ...}` | 🟢 static |
| `/paste` | POST | текст | 🟢 static |
| `/drag` | POST | `{...}` | 🟢 static |
| `/compound` | POST | составная операция | 🟢 static |
| `/multi_key_press_and_hold` | POST | `{...}` | 🟢 static |
| `/clipboard` | GET | — | 🟢 static, runtime `500` |
| `/screenshot` | GET | — | ✅ runtime |
| `/user_context` | GET | — | 🟢 static |
| `/resume` | POST | — | 🟢 static |

Для `/click` и `/double_click` поле `button` — это `u8` (0/1/2 для left/middle/right).

**`/screenshot` подробно:**

- Возвращает `image/png`
- Требует работающего Xvfb и Chromium
- В одном из экспериментов размер ответа ~68 383 байта
- До запуска Xvfb — ошибка
- После запуска Xvfb — `200 image/png`

Пример:

```bash
curl http://127.0.0.1:8085/screenshot -o /tmp/screenshot.png
file /tmp/screenshot.png
```

Ожидаемо: `PNG image data`.

Это важно понимать: `/screenshot` зависит от всего стека: daemon → browser subsystem → X11 → Xvfb. Если хотя бы одно звено не работает, endpoint падает.

### 6.3 Filesystem

| Route | Method | Request | Статус |
|---|---|---|---|
| `/file` | GET | `?path=/home/oai/...` | ✅ runtime |
| `/files` | POST | `{"files":{path: base64}}` | ✅ runtime |
| `/files/:path` | любой | — | ❌ disproved (404) |
| `/list` | GET/POST | — | 🟢 static |

Про `/file` и `/files` подробно — в отдельном файле `05-filesystem.md`. Здесь только факт: guard работает на `/home/oai/**`, всё остальное отсекается с сообщением `blocked path=/...`.

### 6.4 Exec

| Route | Method | Request | Статус |
|---|---|---|---|
| `/exec` | POST | `{"cmd": [...], ...}` | 🟢 static |

Точная JSON-схема не восстановлена. Известно, что endpoint существует и предназначен для выполнения команд. Отличается от shell-порта `:1384` — это отдельный механизм внутри daemon.

### 6.5 Network / Config

| Route | Method | Request | Статус |
|---|---|---|---|
| `/get_ip` | GET | — | 🟢 static |
| `/config` | GET | — | ✅ runtime |
| `/offline_sites` | GET | — | ✅ runtime |
| `/offline_sites` | POST | `{"host": "forward-address"}` | ✅ runtime |

- `/config` возвращает состояние Chrome-окружения: текущий URL, IP, локаль. Используется платформой для понимания состояния browser subsystem.
- `/offline_sites` — таблица Host → forward address для offline_proxy. Подробно ниже.
- `/get_ip` — endpoint получения IP. Но реализация интереснее, чем сам endpoint: при старте daemon автоматически вызывает внешний сервис `https://api.ipify.org/` и сохраняет результат в `AppState`. Подробно ниже.

### 6.6 Diagnostics

| Route | Method | Request | Статус |
|---|---|---|---|
| `/test` | GET/POST | — | 🟢 static |

Существует, конкретный контракт не восстановлен. Может использоваться как быстрый health-check: `curl -i http://127.0.0.1:8085/test`.

## 7. `get_ip` — startup chain

Это один из самых интересных моментов. Разберём подробно.

### Что происходит при старте

При инициализации daemon (в функции `server::create`) выполняется следующий код:

1. Загружается URL `https://api.ipify.org/` (22 символа)
2. Вызывается `reqwest::Client::get()` с этим URL
3. Получается HTTP response
4. Читается body
5. Из body извлекается IP-адрес
6. Результат сохраняется через `container_daemon::server::handlers::config::set_ip` в `AppState`

В ассемблерном виде это выглядит так:

```
0x602bfe: lea r13, [rip+...]     → "https://api.ipify.org/"
0x602c0c: mov ..., 0x16          → длина URL = 22
0x602c9f: call reqwest::...Client::get
...
          → body
          → container_daemon::server::handlers::config::set_ip
```

То есть вызов происходит до того, как HTTP-сервер начинает принимать запросы. Это часть bootstrap.

### Зачем это сделано

`api.ipify.org` — публичный сервис, возвращающий внешний IP клиента. Платформа использует это, чтобы узнать, какой IP у VM с точки зрения внешнего мира. Это важно для логирования и, вероятно, для некоторых внутренних проверок.

### Runtime-результат

Однако при ручном запуске в текущей среде вызов `get_ip` не завершается успешно.

Лог показывает:

```
getaddrinfo("api.ipify.org")
    ↓
gai rc = -3
```

`gai rc = -3` — это код ошибки из `getaddrinfo`. Он означает временный сбой разрешения имени. То есть DNS не может отрезолвить `api.ipify.org`, и TCP-соединение даже не начинается.

Практическое следствие: `set_ip` вызывается, но с пустым или неизвестным значением. Поэтому в `/`-статусе поле `ip: "unknown"`.

Это согласуется с общей моделью egress: DNS закрыт так же, как и TCP. Даже если daemon специально сконфигурирован для внешнего запроса, сетевой путь не работает.

### Что это значит для понимания системы

Ранняя гипотеза (из первых логов) звучала так: «возможно, daemon имеет эксклюзивный сетевой канал наружу, недоступный другим процессам». Runtime это опровергает: daemon упирается в тот же DNS-барьер, что и shell, Python и Chrome.

Строка `api.ipify.org` в бинарнике не мёртвая — код действительно исполняется. Но работает он в тех же сетевых условиях, что и всё остальное внутри VM.

## 8. offline_proxy — подсистема

Это отдельная подсистема внутри `container_daemon`, отвечающая за forward HTTP-прокси. Не путать с обычным интернет-прокси: назначение — перенаправлять запросы к локальным upstream-сервисам по имени хоста.

### Как активируется

По умолчанию `START_PROXY=false`. Чтобы включить:

```bash
export START_PROXY=true
export PROXY_PORT=9999
export CUA_DD_INIT_CONTAINER_DAEMON=true

nohup /usr/local/bin/container_daemon > /tmp/cd.log 2>&1 &
```

Порт `PROXY_PORT` — по умолчанию `9999`.

Если `START_PROXY=false`, в логе появляется сообщение:

```
SKIP offline_proxy
```

То есть подсистема даже не пытается стартовать.

### Модули

В бинарнике присутствуют:

- `container_daemon::offline_proxy::proxy`
- `container_daemon::offline_proxy::server`

Функции: `proxy_request`, `handler`, `create`.

### Pipeline обработки запроса

Когда запрос приходит через прокси:

1. Извлекается `Host` заголовка
2. Ищется в таблице `offline_sites`
3. Находится forward address
4. Разбирается оригинальный URI через `StrSearcher::next_back("/")`
5. Формируется новый URI через `format!("https://{forward}/{prefix}?{suffix}")`
6. Строка превращается в `Bytes::from(String)`
7. `Uri::from_shared(Bytes)` — построение URI
8. Отправляется через `hyper_util::client::legacy::Client<HttpsConnector<HttpConnector>>`

Wildcard route у offline-прокси: `/*path` — принимает любой путь.

### Ошибки

Полная таблица ошибок, которые можно увидеть в логе:

| Сообщение | Значение |
|---|---|
| `no forward address found for host` | Host отсутствует в `offline_sites` |
| `couldn't create updated URI to forward (InvalidUri(...))` | Не удалось собрать URI |
| `InvalidMessage(InvalidContentType)` | Hyper получил ответ и не смог распарсить |
| `tcp connect error: Connection refused` | TCP до upstream не прошёл |
| `dns error: failed to lookup address information` | DNS не сработал |

### Что работает и что нет

**Работает:**

- Внутренние upstream на `127.0.0.1:8080`, `:1384`, `:8085`
- TCP-соединение проходит
- `curl -x http://127.0.0.1:9999 http://myhost.local/status` уходит на заданный upstream

**Не работает:**

- `CONNECT`-метод → `404` (только обычные HTTP-запросы)
- Внешние адреса → `RST` (egress закрыт)
- HTTPS наружу → тот же `RST`

### Нормализация Host

При разборе значения `Host`:

- `"allow"` → невалидный, ошибка `no forward address found`
- `"example.com:80"` → валидный, TCP connect попытается
- `"foo:bar:baz"` → `InvalidUri(InvalidAuthority)`
- `"http://127.0.0.1:8080"` → нормализуется в `127.0.0.1:8080`

## 9. `/offline_sites` — таблица маршрутизации

Это endpoint, через который внешний управляющий слой (или ручной вызов) наполняет таблицу forward-адресов.

### Handler

```rust
add_offline_sites::handler(
    State<Arc<AppState>>,
    Json<HashMap<String, String>>
)
```

Принимает JSON-словарь строк: `host → forward-address`.

Пример запроса:

```bash
curl -X POST http://127.0.0.1:8085/offline_sites \
    -H 'Content-Type: application/json' \
    -d '{"example.test": "127.0.0.1:8080"}'
```

После этого offline_proxy знает: если придёт запрос с `Host: example.test`, надо перенаправить его на `127.0.0.1:8080`.

### Куда пишется состояние

`AppState` — глобальное состояние daemon. Записанное через `/offline_sites` mapping живёт в `AppState` и используется offline_proxy напрямую. Это доказывается связкой:

- endpoint `/offline_sites` (handler принимает `Json<HashMap<String, String>>`)
- offline_proxy ищет адрес по host
- при отсутствии — сообщение `no forward address found for host`

### Концептуально

```
HTTP request с Host: example.test
    │
    ▼
offline_proxy
    │
    ▼
lookup offline_sites["example.test"]
    │
    ▼
forward address = 127.0.0.1:8080
    │
    ▼
URI reconstruction
    │
    ▼
hyper Client → upstream
```

Если mapping отсутствует — прокси возвращает ошибку и запрос не проходит.

### Что это НЕ такое

Это не интернет-прокси. Это механизм для локальной маршрутизации: дать браузеру (или другому компоненту) возможность обращаться к внутренним сервисам по именам. Например, чтобы Chrome открывал `http://myhost.local/status`, а запрос физически шёл на `127.0.0.1:8080/status`.

Наружу через этот прокси всё равно не выйти — egress закрыт на уровне Azure NSG / sidecar.

## 10. Browser subsystem — как daemon работает с Chrome

`container_daemon` не является самим браузером. Он — controller, который через CDP (Chrome DevTools Protocol) управляет Chromium.

Полная цепочка:

```
HTTP request
    │
    ▼
container_daemon :8085
    │
    ▼
Rust browser layer (chromiumoxide)
    │
    ▼
CDP / WebSocket :9222
    │
    ▼
Chromium
    │
    ▼
X11
    │
    ▼
Xvfb :0
```

Проверка, что CDP работает:

```bash
curl http://127.0.0.1:9222/json/version
```

Если Chromium запущен, вернётся JSON с информацией о браузере и CDP.

### Параметры запуска Chromium

Chrome стартует с флагами:

- `--remote-debugging-port=9222`
- `--user-data-dir=/home/oai/.chromium`

Профиль — «ChatGPT Agent», с минимальным набором файлов (`Local State`, `Default/Preferences`, без History / Cookies / Login Data).

### Что происходит при попытке навигации наружу

Даже если Chromium запущен, навигация на внешние адреса блокируется на уровне Chrome Managed Policy. Подробно — в `06-browser-stack.md`. Кратко:

- В `/etc/chromium/policies/managed/001_base_url_blocklist.json` стоит `URLBlocklist: ["*"]`
- Любая навигация на внешний URL → `ERR_BLOCKED_BY_ADMINISTRATOR`
- В ответе на `/browser/new_page` приходит HTML error-page Chrome (~249 800 байт)

### Что значит «browser subsystem недоступен»

Если Chromium не запущен, при попытке `POST /browser/new_page` daemon вернёт:

```
500 Received no response from the chromium instance
```

Это означает, что HTTP-сервер daemon жив, но его downstream (Chromium через CDP) не отвечает. Правильная интерпретация: проблема не в HTTP, а в browser backend.

## 11. Что явно не работает

| Endpoint / ситуация | Результат |
|---|---|
| `/files/:path` | `404` (не зарегистрирован в текущей сборке) |
| `/browser/get_state` | `404` |
| `/browser/js` | `404` |
| `CONNECT` через offline_proxy | `404` |
| Навигация наружу через Chrome | `ERR_BLOCKED_BY_ADMINISTRATOR` |
| `/clipboard` GET | `500` (не `200`) |
| `/file` на директорию | `500 Is a directory (os error 21)` |
| Внешний адрес через offline_proxy | TCP `RST` |

## 12. Проверенный smoke-test

После запуска VM (или после ручного поднятия daemon):

```bash
# 1. Порт
ss -ltnp | grep ':8085'

# 2. Процесс
ps aux | grep '[c]ontainer_daemon'

# 3. Корневой статус
curl -i http://127.0.0.1:8085/

# 4. Diagnostics
curl -i http://127.0.0.1:8085/test

# 5. Browser (если Xvfb + Chromium запущены)
curl -i http://127.0.0.1:8085/browser/page_list

# 6. CDP напрямую
curl -s http://127.0.0.1:9222/json/version

# 7. Screenshot
curl -s http://127.0.0.1:8085/screenshot -o /tmp/smoke.png
file /tmp/smoke.png
```

Это проверяет четыре независимых уровня: процесс, HTTP daemon, browser integration, CDP.

## 13. Связи с другими зонами

- `05-filesystem.md` — файловый API (`/file`, `/files`) детально
- `06-browser-stack.md` — Chrome policies, Xvfb, CDP, redirect.html
- `07-artifact-workbook.md` — Artifact Tool RPC (отдельный сервис, не daemon)
- `08-network-boundary.md` — get_ip runtime fail, offline_proxy SKIP, egress
- `09-hive.md` — hive — отдельный бинарник, не подсистема daemon

## 14. Что остаётся не до конца выясненным

- Точная JSON-схема `/exec` — endpoint есть, полный контракт не восстановлен
- Точная JSON-схема `/list` — назначение понятно, схема нет
- Точная семантика `/test` — существует, поведение не зафиксировано
- Точный контракт `/get_ip` как HTTP-endpoint (не только startup вызова) — сам endpoint есть, response schema не восстановлена
- Полные payload schemas для `/drag`, `/compound`, `/multi_key_press_and_hold` — известны имена, но точные поля не проверялись
- Внутренние browser operations (`history`, `change`, `close`, `get_page`, `get_page_text`, `network_rule`, `reset`) — присутствуют в реализации, но не факт, что каждая имеет отдельный HTTP route
