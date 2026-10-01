# 09-hive.md

**Зона:** Hive WebSocket-шина  
**Бинарник:** `/usr/local/bin/hive` (отдельный ELF, не часть container_daemon)  
**Порт:** `0.0.0.0:50939`  
**Протокол:** WebSocket text/JSON, broadcast pub/sub  
**Статус:** ⭕ OFF по умолчанию (`CUA_DD_INIT_HIVE=true` для включения)

## Содержание

1. Что это такое  
2. Sources и внутренняя архитектура  
3. Запуск  
4. Endpoint  
5. Protocol IN — client → Hive  
6. Protocol OUT — Hive → clients  
7. Главное открытие: kind — просто строка  
8. Валидация сообщений  
9. Поведение broadcast  
10. UUID источника  
11. Внутренний цикл обработки  
12. Жизненный цикл клиента  
13. json_data_summary — лог-сокращение  
14. Binary frames  
15. Что Hive НЕ делает  
16. Полная модель протокола  
17. Связь с остальной системой  
18. Проверенный smoke-test  
19. Открытые вопросы  

# 1. Что это такое

Hive — это локальная WebSocket-шина внутри VM. Она предоставляет очень простой протокол: любой подключённый клиент может отправить JSON-сообщение, и это сообщение будет разослано всем другим клиентам, кроме отправителя.

Это не RPC, не очередь задач, не брокер сообщений в классическом смысле. Это именно pub/sub broadcast: publish → другим subscribers.

Hive — отдельный ELF-бинарник, а не подсистема container_daemon. Ранние логи иногда его путали с модулем daemon, но это самостоятельный процесс `/usr/local/bin/hive`.

Протокол очень маленький — вся спецификация умещается в пару страниц. Зато он полностью раскрыт.

# 2. Sources и внутренняя архитектура

В бинарнике зашиты пути исходников:

```
src/bin/hive.rs
src/hive/server.rs
src/hive/state.rs
src/hive/routes/ws.rs
```

Структура:

```
Hive Server
    │
    ▼
HiveState
    └── clients: HashMap<UUID, ClientHandle>

ClientHandle
    └── channel → send_task → WebSocket
```

Основные методы, подтверждённые статически:

- `HiveState::connect`
- `HiveState::disconnect`
- `HiveState::broadcast`
- `ClientHandle::send`
- `get_from`

Плюс:

- Serialize for `HiveMessageOutbound`
- Deserialize for `HiveMessageInbound`

То есть сериализация/десериализация реализована как отдельные impl'ы для двух типов. Внутри — Rust enum с одним вариантом Broadcast.

# 3. Запуск

## 3.1 Команда

```bash
CUA_DD_HIVE_PORT=50939 /usr/local/bin/hive
```

## 3.2 Env-переменные

| Переменная         | Значение              |
|--------------------|-----------------------|
| CUA_DD_HIVE_PORT   | 50939                 |
| CUA_DD_INIT_HIVE   | по умолчанию false    |

## 3.3 Ручной запуск

```bash
CUA_DD_INIT_HIVE=true CUA_DD_HIVE_PORT=50939 \
    nohup /usr/local/bin/hive > /tmp/hive.log 2>&1 &
```

Так же, как container_daemon, hive при старте проверяет флаг `CUA_DD_INIT_HIVE` в init-скрипте. Чтобы обойти — запускать бинарник напрямую.

## 3.4 Bind

Слушает `0.0.0.0:50939`. То есть внутри VM доступен на всех интерфейсах, но снаружи не проброшен.

## 3.5 Проверка

```bash
ss -ltnp | grep ':50939'
ps aux | grep '[h]ive'
```

# 4. Endpoint

Единственный WebSocket-endpoint:

```
ws://127.0.0.1:50939/ws
```

Обычный HTTP-запрос на `/` даёт:

```
GET / → 404
```

Только WebSocket-апгрейд на `/ws` работает:

```
WebSocket /ws → CONNECTED
```

## 4.1 Пример подключения

```bash
websocat ws://127.0.0.1:50939/ws
```

Или через Python:

```python
import asyncio
import websockets

async def main():
    async with websockets.connect('ws://127.0.0.1:50939/ws') as ws:
        await ws.send('{"kind":"broadcast","id":"x","data":{}}')
        response = await ws.recv()
        print(response)

asyncio.run(main())
```

# 5. Protocol IN — client → Hive

## 5.1 Схема

```json
{
  "kind": "<string>",
  "id": "<string>",
  "data": <any JSON value>
}
```

## 5.2 Обязательные поля

| Поле | Тип          | Обязательное |
|------|--------------|--------------|
| kind | string       | ✅           |
| id   | string       | ✅           |
| data | любой JSON   | ✅           |

## 5.3 Что значит каждое поле

- **kind** — произвольная строка, application-defined. Не enum-дискриминатор (см. раздел 7).
- **id** — идентификатор сообщения для корреляции. Тоже произвольная строка.
- **data** — любое значение JSON: объект, массив, число, строка, null.

## 5.4 Примеры валидных

```json
{"kind": "broadcast", "id": "x", "data": {}}
{"kind": "broadcast", "id": "x", "data": "HELLO"}
{"kind": "broadcast", "id": "x", "data": {"hello": "world"}}
{"kind": "broadcast", "id": "x", "data": [1, 2, "z"]}
{"kind": "broadcast", "id": "x", "data": null}
```

Все принимаются.

# 6. Protocol OUT — Hive → clients

## 6.1 Схема

```json
{
  "kind": "<string>",
  "id": "<string>",
  "data": <same JSON value>,
  "from": "<UUID>"
}
```

## 6.2 Отличия от IN

Добавляется поле `from` — UUID отправителя, сгенерированный сервером.

## 6.3 Пример

Отправлено клиентом A:

```json
{
  "kind": "broadcast",
  "id": "two-client",
  "data": {"hello": "world"}
}
```

Получено клиентом B:

```json
{
  "data": {"hello": "world"},
  "from": "e4eec876-f027-4bc7-a723-21732c21246a",
  "id": "two-client",
  "kind": "broadcast"
}
```

Всё то же самое + `from`.

## 6.4 Обратите внимание

Поле `type` со значением `HiveMessageOutbound::Broadcast` не входит в итоговый JSON. Оно есть только во внутреннем debug-логе Rust. По сети идёт чистый JSON без Rust-специфичных полей.

# 7. Главное открытие: kind — просто строка

Это самое важное и самое неочевидное в Hive.

## 7.1 Как выглядело сначала

Изначально можно было предположить, что `kind` — это переключатель:

```
kind = "broadcast" → enum Broadcast
kind = "send" → enum Send
kind = "connect" → enum Connect
```

## 7.2 Что оказалось на самом деле

Отправляем разные значения `kind`:

```json
{"kind": "Broadcast", "id": "k", "data": {}}
{"kind": "send", "id": "k", "data": {}}
{"kind": "connect", "id": "k", "data": {}}
{"kind": "ping", "id": "k", "data": {}}
```

Все принимаются. Получатель видит то же `kind` в исходящем сообщении:

```
{"data": {}, "from": "...", "id": "k", "kind": "Broadcast"}
{"data": {}, "from": "...", "id": "k", "kind": "send"}
{"data": {}, "from": "...", "id": "k", "kind": "connect"}
{"data": {}, "from": "...", "id": "k", "kind": "ping"}
```

Сервер не делает `if kind == "broadcast"` для выбора операции. Он просто передаёт строку как есть.

## 7.3 Почему в логах Rust всё равно HiveMessageInbound::Broadcast

В Rust-логах при получении любого сообщения пишется:

```
HiveMessageInbound::Broadcast
```

Это имя варианта Rust enum, а не значение поля JSON.

Схема ближе к:

```rust
enum HiveMessageInbound {
    Broadcast {
        kind: String,
        id: String,
        data: serde_json::Value,
    }
}
```

Или эквивалентной конструкции с единственным вариантом. Поэтому даже при `"kind": "ping"` в логе будет `HiveMessageInbound::Broadcast`.

## 7.4 Что это значит

Не искать magic-команды в `kind`. Их там нет. Строка произвольная. Поле `kind` — application-defined, для семантики на уровне клиента.

Все операции Hive сводятся к одной: broadcast. Имя `kind` при этом сохраняется как есть, чтобы клиенты могли отличать свои сообщения.

# 8. Валидация сообщений

## 8.1 Проверенные случаи

| Сообщение                                      | Результат   |
|------------------------------------------------|-------------|
| `{"kind":"broadcast","id":"x","data":{}}`      | ✅ accept   |
| `{"kind":"broadcast","id":"x","data":"HELLO"}` | ✅ accept   |
| `{"kind":"broadcast","id":"x","data":null}`    | ✅ accept   |
| `{"kind":"broadcast","id":"x","data":[1,2,"z"]}` | ✅ accept |
| `{"kind":"broadcast","id":"x"}`                | ❌ reject   |
| `{"kind":"broadcast","data":{}}`               | ❌ reject   |
| `{"kind":"broadcast","id":1,"data":{}}`        | ❌ reject   |
| `{"kind":"broadcast","id":"x","data":{},"extra":123}` | ❌ reject |
| `{}`                                           | ❌ reject   |

## 8.2 Сообщение об ошибке

При отсутствии обязательных полей:

```
data did not match any variant of untagged enum HiveMessageInbound
```

Это стандартное сообщение Serde при неудачной десериализации untagged enum.

## 8.3 Строгая десериализация

Hive не извлекает нужные поля из произвольного JSON. Он разбирает сообщение строго:

- все обязательные поля должны присутствовать
- типы должны совпадать (`id` должен быть string, не число)
- лишние поля приводят к ошибке

Это поведение Serde с `deny_unknown_fields` или эквивалентной конфигурацией.

## 8.4 Что происходит при ошибке

```
client_rx.next error Text deserialize
```

Ошибка разбора происходит внутри receive loop клиента. Что дальше — не закрывается ли соединение, не игнорируется ли сообщение — точное поведение не зафиксировано. Скорее всего, ошибка логируется, но соединение сохраняется.

# 9. Поведение broadcast

## 9.1 Основное правило

Broadcast не возвращается отправителю. Сообщение рассылается другим клиентам, но не самому отправившему.

## 9.2 Экспериментальное подтверждение

Два WebSocket-клиента, A и B:

```
Client A ─── broadcast ───► Hive
                              │
                              └───► Client B
```

Клиент B получает сообщение. Клиент A ждёт 2 секунды и получает:

```
TimeoutError
```

То есть своего сообщения A не получает.

## 9.3 При одном клиенте

Если к Hive подключён только один клиент, его сообщение уходит в никуда. Никакого ответа нет.

## 9.4 Что это значит

Hive — это не request/response. Это именно pub/sub:

- publish → другим subscribers
- никакого echo отправителю
- никакой очереди на потом

## 9.5 Модель использования

Правильная:

```
client A: publish
client B: receive
```

Если нужно что-то типа request/response, это надо строить поверх Hive на уровне клиентов: отправлять запрос с уникальным `id`, а другой клиент отвечает своим сообщением с тем же `id` в поле (или в data).

# 10. UUID источника

## 10.1 Как генерируется

При установке WebSocket-соединения сервер генерирует UUID для клиента.

В логах:

```
connect start
client_id="e4eec876-f027-4bc7-a723-21732c21246a"
total_clients="0"

connected
total_clients="1"
```

UUID присваивается один раз и живёт, пока клиент подключён.

## 10.2 Где используется

При broadcast от этого клиента в поле `from` появляется именно его UUID:

```json
{
  "data": {"hello": "world"},
  "from": "e4eec876-f027-4bc7-a723-21732c21246a",
  "id": "two-client",
  "kind": "broadcast"
}
```

## 10.3 Что это значит

Клиент не передаёт `from` сам. Это поле генерирует сервер. Клиент не может подделать отправителя.

Это важно для trust model: если другой клиент видит сообщение от UUID X, значит оно действительно пришло от того, кому сервер присвоил UUID X.

# 11. Внутренний цикл обработки

По логам и найденным символам:

```
WebSocket
    │
    ▼
receive task
    │
    ▼
JSON deserialize
    │
    ▼
HiveMessageInbound
    │
    ▼
HiveState::broadcast
    │
    ├── создать HiveMessageOutbound
    │
    ├── добавить from = sender UUID
    │
    └── пройти по clients
            │
            ├── sender → пропустить
            │
            └── other client
                    │
                    ▼
              ClientHandle::send
                    │
                    ▼
              WebSocket
```

В исходниках прямо присутствуют сообщения:

```
client_rx.next message
ClientHandle.send
broadcast
send_task end
receive_task end
```

То есть у каждого клиента две задачи:

- **receive task** — читает из WebSocket, десериализует, вызывает broadcast
- **send task** — пишет в WebSocket через канал из ClientHandle

Связь между ними — через `ClientHandle::send`.

# 12. Жизненный цикл клиента

## 12.1 Connect

При подключении WebSocket:

```
connect start
→ генерируется client_id (UUID)
→ в HashMap добавляется ClientHandle
→ total_clients увеличивается
→ connected
```

## 12.2 Работа

Пока клиент подключён:

- его UUID живёт в `HashMap<Uuid, ClientHandle>`
- receive task обрабатывает входящие сообщения
- send task пишет сообщения из других клиентов

## 12.3 Disconnect

При закрытии WebSocket:

```
client_rx.next close
    ↓
disconnect start
    ↓
disconnected
```

`HiveState::disconnect` удаляет клиента из HashMap. `total_clients` уменьшается.

В наблюдениях:

```
total_clients: 2
    ↓
total_clients: 1
    ↓
total_clients: 0
```

# 13. json_data_summary — лог-сокращение

Hive имеет функцию `json_data_summary`, которая сокращает payload в debug-логах.

## 13.1 Что происходит

Поле `data` в debug-логе превращается в summary:

| Оригинал          | В логе                              |
|-------------------|-------------------------------------|
| `{"x": 1}`        | `{"keys": ["x"], "type": "object"}` |
| `[1, 2, "z"]`     | `{"type": "array"}`                 |
| `null`            | `{"type": "null"}`                  |
| `"HELLO"`         | `{"type": "string"}`                |

## 13.2 Что это НЕ значит

Это только для логов. По WebSocket payload передаётся нормально, без сокращений.

Если в логе видно `{"keys":["x"],"type":"object"}`, это не значит, что клиент получил такой объект. Клиент получил оригинальный `{"x":1}`.

Это просто механизм reduce log noise.

## 13.3 Практическое следствие

Не делать выводов о структуре payload по debug-логам. Смотреть на реальный WebSocket-трафик.

# 14. Binary frames

## 14.1 Что поддерживается

Только Text WebSocket frames с JSON внутри.

## 14.2 Что не поддерживается

Binary frames → reject:

```
client_rx.next error Binary message not supported
```

## 14.3 Прочие unhandled

```
client_rx.next error unhandled message not supported
```

Это относится к другим типам WebSocket messages (Ping/Pong обрабатываются автоматически на уровне библиотеки, а что-то ещё может попадать в эту ветку).

## 14.4 Итог

Протокол:

```
WebSocket Text → JSON
```

Никакого бинарного транспорта нет.

# 15. Что Hive НЕ делает

На текущем уровне доказательств Hive не является:

- RPC server
- Command interpreter
- Browser controller
- Shell gateway
- File API
- HTTP proxy
- Generic event bus с typed operations

Он реализует очень узкий механизм:

```
WebSocket client registry
    +
JSON message reception
    +
broadcast to other clients
    +
sender UUID injection
```

Всё. Никаких «скрытых возможностей», никакой магии. Ранние гипотезы о «мосте к платформе» или «скрытых командах» — не подтверждаются.

# 16. Полная модель протокола

## 16.1 IN

```
Client → Hive

{
  "kind": "<string>",
  "id": "<string>",
  "data": <any JSON value>
}
```

## 16.2 OUT

```
Hive → Other Clients

{
  "kind": "<string>",
  "id": "<string>",
  "data": <same JSON value>,
  "from": "<UUID>"
}
```

## 16.3 Семантика полей

| Поле | Смысл                                      |
|------|--------------------------------------------|
| kind | application-defined string                 |
| id   | application-defined correlation/message identifier |
| data | arbitrary JSON value                       |
| from | server-generated sender UUID               |

## 16.4 Схематично

```
                 ┌─────────────┐
 Client A ──────►│             │
                 │    HIVE     │──────► Client B
 Client C ──────►│             │──────► Client D
                 └─────────────┘
                       │
                       ▼
                  inject UUID
```

Одна операция. Broadcast + UUID injection.

# 17. Связь с остальной системой

## 17.1 Отличие от Jupyter

Jupyter (`:8080`):

```
request → execute → result
```

Hive (`:50939`):

```
client A ─┐
          ├──► broadcast ──► client B, client C, client D
client E ─┘
```

Это разные модели. Jupyter — command/response. Hive — publish/subscribe.

## 17.2 Возможные клиенты

Кто реально подключается к Hive — не установлено окончательно.

Гипотезы (не подтверждены XREF'ом):

- Chrome Extension `kcdongibgcplmaagnmgpjhpjgmmaaaaa` (ChatGPT Agent)
- Внутренние демоны VM
- Какие-то компоненты платформы через проброс

Точный клиент нужно искать отдельно. В логах фиксировалось успешное подключение и успешный broadcast между двумя тестовыми клиентами, но кто из реальных компонентов использует Hive — открытый вопрос.

## 17.3 Что это значит

Hive существует и работает. Протокол полностью раскрыт. Но роль в системе — не установлена.

Это нормальная ситуация для reverse engineering: сам механизм понятен, применение — нет.

# 18. Проверенный smoke-test

## 18.1 Запуск

```bash
CUA_DD_INIT_HIVE=true CUA_DD_HIVE_PORT=50939 \
    nohup /usr/local/bin/hive > /tmp/hive.log 2>&1 &
```

## 18.2 Проверка процесса и порта

```bash
ss -ltnp | grep ':50939'
ps aux | grep '[h]ive'
```

## 18.3 Проверка через websocat

**Терминал 1:**

```bash
websocat ws://127.0.0.1:50939/ws
```

**Терминал 2:**

```bash
websocat ws://127.0.0.1:50939/ws
```

В терминале 1 ввести:

```json
{"kind":"broadcast","id":"test1","data":{"hello":"world"}}
```

В терминале 2 должно появиться:

```json
{"data":{"hello":"world"},"from":"<UUID>","id":"test1","kind":"broadcast"}
```

В терминале 1 ничего не появится (sender исключается).

## 18.4 Через Python

```python
import asyncio
import websockets
import json

async def main():
    async with websockets.connect('ws://127.0.0.1:50939/ws') as ws:
        await ws.send(json.dumps({
            "kind": "broadcast",
            "id": "py-test",
            "data": {"msg": "hello"}
        }))
        try:
            response = await asyncio.wait_for(ws.recv(), timeout=2)
            print(response)
        except asyncio.TimeoutError:
            print("Timeout (this is expected for single client)")

asyncio.run(main())
```

При одном клиенте — ожидаем timeout (sender исключается, других нет).

# 19. Открытые вопросы

## 19.1 Кто реальные клиенты

Самый важный открытый вопрос: кто подключается к `:50939/ws` в реальной работе. Возможные кандидаты:

- Chrome Extension
- внутренние демоны VM
- компоненты платформы через проброс (маловероятно, порт снаружи не проброшен)

Для ответа нужно:

- посмотреть на `ss -tnp` при активном подключении
- логировать connect/disconnect события Hive в реальной работе
- найти XREF в бинарниках других сервисов

## 19.2 Что в data

Если найдутся реальные клиенты, следующий вопрос — что они кладут в `data`. Это раскроет семантику.

## 19.3 Как используется id

Аналогично: если клиенты используют `id` для корреляции — это покажет модель использования.

## 19.4 Куда Hive сам подключается

Возможно, hive не только слушает, но и сам куда-то подключается. Это не подтверждено, но в бинарнике могут быть соответствующие механизмы.

## 19.5 Роль в архитектуре

Hive — реальный механизм, но его роль в общей архитектуре пока не установлена окончательно. Возможно, это координация между Chrome-расширением и платформой, а возможно — что-то другое.

Финальный ответ на этот вопрос — задача будущего исследования.
