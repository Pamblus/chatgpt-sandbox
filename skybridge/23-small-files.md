# Мелочи: CSS, внешние заглушки, битые загрузки — полный разбор

Четыре маленьких файла, которые замыкают картину Skybridge.

| Файл | Размер | Что |
|---|---|---|
| `__vite-browser-external-9wXp6ZBx.js` | 31 Б | заглушка Vite для Node-модулей |
| `main-foMHhJws.css` | 510 Б | основной CSS |
| `theme.css` | 215 Б | **сломанная загрузка** (BlobNotFound) |
| `utilities.css` | 215 Б | **сломанная загрузка** (BlobNotFound) |

---

# ЧАСТЬ 1. `__vite-browser-external-9wXp6ZBx.js`

## 1.1 Полное содержимое

```js
var e = {};
export {e as default};
```

**Всё. 31 байт.**

## 1.2 Что это

**`__vite-browser-external`** — стандартный **виртуальный модуль Vite**,
который подставляется вместо встроенных модулей Node.js, когда код
собирается для браузера.

**Зачем:** Vite при сборке под браузер встречает `import fs from "fs"` —
и вместо ошибки подставляет пустой объект.

**Что появляется в браузере:**
```js
import fs from "fs";      // → var e = {}; export default e
fs.readFileSync("...")    // → TypeError: fs.readFileSync is not a function
```

## 1.3 Где используется в Skybridge

**Из `pyodide-CAuAWlEr.js`:**

```js
async function P() {
  if (!y || (W = (await import("./__vite-browser-external-9wXp6ZBx.js")).default,
             $ = await import("./__vite-browser-external-9wXp6ZBx.js"),
             T = await import("./__vite-browser-external-9wXp6ZBx.js"),
             q = (await import("./__vite-browser-external-9wXp6ZBx.js")).default,
             _ = await import("./__vite-browser-external-9wXp6ZBx.js"),
             I = _.sep,
             typeof j < "u")) return;

  let e = $,
      t = await import("./__vite-browser-external-9wXp6ZBx.js"),
      i = await import("./__vite-browser-external-9wXp6ZBx.js"),
      a = await import("./__vite-browser-external-9wXp6ZBx.js"),
      o = {fs: e, crypto: t, ws: i, child_process: a};

  globalThis.require = function (r) { return o[r] }
}
```

### Разбор

**Восемь импортов одного и того же файла** (все — `{}`).

**Переменные:**
- `W`, `q` — `path` (из `default`)
- `$`, `T`, `_` — `path` (модуль)
- `I = _.sep` → **`undefined`** (в `{}` нет `sep`)
- `e, t, i, a` — `fs`, `crypto`, `ws`, `child_process`

**Итог:**
```js
globalThis.require = function (r) {
  return {fs: {}, crypto: {}, ws: {}, child_process: {}}[r]
}
```

**`require("fs")` → `{}`.**

## 1.4 Зачем это нужно

**Pyodide** (точнее, его Vite-сборка) содержит код, который проверяет
наличие Node.js:

```js
var y = typeof process == "object" && process.versions?.node && !process.browser
```

**Если `y` — false (браузер)** — блок `P()` не должен выполниться:

```js
async function P() {
  if (!y || (...)) return;    // ← выход для браузера
  ...
}
```

**Но Vite всё равно пытается разрешить импорты** статически.

**Значит:** `__vite-browser-external` нужен только чтобы **сборка прошла**.
В рантайме **не используется** (блок `if (!y || ...) return`).

## 1.5 Проверка

**`y`** из `pyodide-CAuAWlEr.js`:
```js
var y = typeof process == "object"
     && typeof process.versions == "object"
     && typeof process.versions.node == "string"
     && !process.browser
```

**В браузере `process` нет** → `y = false`.

**В Web Worker'е `process` нет** → `y = false`.

**Значит блок `P()` всегда выходит на первой строке.**

**Вывод: `__vite-browser-external` в Skybridge никогда не выполняется.**

---

# ЧАСТЬ 2. `main-foMHhJws.css`

## 2.1 Полное содержимое

```css
html,body,#root{width:100%;height:100%;margin:0}
body{position:relative}
#root{position:absolute}
html{background-color:transparent}

@media (prefers-color-scheme: dark){
  html.chatgptOled{background-color:#000}
}
@media (prefers-color-scheme: dark){
  html.mobileSkybridge{background-color:#000}
}
@media (prefers-color-scheme: dark){
  html.desktopSkybridge{background-color:#1e1e1e}
}

html.transparentSkybridgeBackground{background-color:transparent}

iframe{all:unset;border:none;margin:0;display:block;width:100%;height:100%}
```

## 2.2 Разбор по правилам

### Базовый layout

```css
html,body,#root{width:100%;height:100%;margin:0}
body{position:relative}
#root{position:absolute}
```

**`body` — relative, `#root` — absolute.** Значит `#root` растягивается
на весь `body`.

**Зачем:** iframe внутри `#root` занимает 100%.

### Прозрачный фон по умолчанию

```css
html{background-color:transparent}
```

**Корень прозрачный.** Виджет может показать фон чата.

### Классы из `main.js` (документ #1)

**Пять классов на `<html>`, заданных из URL-параметров:**

| Класс | Условие | Фон (dark) |
|---|---|---|
| `chatgptOled` | `app=chatgpt&darkModeType=increased` | `#000` |
| `mobileSkybridge` | `app=skybridge&deviceType=mobile` | `#000` |
| `desktopSkybridge` | `app=skybridge` (не mobile) | `#1e1e1e` |
| `transparentSkybridgeBackground` | `app=skybridge&backgroundMode=transparent` | `transparent` |
| `androidSkybridge` | `app=skybridge&clientPlatform=android` | — (**не в CSS!**) |

**Важно:** `androidSkybridge` задаётся в `main.js`, но **стиля для него
НЕТ** в этом CSS.

**Плюс `chatgpt`** (без OLED) — тоже задаётся, но **нет стиля**.

### `iframe{all:unset}`

```css
iframe{
  all:unset;
  border:none;
  margin:0;
  display:block;
  width:100%;
  height:100%
}
```

**`all: unset`** — **сбросить ВСЕ свойства** до начальных или
унаследованных.

**Последствие:** iframe становится **`display: inline`** (initial),
без границы, без размера.

**Затем:**
- `border:none` — избыточно (all:unset уже сбросил)
- `margin:0` — тоже
- `display:block` — переопределить на block
- `width/height: 100%` — заполнить контейнер

**`all: unset` + явные значения** — паттерн «сбросить и задать заново».

**Риск:** `all: unset` сбрасывает и `sandbox`? Нет — `sandbox` атрибут,
не стиль.

## 2.3 Что НЕ покрыто

**Класс `chatgpt`** (без OLED) — нет фона в dark.

**Класс `androidSkybridge`** — нет стилей вовсе.

**`html.chatgpt` в светлой теме** — нет фона (остаётся `transparent`).

---

# ЧАСТЬ 3. `theme.css` и `utilities.css` — обе сломаны

## 3.1 Содержимое

```xml
<?xml version="1.0" encoding="utf-8"?>
<Error>
  <Code>BlobNotFound</Code>
  <Message>The specified blob does not exist.
    RequestId:91d79779-801e-00cc-7129-6be2b2000000
    Time:2026-11-03T00:21:46.6369492Z
  </Message>
</Error>
```

**Оба файла — XML-ошибка Azure Blob Storage.**

## 3.2 Что это значит

**`BlobNotFound`** — файл не существует в хранилище.

**Но URL отдал HTTP 200** с XML-телом ошибки.

**Откуда взялись:** это **не настоящие файлы Skybridge**. Это результат
**моей попытки скачать** `theme.css` / `utilities.css` из
`web-sandbox.oaiusercontent.com/assets/` — и сервер (Azure Blob)
ответил ошибкой.

**Проверить:** в `__vite__mapDeps` эти имена **не встречаются**.

**В `transpile-Cld0n4VL.js` есть виртуальные:**
```js
function _Ge(x, D) {
  if (x === "tailwindcss")
    return {path: "virtual:tailwindcss/index.css", base: D, content: kGe};
  if (x === "tailwindcss/preflight" || x === "tailwindcss/preflight.css" || x === "./preflight.css")
    return {path: "virtual:tailwindcss/preflight.css", base: D, content: PGe};
  if (x === "tailwindcss/theme" || x === "tailwindcss/theme.css" || x === "./theme.css")
    return {path: "virtual:tailwindcss/theme.css", base: D, content: IGe};
  if (x === "tailwindcss/utilities" || x === "tailwindcss/utilities.css" || x === "./utilities.css")
    return {path: "virtual:tailwindcss/utilities.css", base: D, content: NGe};
  throw new Error(`External stylesheets not supported in browser build: "${x}"`)
}
```

**`theme.css` и `utilities.css` — ВИРТУАЛЬНЫЕ.** Их содержимое — константы
`IGe` и `NGe` **внутри** `transpile-Cld0n4VL.js`.

**Не физические файлы.** Не с `oaiusercontent.com`.

## 3.3 Альтернативное объяснение

**Возможно, файлы существуют**, но:
- на другом пути
- или были удалены
- или Azure отдаёт ошибку по другой причине

**Проверить напрямую:**
```bash
curl -sI https://web-sandbox.oaiusercontent.com/assets/theme.css
```

**Если 404** — файлов нет, и это правильно.

**Если 200 с XML** — Azure отдаёт ошибку вместо 404 (типично для
статического хостинга Azure).

## 3.4 Вывод

**Эти два файла — артефакт скачивания, а не часть Skybridge.**

**Их содержимое (реальный CSS Tailwind v4) — внутри
`transpile-Cld0n4VL.js`:**
- `kGe` — `index.css` (полный Tailwind)
- `PGe` — `preflight.css` (reset)
- `IGe` — `theme.css` (переменные)
- `NGe` — `utilities.css` (`@tailwind utilities`)

---

# ЧАСТЬ 4. Полная картина CSS

## 4.1 Что реально грузится

| Файл | Откуда | Размер |
|---|---|---|
| `main-foMHhJws.css` | `/assets/` (реальный) | 510 Б |
| Tailwind CSS | генерируется WASM в `transpile` | динамически |
| shadcn CSS | входит в `transpile` (46 компонентов) | — |
| Темы (recharts) | inline в `run-recharts` (`c`) | 2.4 КБ |
| Документ (recharts) | inline (`a`) | 1.3 КБ |
| Mermaid тема | inline (`X()`) | ~3 КБ |

## 4.2 Порядок применения в `getHtml`

```js
Ke({transpiledCode: _, css: [...H.additionalCss ?? [], ...q]})
```

**1. `additionalCss`** — пользовательские (тема recharts + документ).
**2. `q`** — из транспиляции (shadcn + Tailwind).

**Значит:** пользовательский CSS **раньше**, его правила могут быть
переопределены стилями компонентов.

---

# ЧАСТЬ 5. Итог

## 5.1 Все четыре файла

| Файл | Реальный размер | Что |
|---|---|---|
| `__vite-browser-external` | **31 Б** | `var e={};export{e as default}` |
| `main-foMHhJws.css` | **510 Б** | layout + 4 темы фона + `iframe{all:unset}` |
| `theme.css` | **215 Б** | **XML-ошибка** Azure BlobNotFound |
| `utilities.css` | **215 Б** | **XML-ошибка** Azure BlobNotFound |

## 5.2 Главное

**1. `__vite-browser-external` — виртуальный модуль Vite.**
```js
var e = {}; export {e as default};
```
Подставляется вместо `fs`/`path`/`crypto` для браузерной сборки.

**2. В Skybridge он НИКОГДА не выполняется.**
`pyodide` проверяет `y` (Node.js) и выходит на первой строке блока `P()`.

**3. `main-foMHhJws.css` — 4 правила темы + layout.**
`html` прозрачный, классы из URL-параметров задают фон в тёмной теме.

**4. `androidSkybridge` и `chatgpt` — классы есть, стилей НЕТ.**
Ставятся в `main.js`, но в CSS не покрыты.

**5. `iframe{all:unset}` — сброс ВСЕХ свойств**, потом явные значения.
Избыточные `border:none`, `margin:0` (уже сброшены).

**6. `theme.css` и `utilities.css` — НЕ часть Skybridge.**
Это XML-ошибки `BlobNotFound` от Azure Blob.

**7. Настоящие `theme.css`/`utilities.css` — виртуальные**, внутри
`transpile-Cld0n4VL.js` (`_Ge`).

**8. Реальный порядок CSS в виджете:**
```
additionalCss (тема recharts + документ) → shadcn + Tailwind (сгенерированный)
```

**9. Tailwind генерируется на WASM** (1.64 МБ, документ #20), не
загружается файлом.

**10. Каталог `/assets/` в Azure Blob** отдаёт XML-ошибки вместо 404
для несуществующих файлов.
