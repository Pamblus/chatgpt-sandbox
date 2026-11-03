# `index-hL9okClV.js` — полный разбор (modern-screenshot)

**Размер:** 24 923 байта
**URL:** `https://web-sandbox.oaiusercontent.com/assets/index-hL9okClV.js`
**Роль:** библиотека **modern-screenshot** — конвертация DOM в SVG/Canvas/PNG.
Плюс патч DPI для PNG.

---

# ЧАСТЬ 1. Что это

## 1.1 Назначение

**`modern-screenshot`** — форк `html-to-image`. Превращает произвольный
DOM-элемент в изображение:

```
DOM element  →  <foreignObject>  →  SVG  →  <canvas>  →  dataURL
```

**Зачем в Skybridge:** скриншоты виджетов. Например:
- `screenshotVegaPreview` (параметр в `runWidgetCode`)
- предпросмотр графиков
- сохранение состояния виджета

## 1.2 Экспорты

```js
export {
  Ke as createContext,          // создать контекст
  ht as destroyContext,         // уничтожить контекст
  Dt as domToCanvas,            // DOM → Canvas
  _t as domToDataUrl,           // DOM → dataURL
  kt as domToForeignObjectSvg,  // DOM → SVG с foreignObject
  k as loadMedia,               // загрузить медиа
  Xe as waitUntilLoad           // дождаться загрузки
};
```

---

# ЧАСТЬ 2. Патч PNG-DPI (начало файла)

## 2.1 Зачем

**Стандарт PNG** хранит физический размер в блоке `pHYs`:
```
pixels per unit (X), pixels per unit (Y), unit (1 = метр)
```

Без него картинка рендерится в 72 DPI — на Retina выглядит мелкой.

## 2.2 CRC32

```js
function we() {
  const e = new Int32Array(256);
  for (let t = 0; t < 256; t++) {
    let r = t;
    for (let n = 0; n < 8; n++)
      r = r & 1 ? 3988292384 ^ r >>> 1 : r >>> 1;
    e[t] = r
  }
  return e
}

function pe(e) {
  let t = -1;
  $ || ($ = we());
  for (let r = 0; r < e.length; r++)
    t = $[(t ^ e[r]) & 255] ^ t >>> 8;
  return t ^ -1
}
```

**CRC32** — контрольная сумма для чанков PNG.

**`3988292384`** = `0xEDB88320` — стандартный полином CRC-32.

**`$`** — глобальный кэш таблицы.

## 2.3 `ye()` — поиск блока `pHYs`

```js
const J = 112,   // 'p'
      Q = 72,    // 'H'
      Z = 89,    // 'Y'
      ee = 115;  // 's'

function ye(e) {
  const t = e.length - 1;
  for (let r = t; r >= 4; r--)
    if (e[r-4] === 9 && e[r-3] === J && e[r-2] === Q
     && e[r-1] === Z && e[r] === ee)
      return r - 3;
  return 0
}
```

**Ищет байты `\x09pHYs`** (длина 9 + имя `pHYs`) с конца файла.

**Возвращает позицию `p`** или `0`.

## 2.4 `be()` — вставка/замена `pHYs`

```js
function be(e, t, r = !1) {
  const n = new Uint8Array(13);
  t *= 39.3701;                    // px → px/m (1 м = 39.3701 дюйм... точнее 1 дюйм = 0.0254 м)
  n[0] = J; n[1] = Q; n[2] = Z; n[3] = ee;   // 'pHYs'
  n[4] = t >>> 24; n[5] = t >>> 16; n[6] = t >>> 8; n[7] = t & 255;   // X ppm
  n[8] = n[4]; n[9] = n[5]; n[10] = n[6]; n[11] = n[7];                // Y ppm (то же)
  n[12] = 1;                       // единица: метр

  const i = pe(n),
        a = new Uint8Array(4);
  a[0] = i >>> 24; a[1] = i >>> 16; a[2] = i >>> 8; a[3] = i & 255;    // CRC

  if (r) {
    const s = ye(e);                // найти существующий pHYs
    return e.set(n, s), e.set(a, s + 13), e
  } else {
    const s = new Uint8Array(4);
    s[0] = 0; s[1] = 0; s[2] = 0; s[3] = 9;   // длина чанка = 9
    const o = new Uint8Array(54);
    return o.set(e, 0), o.set(s, 33), o.set(n, 37), o.set(a, 50), o
  }
}
```

### Разбор

**`t *= 39.3701`** — перевод в пиксели на метр.

**Что такое 39.3701:** 1 метр = 39.3701 дюйм. Значит если `t` — DPI
(точек на дюйм), то `t × 39.3701` = точек на метр.

**Проверить:** `72 DPI × 39.3701 = 2834.6` ppm. Стандарт PNG для 72 DPI —
`2835` ppm. **Верно.**

**Кодирование:** big-endian 4 байта для X, те же для Y, `1` (метр).

**Режим 1 (`r = true`)** — **замена существующего** `pHYs`:
```js
const s = ye(e);
e.set(n, s), e.set(a, s + 13);
```

**Режим 2 (`r = false`)** — **вставка нового**:
```js
o.set(e, 0);      // первые 33 байта PNG (сигнатура + IHDR)
o.set(s, 33);     // длина чанка (9)
o.set(n, 37);     // 'pHYs' + данные (13 байт)
o.set(a, 50);     // CRC (4 байта)
```

**Почему 33:** PNG-сигнатура (8 байт) + IHDR-чанк (25 байт: 4 длина +
4 тип + 13 данных + 4 CRC) = 33. `pHYs` вставляется **после IHDR**.

**`pHYs` не должен идти раньше IHDR** по спецификации.

## 2.5 `Se` / `Ee` / `Ce` / `ve()` — поиск sRGB/iCCP-профилей

```js
const Se = "AAlwSFlz",   // sRGB base64 (обрезанный)
      Ee = "AAAJcEhZ",   // pHYs base64 (обрезанный)
      Ce = "AAAACXBI";   // iCCP base64 (обрезанный)

function ve(e) {
  let t = e.indexOf(Se);
  t === -1 && (t = e.indexOf(Ee));
  t === -1 && (t = e.indexOf(Ce));
  return t
}
```

**Это base64-подстроки** — фрагменты PNG-чанков в dataURL.

**`AAlwSFlz`** = `\x00\x09sRGB` (в base64)
**`AAAJcEhZ`** = `\x00\x09pHYs`
**`AAAACXBI`** = `\x00\x00\x09iCCP`

**`ve()`** — находит любой из трёх, чтобы понять позицию вставки.

## 2.6 Патч в `_t()` (конец файла)

```js
async function _t(e, t) {
  const r = await q(e, t),
        {log: n, quality: i, type: a, dpi: s} = r,
        o = await Dt(r);                        // → Canvas

  n.time("canvas to data url");
  let c = o.toDataURL(a, i);

  if (["image/p", ...].includes(a) && s) {       // ← PNG + DPI задан
    const f = ve(c),                            // позиция профиля
          g = c.substring(f + 5, ...),          // ← base64-данные
          y = c.substring(f),                   // ← хвост
          m = window.atob(g),                   // ← декодировать
          w = new Uint8Array(m.length);

    for (let h = 0; h < w.length; h++) w[h] = m.charCodeAt(h);

    const p = a === "image/png" ? be(w, s, d) : he(w, s),
          b = window.btoa(String.fromCharCode(...p));

    c = [u, ",", b, y].join("")
  }

  return n.timeEnd("canvas to data url"), c
}
```

### Что происходит

1. `toDataURL("image/png")` → строка `data:image/png;base64,AAA...`
2. Найти позицию `pHYs`/`sRGB`/`iCCP`
3. Извлечь base64-часть, декодировать в байты
4. **Вставить/заменить `pHYs`** с правильным DPI
5. Кодировать обратно в base64
6. Склеить `data:image/png;base64,` + новые байты + хвост

**`String.fromCharCode(...p)`** — **spread большого массива**.

**Проблема:** при `w.length > ~65535` — `RangeError: Maximum call stack
size exceeded`. Для больших PNG (например, 4000×4000) — **упадёт**.

**Это реальный баг:** большие скриншоты ломаются.

**Правильно:** `String.fromCharCode.apply(null, chunk)` по частям, или
`btoa` на blob.

---

# ЧАСТЬ 3. Определение окружения

```js
const te = "[modern-screenshot]",
      A = typeof window < "u",                    // браузер?
      Ae = A && "Worker" in window,
      Te = A && "atob" in window,
      Ne = A && "btoa" in window,
      O = A ? window.navigator?.userAgent : "",
      re = O.includes("Chrome"),
      P = O.includes("AppleWebKit") && !re,       // Safari?
      B = O.includes("Firefox");
```

**Флаги:**
- `A` — браузер
- `Ae` — Worker доступен
- `Te` / `Ne` — atob/btoa
- `P` — Safari
- `B` — Firefox

**`P`** — AppleWebKit без Chrome = Safari (или iOS WebView).

**Где используются `P` и `B`:** в оптимизациях (drawImage count, fetch
через Worker).

---

# ЧАСТЬ 4. Валидаторы типов

```js
const Ie = e => e && "__CONTEXT__" in e,             // это Context?
      ke = e => e.constructor.name === "CSSFontFaceRule",
      Re = e => e.constructor.name === "CSSImportRule",
      v  = e => e.nodeType === 1,                    // Element
      D  = e => typeof e.className === "object",     // SVG (SVGAnimatedString)
      ne = e => e.tagName === "image",
      De = e => e.tagName === "use",
      I  = e => v(e) && typeof e.style < "u" && !D(e),
      _e = e => e.nodeType === 8,                    // Comment
      Pe = e => e.nodeType === 3,                    // Text
      N  = e => e.tagName === "IMG",
      F  = e => e.tagName === "VIDEO",
      Fe = e => e.tagName === "CANVAS",
      Ue = e => e.tagName === "TEXTAREA",
      xe = e => e.tagName === "INPUT",
      $e = e => e.tagName === "STYLE",
      Le = e => e.tagName === "SCRIPT",
      Me = e => e.tagName === "SELECT",
      Oe = e => e.tagName === "SLOT",
      Be = e => e.tagName === "IFRAME";
```

### Ключевое: `D()` — детектор SVG

```js
const D = e => typeof e.className === "object"
```

**В SVG `element.className` — это объект `SVGAnimatedString`**, а не строка.
Единственный надёжный способ отличить SVG от HTML.

**`I()`** — «это HTML-элемент»:
```js
v(e) && typeof e.style < "u" && !D(e)
```
Element + есть style + не SVG.

**`ne()` / `De()`** — `image`/`use` в нижнем регистре (SVG-теги).
**`N()` / `F()`** — `IMG`/`VIDEO` в верхнем (HTML).

## 4.1 `We()` — поддержка WebP

```js
function We(e) {
  const t = e?.createElement?.call(e, "canvas");
  return t && (t.height = t.width = 1),
    !!t && "toDataURL" in t && !!t.toDataURL("image/webp").includes("image/webp")
}
```

**Проверка:** создаёт canvas 1×1, пробует `toDataURL("image/webp")`,
проверяет, что результат содержит `image/webp`.

**В Safari** старых версий `toDataURL("image/webp")` вернёт PNG →
`includes` = `false`.

## 4.2 `M()` — dataURL?

```js
const M = e => e.startsWith("data:")
```

## 4.3 `oe()` — нормализация URL

```js
function oe(e, t) {
  if (e.match(/^[a-z]+:\/\//i)) return e;              // абсолютный
  if (A && e.match(/^\/\//)) return window.location.protocol + e;   // protocol-relative
  if (e.match(/^[a-z]+:/i) || !A) return e;            // data:, blob:, mailto:

  const r = U().implementation.createHTMLDocument(),
        n = r.createElement("base"),
        i = r.createElement("a");

  r.head.appendChild(n);
  r.body.appendChild(i);
  t && (n.href = t);                                    // база
  i.href = e;
  return i.href                                       // абсолютный URL
}
```

**Четыре случая:**
1. `https://...` — уже абсолютный
2. `//example.com` — добавить протокол
3. `data:`, `blob:`, `mailto:` — как есть
4. Относительный — резолвить через `<base>`

**`U().implementation.createHTMLDocument()`** — **изолированный документ**.
Не влияет на текущий.

**Зачем:** создать `<a>` и прочитать `.href` для резолва — без вставки
в живой DOM.

---

# ЧАСТЬ 5. `createContext` — `Ke()`

## 5.1 Сигнатура

```js
async function Ke(e, t) {
  const {
    scale: r = 1,
    workerUrl: n,
    workerNumber: i = 1,
    debug: a,
    features: s = true,
    ...
  } = t || {}

  const o = e.ownerDocument ?? window.document,
        c = e.ownerDocument?.defaultView ?? window;

  const u = new Map;     // requests
  const l = {
    width: 0,
    height: 0,
    quality: 1,
    type: "image/png",
    scale: r,
    backgroundColor: null,
    style: null,
    filter: null,
    maximumCanvasSize: 0,
    ...
  };
  ...
}
```

**Параметры:**
- `scale` — масштаб
- `workerUrl` — URL воркера для загрузки
- `workerNumber` — число воркеров
- `debug` — отладка
- `features` — набор фич

## 5.2 Структура контекста

Из `ht()` (destroy):

```js
e.ownerDocument = void 0;
e.ownerWindow = void 0;
e.svgStyleElement = void 0;
e.svgDefsElement = void 0;
e.svgStyles.clear();
e.defaultComputedStyles.clear();
e.sandbox && (e.sandbox.remove(), e.sandbox = void 0);
e.workers = [];
e.fontFamilies.clear();
e.fontCssTexts.clear();
e.requests.clear();
e.tasks = [];
e.shadowRoots = [];
```

**Поля контекста:**

| Поле | Что |
|---|---|
| `ownerDocument` | документ источника |
| `ownerWindow` | окно источника |
| `svgStyleElement` | `<style>` в SVG |
| `svgDefsElement` | `<defs>` в SVG |
| `svgStyles` | Map стилей SVG |
| `defaultComputedStyles` | кэш computed styles |
| `sandbox` | iframe-песочница |
| `workers` | пул воркеров |
| `fontFamilies` | Map шрифтов |
| `fontCssTexts` | Map CSS шрифтов |
| `requests` | Map загрузок |
| `tasks` | очередь задач |
| `shadowRoots` | shadow DOM |
| `log` | логгер |

## 5.3 `sandbox` — iframe

**`e.sandbox.remove()`** — значит `sandbox` — это iframe.

**Зачем:** изолированный документ для измерений. Часто в `html-to-image`
для обхода CSS-конфликтов.

## 5.4 `q()` — обёртка с `autoDestruct`

```js
async function q(e, t) {
  return Ie(e) ? e : Ke(e, { ...t, autoDestruct: true })
}
```

**Если передан Context — использовать.**
**Если Element — создать Context с `autoDestruct: true`.**

**`autoDestruct`** — уничтожить контекст после операции.

---

# ЧАСТЬ 6. Загрузка медиа — `R()` и `wt()`

## 6.1 `wt()` — fetch с таймаутом

```js
function wt(e) {
  const {url: t, timeout: r, responseType: n, ...i} = e;

  const a = new AbortController,
        s = r ? setTimeout(() => a.abort(), r) : void 0;

  return fetch(t, {signal: a.signal, ...i})
    .then(o => {
      if (!o.ok) throw new Error("Failed fetch, not 2xx response", {cause: o});
      switch (n) {
        case "arrayBuffer": return o.arrayBuffer();
        case "dataUrl": return o.blob().then(ze);
        case "text":
        default: return o.text()
      }
    })
    .finally(() => clearTimeout(s))
}
```

**`AbortController` + таймаут.**

**Три типа ответа:** `arrayBuffer`, `dataUrl` (через blob), `text`.

**`new Error(msg, {cause})`** — ES2022 `cause`.

## 6.2 `R()` — загрузка с кэшем и воркерами

```js
function R(e, t) {
  const {url: r, requestType: n = "text", responseType: i = "text", imageDom: a} = t;
  let s = r;

  const {timeout: o, acceptOfImage: c, requests: u, fetchFn: l,
         fetch: {requestInit: f, bypassingCache: d, placeholderImage: g},
         font: y, workers: m, fontFamilies: w} = e;

  n === "image" && (P || B) && e.drawImageCount++;

  let p = u.get(r);              // ← кэш по URL
  if (!p) {
    // bypassingCache
    d && d instanceof RegExp && d.test(s)
      && (s += (/\\?/.test(s) ? "&" : "?") + new Date().getTime());

    // минификация шрифта
    const b = n.startsWith("font") && y && y.minify,
          h = new Set;
    b && n.split(";")[1].split(",").forEach(_ => {
      w.has(_) && w.get(_).forEach(j => h.add(j))
    });
    const E = b && h.size;

    const S = {
      url: s,
      timeout: o,
      responseType: E ? "arrayBuffer" : i,
      headers: n === "image" ? {accept: c} : void 0,
      ...f
    };

    p = {type: n, resolve: void 0, reject: void 0, response: null};

    p.response = (async () => {
      // 1. кастомный fetchFn для картинок
      if (l && n === "image") {
        const C = await l(r);
        if (C) return C
      }

      // 2. через воркер (кроме Safari/Firefox и http)
      return !P && r.startsWith("http") && m.length
        ? new Promise((C, _) => {
            m[u.size & m.length - 1].postMessage({rawUrl: r, ...S});
            p.resolve = C;
            p.reject = _
          })
        : wt(S)
    })().catch(C => {
      u.delete(r);                              // ← убрать из кэша при ошибке
      if (n === "image" && g) return ...        // placeholder
      ...
    });

    u.set(r, p);
  }
  ...
}
```

### Ключевые моменты

**1. Кэш `u` по URL.** Один запрос на URL.

**2. `bypassingCache`** — RegExp. Если URL подходит — добавить
`?_t=timestamp`.

**3. Шрифты:**
```js
const b = n.startsWith("font") && y && y.minify
```
Если запрос шрифта и `minify: true` — собираются `fontFamilies`,
которые используют этот шрифт.

**4. Воркеры:**
```js
!P && r.startsWith("http") && m.length
```
**Не Safari, HTTP, есть воркеры** → загрузка через воркер.

**`m[u.size & m.length - 1]`** — выбор воркера по `requests.size & (n-1)`.
Это **работает только если `m.length` — степень двойки**.

**Если `workerNumber = 3`** → `3 & 2` = `2`, `4 & 2` = `0` — распределение
неравномерное.

**5. При ошибке** — `u.delete(r)` (кэш не отравляется).

## 6.3 `Xe()` — `waitUntilLoad`

```js
async function Xe(e, t) {
  I(e) && (
    N(e) || F(e)
      ? await k(e, t)                                    // IMG или VIDEO
      : await Promise.all(
          ["img", "video"].flatMap(r =>
            Array.from(e.querySelectorAll(r)).map(n => k(n, t))
          )
        )
  )
}
```

**Если элемент — IMG или VIDEO** → загрузить его.
**Иначе** → найти все `img` и `video` внутри и загрузить.

**`flatMap`** — объединить два массива промисов.

---

# ЧАСТЬ 7. Рендер

## 7.1 `kt()` — `domToForeignObjectSvg`

```js
async function kt(e, t) {
  const r = await q(e, t);
  if (v(r.node) && D(r.node)) return r.node;      // уже SVG → вернуть

  const {ownerDocument: n, log: i, tasks: a, svgStyleElement: s,
         svgDefsElement: o, svgStyles: c, font: u, progress: l,
         autoDestruct: f, onCloneNode: d, onEmbedNode: g,
         onCreateForeignObjectSvg: y} = r;

  i.time("create foreignObjectSVG");

  // ... создание <svg><foreignObject><div>...</div></foreignObject></svg>

  return ...
}
```

**`v(r.node) && D(r.node)`** — если `node` — SVG-элемент → вернуть как есть.

**Из `Dt()`:**
```js
const i = r.ownerDocument.createElementNS(null, "svg");
i.setAttributeNS(null, "width", "100%");
i.setAttributeNS(null, "height", "100%");
a.append(e);
i.appendChild(a);
```

**Создаётся:**
```html
<svg width="100%" height="100%">
  <foreignObject>
    ...DOM...
  </foreignObject>
</svg>
```

**`createElementNS(null, "svg")`** — **без namespace**! То есть `null` вместо
`"http://www.w3.org/2000/svg"`.

**Это работает** для создания, но атрибуты `xlink` могут не работать.

## 7.2 `Dt()` — `domToCanvas`

```js
async function Dt(e, t) {
  const r = await q(e, t),
        n = await kt(r),                                    // SVG
        i = Ve(n, r.isEnable("removeControlCharacter"));     // убрать控制-символы

  r.autoDestruct || (
    r.svgStyleElement = ie(r.ownerDocument),
    r.svgDefsElement = r.ownerDocument?.createElementNS(x, "defs"),
    r.svgStyles.clear()
  );

  const a = T(i, n.ownerDocument);                          // SVG → Image
  return await Qe(a, r)                                     // Image → Canvas
}
```

### Шаги

**1. `q(e, t)`** — Context.
**2. `kt(r)`** — SVG с foreignObject.
**3. `Ve(n, removeControlCharacter)`** — удалить управляющие символы.
**4. `T(i, doc)`** — SVG-строка → `Image` через dataURL.
**5. `Qe(a, r)`** — нарисовать Image в Canvas.

**`x`** — namespace SVG (`"http://www.w3.org/2000/svg"`).

**Интересно:** в `kt()` используется `createElementNS(null, "svg")`, а
в `Dt()` — `createElementNS(x, "defs")` с правильным namespace.

## 7.3 `_t()` — `domToDataUrl`

```js
async function _t(e, t) {
  const r = await q(e, t),
        {log: n, quality: i, type: a, dpi: s} = r,
        o = await Dt(r);

  n.time("canvas to data url");

  let c = o.toDataURL(a, i);
  if (/* PNG */ && s) { /* патч pHYs */ }

  return n.timeEnd("canvas to data url"), c
}
```

---

# ЧАСТЬ 8. Схема

```
domToDataUrl(element, options)
 │
 ├─ q(element, options)
 │    └─ Context (ownerDocument, workers, requests, svgStyles, ...)
 │
 ├─ kt(context) → SVG
 │    ├─ if (element is SVG) return element
 │    ├─ createElementNS(null, "svg")
 │    ├─ width/height = "100%"
 │    ├─ <foreignObject> + клонированный DOM
 │    ├─ инлайн всех стилей (computed styles)
 │    ├─ <style> с @font-face
 │    ├─ <defs> с загруженными картинками (base64)
 │    ├─ замена img/video на dataURL
 │    └─ → строка SVG
 │
 ├─ Ve(svg) → убрать управляющие символы
 │
 ├─ T(svg, doc) → Image
 │    └─ через data:image/svg+xml;base64,...
 │
 ├─ Qe(image, context) → Canvas
 │    ├─ canvas.width = width * scale
 │    ├─ canvas.height = height * scale
 │    ├─ ctx.drawImage
 │    └─ свои воркеры для растрирования
 │
 ├─ canvas.toDataURL(type, quality)
 │
 └─ if (type === "image/png" && dpi)
      └─ be(pngBytes, dpi) — вставить/заменить pHYs
```

---

# ЧАСТЬ 9. Слабые места

## 9.1 `String.fromCharCode(...p)` — переполнение стека

```js
const b = window.btoa(String.fromCharCode(...p));
```

**`p`** — Uint8Array всего PNG. Для 4000×4000 PNG это ~10-50 МБ.

**Math:** аргументы в стек-фрейме. Лимит ~65 000–125 000.

**`RangeError: Maximum call stack size exceeded`.**

**Реально:** любой скриншот больше ~8000×8000 (или ~65 КБ после
сжатия... нет, `p` — весь массив байт) **сломает** функцию.

**Порог:** ~65 000 байт. То есть **PNG больше 65 КБ**.

**Это почти любой реальный скриншот.** DPI-патч работает только для
очень маленьких картинок.

**Проверить:** `_t()` с `dpi` вызывается в `screenshotVegaPreview`.
Для Vega-графика (400×300) PNG ~10 КБ — пройдёт. Для полного экрана —
упадёт.

## 9.2 `createElementNS(null, "svg")`

```js
const i = r.ownerDocument.createElementNS(null, "svg");
```

**`null` вместо `"http://www.w3.org/2000/svg"`.**

**Последствие:** атрибуты без namespace (`width`, `height`) работают,
но `xlink:href` — нет.

**В SVG-строках картинки через `<image xlink:href="...">`** — не
отрисуются.

## 9.3 Воркер-распределение требует степень двойки

```js
m[u.size & m.length - 1]
```

**`& (n-1)`** — правильная маска **только если `n` — степень двойки**.

**`workerNumber: 3`** → `m.length = 3` → маска `2` (бинарно `10`).
- `0 & 2 = 0`
- `1 & 2 = 0`
- `2 & 2 = 2`
- `3 & 2 = 2`
- `4 & 2 = 0`

**Воркер 1 никогда не используется.** Нагрузка: 0, 2, 0, 2...

## 9.4 `bypassingCache` без URL-энкодинга

```js
s += (/\?/.test(s) ? "&" : "?") + new Date().getTime()
```

**Добавляет `?_t=1699...`.** Работает, но если URL содержит `#hash` —
параметр попадёт **внутрь** fragment (после `#`), и сервер его не увидит.

**Пример:** `https://x.com/a.png#frag` → `https://x.com/a.png#frag?_t=123`
→ браузер запросит `a.png`.

## 9.5 Кэш `requests` — `Map`, не ограничен

```js
const u = new Map;
...
u.get(r), u.set(r, p), u.delete(r)
```

**Растёт без ограничения.** При множестве URL — память.

**`delete` только при ошибке.** Успешные запросы остаются навсегда
(до `destroyContext`).

## 9.6 `val` — глобальный кэш CRC

```js
let $;
function pe(e) { ... $ || ($ = we()); ... }
```

**Не проблема** — таблица 256 int, создаётся раз.

## 9.7 `P` не учитывает iOS

```js
P = O.includes("AppleWebKit") && !re
```

**`P`** = «Safari или iOS WebView». Обе не поддерживают загрузку через
Worker так же, как Chrome.

**`!P && r.startsWith("http") && m.length`** — в Safari воркеры
не используются.

## 9.8 `q()` при `autoDestruct: false`

```js
return Ie(e) ? e : Ke(e, { ...t, autoDestruct: true });
```

**Если пользователь передал Context** — `autoDestruct` остаётся его.
**Если Element** — принудительно `true`.

**То есть:** нельзя сделать «создать контекст, использовать N раз,
уничтожить» — если передан Element, контекст уничтожится после **первой**
операции.

**Для повторных операций** нужно сначала `createContext()`, потом передать.

## 9.9 `destroyContext` не ждёт завершения воркеров

```js
e.workers = []
```

**Просто обнуляет массив.** Воркеры не `terminate()`.

**Утечка:** воркеры продолжают работать после `destroyContext`.

## 9.10 `fontFamilies.clear()` без отмены запросов

```js
e.fontFamilies.clear();
e.fontCssTexts.clear();
```

**Промисы загрузки шрифтов не отменяются** — только очищаются Map.

---

# ЧАСТЬ 10. Итог

## 10.1 Ключевые числа

| Что | Значение |
|---|---|
| Размер | 24 923 байта |
| Экспортов | 7 |
| Проверок типов элементов | 22 |
| PNG-чанк `pHYs` | 13 байт + 4 CRC |
| Константа DPI→метры | 39.3701 |
| Начало вставки pHYs | байт 33 |
| Порог переполнения стека | ~65 000 байт |
| Воркеров по умолчанию | 1 |

## 10.2 Главное

**1. `String.fromCharCode(...p)` — реальный баг.**
PNG > 65 КБ → `RangeError: Maximum call stack size exceeded`.
DPI-патч работает только для маленьких картинок.

**2. `createElementNS(null, "svg")`** — без namespace. `xlink` не работает.

**3. Воркер-распределение `& (n-1)` требует степень двойки.**
`workerNumber: 3` → воркер 1 не используется.

**4. `D()` — детектор SVG через `typeof className === "object"`.**
Единственный надёжный способ (SVGAnimatedString).

**5. `bypassingCache` ломается на URL с `#hash`.**

**6. `destroyContext` не вызывает `terminate()` на воркерах.**

**7. `q()` при Element принудительно `autoDestruct: true`** — контекст
одноразовый.

**8. `requests` — Map без ограничения, растёт до `destroyContext`.**

**9. `P` (Safari) отключает загрузку через воркеры.**

**10. `pHYs` вставляется после IHDR (байт 33) — правильно по спецификации.**
`39.3701` — точный перевод DPI→ppm.
