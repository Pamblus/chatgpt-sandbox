# 99-reference.md

**Назначение:** Справочник по всем системным данным: порты, endpoints, env-переменные, идентификаторы, типы, глоссарий, matrix-таблицы.  
**Использование:** Когда нужно быстро найти конкретный факт — не читая зону целиком.

## Содержание

1. Сводная таблица портов  
2. Сводная таблица endpoints  
3. Env-переменные  
4. Внутренние идентификаторы  
5. Типы и модели  
6. Публичное / непубличное  
7. Глоссарий  
8. Debugging matrix  
9. Health check recipes  
10. Минимальные smoke-tests по зонам  
11. Правила работы  

# 1. Сводная таблица портов

## 1.1 TCP-порты

| Порт  | Сервис                        | Тип                        | Default   | Триггер включения                          |
|-------|-------------------------------|----------------------------|-----------|--------------------------------------------|
| 8080  | Python Tool (Jupyter)         | Python / FastAPI / Uvicorn | ✅ ACTIVE | —                                          |
| 8080  | nginx (альтернатива)          | nginx                      | ⭕ OFF    | CUA_DD_INIT_NGINX=true                     |
| 1384  | Terminal Server               | Python / FastAPI + PTY     | ✅ ACTIVE | —                                          |
| 8085  | container_daemon               | Rust / Axum / Tokio        | ⭕ OFF    | CUA_DD_INIT_CONTAINER_DAEMON=true          |
| 50939 | hive                          | Rust / WebSocket           | ⭕ OFF    | CUA_DD_INIT_HIVE=true                      |
| 8888  | Jupyter notebook              | Python                     | ⭕ OFF    | CUA_DD_ENABLE_NOTEBOOK_SERVER=true         |
| 9222  | Chromium CDP                  | DevTools Protocol          | ⭕ OFF    | CUA_DD_ENABLE_CHROME=true                  |
| 4444  | mitmproxy                     | MITM                       | ⭕ OFF    | CUA_DD_MITMPROXY=true                      |
| 5901  | x11vnc                        | VNC                        | ⭕ OFF    | CUA_DD_ENABLE_VNC=true                     |
| 6902  | novnc                         | VNC-HTTP                   | ⭕ OFF    | CUA_DD_ENABLE_VNC=true                     |
| 8081  | neko                          | browser/UI                 | ⭕ OFF    | CUA_DD_INIT_NEKO=true                      |
| 8082  | neko proxy                    | —                          | ⭕ OFF    | —                                          |
| 9999  | offline_proxy                 | forward HTTP proxy         | ⭕ OFF    | START_PROXY=true PROXY_PORT=9999           |

## 1.2 Unix sockets

| Путь                                              | Сервис                        |
|---------------------------------------------------|-------------------------------|
| /tmp/artifact_tool_rpc_<PID>_<UUID>.sock          | artifact_tool_rpc_daemon      |
| /tmp/artifact_tool_rpc_<PID>_<UUID>.ready         | готовность daemon             |
| /tmp/.X11-unix/X0                                 | Xvfb X-сервер                 |

## 1.3 Ингресс снаружи

**Проброшены:** только `:8080` и `:1384`.

**Не проброшены:** `:8085`, `:50939`, `:8888`, `:9222`, `:9999`, `:4444` — слушаются внутри VM, но снаружи недоступны.

## 1.4 Динамические localhost-порты (не идентифицированы)

Из ранних логов, назначение не установлено:

```
40231    (возможно shell channel Jupyter)
43929
39473
44849
36603
41903
42553
50229    (возможно control channel Jupyter)
54325    (возможно heartbeat Jupyter)
56612    (возможно iopub Jupyter)
56620    (возможно stdin Jupyter)
59331    (возможно iopub Jupyter)
```

Все — 🟡, порты меняются при перезапуске kernel.

# 2. Сводная таблица endpoints

## 2.1 Python Tool — :8080

| Route                                      | Method | Request                                                                 | Response              | Статус |
|--------------------------------------------|--------|-------------------------------------------------------------------------|-----------------------|--------|
| /status                                    | GET    | —                                                                       | {kernel_status, version} | ✅   |
| /execute                                   | POST   | {"code": str}                                                           | 200 / 422             | ✅     |
| /pull_message                              | POST   | {"timeout": float}                                                      | PullMessageResponse   | ✅     |
| /interrupt                                 | POST   | —                                                                       | 200                   | ✅     |
| /reset_kernel                              | POST   | —                                                                       | 200 (~8.75 s)         | ✅     |
| /caas_jupyter_tool/callback                | POST   | {name, args, kwargs}                                                    | 200 / 400             | ✅     |
| /caas_jupyter_tool/log_exception           | POST   | {message, exception, orig_func_name, orig_func_args, orig_func_kwargs}  | 200                   | ✅     |
| /caas_jupyter_tool/log_matplotlib_img_fallback | POST | —                                                                       | —                     | 🟢     |

## 2.2 Terminal Server — :1384

| Route          | Method | Request                  | Response     |
|----------------|--------|--------------------------|--------------|
| /healthcheck   | GET    | —                        | 200          |
| /open          | POST   | {cmd, env, user, cwd}    | {pid}        |
| /read/{pid}    | POST   | размер буфера            | stdout/stderr|
| /write/{pid}   | POST   | данные                   | —            |
| /kill/{pid}    | POST   | —                        | —            |

## 2.3 container_daemon — :8085

### Browser (управление вкладками)

| Route                | Method | Request            | Статус |
|----------------------|--------|--------------------|--------|
| /browser/new_page    | POST   | {"url": "..."}     | ✅     |
| /browser/state       | POST   | {}                 | ✅     |
| /browser/page_list   | POST   | {}                 | ✅     |
| /browser/get_state   | —      | —                  | ❌ 404 |
| /browser/js          | —      | —                  | ❌ 404 |

### GUI actions

| Route                      | Method | Request              | Статус          |
|----------------------------|--------|----------------------|-----------------|
| /click                     | POST   | {x, y, button: u8}   | ✅              |
| /double_click              | POST   | аналогично           | ✅              |
| /scroll                    | POST   | {x, y, delta_*}      | 🟢              |
| /move_mouse                | POST   | {x, y}               | 🟢              |
| /type                      | POST   | {text}               | 🟢              |
| /paste                     | POST   | текст                | 🟢              |
| /drag                      | POST   | {...}                | 🟢              |
| /compound                  | POST   | —                    | 🟢              |
| /multi_key_press_and_hold  | POST   | —                    | 🟢              |
| /clipboard                 | GET    | —                    | 🟢 (500 runtime)|
| /screenshot                | GET    | —                    | ✅ image/png    |
| /user_context              | GET    | —                    | 🟢              |
| /resume                    | POST   | —                    | 🟢              |

### Filesystem

| Route         | Method   | Request                          | Статус |
|---------------|----------|----------------------------------|--------|
| /file         | GET      | ?path=/home/oai/...              | ✅     |
| /files        | POST     | {"files": {path: base64}}        | ✅     |
| /files/:path  | —        | —                                | ❌ 404 |
| /list         | GET/POST | —                                | 🟢     |

### Exec

| Route | Method | Request     |
|-------|--------|-------------|
| /exec | POST   | {cmd: [...]}| 🟡

### Network / Config

| Route          | Method | Request                        | Статус |
|----------------|--------|--------------------------------|--------|
| /get_ip        | GET    | —                              | 🟢     |
| /config        | GET    | —                              | ✅     |
| /offline_sites | GET    | —                              | ✅     |
| /offline_sites | POST   | {"host": "forward-address"}    | ✅     |

### Diagnostics

| Route | Method   |
|-------|----------|
| /test | GET/POST | 🟢

### Root

| Route | Method | Response                                          |
|-------|--------|---------------------------------------------------|
| /     | GET    | {error, data: {agi_status, build, commit_sha, ip}}|

## 2.4 hive — :50939

| Route | Протокол          |
|-------|-------------------|
| /ws   | WebSocket upgrade |

## 2.5 Chromium CDP — :9222

| Route                    | Method | Назначение                |
|--------------------------|--------|---------------------------|
| /json/version            | GET    | версия браузера и CDP     |
| /json/list               | GET    | список target'ов          |
| /json/new                | GET    | новая вкладка             |
| /json/close/{id}         | GET    | закрыть вкладку           |
| /devtools/browser/{id}   | WebSocket | CDP control            |

## 2.6 artifact_tool RPC

Методы (JSON-RPC 2.0):

```
construct
call
callStatic
getAttr
setAttr
snapshot
dispose
```

# 3. Env-переменные

## 3.1 CUA_DD_* (~60 флагов)

### Режимы

| Флаг                            | Значение                  |
|---------------------------------|---------------------------|
| CUA_DD_TERMINAL_MODE            | только терминал           |
| CUA_DD_DESKTOP_MODE             | полный рабочий стол       |
| CUA_DD_TERMINAL_LOCAL_LOOPBACK  | loopback-only             |

### Инициализация сервисов

```
CUA_DD_INIT_TERMINAL_SERVER
CUA_DD_INIT_CONTAINER_DAEMON
CUA_DD_INIT_HIVE
CUA_DD_PYTHON_TOOL
CUA_DD_ENABLE_CHROME
CUA_DD_MITMPROXY
CUA_DD_ENABLE_VNC
CUA_DD_ENABLE_NOTEBOOK_SERVER
CUA_DD_INIT_XVFB
CUA_DD_INIT_X11VNC
CUA_DD_INIT_NEKO
CUA_DD_INIT_OPENBOX
CUA_DD_INIT_XFCE4
CUA_DD_INIT_PICOM
CUA_DD_INIT_DBUS
CUA_DD_INIT_NGINX
```

### Chrome

```
CUA_DD_CHROME_USER=oai
CUA_DD_CHROME_AUTO_RESTART
CUA_DD_CHROME_DEVTOOLS
CUA_DD_CHROME_KIOSK_PRINTING
CUA_DD_CHROME_CAAS_ARGS
```

### Порты

```
CUA_DD_CONTAINER_DAEMON_PORT=8085
CUA_DD_HIVE_PORT=50939
CUA_DD_NGINX_PORT=8080
CUA_DD_DUO_APP_SERVER_PORT
CUA_DD_PDF_READER_PORT
```

### Users

```
CUA_DD_TERMINAL_SERVER_USER=oai
CUA_DD_PYTHON_TOOL_USER=oai
CUA_DD_CHROME_USER=oai
```

### Behavior

```
CUA_DD_INIT_REMOVE_CONTAINER_SKILLS
CUA_DD_INIT_ARTIFACT_TOOL_V2
CUA_DD_INIT_ARTIFACT_TOOL_V2_RECORD_OPERATIONS
CUA_DD_INIT_ARTIFACT_TOOL_WARN_ABOUT_OVERLAPS_ON_EXPORT
CUA_DD_PYTHON_TOOL_WARM_SPREADSHEET_RUNTIME
CUA_DD_PYTHON_TOOL_DISABLE_MATPLOTLIB_SUPPORT
CUA_DD_BING_AT_HOME
CUA_DD_PDF_READER_SERVICE
CUA_DD_STARTUP_LIBREOFFICE
```

### Network

```
CUA_DD_MITM_NETWORK_CONFIG
CUA_DD_MITMPROXY
```

### Debug / monitoring

```
CUA_DD_SCREENSHOT_MIDDLEWARE
CUA_DD_SCREENSHOT_DRAW_RED_POSITION_DOT
CUA_DD_SCREENSHOT_WAIT_FOR_RESOURCES
CUA_DD_SCREENSHOT_DELAY
CUA_DD_CD_LOG_MIDDLEWARE_FILTER_RE
CUA_DD_CD_BROWSER_CONNECTION_MODE
CUA_DD_CD_NAVIGATE_TIMEOUT_MS
CUA_DD_CD_WAIT_FOR_RESOURCES_*
CUA_DD_CD_OPERATOR_STEALTH_MODE_TIMEOUT_MS
CUA_DD_CD_PASTE_PYAUTOGUI_TYPEWRITE
CUA_DD_NEXUS_HEALTH_CHECK
CUA_DD_COMPUTER_WAIT_MS
```

### Eval

```
CUA_DD_SAMPLE_ID_URL
CUA_DD_EXPERIMENT_NAME
EVAL_TASK_ID
```

## 3.2 Nebula

```
NEBULA_RUN=test-run
NEBULA_VM_ID=test-vm
NEBULA_USER=test-user
NEBULA_VM_LOGS_CAAS
```

**HTTP headers:**

```
X-Nebula-VM-Application
X-Nebula-VM-URL
X-Nebula-VM-Content-Type
X-Nebula-VM-Title
X-Nebula-Position-Dot
X-Nebula-Screenshot-Wait-For-Resources
X-Nebula-VM-Screenshot-Delay
```

## 3.3 MITM (Chaussette)

```
MITM_LOG_LEVEL
MITM_SERP_NO_CACHE
MITM_SERP_BING
MITM_WEBCACHE_HOST
MITM_WEBCACHE_CALLER_ID
MITM_WEBCACHE_CALLER_SECRET
MITM_WEBCACHE_IDENTITY
MITM_WEBCACHE_TIMEOUT_MS
MITM_WEBCACHE_EGRESS
MITM_WEBCACHE_V3_HOST
MITM_OFFLINE_GOOGLE_DOC_HOST
```

## 3.4 ACE

```
ACE_TOOLS_FEATURE_SET=chatgpt-applied
OAI_IS_JUPYTER_KERNEL=true
KERNEL_CALLBACK_ID   (не задан в текущей сессии)
```

## 3.5 Прочие системные

```
CUA_DD_VM_BUILD=openaiappliedcaasprod.azurecr.io/chrome-chatgpt-prod-5p4:...
OPENAI_CLUSTER=applied-caas34
NETWORK=caas_packages_only
TARGET=chatgpt-prod-5p4
VM_COMMIT_SHA=770cc6d08333b9844b1b63135928991b70e3343b

TERMINAL_SERVER_API_PORT=1384
JUPYTER_SERVER_API_PORT=8080

CDP_PORT=9222
NEKO_PORT=8081
NEKO_PROXY_PORT=8082
NOVNC_PROXY_PORT=6902

HTTP_PROXY   (supervisor env only, не в kernel env)
HTTPS_PROXY  (supervisor env only, не в kernel env)
ALL_PROXY    (supervisor env only, не в kernel env)

ARTIFACT_TOOL_RPC_SOCKET
ARTIFACT_TOOL_RPC_READY_FILE
```

# 4. Внутренние идентификаторы

## 4.1 Azure / OpenAI infrastructure

| Идентификатор                                      | Что                                      |
|----------------------------------------------------|------------------------------------------|
| openaiappliedcaasprod.azurecr.io                   | Приватный ACR OpenAI                     |
| chrome-chatgpt-prod-5p4                            | Production сборка ChatGPT (вариант 5p4)  |
| 20260803182207                                     | Дата сборки (3 августа 2026)             |
| 770cc6d08333...                                    | Коммит                                   |
| applied-caas24 / applied-caas34                    | Кластеры                                 |
| applied-caas-gateway1.internal                     | Gateway для пакетов                      |
| packages.applied-caas-gateway1.internal            | Миррор apt/npm                           |
| packages-{5,6,8,9,11,12,13,16,18}-{east,west,eu}.internal | Региональные мирроры              |
| buildkite.com/openai-mono/monorepo/builds/610973   | Внутренний CI                            |
| http://go/cua-caas-vm/                             | Внутренняя документация                  |
| http://go/vmlogs/                                  | Документация по логам                    |
| http://go/sb?profile=strawberry&experiment_id=     | Внутренний эксперимент                   |

## 4.2 Модули

```
@oai/granola              движок офисных документов
@oai/walnut               .NET → WASM OpenXML
qstar.common.tools.unified_tool   внутренний Python модуль
chaussette                MITM-прокси OpenAI Operator
```

## 4.3 Сертификаты

```
chaussette.crt     OpenAI Operator Proxy CA
nebula-dns.crt     OpenAI LLC, CN=openai.com
```

## 4.4 Chrome

```
Profile name: "ChatGPT Agent"
Extension ID: kcdongibgcplmaagnmgpjhpjgmmaaaaa
Отключены: Ctrl+Shift+W, Ctrl+S, Ctrl+N
```

## 4.5 Хосты

```
default.mitmproxy.hub.ace-research.openai.org
```

## 4.6 Файлы и пути

```
/opt/python-tool/openai/jupyter_server/app.py
/opt/terminal-server/openai/server.py
/usr/local/bin/container_daemon
/usr/local/bin/hive
/opt/pyvenv/lib/python3.13/site-packages/artifact_tool/bin/artifact_tool_rpc_daemon
/opt/pyvenv/lib/python3.13/site-packages/artifact_tool/bin/artifact_tool_rpc_daemon-bun
/etc/chromium/policies/managed/
/home/oai/.chromium/
/home/oai/skills/
/home/oai/redirect.html
/etc/skel/redirect.html
/etc/supervisord.conf
/usr/local/init_scripts/
/usr/local/scripts/oai_service.sh
```

# 5. Типы и модели

## 5.1 Python Tool — Pydantic

```python
class ExecuteRequest(BaseModel):
    code: str

class PullMessageRequest(BaseModel):
    timeout: float

class CallbackRequest(BaseModel):
    name: str
    args: list[Any]
    kwargs: dict[str, Any]

class LogExceptionRequest(BaseModel):
    message: str
    exception: str | object
    orig_func_name: str
    orig_func_args: str
    orig_func_kwargs: str

class RecordedCallback(BaseModel):
    name: str
    args: list[Any]
    kwargs: dict[str, Any]

class PullMessageResponse(BaseModel):
    message: IOPubMessage | None
    callbacks: list[RecordedCallback]
    kernel_status: str
```

## 5.2 container_daemon — Rust

```rust
FileUploadBody {
    files: HashMap<String, String>  // path → base64
}

add_offline_sites::handler(
    State<Arc<AppState>>,
    Json<HashMap<String, String>>
)
```

## 5.3 hive — Rust

```rust
enum HiveMessageInbound {
    Broadcast {
        kind: String,
        id: String,
        data: serde_json::Value,
    }
}

// Outbound добавляет from
// { kind, id, data, from }
```

## 5.4 ACE types

```
MethodCall
MethodCallException
MethodCallReturnValue
UploadFileRequest
UploadFileFromUrlRequest
DownloadFileToUrlRequest
CheckFileResponse
CreateKernelRequest
CreateKernelResponse
RegisterActivityRequest
EnsureUserMachineRequest
```

**EnsureUserMachineRequest:**

```python
message_type: str
timeout: ...
user_id: ...
max_time_alive: ...
allow_internet: bool | None
user_id_label: ...
```

Производное: `internet_access_level ∈ {"true", "false", None}`

## 5.5 Artifact Tool — JS

```javascript
enum HiveMessageInbound  // см. выше

// Workbook
Workbook.create()
Workbook.load(proto)
Workbook.fromProtoAndYdocBase64(proto_b64, crdt_b64)
workbook.apply([ops])
workbook.recalculate()
workbook.undo()
workbook.redo()
workbook.getCrdtDoc()           → AG (Y.Doc)
workbook.applyCrdtUpdateV2(bytes)
workbook.loadInitialCrdtStateV2(bytes)
workbook.hydrateCrdtFromProto()
workbook.toProto()
workbook.toHTML()
workbook.record(callback)
workbook.getRecorder()
```

**SpreadsheetRealtimeSession:**

```javascript
createEmpty()
getSummary()
getCrdtStateUpdateV2()
applyToolCrdtUpdateV2()
applyToolCrdtUpdateV2Base64()
fromProtoAndYdocBase64()

// static
SpreadsheetRealtimeSession.getWorkbookProtoBase64(workbook)
SpreadsheetRealtimeSession.getWorkbookCrdtStateUpdateV2(workbook)
```

## 5.6 Google bridges

```
GoogleSheetsAdapter     = Pi0
FetchGoogleSheetsClient = Vi0
GapiGoogleSheetsClient  = Ci0

GoogleSlidesAdapter     = Fp
FetchGoogleSlidesClient = Oi0
GapiGoogleSlidesClient  = Ti0
```

# 6. Публичное / непубличное

## 6.1 Публично

- `artifact_tool` — github.com/openai/artifact-tool, PyPI
- `@oai/walnut` — npm
- `apply_patch` — Codex CLI
- Skills — github.com/openai/skills
- `render_docx.py`, `render_slides.py`
- Chrome policies — публичный формат Chromium
- OOXML, Yjs, protobuf — открытые форматы
- hyper, axum, tokio, reqwest, chromiumoxide — публичные Rust crates
- Xvfb, Chromium — публичные пакеты

## 6.2 Непублично

- `CUA_DD_*` (~60 флагов)
- `NEBULA_*`
- `MITM_*`
- `qstar.*`
- `ace_*` протокол
- CA-сертификаты (`chaussette.crt`, `nebula-dns.crt`)
- `openaiappliedcaasprod.azurecr.io`
- `applied-caas34`
- `packages.applied-caas-gateway1.internal`
- `buildkite.com/openai-mono/...`
- `@oai/granola`
- `go/cua-caas-vm`, `go/vmlogs`
- Chrome profile "ChatGPT Agent"
- `container_daemon` (Rust-бинарник)
- `hive` (Rust-бинарник)
- `redirect.html` sentinel

## 6.3 Серая зона

| Что                        | Статус                                      |
|----------------------------|---------------------------------------------|
| artifact_tool 2.2.6        | версия может отличаться от публичной        |
| pptxgenjs_helpers 1.2.0    | надстройка OpenAI                           |
| SKILL.md                   | тексты могут отличаться                     |
| ace-tools 0.0.1            | версия, возможно, устаревшая                |

# 7. Глоссарий

| Термин                      | Значение                                              |
|-----------------------------|-------------------------------------------------------|
| ACE                         | Applied Computing Engine — внутренний RPC-протокол между моделью и VM |
| Nebula                      | платформа управления VM OpenAI                        |
| CaaS                        | Containers as a Service                               |
| CUA                         | Computer-Use Agent                                    |
| CUA DD                      | CUA Data Daemon — префикс флагов                      |
| Chaussette                  | внутреннее имя MITM-прокси OpenAI Operator            |
| Granola                     | внутренний движок офисных документов                  |
| Walnut                      | .NET → WASM OpenXML движок                            |
| Hive                        | локальная WebSocket broadcast-шина                    |
| Kata                        | Kata Containers — микро-VM                            |
| Offline Proxy               | forward HTTP proxy внутри container_daemon            |
| Skills                      | навыки ChatGPT (docx, pdfs, slides, spreadsheets)     |
| Terminal Server             | PTY-сервер через :1384                                |
| Python Tool                 | Jupyter-сервер через :8080                            |
| Artifact Tool               | RPC-движок для Workbook/Slides/Docs                   |
| CRDT                        | Conflict-free Replicated Data Type                    |
| Yjs                         | библиотека CRDT (Y.Doc)                               |
| Proto                       | protocol buffers snapshot                             |
| CDP                         | Chrome DevTools Protocol                              |
| URLBlocklist                | Chrome Managed Policy                                 |
| ERR_BLOCKED_BY_ADMINISTRATOR| результат блокировки Chrome                           |
| redirect.html               | sentinel-файл для двухступенчатой навигации           |

# 8. Debugging matrix

| Симптом                              | Что проверять                                      |
|--------------------------------------|----------------------------------------------------|
| :8080 closed                         | процесс python_tool                                |
| HTTP 500 на :8080                    | Python handler / kernel                            |
| HTTP 422 на :8080                    | request schema (Pydantic)                          |
| execute принят, но нет output        | IOPub, /pull_message                               |
| stdout есть, callbacks нет           | callback pipeline                                  |
| callback queue пустой                | callback generation / pull                         |
| старые callbacks исчезают            | maxlen=1000                                        |
| kernel недоступен                    | connection file / процесс kernel                   |
| старые порты не работают             | kernel перезапущен                                 |
| browser работает, Python нет         | Jupyter layer                                      |
| Python работает, browser нет         | browser layer                                      |
| :8085 closed                         | container_daemon не запущен                        |
| /browser/new_page → 500              | Chromium не отвечает через CDP                     |
| /screenshot падает                   | Xvfb не запущен                                    |
| CDP connection refused               | Chromium без --remote-debugging-port               |
| Chrome навигация → 200, но error page| URLBlocklist ["*"]                                 |
| /file → 500 Is a directory           | передан путь директории                            |
| /files → 400 blocked path            | путь вне /home/oai/**                              |
| /files/:path → 404                   | endpoint не зарегистрирован                        |
| offline_proxy → RST                  | egress закрыт                                      |
| getaddrinfo → gai rc=-3              | DNS закрыт                                         |
| /offline_sites → 400                 | невалидный Host                                    |
| Hive broadcast не приходит отправителю | ожидаемое поведение                             |
| Hive reject сообщения                | обязательны kind, id, data; лишние поля reject     |
| Artifact RPC → Unknown class         | класс не в hK0                                     |
| Artifact RPC → Method not found      | метод не существует у объекта                      |
| Artifact $ref не работает            | используйте {"$ref": "obj_N"}, не строку           |
| Workbook.apply → not iterable        | нужен массив                                       |
| applyCrdtUpdateV2 не работает        | нужен активный session                             |
| No active realtime dispatch          | используйте applyToolCrdtUpdateV2Base64            |
| hydrateCrdtFromProto на непустом     | требует пустой doc                                 |

# 9. Health check recipes

## 9.1 Полный smoke-test VM

```bash
#!/bin/bash

echo "=== 1. Процессы ==="
ps auxf | head -30

echo
echo "=== 2. TCP listeners ==="
ss -ltnp

echo
echo "=== 3. Jupyter status ==="
curl -s http://127.0.0.1:8080/status | head

echo
echo "=== 4. Terminal healthcheck ==="
curl -s http://127.0.0.1:1384/healthcheck

echo
echo "=== 5. Проверка egress (ожидаем RST) ==="
python3 -c "
import socket
for h, p in [('1.1.1.1', 443), ('8.8.8.8', 53)]:
    try:
        s = socket.create_connection((h, p), 2)
        print(f'OK {h}:{p}')
        s.close()
    except Exception as e:
        print(f'FAIL {h}:{p}: {type(e).__name__}')
"

echo
echo "=== 6. Chrome policies ==="
cat /etc/chromium/policies/managed/001_base_url_blocklist.json 2>/dev/null
```

## 9.2 Health check Jupyter (7 уровней)

```bash
# L1 — TCP :8080
ss -ltnp | grep ':8080'

# L2 — HTTP /status
curl -i http://127.0.0.1:8080/status

# L3 — kernel process
ps aux | grep '[i]python'

# L4 — connection file
ls -la /tmp/kernel-*.json
cat /tmp/kernel-*.json | python3 -c "import sys,json; print(json.load(sys.stdin))"

# L5 — heartbeat (нужен jupyter_client)
python3 -c "
from jupyter_client import BlockingKernelClient
import json
cfg = json.load(open('/tmp/kernel-XXX.json'))  # заменить на актуальный
kc = BlockingKernelClient()
kc.load_connection_file('/tmp/kernel-XXX.json')
kc.start_channels()
kc.wait_for_ready(timeout=5)
print('Kernel ready')
"

# L6 — execute test
curl -X POST http://127.0.0.1:8080/execute \
    -H 'Content-Type: application/json' \
    -d '{"code":"print(\"JUPYTER_HEALTH_OK\")"}'

# L7 — pull
curl -X POST http://127.0.0.1:8080/pull_message \
    -H 'Content-Type: application/json' \
    -d '{"timeout":5.0}'
```

## 9.3 Health check browser stack

```bash
# Xvfb
ps aux | grep '[X]vfb'
ls -la /tmp/.X11-unix/X0

# Chromium
ps aux | grep '[c]hromium'

# CDP
curl -s http://127.0.0.1:9222/json/version | head

# container_daemon
ss -ltnp | grep ':8085'

# browser subsystem
curl -i http://127.0.0.1:8085/browser/page_list

# screenshot
curl -s http://127.0.0.1:8085/screenshot -o /tmp/smoke.png
file /tmp/smoke.png
```

# 10. Минимальные smoke-tests по зонам

## 10.1 Jupyter

```bash
# 1. Выполнить код
curl -s -X POST http://127.0.0.1:8080/execute \
    -H 'Content-Type: application/json' \
    -d '{"code":"print(\"HELLO\")"}'

# 2. Прочитать результат
curl -s -X POST http://127.0.0.1:8080/pull_message \
    -H 'Content-Type: application/json' \
    -d '{"timeout":2.0}'
```

## 10.2 Terminal

```bash
PID=$(curl -s -X POST http://127.0.0.1:1384/open \
    -H "Content-Type: application/json" \
    -d '{"cmd":["echo","OK"],"env":{},"user":"","cwd":null}' \
    | jq -r .pid)

sleep 0.3
curl -s -X POST "http://127.0.0.1:1384/read/$PID" -d '1024'
curl -s -X POST "http://127.0.0.1:1384/kill/$PID"
```

## 10.3 container_daemon

```bash
# Root status
curl -s http://127.0.0.1:8085/

# Test
curl -s http://127.0.0.1:8085/test

# File write + read
B64=$(echo -n "hello" | base64)
curl -s -X POST http://127.0.0.1:8085/files \
    -H 'Content-Type: application/json' \
    -d "{\"files\":{\"/home/oai/share/test.txt\":\"$B64\"}}"

curl -s --get --data-urlencode "path=/home/oai/share/test.txt" \
    http://127.0.0.1:8085/file
```

## 10.4 hive

```bash
# Запустить
CUA_DD_INIT_HIVE=true CUA_DD_HIVE_PORT=50939 \
    nohup /usr/local/bin/hive > /tmp/hive.log 2>&1 &

sleep 2

# Проверить порт
ss -ltnp | grep ':50939'

# Отправить через websocat (нужен websocat)
echo '{"kind":"broadcast","id":"smoke","data":{"x":1}}' \
    | websocat -1 ws://127.0.0.1:50939/ws
```

## 10.5 Artifact Tool RPC

```python
import socket, json

s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.connect('/tmp/artifact_tool_rpc.sock')

def rpc(method, params):
    req = {"jsonrpc": "2.0", "id": 1, "method": method, "params": params}
    s.sendall((json.dumps(req) + "\n").encode())
    buf = b''
    while b'\n' not in buf:
        buf += s.recv(65536)
    return json.loads(buf.split(b'\n')[0])

# Создать Workbook
r = rpc("callStatic", {"class": "Workbook", "method": "create", "args": []})
print(r["result"]["objectId"])
```

## 10.6 Filesystem

```bash
# Создать файл в /mnt/data (для публикации в UI)
python3 -c "open('/mnt/data/smoke.txt','w').write('hello')"

# Проверить
cat /mnt/data/smoke.txt
```

# 11. Правила работы

## 11.1 Общие

1. Не путать `:8080`, `:1384`, `:8085`, `:50939`, `:9222` — это разные сервисы.  
2. Ingress снаружи только `:8080` и `:1384` — остальные внутренние.  
3. Egress закрыт — двухслойная блокировка, обход невозможен.  
4. `$ref` синтаксис обязателен в Artifact RPC.  
5. `Workbook.apply()` принимает массив, не объект.  
6. `snapshot` требует `maxDepth` для глубоких структур.  
7. `applyCrdtUpdateV2` принимает массив чисел, не base64.  
8. `hydrateCrdtFromProto` работает только на пустом doc.  
9. `Workbook.applyCrdtUpdateV2` требует active session.  
10. `GoogleSheetsAdapter.applyPatch()` — отдельный путь, не даёт CRDT update.

## 11.2 Работа с Artifact Tool

- Настоящая CRDT-мутация — только через Workbook/Worksheet/Range  
- Bootstrap обязателен: сначала `fromProtoAndYdocBase64`, потом incremental  
- Использовать `SpreadsheetRealtimeSession.applyToolCrdtUpdateV2Base64` для приёма incremental  
- Proto — base64, CRDT — base64, но `applyCrdtUpdateV2` — массив чисел  

## 11.3 Работа с Chrome

- Chrome policies нельзя обойти разумными способами  
- Даже при отключённых policies egress закрыт Azure NSG  
- `/browser/*` требует работающего Chromium + CDP + Xvfb  

## 11.4 Работа с egress

- Не пытаться обходить ограничения  
- Если нужен интернет — платформа решает это сама через `allow_internet=true`  
- Триггеры: Operator mode, Deep Research, SearchGPT, CUA  

## 11.5 Работа с файлами

- `/home/oai/**` — рабочая зона  
- `/mnt/data/` — публикация в UI  
- `/tmp`, `/etc`, `/root` — доступны через shell/Python, но не через container_daemon API  

## Итог

Собрано 11 файлов:

| Файл                        | Содержание                                      |
|-----------------------------|-------------------------------------------------|
| 00-overview.md              | Карта системы, TL;DR                            |
| 01-infrastructure.md        | Kata, Azure, Nebula, ресурсы, capabilities      |
| 02-container-daemon.md      | Управление VM через :8085                       |
| 03-jupyter-python-tool.md   | Python execution через :8080                    |
| 04-terminal-server.md       | Shell через :1384                               |
| 05-filesystem.md            | Три пути работы с файлами                       |
| 06-browser-stack.md         | Chrome, CDP, Xvfb, policies                     |
| 07-artifact-workbook.md     | RPC, CRDT, Google bridges                       |
| 08-network-boundary.md      | Egress, двухслойная блокировка                  |
| 09-hive.md                  | WebSocket broadcast                             |
| 99-reference.md             | Справочник                                      |
