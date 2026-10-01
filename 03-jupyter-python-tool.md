# 03 · Jupyter / Python Tool (`:8080`)

**Зона:** Python execution / Jupyter kernel
**Порт:** `0.0.0.0:8080`
**Статус:** ✅ ACTIVE по умолчанию
**Модуль:** `/opt/python-tool/openai/jupyter_server/app.py`

## Содержание

1. Что это такое
2. Process model
3. Два пути: command и event
4. Полная карта endpoints
5. `/execute` — команда
6. `/pull_message` — события
7. Callback subsystem
8. Long-polling — как это доказано
9. IOPub — типы сообщений
10. Connection file
11. Jupyter channels
12. Pydantic-модели
13. Bearer token — статус
14. Machine → UI
15. Что не работает
16. Health check
17. Debugging matrix
18. Связи
19. Открытые вопросы

## 1. Что это такое

Python Tool — основной execution layer песочницы. Это HTTP-сервис на FastAPI/Uvicorn, который соединяет внешний consumer (платформу) с Jupyter kernel внутри VM.

Это не обычный публичный Jupyter Notebook API. Это специальная прослойка, написанная OpenAI, которая:

- принимает HTTP-запросы по фиксированному протоколу
- транслирует их в Jupyter-протокол через `AsyncKernelClient`
- собирает события обратно и отдаёт consumer'у через long-poll
- поддерживает отдельный callback-канал для UI/artifact событий

В текущей сборке сервер на Python. В других сборках (наблюдалось ранее) на `:8080` мог быть Rust-прокси — но спецификация OpenAPI одинакова. Это полиморфизм инфраструктуры: внешний контракт не меняется, реализация может меняться.

## 2. Process model

Внутри VM работают два связанных процесса:

```
python_tool (Jupyter API server)
    │
    └── ipykernel (Python execution backend)
```

В одном из наблюдений:

```
PID 225  — python_tool (Jupyter server)
PID 239  — ipykernel
PPID(239) = 225
```

То есть kernel — дочерний процесс server'а. Если server умирает, kernel тоже уходит.

Команда запуска server'а:

```bash
tini -- /opt/pyvenv-python-tool/bin/python -m uvicorn \
    --host 0.0.0.0 --port 8080 \
    ... \
    jupyter_server.app:app
```

Команда запуска kernel:

```bash
/opt/pyvenv/bin/python -m ipykernel_launcher -f /tmp/....json
```

Флаг `-f` указывает на connection file — JSON с параметрами подключения. Подробно в разделе 10.

## 3. Два пути: command и event

Ключевая архитектурная идея: команда и результат — это два разных канала.

```
        COMMAND PATH                    EVENT PATH

client                                  kernel
   │                                       │
   │ POST /execute                         │
   ▼                                       ▼
Jupyter server ──── kc.execute() ───► shell channel
                                           │
                                           ▼
                                        ipykernel
                                           │
                                           ▼
                                        execution
                                           │
                                           ▼
                                        IOPub events
                                           │
                                           ▼
                                    deque(maxlen=1000)
                                           │
                                           │ POST /pull_message
                                           ▼
                                        client
```

То есть:

| Путь | Что делает | Направление |
|---|---|---|
| `/execute` | отправляет код в kernel | client → kernel |
| `/pull_message` | забирает события и callbacks | kernel → client |

Строить клиента на одной только паре request → response нельзя. Основной поток результатов асинхронный: после `/execute` kernel генерирует множество IOPub-событий, которые нужно собирать отдельно.

## 4. Полная карта endpoints

### 4.1 Список

| Route | Method | Request | Response | Статус |
|---|---|---|---|---|
| `/status` | GET | — | `{kernel_status, version}` | ✅ runtime |
| `/execute` | POST | `{"code": str}` | 200 / 422 | ✅ runtime |
| `/pull_message` | POST | `{"timeout": float}` | `PullMessageResponse` | ✅ runtime |
| `/interrupt` | POST | — | 200 | ✅ runtime |
| `/reset_kernel` | POST | — | 200 (~8.75 сек) | ✅ runtime |
| `/caas_jupyter_tool/callback` | POST | `{name, args, kwargs}` | 200 / 400 | ✅ runtime |
| `/caas_jupyter_tool/log_exception` | POST | `{message, exception, orig_func_name, orig_func_args, orig_func_kwargs}` | 200 | ✅ runtime |
| `/caas_jupyter_tool/log_matplotlib_img_fallback` | POST | — | — | 🟢 static |

### 4.2 Проверенные коды ошибок

| Запрос | Ответ |
|---|---|
| `POST /execute` с `{}` | 422 |
| `POST /execute` с `{"code": 123}` | 422 |
| `POST /execute` с `{"bad": true}` | 422 |
| `POST /caas_jupyter_tool/callback` с `name="UNKNOWN"` | 400 |

Валидация на входе строгая, на базе Pydantic-моделей. Это не passthrough — это типизированный API.

## 5. `/execute` — команда

### 5.1 Назначение

Передать Python-код во внутренний Jupyter kernel.

### 5.2 Схема

```json
{
  "code": "print('hello')"
}
```

Поле `code` — строка, обязательное.

### 5.3 Внутренний вызов

Server транслирует HTTP-запрос в Jupyter-протокол:

```python
kc.execute(
    request.code,
    silent=False,
    store_history=True,
    allow_stdin=False
)
```

Где `kc` — это `AsyncKernelClient` из `jupyter_client`.

Параметры:

- `silent=False` — kernel генерирует полный IOPub поток
- `store_history=True` — исполнение попадает в историю kernel (важно для `_` и `__`)
- `allow_stdin=False` — kernel не может запрашивать stdin у клиента

### 5.4 Что происходит после вызова

Kernel получает команду через shell channel, начинает исполнение, и параллельно публикует события в IOPub:

```
status busy
execute_input
stream
display_data
status idle
```

HTTP-ответ от `/execute` приходит сразу (это подтверждение приёма), но не содержит stdout. Stdout идёт через `/pull_message`.

### 5.5 Пример

```bash
curl -s -X POST http://127.0.0.1:8080/execute \
    -H 'Content-Type: application/json' \
    -d '{"code":"print(\"HELLO FROM INTERNAL CONTROL\")"}'
```

### 5.6 Persistent state

Kernel живёт между вызовами. Переменные, импорты, объекты сохраняются:

```python
# Первый запрос
x = 123

# Второй запрос (отдельный HTTP)
print(x)   # → 123
```

Это фундаментально отличается от `python -c '...'`, где каждый запуск — чистый процесс.

Сбросить состояние можно только через `/reset_kernel`.

## 6. `/pull_message` — события

### 6.1 Назначение

Забрать накопленные события и callbacks. Это длинный polling: запрос висит, пока событие не появится или не истечёт timeout.

### 6.2 Схема

```json
{
  "timeout": 5.0
}
```

Поле `timeout` — float, секунды.

### 6.3 Внутренний вызов

Server вызывает:

```python
kc.get_iopub_msg(timeout=timeout)
```

То есть блокируется до первого IOPub-сообщения или до истечения timeout.

### 6.4 Ответ

Тип ответа — `PullMessageResponse`. В него входят:

- IOPub message (если пришла — с полем `message`)
- накопленные callbacks (список)
- kernel status

Если IOPub-событий нет:

```json
{
  "message": null,
  "callbacks": [],
  "kernel_status": "idle"
}
```

Если timeout истёк без событий:

```json
{
  "message": null,
  "callbacks": [],
  "kernel_status": "idle"
}
```

То есть timeout и «нет события» визуально похожи — их надо различать по времени ответа.

### 6.5 Pull limit

За один вызов возвращается не более 100 callbacks. Если в очереди их накопилось больше — остальные остаются до следующего вызова.

### 6.6 Пример

```bash
curl -s -X POST http://127.0.0.1:8080/pull_message \
    -H 'Content-Type: application/json' \
    -d '{"timeout":5.0}'
```

### 6.7 Мультиплексирование

Важно понять: `/pull_message` — мультиплексор. Через него идут:

1. IOPub-события kernel'а (stream, display_data, error, status)
2. Callbacks из очереди `recorded_callbacks`

То есть один endpoint обслуживает оба потока. Это позволяет consumer'у держать единый event loop:

```
while running:
    response = POST /pull_message {timeout: 5.0}
    process_events(response)
```

## 7. Callback subsystem

### 7.1 Назначение

Помимо стандартных IOPub-событий, есть отдельный механизм callback'ов. Он предназначен для platform-specific результатов, которые обычный stream не может описать: таблицы, графики, artifact-операции.

### 7.2 Разрешённые callbacks (allowlist)

```python
_ALLOWED_CALLBACKS = {
    "display_dataframe_to_user",
    "display_chart_to_user",
    "display_matplotlib_image_to_user",
    "record_artifact_tool_operations",
}
```

Любое другое имя → `400 Bad Request`.

Это важный факт: callback endpoint — не произвольный RPC. Только 4 известных типа.

### 7.3 Схема запроса

```json
{
  "name": "display_dataframe_to_user",
  "args": ["CONTROL_PROBE"],
  "kwargs": {"date": "2026-09-18"}
}
```

Внутренняя модель:

```python
class RecordedCallback:
    name: str
    args: list[Any]
    kwargs: dict[str, Any]
```

### 7.4 Очередь

Callback'и сохраняются в:

```python
deque(maxlen=1000)
```

Название: `recorded_callbacks`.

Операционные свойства:

- Максимум хранимых: 1000
- Максимум за один pull: 100
- Порядок: FIFO
- При переполнении: старые вытесняются

### 7.5 Защита от гонок

Доступ к очереди сериализуется через:

```python
async with app.state.callback_lock:
    ...
```

Это `asyncio.Lock`, который берётся на время чтения очереди в `/pull_message`. Он не блокирует ядро, только очередь.

### 7.6 Экспериментальные данные

Проверено runtime:

| Отправлено | Получено за первый pull | За второй |
|---|---|---|
| 105 | 100 | 5 |
| 1005 | 100 (старые выпали) | — |

То есть поведение соответствует `deque(maxlen=1000)` + `_CALLBACK_PULL_LIMIT = 100`.

### 7.7 Пример отправки

```bash
curl -X POST http://127.0.0.1:8080/caas_jupyter_tool/callback \
    -H 'Content-Type: application/json' \
    -d '{"name":"display_dataframe_to_user","args":["probe"],"kwargs":{}}'
```

## 8. Long-polling — как это доказано

### 8.1 Эксперимент

Был запущен запрос:

```
POST /pull_message {"timeout": 5.0}
```

Пока он висел, отдельно было выполнено:

```python
print("WAKE_SEVEN_TEST")
```

Примерно через 1.2 секунды `/pull_message` завершился и вернул IOPub message.

### 8.2 Что это доказывает

- Server не отвечает немедленно
- Он блокируется на `kc.get_iopub_msg(timeout=timeout)`
- Реально ждёт появления события
- Как только событие приходит — отвечает

### 8.3 Модель consumer'а

Правильная модель использования:

```
loop:
    response = POST /pull_message {timeout: 5.0}
    if response.message:
        process(response.message)
    for cb in response.callbacks:
        process(cb)
```

Никаких `sleep(10ms)` — время ожидания отдаётся серверу.

## 9. IOPub — типы сообщений

IOPub — асинхронный канал kernel'а. Через него проходит всё, что происходит во время execution.

В наших экспериментах наблюдались типы:

| Тип | Что означает |
|---|---|
| `status` | busy / idle — kernel занят или свободен |
| `execute_input` | kernel начал выполнять команду |
| `stream` | текстовый вывод (stdout/stderr) |
| `display_data` | rich output (HTML, PNG, JSON) |
| `error` | исключение Python |

Порядок для `print("hello")`:

```
status busy
execute_input
stream ("hello\n")
status idle
```

Порядок для `display(df)`:

```
status busy
execute_input
display_data (text/html)
status idle
```

### 9.1 Статус idle — почему его недостаточно

Если несколько клиентов пишут в один kernel, `status idle` не гарантирует, что именно твой execution завершён. Нужно коррелировать по:

- `msg_id` — идентификатор сообщения
- `parent_header.msg_id` — ссылка на родительский запрос

Каждое IOPub-событие содержит `parent_header`, ссылающийся на запрос, который его породил.

### 9.2 IOPub — не гарантированное хранилище

IOPub — это поток. Если consumer не читает события вовремя, они не накапливаются в отдельном буфере indefinitely. Отдельно есть очередь `recorded_callbacks` (maxlen=1000), но это другое хранилище — только для callbacks, не для IOPub.

## 10. Connection file

### 10.1 Что это

При старте kernel Jupyter генерирует JSON-файл с параметрами подключения. Kernel получает его через флаг `-f /tmp/....json`.

Типовая структура:

```json
{
  "shell_port": 40231,
  "iopub_port": 56612,
  "stdin_port": 56620,
  "control_port": 50229,
  "hb_port": 54325,
  "ip": "127.0.0.1",
  "key": "...",
  "transport": "tcp",
  "signature_scheme": "hmac-sha256",
  "kernel_name": "python3"
}
```

### 10.2 Фактические значения

Порты динамические. В разных наблюдениях встречались:

```
40231, 50229, 54325, 56612, 56620
```

Соответствие конкретного порта конкретному каналу нужно брать из актуального connection file. Хардкодить нельзя.

### 10.3 Правило

```
connection file = authoritative source
```

Алгоритм:

1. Найти актуальный connection file
2. Прочитать JSON
3. Извлечь ports
4. Подключиться

Если kernel перезапущен — старые порты не работают.

### 10.4 Где искать

Стандартный путь — Jupyter runtime directory:

```bash
jupyter --runtime-dir
```

Файлы вида `kernel-*.json`.

Более широкий поиск:

```bash
find /tmp /run /home/oai -name 'kernel-*.json' 2>/dev/null
```

### 10.5 Поле `key` — не логировать

`key` используется для HMAC-подписи Jupyter-сообщений. Его не следует выводить без необходимости. Правильная работа — читать программно и передавать сразу в клиент, не показывая в логах.

### 10.6 Почему нельзя telnet

Jupyter-каналы — это не line-oriented TCP. Это ZMQ multipart с подписью HMAC. Просто подключиться к порту и отправить JSON нельзя — kernel отвергнет сообщение как неподписанное.

## 11. Jupyter channels

Стандартный Jupyter kernel использует 5 отдельных каналов (ZMQ endpoints):

| Канал | Назначение |
|---|---|
| `shell` | request/reply: команды, `execute_request` |
| `iopub` | publish/subscribe: события выполнения |
| `stdin` | запрос ввода у клиента |
| `control` | управляющие команды (interrupt, shutdown) |
| `heartbeat` | проверка живости соединения |

Это не пять HTTP endpoints. Это пять ZMQ-сокетов kernel'а.

### 11.1 Как соотносятся с HTTP API

- `POST /execute` → внутри используется `AsyncKernelClient.execute()` → shell channel
- `POST /pull_message` → внутри `kc.get_iopub_msg()` → iopub channel
- `POST /interrupt` → control channel
- `POST /reset_kernel` → restart через kernel manager

То есть HTTP-обёртка скрывает ZMQ-протокол от клиента.

### 11.2 Multipart envelope

Jupyter-сообщение — это ZMQ multipart:

```
[identities...]
<IDS|MSG>
<signature>
<header>
<parent_header>
<metadata>
<content>
```

`<header>` содержит:

```json
{
  "msg_id": "...",
  "username": "...",
  "session": "...",
  "msg_type": "execute_request",
  "version": "5.x"
}
```

`msg_type` определяет тип сообщения.

## 12. Pydantic-модели

Обнаруженные модели в исходниках.

### 12.1 `ExecuteRequest`

```python
class ExecuteRequest(BaseModel):
    code: str
```

### 12.2 `PullMessageRequest`

```python
class PullMessageRequest(BaseModel):
    timeout: float
```

### 12.3 `CallbackRequest`

```python
class CallbackRequest(BaseModel):
    name: str
    args: list[Any]
    kwargs: dict[str, Any]
```

### 12.4 `LogExceptionRequest`

```python
class LogExceptionRequest(BaseModel):
    message: str
    exception: str | object
    orig_func_name: str
    orig_func_args: str
    orig_func_kwargs: str
```

### 12.5 `RecordedCallback`

```python
class RecordedCallback(BaseModel):
    name: str
    args: list[Any]
    kwargs: dict[str, Any]
```

### 12.6 `PullMessageResponse`

Структура ответа `/pull_message`. Точные поля не зафиксированы в логах, но концептуально:

```python
class PullMessageResponse(BaseModel):
    message: IOPubMessage | None
    callbacks: list[RecordedCallback]
    kernel_status: str  # "idle" | "busy" | ...
```

## 13. Bearer token — статус

В исходниках server'а есть проверка `expected_bearer_token`. Наличие переменной в коде не означает, что она реально установлена.

В одном из осмотров environment процесса python_tool соответствующая переменная не была обнаружена.

Окончательная проверка `app.state.bearer_token` в текущем execution context не проводилась.

Поэтому статус: **НЕ УСТАНОВЛЕНО** — не «токена нет», не «токен есть».

Для operational-практики это означает: запросы проходят без предъявления токена в наших наблюдениях, но нельзя утверждать, что это гарантировано для всех сборок.

## 14. Machine → UI

Одно из важнейших подтверждённых свойств: результат выполнения Python доходит до UI ChatGPT, включая rich output.

### 14.1 Подтверждено экспериментально

| Что отправлено из Python | Что появилось в UI |
|---|---|
| `<div style="border:3px solid red;...">UI_HTML_DIRECT_...</div>` | HTML-блок |
| `<button>UI_BUTTON_PROBE_...</button>` | Кнопка |
| `<input value="UI_INPUT_PROBE_...">` | Поле ввода |
| PNG через PIL | Изображение |

То есть UI обрабатывает вывод как HTML, а не как экранированную строку.

### 14.2 Цепочка

```
Linux Python runtime
    │
    ▼
ipykernel
    │
    ▼
IOPub
    │
    ▼
Jupyter/API layer
    │
    ▼
platform transport (host-side)
    │
    ▼
ChatGPT UI
```

Последнее звено (platform transport → UI) реализовано вне VM. Оно недоступно для наблюдения изнутри.

### 14.3 Файлы

PNG из runtime → отображается как attachment. Файлы, созданные в `/mnt/data/`, автоматически подхватываются платформой. Подробно → `05-filesystem.md`.

### 14.4 Обратное направление — не работает

Попытка из UI вызвать runtime через `fetch('http://127.0.0.1:8765/...')`:

```
TypeError: Failed to fetch
```

Повторено дважды, результат стабильный. На listener'е внутри VM — тишина.

Это не доказывает отсутствие любого обратного канала. Доказывает только, что конкретно этот способ не работает. Возможные причины: другой network namespace у UI, sandbox браузера, CORS, отсутствие маршрута, platform proxy, renderer.

Важное различие: `127.0.0.1` в контексте UI и `127.0.0.1` в контексте Linux runtime — разные среды.

## 15. Что не работает

| Что | Результат | Причина |
|---|---|---|
| Bearer token отсутствует | НЕ УСТАНОВЛЕНО | не проверено |
| UI → `127.0.0.1:8765` через JS | `Failed to fetch` | разные network contexts |
| Callback с неизвестным `name` | 400 | allowlist |
| Пустой `{}` на `/execute` | 422 | schema |
| `{"code": 123}` | 422 | неверный тип |
| Точные порты Jupyter из лога №1 (`40231`, `59331`) | 🟡 | нужно перепроверить в текущем connection file |

## 16. Health check

Полная диагностика execution layer состоит из 7 уровней:

| Уровень | Проверка |
|---|---|
| L1 | `ss -ltnp \| grep ':8080'` — TCP-порт слушается |
| L2 | `curl -i http://127.0.0.1:8080/status` — HTTP отвечает |
| L3 | `ps aux \| grep '[i]python'` — kernel процесс жив |
| L4 | connection file существует и читается |
| L5 | heartbeat по `hb_port` — kernel отвечает |
| L6 | `/execute` с `print("OK")` — kernel выполняет код |
| L7 | `/pull_message` возвращает stdout — события доходят |

Все 7 уровней должны проходить для полностью рабочего execution layer.

### 16.1 Различие HTTP server alive vs kernel alive

`:8080 reachable` ≠ `kernel healthy`.

HTTP-сервер может отвечать, а kernel — быть мёртвым, hung или disconnected. Поэтому `/status` нужно проверять отдельно от TCP-порта.

## 17. Debugging matrix

| Симптом | Что проверять |
|---|---|
| `:8080` closed | процесс python_tool |
| HTTP 500 | Python handler / kernel |
| HTTP 422 | request schema (Pydantic) |
| execute принят, но нет output | IOPub, `/pull_message` |
| stdout есть, callbacks нет | callback pipeline |
| callback queue пустой | callback generation / pull |
| старые callbacks исчезают | `maxlen=1000` |
| kernel недоступен | connection file / процесс kernel |
| старые порты не работают | kernel перезапущен |
| browser работает, Python нет | Jupyter layer |
| Python работает, browser нет | browser layer (container_daemon) |

## 18. Связи

- `02-container-daemon.md` — отдельный сервис на `:8085`, не путать с `:8080`
- `04-terminal-server.md` — shell на `:1384`, отдельный механизм
- `05-filesystem.md` — `/mnt/data/` → platform publish → UI
- `07-artifact-workbook.md` — callbacks `record_artifact_tool_operations` связывают Jupyter с Artifact Tool
- `08-network-boundary.md` — почему `127.0.0.1` из UI не работает
- `09-hive.md` — отдельный WebSocket, не имеет отношения к Jupyter

## 19. Открытые вопросы

### 19.1 Точный host-side consumer `/pull_message`

Известно:

- ingress снаружи VM только `:8080` и `:1384`
- machine → UI работает (HTML, PNG, файлы)
- последний шаг делает платформа, не sandbox

Неизвестно:

- точный транспорт от host-side consumer к `:8080/pull_message`
- как платформа отличает обычный вывод от attachment

Это единственный крупный незакрытый участок во всей документации по Jupyter.

### 19.2 Точный контракт `_log_matplotlib_img_fallback`

Endpoint есть в статике, полное поведение не восстановлено.

### 19.3 Точное соответствие Jupyter ports

В логах №1 упоминались `40231` и `59331` как возможные `shell_port` и `iopub_port`. В текущей VM их роль нужно проверять по актуальному connection file.
