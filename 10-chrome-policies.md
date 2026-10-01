# 10-chrome-policies.md

**Зона:** Chrome Enterprise Managed Policies  
**Путь:** `/etc/chromium/policies/managed/`  
**Назначение:** Документация всех политик, которые Chrome читает при запуске.

## Содержание

1. Что это такое  
2. Как Chrome читает политики  
3. Коды значений  
4. Полный список файлов  
5. 000_policy_merge.json — базовый  
6. 001_base_url_blocklist.json — ключевой  
7. 001_base_extensions.json  
8. 001_base_miscellaneous.json  
9. 010_dev_*.json — dev-профиль  
10. 020_operator_*.json — оператор  
11. 020_research_*.json — research  
12. 040_caas_*.json — CaaS-специфичные  
13. 100_cua_dd_chrome_devtools.json  
14. Как политики влияют на работу  
15. Проверка политик  
16. Открытые вопросы  

# 1. Что это такое

Chrome Enterprise Managed Policies — это механизм, позволяющий администратору задавать политики браузера через JSON-файлы. Chrome читает их при старте и применяет как жёсткие правила, которые нельзя обойти из UI.

В отличие от пользовательских настроек, managed policies:

- не могут быть изменены через интерфейс Chrome
- не отключаются пользователем
- применяются ко всем профилям
- имеют приоритет над любыми пользовательскими настройками

В песочнице ChatGPT этот механизм используется для двух целей:

1. Ограничить возможности браузера — урезать функциональность до минимума  
2. Заблокировать навигацию — `URLBlocklist: ["*"]` как первый слой egress-защиты

Файлы политик лежат в `/etc/chromium/policies/managed/` и собираются из шаблонов при старте VM. Набор зависит от режима (dev / operator / research / CaaS) и флагов запуска.

# 2. Как Chrome читает политики

## 2.1 Директория

```
/etc/chromium/policies/managed/
```

Все `*.json` файлы в этой директории объединяются в одну конфигурацию. Позже по имени файла — приоритет выше (файлы с бо́льшим префиксом переопределяют меньшие).

**Порядок:**

```
000_policy_merge.json      (наименьший приоритет)
001_*.json
010_*.json
020_*.json
040_*.json
100_*.json                 (наибольший приоритет)
```

## 2.2 Когда читаются

Chrome читает политики:

- при старте браузера
- при перезагрузке политик (если поддерживается платформой)
- в некоторых сборках — периодически

В нашей песочнице перезагрузка на лету не наблюдалась — для применения новых политик требуется перезапуск Chromium.

## 2.3 Формат

Каждый файл — JSON с полями политик:

```json
{
  "PolicyName": value
}
```

Значения могут быть:

- числа (для enum-кодов)
- строки
- массивы
- объекты

## 2.4 Применение

Политики, помеченные как managed, жёстко применяются ко всем профилям. Chrome может показать значок "Managed by your organization" в UI, если заданы соответствующие политики.

# 3. Коды значений

Chrome использует числовые коды для многих политик. Общая схема:

| Код | Значение                                      |
|-----|-----------------------------------------------|
| 0   | Разрешено (allowed)                           |
| 1   | Заблокировано (blocked)                       |
| 2   | Запрещено (disallowed) — жёстче, чем blocked  |
| 3   | Force (принудительно включить)                |

Для некоторых политик используются специфичные enum'ы. Например, `DownloadRestrictions`:

| Код | Значение                                          |
|-----|---------------------------------------------------|
| 0   | Без ограничений                                   |
| 1   | Блокировать опасные загрузки                      |
| 2   | Блокировать опасные + потенциально нежелательные  |
| 3   | Блокировать все загрузки                          |

Правильная расшифровка кода требует справки по конкретной политике. Ниже — что известно по каждой из наших.

# 4. Полный список файлов

В `/etc/chromium/policies/managed/` в текущей сборке присутствуют:

```
000_policy_merge.json
001_base_extensions.json
001_base_miscellaneous.json
001_base_url_blocklist.json
010_dev_*.json
020_operator_*.json
020_research_*.json
040_caas_bing_live_search.json
040_caas_searchgpt_search.json
040_caas_next_google_docs_printing.json
040_caas_next_google_docs_search.json
100_cua_dd_chrome_devtools.json
```

Не все файлы активны одновременно. Набор зависит от режима работы VM. Часть файлов (`010_dev_*`, `020_operator_*`, `020_research_*`) применяется только в соответствующих режимах.

# 5. 000_policy_merge.json — базовый

Этот файл содержит базовый набор ограничений. В нашей сборке наблюдалось:

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

## 5.1 Построчно

| Политика                          | Значение              | Что значит                                      |
|-----------------------------------|-----------------------|-------------------------------------------------|
| DefaultDirectSocketsSetting       | 2                     | Direct Sockets запрещены                        |
| DeveloperToolsAvailability        | 0                     | DevTools недоступны                             |
| DefaultFileSystemReadGuardSetting | 2                     | File System API read запрещён                   |
| DefaultFileSystemWriteGuardSetting| 2                     | File System API write запрещён                  |
| URLBlocklist                      | ["*"]                 | Все URL заблокированы                           |
| ExtensionInstallBlocklist         | ["*"]                 | Все расширения запрещены (кроме force-list)     |
| DefaultGeolocationSetting         | 2                     | Геолокация запрещена                            |
| DefaultNotificationsSetting       | 2                     | Уведомления запрещены                           |
| DownloadDirectory                 | /home/oai/share/      | Куда сохранять файлы                            |
| DownloadRestrictions              | 1                     | Опасные загрузки блокируются                    |

## 5.2 Что это даёт

Chrome запускается в максимально ограниченном режиме:

- Никакой навигации наружу
- Никаких расширений (кроме явно разрешённых платформой)
- Никаких DevTools
- Никакого файлового доступа через Web API
- Никакой геолокации и уведомлений

Единственное, что разрешено — работа с локальными `file://` (для redirect.html) и отрисовка в Xvfb.

# 6. 001_base_url_blocklist.json — ключевой

Наименьший файл, но именно он блокирует навигацию:

```json
{
  "URLBlocklist": ["*"]
}
```

## 6.1 Что означает

Wildcard `*` — все URL запрещены. Chrome не откроет ни один внешний ресурс.

## 6.2 Что происходит при попытке навигации

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

TCP-соединение не открывается. Chrome блокирует навигацию до сетевого слоя.

## 6.3 Что это НЕ блокирует

- `file://` URL (используется для redirect.html)
- `chrome://` внутренние страницы
- `chrome-error://` страницы ошибок
- Локальные скрипты расширений

## 6.4 URLAllowlist — противоположная политика

Если в политике задан `URLAllowlist`, он работает как исключение из `URLBlocklist`. В наших логах `URLAllowlist: ["*"]` упоминался для режимов dev и research — это означает полное разрешение всех URL.

**Порядок разрешения:**

1. Если URL в `URLAllowlist` → разрешено  
2. Если URL в `URLBlocklist` → заблокировано  
3. Иначе → разрешено

Для режима CaaS: только `URLBlocklist: ["*"]`, без `URLAllowlist`. Поэтому всё заблокировано.

# 7. 001_base_extensions.json

Blocklist всех расширений:

```json
{
  "ExtensionInstallBlocklist": ["*"]
}
```

Все расширения запрещены. Разрешены только те, что добавлены через `ExtensionInstallForcelist` или `ExtensionInstallAllowlist` — это внутренний механизм платформы.

Расширение `kcdongibgcplmaagnmgpjhpjgmmaaaaa` ("ChatGPT Agent") установлено через force-list, а не через обычную установку. Это позволяет обойти блокировку.

# 8. 001_base_miscellaneous.json

Общие ограничения. В логах упоминался как жёсткая песочница:

- нет микрофона
- нет камеры
- нет автозаполнения
- нет закладок
- нет режима инкогнито
- нет сохранения паролей
- нет печати
- нет синхронизации
- нет WebRTC напрямую

Точный JSON не зафиксирован, но по названию и упоминаниям — это набор отключающих политик для приватности и периферии.

Возможные политики в этом файле:

```json
{
  "AudioCaptureAllowed": false,
  "VideoCaptureAllowed": false,
  "AutofillAddressEnabled": false,
  "AutofillCreditCardEnabled": false,
  "BookmarkBarEnabled": false,
  "IncognitoModeAvailability": 1,
  "PasswordManagerEnabled": false,
  "PrintingEnabled": false,
  "SyncDisabled": true,
  "WebRtcUdpPortRange": ""
}
```

Точное содержимое требует отдельного чтения файла.

# 9. 010_dev_*.json — dev-профиль

Применяется в режиме разработки. В логах упоминалось:

- `URLAllowlist: ["*"]` — все URL разрешены
- DevTools включены

Это противоположность production-режима: dev позволяет всё для отладки. В нашей сессии этот файл не применяется.

# 10. 020_operator_*.json — оператор

Применяется в режиме Operator (браузерный агент для выполнения задач в интернете).

В логах упоминалось:

- Расширение с `localhost:31460/agent.xml` — внутренний Operator extension
- Поиск через Bing (с политикой `040_caas_bing_live_search.json`)

Этот режим требует интернета и, соответственно, применяется на VM с `allow_internet=true`.

# 11. 020_research_*.json — research

Применяется в режиме Deep Research.

В логах упоминалось:

- `URLAllowlist: ["*"]` — все URL разрешены
- DevTools включены (для отладки)

Режим research позволяет ходить в интернет для сбора информации.

# 12. 040_caas_*.json — CaaS-специфичные

Эти файлы описывают поведение в CaaS-окружении.

## 12.1 040_caas_bing_live_search.json

Поиск через Bing. Возможно, настройка поискового движка и перехвата поисковых запросов через MITM.

## 12.2 040_caas_searchgpt_search.json

Поиск через ChatGPT. Альтернативный режим поиска.

## 12.3 040_caas_next_google_docs_printing.json

Google Docs → PDF в `/home/oai/share/`. Возможно, определяет, куда сохранять PDF, полученные при печати Google Docs.

## 12.4 040_caas_next_google_docs_search.json

Google Docs расширения. По-видимому, интеграция с Google Docs для поиска и работы с документами.

Эти файлы применяются в специфических режимах CaaS (связанных с Google Docs/Sheets интеграцией).

# 13. 100_cua_dd_chrome_devtools.json

Политика с наивысшим приоритетом. В логах упоминалась как блокирующая `devtools://` для CUA (Computer-Use Agent).

Содержимое не зафиксировано полностью, но по названию — запрет на использование DevTools для агентов CUA. Это дополняет `DeveloperToolsAvailability: 0` из базового файла.

# 14. Как политики влияют на работу

## 14.1 Навигация

Основное следствие — невозможно открыть внешние URL. Это первый слой egress-защиты (второй — Azure NSG, см. `08-network-boundary.md`).

## 14.2 Расширения

Установка произвольных расширений невозможна. Только те, что добавлены платформой.

## 14.3 DevTools

Недоступны. Нельзя открыть консоль, инспектор элементов, сетевую панель.

## 14.4 Файловая система

Web File System API отключён. Chrome не может читать/писать файлы пользователя через web-страницы.

## 14.5 Загрузки

Опасные типы файлов блокируются. `DownloadDirectory` — `/home/oai/share/`. Это значит, что любые разрешённые загрузки идут в эту папку.

## 14.6 Приватность

Геолокация, уведомления, автозаполнение, закладки — всё отключено.

# 15. Проверка политик

## 15.1 Список файлов

```bash
ls -la /etc/chromium/policies/managed/
```

## 15.2 Просмотр содержимого

```bash
cat /etc/chromium/policies/managed/000_policy_merge.json
cat /etc/chromium/policies/managed/001_base_url_blocklist.json
```

## 15.3 Поиск по всем

```bash
grep -r 'URLBlocklist\|URLAllowlist' /etc/chromium/policies/managed/
```

## 15.4 В браузере

Если бы DevTools работали, можно было бы открыть `chrome://policy` и увидеть активные политики. Но DevTools заблокированы.

## 15.5 Через API

При попытке навигации Chrome возвращает `ERR_BLOCKED_BY_ADMINISTRATOR` — это косвенное подтверждение применения политик.

# 16. Открытые вопросы

- Точное содержимое `001_base_miscellaneous.json` — какие именно периферийные API отключены  
- Точное содержимое `010_dev_*.json`, `020_operator_*.json`, `020_research_*.json` — мы их не видели в текущей сборке  
- Что именно содержит `100_cua_dd_chrome_devtools.json`  
- Применяются ли политики динамически (при перезагрузке политик) или только при старте Chrome  
- Как платформа меняет набор политик при переходе между режимами (dev → prod → operator)  
- Взаимодействие `URLBlocklist` и `URLAllowlist` при одновременном задании  

Для полного раскрытия нужно либо посмотреть на VM в разных режимах, либо иметь доступ к шаблонам политик на хосте.
