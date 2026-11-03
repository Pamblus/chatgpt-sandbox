# `transpile-Cld0n4VL.js` — полный разбор (3.5 МБ)

**Размер:** 3 544 292 байта
**URL:** `https://web-sandbox.oaiusercontent.com/assets/transpile-Cld0n4VL.js`
**Роль:** сборщик кода. Babel + Tailwind + esbuild-wasm + таблица shadcn.

---

# ЧАСТЬ 0. Как читать этот файл

**3.5 МБ — это не один модуль.** Это **~10 библиотек**, склеенных Vite:

| Байты | Что | Размер |
|---|---|---|
| 0 – 33 000 | Go-хелперы (`__async`), преамбула Vite | 33 КБ |
| 33 000 – 177 000 | **shadcn-компоненты** (46 штук, исходники TSX) | 144 КБ |
| 177 000 – 229 000 | shadcn-таблица + toast-компонент | 52 КБ |
| 229 000 – 945 000 | **@babel/standalone** — парсер, traverse, generator | 716 КБ |
| 945 000 – 1 061 000 | **buffer** (polyfill для Node) | 116 КБ |
| 1 061 000 – 2 211 000 | **@babel/preset-env + core-js compat data** | 1 150 КБ |
| 2 211 000 – 2 290 000 | **core-js polyfills** (список имён) | 79 КБ |
| 2 290 000 – 2 500 000 | **babel-preset-env data** (`es.*` → версии браузеров) | 210 КБ |
| 2 500 000 – 2 835 000 | **babel-plugin-polyfill-corejs** (815 КБ) | 335 КБ |
| 2 835 000 – 3 250 000 | **Tailwind CSS** (Rust→WASM обвязка) | 415 КБ |
| 3 250 000 – 3 544 292 | **Skybridge-логика**: `transpileReact`, esbuild-плагины | 294 КБ |

**Реальная логика Skybridge — последние ~40 КБ.** Остальное — библиотеки.

---

# ЧАСТЬ 1. Экспорты

```js
export {
  are as L,      // RESOLVE_PLUGIN_NAME = "onResolve-plugin"
  tre as R,      // LOAD_PLUGIN_NAME = "onLoad-plugin"
  Bm as U,       // USER_COMPONENT_FILE = "user-component.js"
  wJe as a,      // модуль {LOAD_PLUGIN_NAME, RESOLVE_PLUGIN_NAME, transpileReact}
  yJe as t       // transpileReact
};
```

| Экспорт | Что |
|---|---|
| `a` | модуль-обёртка (`wJe`) |
| `t` | **`transpileReact`** — главная функция |
| `L` | `"onResolve-plugin"` |
| `R` | `"onLoad-plugin"` |
| `U` | `"user-component.js"` |

---

# ЧАСТЬ 2. `transpileReact` — построчно

## 2.1 Полный код

```js
yJe = async ({id: x, code: D, sandboxLogger: g, worker: f,
              minify: v = !1, disableShadcnImports: S = !1}) => {
  // 1. esbuild-wasm
  const A = await hWe({worker: f});

  // 2. Babel: парсинг + AST + плагины
  g.time("babel.transform");
  let M, P = null;
  ({code: D, ast: M, map: P} = await AGe(D));
  g.timeEnd("babel.transform");

  // 3. shadcn-импорты
  let $ = "", H = [], W = new Set;
  S || (
    g.time("generateShadcnImportBlock"),
    {imports: $, components: H, classNames: W} = await wGe(M),
    g.timeEnd("generateShadcnImportBlock")
  );

  // 4. склеить
  D = `${$}\n\n${D}`;

  // 5. обёртка index.js
  const U = jGe(H);

  // 6. Tailwind CSS
  const I = fJe({language: "jsx", code: D, additionalClassNames: W, sandboxLogger: g});

  // 7. esbuild
  g.time("esbuild.transpile");
  if (!La || !La.id || La.id !== x) {
    La = {
      id: x, css: [], indexFile: U, sourceCode: D,
      buildContext: await A.context({
        entryPoints: ["index.js"],
        bundle: !0,
        write: !1,
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

  // 8. извлечь файлы
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
  }
};
```

## 2.2 Шаг 1 — `hWe({worker})`

```js
hWe = async ({worker: x = !1} = {}) => {
  if (QD) return await QD;                         // ← кэш промиса

  const D = [
    `https://cdn.jsdelivr.net/npm/esbuild-wasm@${fZ}/esbuild.wasm`,
    `https://unpkg.com/esbuild-wasm@${fZ}/esbuild.wasm`
  ];

  for (let g = 0; g < D.length; g++) {
    const f = D[g];
    try {
      return QD = new Promise(v => {
        pZ.initialize({wasmURL: f, worker: x}).then(() => { v(pZ) })
      }), await QD
    } catch (v) {
      if (g === D.length - 1) throw v
    }
  }

  throw new Error("Failed to initialize esbuild")
};
```

**`fZ = "0.24.2"`** — версия esbuild-wasm.

**`pZ`** — модуль esbuild-wasm.

**Два CDN:** jsdelivr → unpkg (fallback).

**`worker: true`** — esbuild запускается в Web Worker (не блокирует UI).

**`QD`** — глобальный кэш промиса. После успеха `return await QD` — мгновенно.

**Проблема:** если первая инициализация упала, `QD` остался rejected-
промисом? Нет — при ошибке `catch (v)` сработает, и `QD` перезапишется
на следующей итерации. Но после **последней** неудачи `QD` остаётся
rejected → следующий вызов `if (QD) return await QD` → сразу бросит.

**То есть после полного провала `hWe` повторные вызовы падают мгновенно**
(без попытки загрузки). Это может быть фичей (не долбить CDN), но
не даёт восстановиться.

**Без SRI.** `wasmURL` — просто URL, никакой проверки целостности.

## 2.3 Шаг 2 — `AGe(code)` (Babel)

```js
AGe = async x => {
  const {types: D} = wB.packages,
        {code: g, ast: f = null, map: v} = wB.transform(x, {
          filename: "index.tsx",
          ast: !0,
          sourceMaps: !0,
          parserOpts: {plugins: ["jsx", "typescript"]},
          plugins: [
            hGe(D),                                    // source-map-data-attrs
            bGe(D),                                    // lucide-deep-imports
            FGe([/^(?:\/|@)?components(?:\/.*)?$/,
                 /^(?:\/|@)?shadcn(?:\/.*)?$/,
                 /^(?:\/|@)?ui(?:\/.*)?$/])            // remove-matching-imports
          ]
        });

  if (!f) throw new Error("Failed to parse code");
  return {code: g ?? x, ast: f, map: v}
};
```

**`wB`** = `@babel/standalone` (716 КБ).

**`filename: "index.tsx"`** — **всегда TSX**, независимо от реального
расширения. Значит `interface Foo {}` и `let x: number` пройдут.

**Плагины парсера:** `jsx`, `typescript`.

### Плагин 1 — `hGe` (`source-map-data-attrs`)

```js
hGe = x => ({
  name: "source-map-data-attrs",
  visitor: {
    JSXElement(D) {
      const {openingElement: g, loc: f} = D.node;

      // имя элемента (включая MemberExpression)
      let v = "";
      const S = g.name;
      if (x.isJSXIdentifier(S)) v = S.name;
      else if (x.isJSXMemberExpression(S)) {
        const I = [];
        let ee = S;
        for (; x.isJSXMemberExpression(ee);) I.unshift(ee.property.name), ee = ee.object;
        x.isJSXIdentifier(ee) && I.unshift(ee.name);
        v = I.join(".")
      }

      // позиция: startLine:startCol-endLine:endCol
      const A = f?.start.line ?? 0,
            M = f?.start.column ?? 0,
            P = f?.end.line ?? 0,
            $ = f?.end.column ?? 0,
            H = `${A}:${M}-${P}:${$}`;

      // проверка, есть ли атрибут
      const W = I => g.attributes.some(ee =>
        x.isJSXAttribute(ee) && x.isJSXIdentifier(ee.name, {name: I}));

      // вставить, если нет
      const U = (I, ee) => {
        if (W(I)) return;
        const Ce = x.jsxAttribute(x.jsxIdentifier(I), x.stringLiteral(ee));
        g.attributes.unshift(Ce)
      };

      U("data-source-name", v),
      U("data-source-pos", H)
    }
  }
});
```

**Что делает:** каждому JSX-элементу добавляет **два атрибута**:

```jsx
<Button className="x" />
```
→
```jsx
<Button data-source-pos="5:4-5:25" data-source-name="Button" className="x" />
```

### Зачем

**Это механизм отладки генератора.** Когда модель пишет JSX и он
ломается — по `data-source-name` и `data-source-pos` можно понять,
**какой именно элемент** из кода пользователя соответствует элементу
в DOM.

**Виджет получает эти атрибуты.** Они видны в DevTools, в `outerHTML`,
в `document.querySelectorAll("[data-source-name]")`.

**Это также утечка структуры исходного кода:** атрибуты раскрывают
имена компонентов и позиции.

**`g.attributes.unshift(Ce)`** — атрибуты добавляются **в начало** списка.
Значит в HTML они идут первыми.

### Плагин 2 — `bGe` (`lucide-deep-imports`)

```js
bGe = x => ({
  name: "lucide-deep-imports",
  visitor: {
    ImportDeclaration(D, g) {
      const f = D.node.source && D.node.source.value;
      if (f !== "lucide-react" || D.node.importKind === "type") return;

      const v = "lucide-react/dist/esm/icons",
            S = D.node.specifiers;
      if (!S || S.length === 0) return;

      const A = [], M = [];

      for (const P of S) {
        if (x.isImportSpecifier(P)) {
          const $ = P.imported.name,
                H = P.local.name,
                W = vGe($),                          // CamelCase → kebab-case
                U = `${v}/${W}`;
          A.push(x.importDeclaration(
            [x.importDefaultSpecifier(x.identifier(H))],
            x.stringLiteral(U)
          ))
        } else {
          (x.isImportDefaultSpecifier(P) || x.isImportNamespaceSpecifier(P))
            && M.push(P.local.name)
        }
      }

      A.length > 0 && (D.insertBefore(A), M.length === 0 ? D.remove() : D.replaceWith(...))
    }
  }
});
```

**`vGe()` — CamelCase → kebab-case:**
```js
vGe = x => x
  .replace(/([a-z0-9])([A-Z])/g, "$1-$2")
  .replace(/([A-Z]+)([A-Z][a-z0-9]+)/g, "$1-$2")
  .replace(/(\d+)/g, "-$1")
  .replace(/-+/g, "-")
  .toLowerCase();
```

**Примеры:**
- `ChevronDown` → `chevron-down`
- `SunMoon` → `sun-moon`
- `ArrowUpRight` → `arrow-up-right`
- `CircleAlert` → `circle-alert`

**Что делает плагин:**
```jsx
import { Sun, Moon } from "lucide-react";
```
→
```jsx
import Sun from "lucide-react/dist/esm/icons/sun";
import Moon from "lucide-react/dist/esm/icons/moon";
```

**Зачем:** `import {Sun} from "lucide-react"` тянет **весь пакет**
(1000+ иконок). Deep-import тянет одну иконку.

**Как это работает в esbuild:** `resolve` для `lucide-react/dist/esm/icons/sun`
не подходит под `/^[^./]/` → уйдёт в jsdelivr:
```
https://cdn.jsdelivr.net/npm/lucide-react/dist/esm/icons/sun/+esm
```

**Но:** `external: ["react", "react-dom", "lucide-react", "@/components/*"]` —
`lucide-react` **внешний**. Значит `lucide-react/dist/esm/icons/sun` — не
равно `lucide-react`, **не** matches external? esbuild external матчит по
префиксу пути? Нет — точное совпадение или wildcard.

**`external: ["lucide-react"]`** — исключает только ровно `"lucide-react"`.
`"lucide-react/dist/esm/icons/sun"` — **не** исключён → пойдёт в resolve.

**Значит:** бандл будет содержать **отдельные иконки с jsdelivr**.

### Плагин 3 — `FGe` (`remove-matching-imports`)

```js
FGe = (x = []) => ({
  name: "remove-matching-imports",
  visitor: {
    ImportDeclaration(D) {
      EGe(D.node.source.value, x) && D.remove()
    }
  }
});

function EGe(x, D) {
  return D.some(g => typeof g === "string" ? g === x : g.test(x))
}
```

**Убирает импорты**, подходящие под шаблоны:
```js
[/^(?:\/|@)?components(?:\/.*)?$/,
 /^(?:\/|@)?shadcn(?:\/.*)?$/,
 /^(?:\/|@)?ui(?:\/.*)?$/]
```

**Что попадает:**
- `components/foo`
- `@components/foo`
- `/components/foo`
- `shadcn/bar`, `@shadcn/bar`
- `ui/baz`, `@ui/baz`

**Зачем:** модель может написать `import {X} from "@/ui/x"` — такой импорт
надо убрать, потому что реальные компоненты подставляются через
`generateShadcnImportBlock`.

**НО:** `@/components/ui/button` — matчится на `/^(?:\/|@)?components/`?
Нет: `@/components/ui/button` начинается с `@/`, а regex требует
`@` **сразу** за которым `components`. `@/c` ≠ `@c`. Значит
`@/components/...` **не** удаляется.

**Проверим:** `/^(?:\/|@)?components/` — `(?:/|@)?` опционально, потом
литерал `components`. `"@/components/ui/button"` → первый символ `@`,
потом `/`, потом `c`. Regex требует `@` + `components`, а тут `@` + `/`.
**Не совпадёт.**

**То есть плагин удаляет только `@components/ui/button` (без слэша)** —
что редкость. Основной путь `@/components/ui/...` — не удаляется.

**Это, вероятно, баг** (или намеренно: shadcn-импорты обрабатываются
позже, в esbuild-плагине `mJe`).

## 2.4 Шаг 3 — `wGe(ast)` (shadcn-импорты)

```js
RGe = async x => {
  const {packages: D} = await vee(() => Promise.resolve().then(() => CGe), void 0);
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

  return v
};
```

**Что делает:** обходит все JSX-элементы, собирает имена, у которых
**есть заглавные буквы**.

```jsx
<div>          → "div"      → не компонент
<Button>       → "Button"   → компонент
<Card.Header>  → "Card.Header"
```

**`P.toLowerCase() !== P`** — фильтр.

**Результат:** `["Button", "Card", "CardHeader", "CardTitle"]`.

Дальше `wGe` (полная версия) сопоставляет с таблицей `BGe`:

```js
BGe = [{
  importPath: "@/components/ui/accordion",
  importStatement: 'import { Accordion, AccordionItem, AccordionTrigger, AccordionContent } from "@/components/ui/accordion";',
  exports: ["Accordion", "AccordionItem", "AccordionTrigger", "AccordionContent"],
  classNames: ["border-b", "flex", "flex-1", "items-center", ...]
}, ...]
```

**46 компонентов** в таблице (из них 44 через `gGe`/`DGe`/`mGe` + 2 отдельных —
`sonner`, `toast`).

**`classNames`** — Tailwind-классы, используемые внутри компонента. Нужны
для Tailwind-генератора: он должен знать, какие утилиты генерировать.

**`imports`** — собирается строка импортов (добавляется в начало кода).
**`components`** — имена, использованные пользователем.
**`classNames`** — Set всех классов.

## 2.5 Шаг 4 — вставка импортов

```js
D = `${$}\n\n${D}`;
```

**Строка shadcn-импортов добавляется в начало.**

**Пример:**
```jsx
// было
export default () => <Button>Hi</Button>

// стало
import { Button } from "@/components/ui/button";

export default () => <Button>Hi</Button>
```

## 2.6 Шаг 5 — `jGe(components)` (обёртка)

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

### `SGe` — с TooltipProvider

```jsx
import { createElement, Component } from 'react';
import { createRoot } from 'react-dom/client';
import UserComponent from './user-component.js';
import { TooltipProvider } from "@/components/ui/tooltip";

class UserComponentWithErrorBoundary extends Component {
  public componentDidCatch(error) {
    document.documentElement.setAttribute('data-fatal', '')
  }

  render() {
    return (
      <TooltipProvider>
        <UserComponent />
      </TooltipProvider>
    );
  }
}
```

### `TGe` — без Tooltip

```jsx
import { createElement, Component } from 'react';
import { createRoot } from 'react-dom/client';
import UserComponent from './user-component.js';

class UserComponentWithErrorBoundary extends Component {
  public componentDidCatch(error) {
    document.documentElement.setAttribute('data-fatal', '')
  }

  render() {
    return <UserComponent />
  }
}
```

### Разбор

**`UserComponentWithErrorBoundary`** — класс-компонент с `componentDidCatch`.

**`componentDidCatch` ставит `data-fatal`** — сигнал `runHTMLAsync` для
завершения с ошибкой.

**`data-ready` через двойной `requestAnimationFrame`** — гарантия, что
React закончил рендер.

**`public componentDidCatch(error)`** — TypeScript-синтаксис (`public`)
в `.tsx`-файле. Babel с `typescript`-плагином съест.

**Ошибка в `render()`** — не поймается `catch`. Поймается только
`componentDidCatch` (async-фаза React).

**`try/catch` вокруг `createRoot`** — ловит синхронные ошибки
инициализации.

**Tooltip определяется через `components.includes("Tooltip")`** —
строковая проверка. Если модель использовала `<Tooltip>`, будет
`TooltipProvider`.

## 2.7 Шаг 6 — `fJe` (Tailwind)

```js
fJe = async ({sandboxLogger: x, code: D, language: g = "jsx",
              additionalClassNames: f = []}) => {
  const v = UB(f),                                  // нормализовать классы
        S = cJe(D, g, v),                           // ключ кэша
        A = Fee.get(S);                             // из кэша

  return A || pJe.runExclusive(async () => {
    x.time("tailwind");
    try {
      const M = await Ree({content: D, extension: g, returnPositions: !1}),
            P = UB(M.filter(H => typeof H === "string").concat(v)),
            $ = await oJe({css: lJe, candidates: P});
      return Fee.set(S, $), $;
    } catch (M) {
      throw x.error("Tailwind generation failed", M), M;
    } finally { x.timeEnd("tailwind") }
  })
};
```

### `UB()` — нормализация списка классов

```js
function UB(x) {
  const D = Array.from(x)
    .filter(g => typeof g === "string" && g.length > 0)
    .map(g => g.trim())
    .filter(Boolean);
  return Array.from(new Set(D)).sort()
}
```

**Тип → строка, trim, дедуп, sort.**

**`sort()`** — для стабильности ключа кэша.

### `cJe()` — ключ кэша

```js
cJe = (x, D, g) => {
  const f = UB(g);
  return `${D}:${f.join(".")}::${x}`
};
```

**Формат:** `"jsx:flex.p-4.text-sm::<весь код>"`.

**Кэш `Fee`** — `new mWe({max: 10})` — **LRU на 10 записей**.

### `pJe.runExclusive()` — мьютекс

```js
class xJe {
  constructor() { kn(this, "q", []); kn(this, "locked", !1) }
  async acquire() {
    return new Promise(D => {
      const g = () => {
        const f = this.q.shift();
        f ? f() : this.locked = !1
      };
      this.locked ? this.q.push(() => D(g)) : (this.locked = !0, D(g))
    })
  }
  async runExclusive(D) {
    const g = await this.acquire();
    ...
  }
}
```

**Простейший мьютекс.** `locked` + очередь `q`.

**Зачем:** `tailwindcss-oxide` (WASM) не потокобезопасен. Два параллельных
вызова → повреждение памяти WASM.

**Проблема:** если задача зависнет — `locked` навсегда `true`, очередь
стоит. Нет таймаута.

### `Ree()` — сканирование классов

```js
const OGe = "/assets/tailwindcss_oxide_bg-DaSLdPNK.wasm";

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

**`tailwindcss_oxide_bg-DaSLdPNK.wasm`** — 1.72 МБ Rust-компиляция Tailwind.

**`WasmScanner`** — сканер классов на Rust.

**`WasmChangedContent(content, extension)`** — контент для сканирования.

**`getCandidatesWithPositions()`** — возвращает найденные классы с позициями.

**`returnPositions: false`** → только строки-классы.

### `oJe()` — компиляция CSS

```js
async function oJe({content: x = "", css: D = "", importCSS: g = '@import "tailwindcss";',
                    candidates: f = []}) {
  f.length || (f = await Ree({content: x}));
  const v = D ? `${g}\n${D}` : g;

  return (await iJe(v, {
    base: "/",
    loadStylesheet: _Ge,
    loadModule: () => { throw new Error("External modules not supported in browser build") }
  })).build(f)
}
```

**`importCSS` по умолчанию** — `@import "tailwindcss";`.

**`v`** — `@import "tailwindcss";\n<наш CSS>`.

**`loadStylesheet: _Ge`** — резолвер виртуальных стилей Tailwind:

```js
async function _Ge(x, D) {
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

**Четыре виртуальных файла** (`kGe`, `PGe`, `IGe`, `NGe`) — вшитые
исходники Tailwind v4.

**Любой другой `@import` → ошибка.** Внешние стили в браузерной сборке
не поддерживаются.

**`loadModule`** — всегда бросает. `@plugin`/`@config` не работают.

### `lJe` — базовая CSS-тема

```js
lJe = `
@layer base {
  :root {
    ${bB.map(x => `--${x}: ${uJe[x]};`).join("\n\t")}
  }

  @media (prefers-color-scheme: dark) {
    ${bB.map(x => `--${x}: ${dJe[x]};`).join("\n\t")}
  }

  *,
  ::before,
  ::after {
    @apply border-border;
  }
}

@theme {
    ${bB.map(x => `--color-${x}: hsl(var(--${x}));`).join("\n\t")}
}
`;
```

**`bB`** — список переменных shadcn:
```js
["background", "foreground", "muted", "muted-foreground", "popover",
 "popover-foreground", "border", "input", "card", "card-foreground",
 "primary", "primary-foreground", "secondary", "secondary-foreground",
 "accent", "accent-foreground", "destructive", "destructive-foreground",
 "ring", "radius", "chart-1", "chart-2", "chart-3", "chart-4", "chart-5",
 "sidebar-background", "sidebar-foreground", "sidebar-primary",
 "sidebar-primary-foreground", ...]
```

**`uJe`** — светлые значения (HSL-компоненты):
```js
{
  background: "0 0% 100%",
  foreground: "222.2 47.4% 11.2%",
  muted: "210 40% 96.1%",
  "muted-foreground": "215.4 16.3% 46.9%",
  primary: "222.2 47.4% 11.2%",
  ...
}
```

**`dJe`** — тёмные:
```js
{
  background: "224 71% 4%",
  foreground: "213 31% 91%",
  muted: "223 47% 11%",
  "muted-foreground": "215.4 16.3% 56.9%",
  primary: "210 40% 98%",
  ...
}
```

**Тёмная тема через `@media (prefers-color-scheme: dark)`** — системная,
не управляется классом.

**`@apply border-border`** — все элементы получают границу по умолчанию.

## 2.8 Шаг 7 — esbuild `context()`

```js
A.context({
  entryPoints: ["index.js"],
  bundle: !0,
  write: !1,
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
```

| Параметр | Значение | Что |
|---|---|---|
| `entryPoints` | `["index.js"]` | виртуальная точка входа |
| `bundle` | `true` | собрать всё в один файл |
| `write` | `false` | в память, не на диск |
| `format` | `"esm"` | ES-модули |
| `minify` | параметр | минификация |
| `treeShaking` | `true` | удалять неиспользуемое |
| `splitting` | `false` | один файл (без чанков) |
| `jsx` | `"automatic"` | `jsx()` вместо `React.createElement` |
| `jsxImportSource` | `"react"` | из `react/jsx-runtime` |
| `sourcemap` | `"external"` | отдельный `.js.map` |
| `external` | 4 пути | не бандлить |

**`external`:**
- `react` — берётся из глобала?
- `react-dom`
- `lucide-react`
- `@/components/*`

**Но:** если `react` external, а в HTML нет глобального React — бандл
упадёт при `import React from "react"`.

**Значит должен быть importmap или инъекция.** В `getHtml` шаблон:
```html
<script type="module">${TRANSPILED_CODE}</script>
```

`transpiledCode` содержит `import{...}from"react"`. Браузер попытается
разрешить `"react"` как URL → **ошибка**.

**Если только** `transpiledCode` **не** содержит импорта React. При
`jsx: "automatic"` и `jsxImportSource: "react"` esbuild генерирует:
```js
import {jsx} from "react/jsx-runtime";
```

**`react` external** → `react/jsx-runtime` тоже external → в бандле
останется `import{jsx}from"react/jsx-runtime"` → **браузер не разрешит**.

**Разгадка:** в `csp` есть `resourceDomains`. И в `apply-csp` `script-src`
включает `https://esm.sh`, `https://cdn.jsdelivr.net`, `https://unpkg.com`.

**То есть ожидается importmap или Service Worker.** Но в HTML-шаблоне
`getHtml` **нет** importmap.

**Проверим `run-react`:** `html: X` из `getHtml`. Никакого importmap.

**Гипотеза:** esbuild при `external` с `format: esm` и без importmap
генерирует `import ... from "react"`, и это **работает** только если
хост подставляет importmap через `prepareDocument` или
если в бандле React **вшит** через `@/components/*` → нет.

**Логичное объяснение:** `external` перечисляет пакеты, которые **уже
есть в бандле** через shadcn-компоненты. То есть `react` попадает в бандл
через `import {createElement} from 'react'` в `TGe`... нет, это тоже
внешний.

**Реальное объяснение:** в esbuild `external` с `format: esm` и
`export`-режимом оставляет импорты как есть. Но **`bundle: true`**
означает, что всё, что **не** external, будет включено.

**`react` external** → в бандле `import{jsx}from"react/jsx-runtime"`.
Браузеру нужен importmap.

**Возможно, хост подставляет importmap.** В `runWidgetCode` есть
`prepareDocument` (через `prepareWidgetDocument`). В `runUserCode` — нет.

**Тест:** если запустить React-виджет и он работает — значит либо
importmap, либо React вшит, либо внешние импорты резолвятся через
jsdelivr `+esm`.

**Проверить в реальном HTML:** в вашем дампе был `iframe` с
`<script type="module" src=".../assets/main-BR1bOQyH.js">`. Но это
**главный** документ, не iframe с виджетом.

**Вывод:** механизм разрешения `react` в итоговом бандле — **не
подтверждён**. Скорее всего через `importmap` в `prepareDocument`
или через то, что `react` **не** остаётся external при `bundle: true`
(esbuild с `external` в `bundle`-режиме **оставляет** импорт).

**Проверить экспериментально:** собрать и посмотреть `transpiledCode`.

### `La` — кэш buildContext

```js
if (!La || !La.id || La.id !== x) {
  La = {id: x, css: [], indexFile: U, sourceCode: D,
        buildContext: await A.context({...})}
} else {
  La.indexFile = U;
  La.sourceCode = D;
  La.css = [];
}
```

**Один `buildContext` на `id`.**

**Почему кэш:** `A.context()` создаёт Rust-структуру (esbuild-wasm),
дорого. `rebuild()` дешевле.

**При совпадении `id`** — обновляются `indexFile` и `sourceCode`,
`css` сбрасывается.

**`La.css`** — массив CSS, собранный из `external`-файлов (в `gJe`):
```js
else if (D.endsWith(".css") || f?.includes("text/css")) {
  const P = await g.text();
  La?.css.push(P);
  ...
}
```

## 2.9 Шаг 8 — извлечение

```js
const {outputFiles: ee} = await La.buildContext.rebuild();

if (!ee) throw new Error("Failed to transpile React code");

const Ce = ee.find(ze => ze.path.endsWith(".js")),
      Se = ee.find(ze => ze.path.endsWith(".js.map"));
if (!Ce) throw new Error("Expected JS file");
if (!Se) throw new Error("Expected source map file");
```

**Ищет `.js` и `.js.map`.**

**Проблема:** `.js.map` оканчивается на `.map`, не на `.js`. Но
`"bundle.js.map".endsWith(".js")` = `false`. Значит `Ce` найдёт
`bundle.js`, а `Se` — `bundle.js.map`. **Порядок правильный.**

**Но:** `find(endsWith(".js"))` вернёт **первый** файл, оканчивающийся
на `.js`. Если в `outputFiles` есть `bundle.js` и `bundle.js.map`,
`find` вернёт `bundle.js` (порядок в массиве).

**`JSON.parse(Se.text)`** в `try/catch` — если sourcemap битый, `Ve = null`.

**Возврат:**
```js
{
  transpiledCode: dr,
  sourceMap: {babel: P, esbuild: Ve},
  css: [...La?.css ?? [], await I]
}
```

**`css`** — CSS из внешних файлов + Tailwind.

---

# ЧАСТЬ 3. esbuild-плагины

## 3.1 `mJe()` — resolve

```js
mJe = () => ({
  name: tre,        // "onResolve-plugin"
  setup(x) {
    // 1. index.js → internal
    x.onResolve({filter: /^index\.js$/}, ({path: D}) => ({path: D, namespace: "internal"}));

    // 2. ./user-component.js → internal
    x.onResolve({filter: /^\.\/user-component\.js$/}, () => ({path: Bm, namespace: "internal"}));

    // 3. относительные → external (URL)
    x.onResolve({filter: /^\.+\//}, ({path: D, importer: g}) => {
      let f = "";
      try { f = new URL(D, g).toString() }
      catch { if (!DJe.test(g)) throw new Error(`File not found: ${D}`) }
      return {path: f, namespace: "external"}
    });

    // 4. @/... → internal если в таблице, иначе null
    x.onResolve({filter: /^@\/.*/}, ({path: D}) =>
      D in AB ? {path: D, namespace: "internal"} : null);

    // 5. пакеты → jsdelivr +esm
    x.onResolve({filter: /^[^./]/}, ({path: D}) => {
      const g = Aee(D);
      return {path: RB(`${Eee}/npm/${g}/+esm`), namespace: "external"}
    });

    // 6. абсолютные → jsdelivr
    x.onResolve({filter: /^\//}, ({path: D}) => ({
      path: RB(`${Eee}${D}`), namespace: "external"
    }));
  }
});
```

### Таблица

| Regex | Что | Куда |
|---|---|---|
| `/^index\.js$/` | точка входа | `internal` |
| `/^\.\/user-component\.js$/` | код пользователя | `internal` |
| `/^\.+\//` | `./x`, `../x` | URL → `external` |
| `/^@\/.*/` | `@/components/ui/x` | `internal` если в `AB`, иначе `null` |
| `/^[^./]/` | `react`, `lucide-react/dist/...` | `jsdelivr/npm/.../+esm` |
| `/^\//` | `/foo` | `jsdelivr/foo` |

### **КЛЮЧЕВОЕ: любой пакет идёт на jsdelivr**

```js
x.onResolve({filter: /^[^./]/}, ({path: D}) => {
  const g = Aee(D);
  return {path: RB(`${Eee}/npm/${g}/+esm`), namespace: "external"}
});
```

**`Eee = "https://cdn.jsdelivr.net"`**.

**`import x from "любой-пакет"`** → `https://cdn.jsdelivr.net/npm/любой-пакет/+esm`
→ `gJe` сделает `fetch()` → код выполнится.

**Это главный вектор:** любой npm-пакет, опубликованный кем угодно,
будет исполнен в виджете.

### `Aee()` — нормализация lucide

```js
function Aee(x) {
  return x.replace(/(lucide-react\/dist\/esm\/icons\/[^/"'?]+)\.js(?=\/|["'?]|$)/g, "$1")
}
```

**Убирает `.js` из путей lucide-иконок.** Зачем — неясно (jsdelivr всё
равно отдаст).

### `RB()` — пиннинг версий

```js
function RB(x) {
  let D = Aee(x);
  for (const [g, f] of Object.entries(yGe)) {
    const v = g.replace(/[.*+?^${}()|[\]\\]/g, "\\$&"),
          S = new RegExp(`((?:^|/))((?:npm/)${v})(@[^/]+)?(?=/|$|\\?)`, "g");
    D = D.replace(S, `$1$2@${f}`)
  }
  return D
}
```

**Что делает:** вставляет версию в URL, если её нет.

**`yGe`:**
```js
{
  react: "18.3.1",
  "react-dom": "18.3.1",
  recharts: "3.8.1",
  three: "0.172.0",
  "@react-three/fiber": "8.17.7",
  "@react-three/drei": "9.121.3"
}
```

**Пример:**
```
https://cdn.jsdelivr.net/npm/react/+esm
→ https://cdn.jsdelivr.net/npm/react@18.3.1/+esm
```

**Regex** `((?:^|/))((?:npm/)${v})(@[^/]+)?(?=/|$|\?)`:
- `((?:^|/))` — начало или `/`
- `((?:npm/)react)` — `npm/react`
- `(@[^/]+)?` — опциональная версия
- `(?=/|$|\?)` — смотри вперёд: `/`, конец или `?`

**Замена:** `$1$2@${f}` — сохраняет префикс, добавляет версию.

**Если версия уже есть** (`@19.0.0`) — regex захватит её в группу 3, но
замена **игнорирует** группу 3 (в шаблоне `$1$2@${f}`) → **версия
перезаписывается на закреплённую**.

**То есть:** даже если пользователь напишет `import x from "react@19"` —
получит `react@18.3.1`. **Пиннинг жёсткий.**

**`three: "0.172.0"`** — но в `run-three` используется `0.179.1` (из unpkg).
**Расхождение:** разные версии для разных путей.

## 3.2 `gJe()` — load

```js
gJe = () => ({
  name: are,        // "onLoad-plugin"
  setup(x) {
    // 1. index.js
    x.onLoad({filter: /^index\.js$/, namespace: "internal"},
      () => ({contents: La?.indexFile ?? "", loader: "tsx"}));

    // 2. user-component.js
    x.onLoad({filter: new RegExp(`^${Bm}$`), namespace: "internal"},
      () => ({contents: La?.sourceCode ?? "", loader: "tsx"}));

    // 3. @/... из таблицы
    x.onLoad({filter: /^@\/.*/, namespace: "internal"},
      async ({path: D}) => D in AB ? {contents: AB[D], loader: "tsx"} : null);

    // 4. всё внешнее → fetch
    x.onLoad({filter: /.*/, namespace: "external"}, async ({path: D}) => {
      if (um[D]) return um[D];                     // кэш

      const g = await fetch(D);                    // ← СЕТЬ
      if (!g.ok) throw new Error(`Failed to fetch ${D}`);

      const f = g.headers.get("Content-Type");
      let v = "jsx";
      if (f?.includes("application/typescript") || D.endsWith(".ts")) v = "ts";
      else if (D.endsWith(".tsx")) v = "tsx";
      else if (D.endsWith(".js")) v = "jsx";
      else if (D.endsWith(".css") || f?.includes("text/css")) {
        const P = await g.text();
        La?.css.push(P);                            // ← CSS отдельно
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

### Загрузка из сети

**`await fetch(D)`** — реальный сетевой запрос.

**`um[D]`** — кэш по URL (глобальный объект, не очищается).

**`La?.css.push(P)`** — CSS-файлы **не** отдаются esbuild, а складываются
в `La.css`. Возвращается пустой JS-модуль.

**`RB(S)`** — пиннит версии **внутри** загруженного кода. То есть если
`react@18.3.1/+esm` содержит `import ... from "/npm/react-dom@18.3.1/+esm"` —
`RB` ничего не поменяет (версия уже есть, но перезапишется на ту же).

**Определение loader по расширению/Content-Type:**
- `application/typescript` или `.ts` → `ts`
- `.tsx` → `tsx`
- `.js` → `jsx`
- `.css` или `text/css` → CSS в `La.css`

**Проблема:** `D.endsWith(".js")` → `loader: "jsx"`. То есть `.js`
парсится как JSX. Это может ломаться на коде с `<` в сравнениях.

**`D.endsWith(".ts")`** → `loader: "ts"` (без JSX).

### Проблема кэша `um`

**Глобальный, не очищается.** Если пользователь дважды импортирует
разные версии — кэш отдаст первую.

**Память:** при множестве виджетов `um` растёт.

---

# ЧАСТЬ 4. Структура: 46 shadcn-компонентов

## 4.1 Таблица `BGe`

**46 компонентов:**

```
accordion, alert, alert-dialog, aspect-ratio, avatar, badge, breadcrumb,
button, calendar, card, carousel, chart, checkbox, collapsible, command,
context-menu, dialog, drawer, dropdown-menu, form, hover-card, input,
input-otp, label, menubar, navigation-menu, pagination, popover, progress,
radio-group, resizable, scroll-area, select, separator, sheet, sidebar,
skeleton, slider, sonner, switch, table, tabs, textarea, toast, toaster,
toggle, toggle-group, tooltip
```

## 4.2 Структура записи

```js
{
  importPath: "@/components/ui/accordion",
  importStatement: 'import { Accordion, AccordionItem, ... } from "@/components/ui/accordion";',
  exports: ["Accordion", "AccordionItem", "AccordionTrigger", "AccordionContent"],
  classNames: ["border-b", "flex", "flex-1", "items-center", "justify-between", ...]
}
```

**`importPath`** — откуда импортировать.
**`importStatement`** — готовая строка импорта.
**`exports`** — имена экспортов.
**`classNames`** — Tailwind-классы компонента.

## 4.3 Исходники компонентов

В `AB`:
```js
AB = {...gGe, ...DGe, ...mGe, "@/components/ui/sonner": nGe, "@/components/ui/toast": dGe}
```

**`gGe`, `DGe`, `mGe`** — объекты с исходниками TSX (сгруппированы).

**Пример `FWe`** (из начала файла):
```jsx
"use client"
import * as React from "react"
import * as AccordionPrimitive from "@radix-ui/react-accordion"
import ChevronDown from "lucide-react/dist/esm/icons/chevron-down";
import { cn } from "@/lib/utils"
const Accordion = AccordionPrimitive.Root
const AccordionItem = React.forwardRef<...>(({ className, ...props }, ref) => (
  <AccordionPrimitive.Item ref={ref} className={cn("border-b", className)} {...props} />
))
...
```

**Что видно:**
- `"use client"` — директива Next.js (безвредна в браузере)
- `@radix-ui/react-accordion` — **импорт внешнего пакета**
- `lucide-react/dist/esm/icons/chevron-down` — уже deep-import
- `@/lib/utils` — `cn()` из таблицы

**`@radix-ui/react-accordion`** → в `mJe` попадёт под `/^[^./]/` →
`https://cdn.jsdelivr.net/npm/@radix-ui/react-accordion@latest/+esm`
(**без версии!** — `yGe` не содержит `@radix-ui/*`).

**Это уязвимость цепочки поставок:** при каждом запуске тянется
`@latest` с jsdelivr. Если пакет скомпрометируют — код выполнится.

**Проверить:** в `BGe` 33 вхождения `@radix-ui`. Все — без пина версии.

## 4.4 `@/lib/utils` — `cn()`

Из таблицы. Реализация (типичная):
```js
import { clsx } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs) {
  return twMerge(clsx(inputs))
}
```

**`clsx` и `tailwind-merge`** — тоже с jsdelivr без версии.

---

# ЧАСТЬ 5. Babel preset-env — зачем 1.15 МБ

## 5.1 Что внутри

**`@babel/preset-env`** с полной базой совместимости:

```js
var Il = {},
    a4 = {
      "es.symbol": {android: "49", bun: "0.1.1", chrome: "49", deno: "1.0",
                    edge: "15", electron: "0.37", firefox: "51",
                    hermes: "0.1", ios: "10.0", node: "6.0", safari: "10.0", ...},
      "es.symbol.description": {android: "70", ...},
      ...
    }
```

**815 КБ — таблица `es.* → минимальная версия в каждом браузере`.**

**520 уникальных имён** ES-фич, каждое с ~20 браузерами = ~10 000 записей.

## 5.2 Используется ли?

**`AGe` не вызывает `preset-env`.** Только парсер + 3 плагина.

**Гипотеза:** `@babel/standalone` включает `preset-env` **всегда**, потому
что это единый бандл. Данные просто лежат.

**То есть 1.15 МБ загружаются впустую.**

**Проверка:** `AGe` использует `wB.transform` с явным списком плагинов.
`preset-env` не указан → не применяется.

**Но** `@babel/standalone` **загружает** данные при инициализации
(статический импорт).

## 5.3 `core-js` (2 211 000 – 2 290 000)

**3343 вхождения `corejs`.** Это `babel-plugin-polyfill-corejs`,
который заменяет нативные вызовы на полифиллы.

**Не используется** (нет `useBuiltIns`).

**Проверить:** `usebuiltins` — 46 вхождений, но все — в описаниях опций
preset-env, не в конфиге.

## 5.4 Вывод

**~1.15 МБ + 335 КБ = 1.5 МБ** — данные для `preset-env` и `core-js`,
которые **не применяются** при транспиляции Skybridge.

**Загружаются потому что `@babel/standalone` — монолитный бандл.**

---

# ЧАСТЬ 6. Поток целиком

```
transpileReact({id, code, sandboxLogger, worker, minify, disableShadcnImports})
 │
 ├─ 1. hWe({worker})
 │     ├─ jsdelivr/npm/esbuild-wasm@0.24.2/esbuild.wasm
 │     ├─ fail → unpkg
 │     └─ pZ.initialize({wasmURL, worker})
 │
 ├─ 2. AGe(code)
 │     ├─ wB.transform({filename:"index.tsx", parserOpts:{jsx,typescript}})
 │     ├─ hGe → добавить data-source-name, data-source-pos в JSX
 │     ├─ bGe → lucide-react {Sun} → import Sun from ".../icons/sun"
 │     ├─ FGe → удалить import из components/shadcn/ui
 │     └─ → {code, ast, map}
 │
 ├─ 3. wGe(ast)  [если !disableShadcnImports]
 │     ├─ traverse JSXOpeningElement → имена с заглавными
 │     ├─ сопоставить с BGe (46 компонентов)
 │     └─ → {imports: строка, components: [], classNames: Set}
 │
 ├─ 4. code = imports + "\n\n" + code
 │
 ├─ 5. jGe(components)
 │     ├─ includes("Tooltip") ? SGe (с TooltipProvider) : TGe
 │     └─ + createRoot / data-ready / data-fatal
 │
 ├─ 6. fJe({code, additionalClassNames})
 │     ├─ Fee.get(ключ) — LRU 10
 │     ├─ pJe.runExclusive() — мьютекс
 │     ├─ Ree({content}) → tailwindcss_oxide_bg.wasm (1.72 МБ) → классы
 │     ├─ oJe({css: lJe, candidates}) → виртуальные стили → CSS
 │     └─ → CSS-строка
 │
 ├─ 7. A.context({...})  [кэш по id]
 │     ├─ mJe: resolve
 │     │    ├─ index.js → internal
 │     │    ├─ user-component.js → internal
 │     │    ├─ ./x → new URL → external
 │     │    ├─ @/x → internal если в AB
 │     │    ├─ пакет → jsdelivr/npm/<pkg>@<pinned>/+esm → external
 │     │    └─ /x → jsdelivr/x → external
 │     ├─ gJe: load
 │     │    ├─ internal → из La (память)
 │     │    ├─ external → fetch() → кэш um[url]
 │     │    └─ .css → La.css.push
 │     └─ rebuild()
 │
 └─ 8. Извлечь bundle.js + bundle.js.map
       └─ → {transpiledCode, sourceMap:{babel, esbuild}, css}
```

---

# ЧАСТЬ 7. Слабые места

## 7.1 Любой npm-пакет исполняется с jsdelivr

```js
x.onResolve({filter: /^[^./]/}, ({path: D}) => {
  const g = Aee(D);
  return {path: RB(`${Eee}/npm/${g}/+esm`), namespace: "external"}
});
```

**`import x from "любой-пакет-на-npm"`** → загрузка и исполнение.

**CSP** разрешает `cdn.jsdelivr.net` в `script-src` (`fi` в `main.js`).

**Вектор:** typosquatting, компрометация пакета, публикация
вредоносного пакета с популярным именем.

## 7.2 `@radix-ui/*` и `clsx`/`tailwind-merge` — без пина версии

```js
yGe = {react, "react-dom", recharts, three, "@react-three/fiber", "@react-three/drei"}
```

**`@radix-ui/react-accordion`** (и 32 других radix) — **не в списке**.
Значит `RB()` не добавит версию → jsdelivr отдаст **`@latest`**.

**`RB()` имеет обработку `(@[^/]+)?`** — если версия есть, она
**перезаписывается** на пиннинг. Но если пиннинга нет — остаётся
`@latest`.

**То есть:** каждый запуск React-виджета с радикс-компонентом тянет
`@radix-ui/react-accordion@latest`. Смена версии на CDN = смена кода
в виджете.

## 7.3 `esbuild-wasm` с CDN без SRI

```js
`https://cdn.jsdelivr.net/npm/esbuild-wasm@0.24.2/esbuild.wasm`
`https://unpkg.com/esbuild-wasm@0.24.2/esbuild.wasm`
```

**Версия закреплена**, но `integrity` нет. Подмена на CDN = подменённый
WASM (компилятор).

## 7.4 `tailwindcss_oxide.wasm` локально — хорошо

```js
const OGe = "/assets/tailwindcss_oxide_bg-DaSLdPNK.wasm";
new URL(OGe.replace("?url", ""), import.meta.url)
```

**Локальный файл** с того же origin. Единственный WASM без внешнего CDN.

## 7.5 `data-source-name` / `data-source-pos` — утечка структуры

```js
U("data-source-name", v),
U("data-source-pos", H)
```

**Каждый JSX-элемент получает атрибуты с именем компонента и позицией.**

**Что это даёт:**
- раскрывает структуру исходного кода
- раскрывает позиции (строки/столбцы)
- может использоваться для реконструкции кода

**Не критично**, но выявляет внутреннюю механику.

## 7.6 `filename: "index.tsx"` всегда

**JS-код парсится как TSX.** `interface Foo {}` в `.js` — пройдёт
(и вырежется).

**Побочный эффект:** нельзя отловить «это JS, а не TS».

## 7.7 `try/catch` вокруг `JSON.parse` sourcemap — молча

```js
let Ve = null;
try { Ve = JSON.parse(Se.text) } catch {}
```

**Если sourcemap битый** — `Ve = null`, ошибок нет.
`Qe()` в `run-react` вернёт нетрансформированные позиции.

## 7.8 `um` — глобальный кэш с багом

```js
if (um[D]) return um[D];
...
return um[D] = M, M
```

**Кэш по URL**, но `RB(S)` применяется к **содержимому**, а не к ключу.
Если два URL отличаются только версией — разные ключи, ок.

**Утечка памяти:** `um` не очищается. При длинной сессии растёт.

## 7.9 `Fee` — LRU на 10

```js
Fee = new mWe({max: 10})
```

**10 записей.** Больше виджетов = вытеснение.

## 7.10 `pJe` мьютекс без таймаута

```js
async runExclusive(D) {
  const g = await this.acquire();
  ...
}
```

**Если `D` зависнет** (например, Tailwind-wasm упадёт в бесконечный
цикл) — `locked` навсегда `true`. Все последующие вызовы — в очередь
навсегда.

## 7.11 `La` — один buildContext на id

```js
if (!La || !La.id || La.id !== x) { ... }
```

**Если два виджета с одним `id`** — общий `buildContext`.

**`La.css` сбрасывается** при переиспользовании:
```js
} else { La.indexFile = U; La.sourceCode = D; La.css = [] }
```

**Но `um` (кэш fetch) не сбрасывается** → CSS-файлы, загруженные
в первом запуске, не попадут в `La.css` во втором.

**Баг:** при повторном запуске с тем же `id` CSS из внешних файлов
**потеряется**.

## 7.12 `three: "0.172.0"` vs `run-three` 0.179.1

**Расхождение версий.** `transpileReact` пинит `three@0.172.0`, а
`run-three` использует `unpkg.com/three@0.179.1` в importmap.

**Если React-виджет импортирует `three`** — получит 0.172.0.
**Three.js-виджет** — 0.179.1.

## 7.13 `FGe` не удаляет `@/components/...`

```js
/^(?:\/|@)?components(?:\/.*)?$/
```

`"@/components/ui/button"` → `@` + `/` + `components`. Regex требует
`@` **сразу** за `components`. **Не совпадает.**

**То есть плагин бесполезен для основного пути.**

## 7.14 `external` для `react` без importmap

```js
external: ["react", "react-dom", "lucide-react", "@/components/*"]
```

**`react` external** → в бандле `import{jsx}from"react/jsx-runtime"`.

**В HTML-шаблоне `getHtml` нет importmap.**

**Возможные объяснения:**
1. Хост подставляет importmap через `prepareDocument`
2. React **вшит** через shadcn-компоненты (`import * as React from "react"`)
3. esbuild при `bundle: true` + `external` **резолвит** через `mJe`

**Проверить:** `mJe` для `"react"` — `filter: /^[^./]/` → 
`https://cdn.jsdelivr.net/npm/react@18.3.1/+esm` → `external` → `fetch` →
**но `external` в esbuild означает «не бандлить»**.

**Противоречие:** если `react` объявлен `external`, плагин `onResolve`
**не вызывается** (esbuild пропускает external раньше).

**Значит:** `import{jsx}from"react/jsx-runtime"` остаётся как есть.

**Браузеру нужен importmap.** В `runWidgetCode` есть `prepareDocument` —
возможно, там.

**В `runUserCode` — только `runHTMLAsync` с `prepareDocument` = undefined.**

**Вывод:** для `runUserCode` React-виджет **не заведётся** без importmap.

**Проверить экспериментально.**

---

# ЧАСТЬ 8. Итог

## 8.1 Ключевые числа

| Что | Значение |
|---|---|
| Размер | 3 544 292 байта |
| Реальная логика | ~40 КБ (1.1%) |
| shadcn-компонентов | 46 |
| Записей в таблице `BGe` | 46 |
| Переменных темы (`bB`) | ~30 |
| Версий пиннинга (`yGe`) | 6 |
| Плагинов Babel | 3 |
| Плагинов esbuild | 2 |
| esbuild-wasm | 0.24.2 |
| React (пиннинг) | 18.3.1 |
| three (пиннинг) | 0.172.0 |
| Tailwind WASM | 1.72 МБ |
| LRU-кэш Tailwind | 10 |
| Core-js имён | 520 |
| Core-js compat записей | ~10 000 |

## 8.2 Главное

**1. 99% файла — библиотеки. Реальная логика ~40 КБ.**

**2. Любой npm-пакет загружается с jsdelivr и исполняется.**
`import x from "pkg"` → `https://cdn.jsdelivr.net/npm/pkg/+esm` → `fetch()`.
CSP разрешает jsdelivr.

**3. `@radix-ui/*` без пина версии → `@latest` с jsdelivr.**
33 вхождения в shadcn-компонентах. Цепочка поставок не защищена.

**4. Три Babel-плагина:**
- `source-map-data-attrs` — `data-source-name`/`data-source-pos` на каждом JSX
- `lucide-deep-imports` — `{Sun}` → `import Sun from ".../icons/sun"`
- `remove-matching-imports` — но **не работает** для `@/components/*`

**5. `jGe()` — обёртка с `ErrorBoundary` + двойной rAF + `data-fatal`.**

**6. Tailwind генерируется на WASM в мьютексе без таймаута.**
`tailwindcss_oxide_bg.wasm` 1.72 МБ локально (не CDN).

**7. `external: ["react", "react-dom", "lucide-react", "@/components/*"]`**
— React не бандлится. Importmap в HTML-шаблоне нет → механизм
разрешения не подтверждён.

**8. `La` кэш — один `buildContext` на `id`. При переиспользовании `La.css` теряется.**

**9. `three@0.172.0` в пинах, но `run-three` использует `0.179.1`. Расхождение.**

**10. 1.5 МБ данных `preset-env` + `core-js` загружаются впустую** (не применяются).

**11. `filename: "index.tsx"` жёстко — JS парсится как TSX.**

**12. `hWe` после полного провала — кэш отравлен (rejected-промис).**
