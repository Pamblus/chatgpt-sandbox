# 12-cua-dd-flags.md

**Зона:** Флаги `CUA_DD_*` и связанные env-переменные  
**Назначение:** Полный справочник внутренних флагов, которые управляют запуском сервисов и поведением VM.  
**Использование:** Когда нужно понять, что делает конкретный флаг, или какие есть способы включить тот или иной сервис.

## Содержание

1. Что такое CUA_DD  
2. Префиксы и семейства  
3. Режимы работы  
4. Инициализация сервисов  
5. Chrome  
6. Порты  
7. Пользователи  
8. Behavior  
9. Network и MITM  
10. Screenshot и monitoring  
11. Container Daemon CD  
12. Eval  
13. Прочие  
14. Как работают флаги в init-скриптах  
15. Как использовать вручную  
16. Открытые вопросы  

# 1. Что такое CUA_DD

`CUA_DD` — это префикс группы переменных окружения, которые управляют работой песочницы.

**Расшифровка:**

- **CUA** — Computer-Use Agent (агент, работающий с компьютером)  
- **DD** — Data Daemon (внутренний компонент, отвечающий за данные и сервисы)

Вместе: CUA Data Daemon — набор флагов конфигурации для всего, что связано с Computer-Use Agent и его инфраструктурой.

## 1.1 Где читаются

Флаги читаются:

- supervisord при старте VM
- init-скриптами (`/usr/local/init_scripts/*.sh`)
- враппером `/usr/local/scripts/oai_service.sh`
- самими сервисами при запуске

## 1.2 Когда применять

Флаги задаются до старта VM (на уровне платформы) или вручную при запуске сервиса. После того как процесс запущен, изменить флаг для него нельзя — только перезапустить.

## 1.3 Значения

По умолчанию большинство флагов выключены (`false` или не заданы). Active-сервисы (`:8080`, `:1384`) работают независимо от флагов.

# 2. Префиксы и семейства

Помимо `CUA_DD_*`, есть несколько других префиксов, каждый со своим назначением.

| Префикс     | Семейство                          |
|-------------|------------------------------------|
| CUA_DD_*    | флаги Computer-Use Agent           |
| NEBULA_*    | платформа управления VM            |
| MITM_*      | MITM-прокси (Chaussette)           |
| ACE_*       | Applied Computing Engine           |
| OAI_*       | внутренние флаги OpenAI            |
| JUPYTER_*   | Jupyter server                     |
| TERMINAL_*  | terminal server                    |
| CDP_*       | Chrome DevTools Protocol           |
| NEKO_*      | Neko browser                       |
| NOVNC_*     | noVNC                              |

Флаги `CUA_DD_*` — самые многочисленные, около 60 штук.

# 3. Режимы работы

Определяют базовое поведение VM.

## 3.1 CUA_DD_TERMINAL_MODE

**Что:** режим «только терминал», без GUI.  
**Возможные значения:** `true` / `false`

**Поведение при true:**

- не запускается GUI-стек (Xvfb, Chromium, VNC)
- отключаются browser-сервисы
- остаётся shell через `:1384`

**Когда используется:** для задач, требующих только shell/Python, без браузера.

## 3.2 CUA_DD_DESKTOP_MODE

**Что:** режим «полный рабочий стол».  
**Возможные значения:** `true` / `false`

**Поведение при true:**

- запускается Xvfb, X11VNC, Neko, Openbox, XFCE4
- включается графический интерфейс
- доступен VNC через проброшенные порты

**Когда используется:** для Computer-Use Agent, работающего с рабочим столом.

## 3.3 CUA_DD_TERMINAL_LOCAL_LOOPBACK

**Что:** ограничить терминал loopback-интерфейсом.  
**Возможные значения:** `true` / `false`

**Поведение при true:**

- `terminal_server` слушает только `127.0.0.1`
- снаружи недоступен

**Когда используется:** в конфигурациях с повышенной изоляцией.

# 4. Инициализация сервисов

Определяют, какие сервисы запускать при старте VM.

## 4.1 CUA_DD_INIT_TERMINAL_SERVER

**Что:** запускать `terminal_server`.  
**Значение по умолчанию:** `true` (сервис активен)  
**Порт:** `:1384`

## 4.2 CUA_DD_INIT_CONTAINER_DAEMON

**Что:** запускать `container_daemon`.  
**Значение по умолчанию:** `false` (спящий)  
**Порт:** `:8085`

**Поведение при false:** init-скрипт завершается с `exit 42`, сервис не запускается.

## 4.3 CUA_DD_INIT_HIVE

**Что:** запускать `hive`.  
**Значение по умолчанию:** `false` (спящий)  
**Порт:** `:50939`

## 4.4 CUA_DD_PYTHON_TOOL

**Что:** запускать `python_tool` (Jupyter).  
**Значение по умолчанию:** `true`  
**Порт:** `:8080`

## 4.5 CUA_DD_ENABLE_CHROME

**Что:** запускать Chromium.  
**Значение по умолчанию:** `false`  
**Порт CDP:** `:9222`

## 4.6 CUA_DD_MITMPROXY

**Что:** запускать mitmproxy.  
**Значение по умолчанию:** `false`  
**Порт:** `:4444`

## 4.7 CUA_DD_ENABLE_VNC

**Что:** запускать VNC-стек (X11VNC + noVNC).  
**Значение по умолчанию:** `false`  
**Порты:** `:5901` (X11VNC), `:6902` (noVNC)

## 4.8 CUA_DD_ENABLE_NOTEBOOK_SERVER

**Что:** запускать Jupyter notebook (интерфейс, не kernel).  
**Значение по умолчанию:** `false`  
**Порт:** `:8888`

## 4.9 GUI-компоненты

Отдельные флаги для каждого компонента GUI:

| Флаг                  | Что запускает                          |
|-----------------------|----------------------------------------|
| CUA_DD_INIT_XVFB      | Xvfb (виртуальный X-сервер)            |
| CUA_DD_INIT_X11VNC    | X11VNC                                 |
| CUA_DD_INIT_NEKO      | Neko (браузер через браузер)           |
| CUA_DD_INIT_OPENBOX   | Openbox (оконный менеджер)             |
| CUA_DD_INIT_XFCE4     | XFCE4 (десктоп-окружение)              |
| CUA_DD_INIT_PICOM     | Picom (композитор)                     |
| CUA_DD_INIT_DBUS      | D-Bus (межпроцессная шина)             |
| CUA_DD_INIT_NGINX     | nginx (в некоторых сборках — фронт для daemon) |

# 5. Chrome

Управляют поведением браузера.

## 5.1 CUA_DD_CHROME_USER

**Что:** от какого пользователя запускать Chrome.  
**Значение:** `oai`

## 5.2 CUA_DD_CHROME_AUTO_RESTART

**Что:** автоматически перезапускать Chrome при падении.  
**Возможные значения:** `true` / `false`

**Поведение при true:** если Chrome умирает, supervisord запускает его заново.

## 5.3 CUA_DD_CHROME_DEVTOOLS

**Что:** включать DevTools.  
**Возможные значения:** `true` / `false`

**Поведение при true:** `DeveloperToolsAvailability` может быть установлено в `1` (разрешено).

**Примечание:** в нашей сборке Chrome policies принудительно отключают DevTools. Флаг сам по себе может не помочь.

## 5.4 CUA_DD_CHROME_KIOSK_PRINTING

**Что:** режим печати для kiosk-режима.  
**Возможные значения:** `true` / `false`

**Назначение:** разрешить печать в ограниченном режиме (для интеграции с Google Docs).

## 5.5 CUA_DD_CHROME_CAAS_ARGS

**Что:** дополнительные аргументы командной строки Chrome.  
**Формат:** строка с аргументами, разделёнными пробелами.

**Пример использования:** `--disable-features=X,Y --no-sandbox`

**Назначение:** тонкая настройка Chrome в CaaS-окружении.

# 6. Порты

Задают порты для сервисов.

## 6.1 CUA_DD_CONTAINER_DAEMON_PORT

**Значение:** `8085`  
**Назначение:** порт `container_daemon`.

## 6.2 CUA_DD_HIVE_PORT

**Значение:** `50939`  
**Назначение:** порт `hive`.

## 6.3 CUA_DD_NGINX_PORT

**Значение:** `8080`  
**Назначение:** порт nginx.

**Примечание:** тот же порт, что у `python_tool`. Если оба запущены — конфликт. Обычно используется один из двух.

## 6.4 CUA_DD_DUO_APP_SERVER_PORT

**Значение:** не зафиксировано  
**Назначение:** внутренний сервис `duo_app_server`.

**Детали:** в логах только упоминание флага, без значения.

## 6.5 CUA_DD_PDF_READER_PORT

**Значение:** не зафиксировано  
**Назначение:** порт PDF reader service.

**Детали:** упоминается как часть PDF-интеграции.

# 7. Пользователи

Определяют, от каких пользователей работают сервисы.

## 7.1 CUA_DD_TERMINAL_SERVER_USER

**Значение:** `oai`  
**Назначение:** пользователь для `terminal_server`.

## 7.2 CUA_DD_PYTHON_TOOL_USER

**Значение:** `oai`  
**Назначение:** пользователь для `python_tool`.

## 7.3 CUA_DD_CHROME_USER

**Значение:** `oai`  
**Назначение:** пользователь для Chrome.

Во всех случаях используется пользователь `oai`. Плюс есть группа `oai_shared` для общих файлов.

# 8. Behavior

Управляют поведением сервисов и инструментов.

## 8.1 Artifact Tool

**CUA_DD_INIT_ARTIFACT_TOOL_V2**

- **Что:** включать Artifact Tool v2.  
- **Возможные значения:** `true` / `false`  
- **Назначение:** активирует новый RPC-движок для офисных документов.

**CUA_DD_INIT_ARTIFACT_TOOL_V2_RECORD_OPERATIONS**

- **Что:** записывать semantic operations.  
- **Возможные значения:** `true` / `false`  
- **Поведение при true:** каждая мутация Workbook оборачивается в `record()` и даёт `operations` в ответе.

**CUA_DD_INIT_ARTIFACT_TOOL_WARN_ABOUT_OVERLAPS_ON_EXPORT**

- **Что:** предупреждать о пересечениях при экспорте.  
- **Возможные значения:** `true` / `false`  
- **Назначение:** если при экспорте документа есть перекрывающиеся элементы — выдавать предупреждение.

## 8.2 Python Tool

**CUA_DD_PYTHON_TOOL_WARM_SPREADSHEET_RUNTIME**

- **Что:** прогревать spreadsheet runtime заранее.  
- **Возможные значения:** `true` / `false`  
- **Поведение при true:** при старте `python_tool` превентивно загружает runtime Artifact Tool, чтобы первый вызов был быстрее.

**CUA_DD_PYTHON_TOOL_DISABLE_MATPLOTLIB_SUPPORT**

- **Что:** отключать поддержку matplotlib.  
- **Возможные значения:** `true` / `false`  
- **Поведение при true:** matplotlib-графики не будут автоматически отображаться в UI (callback `display_matplotlib_image_to_user` может не работать).

## 8.3 Вспомогательные сервисы

**CUA_DD_INIT_REMOVE_CONTAINER_SKILLS**

- **Что:** удалять container skills при старте.  
- **Возможные значения:** `true` / `false`  
- **Поведение при true:** директория `/home/oai/skills/` очищается.

**CUA_DD_BING_AT_HOME**

- **Что:** Bing at Home режим.  
- **Возможные значения:** `true` / `false`  
- **Назначение:** вероятно, локальный режим поиска через Bing (без внешнего интернета).

**CUA_DD_PDF_READER_SERVICE**

- **Что:** включать сервис PDF reader.  
- **Возможные значения:** `true` / `false`  
- **Назначение:** сервис для чтения/парсинга PDF.

**CUA_DD_STARTUP_LIBREOFFICE**

- **Что:** стартовать LibreOffice при запуске VM.  
- **Возможные значения:** `true` / `false`  
- **Назначение:** LibreOffice используется для конвертации форматов (docx, xlsx, pptx → другие).

# 9. Network и MITM

## 9.1 CUA_DD_MITM_NETWORK_CONFIG

**Что:** конфигурация MITM-сети.  
**Формат:** вероятно, JSON или путь к файлу.  
**Назначение:** описывает, какие хосты перехватывать через MITM.

## 9.2 CUA_DD_MITMPROXY

**Что:** включать mitmproxy.  
**Возможные значения:** `true` / `false`  
**Порт:** `:4444`  
**Назначение:** перехват HTTPS-трафика для анализа или модификации.

## 9.3 Связанные MITM-переменные

Полный список MITM-переменных (не `CUA_DD`, но относятся к той же инфраструктуре):

| Переменная                    | Назначение                  |
|-------------------------------|-----------------------------|
| MITM_LOG_LEVEL                | уровень логов               |
| MITM_SERP_NO_CACHE            | Bing без кэша               |
| MITM_SERP_BING                | перехват Bing               |
| MITM_WEBCACHE_HOST            | хост кэша веб-трафика       |
| MITM_WEBCACHE_CALLER_ID       | ID клиента                  |
| MITM_WEBCACHE_CALLER_SECRET   | секрет                      |
| MITM_WEBCACHE_IDENTITY        | идентификатор               |
| MITM_WEBCACHE_TIMEOUT_MS      | таймаут                     |
| MITM_WEBCACHE_EGRESS          | egress                      |
| MITM_WEBCACHE_V3_HOST         | v3 кэша                     |
| MITM_OFFLINE_GOOGLE_DOC_HOST  | офлайн-хост Google Docs     |

# 10. Screenshot и monitoring

Управляют поведением скриншотов и логированием.

## 10.1 CUA_DD_SCREENSHOT_MIDDLEWARE

**Что:** включать middleware для скриншотов.  
**Возможные значения:** `true` / `false`  
**Назначение:** автоматически добавлять обработку скриншотов в pipeline.

## 10.2 CUA_DD_SCREENSHOT_DRAW_RED_POSITION_DOT

**Что:** рисовать красную точку в позиции курсора.  
**Возможные значения:** `true` / `false`

**Поведение при true:** в центре скриншота будет видна красная точка — это позиция, где курсор.

**Зачем:** помогает модели понимать, где находится курсор, при работе с UI.

## 10.3 CUA_DD_SCREENSHOT_WAIT_FOR_RESOURCES

**Что:** ждать загрузки ресурсов перед скриншотом.  
**Возможные значения:** `true` / `false`

**Поведение при true:** скриншот берётся только после того, как все ресурсы страницы загружены.

## 10.4 CUA_DD_SCREENSHOT_DELAY

**Что:** задержка перед скриншотом.  
**Формат:** миллисекунды.  
**Назначение:** подождать N мс перед снятием, чтобы страница успела отрисоваться.

# 11. Container Daemon CD

Специфичные для `container_daemon` флаги. Префикс CD = Container Daemon.

## 11.1 CUA_DD_CD_LOG_MIDDLEWARE_FILTER_RE

**Что:** regex для фильтрации логов `container_daemon`.  
**Формат:** строка-regex.  
**Назначение:** из потока логов выбирать только те, что подходят под паттерн. Полезно для уменьшения объёма логов.

## 11.2 CUA_DD_CD_BROWSER_CONNECTION_MODE

**Что:** режим подключения browser subsystem.  
**Возможные значения:** не зафиксировано.  
**Назначение:** определяет, как daemon подключается к Chromium через CDP.

## 11.3 CUA_DD_CD_NAVIGATE_TIMEOUT_MS

**Что:** таймаут навигации.  
**Формат:** миллисекунды.  
**Назначение:** сколько ждать ответа от Chromium при навигации. По умолчанию, вероятно, 30000 (30 секунд).

## 11.4 CUA_DD_CD_WAIT_FOR_RESOURCES_*

**Что:** группа флагов для ожидания ресурсов.  
**Формат:** семейство переменных с суффиксами.  
**Назначение:** контроль над тем, когда daemon считает страницу «загруженной».

## 11.5 CUA_DD_CD_OPERATOR_STEALTH_MODE_TIMEOUT_MS

**Что:** таймаут для stealth-режима Operator.  
**Формат:** миллисекунды.  
**Назначение:** в Operator-режиме Chrome работает в stealth-режиме (скрывает автоматизацию). Этот таймаут определяет, сколько ждать в этом режиме.

## 11.6 CUA_DD_CD_PASTE_PYAUTOGUI_TYPEWRITE

**Что:** использовать pyautogui для ввода текста вместо нативного.  
**Возможные значения:** `true` / `false`

**Поведение при true:** ввод эмулируется через pyautogui typewrite, что даёт более «человеческое» поведение.

# 12. Eval

Флаги для eval-режима (используются при тестировании модели).

## 12.1 CUA_DD_SAMPLE_ID_URL

**Что:** URL для получения sample id.  
**Формат:** URL.  
**Назначение:** в eval-режиме приложение запрашивает sample id по этому URL.

## 12.2 CUA_DD_EXPERIMENT_NAME

**Что:** имя эксперимента.  
**Формат:** строка.  
**Назначение:** идентификатор эксперимента для логирования и аналитики.

## 12.3 EVAL_TASK_ID

**Что:** ID задачи в eval-режиме.  
**Формат:** строка.  
**Особенность:** без префикса `CUA_DD_`.  
**Назначение:** идентификатор конкретной задачи.

# 13. Прочие

## 13.1 CUA_DD_NEXUS_HEALTH_CHECK

**Что:** healthcheck для nexus.  
**Возможные значения:** `true` / `false`  
**Назначение:** включает проверку живости nexus (внутренний сервис).

## 13.2 CUA_DD_COMPUTER_WAIT_MS

**Что:** задержка для Computer-Use Agent.  
**Формат:** миллисекунды.  
**Назначение:** сколько ждать между действиями Computer-Use Agent (клик → пауза → следующий клик).

## 13.3 CUA_DD_VM_BUILD

**Что:** полное имя сборки VM.  
**Значение:** `openaiappliedcaasprod.azurecr.io/chrome-chatgpt-prod-5p4:20260803182207-770cc6d08333-linux-amd64`  
**Назначение:** используется для идентификации конкретной сборки в логах.

**Примечание:** это часть ядра идентификации, но формально не флаг поведения.

# 14. Как работают флаги в init-скриптах

## 14.1 Общий враппер

`/usr/local/scripts/oai_service.sh` читает флаги и решает, запускать сервис или нет.

Условная схема:

```bash
if [ "$CUA_DD_INIT_CONTAINER_DAEMON" = "true" ]; then
    exec /usr/local/init_scripts/container_daemon.sh
else
    exit 0
fi
```

## 14.2 Проверка в конкретном скрипте

`/usr/local/init_scripts/container_daemon.sh` содержит:

```bash
if [ "$CUA_DD_INIT_CONTAINER_DAEMON" = "false" ]; then
    exit 42
fi
```

Если флаг `false` — скрипт завершается с кодом 42 (нестандартный код, чтобы отличить от обычной ошибки).

## 14.3 Обход проверки

Чтобы запустить сервис вручную в обход враппера:

```bash
CUA_DD_INIT_CONTAINER_DAEMON=true DISPLAY=:0 CDP_PORT=9222 \
    /usr/local/bin/container_daemon
```

Мы вызываем бинарник напрямую, а не shell-скрипт. Проверка `exit 42` не срабатывает.

# 15. Как использовать вручную

## 15.1 Пример: включить container_daemon

```bash
export CUA_DD_INIT_CONTAINER_DAEMON=true
export DISPLAY=:0
export CDP_PORT=9222

nohup /usr/local/bin/container_daemon > /tmp/cd.log 2>&1 &
```

## 15.2 Пример: включить hive

```bash
export CUA_DD_INIT_HIVE=true
export CUA_DD_HIVE_PORT=50939

nohup /usr/local/bin/hive > /tmp/hive.log 2>&1 &
```

## 15.3 Пример: включить offline_proxy

```bash
export START_PROXY=true
export PROXY_PORT=9999
export CUA_DD_INIT_CONTAINER_DAEMON=true

nohup /usr/local/bin/container_daemon > /tmp/cd.log 2>&1 &
```

## 15.4 Пример: запустить Chrome stack

```bash
# Xvfb
mkdir -p /tmp/.X11-unix && chmod 1777 /tmp/.X11-unix
DISPLAY=:0 XAUTHORITY=/home/oai/.Xauthority \
    nohup Xvfb :0 -screen 0 1920x1080x24 -ac > /tmp/xvfb.log 2>&1 &

sleep 2

# container_daemon с Chrome
export CUA_DD_ENABLE_CHROME=true
export CUA_DD_INIT_CONTAINER_DAEMON=true
export DISPLAY=:0
export CDP_PORT=9222

nohup /usr/local/bin/container_daemon > /tmp/cd.log 2>&1 &
```

## 15.5 Через supervisorctl

Если сервисы описаны в supervisord:

```bash
export CUA_DD_INIT_CONTAINER_DAEMON=true
supervisorctl start container_daemon

supervisorctl status
```

## 15.6 Проверка после запуска

```bash
ss -ltnp | grep ':8085'
ps aux | grep '[c]ontainer_daemon'
curl -i http://127.0.0.1:8085/
```

# 16. Открытые вопросы

- Точные значения по умолчанию для флагов, где мы видели только упоминание (`CUA_DD_DUO_APP_SERVER_PORT`, `CUA_DD_PDF_READER_PORT`)
- Полное содержимое флагов `CUA_DD_CD_WAIT_FOR_RESOURCES_*` — какие именно суффиксы используются
- Взаимодействие `CUA_DD_CHROME_DEVTOOLS` с Chrome policies — можно ли включить DevTools через флаг, или policies всегда переопределяют
- Что именно прогревает `CUA_DD_PYTHON_TOOL_WARM_SPREADSHEET_RUNTIME`
- Формат `CUA_DD_MITM_NETWORK_CONFIG` — JSON или путь к файлу
- Полный список флагов, которые мы не видели — есть ли ещё
- Как платформа выбирает набор флагов при создании VM (в зависимости от режима: chat / operator / research)
- Есть ли флаги, которые можно менять на лету (без перезапуска сервиса)

Для практической работы уже известных флагов достаточно. Остальное — область дальнейшего исследования.
