# 07-artifact-workbook.md

**Зона:** Artifact Tool (Workbook / Spreadsheet / CRDT / Google Bridges)  
**Компонент:** `/opt/pyvenv/lib/python3.13/site-packages/artifact_tool/bin/artifact_tool_rpc_daemon` (Bun standalone, 126 MB)  
**Транспорт:** Unix socket / TCP, JSON-RPC 2.0 NDJSON  
**Статус:** ✅ runtime + 🟢 static (глубокий разбор)

## Содержание

### Часть A — RPC
1. Что это такое  
2. Транспорт  
3. Автозапуск daemon  
4. Семь базовых методов  
5. Two registries  
6. Формат ответа  
7. Пример полного цикла на Python  

### Часть B — Recorder pipeline
8. Автоматическое оборачивание  
9. Код record()  
10. Wrapper tG6  

### Часть C — Объектная модель
11. 194 класса  
12. Workbook API  
13. Worksheet / Range / Cell  

### Часть D — CRDT
14. Двухслойное состояние  
15. AG = Y.Doc  
16. SpreadsheetRealtimeSession  
17. Bootstrap и restore  
18. Incremental replication  
19. Что внутри Proto  
20. Что внутри CRDT snapshot  

### Часть E — Semantic operations
21. Полный список ops  
22. Правильные формы аргументов  
23. range.format.set — детально  

### Часть F — Google bridges
24. GoogleSheetsAdapter и клиенты  
25. Маппинг Artifact op → Google API  
26. Wire-format  
27. Google Slides bridge  
28. Плагинная мёртвая точка  

### Часть G — Практика
29. Минимальный рецепт  
30. Правила работы  
31. Открытые вопросы  

# Часть A — RPC

## 1. Что это такое

Artifact Tool — это внутренний движок OpenAI для работы с офисными документами: Workbook (аналог Excel/Sheets), Presentation (аналог Slides), Document (аналог Docs). Он представляет собой Bun standalone executable размером 126 MB, собранный в один файл.

Внутри — полный stateful-движок с объектной моделью, CRDT-синхронизацией и сериализацией. Наружу он предоставляет JSON-RPC API через Unix socket или TCP.

**Размещение:**

```
/opt/pyvenv/lib/python3.13/site-packages/artifact_tool/
├── bin/
│   ├── artifact_tool_rpc_daemon         shell launcher
│   ├── artifact_tool_rpc_daemon-bun     126 MB ELF (Bun standalone)
│   └── node_modules/@oai/walnut/        WASM .NET OpenXML
├── rpc/                                 .pyc: client, daemon, connection
├── generated/                           API интерфейс
└── patches/
```

Daemon запускается лениво — когда Python-код впервые импортирует `artifact_tool`.

## 2. Транспорт

### 2.1 Unix socket (основной)

```
/tmp/artifact_tool_rpc_<PID>_<UUID>.sock
```

- `<PID>` — PID процесса, создавшего daemon.  
- `<UUID>` — уникальный идентификатор сессии.

### 2.2 TCP fallback (для Windows)

```
tcp://127.0.0.1:0     ← ephemeral port
/tmp/artifact_tool_rpc_<PID>_<UUID>.ready   ← файл с фактическим endpoint
```

Если Unix socket недоступен, daemon использует TCP + `.ready`-файл с реальным адресом.

### 2.3 Формат сообщений

JSON-RPC 2.0, newline-delimited (NDJSON):

```
{"jsonrpc":"2.0","id":N,"method":"...","params":{...}}\n
```

Каждое сообщение — одна строка JSON, заканчивающаяся `\n`.

### 2.4 Env-переменные

- `ARTIFACT_TOOL_RPC_SOCKET` — путь к socket  
- `ARTIFACT_TOOL_RPC_READY_FILE` — путь к `.ready`-файлу

## 3. Автозапуск daemon

Когда Python-код вызывает `get_or_create_client()`:

```
1. Читает ARTIFACT_TOOL_RPC_SOCKET
2. Если нет — start_daemon():
     Popen(artifact_tool_rpc_daemon,
           env={SOCKET, READY_FILE, ...os.environ},
           stdout=None, stderr=None,
           cwd=<bin dir>,
           start_new_session=True)
     → ждёт .ready (шаг 50 мс)
3. Возвращает socket_path
```

`start_new_session=True` создаёт отдельную session group — daemon переживёт закрытие родителя.

## 4. Семь базовых методов

| Метод       | Параметры              | Что делает                          |
|-------------|------------------------|-------------------------------------|
| construct   | `{class, args}`        | `new hK0[class](...args)` → objectId |
| call        | `{target, method, args}` | `obj[method].apply(obj, args)`    |
| callStatic  | `{class, method, args}` | `hK0[class][method](...args)`     |
| getAttr     | `{target, attr}`       | `obj[attr]`                         |
| setAttr     | `{target, attr, value}` | `obj[attr] = value`                |
| snapshot    | `{target, maxDepth}`   | сериализация структуры              |
| dispose     | `{target}`             | удалить objectId из registry        |

### 4.1 Реализация construct

```javascript
case "construct": {
    const {class: Z, args: U=[]} = F;
    const Y = hK0[Z];

    if (!Y)
        throw new Error(`Unknown class ${Z}`);

    return b_(q, () => new Y(...K), ...)
}
```

### 4.2 Реализация call

```javascript
case "call": {
    const {target: Z, method: U, args: Y=[]} = F;
    const q = vL.get(Z);

    ...

    const K = q[U];

    if (typeof K !== "function")
        throw new Error(`Method ${U} not found`);

    return b_(X, () => K.apply(q, G), ...)
}
```

## 5. Two registries

Внутри daemon два реестра:

1. `hK0[className]` — реестр классов. 194 записи. Используется для `construct` и `callStatic`.
2. `vL.get(objectId)` — реестр живых объектов. Используется для `call`, `getAttr`, `setAttr`, `snapshot`, `dispose`.

### 5.1 Синтаксис object reference

В аргументах `$ref` — специальный синтаксис:

```json
{"$ref": "obj_194"}
```

Просто строка `"obj_194"` не работает. Обязательно через объект с ключом `$ref`.

### 5.2 Регистрация

```javascript
hK0 = {
    ...jL,
    SpreadsheetRealtimeSession: x_
}

wL4 = new Map;

for (const [J,Q] of Object.entries(hK0))
    wL4.set(Q,J)
```

То есть каждому классу соответствует свой объект-конструктор, а обратный map `wL4` позволяет по классу найти имя.

## 6. Формат ответа

### 6.1 Успех

```json
{
  "result": {"objectId": "obj_N", "class": "ClassName"},
  "operations": {
    "artifactHandle": {"objectId": "obj_1", "class": "Workbook"},
    "operations": [{"op": "sheet.add", "name": "Sheet1", "as": "@sheet1"}],
    "crdtUpdateV2": "AAAGy6mMqQgCAyhEAgAFJ..."
  }
}
```

Поле `operations` присутствует только у Artifact-мутаций.

### 6.2 Ошибка

```json
{
  "error": {
    "code": -32000,
    "message": "Server error",
    "data": {
      "name": "TypeError",
      "message": "undefined is not an object",
      "stack": "...",
      "code": "JS_ERROR"
    }
  }
}
```

`code: -32000` — стандартный код JSON-RPC для server error.

## 7. Пример полного цикла на Python

```python
import socket, json

s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.connect('/tmp/artifact_tool_rpc.sock')

def rpc(method, params, id=1):
    req = {"jsonrpc": "2.0", "id": id, "method": method, "params": params}
    s.sendall((json.dumps(req) + "\n").encode())
    buf = b''
    while b'\n' not in buf:
        buf += s.recv(65536)
    return json.loads(buf.split(b'\n')[0])

# Создать Workbook
r = rpc("callStatic", {"class": "Workbook", "method": "create", "args": []})
wb_id = r["result"]["objectId"]

# Получить коллекцию worksheets
r = rpc("getAttr", {"target": wb_id, "attr": "worksheets"})
coll_id = r["result"]["objectId"]

# Добавить лист
r = rpc("call", {"target": coll_id, "method": "add", "args": ["Sheet1"]})
sheet_id = r["result"]["objectId"]

# Получить Range
r = rpc("call", {"target": sheet_id, "method": "getRange", "args": ["A1:B2"]})
range_id = r["result"]["objectId"]

# Записать значения
r = rpc("setAttr", {
    "target": range_id,
    "attr": "values",
    "value": [["Hello", "World"], ["Foo", "Bar"]]
})
```

# Часть B — Recorder pipeline

## 8. Автоматическое оборачивание

Все Artifact-мутации автоматически записываются через `record()`. Это не требует ручного вызова — обёртка стоит на уровне RPC.

## 9. Код record()

Ключевая функция:

```javascript
record(J){
    if(this.#m)
        throw new Error("Workbook.record does not support nested recordings.");

    if(!this.#e)
        this.hydrateCrdtFromProto();

    const Q = new Al(this);
    const F = [];

    const Z = (U,Y) => {
        if(Y !== this.#u && Y !== this.#b) return;
        F.push(new Uint8Array(U));
    };

    this.#f.on("updateV2", Z);
    this.#m = Q;

    try {
        const U = this.#P0(this.#u, J);
        this.#x0();

        return {
            result: U,
            patch: Q.getPatch(),
            idMap: Q.getIdMap(),
            crdtUpdateV2: F.length > 0 ? $T(F) : undefined
        };
    }
    finally {
        this.#f.off("updateV2", Z);
        this.#k.stopCapturing();
        this.#m = undefined;
    }
}
```

### 9.1 Что это значит

Одна мутация Workbook генерирует:

- `patch` — semantic operations (что изменилось)
- `idMap` — соответствие внутренних id
- `crdtUpdateV2` — incremental CRDT update (если были Yjs изменения)

### 9.2 Исключения

Некоторые методы не оборачиваются:

- `loadInitialCrdtStateV2` — низкоуровневый CRDT
- `applyCrdtUpdateV2` — то же

Это низкоуровневые примитивы, которые не должны попадать в recorder.

## 10. Wrapper tG6

Artifact-вызовы оборачиваются в `tG6`:

```javascript
async function tG6(J,Q){
    if(!J || !aG6(J.artifact))
        return {
            result: await Q(),
            operations: []
        };

    const F = J.artifact.record(() => Q());

    return {
        result: await F.result,
        operations: F.patch,
        crdtUpdateV2: rG6(F.crdtUpdateV2)
    }
}
```

Если объект не Artifact — `operations: []`, без CRDT.

CRDT update сериализуется в Base64:

```javascript
u_.from(
    K.buffer,
    K.byteOffset,
    K.byteLength
).toString("base64")
```

# Часть C — Объектная модель

## 11. 194 класса

Классы распределены по категориям.

**Spreadsheet**

```
Workbook, WorkbookArtifact, WorkbookRecorder
Worksheet, WorksheetCollection, WorksheetCells
Range, Cell, Row
Table, TableCollection
Style, StylesCollection, Theme, Color, Fill
ConditionalFormat
```

**Charts**

```
Chart, ChartCollection, ChartSeries, ChartSeriesCollection
ChartAxis, ChartLegend, ChartDataLabels
```

**Presentation**

```
Presentation, Slide, SlideCollection, PresentationTheme
PresentationTable
Shape, ShapeCollection
Image, ImageCollection
Text, TextRange, TextStyle
Layout, LayoutCollection
```

**Documents**

```
DocumentModel, DocumentFile
PresentationFile, SpreadsheetFile
DocumentTheme
```

**Google**

```
GoogleSheetsAdapter, FetchGoogleSheetsClient, GapiGoogleSheetsClient
GoogleSlidesAdapter, FetchGoogleSlidesClient, GapiGoogleSlidesClient
```

**Realtime**

```
SpreadsheetRealtimeSession
```

## 12. Workbook API

### 12.1 Жизненный цикл

```
Workbook.create()                             — пустая книга
Workbook.load(proto)                          — из protobuf
Workbook.fromProtoAndYdocBase64(proto, crdt)  — полное состояние
```

### 12.2 Мутации

```
workbook.apply([ops])       — batch-мутации
workbook.recalculate()      — пересчёт формул
workbook.undo()
workbook.redo()
```

### 12.3 CRDT

```
workbook.getCrdtDoc()             → AG (Y.Doc)
workbook.applyCrdtUpdateV2(bytes)
workbook.loadInitialCrdtStateV2(bytes)
workbook.hydrateCrdtFromProto()
```

### 12.4 Сериализация

```
workbook.toProto()   → proto object
workbook.toHTML()
```

### 12.5 Recorder

```
workbook.record(callback)       → запись изменений
workbook.getRecorder()          → WorkbookRecorder
```

## 13. Worksheet / Range / Cell

**Worksheet**

```
name, id, index      — свойства
cells                — WorksheetCells
getRange("A1:B2")    — получить Range
usedRange            — используемый диапазон
```

**Range**

```
values               — [[...]] (чтение/запись)
formulas             — [[...]] (чтение/запись)
format               — RangeFormat
write([[matrix]])    — массовая запись
```

**Полный пример**

```python
# 1. Create
wb = construct("Workbook", [])

# 2. Sheet
sheets = getAttr(wb, "worksheets")
sheet = call(sheets, "add", ["Sheet1"])

# 3. Values
rng = call(sheet, "getRange", ["A1:B2"])
setAttr(rng, "values", [["Hello", "World"], ["Foo", "Bar"]])

# 4. Formula
rng2 = call(sheet, "getRange", ["C1"])
setAttr(rng2, "formulas", [["=A1&B1"]])

# 5. Format
setAttr(rng, "format", {"font": {"bold": True}, "fill": "#FF0000"})

# 6. Merge
call(wb, "apply", [[{"op": "range.merge",
                     "target": {"sheet": "Sheet1", "range": "D1:E2"}}]])

# 7. Recalculate
call(wb, "recalculate", [])

# 8. Render → PNG
png = call(wb, "render", [])
```

# Часть D — CRDT

## 14. Двухслойное состояние

Workbook имеет два связанных представления:

```
                    Workbook
                   /        \
                  /          \
         Proto snapshot      Y.Doc / CRDT
                  \          /
                   \        /
                    restore
                       ↓
                 Workbook B
```

- **Proto** — базовый protobuf snapshot  
- **CRDT** — collaborative state на базе Yjs  

Оба можно сериализовать и восстановить.

## 15. AG = Y.Doc

`Workbook.getCrdtDoc()` возвращает объект класса `AG`. Это Y.Doc-подобная структура.

**Поля**

```
clientID
guid
collectionid
gc
gcFilter
meta
share = Map
store
subdocs = Set
```

**Методы**

```
transact
get
getArray
getText
getMap
getXmlElement
getXmlFragment
toJSON
destroy
```

Это прямое подтверждение, что CRDT здесь — Yjs-совместимая модель, а не абстрактная «похожая».

## 16. SpreadsheetRealtimeSession

**Методы**

```
createEmpty()
getSummary()
getCrdtStateUpdateV2()
applyToolCrdtUpdateV2()
applyToolCrdtUpdateV2Base64()
fromProtoAndYdocBase64()
```

**Не являются методами**

```
getWorkbook()          — нет
getCrdtDoc()           — нет
getWorkbookState()     — нет
```

**Property**

```
session.workbook       — доступ к Workbook
```

## 17. Bootstrap и restore

### 17.1 Static-методы

```javascript
static getWorkbookProtoBase64(J){
    const Q = J.toProto();
    return Buffer.from(
        zK.encode(Q).finish()
    ).toString("base64");
}

static getWorkbookCrdtStateUpdateV2(J){
    return gR(J.getCrdtDoc());
}

static fromProtoAndYdocBase64(J,Q){
    const F = zK.decode(Buffer.from(J,"base64"));
    const Z = kQ.load(F);

    const U = new Uint8Array(
        Buffer.from(Q,"base64")
    );

    Z.loadInitialCrdtStateV2(
        U,
        {recalculate:true}
    );

    Z.__flushPendingCollaborativePublishes();

    return new x_(Z);
}
```

### 17.2 Полный цикл

```python
# A: создать + заполнить
wb_a = construct("Workbook", [])
sheets_a = getAttr(wb_a, "worksheets")
sheet_a = call(sheets_a, "add", ["Sheet1"])
rng_a = call(sheet_a, "getRange", ["A1:B2"])
setAttr(rng_a, "values", [["Hello", "World"], ["Foo", "Bar"]])

# Сериализация
proto_b64 = callStatic("SpreadsheetRealtimeSession",
                       "getWorkbookProtoBase64", [{"$ref": wb_a}])
crdt_b64  = callStatic("SpreadsheetRealtimeSession",
                       "getWorkbookCrdtStateUpdateV2", [{"$ref": wb_a}])

# B: восстановление
ssrs_b = callStatic("SpreadsheetRealtimeSession",
                    "fromProtoAndYdocBase64", [proto_b64, crdt_b64])
wb_b = getAttr(ssrs_b, "workbook")
```

Workbook B — отдельный объект, не ссылка на A.

## 18. Incremental replication

### 18.1 Разница двух режимов

- `getWorkbookCrdtStateUpdateV2()` — FULL CRDT state snapshot  
- mutation response `crdtUpdateV2` — INCREMENTAL update  

Это экспериментально подтверждено:

- применение только одного incremental update к чистому B даёт только это изменение  
- `getWorkbookCrdtStateUpdateV2()` содержит всё состояние целиком  

### 18.2 Применение incremental

```python
callStatic("SpreadsheetRealtimeSession", "applyToolCrdtUpdateV2Base64",
           [{"$ref": session_b}, incremental_b64])
```

**НЕ** через `Workbook.applyCrdtUpdateV2` — он даёт:

```
No active realtime dispatch for broadcast payload.
```

### 18.3 Матрица репликации A → B

| Тип изменения          | Работает |
|------------------------|----------|
| values                 | ✅       |
| formulas               | ✅       |
| calculated values      | ✅       |
| formatting             | ✅       |
| merge                  | ✅       |
| повторная доставка merge | ✅ идемпотентно |

### 18.4 Ограничение

Нельзя взять пустую Session и скормить ей первый incremental update. Нужен bootstrap: сначала `fromProtoAndYdocBase64` (Proto + full CRDT), потом incremental.

## 19. Что внутри Proto

При разборе `/tmp/workbook.proto` простым protobuf wire-format parser найдены:

```
Sheet1
Hello
World
Foo
Bar
A1
B2
formula data
```

Merge найден как отдельный embedded protobuf:

```
Sheet1
D1
E2
```

Формат `FF0000` (цвет fill) — тоже внутри Proto.

То есть Proto содержит и значения, и формулы, и рассчитанные значения, и formatting, и merge.

## 20. Что внутри CRDT snapshot

В бинарном CRDT snapshot найдены printable fragments:

```
Hello
World
Foo
Bar
font
fill
bold
FF0000
```

Ключевые фрагменты:

```
"fontProto":{"bold":true,"fontSize":11,"typeface...
"mergedCells":[{"sheetName":"Sheet1"...
```

Прямое бинарное свидетельство, что formatting и merged-cells живут внутри CRDT.

# Часть E — Semantic operations

## 21. Полный список ops

Обнаруженный обработчик `tM4` распознаёт:

```
sheet.add
sheet.set
sheet.remove

range.values.set
range.formulas.set
range.merge
range.unmerge
range.format.set
range.format.clear

conditionalformat.add
conditionalformat.clear

names.range.add
names.function.add
names.remove

table.add
table.rows.add
table.set
table.remove

chart.add
chart.set
chart.remove

datavalidation.set
datavalidation.clear
```

## 22. Правильные формы аргументов

### 22.1 Workbook.apply() — только iterable

```python
# ❌ НЕ работает
wb.apply({"op": "sheet.add", "name": "Sheet2"})
# → {} is not iterable

# ✅ работает
wb.apply([{"op": "sheet.add", "name": "Sheet2"}])
```

### 22.2 range.format.set — через props, не format

```json
{
  "op": "range.format.set",
  "target": {"sheet": "Sheet1", "range": "A1"},
  "props": {
    "fill": "#FF0000"
  }
}
```

**НЕ** `"format"` — только `"props"`.

### 22.3 applyCrdtUpdateV2 — массив чисел

```json
{
  "update": [1, 2, 3, 4]
}
```

**НЕ** `"AQ=="` (base64) — RPC не декодирует произвольно.

## 23. range.format.set — детально

### 23.1 Полный пример

```json
{
  "op": "range.format.set",
  "target": {"sheet": "Sheet1", "range": "A1"},
  "props": {
    "fill": "#FF0000",
    "font": {"bold": true, "italic": true, "size": 14, "name": "Arial", "color": "#0000FF"},
    "numberFormat": "0.00%",
    "wrapText": true,
    "horizontalAlignment": "center",
    "verticalAlignment": "middle"
  }
}
```

### 23.2 Схема цветов

| Формат                          | Результат          |
|---------------------------------|--------------------|
| `fill: "#RRGGBB"`               | ✅                 |
| `fill: {"type":"solid","color":"#RRGGBB"}` | ✅          |
| `fill: {"color":"#RRGGBB"}`     | ⚠️ пусто           |
| `font.color: "#RRGGBB"`         | ✅                 |
| `font.color: "red"`             | ⚠️ чёрный (не color name) |

### 23.3 numberFormat — автоклассификация

```
"0.00%" → {"type": "PERCENT", "pattern": "0.00%"}
```

Это правильный формат Google API.

### 23.4 Внутренний builder jM4()

Строит `userEnteredFormat` со следующими полями:

```
fill
font
numberFormat
wrapText
horizontalAlignment
verticalAlignment
```

Плюс отдельно формируются:

```
borders
rowHeight
columnWidth
```

### 23.5 Точный JSON для Google

Результат `range.format.set` в Google API:

```json
{
  "repeatCell": {
    "range": {
      "sheetId": 1,
      "startRowIndex": 0,
      "endRowIndex": 1,
      "startColumnIndex": 0,
      "endColumnIndex": 1
    },
    "cell": {
      "userEnteredFormat": {
        "backgroundColorStyle": {
          "rgbColor": {"red": 1, "green": 0, "blue": 0}
        },
        "textFormat": {
          "bold": true,
          "italic": true,
          "fontSize": 14,
          "fontFamily": "Arial",
          "foregroundColorStyle": {
            "rgbColor": {"red": 0, "green": 0, "blue": 1}
          }
        },
        "numberFormat": {"type": "PERCENT", "pattern": "0.00%"},
        "wrapStrategy": "WRAP",
        "horizontalAlignment": "CENTER",
        "verticalAlignment": "MIDDLE"
      }
    },
    "fields": "userEnteredFormat.backgroundColorStyle,userEnteredFormat.textFormat,..."
  }
}
```

# Часть F — Google bridges

## 24. GoogleSheetsAdapter и клиенты

**Классы**

```
GoogleSheetsAdapter  = Pi0
FetchGoogleSheetsClient = Vi0
GapiGoogleSheetsClient = Ci0
```

**FetchGoogleSheetsClient**

```javascript
constructor({
    accessToken,
    apiKey,
    baseUrl = "https://sheets.googleapis.com/"
})

async getSpreadsheet(id, {fields})
async batchUpdate(id, body)
```

## 25. Маппинг Artifact op → Google API

| Artifact op              | Google API                                      |
|--------------------------|-------------------------------------------------|
| range.values.set         | UpdateCellsRequest + userEnteredValue.stringValue |
| range.formulas.set       | UpdateCellsRequest + userEnteredValue.formulaValue |
| range.merge              | MergeCellsRequest                               |
| range.unmerge            | UnmergeCellsRequest                             |
| range.format.set         | RepeatCellRequest + userEnteredFormat           |
| range.format.clear       | RepeatCellRequest (пустой)                      |
| sheet.add                | AddSheetRequest                                 |
| sheet.set                | UpdateSheetPropertiesRequest                    |
| sheet.remove             | DeleteSheetRequest                              |
| datavalidation.set       | SetDataValidationRequest                        |
| conditionalformat.add    | AddConditionalFormatRuleRequest                 |
| conditionalformat.clear  | DeleteConditionalFormatRuleRequest              |
| names.range.add          | AddNamedRangeRequest                            |
| names.function.add       | (named function)                                |
| names.remove             | DeleteNamedRangeRequest                         |
| table.add                | AddTableRequest                                 |
| table.rows.add           | (table rows)                                    |
| table.set                | UpdateTableRequest                              |
| table.remove             | DeleteTableRequest                              |
| chart.add                | AddChartRequest (embeddedObject)                |
| chart.set                | UpdateChartSpecRequest                          |
| chart.remove             | DeleteEmbeddedObjectRequest                     |

## 26. Wire-format

### 26.1 URL-структура

```
GET  {baseUrl}/v4/spreadsheets/{id}?key={apiKey}
     Authorization: Bearer {accessToken}
     User-Agent: Bun/1.3.0

POST {baseUrl}/v4/spreadsheets/{id}:batchUpdate?key={apiKey}
     Authorization: Bearer {accessToken}
     Content-Type: application/json
     Body: {"requests":[...]}
```

### 26.2 applyPatch — конвертер

```python
applyPatch(
  patch,           # массив op
  {"warnings": []},# applyResult
  {}               # preApplyState
)
```

### 26.3 Точный wire-format range.values.set

```json
{
  "spreadsheetId": "FAKE",
  "requests": [
    {
      "updateCells": {
        "start": {"sheetId": 1, "rowIndex": 0, "columnIndex": 0},
        "rows": [
          {"values": [{"userEnteredValue": {"stringValue": "A"}},
                      {"userEnteredValue": {"stringValue": "B"}}]},
          {"values": [{"userEnteredValue": {"stringValue": "C"}},
                      {"userEnteredValue": {"stringValue": "D"}}]}
        ],
        "fields": "userEnteredValue"
      }
    }
  ]
}
```

### 26.4 Конвертеры цветов

`kL(color)`:

- Вход: `"#RRGGBB"` или `{type:"theme", value:...}`
- Выход: `{rgbColor: {red: 0-1, green: 0-1, blue: 0-1}}` или `{themeColor: K}`

`y1(fill)`:

- Вход: `"#RRGGBB"` или `{type:"solid", color:"..."}`
- Выход: `backgroundColorStyle`

### 26.5 datavalidation.set — точный маппинг

**Artifact:**

```json
{
  "op": "datavalidation.set",
  "target": {"sheet": "Sheet1", "range": "A1:A10"},
  "props": {
    "rule": {
      "type": "list",
      "values": ["yes", "no"]
    }
  }
}
```

**Google:**

```json
{
  "setDataValidation": {
    "range": {},
    "rule": {
      "condition": {
        "type": "ONE_OF_LIST",
        "values": [
          {"userEnteredValue": "yes"},
          {"userEnteredValue": "no"}
        ]
      }
    }
  }
}
```

### 26.6 Подмена baseUrl

`baseUrl` подменяется — можно направить запросы на свой сервер. Это полезно для экспериментов:

```python
client = construct("FetchGoogleSheetsClient", [{
    "accessToken": "FAKE",
    "apiKey": "FAKE",
    "baseUrl": "http://127.0.0.1:9998/"
}])
```

## 27. Google Spides bridge

**Классы**

```
GoogleSlidesAdapter = Fp
FetchGoogleSlidesClient = Oi0
GapiGoogleSlidesClient = Ti0
```

**Wire-format**

```
GET  {baseUrl}/v1/presentations/{id}
POST {baseUrl}/v1/presentations/{id}:batchUpdate
Authorization: Bearer {accessToken}
```

`applyPatch(patch, {warnings:[]}, {})` — тот же паттерн, что Sheets.

Использование аналогично Sheets, но для Presentation / Slide / Shape.

## 28. Плагинная мёртвая точка

### 28.1 Registry `_Z1`

В Granola (`@oai/granola`) есть plugin registry `_Z1`. Он должен регистрировать установленные плагины.

### 28.2 Проблема

Installer `installGoogleSheetsPlugin` не забандлен в bundle. Причина: `bun build --compile` не трассирует динамические `import()`.

### 28.3 Следствие

Некоторые методы падают:

- `Workbook.fromGoogleSheets` — fails (plugin not installed)
- `Workbook.configureGoogleSheets` — fails

**НО:** `GoogleSheetsAdapter` работает напрямую через RPC. То есть bridge доступен, просто не через удобные методы Workbook.

# Часть G — Практика

## 29. Минимальный рецепт

```python
# 1. Запустить RPC daemon (если не запущен автоматически)
import subprocess, os, time
sock = "/tmp/at.sock"
env = {**os.environ,
       "ARTIFACT_TOOL_RPC_SOCKET": sock,
       "ARTIFACT_TOOL_RPC_READY_FILE": "/tmp/at.ready"}
proc = subprocess.Popen(
    ["/opt/pyvenv/lib/python3.13/site-packages/artifact_tool/bin/artifact_tool_rpc_daemon"],
    env=env, start_new_session=True)
time.sleep(3)

# 2. Создать Workbook
wb = rpc("callStatic", {"class": "Workbook", "method": "create", "args": []})["result"]["objectId"]

# 3. Добавить лист
sheets = rpc("getAttr", {"target": wb, "attr": "worksheets"})["result"]["objectId"]
sheet = rpc("call", {"target": sheets, "method": "add", "args": ["Sheet1"]})["result"]["objectId"]

# 4. Записать values
rng = rpc("call", {"target": sheet, "method": "getRange", "args": ["A1:C3"]})["result"]["objectId"]
rpc("setAttr", {"target": rng, "attr": "values", "value": [
    ["Name", "Age", "City"],
    ["Alice", 30, "NY"],
    ["Bob", 25, "LA"]
]})

# 5. Записать формулу
rng2 = rpc("call", {"target": sheet, "method": "getRange", "args": ["D1"]})["result"]["objectId"]
rpc("setAttr", {"target": rng2, "attr": "formulas", "value": [["=SUM(B2:B3)"]]})

# 6. Формат
rpc("setAttr", {"target": rng, "attr": "format", "value": {
    "font": {"bold": True},
    "fill": "#FFD700"
}})

# 7. Recalculate
rpc("call", {"target": wb, "method": "recalculate", "args": []})

# 8. Сериализация
proto = rpc("callStatic", {
    "class": "SpreadsheetRealtimeSession",
    "method": "getWorkbookProtoBase64",
    "args": [{"$ref": wb}]
})["result"]

crdt = rpc("callStatic", {
    "class": "SpreadsheetRealtimeSession",
    "method": "getWorkbookCrdtStateUpdateV2",
    "args": [{"$ref": wb}]
})["result"]["data"]

# 9. Сохранить
with open("/tmp/workbook.json", "w") as f:
    json.dump({"proto": proto, "crdt": crdt}, f)
```

## 30. Правила работы

1. `$ref` обязателен. `"obj_1"` как строка — не работает.  
2. `Workbook.apply()` — только массив. Не объект.  
3. `snapshot` требует `maxDepth` для глубоких структур.  
4. `applyCrdtUpdateV2` — массив чисел, не base64.  
5. `hydrateCrdtFromProto` — только на пустом doc.  
6. `Workbook.applyCrdtUpdateV2` требует active session. Иначе `No active realtime dispatch`.  
7. `GoogleSheetsAdapter.applyPatch()` — отдельный путь. Он даёт `operations: []`, не даёт CRDT update.  
8. Настоящая CRDT-мутация — только через Workbook/Worksheet/Range.  
9. Proto — base64, CRDT — base64. Но аргумент в `applyCrdtUpdateV2` — массив чисел.  
10. Bootstrap обязателен. Сначала Proto+CRDT, потом incremental.

## 31. Открытые вопросы

- Точная protobuf schema: field numbers, message names, Cell proto, Sheet proto, Format proto, Merge proto  
- Точная Yjs структура: root maps, nested maps, keys, item IDs, client clocks, update encoding  
- Связь cell → style внутри CRDT  
- Полный lifecycle: semantic operation → Workbook state → Y.Doc update → CRDT merge → recalculation  
- Incremental collaboration: A → B через `applyToolCrdtUpdateV2` без полного bootstrap  
- Разница Proto snapshot vs CRDT snapshot vs incremental update (частично установлена)  
- Какие операции не дают CRDT update или требуют специальных transaction paths  
- Точные схемы datavalidation, conditionalformat, chart, table, names на входе (не только выходной маппинг в Google)
