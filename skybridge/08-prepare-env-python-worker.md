# `prepare-environment` + `python-worker` + `pyodide` — полный разбор

| Файл | Размер | Роль |
|---|---|---|
| `prepare-environment-DVxA1AjJ.js` | 1 674 байта | прогрев окружения (lazy load) |
| `python-worker-Y5Z0ImDG.js` | 4 841 байт | Python-исполнитель в Web Worker |
| `pyodide-CAuAWlEr.js` | 13 967 байт | загрузчик Pyodide |

---

# ЧАСТЬ 1. `prepare-environment-DVxA1AjJ.js`

## 1.1 Полный код

```js
const __vite__mapDeps = (i, m = __vite__mapDeps, d = (m.f || (m.f = [
  "assets/transpile-Cld0n4VL.js",              // [0]
  "assets/main-BR1bOQyH.js",                   // [1]
  "assets/main-foMHhJws.css",                  // [2]
  "assets/run-recharts-CQOvWM4G.js",           // [3]
  "assets/run-react-vAjeCFay.js",              // [4]
  "assets/get-html-D9pLuxCy.js",               // [5]
  "assets/parse-transpile-error-CaCAx2Mb.js",  // [6]
  "assets/index-C0fqnZRQ.js",                  // [7]
  "assets/client-BjybnHiv.js",                 // [8]
  "assets/index-BTTCYQt9.js",                  // [9]
  "assets/three.module-C846Yxd8.js",           // [10]
  "assets/events-5a94e5eb.esm-DZJsuX-L.js",    // [11]
  "assets/react-three-fiber.esm-C_wkhiKk.js"   // [12]
]))) => i.map(i => d[i]);

import {_ as t} from "./main-BR1bOQyH.js";

let a = !1;    // ← ГЛОБАЛЬНЫЙ замок

const p = async (i, {language: s, id: o, code: n = ""}) => {
  if (!a) {
    a = !0;
    try {
      switch (s) {
        case "react": {
          const {transpileReact: r} = await t(
            async () => {
              const {transpileReact: e} = await import("./transpile-Cld0n4VL.js").then(_ => _.a);
              return {transpileReact: e}
            },
            __vite__mapDeps([0, 1, 2])
          );
          await r({id: o, sandboxLogger: i, code: n, worker: !0});
          return
        }
        case "recharts": {
          const [{transpileReact: r}, {prepareRechartsCode: e}] = await Promise.all([
            t(() => import("./transpile-Cld0n4VL.js").then(_ => _.a), __vite__mapDeps([0, 1, 2])),
            t(() => import("./run-recharts-CQOvWM4G.js"), __vite__mapDeps([3, 1, 2, 4, 0, 5, 6]))
          ]);
          await r({id: o, sandboxLogger: i, code: e(n), worker: !0, disableShadcnImports: !0});
          return
        }
        case "threejs": {
          await Promise.all([
            t(() => import("./index-C0fqnZRQ.js").then(r => r.i), __vite__mapDeps([7, 1, 2])),
            t(() => import("./client-BjybnHiv.js").then(r => r.c), __vite__mapDeps([8, 1, 2, 7])),
            t(() => import("./index-BTTCYQt9.js"), __vite__mapDeps([9, 7, 1, 2, 8, 10, 11])),
            t(() => import("./react-three-fiber.esm-C_wkhiKk.js"), __vite__mapDeps([12, 11, 7, 1, 2, 10])),
            t(() => import("./three.module-C846Yxd8.js").then(r => r.T), [])
          ]);
          return
        }
        default:
          return
      }
    } catch {} finally {
      a = !1
    }
  }
};

export {p as prepareEnvironment};
```

## 1.2 Что это

**Одна функция `p(logger, {language, id, code})`.** Задача: **прогреть** модули,
чтобы первый реальный запуск был быстрым.

**Ничего не запускает** — только загружает и вызывает транспиляцию.

## 1.3 Глобальный замок `a`

```js
let a = !1;

const p = async (i, {...}) => {
  if (!a) {
    a = !0;
    try { ... } catch {} finally { a = !1 }
  }
};
```

**Что происходит при параллельных вызовах:**

```js
prepareEnvironment(log, {language: "react", ...})   // a = true, пошёл работать
prepareEnvironment(log, {language: "react", ...})   // a === true → НИЧЕГО не делает, возвращает undefined
```

**Второй вызов молча игнорируется.** Не ждёт первого, не ставится в очередь,
просто выходит.

**Почему так:** прогревать параллельно нет смысла — модули одни и те же,
`<link modulepreload>` и так дедуплицируются через `An` в `_t`. Но
**второй вызов теряется**, и если он был с **другим** языком — прогрев
не произойдёт.

**Пример:**
```js
prepareEnvironment(log, {language: "python"})   // запустил
prepareEnvironment(log, {language: "threejs"})  // ОТБРОШЕН — a = true
```

Хост думает, что прогрел three.js, а на деле — нет.

## 1.4 `react` — прогон транспиляции

```js
case "react": {
  const {transpileReact: r} = await t(
    async () => {
      const {transpileReact: e} = await import("./transpile-Cld0n4VL.js").then(_ => _.a);
      return {transpileReact: e}
    },
    __vite__mapDeps([0, 1, 2])
  );
  await r({id: o, sandboxLogger: i, code: n, worker: !0});
  return
}
```

**Пошагово:**
1. `_t(...)` — preload 3 файла (транспилер, main, css)
2. `import("./transpile-Cld0n4VL.js").then(_ => _.a)` — экспорт `a` =
   `wJe` (модуль с `transpileReact`)
3. **Вызов `transpileReact({code, worker: true})`** — реальная транспиляция!

**Что прогревается:**
- esbuild-wasm инициализируется (`hWe` → `pZ.initialize`)
- babel парсит код
- tailwind-wasm компилирует CSS
- esbuild.context создаётся

**`minify` не передан** → `false` (по умолчанию в `yJe`).

**`id: o`** — важен! В `transpileReact` есть кэш `La`:

```js
if (!La || !La.id || La.id !== x) {
  La = {id: x, ..., buildContext: await A.context({...})}
} else {
  La.indexFile = U;
  La.sourceCode = D;
  La.css = [];
}
```

**Если `id` совпадает с тем, что будет при реальном запуске** — `buildContext`
**переиспользуется**. Если нет — создаётся новый (дорого).

**То есть хост должен передавать в `prepareEnvironment` тот же `id`, что
и в `runUserCode`.** Иначе прогрев бесполезен для esbuild-контекста.

**`code: n = ""`** — если код не передан, транспилируется пустая строка.
Всё равно прогревает babel/esbuild/tailwind.

## 1.5 `recharts`

```js
case "recharts": {
  const [{transpileReact: r}, {prepareRechartsCode: e}] = await Promise.all([
    t(() => import("./transpile-Cld0n4VL.js").then(_ => _.a), __vite__mapDeps([0, 1, 2])),
    t(() => import("./run-recharts-CQOvWM4G.js"), __vite__mapDeps([3, 1, 2, 4, 0, 5, 6]))
  ]);
  await r({id: o, sandboxLogger: i, code: e(n), worker: !0, disableShadcnImports: !0});
  return
}
```

**Отличия от react:**
- `Promise.all` — два параллельных импорта
- `code: e(n)` — код проходит через `prepareRechartsCode`
- **`disableShadcnImports: true`**

**Зачем `disableShadcnImports`:** recharts-виджеты не используют shadcn-компоненты.
Отключение экономит время (не сканируется AST на компоненты) и убирает
импорты из бандла.

## 1.6 `threejs`

```js
case "threejs": {
  await Promise.all([
    t(() => import("./index-C0fqnZRQ.js").then(r => r.i), __vite__mapDeps([7, 1, 2])),
    t(() => import("./client-BjybnHiv.js").then(r => r.c), __vite__mapDeps([8, 1, 2, 7])),
    t(() => import("./index-BTTCYQt9.js"), __vite__mapDeps([9, 7, 1, 2, 8, 10, 11])),
    t(() => import("./react-three-fiber.esm-C_wkhiKk.js"), __vite__mapDeps([12, 11, 7, 1, 2, 10])),
    t(() => import("./three.module-C846Yxd8.js").then(r => r.T), [])
  ]);
  return
}
```

**Пять параллельных импортов:**

| # | Файл | Экспорт | Что |
|---|---|---|---|
| 1 | `index-C0fqnZRQ.js` (8.5 КБ) | `i` | React Three Fiber core |
| 2 | `client-BjybnHiv.js` (208 КБ) | `c` | клиент three |
| 3 | `index-BTTCYQt9.js` (1.6 МБ) | целиком | основной three-бандл |
| 4 | `react-three-fiber.esm-C_wkhiKk.js` (5.7 КБ) | целиком | R3F ESM |
| 5 | `three.module-C846Yxd8.js` (704 КБ) | `T` | three.js модуль |

**Итого: ~2.5 МБ** для threejs-прогрева.

**Обрати внимание:** это **не** то же, что `runThree`. `runThree` грузит
three через `importmap` с **unpkg**. А `prepareEnvironment` грузит **локальные**
бандлы (вшитые в Skybridge).

**То есть прогрев threejs не помогает `runThree`!** Он грузит другой источник.

**Вывод: `prepareEnvironment` для threejs бесполезен** — прогревает
вшитые копии, которые `runThree` не использует.

Скорее всего эти бандлы нужны для какого-то другого пути (возможно,
`runComponent` — `di()`, который вызывает `Dr()` напрямую).

## 1.7 `default` — молча

```js
default:
  return
```

Для `python`, `javascript`, `html`, `typescript` — ничего.

**Python не прогревается.** Первый запуск будет ждать загрузку Pyodide
(~10 МБ с jsdelivr).

**Это упущение:** для Python прогрев был бы полезнее всего.

## 1.8 `catch {}`

```js
} catch {} finally {
  a = !1
}
```

**Ошибки прогрева глотаются молча.** Если транспиляция упала — хост не узнает.
Прогрев — best-effort.

## 1.9 Слабые места

### 1. Второй вызов теряется

```js
if (!a) { a = !0; ... }
```

Параллельный вызов с другим языком — отброшен.

### 2. `threejs` прогревает не то

Грузит вшитые бандлы (`three.module-C846Yxd8.js`), а `runThree` использует
unpkg через importmap. **Прогрев не помогает.**

### 3. `python` не прогревается вообще

`default: return`. Самый тяжёлый язык (Pyodide ~10 МБ) — без прогрева.

### 4. `catch {}` скрывает ошибки

Если `transpileReact` упал на прогреве, хост не узнает — а реальный запуск
упадёт тоже.

### 5. `id` должен совпадать

Кэш `La` в `transpileReact` ключуется по `id`. Если хост передаёт разные
`id` в `prepareEnvironment` и `runUserCode` — прогрев не сэкономит esbuild-контекст.

---

# ЧАСТЬ 2. `python-worker-Y5Z0ImDG.js`

## 2.1 Общая структура

Это **код, который выполняется в Web Worker**. Не модуль, а сам воркер.
Внутри — всё для запуска Python через Pyodide.

```js
var c = (t => (t.LOG="log", t.RUN_START="run_start", ..., t))(c||{}),
    g = (t => (t[t.INITIALIZING=0]="INITIALIZING", ...))(g||{});

Function.prototype.toString.call(Object);

const N = (t, e) => {...};    // reformat_exception
const h = t => {...};         // извлечь строку из сообщения
const b = t => {...};         // извлечь имя класса
const I = t => {...};         // разобрать Python-исключение
const P = async () => {...};  // loadPyodide
const E = (t, e) => {...};    // настроить matplotlib
const m = `image_bytes_list_${Math.random()...}`;   // ← случайное имя
const T = (t, e) => {...};    // закрыть графики
const A = (t, e) => {...};    // извлечь PNG
const k = (t, e, o) => {...}; // перехват print
const L = async (t, e, o) => {...};  // установка пакетов
const l = postMessage;
let y = null;                 // кэш Pyodide
const R = async () => y ?? (y = await P());
const _ = t => {...};         // ошибка JS
const O = (t, e) => {...};    // ошибка Python
const U = async t => {...};   // ГЛАВНАЯ функция запуска
self.onmessage = async t => {...};   // обработчик
l({type: "ready"});           // ← сигнал готовности
```

## 2.2 `N` — `reformat_exception`

```js
const N = (t, e) => {
  t.runPython(`
def reformat_exception():
  import sys
  from traceback import format_exception, format_tb, extract_tb
  return {
      "line": extract_tb(sys.last_traceback)[-1].lineno,
      "lastValue": str(sys.last_value),
      "lastType": str(sys.last_type),
      "traceback": format_tb(sys.last_traceback)[2:]
  }
`.trim(), {globals: e})
};
```

**Внедряет Python-функцию** `reformat_exception` в глобальное пространство
пользователя.

**Что извлекает:**
- `line` — номер строки последнего фрейма (`extract_tb(...)[-1].lineno`)
- `lastValue` — текст исключения (`str(sys.last_value)`)
- `lastType` — тип (`str(sys.last_type)`) — например `<class 'ValueError'>`
- `traceback` — список строк, **начиная с [2:]**

**`[2:]`** — обрезает первые две строки. Первая — `Traceback (most recent call last):`,
вторая — фрейм самого Pyodide (`_pyodide/_base.py`). Пользователю они не нужны.

**`sys.last_traceback`** — заполняется только после исключения. Значит
`reformat_exception` нужно вызывать **в блоке `except`**.

## 2.3 `h` — извлечь строку

```js
const h = t => {
  if (typeof t != "string") return null;
  const e = t.match(/\(<exec>, line (\d+)\)/);
  if (!e) return null;
  const [, o] = e, n = parseInt(o);
  return isNaN(n) ? null : n;
};
```

**Что ищет:** `(<exec>, line 5)` в сообщении об ошибке.

**Откуда такой формат:** Pyodide оборачивает код в `<exec>` для `runPython`.
Сообщения об ошибках содержат эту метку.

**Пример:**
```
ValueError: invalid literal for int()
(<exec>, line 5)
```
→ вернёт `5`.

**`isNaN`** — защита от `parseInt("abc")`.

## 2.4 `b` — извлечь имя класса

```js
const b = t => {
  if (typeof t != "string") return null;
  const e = t.match(/<class '([a-zA-Z0-9]*)'/);
  if (!e) return null;
  const [, o] = e;
  return o || null;
};
```

**Что ищет:** `<class 'ValueError'` → `ValueError`.

**`[a-zA-Z0-9]*`** — только буквы и цифры. Точки не входят (для
`module.Class` вернёт только первую часть... нет, не вернёт — `[a-zA-Z0-9]*`
жадный, но точка не входит. `<class 'foo.Bar'>` → `foo`).

**Ограничение:** для исключений с модулем в имени (`asyncio.TimeoutError`)
вернёт `asyncio`.

## 2.5 `I` — разбор Python-исключения

```js
const I = t => {
  const o = t.get("reformat_exception")(),
        {line: n, lastValue: r, lastType: i, traceback: s} =
          o.toJs({create_proxies: !1, dict_converter: Object.fromEntries}),
        a = s
          .filter(f => !String(f).includes("_pyodide/_base.py"))
          .flatMap(f => f.replace('File "<exec>",', "").split("\n"))
          .filter(Boolean),
        p = h(r),
        u = b(i);

  o.destroy();

  return {name: u, message: r, line: p ?? n, stack: a.length > 1 ? a : []}
};
```

### Пошагово

**1. `t.get("reformat_exception")()`** — вызвать внедрённую Python-функцию.
Возвращает PyProxy на Python-dict.

**2. `.toJs({create_proxies: false, dict_converter: Object.fromEntries})`** —
конвертировать Python-dict в JS-объект.

- `create_proxies: false` — не создавать прокси для вложенных объектов
  (быстрее, но вложенные списки станут JS-массивами)
- `dict_converter: Object.fromEntries` — использовать `Object.fromEntries`
  вместо стандартного конвертера (даёт обычный JS-объект вместо Map)

**3. `a` — очистка стектрейса:**

```js
s.filter(f => !String(f).includes("_pyodide/_base.py"))
```
Убрать фреймы самого Pyodide.

```js
.flatMap(f => f.replace('File "<exec>",', "").split("\n"))
```
Разбить каждую запись на строки (формат traceback — многострочный).

```js
.filter(Boolean)
```
Убрать пустые.

**4. `p = h(r)`** — строка из **сообщения** (regex `<exec>, line N`).

**5. `u = b(i)`** — имя класса из `lastType`.

**6. `o.destroy()`** — освободить PyProxy. **Критично** — иначе утечка памяти
в WASM.

**7. Возврат:**
```js
{
  name: u,                        // "ValueError"
  message: r,                     // "invalid literal for int()..."
  line: p ?? n,                   // строка из сообщения, иначе из traceback
  stack: a.length > 1 ? a : []    // стек только если > 1 строки
}
```

**`line: p ?? n`** — приоритет: сначала regex из сообщения, потом
`extract_tb`. Опять `??` — только для `null`/`undefined`.

**`stack: a.length > 1 ? a : []`** — если в стеке **одна** строка, вернуть
пустой массив. То есть для одиночных ошибок стек не показывается.

## 2.6 `P` — загрузка Pyodide

```js
const P = async () => {
  const {loadPyodide: t} = await import("./pyodide-CAuAWlEr.js");
  return await t({indexURL: "https://cdn.jsdelivr.net/pyodide/v0.26.4/full"})
};
```

**Версия:** Pyodide **0.26.4**.

**Источник:** `https://cdn.jsdelivr.net/pyodide/v0.26.4/full`

**Что грузится:**
- `pyodide.asm.js` (~10 МБ)
- `pyodide.asm.wasm` (~9 МБ)
- `python_stdlib.zip` (~1.5 МБ)
- `pyodide-lock.json`

**Итого ~20 МБ** при первом запуске.

**Без SRI.** Компрометация jsdelivr = произвольный Python-код.

## 2.7 `E` — настройка matplotlib

```js
const E = (t, e) => {
  t.runPython(`
import warnings
import matplotlib

matplotlib.use("AGG")
warnings.filterwarnings(
  "ignore",
  message="Matplotlib is currently using agg, which is a non-GUI backend, so cannot show the figure."
)
`.trim(), {globals: e})
};
```

**`matplotlib.use("AGG")`** — бэкенд без GUI. В браузере нет дисплея,
поэтому нужен именно AGG (рендер в память).

**Фильтр предупреждений** — matplotlib ругается, что AGG не GUI. Отключаем.

**Вызывается только если `code.includes("matplotlib")`.**

## 2.8 `m` — магическое имя

```js
const m = `image_bytes_list_${Math.random().toString().split(".")[1]}`;
```

**Случайное имя глобальной переменной** для списка PNG.
`image_bytes_list_3829471932847`.

**Зачем случайное:** чтобы не столкнуться с переменной пользователя.
Если пользователь объявит `image_bytes_list`, конфликта не будет.

**Проблема:** `Math.random().toString()` может дать `"0.5"` → `split(".")[1]`
= `"5"`. Или для очень больших чисел — экспоненциальную запись
`"1.2345e-7"` → `split(".")[1]` = `"2345e-7"`. Тогда имя станет
`image_bytes_list_2345e-7` — **невалидный идентификатор Python**!

**Вероятность:** `Math.random()` даёт `[0, 1)`. Экспоненциальная запись
для чисел < 1e-6. Вероятность ~1e-6. Редко, но возможно.

## 2.9 `T` — закрыть графики

```js
const T = (t, e) => {
  t.runPython(`
from matplotlib import pyplot as plt
for i, fig in enumerate(plt.get_fignums()):
    try:
      plt.close(fig)
    except Exception:
      continue
`.trim(), {globals: e})
};
```

**Закрывает все открытые фигуры** после запуска. Освобождает память WASM.

**`try/except`** — если фигура уже закрыта, не падать.

## 2.10 `A` — извлечь PNG

```js
const A = (t, e) => {
  t.runPython(`
import io
from matplotlib import pyplot as plt

${m} = []
for i, fig in enumerate(plt.get_fignums()):
    plt.figure(fig)

    # Save the plot to a BytesIO buffer
    buf = io.BytesIO()
    plt.savefig(buf, format='png')
    buf.seek(0)
    ${m}.append(buf.read())
`.trim(), {globals: e});

  const o = e.get(m);
  if (!o) return [];

  const n = [];
  for (let r = 0; r < o.length; r++) {
    const i = o.get(r),
          s = new Uint8Array(i.toJs());

    let a = "";
    for (const f of s) a += String.fromCharCode(f);

    const u = `data:image/png;base64,${btoa(a)}`;
    n.push(u);
  }

  o.destroy();
  return n
};
```

**Python-часть:**
1. Для каждой фигуры (`plt.get_fignums()`)
2. `plt.figure(fig)` — сделать активной
3. `plt.savefig(buf, format='png')` — сохранить в память
4. `buf.read()` — прочитать байты
5. Добавить в список `m`

**JS-часть:**
1. `e.get(m)` — получить список из Python-глобалов
2. Для каждого элемента: `i.toJs()` → `Uint8Array` → base64
3. Вернуть массив `data:image/png;base64,...`

**`btoa`** — бинарное в ASCII. Для больших изображений — медленно
(цикл `String.fromCharCode` побайтово).

**Оптимизация была бы:** `btoa(String.fromCharCode(...s))` — но для больших
массивов переполнение стека. Или `chunked`-подход.

**`o.destroy()`** — освободить PyProxy списка. **Но `i` (элементы) не**
`destroy`-ятся — потенциальная утечка.

## 2.11 `k` — перехват `print`

```js
const w = "|", d = " | ";

const k = (t, e, o) => {
  t.runPython(`
import inspect
_print = print
def print(*args):
   _print(inspect.getframeinfo(inspect.currentframe().f_back).lineno, "${w}", *args)
`.trim(), {globals: e}),

  t.setStdout({
    write: n => {
      const r = new TextDecoder("utf-8").decode(n),
            [i, ...s] = r.split(d),
            a = s.length ? parseInt(i, 10) : null,
            p = s.length ? s.join(d) : i,
            u = a != null && !Number.isNaN(a) ? a : null;
      return p.trim().length === 0 || o({level: "info", log: p, line: u}), n.length
    }
  }),

  t.setStderr({batched: n => { o({level: "warn", log: n, line: null}) }})
};
```

### Что делает

**Python-часть:** переопределяет `print`, чтобы он **добавлял номер строки**
в начало вывода.

```python
import inspect
_print = print
def print(*args):
   _print(inspect.getframeinfo(inspect.currentframe().f_back).lineno, "|", *args)
```

`inspect.currentframe().f_back` — фрейм **вызывающего** (пользовательский код).
`.lineno` — номер строки.

**Результат:** `print("hello")` на строке 5 → `5 | hello`.

**JS-часть:** `setStdout({write})`
1. Декодировать байты в строку
2. Разбить по `" | "` — первый элемент номер строки, остальное — текст
3. `parseInt(i, 10)` — строка
4. Вернуть `n.length` — **сколько байт обработано!** (Pyodide требует)

**`p.trim().length === 0 || o(...)`** — если пустая строка после trim,
**не** отправлять (но всё равно вернуть `n.length`).

**`setStderr({batched})`** — stderr собирается и отправляется пачкой,
с `level: "warn"`, без номера строки.

### Слабости

**1. `w = "|"`, `d = " | "`.** Если пользователь напечатает `" | "`
в тексте — разбор сломается:

```python
print("a | b")
```
→ `5 | a | b` → `split(" | ")` = `["5", "a", "b"]` → `i = "5"`,
`s = ["a", "b"]` → `p = "a | b"`. **Работает!** Потому что `join(d)`
собирает обратно.

**Но:** если текст начинается с числа:
```python
print("5 | x")
```
→ `5 | 5 | x` → `split(" | ")` = `["5", "5", "x"]` → `i = "5"` (ok),
`s = ["5", "x"]` → `p = "5 | x"`. Работает.

**Проблема если номер строки не число:** не бывает, `f_back.lineno` всегда число.

**2. stderr без номера строки** — `line: null`.

**3. `setStdout` перехватывает всё** — включая вывод из импортов.

## 2.12 `L` — установка пакетов

```js
const L = async (t, e, o) => {
  const n = t.pyodide_py.code.find_imports(e).toJs(),
        r = await t.loadPackagesFromImports(e),
        i = new Set(r.map(({name: a}) => a)),
        s = n.filter(a => !i.has(a));

  if (s.length) {
    await t.loadPackage("micropip");
    const a = `
import micropip
${s.map(p => `
try:
  await micropip.install("${p}")
except:
  pass
`).join("\n")}
  `.trim();
    await t.runPythonAsync(a, {globals: o})
  }
};
```

### Три источника пакетов

**1. `find_imports(code)`** — статический анализ, находит все `import X`.

**2. `loadPackagesFromImports(code)`** — Pyodide сам загружает известные
пакеты из своего lock-файла (numpy, pandas, matplotlib и т.д.).

**3. `s = n.filter(a => !i.has(a))`** — то, что **не** загрузилось из lock —
в `micropip`.

### `micropip` — установка с PyPI

```python
import micropip
try:
  await micropip.install("requests")
except:
  pass
try:
  await micropip.install("beautifulsoup4")
except:
  pass
```

**`try/except: pass`** — если пакет не ставится, **молча** пропустить.
Пользователь увидит `ImportError` при запуске, но не узнает причину.

**`micropip.install`** грузит с `https://pypi.org` / `https://files.pythonhosted.org`.
Оба в CSP (`_` в apply-csp).

**Ограничение:** многие пакеты не имеют WASM-сборки под Pyodide.
`try/except` их тихо пропускает.

## 2.13 `U` — главная функция запуска

```js
const U = async t => {
  const e = performance.now(),
        o = t.includes("matplotlib");

  let n = null, r = null;

  l({type: c.ENVIRONMENT_STATUS, status: g.INITIALIZING});

  try {
    // 1. Pyodide
    n = await R();

    // 2. свежие глобалы
    r = n.globals.get("dict")();
    n.runPython('globals()["__name__"] = "__main__"', {globals: r});

    // 3. внедрить reformat_exception
    N(n, r);

    // 4. перехват print
    k(n, r, s => l({type: c.LOG, ...s}));

    // 5. статус
    l({type: c.ENVIRONMENT_STATUS, status: g.INSTALLING_PACKAGES});

    // 6. поставить пакеты
    await L(n, t, r);

    // 7. статус
    l({type: c.ENVIRONMENT_STATUS, status: g.RUNNING_CODE});

    // 8. matplotlib
    o && E(n, r);

    // 9. ЗАПУСК
    const i = await n.runPythonAsync(t, {globals: r});

    // 10. графики
    if (o) {
      const s = A(n, r);
      for (const a of s) l({type: c.OUTPUT, output: a})
    }

    // 11. результат выражения
    i !== void 0 && l({type: c.OUTPUT, output: i})

  } catch (i) {
    const s = O(r, i);
    l({type: c.ERROR, line: s.line, error: s})
  } finally {
    o && n && r && T(n, r);
    const i = performance.now();
    l({type: c.RUN_COMPLETE, duration: i - e});
    r?.destroy()
  }
};
```

### Пошагово

**`e = performance.now()`** — старт замера.

**`o = t.includes("matplotlib")`** — нужен ли matplotlib-режим.
**Строковая проверка**, не импорт. Если код содержит `"matplotlib"` в комментарии —
сработает.

**`n = await R()`** — Pyodide (из кэша `y`, если уже загружен).

**`r = n.globals.get("dict")()`** — **новый** Python-dict для глобалов.

**Почему новый каждый раз:** изоляция между запусками. Переменные
предыдущего запуска **не видны**.

**НО:** `y` (Pyodide) кэшируется. Значит **модули** остаются загруженными,
но **переменные** — нет.

**`globals()["__name__"] = "__main__"`** — чтобы `if __name__ == "__main__":`
работал.

**`N(n, r)`** — внедрить `reformat_exception`.

**`k(n, r, ...)`** — перехват `print`. Каждая строка вывода → событие `LOG`.

**`l({type: RUNNING_CODE})`** — сообщить хосту, что сейчас будет запуск.

**`o && E(n, r)`** — matplotlib-режим.

**`await n.runPythonAsync(t, {globals: r})`** — **ЗАПУСК КОДА**.

**`runPythonAsync`** — асинхронный, поддерживает `await` в Python.

**Возврат:** PyProxy результата **последнего выражения**. Если код
`1 + 1`, вернёт PyProxy на `2`.

**`i !== void 0 && l({type: OUTPUT, output: i})`** — если код вернул
значение (как в REPL), отправить как `OUTPUT`.

**`l({type: OUTPUT, output: i})`** — `i` это **PyProxy**, не строка!
Отправка через `postMessage` требует **structured clone**. PyProxy
не клонируется — **упадёт `DataCloneError`**.

**Это баг.** Скорее всего `.output` ожидается строкой, но передаётся
PyProxy. Возможно, хост сериализует сам.

**Проверим `_} `:**
```js
const _ = t => {
  if (t instanceof Error) return {line: null, name: t.name, message: t.message, stack: ...};
  return {line: null, name: null, message: String(t), stack: []}
};
```

**`String(t)`** — для PyProxy это `"<pyproxy at 0x...>"`. То есть
в сообщении об ошибке будет бесполезная строка.

### `catch` — обработка ошибки

```js
} catch (i) {
  const s = O(r, i);
  l({type: c.ERROR, line: s.line, error: s})
}
```

**`O(r, i)`** — разбор:
```js
const O = (t, e) => {
  if (!t) return _(e);              // нет глобалов → JS-ошибка
  try {
    return I(t)                      // Python-исключение
  } catch {
    return _(e)                      // не удалось → JS
  }
};
```

**`if (!t)`** — если `r` (глобалы) не созданы (Pyodide не загрузился) —
это JS-ошибка, `_()`.

**`I(t)`** — попытка разобрать как Python-исключение.

**Если `I` бросит** — значит `reformat_exception` не сработала (нет
`sys.last_traceback`) → JS-ошибка.

### `finally` — уборка

```js
} finally {
  o && n && r && T(n, r);           // закрыть графики
  const i = performance.now();
  l({type: c.RUN_COMPLETE, duration: i - e});
  r?.destroy()                       // освободить глобалы
}
```

**`T(n, r)`** — закрыть фигуры matplotlib.

**`RUN_COMPLETE`** — всегда, с длительностью.

**`r?.destroy()`** — освободить PyProxy глобалов. **Важно:** иначе
память WASM растёт.

**НО:** `n` (Pyodide) не destroy-ится — он в кэше `y`.

## 2.14 `self.onmessage`

```js
self.onmessage = async t => {
  const {data: e} = t;
  if (e && typeof e === "object" && "code" in e) {
    const {code: o} = e;
    await U(o)
  }
};
```

**Принимает `{code: "..."}`.**

**`await U(o)`** — последовательная обработка. Если два сообщения пришли
быстро — обрабатываются по очереди (JS однопоточный).

**`l({type: "ready"})`** — в конце файла: сигнал, что воркер загружен.

## 2.15 securitypolicyviolation

```js
self.addEventListener("securitypolicyviolation", t => {
  const e = {
    effectiveDirective: t.effectiveDirective,
    violatedDirective: t.violatedDirective,
    blockedURI: t.blockedURI,
    sourceFile: t.sourceFile
  };
  l({type: "security_policy_violation", payload: e})
});
```

**В воркере** ловится нарушение CSP. Отправляется в главный поток.

**Зачем:** если Python-код попытается загрузить что-то запрещённое
(например `micropip.install` с непроверенного домена) — хост узнает.

## 2.16 Слабые места python-worker

### 1. PyProxy в OUTPUT

```js
i !== void 0 && l({type: c.OUTPUT, output: i})
```

`i` — PyProxy. `postMessage` не клонирует PyProxy → `DataCloneError`.

**Если это происходит — `catch` поймает, и ошибка будет вида
«DataCloneError», но пользователь не увидит результат.**

### 2. `try/except: pass` при установке пакетов

```python
try:
  await micropip.install("requests")
except:
  pass
```

Любая ошибка установки — молча. Пользователь увидит `ImportError` позже
и не поймёт, что пакет не встал.

### 3. `m` может стать невалидным

```js
const m = `image_bytes_list_${Math.random().toString().split(".")[1]}`;
```

`Math.random()` = `1.234e-7` → `"1.234e-7"` → `split(".")[1]` = `"234e-7"`
→ `image_bytes_list_234e-7` — **невалидный Python-идентификатор**.

Вероятность ~1e-6, но если случится — `A()` и `T()` упадут с `SyntaxError`.

### 4. `t.includes("matplotlib")` — строковая проверка

```js
const o = t.includes("matplotlib");
```

Код `# я не использую matplotlib` → включит matplotlib-режим зря.

Код `import mpl` (алиас) → не включит.

### 5. Глобалы уничтожаются на каждом запуске

```js
r?.destroy()
```

Переменные **не сохраняются** между запусками. Пользователь не может
запустить код, потом дописать.

**Хотя `y` (Pyodide) кэшируется** — модули загружены, но переменные нет.

### 6. `await n.runPythonAsync(t, {globals: r})` — PyProxy утечка

**`i` не destroy-ится.** Если бросать PyProxy в `postMessage` — ошибка.
Если не бросать — утечка.

### 7. Стек только если > 1 строки

```js
stack: a.length > 1 ? a : []
```

Для одиночных ошибок стек теряется.

### 8. Ошибки JS в Python-коде

`_(t)` возвращает `String(t)` для не-Error. PyProxy → `"<pyproxy at 0x...>"`.

---

# ЧАСТЬ 3. `pyodide-CAuAWlEr.js`

## 3.1 Что это

**Загрузчик Pyodide**, собранный Vite. Содержит:

| Компонент | Что |
|---|---|
| `error-stack-parser` | парсер стектрейсов (~2 КБ) |
| `StackFrame` | класс фрейма |
| `loadPyodide` | основная функция |
| `MEMFS` / `NODEFS` / NativeFS | файловые системы |
| `X` | OPFS-монтирование |

## 3.2 `StackFrame` и парсер

```js
var S = ["isConstructor", "isEval", "isNative", "isToplevel"],
    F = ["columnNumber", "lineNumber"],
    O = ["fileName", "functionName", "source"],
    le = ["args"], ce = ["evalOrigin"],
    k = S.concat(F, O, le, ce);
```

**Класс `p` (StackFrame)** с геттерами/сеттерами для каждого поля.

**`M()` — ErrorStackParser:**
```js
{
  parse: (i) => {
    if (i.stack && i.stack.match(/^\s*at .*(\S+:\d+|\(native\))/m)) return this.parseV8OrIE(i);
    if (i.stack) return this.parseFFOrSafari(i);
    throw new Error("Cannot parse given Error object")
  },
  parseV8OrIE: ...,      // Chrome/Node формат
  parseFFOrSafari: ...   // Firefox/Safari формат
}
```

**Зачем в Pyodide:** разбирать стектрейсы JS-ошибок внутри Python-контекста.

## 3.3 Определение окружения

```js
var y = typeof process == "object" && typeof process.versions == "object"
        && typeof process.versions.node == "string" && !process.browser,   // Node?
    C = y && typeof module < "u" && ... && typeof __dirname < "u",        // Node CJS?
    fe = y && !C,                                                          // Node ESM?
    me = typeof Deno < "u",                                                // Deno?
    H = !y && !me,                                                         // браузер?
    pe = H && typeof window == "object" && typeof document == "object"
         && "sessionStorage" in window && typeof importScripts != "function",  // main thread?
    ye = H && typeof importScripts == "function" && typeof self == "object";   // worker?
```

**Пять флагов:**
- `y` — Node.js
- `C` — Node CommonJS
- `fe` — Node ESM
- `me` — Deno
- `H` — браузер
- `pe` — браузер, главный поток
- `ye` — браузер, Web Worker

**В нашем случае `ye = true`** — воркер.

## 3.4 Загрузка файлов

```js
async function J(e, t) {
  let {response: i, binary: a} = L(e, t);
  if (a) return a;                                  // Node: fs
  let o = await i;                                  // браузер: fetch
  if (!o.ok) throw new Error(`Failed to load '${e}': request failed.`);
  return new Uint8Array(await o.arrayBuffer())
}
```

**`Y` — браузерная версия `L`:**
```js
function Y(e, t) {
  let i = new URL(e, location);
  return {response: fetch(i, t ? {integrity: t} : {})}
}
```

**`integrity`** — если передан, `fetch` проверит SRI!

**В нашем случае** `l({indexURL: "https://cdn.jsdelivr.net/pyodide/v0.26.4/full"})`
— **без integrity**. SRI не проверяется.

## 3.5 `Q()` — определить indexURL

```js
async function Q() {
  if (C) return __dirname;
  let e;
  try { throw new Error } catch (a) { e = a }
  let t = de.parse(e)[0].fileName;
  if (y && !t.startsWith("file://") && (t = `file://${t}`), fe) { ... }
  let i = t.lastIndexOf(I);          // I = "/"
  if (i === -1) throw new Error("Could not extract indexURL path from pyodide module location");
  return t.slice(0, i)
}
```

**Что делает:** бросает `Error`, ловит его, парсит стектрейс и берёт
**имя файла** первого кадра. Из пути извлекает директорию.

**Зачем:** если `indexURL` не передан — определить по расположению
самого модуля `pyodide.js`.

**В нашем случае `indexURL` передан явно**, поэтому `Q()` не вызывается.

## 3.6 Файловая система

```js
if (pe) x = async e => await import(e);            // main thread
else if (ye) x = async e => {                      // worker
  try { globalThis.importScripts(e) }
  catch (t) { if (t instanceof TypeError) await import(e); else throw t }
};
else if (y) x = G;                                 // Node
else throw new Error("Cannot determine runtime environment")
```

**В воркере:** `importScripts` с fallback на `import()`.

## 3.7 OPFS-монтирование (`X`)

```js
function X(e) {
  let t = e.FS, i = e.FS.filesystems.MEMFS, a = e.PATH,
      o = {
        DIR_MODE: 16895,    // 040777
        FILE_MODE: 33279,   // 0100777
        mount: function (r) { ... },
        syncfs: async (r, n, s) => { ... },
        getLocalSet: ..., getRemoteSet: ..., loadLocalEntry: ...,
        storeLocalEntry: ..., removeLocalEntry: ...,
        loadRemoteEntry: ..., storeRemoteEntry: ..., removeRemoteEntry: ...,
        reconcile: async (r, n, s) => { ... }
      };
  e.FS.filesystems.NATIVEFS_ASYNC = o
}
```

**OPFS** = Origin Private File System. Позволяет Pyodide монтировать
**настоящую** файловую систему браузера (постоянную между сессиями).

**Режимы:** `040777` для директорий, `0100777` для файлов — права rwxrwxrwx.

**`reconcile`** — синхронизация: сравнить local и remote, обновить более новые.

**Используется ли:** только если Python-код явно монтирует через
`pyodide.mountNativeFS()`. Обычно нет.

## 3.8 Что важно знать

**1. Версия Pyodide ≠ версия в `python-worker`.** Здесь модуль — только
**загрузчик**. Реальная версия задаётся `indexURL` в `python-worker`.

**2. `integrity` поддерживается, но не используется.** В `python-worker`:
```js
await t({indexURL: "https://cdn.jsdelivr.net/pyodide/v0.26.4/full"})
```
Никакой проверки целостности.

**3. `loadPyodide` — из npm-пакета** `pyodide`, собранного Vite.

---

# ЧАСТЬ 4. Полная схема Python-запуска

```
Хост: runUserCode({language: "python", code})
  │
  ├─ applyCsp() → CSP
  ├─ yield RUN_START
  ├─ hn({isFirstParty: false}) → глобалы
  ├─ _t(import main.m, [1,2]) → {runPythonInWorker, terminatePythonWorker}
  ├─ p && terminatePythonWorker()    ← если CSP ужесточилась
  │
  └─ yield* ni((emit, done) => {
       runPythonInWorker(code, logger, msg => {
         emit(msg)
         msg.type === "run_complete" && done()
       })
     })
       │
       ├─ Lt ??= await js()        ← создание Worker (ОДИН раз)
       │
       └─ worker.postMessage({code})
            │
            ├─ [python-worker.js]
            ├─ ENVIRONMENT_STATUS INITIALIZING
            ├─ R() → loadPyodide({indexURL: cdn.jsdelivr.net/pyodide/v0.26.4/full})
            │    ├─ fetch pyodide.asm.js (~10 МБ)
            │    ├─ fetch pyodide.asm.wasm (~9 МБ)
            │    └─ fetch python_stdlib.zip (~1.5 МБ)
            ├─ globals = pyodide.globals.get("dict")()
            ├─ runPython('globals()["__name__"] = "__main__"')
            ├─ N() → внедрить reformat_exception
            ├─ k() → перехват print (setStdout/setStderr)
            ├─ ENVIRONMENT_STATUS INSTALLING_PACKAGES
            ├─ L() → find_imports + loadPackagesFromImports + micropip
            ├─ ENVIRONMENT_STATUS RUNNING_CODE
            ├─ code.includes("matplotlib") && E() → matplotlib.use("AGG")
            ├─ await pyodide.runPythonAsync(code, {globals})
            │    ├─ LOG (из print)
            │    ├─ OUTPUT (графики из A())
            │    └─ ERROR (из O() при исключении)
            └─ RUN_COMPLETE {duration}
```

---

# ЧАСТЬ 5. Сравнение: три способа запуска

| | `runReact` | `runThree` | Python |
|---|---|---|---|
| Где исполняется | iframe (главный поток) | iframe (главный поток) | **Web Worker** |
| DOM доступен | ✅ | ✅ | ❌ |
| `window.openai` | ✅ | ❌ | ❌ |
| События | INIT+INSTALL+RUNNING | INIT+RUNNING | INIT+INSTALL+RUNNING |
| Измерения | `measureHeight` | — | — |
| Загрузка | esbuild-wasm 0.24.2 | three 0.179.1 (unpkg) | Pyodide 0.26.4 |
| Источник | jsdelivr/unpkg | unpkg (importmap) | jsdelivr |
| Размер загрузки | ~1 МБ (esbuild.wasm) | ~700 КБ | **~20 МБ** |
| Состояние между запусками | — | — | модули да, переменные нет |
| Время первого запуска | ~1-2 с | ~0.5 с | **~5-15 с** |

---

# ЧАСТЬ 6. Итог

## 6.1 Ключевые цифры

| Что | Значение |
|---|---|
| Pyodide | 0.26.4 |
| Размер Pyodide | ~20 МБ |
| CDN Pyodide | cdn.jsdelivr.net |
| SRI для Pyodide | ❌ нет |
| `prepareEnvironment` для threejs | ❌ бесполезен |
| `prepareEnvironment` для python | ❌ отсутствует |
| Глобальный замок | `let a = !1` |
| Переменные между запусками | ❌ теряются |
| Модули между запусками | ✅ кэшируются |

## 6.2 Главные находки

**1. `prepareEnvironment` имеет глобальный замок — второй вызов теряется.**

```js
let a = !1;
const p = async (i, {language: s, ...}) => {
  if (!a) { a = !0; ... finally { a = !1 } }
};
```

Параллельный вызов с другим языком — отброшен молча.

**2. `prepareEnvironment` для threejs бесполезен** — грузит вшитые бандлы
(`three.module-C846Yxd8.js`), а `runThree` использует unpkg через importmap.

**3. `prepareEnvironment` для python отсутствует** — `default: return`.
Самый тяжёлый язык (~20 МБ) без прогрева.

**4. PyProxy в OUTPUT — баг.**

```js
i !== void 0 && l({type: c.OUTPUT, output: i})
```

`i` — PyProxy, не клонируется через `postMessage` → `DataCloneError`.

**5. `try/except: pass` при установке пакетов.**

Любая ошибка micropip — молча. Пользователь увидит `ImportError` позже
без объяснения.

**6. Случайное имя может стать невалидным.**

```js
const m = `image_bytes_list_${Math.random().toString().split(".")[1]}`;
```

`1.234e-7` → `"234e-7"` → невалидный Python-идентификатор.

**7. `matplotlib` определяется строкой.**

```js
const o = t.includes("matplotlib");
```

Комментарий с словом `matplotlib` включит режим. Алиасный импорт — нет.

**8. Переменные Python не сохраняются между запусками.**

```js
r = n.globals.get("dict")();   // новый dict
...
r?.destroy()                    // уничтожить после
```

Модули (Pyodide кэш `y`) — да, переменные — нет.

**9. `integrity` в Pyodide поддерживается, но не используется.**

```js
await t({indexURL: "https://cdn.jsdelivr.net/pyodide/v0.26.4/full"})
```

В загрузчике есть `fetch(i, t ? {integrity: t} : {})`, но `t` не передан.
