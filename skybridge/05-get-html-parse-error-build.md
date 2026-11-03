# Пакет `get-html` + `parse-transpile-error` + `build-user-code` — полный разбор

Три маленьких файла, которые вместе собирают HTML из транспилированного кода
и обрабатывают ошибки.

| Файл | Размер | Роль |
|---|---|---|
| `get-html-D9pLuxCy.js` | 502 байта | шаблон HTML-документа |
| `parse-transpile-error-CaCAx2Mb.js` | 846 байт | разбор ошибок esbuild |
| `build-user-code-BQgOUc-V.js` | 1 340 байт | сборка HTML из кода |

---

# ЧАСТЬ 1. `get-html-D9pLuxCy.js`

## 1.1 Полный код

```js
const STYLES = "<!-- EXTRA_STYLES -->",
      CODE   = "<!-- TRANSPILED_CODE -->",
      getHtml = ({transpiledCode: o, css: r}) => {
  const t = `
<head>
  <meta charset="UTF-8" />
  ${STYLES}
</head>
`.trim(),
  a = `
<!DOCTYPE html>
<html lang="en">
  ${t}
  <body>
    <div id="root"></div>
    <script type="module">${CODE}<\/script>
  </body>
</html>
`.trim(),
  n = `<style>
${r.join(`
`)}
</style>`,
  c = t.replace(STYLES, n),
  s = a.replace(t, c),
  i = s.split(CODE)[0].split(`
`).length + 1,
  [p, T] = s.split(CODE);
  return {html: p + o + T, size: o.length, sourceMapLineOffset: i};
};
export {getHtml as g};
```

## 1.2 Константы-маркеры

```js
STYLES = "<!-- EXTRA_STYLES -->"
CODE   = "<!-- TRANSPILED_CODE -->"
```

**Это HTML-комментарии как плейсхолдеры.** Почему комментарии, а не `{{STYLES}}`?

Потому что если заменить не удастся — в HTML останется **валидный комментарий**,
а не мусор вида `{{STYLES}}`. Браузер его проигнорирует, страница отрендерится.

## 1.3 Построчно

### Шаг 1 — `head`

```js
const t = `
<head>
  <meta charset="UTF-8" />
  <!-- EXTRA_STYLES -->
</head>
`.trim()
```

**Результат (после `.trim()`):**
```html
<head>
  <meta charset="UTF-8" />
  <!-- EXTRA_STYLES -->
</head>
```

**Только `charset`.** Никаких `<meta viewport>`, `<meta description>`,
`<title>`. Всё, что нужно, добавит сам код виджета или хост.

**Почему `.trim()`:** убирает ведущий/замыкающий перевод строки от
шаблонной строки. Без него `sourceMapLineOffset` был бы на 1 больше.

### Шаг 2 — полный документ

```js
const a = `
<!DOCTYPE html>
<html lang="en">
  ${t}
  <body>
    <div id="root"></div>
    <script type="module"><!-- TRANSPILED_CODE --><\/script>
  </body>
</html>
`.trim()
```

**`${t}`** — вставляет блок `<head>` целиком.

**`<div id="root">`** — точка монтирования React. `createRoot(document.getElementById("root"))`.

**`<script type="module">`** — **module**, а не обычный скрипт. Это важно:
- работает `import` (для 248 компонентов shadcn)
- код выполняется в строгом режиме
- переменные не попадают в глобальную область

**`<\/script>`** — экранирование `/`. Внутри JS-строки это даёт `</script>`.
Экранирование нужно, чтобы **парсер JS не закрыл шаблонную строку раньше
времени**. Если написать `</script>` напрямую в JS-файле — браузер, встретив
эту последовательность в `<script src>`, может её интерпретировать как конец
тега (в старых движках).

**Результат:**
```html
<!DOCTYPE html>
<html lang="en">
  <head>
  <meta charset="UTF-8" />
  <!-- EXTRA_STYLES -->
</head>
  <body>
    <div id="root"></div>
    <script type="module"><!-- TRANSPILED_CODE --></script>
  </body>
</html>
```

Обрати внимание: отступы `<head>` съехали — `t` уже был `.trim()`-нут,
поэтому `${t}` вставился без ведущего перевода строки. Это **косметика**,
на рендер не влияет.

### Шаг 3 — тег стилей

```js
const n = `<style>
${r.join(`
`)}
</style>`
```

**`css`** — массив строк. Каждый элемент — отдельный CSS-блок от транспиляции
(стили компонентов shadcn, стили пользователя).

**`r.join("\n")`** — склеивает через перевод строки.

**Если `css` пустой:** `[].join("\n")` → `""` → получится
```html
<style>

</style>
```
Пустой, но валидный `<style>`.

**Если `css` не массив** (например `undefined`): `r.join` → `TypeError`.
Если бы код вызывался с `css: undefined`, сборка упала бы.

### Шаг 4 — подстановка стилей в head

```js
const c = t.replace(STYLES, n)
```

`t` = `<head>...<!-- EXTRA_STYLES -->...</head>`
`n` = `<style>...</style>`
`c` = `<head>...<style>...</style>...</head>`

**`String.replace` с подстрокой** (не regex) заменяет **первое вхождение**.
Маркер один — заменится он.

**Опасность:** если CSS содержит строку `<!-- EXTRA_STYLES -->`, она не
пострадает, потому что `replace` работает по **исходному** `t`, а не по
результату.

### Шаг 5 — подстановка head в документ

```js
const s = a.replace(t, c)
```

**Важно:** `a.replace(t, c)` — заменяет **`t` на `c`**. То есть находит
в `a` строку `<head>...<!-- EXTRA_STYLES -->...</head>` и меняет её на
`<head>...<style>...</style>...</head>`.

**Проблема `String.replace` с `$`:** если `c` содержит `$&`, `$1`, `$'` и т.п.,
они будут интерпретированы как спецсимволы замены.

Например, CSS с `content: "$&"` → подставится весь найденный `t`.
**Это известная ловушка `String.replace`.** В нашем случае CSS генерируется
транспайлером и редко содержит `$`, но теоретически возможно.

Правильно было бы: `a.replace(t, () => c)` — функция не интерпретирует `$`.

### Шаг 6 — вычисление offset для sourcemap

```js
const i = s.split(CODE)[0].split("\n").length + 1
```

**Что делает по частям:**

1. `s.split(CODE)` — разбить по маркеру `<!-- TRANSPILED_CODE -->`
   → `[часть_до, часть_после]`
2. `[0]` — взять часть **до** маркера
3. `.split("\n").length` — посчитать строки
4. `+1` — потому что маркер на следующей строке

**Пример.** Допустим `s`:
```
1: <!DOCTYPE html>
2: <html lang="en">
3:   <head>
4:   <meta charset="UTF-8" />
5:   <style>
6: .btn{color:red}
7: </style>
8: </head>
9:   <body>
10:     <div id="root"></div>
11:     <script type="module"><!-- TRANSPILED_CODE --></script>
```

`s.split(CODE)[0]` = строки 1–11 до маркера.
От `DOCTYPE` до `<script type="module">` = 11 элементов в `split("\n")`.
`11 + 1 = 12`.

**Значит:** первой строке транспилированного кода соответствует **строка 12**
в итоговом документе.

**Зачем:** если в рантайме произойдёт ошибка на строке 5 пользовательского
кода, в HTML это будет строка `12 + 5 - 1 = 16`. Транспайлер возвращает
номера строк относительно **своего** кода, а хост хочет — относительно HTML.

### Шаг 7 — разделение и сборка

```js
const [p, T] = s.split(CODE)
return {html: p + o + T, size: o.length, sourceMapLineOffset: i}
```

**`s.split(CODE)`** даёт два элемента:
- `p` — всё до маркера (HTML-шапка, включая `<script type="module">`)
- `T` — всё после маркера (`</script></body></html>`)

**Почему разделили:** чтобы **не** делать `s.replace(CODE, o)`. Причина та же —
`o` может содержать `$`-последовательности или сам маркер.

**`p + o + T`** — конкатенация. Никаких спецсимволов.

**Возврат:**
```js
{
  html: "<!DOCTYPE html>...<script type='module'>КОД</script>...",
  size: длина кода в символах,
  sourceMapLineOffset: 12
}
```

## 1.4 Что получается на выходе — полный пример

**Вход:**
```js
{
  transpiledCode: `import{useState}from"react";import{jsx}from"react/jsx-runtime";
function App(){const[n,s]=useState(0);return jsx("button",{onClick:()=>s(n+1),children:n})}
export default App;`,
  css: [".btn{padding:12px 24px;border-radius:8px}"]
}
```

**Выход:**
```js
{
  html: `<!DOCTYPE html>
<html lang="en">
  <head>
  <meta charset="UTF-8" />
<style>
.btn{padding:12px 24px;border-radius:8px}
</style>
</head>
  <body>
    <div id="root"></div>
    <script type="module">import{useState}from"react";import{jsx}from"react/jsx-runtime";
function App(){const[n,s]=useState(0);return jsx("button",{onClick:()=>s(n+1),children:n})}
export default App;</script>
  </body>
</html>`,
  size: 234,
  sourceMapLineOffset: 12
}
```

## 1.5 Слабые места `get-html`

### 1. Маркеры — произвольные строки

Если **транспилированный код** содержит `<!-- TRANSPILED_CODE -->`
(например, пользователь написал это в строке):
```js
const s = "<!-- TRANSPILED_CODE -->";
```
Тогда `s.split(CODE)` даст **три** элемента, а `[p, T] = ...` возьмёт только
первые два. **Часть кода потеряется.**

**Практически:** маловероятно, но возможно. Пользователь может писать HTML
в виджете, и такой комментарий вполне естественен.

### 2. `String.replace` с `$` в CSS

```js
const c = t.replace(STYLES, n)
const s = a.replace(t, c)
```

`n` (CSS) подставляется как **строка замены**, а не как функция. Если CSS
содержит `$&`, `$1`, `$'`, `$\`` — они интерпретируются.

**Что могло бы сломать:**
```css
.x::after { content: "$&"; }
```
→ подставится весь `<head>` вместо `$&`.

**Правильно:** `t.replace(STYLES, () => n)`.

### 3. Пустой `css` даёт пустой `<style>`

Мелочь, но засоряет DOM.

## 1.6 Итог по `get-html`

**Что делает:** 48 строк, собирает HTML-документ из трёх частей.

**Что важно:** маркеры — комментарии, а не `{{}}`. Это правильно: если
подстановка не удастся, HTML останется валидным.

**Что слабо:** `String.replace` с потенциальными `$`-последовательностями,
и маркеры не защищены от коллизии с кодом пользователя.

---

# ЧАСТЬ 2. `parse-transpile-error-CaCAx2Mb.js`

## 2.1 Полный код

```js
import {U as o, L as u, R as E} from "./transpile-Cld0n4VL.js";
import "./main-BR1bOQyH.js";

const p = t => {
  const e = new RegExp(`internal:${o}:(\\d+)`),
        n = t.match(e),
        r = n?.at(1);
  return r ? parseInt(r) : null;
};

const R = t => {
  const e = t.join("\n"),
        n = e.includes("ERROR: Unexpected") ||
            e.includes("ERROR: Unterminated") ||
            e.includes("ERROR: Expected"),
        r = e.includes(`ERROR: [plugin: ${u}]`),
        s = e.includes(`ERROR: [plugin: ${E}]`);
  return n ? "syntax" : r ? "package_load" : s ? "package_resolve" : "unknown";
};

function _(t, e) {
  if (!(e instanceof Error))
    return t.count("react_transpile_error", 1, [{key: "type", value: "unknown"}]), null;

  const {name: n, message: r} = e,
        [s, ...i] = r.split("\n"),
        a = i.filter(l => l.trim().startsWith(`internal:${o}`));
  t.count("react_transpile_error", 1, [{key: "type", value: R(a)}]);

  const c = p(r);
  return {name: n, line: c, message: s, stack: a};
}

export {_ as parseAndLogTranspileError};
```

## 2.2 Импорты — что за `U`, `L`, `R`

```js
import {U as o, L as u, R as E} from "./transpile-Cld0n4VL.js";
```

Из `transpile` экспортируются три строки:

```js
Bm  = "user-component.js"      // экспорт как U
are = "onLoad-plugin"          // экспорт как L
tre = "onResolve-plugin"       // экспорт как R
```

**Откуда они:**
```js
tre = "onResolve-plugin",
are = "onLoad-plugin",
mJe = () => ({
  name: tre,
  setup(x) {
    x.onResolve({filter: /^index\.js$/}, ({path: D}) => ({path: D, namespace: "internal"}));
    x.onResolve({filter: /^\.\/user-component\.js$/}, () => ({path: Bm, namespace: ...}));
    ...
  }
})
```

**То есть:** это **имена esbuild-плагинов**, которые транспилер регистрирует
для подмены импортов. Когда esbuild падает внутри такого плагина, в сообщении
появляется `ERROR: [plugin: onResolve-plugin]`.

**Зачем парсеру знать имена:** по ним он классифицирует ошибку — это ошибка
синтаксиса, ошибка загрузки пакета, или ошибка разрешения импорта.

## 2.3 Функция `p(t)` — извлечь номер строки

```js
const p = t => {
  const e = new RegExp(`internal:${o}:(\\d+)`),   // /internal:user-component\.js:(\d+)/
        n = t.match(e),
        r = n?.at(1);
  return r ? parseInt(r) : null;
};
```

**`o`** = `"user-component.js"` → regex `internal:user-component.js:(\d+)`

**Что ищет:** в тексте ошибки строку вида `internal:user-component.js:5:12`

**`n?.at(1)`** — первая группа захвата (цифры после двоеточия).

**`parseInt(r)`** — в число.

**Если не найдено:** `null`.

**Важная деталь:** `new RegExp` создаётся **при каждом вызове** — не кэшируется.
Мелкая неэффективность, но не критично (функция вызывается редко).

**`.at(1)`** — метод массива. `n.at(1)` = `n[1]`. Работает с отрицательными
индексами (`at(-1)` — последний), но здесь обычный.

## 2.4 Функция `R(t)` — классификация ошибки

```js
const R = t => {
  const e = t.join("\n"),
        n = e.includes("ERROR: Unexpected") ||
            e.includes("ERROR: Unterminated") ||
            e.includes("ERROR: Expected"),
        r = e.includes(`ERROR: [plugin: ${u}]`),      // onLoad-plugin
        s = e.includes(`ERROR: [plugin: ${E}]`);      // onResolve-plugin
  return n ? "syntax" : r ? "package_load" : s ? "package_resolve" : "unknown";
};
```

**Три категории:**

| Условие | Тип |
|---|---|
| `ERROR: Unexpected` / `Unterminated` / `Expected` | `"syntax"` |
| `ERROR: [plugin: onLoad-plugin]` | `"package_load"` |
| `ERROR: [plugin: onResolve-plugin]` | `"package_resolve"` |
| иначе | `"unknown"` |

**Порядок проверки важен:** `syntax` приоритетнее. Если в сообщении есть и
синтаксическая ошибка, и ошибка плагина — вернётся `"syntax"`.

**Что реально ловит:**
- `ERROR: Unexpected end of file` — незакрытая скобка
- `ERROR: Unterminated string literal` — незакрытая кавычка
- `ERROR: Expected "}" but found ")"` — синтаксис
- `ERROR: [plugin: onLoad-plugin] ...` — не удалось загрузить пакет из CDN
- `ERROR: [plugin: onResolve-plugin] ...` — не удалось разрешить импорт

## 2.5 Функция `_()` — главная

```js
function _(t, e) {
  // t — SandboxLogger
  // e — ошибка

  // 1. не Error → метрика + null
  if (!(e instanceof Error))
    return t.count("react_transpile_error", 1, [{key: "type", value: "unknown"}]), null;

  const {name: n, message: r} = e,

        // 2. первая строка — краткое сообщение, остальные — стек
        [s, ...i] = r.split("\n"),

        // 3. оставить только строки, относящиеся к user-component.js
        a = i.filter(l => l.trim().startsWith(`internal:${o}`));

  // 4. метрика с классификацией
  t.count("react_transpile_error", 1, [{key: "type", value: R(a)}]);

  // 5. номер строки
  const c = p(r);

  return {name: n, line: c, message: s, stack: a};
}
```

### Пошагово

**Шаг 1 — не Error:**
```js
if (!(e instanceof Error))
  return t.count("react_transpile_error", 1, [{key: "type", value: "unknown"}]), null;
```
Comma-оператор: сначала `count`, потом `return null`.

**Метрика:** `count("react_transpile_error", 1, [{key:"type", value:"unknown"}])`
→ `sendInstrument({type: "count", label: "react_transpile_error", tags: [...], count: 1})`.

**Шаг 2 — разделение сообщения:**
```js
const [s, ...i] = r.split("\n")
```
`s` — первая строка (тип ошибки), `i` — остальные (стек).

**Пример сообщения esbuild:**
```
ERROR: Expected "}" but found ")"
    internal:user-component.js:3:15

  1 | function App() {
  2 |   return <div>
  3 |     {count
    |            ^
  4 |   </div>
  5 | }
    at /assets/transpile.js:1234:56
```

`s` = `"ERROR: Expected \"}\" but found \")\""`
`i` = `["    internal:user-component.js:3:15", "", "  1 | function App() {", ...]`

**Шаг 3 — фильтр стека:**
```js
a = i.filter(l => l.trim().startsWith(`internal:${o}`))
```
Оставляет только строки, начинающиеся с `internal:user-component.js`.

**Результат для примера:**
```js
a = ["    internal:user-component.js:3:15"]
```

**Что отсекается:** строки с кодом (`1 | function App()`), указатели (`^`),
пути к внутренним файлам транспайлера (`at /assets/transpile.js:...`).

**Шаг 4 — метрика:**
```js
t.count("react_transpile_error", 1, [{key: "type", value: R(a)}])
```
Классифицирует по отфильтрованному стеку.

**Проблема:** `R(a)` ищет `ERROR: ...` в стеке, но после фильтрации остались
только строки `internal:user-component.js:3:15` — **без** `ERROR: ...`.

**То есть классификация почти всегда вернёт `"unknown"`**, если ошибка не
попала в первую строку `s` (которая в фильтр не попала).

**Это баг.** Логичнее было бы `R(r.split("\n"))` — по всему сообщению.

**Шаг 5 — номер строки:**
```js
const c = p(r)
```
Ищет `internal:user-component.js:(\d+)` во **всём** сообщении `r`.
Возвращает `3` для примера.

**Шаг 6 — возврат:**
```js
return {name: n, line: c, message: s, stack: a}
```
```js
{
  name: "Error",
  line: 3,
  message: 'ERROR: Expected "}" but found ")"',
  stack: ["    internal:user-component.js:3:15"]
}
```

## 2.6 Что происходит при успехе

`parseAndLogTranspileError` возвращает объект при **ошибке**. При успехе
(в `validate()` из main.js):
```js
async validate(t, e) {
  try {
    await validateReact(t, e);
    return true;
  } catch (r) {
    return parseAndLogTranspileError(t, r) ?? false;
  }
}
```
Успех → `true`, ошибка → объект (truthy). То есть **всегда truthy** при
наличии объекта.

**`?? false`** — если парсер вернул `null` (не Error), тогда `false`.

## 2.7 Слабые места `parse-transpile-error`

### 1. Классификация почти всегда `"unknown"`

```js
a = i.filter(l => l.trim().startsWith(`internal:${o}`))
...
t.count("react_transpile_error", 1, [{key: "type", value: R(a)}])
```

`R(a)` ищет `ERROR: Unexpected` в **отфильтрованном** стеке, где остались
только пути вида `internal:user-component.js:3:15`. `ERROR:` там нет.

**Результат:** метрика `type: "unknown"` в большинстве случаев.

### 2. `new RegExp` на каждом вызове

```js
const p = t => {
  const e = new RegExp(`internal:${o}:(\\d+)`), ...
```
Regex создаётся заново. Мог бы быть `const RE = new RegExp(...)` на уровне модуля.

### 3. Точка в имени файла не экранирована

```js
`internal:${o}:(\\d+)`     // o = "user-component.js"
```
→ `internal:user-component.js:(\d+)`

`.` в regex = **любой символ**. То есть подойдёт и `user-componentXjs`.
Не критично, но неточно.

### 4. `parseInt` без radix

```js
parseInt(r)
```
`parseInt("08")` → `8` (в современном JS ok). Но лучше `parseInt(r, 10)`.

---

# ЧАСТЬ 3. `build-user-code-BQgOUc-V.js`

## 3.1 Полный код

```js
const __vite__mapDeps = (i, m = __vite__mapDeps, d = (m.f || (m.f = [
  "assets/transpile-Cld0n4VL.js",              // [0]
  "assets/main-BR1bOQyH.js",                   // [1]
  "assets/main-foMHhJws.css",                  // [2]
  "assets/run-recharts-CQOvWM4G.js",           // [3]
  "assets/run-react-vAjeCFay.js",              // [4]
  "assets/get-html-D9pLuxCy.js",               // [5]
  "assets/parse-transpile-error-CaCAx2Mb.js"   // [6]
]))) => i.map(i => d[i]);

import {_ as o} from "./main-BR1bOQyH.js";
import {g as u} from "./get-html-D9pLuxCy.js";

async function d(t, {code: n, language: p, id: _}) {
  switch (p) {
    case "react": {
      const {transpileReact: e} = await o(
        async () => {
          const {transpileReact: i} = await import("./transpile-Cld0n4VL.js").then(c => c.a);
          return {transpileReact: i};
        },
        __vite__mapDeps([0, 1, 2])
      );
      const {transpiledCode: r, css: a} = await e({
        code: n, id: _, sandboxLogger: t, worker: true, minify: true
      });
      const {html: s} = u({transpiledCode: r, css: a});
      return s;
    }
    case "recharts": {
      const [{transpileReact: e}, {prepareRechartsCode: r}] = await Promise.all([
        o(() => import("./transpile-Cld0n4VL.js").then(c => c.a), __vite__mapDeps([0, 1, 2])),
        o(() => import("./run-recharts-CQOvWM4G.js"), __vite__mapDeps([3, 1, 2, 4, 0, 5, 6]))
      ]);
      const {transpiledCode: a, css: s} = await e({
        code: r(n), id: _, sandboxLogger: t, worker: true, minify: true,
        disableShadcnImports: true
      });
      const {html: i} = u({transpiledCode: a, css: s});
      return i;
    }
    case "python":
      break;
    case "javascript":
    case "html":
    case "threejs":
    case "typescript": {
      t.error("Unsupported language for buildUserCode", {language: p});
      break;
    }
  }
  return null;
}

export {d as buildUserCode};
```

## 3.2 Что делает `buildUserCode`

**Отличие от `runUserCode`:**

| | `runUserCode` | `buildUserCode` |
|---|---|---|
| Что делает | запускает код | собирает **HTML-строку** |
| Возврат | генератор событий | строка HTML или `null` |
| Python | запускает | — |
| HTML | запускает | — |
| React | запускает | **только собирает** |

**Зачем нужен:** для **превью** без запуска, для кэширования, для проверки
перед показом.

## 3.3 Ветка `react`

```js
const {transpileReact: e} = await o(
  async () => {
    const {transpileReact: i} = await import("./transpile-Cld0n4VL.js").then(c => c.a);
    return {transpileReact: i};
  },
  __vite__mapDeps([0, 1, 2])
);

const {transpiledCode: r, css: a} = await e({
  code: n, id: _, sandboxLogger: t,
  worker: true, minify: true
});

const {html: s} = u({transpiledCode: r, css: a});
return s;
```

**Пошагово:**

1. **Preload 3 файла:** транспилер, main, css
2. **`import("./transpile-Cld0n4VL.js").then(c => c.a)`** — экспорт `a` = `transpileReact`
3. **`transpileReact({worker: true, minify: true})`**:
   - `worker: true` — в Web Worker
   - **`minify: true`** — минифицировать! (в `runUserCode` минификации нет)
   - Возврат: `{transpiledCode, css}`
4. **`getHtml({transpiledCode, css})`** — собрать документ
5. **Вернуть `html`**

**`minify: true` — отличие от `runUserCode`.** В `runUserCode` → `runReact`
передаётся только `worker: true`, минификации нет. Здесь — есть.

**Почему:** `buildUserCode` готовит код для превью/кэша, размер важен.
`runUserCode` — для интерактивного запуска, где нужны точные номера строк
для отладки.

## 3.4 Ветка `recharts`

```js
const [{transpileReact: e}, {prepareRechartsCode: r}] = await Promise.all([
  o(() => import("./transpile-Cld0n4VL.js").then(c => c.a), __vite__mapDeps([0, 1, 2])),
  o(() => import("./run-recharts-CQOvWM4G.js"), __vite__mapDeps([3, 1, 2, 4, 0, 5, 6]))
]);
```

**`Promise.all`** — грузит два модуля параллельно:
- `transpileReact` из транспайлера (preload 3)
- `prepareRechartsCode` из речартс-раннера (preload 7)

**Зачем `prepareRechartsCode`:** подготовить код — заменить импорты Recharts
на глобалы (чтобы не тянуть библиотеку из сети).

**Что делает `prepareRechartsCode(code)` (из run-recharts):**
```js
import { LineChart, Line } from "recharts";
```
→
```js
const { LineChart, Line } = window.__recharts;
```

Или что-то похожее — точная реализация в `run-recharts-CQOvWM4G.js`.

**Дальше:**
```js
const {transpiledCode: a, css: s} = await e({
  code: r(n),                      // ← ПОДГОТОВЛЕННЫЙ код
  id: _, sandboxLogger: t,
  worker: true, minify: true,
  disableShadcnImports: true       // ← НОВОЕ
});
```

**`disableShadcnImports: true`** — отключить подстановку 248 shadcn-компонентов.

**Почему:** Recharts-виджеты не должны иметь доступ к UI-библиотеке
(Button, Dialog, Form и т.д.). Это лишний вес и лишняя поверхность.

**Если пользователь напишет `import {Button} from "@/components/ui/button"`
в recharts-виджете** — при `disableShadcnImports: true` импорт не разрешится
и транспиляция упадёт.

## 3.5 Ветка `python`

```js
case "python":
  break;
```

**Молча ничего.** Потом сработает `return null` в конце.

**То есть:** `buildUserCode({language: "python"})` → `null`. Без ошибки, без
предупреждения.

## 3.6 Ветки `javascript` / `html` / `threejs` / `typescript`

```js
case "javascript":
case "html":
case "threejs":
case "typescript": {
  t.error("Unsupported language for buildUserCode", {language: p});
  break;
}
```

**Здесь ошибка логируется:** `t.error("Unsupported language for buildUserCode", {language})`.

`SandboxLogger.error()` → `sendInstrument({type: "error", ...})` → хост получит.

**Но всё равно `return null`.** Никакого исключения.

**Асимметрия с python:** Python — молча, остальные — с ошибкой. Странно.

## 3.7 Что возвращает функция

| Язык | Возврат |
|---|---|
| `react` | HTML-строка |
| `recharts` | HTML-строка |
| `python` | `null` (молча) |
| `javascript` | `null` + ошибка в лог |
| `html` | `null` + ошибка в лог |
| `threejs` | `null` + ошибка в лог |
| `typescript` | `null` + ошибка в лог |
| неизвестный | `null` (молча — нет `default`) |

## 3.8 Где используется

В фасаде `main.js`:

```js
async buildUserCode(t) {
  const e = new rt;                       // SandboxLogger
  try {
    const {buildUserCode: n} = await _t(
      async () => {
        const {buildUserCode: r} = await import("./build-user-code-BQgOUc-V.js");
        return {buildUserCode: r};
      },
      __vite__mapDeps([2, 3])              // css, transpile
    );
    return await n(e, t);
  } catch (n) { e.environmentError(n); }
  return null;
}
```

То есть хост может вызвать:
```js
await wt.buildUserCode({code: "...", language: "react", id: "run-1"})
// → "<!DOCTYPE html>..."
```

## 3.9 Слабые места `build-user-code`

### 1. `minify: true` без возможности отключить

```js
await e({code: n, id: _, sandboxLogger: t, worker: true, minify: true});
```

`minify` захардкожен. Если нужны точные номера строк (для отладки) — нельзя
получить. В `runUserCode` минификации нет, там отладка работает.

### 2. Python молчит, остальные логируют

```
python      → break (тихо)
javascript  → error + break
```

Непоследовательно.

### 3. `prepareRechartsCode(n)` до транспиляции

```js
code: r(n)
```

Функция `prepareRechartsCode` модифицирует исходный код **до** esbuild.
Если она ошибётся — упадёт вся транспиляция, но ошибка будет «syntax»,
а не «recharts».

### 4. Нет кэширования

Каждый вызов `buildUserCode` с тем же кодом транспилирует заново.
При повторных превью — лишняя работа.

---

# ЧАСТЬ 4. Как три файла работают вместе

## 4.1 Цепочка react

```
buildUserCode(logger, {code, language: "react", id})
  │
  ├─ _t(import transpile, [0,1,2])
  │   ├─ preload: transpile, main, css
  │   └─ import() → transpileReact
  │
  ├─ transpileReact({code, id, worker: true, minify: true})
  │   ├─ esbuild-wasm в Worker
  │   ├─ подстановка 248 shadcn
  │   └─ → {transpiledCode, css}
  │
  ├─ getHtml({transpiledCode, css})
  │   ├─ собрать <head> с маркером
  │   ├─ вставить <style> из css
  │   ├─ вставить transpiledCode вместо маркера
  │   └─ → {html, size, sourceMapLineOffset}
  │
  └─ return html
```

## 4.2 Цепочка ошибки

```
transpileReact → throw Error
  │
  ├─ message: "ERROR: Expected \"}\" ...\n    internal:user-component.js:3:15\n..."
  │
validateReact → catch
  │
  └─ parseAndLogTranspileError(logger, error)
      ├─ e instanceof Error?  да
      ├─ [message, ...stack] = message.split("\n")
      ├─ stack.filter(=> internal:user-component.js)
      ├─ logger.count("react_transpile_error", 1, [{type: R(...)}])
      ├─ line = parseInt(regex match)
      └─ → {name, line, message, stack}
```

## 4.3 Полный пример ошибки

**Код пользователя:**
```jsx
function App() {
  return <div>
    {count
  </div>
}
```

**esbuild выдаёт:**
```
ERROR: Expected "}" but found ")"
    internal:user-component.js:3:15

  1 | function App() {
  2 |   return <div>
  3 |     {count
    |            ^
  4 |   </div>
  5 | }
    at /assets/transpile-Cld0n4VL.js:1234:56
```

**`parseAndLogTranspileError`:**
```js
line    = 3
message = 'ERROR: Expected "}" but found ")"'
stack   = ["    internal:user-component.js:3:15"]
```

**Что отдаётся хосту:** эти три поля. Хост показывает пользователю
«строка 3: Expected "}" but found ")"».

**Что теряется:** контекст кода (`{count`, указатель `^`). Только одна строка
стека.

---

# ЧАСТЬ 5. Итог

## 5.1 Таблица файлов

| Файл | Строк | Что делает | Слабость |
|---|---|---|---|
| `get-html` | 48 | собирает HTML из кода + css | `$` в `String.replace`, коллизия маркеров |
| `parse-error` | 27 | извлекает line/message/stack | классификация всегда `"unknown"` |
| `build-user-code` | 40 | диспетчер сборки | `minify` захардкожен, Python молчит |

## 5.2 Что важно понять

**1. `get-html` — единственное место, где решается структура документа.**

Всё, что видит браузер: `<!DOCTYPE html>` + `<head>` + `<div id="root">` +
`<script type="module">`. Ничего больше. Ни viewport, ни title.

**2. `parse-transpile-error` — только для отладки.**

Его данные идут в метрики и в ошибку пользователю. На безопасность
не влияет.

**3. `build-user-code` — только react и recharts.**

Остальные языки возвращают `null`. Для python/html/threejs сборка HTML
не нужна — они не используют шаблон.

**4. `minify: true` — только здесь.**

`runUserCode` → `runReact` без минификации (нужны точные строки).
`buildUserCode` — с минификацией (размер важнее).
