# `tailwindcss_oxide-D8b55HqX.js` — полный разбор (WASM-обвязка Tailwind)

**Размер:** 6 058 байт
**URL:** `https://web-sandbox.oaiusercontent.com/assets/tailwindcss_oxide-D8b55HqX.js`
**Роль:** **`wasm-bindgen`-обвязка** для Rust-компилятора Tailwind CSS.
JS-клей для `tailwindcss_oxide_bg-DaSLdPNK.wasm` (1.72 МБ).

---

# ЧАСТЬ 1. Что это

## 1.1 Назначение

**`tailwindcss-oxide`** — Tailwind CSS v4, скомпилированный из Rust в WASM.
Сканирует код и находит Tailwind-классы (`flex`, `p-4`, `text-sm`).

**Этот файл** — **сгенерированный `wasm-bindgen` клей:**
- загрузка `.wasm`
- связывание JS ↔ WASM
- экспорт `WasmScanner` / `WasmChangedContent` / `WasmCandidateWithPosition`

**Не содержит логики** — только мост.

## 1.2 Экспорты

```js
export {
  p as WasmCandidateWithPosition,
  v as WasmChangedContent,
  O as WasmScanner,
  C as default,          // init (async)
  D as initSync,         // init (sync)
  M as wasm_init
};
```

| Экспорт | Что |
|---|---|
| `WasmScanner` | класс-сканер классов |
| `WasmChangedContent` | класс-контейнер контента |
| `WasmCandidateWithPosition` | результат с позицией |
| `default` | `init({module_or_path})` — асинхронная инициализация |
| `initSync` | синхронная инициализация |
| `wasm_init` | инициализация (алиас?) |

---

# ЧАСТЬ 2. `WasmScanner`

```js
class O {
  constructor() {
    const e = i.wasmscanner_new();
    return this.__wbg_ptr = e >>> 0,
      W.register(this, this.__wbg_ptr, this),
      this
  }

  getCandidatesWithPositions(e) {
    S(e, v);
    var n = e.__destroy_into_raw();
    const r = i.wasmscanner_getCandidatesWithPositions(this.__wbg_ptr, n);
    var s = T(r[0], r[1]).slice();
    return i.__wbindgen_free(r[0], r[1] * 4, 4), s
  }
}
```

## 2.1 Конструктор

```js
const e = i.wasmscanner_new();       // ← вызов WASM-функции
this.__wbg_ptr = e >>> 0;            // ← указатель (32-бит)
W.register(this, this.__wbg_ptr, this)
```

**`i`** — экспорты WASM-инстанса.

**`wasmscanner_new()`** — создать Rust-объект, вернуть указатель.

**`e >>> 0`** — привести к unsigned 32-bit.

**`W.register(this, ptr, this)`** — зарегистрировать JS-объект по
указателю. Это таблица **externref** — JS-значения в памяти WASM.

**Зачем:** Rust-сторона ссылается на JS-объект по индексу.

## 2.2 `getCandidatesWithPositions`

```js
S(e, v);                     // проверка типа (WasmChangedContent)
var n = e.__destroy_into_raw();   // ← consume (передача владения)
const r = i.wasmscanner_getCandidatesWithPositions(this.__wbg_ptr, n);

var s = T(r[0], r[1]).slice();    // ← Uint32Array → Array
return i.__wbindgen_free(r[0], r[1] * 4, 4), s
```

### Разбор

**`S(e, v)`** — `assertClass(e, WasmChangedContent)`. Проверка, что
передан правильный класс.

**`e.__destroy_into_raw()`** — **отдать владение в Rust**. После этого
JS-объект помечается мёртвым (`__wbg_ptr = 0`).

**Зачем:** `WasmChangedContent` — одноразовый. Rust забирает его себе.

**`i.wasmscanner_getCandidatesWithPositions(scannerPtr, contentPtr)`** —
основной вызов.

**Возврат `r`** — `[ptr, len]` (указатель + длина в памяти WASM).

**`T(r[0], r[1])`** — создать `Uint32Array` view на память WASM:
```js
function T(ptr, len) {
  return new Uint32Array(i.memory.buffer, ptr, len)
}
```

**`.slice()`** — **скопировать** в JS-массив.

**Критично:** без `slice()` view станет невалидным при следующем
аллокаторе WASM.

**`i.__wbindgen_free(r[0], r[1] * 4, 4)`** — освободить память:
- `ptr` — указатель
- `len * 4` — байт (Uint32 = 4 байта)
- `4` — alignment

**Утечка:** без `free` память WASM растёт.

## 2.3 Что возвращается

**`Uint32Array` → Array.**

В `render-X3XFXER2`:
```js
const M = S.getCandidatesWithPositions(A);
return D ? M : M.map(P => P.candidate)
```

**Если `returnPositions: false`** — каждый элемент — объект
`WasmCandidateWithPosition` с полем `candidate`.

**Если `true`** — массив объектов с позициями.

---

# ЧАСТЬ 3. `WasmChangedContent`

```js
class v {
  // (конструктор не показан, но есть где-то выше)
}
```

**Из `render-X3XFXER2`:**
```js
const A = new v(x, g);   // new WasmChangedContent(content, extension)
```

**Параметры:**
- `x` — контент (строка/код)
- `g` — расширение (`"jsx"`, `"html"`, ...)

**Поля (обычно):**
- `content` — текст
- `extension` — тип файла

**`__destroy_into_raw()`** — consume (используется в `getCandidatesWithPositions`).

---

# ЧАСТЬ 4. `WasmCandidateWithPosition`

```js
class p {
  // из exports
}
```

**Экспорт `p`.**

**Поля (из `render-X3XFXER2`):**
```js
M.map(P => P.candidate)
```

**То есть `.candidate` — строка класса** (`"flex"`, `"p-4"`).

**Плюс позиция** (при `returnPositions: true`):
- `.start` / `.end` — offset
- `.line` / `.column`

**`p.__wrap(e)`** — конструктор из указателя (используется в
`__wbg_wasmcandidatewithposition_new`).

---

# ЧАСТЬ 5. Загрузка WASM

## 5.1 `z()` — универсальный instantiate

```js
async function z(t, e) {
  // 1. Response → streaming
  if (typeof Response == "function" && t instanceof Response) {
    if (typeof WebAssembly.instantiateStreaming == "function")
      try {
        return await WebAssembly.instantiateStreaming(t, e)
      } catch (r) {
        if (t.headers.get("Content-Type") != "application/wasm")
          console.warn("`WebAssembly.instantiateStreaming` failed because your server does not serve Wasm with `application/wasm` MIME type. Falling back to `WebAssembly.instantiate` which is slower. Original error:\n", r);
        else throw r
      }

    const n = await t.arrayBuffer();
    return await WebAssembly.instantiate(n, e)
  }

  // 2. Buffer/ArrayBuffer
  else {
    const n = await WebAssembly.instantiate(t, e);
    return ...
  }
}
```

### Три пути

**1. `Response` + `instantiateStreaming`** — самый быстрый (компиляция
на лету).

**Проверка MIME:**
```js
if (t.headers.get("Content-Type") != "application/wasm") console.warn(...)
```

**Если сервер отдал без `application/wasm`** — предупреждение + fallback
на `arrayBuffer` + `instantiate` (медленнее).

**2. `Response` без streaming** — `arrayBuffer` + `instantiate`.

**3. ArrayBuffer/Buffer** — `instantiate` напрямую.

## 5.2 `A()` — импорты WASM

```js
function A() {
  const t = {};
  t.wbg = {};

  // 1. ошибка
  t.wbg.__wbg_error_7534b8e9a36f1ab4 = function (e, n) {
    let r, s;
    try {
      r = e, s = n,
      console.error(d(e, n))     // ← декодировать UTF-8 и логировать
    } finally {
      i.__wbindgen_free(r, s, 1)   // освободить
    }
  };

  // 2. new Error
  t.wbg.__wbg_new_8a6f238a6ece86ea = function () { return new Error };

  // 3. stack
  t.wbg.__wbg_stack_0ed75d68575b0f3c = function (e, n) {
    const r = n.stack,
          s = l(r, i.__wbindgen_malloc, i.__wbindgen_realloc),
          c = g;
    y().setInt32(e + 4 * 1, c, !0),   // записать в память
    y().setInt32(e + 4 * 0, s, !0)
  };

  // 4. новый объект
  t.wbg.__wbg_wasmcandidatewithposition_new = function (e) {
    return p.__wrap(e)
  };

  // 5. externref table
  t.wbg.__wbindgen_init_externref_table = function () {
    const e = i.__wbindgen_export_3,
          n = e.grow(4);
    e.set(0, void 0);
    e.set(n + 0, void 0);
    e.set(n + 1, null);
    e.set(n + 2, !0);
    e.set(n + 3, !1)
  };

  // 6. throw
  t.wbg.__wbindgen_throw = function (e, n) {
    throw new Error(d(e, n))
  };

  return t
}
```

### 6 импортов

| Имя | Что |
|---|---|
| `__wbg_error_*` | `console.error(msg)` из Rust |
| `__wbg_new_*` | создать `Error` |
| `__wbg_stack_*` | получить `.stack` (записать в память) |
| `__wbg_wasmcandidatewithposition_new` | создать JS-обёртку из указателя |
| `__wbindgen_init_externref_table` | инициализация таблицы JS-значений |
| `__wbindgen_throw` | бросить исключение из Rust |

### `__wbindgen_init_externref_table`

```js
const e = i.__wbindgen_export_3,   // ← таблица externref
      n = e.grow(4);
e.set(0, void 0);       // индекс 0 = undefined
e.set(n + 0, void 0);   // undefined
e.set(n + 1, null);     // null
e.set(n + 2, !0);       // true
e.set(n + 3, !1);       // false
```

**Стандартные сентинелы wasm-bindgen** для `Option`, `null`, `boolean`.

**Зачем:** Rust не может напрямую хранить JS-примитивы в таблице —
используются эти индексы.

### `__wbg_stack_*`

```js
y().setInt32(e + 4 * 1, c, !0),   // len
y().setInt32(e + 4 * 0, s, !0)    // ptr
```

**`y()`** — `DataView` на память WASM.

**Запись по адресу `e`:** `[ptr (4 байта), len (4 байта)]`.

**Стиль Rust:** `(ptr, len)` пара для строки.

## 5.3 `R()` — финализация

```js
function R(t, e) {
  return i = t.exports,                              // ← экспорты WASM
    C.__wbindgen_wasm_module = e,
    _ = null,                                        // сбросить cached memory
    w = null,
    i.__wbindgen_start(),                            // ← старт
    i
}
```

**`i = t.exports`** — глобальная ссылка на экспорты.

**`_ = null, w = null`** — сбросить кэш `DataView`/`Uint8Array`.

**`i.__wbindgen_start()`** — вызвать стартовую функцию Rust
(`main`/`#[wasm_bindgen(start)]`).

## 5.4 `C()` — `init` (default)

```js
async function C(t) {
  if (i !== void 0) return i;                        // уже инициализирован

  typeof t < "u" && (
    Object.getPrototypeOf(t) === Object.prototype
      ? {module_or_path: t} = t
      : console.warn("using deprecated parameters for the initialization function; pass a single object instead")
  );

  typeof t > "u" && (t = new URL("/assets/tailwindcss_oxide_bg-DaSLdPNK.wasm", import.meta.url));

  const e = A();
  (typeof t == "string"
   || typeof Request == "function" && t instanceof Request
   || typeof URL == "function" && t instanceof URL)
    && (t = fetch(t));

  const {instance: n, module: r} = await z(await t, e);
  return R(n, r)
}
```

### Разбор

**1. Кэш:** если `i` уже установлен — вернуть.

**2. Параметры:**
- объект `{module_or_path}` → распаковать
- старый стиль (строка/URL) → предупреждение

**3. По умолчанию:**
```js
new URL("/assets/tailwindcss_oxide_bg-DaSLdPNK.wasm", import.meta.url)
```

**Путь от `import.meta.url`** — то есть от расположения JS-файла.

**4. `fetch(t)`** — если строка/URL/Request.

**5. `z(await t, e)`** — instantiate.

**6. `R(n, r)`** — финализация.

### Важно: URL жёстко относительный

```js
new URL("/assets/tailwindcss_oxide_bg-DaSLdPNK.wasm", import.meta.url)
```

**`/assets/...` — абсолютный путь от origin**, не от `import.meta.url`!

**`new URL(relative, base)`** — если `relative` начинается с `/` —
абсолютный от origin `base`.

**То есть:** `https://web-sandbox.oaiusercontent.com/assets/tailwindcss_oxide_bg-DaSLdPNK.wasm`.

**Если JS-файл перенесут в другой путь** — `.wasm` всё равно ищется в
`/assets/`.

## 5.5 `D()` — `initSync`

```js
function D(t) {
  if (i !== void 0) return i;

  typeof t < "u" && (
    Object.getPrototypeOf(t) === Object.prototype
      ? {module: t} = t
      : console.warn("using deprecated parameters for `initSync()`; pass a single object instead")
  );

  t instanceof WebAssembly.Module || (t = new WebAssembly.Module(t));

  const e = A(),
        n = new WebAssembly.Instance(t, e);

  return R(n, t)
}
```

**Синхронная версия.** Принимает `WebAssembly.Module` или байты.

**`new WebAssembly.Module(bytes)`** — компиляция синхронно (блокирует).

**`new WebAssembly.Instance(module, imports)`** — инстанцирование.

---

# ЧАСТЬ 6. Работа с памятью WASM

## 6.1 `b()` / `d()` — чтение

```js
let w = null;

function b() {
  return (w === null || w.byteLength === 0)
    && (w = new Uint8Array(i.memory.buffer)),
    w
}

function d(t, e) {
  return t = t >>> 0, x.decode(b().subarray(t, t + e))
}
```

**`b()`** — `Uint8Array` view на всю память WASM.

**Кэш `w`:** создаётся один раз. **Но** если память **выросла** (Rust
выделил больше) — `byteLength` изменится, старый view станет detached.

**Проверка `w.byteLength === 0`** — детект detached view. **Ок.**

**`d(ptr, len)`** — декодировать UTF-8.

## 6.2 `y()` — DataView

```js
function y() {
  // (не показано, но из использования)
  return new DataView(i.memory.buffer)
}
```

**Скорее всего тоже кэшируется.**

**Используется** для `setInt32` в `__wbg_stack_*`.

## 6.3 `l()` — строка → память

```js
function l(t, e, n) {
  // (из использования)
  // encode UTF-8, malloc, вернуть ptr
}
```

**`l(str, malloc, realloc)`** — закодировать строку и поместить в память
WASM.

## 6.4 `T()` — view

```js
function T(t, e) {
  return new Uint32Array(i.memory.buffer, t, e)
}
```

**`Uint32Array(ptr, len)`** — view на массив `u32`.

---

# ЧАСТЬ 7. TextEncoder / TextDecoder

```js
let i;    // wasm exports

const x = typeof TextDecoder < "u"
  ? new TextDecoder("utf-8", {ignoreBOM: !0, fatal: !0})
  : {decode: () => { throw Error("TextDecoder not available") }};

typeof TextDecoder < "u" && x.decode();

let w = null;

const g = 0;

const u = typeof TextEncoder < "u"
  ? new TextEncoder("utf-8")
  : {encode: () => { throw Error("TextEncoder not available") }};

const F = typeof u.encodeInto == "function"
  ? function (t, ...) { ... }
  : ...;
```

## 7.1 `ignoreBOM: true, fatal: true`

**`ignoreBOM: true`** — не удалять BOM (U+FEFF) из начала.

**`fatal: true`** — при невалидном UTF-8 **бросить**, а не заменить на
U+FFFD.

**`x.decode()`** — пустой вызов при инициализации (прогрев).

## 7.2 `g = 0`

**Загадочная константа.** Объявлена `const g = 0`, используется в
`__wbg_stack_*`:
```js
const c = g;
y().setInt32(e + 4 * 1, c, !0)
```

**Значит `c = 0`** — длина стека записывается как 0?

**Проверить:** `g` объявлена после `d()`. В `__wbg_stack_*` — `c = g`.

**Возможно, `g` — результат `u.encodeInto(...)` от последней операции**
(позиция записи). Но `const g = 0` — фиксировано.

**Гипотеза:** это **баг или остаток** от `encodeInto`-пути. `g` должна
обновляться в `F`:
```js
const F = typeof u.encodeInto == "function"
  ? function (t, e) { g = u.encodeInto(t, e).written }   // ← но const!
```

**`const g = 0`** — нельзя изменить. Значит стек всегда пишется с
длиной 0.

**Последствие:** `.stack` в ошибках WASM будет **пустым**.

---

# ЧАСТЬ 8. `W` — реестр объектов

```js
W.register(this, this.__wbg_ptr, this)
```

**`W`** — `FinalizationRegistry` или `Map` JS-объектов по указателям.

**Скорее всего `FinalizationRegistry`:**
```js
W = new FinalizationRegistry(ptr => { ... free ... })
```

**Зачем:** когда JS-объект собирается GC, освободить Rust-объект.

**`register(target, heldValue, unregisterToken)`** — стандартный API.

**Риск:** `FinalizationRegistry` ненадёжен (GC не гарантирует вызов).
Утечки возможны.

---

# ЧАСТЬ 9. Схема

```
init({module_or_path})    [default = C]
 │
 ├─ если i уже есть → вернуть
 ├─ по умолчанию: new URL("/assets/tailwindcss_oxide_bg-DaSLdPNK.wasm")
 ├─ fetch(url) → Response
 ├─ A() → объект импортов (6 функций)
 ├─ z(response, imports)
 │    ├─ instantiateStreaming (если MIME application/wasm)
 │    └─ fallback: arrayBuffer + instantiate
 └─ R(instance, module)
      ├─ i = instance.exports
      ├─ сбросить кэш памяти (w = null)
      └─ i.__wbindgen_start()

new WasmScanner()           [O]
 ├─ i.wasmscanner_new() → ptr
 ├─ this.__wbg_ptr = ptr
 └─ W.register(this, ptr, this)

new WasmChangedContent(content, ext)    [v]
 └─ (Rust-объект с content/extension)

scanner.getCandidatesWithPositions(content)
 ├─ assertClass(content, WasmChangedContent)
 ├─ ptr = content.__destroy_into_raw()    ← consume
 ├─ [outPtr, outLen] = i.wasmscanner_getCandidatesWithPositions(scannerPtr, ptr)
 ├─ arr = Uint32Array(memory, outPtr, outLen).slice()
 ├─ i.__wbindgen_free(outPtr, outLen * 4, 4)
 └─ → Array of WasmCandidateWithPosition
```

---

# ЧАСТЬ 10. Слабые места

## 10.1 `g = 0` — `.stack` всегда пустой

```js
const g = 0;
...
const c = g;
y().setInt32(e + 4 * 1, c, !0)
```

**`g` — `const`.** Не обновляется.

**Значит длина стека всегда 0** → Rust получит пустой стек.

**Последствие:** ошибки WASM без стектрейса.

## 10.2 `new URL("/assets/...", import.meta.url)` — абсолютный путь

```js
t = new URL("/assets/tailwindcss_oxide_bg-DaSLdPNK.wasm", import.meta.url)
```

**`/assets/...`** — от origin, не от `import.meta.url`.

**Если `.wasm` перенесут** — 404.

**Правильно:** `new URL("./tailwindcss_oxide_bg-DaSLdPNK.wasm", import.meta.url)`.

## 10.3 `slice()` после view — обязательно

```js
var s = T(r[0], r[1]).slice();
return i.__wbindgen_free(r[0], r[1] * 4, 4), s
```

**`slice()` создаёт копию.** Без неё view стал бы невалидным.

**Правильно.**

**Но:** лишняя аллокация. Для больших массивов — заметно.

## 10.4 `__destroy_into_raw()` — consume

```js
var n = e.__destroy_into_raw();
```

**`WasmChangedContent` одноразовый.** После вызова — `__wbg_ptr = 0`.

**Повторный вызов** → `Error: null pointer passed to rust`.

## 10.5 `FinalizationRegistry` (`W`) — ненадёжен

**GC не гарантирует вызов.** Rust-объект может не освободиться.

**Утечка памяти WASM** при многих сканерах.

## 10.6 Кэш `w` при росте памяти

```js
function b() {
  return (w === null || w.byteLength === 0)
    && (w = new Uint8Array(i.memory.buffer)),
    w
}
```

**Проверка `byteLength === 0`** — детект detached. **Но:**

**`memory.grow()`** создаёт **новый** `ArrayBuffer`. Старый `w` станет
detached → `byteLength === 0` → пересоздастся. **Ок.**

**Но если `memory.grow()` не было** — `w` валиден. **Ок.**

**Логика верная.**

## 10.7 `_ = null, w = null` в `R()`

```js
function R(t, e) {
  return i = t.exports,
    C.__wbindgen_wasm_module = e,
    _ = null,
    w = null,
    i.__wbindgen_start(),
    i
}
```

**Сброс кэшей** после инициализации. **Правильно.**

**Но:** если кто-то держал старый `w` — станет detached.

## 10.8 Нет `fetch` options

```js
&& (t = fetch(t));
```

**Без `credentials`, `integrity`, `cache`.**

**SRI не проверяется.** Подмена `.wasm` на CDN/сервере = выполнение
чужого кода.

**НО:** в этом случае `.wasm` локальный (`/assets/`, тот же origin).

## 10.9 `console.warn` вместо ошибки при неверном MIME

```js
console.warn("`WebAssembly.instantiateStreaming` failed because your server does not serve Wasm with `application/wasm` MIME type...")
```

**Fallback на `arrayBuffer`.** Работает, но медленнее.

**Проблема:** если сервер отдаёт `.wasm` как `text/plain` — каждый
запуск будет предупреждать.

## 10.10 Нет `deprecated` пути для `initSync`

```js
typeof t < "u" && (
  Object.getPrototypeOf(t) === Object.prototype
    ? {module: t} = t
    : console.warn("using deprecated parameters for `initSync()`; ...")
);
```

**Потом:**
```js
t instanceof WebAssembly.Module || (t = new WebAssembly.Module(t));
```

**Если `t` — строка** (путь) → `new WebAssembly.Module("path")` → ошибка.

**Не поддерживается загрузка по пути в `initSync`.**

## 10.11 `x.decode()` при инициализации — пустой вызов

```js
typeof TextDecoder < "u" && x.decode();
```

**Прогрев декодера.** Не критично.

---

# ЧАСТЬ 11. Связка с `render-X3XFXER2`

## 11.1 Как используется

```js
async function Ree({content: x = "", returnPositions: D = !1, extension: g = "html"}) {
  let f, v;
  try {
    const P = await vee(() => import("./tailwindcss_oxide-D8b55HqX.js"), []);
    await P.default({module_or_path: new URL(OGe.replace("?url", ""), import.meta.url)});
    f = P.WasmScanner;
    v = P.WasmChangedContent;
  } catch (P) {
    throw new Error("Failed to load WASM scanner: " + P.message)
  }

  const S = new f,
        A = new v(x, g),
        M = S.getCandidatesWithPositions(A);

  return D ? M : M.map(P => P.candidate)
}
```

### Разбор

**1. `import()`** — динамическая загрузка JS-клея.

**2. `P.default({module_or_path: new URL(...)})`** — инициализация:

```js
const OGe = "/assets/tailwindcss_oxide_bg-DaSLdPNK.wasm";
...
new URL(OGe.replace("?url", ""), import.meta.url)
```

**Итог:** `https://web-sandbox.oaiusercontent.com/assets/tailwindcss_oxide_bg-DaSLdPNK.wasm`

**3. `new WasmScanner()`** — создать сканер.

**4. `new WasmChangedContent(code, "jsx")`** — контент.

**5. `scanner.getCandidatesWithPositions(content)`** — сканировать.

**6. `M.map(P => P.candidate)`** — извлечь строки.

## 11.2 Проблема: сканер не закрывается

```js
const S = new f,          // WasmScanner
      A = new v(x, g),    // WasmChangedContent
      M = S.getCandidatesWithPositions(A);
```

**`S` (WasmScanner) не освобождается.**

**Утечка:** каждый вызов `Ree()` создаёт Rust-объект, который живёт до GC.

**При многих виджетах** — память WASM растёт.

**Правильно:** `S.free()` (если есть метод) или `S.__destroy_into_raw()`.

**`wasm-bindgen` обычно генерирует `free()`** — но в этом файле **его нет**.

**Проверить:** в `O` (WasmScanner) только `constructor` и
`getCandidatesWithPositions`. **Нет `free`.**

**Вывод: утечка гарантирована.**

---

# ЧАСТЬ 12. Итог

## 12.1 Ключевые числа

| Что | Значение |
|---|---|
| Размер JS-клея | 6 058 байт |
| Размер WASM | 1 721 212 байт (1.64 МБ) |
| Экспортов | 6 |
| Импортов WASM | 6 |
| Реальный код | ~4 КБ (остальное — boilerplate) |
| Путь WASM | `/assets/tailwindcss_oxide_bg-DaSLdPNK.wasm` |

## 12.2 Главное

**1. Это автогенерированный `wasm-bindgen` клей.** Никакой логики —
только мост JS ↔ WASM.

**2. 6 импортов для Rust:** error, new Error, stack, new Candidate,
init externref table, throw.

**3. `__wbindgen_init_externref_table`** — сентинелы `undefined`, `null`,
`true`, `false` по индексам.

**4. `getCandidatesWithPositions` consume-ит `WasmChangedContent`**
через `__destroy_into_raw()`.

**5. `slice()` после view — обязательно**, иначе detached.

**6. `__wbindgen_free(ptr, len*4, 4)`** — освобождение массива `u32`.

**7. `g = 0` — `.stack` всегда пустой.**
`const g = 0` не обновляется, а используется как длина.

**8. URL `.wasm` — абсолютный `/assets/`**, не от `import.meta.url`.

**9. Без SRI, без `credentials`** — `fetch(t)` без опций.

**10. `WasmScanner` не имеет `free()` — утечка Rust-объектов.**

**11. `FinalizationRegistry` (`W`) ненадёжен** — GC не гарантирует
освобождение.

**12. Правильный fallback при неверном MIME:** `console.warn` +
`arrayBuffer` + `instantiate`.
