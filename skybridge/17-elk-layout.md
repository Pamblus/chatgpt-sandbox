# `render-X3XFXER2-qgBbUjf1.js` — полный разбор (ELK layout engine)

**Размер:** 1 454 844 байта
**URL:** `https://web-sandbox.oaiusercontent.com/assets/render-X3XFXER2-qgBbUjf1.js`
**Роль:** **ELK (Eclipse Layout Kernel)** — движок автоматической раскладки
графов, скомпилированный через GWT из Java в JS. Плюс **адаптер Mermaid →
ELK**.

---

# ЧАСТЬ 1. Что это

## 1.1 Происхождение

**ELK** — Eclipse Layout Kernel. Оригинал написан на **Java** (Xtext/Xtend).
Здесь — скомпилирован через **GWT** (Google Web Toolkit) в JavaScript.

**Признаки GWT-компиляции в коде:**

```js
function DT() {
  DT = F, x9 = new mt("org.eclipse.elk.labels.labelManager")
}
```

**`org.eclipse.elk.*`** — 358 вхождений. Это **отладочные имена**,
сохранённые GWT.

```js
com.google.gwt.*       7 вхождений
ClassCastException     1
NullPointerException   1
```

**Java-исключения** сохранены в рантайме.

**Структура `function X(){X=F, ...}`** — паттерн **ленивой инициализации**
статических полей Java-класса:

```java
// Java
class Layered {
    static LayoutProvider INSTANCE;
    static { INSTANCE = new ... }
}
```
→
```js
// GWT-выход
function Ik() {
  Ik = F;
  DU = new nV("FIXED", 0);
  DO = new nV("CENTER_NODE", 1);
}
```

**`Ik = F`** — флаг «уже инициализировано» (`F` = `true`).

## 1.2 Экспорты

```js
export {
  $oe as buildSubgraphLayoutOptions,
  dbn as dir2ElkDirection,
  GNe as render
};
```

| Экспорт | Что |
|---|---|
| `buildSubgraphLayoutOptions` | опции ELK для подграфа |
| `dir2ElkDirection` | `"LR"`/`"TB"` → `"RIGHT"`/`"DOWN"` |
| `render` | **главный** — раскладка Mermaid-диаграммы |

## 1.3 Импорты

```js
import {_ as Lc} from "./render-mermaid-svg-CkMiccpV.js";   // _t (preload)
import {c as lbn} from "./transform-4I4J0RKg.js";           // curve transformer
import {k as Noe, j as LNe} from "./main-BR1bOQyH.js";      // Object
```

**Циклическая зависимость:** `render` импортирует из `render-mermaid-svg`,
а тот динамически импортирует `render`. Разрыв через `_t` (lazy).

---

# ЧАСТЬ 2. Структура файла

| Байты | Что |
|---|---|
| 0 – 1 600 | преамбула + загрузчик модулей (rollup-commonjs) |
| 1 600 – 2 500 | **`ELK` класс** — обёртка над worker |
| 2 500 – 134 000 | GWT-рантайм (java.util, java.lang, emulation) |
| 134 000 – 150 000 | `org.eclipse.elk.*` — метаданные |
| 150 000 – 880 000 | **ELK core** — графы, свойства, layouts |
| 880 000 – 980 000 | Java-классы (runtime, exception, reflect) |
| 980 000 – 1 400 000 | **алгоритмы раскладки** (layered, force, stress, ...) |
| 1 400 000 – 1 437 800 | финал алгоритмов |
| 1 437 800 – 1 454 844 | **адаптер Mermaid** (`render`, `$oe`, `dbn`) |

**Реальная логика Skybridge — последние 17 КБ.** Остальное — ELK.

---

# ЧАСТЬ 3. Загрузчик модулей

```js
var Foe = {exports: {}};

(function (st, wt) {
  (function (yt) { st.exports = yt() })(function () {
    return function () {
      function yt(pi, Qi, ls) {
        function y(De, qe) {
          if (!Qi[De]) {
            if (!pi[De]) {
              var je = typeof TG == "function" && TG;
              if (!qe && je) return je(De, !0);
              if (rc) return rc(De, !0);
              var ir = new Error("Cannot find module '" + De + "'");
              throw ir.code = "MODULE_NOT_FOUND", ir
            }
            var Lr = Qi[De] = {exports: {}};
            pi[De][0].call(
              Lr.exports,
              function (Ln) {
                var Yl = pi[De][1][Ln];
                return y(Yl || Ln)
              },
              Lr, Lr.exports, yt, pi, Qi, ls
            )
          }
          return Qi[De].exports
        }

        for (var rc = typeof TG == "function" && TG, ks = 0; ks < ls.length; ks++)
          y(ls[ks]);

        return y
      }
      return yt
    }()
  })({
    1: [function (yt, pi, Qi) { ... }, { ... }],
    2: [function (yt, pi, Qi) { ... }, { ... }],
    ...
  })
})(Foe, Foe.exports)
```

## 3.1 Что это

**Классический browserify/rollup-commonjs загрузчик.** Реализует
`require()` в браузере.

**Механизм:**
- `pi` — карта модулей `{id: [factory, deps]}`
- `Qi` — кэш выполненных
- `y(id)` — загрузить модуль:
  1. Если в кэше — вернуть
  2. Если не в карте — попробовать `TG` (dynamic require) или `rc`
  3. Создать `{exports: {}}`
  4. Вызвать фабрику с `(require, module, exports, __webpack_require__, ...)`

**`TG`** — `dynamicRequire` (не сработает в браузере).

**`MODULE_NOT_FOUND`** — стандартная ошибка Node.

## 3.2 Проблема

```js
function TG(st) {
  throw new Error('Could not dynamically require "' + st + '". Please configure the dynamicRequireTargets or/and ignoreDynamicRequires option of @rollup/plugin-commonjs appropriately for this require call to work.')
}
```

**Любой динамический `require(x)` упадёт** — GWT не поддерживает.

**Практически:** ELK не использует динамические require, так что не
вызывается.

---

# ЧАСТЬ 4. Класс `ELK` — обёртка над Worker

```js
var rc = function () {
  function qi() {
    var Ni = this,
        je = arguments.length > 0 && arguments[0] !== void 0 ? arguments[0] : {},

        ir = je.defaultLayoutOptions,
        Lr = ir === void 0 ? {} : ir,

        Ln = je.algorithms,
        Yl = Ln === void 0 ? [
          "layered", "stress", "mrtree", "radial", "force",
          "disco", "sporeOverlap", "sporeCompaction", "rectpacking"
        ] : Ln,

        pl = je.workerFactory,
        ml = je.workerUrl;

    if (y(this, qi),
        this.defaultLayoutOptions = Lr,
        this.initialized = !1,
        typeof ml > "u" && typeof pl > "u")
      throw new Error("Cannot construct an ELK without both 'workerUrl' and 'workerFactory'.");

    var Zl = pl;
    typeof ml < "u" && typeof pl > "u" && (Zl = function (l7) { return new Worker(l7) });

    var r6 = Zl(ml);
    if (typeof r6.postMessage != "function")
      throw new TypeError("Created worker does not provide the required 'postMessage' function.");

    this.worker = new ks(r6),
    this.worker.postMessage({cmd: "register", algorithms: Yl})
  }
  ...
}()
```

## 4.1 9 алгоритмов раскладки

| Алгоритм | Что |
|---|---|
| `layered` | **слоистая** (hierarchical) — стандарт для flowchart |
| `stress` | минимизация напряжений |
| `mrtree` | дерево (multi-rooted) |
| `radial` | радиальная |
| `force` | силовая (пружины) |
| `disco` | компоненты связности |
| `sporeOverlap` | перекрывающиеся подграфы |
| `sporeCompaction` | сжатие подграфов |
| `rectpacking` | упаковка прямоугольников |

**Регистрация:** `worker.postMessage({cmd: "register", algorithms: Yl})`.

## 4.2 Регистрация в worker

**ELK работает в Web Worker.** Основной поток только отправляет граф,
получает результат.

**Зачем:** раскладка — дорогая операция (O(n²) или хуже для больших
графов). Worker не блокирует UI.

**Проблема:** `render` в `render-mermaid-svg` использует
`new xNe` (ELK) **без workerUrl** — значит `workerFactory` обязателен.
В `render-X3XFXER2` (этот файл) — `a7 = new xNe` без аргументов.

**Проверить:** если `workerUrl` и `workerFactory` не переданы →
`Error: Cannot construct an ELK without both 'workerUrl' and 'workerFactory'`.

**Значит:** `xNe` — другой класс (не `rc`). Скорее всего, синхронный
адаптер без worker.

---

# ЧАСТЬ 5. Свойства ELK

## 5.1 Перечисления

```js
function vf() {
  vf = F,                 // ← флаг инициализации
  ug = new XX("REGULAR", 0),
  xa = new XX("CRITICAL", 1)
}

function Ik() {
  Ik = F,
  DU = new nV("FIXED", 0),
  DO = new nV("CENTER_NODE", 1)
}

function LT() {
  LT = F,
  wfn = new RX("QUADRATIC", 0),
  KH = new RX("SCANLINE", 1)
}
```

**Типичный GWT-паттерн:**
- `ug` / `xa` — `LayerConstraint` (`REGULAR` / `CRITICAL`)
- `DU` / `DO` — `NodeLabelPlacement` (`FIXED` / `CENTER_NODE`)
- `wfn` / `KH` — `EdgeLabelSideSelection` (`QUADRATIC` / `SCANLINE`)

## 5.2 Списки алгоритмов

```js
function _Ln() {
  _Ln = F,
  dne = Se((C5(), S(T(mfn, 1), G, 322, 0, [t9, _j, pfn])))
}
```

**`Se(...)`** — создать Java-массив (`java.util.Arrays.asList`).
**`T(cls, 1)`** — конструктор массива.

Функции `_Ln`, `HLn`, `qLn`, `ULn`, `GLn`, `zLn`, `XLn`, `VLn`, `WLn`,
`JLn`, `QLn`, `YLn` — **инициализация статических массивов** ELK.

## 5.3 `org.eclipse.elk.*` — 358 вхождений

**Ключи опций:**
```
org.eclipse.elk.algorithm
org.eclipse.elk.direction
org.eclipse.elk.spacing.nodeNode
org.eclipse.elk.layered.crossingMinimization.strategy
org.eclipse.elk.layered.nodePlacement.strategy
org.eclipse.elk.layered.mergeEdges
org.eclipse.elk.hierarchyHandling
org.eclipse.elk.edgeRouting
org.eclipse.elk.padding
org.eclipse.elk.port.side
...
```

**Это полная схема опций ELK** — сотни параметров.

---

# ЧАСТЬ 6. Адаптер Mermaid

## 6.1 `dbn()` — `dir2ElkDirection`

```js
function dbn(st) {
  switch (st) {
    case "LR": return "RIGHT";
    case "RL": return "LEFT";
    case "TB":
    case "TD": return "DOWN";
    case "BT": return "UP";
    default: return "DOWN"
  }
}
```

**5 направлений Mermaid:**

| Mermaid | ELK |
|---|---|
| `LR` | `RIGHT` |
| `RL` | `LEFT` |
| `TB` | `DOWN` |
| `TD` | `DOWN` |
| `BT` | `UP` |
| иное | `DOWN` |

**`TB` и `TD` — синонимы** в Mermaid (top-bottom / top-down).

## 6.2 `$oe()` — `buildSubgraphLayoutOptions`

```js
function $oe(st, wt, yt) {
  const pi = {
    "spacing.baseValue": 30,
    "nodeLabels.placement": "[H_CENTER V_TOP, INSIDE]",
    "elk.layered.mergeEdges": wt?.mergeEdges,
    "nodePlacement.strategy": wt?.nodePlacementStrategy
  };

  return st.dir && (
    pi["elk.algorithm"] = yt,
    pi["elk.direction"] = dbn(st.dir),
    pi["elk.hierarchyHandling"] = "SEPARATE_CHILDREN"
  ), pi
}
```

### Параметры

- `st` — узел (с `dir`)
- `wt` — `config.elk` (общие настройки ELK)
- `yt` — алгоритм (`"layered"` и т.п.)

### Опции

```js
{
  "spacing.baseValue": 30,
  "nodeLabels.placement": "[H_CENTER V_TOP, INSIDE]",
  "elk.layered.mergeEdges": mergeEdges,
  "nodePlacement.strategy": nodePlacementStrategy
}
```

**`spacing.baseValue: 30`** — базовый отступ 30 px.

**`nodeLabels.placement: "[H_CENTER V_TOP, INSIDE]"`** — метки:
- `H_CENTER` — горизонтально по центру
- `V_TOP` — вертикально сверху
- `INSIDE` — внутри узла

### Только если `dir` задан

```js
st.dir && (
  pi["elk.algorithm"] = yt,
  pi["elk.direction"] = dbn(st.dir),
  pi["elk.hierarchyHandling"] = "SEPARATE_CHILDREN"
)
```

**`SEPARATE_CHILDREN`** — подграфы раскладываются **отдельно** от
родителя. Альтернатива `INCLUDE_CHILDREN` — все вместе.

**Зачем:** подграф с `direction: LR` внутри графа `TB` должен
раскладываться горизонтально.

## 6.3 `render()` — главная функция

```js
var GNe = Lc(async (st, wt, {
  common: yt,
  getConfig: pi,
  insertCluster: Qi,
  insertEdge: ls,
  insertEdgeLabel: y,
  insertMarkers: rc,
  insertNode: ks,
  interpolateToCurve: qi,
  labelHelper: Ni,
  log: je,
  positionEdgeLabel: ir
}, {algorithm: Lr}) => { ... }, "render")
```

### Параметры

- `st` — граф Mermaid (`{nodes, edges, direction, config, markers, diagramId, type}`
- `wt` — Svg-селектор (d3)
- Объект с колбэками:
  - `common` — common DB
  - `getConfig` — получить конфиг
  - `insertCluster` — вставить subgraph
  - `insertEdge` — вставить ребро
  - `insertEdgeLabel` — метку ребра
  - `insertMarkers` — маркеры (стрелки)
  - `insertNode` — узел
  - `interpolateToCurve` — преобразовать кривую
  - `labelHelper` — метка
  - `log`
  - `positionEdgeLabel` — позиция метки

### Внутренние функции

| Функция | Что |
|---|---|
| `pl` (`addVertex`) | добавить один узел |
| `ml` (`addVertices`) | добавить все узлы параллельно |
| `Zl` (`drawNodes`) | нарисовать узлы с offset |
| `r6` (`addSubGraphs`) | построить карту parent/children |
| `h7` (`getEdgeStartEndPoint`) | source/target ребра |
| `l7` (`calcOffset`) | offset кластера |
| `bD` (`computeStroke`) | стиль линии |
| `wD` (`getCurve`) | кривая |
| `gD` (`buildEdgeData`) | данные ребра |
| `pD` (`addEdges`) | все рёбра |
| `c6` (`setIncludeChildrenPolicy`) | `INCLUDE_CHILDREN` для предков |
| `_b` (`getEffectiveGroupWidth`) | ширина группы |
| `F` (`boundsFor`) | границы |
| `qE` (`approxEq`) | ≈ сравнение |
| `i4` (`isCenterApprox`) | центр ≈ |
| `UE` (`getCandidateBorderPoint`) | точка на границе |
| `GE` (`dropAutoCenterPoint`) | удалить auto-center |
| `mD` | поиск индекса |
| `vD` (`applyEndIntersectionIfNeeded`) | обрезка конца |
| `kD` (`cutter2`) | обрезка обеих сторон |
| `aD` | проверка внутри границ |
| `AG` | точка пересечения |
| `SG` | замена точки |

### `pl` — `addVertex`

```js
const pl = Lc(async (Fn, ae, Mt, se) => {
  const Xe = {width: 0, height: 0},
        at = pi();

  if (se.isGroup) {
    // 1. Группа (subgraph)
    const tt = {...se, domId: void 0, children: []};
    ae.children.push(tt);
    Ln[se.id] = tt;

    await ml(Fn, Mt, tt, se.id);            // рекурсия

    if (se.label) {
      const {shapeSvg: oh, bbox: _u} = await Ni(Fn, se, void 0, !0);
      Xe.width = _u.width;
      Xe.wrappingWidth = at.flowchart.wrappingWidth;
      Xe.height = _u.height - 2;
      Xe.labelNode = oh.node();
      oh.remove()
    } else Xe.width = 0, Xe.height = 0;

    tt.labelData = Xe;
    tt.domId = Fn
  } else {
    // 2. Обычный узел
    const tt = {
      id: se.id, width: se.width, height: se.height,
      label: se.label, isGroup: se.isGroup, shape: se.shape,
      padding: se.padding, cssClasses: se.cssClasses, cssStyles: se.cssStyles,
      look: se.look, parentId: se.parentId
    };
    ae.children.push(tt);
    Ln[se.id] = se;

    const oh = await ks(Fn, se, {config: at, dir: se.dir}),
          _u = oh.node().getBBox();
    tt.domId = oh;
    tt.width = _u.width;
    tt.height = _u.height
  }
}, "addVertex")
```

**Ключевое: размеры берутся из реального DOM.**

```js
const _u = oh.node().getBBox();
tt.width = _u.width;
tt.height = _u.height
```

**`getBBox()`** — после вставки в SVG. Значит узлы **сначала рисуются**,
потом измеряются, потом раскладываются.

**Почему:** текст метки может переноситься, шрифт разный — только реальный
DOM даёт правильный размер.

**`Xe.height = _u.height - 2`** — для групп минус 2 (компенсация границы).

### `Zl` — `drawNodes`

```js
const Zl = Lc(async (Fn, ae, Mt, se, Xe, at) => {
  await Promise.all(Mt.map(async function (tt) {
    if (!tt) return;

    Ln[oh = tt.id] ?? (Ln[oh] = {});
    Ln[tt.id].offset = {
      posX: tt.x + Fn,
      posY: tt.y + ae,
      x: Fn, y: ae,
      depth: at,
      width: Math.max(tt.width, tt.labels?.[0]?.width || 0),
      height: tt.height
    };

    if (tt.isGroup) {
      const ys = Xe.insert("g").attr("class", "subgraph"),
            fh = JSON.parse(JSON.stringify(tt));

      fh.x = tt.offset.posX + tt.width / 2;
      fh.y = tt.offset.posY + tt.height / 2;
      fh.width = Math.max(fh.width, tt.labelData.width);

      await Qi(ys, fh)
    } else {
      tt.domId.attr("transform",
        `translate(${tt.x + Fn + tt.width/2}, ${tt.y + ae + tt.height/2})`)
    }
  }));

  await Promise.all(Mt.map(async function (tt) {
    tt?.isGroup && await Zl(Fn + tt.x, ae + tt.y, tt.children, se, Xe, at + 1)
  }))
}, "drawNodes")
```

### Разбор

**1. Offset аккумулируется рекурсивно:**
```js
posX: tt.x + Fn,    // родительский offset + локальный
posY: tt.y + ae
```

**2. Обычный узел** — `transform="translate(cx, cy)"`. **Центр** узла,
не левый верхний угол.

**3. Группа** — `JSON.parse(JSON.stringify(tt))` (глубокий клон),
позиция центра, `width = max(width, labelWidth)`.

**4. Рекурсия для детей:**
```js
Zl(Fn + tt.x, ae + tt.y, tt.children, se, Xe, at + 1)
```
**`depth`** — уровень вложенности.

### `pD` — `addEdges`

```js
return await Promise.all(se.map(async function (_u) {
  const ys = _u.id;
  at[ys] === void 0 ? (at[ys] = 0) : (at[ys]++, ...);

  const fh = ys;
  _u.id = fh;

  const js = "LS_" + _u.start,              // LS_<id>
        Hh = "LE_" + _u.end,                // LE_<id>
        hp = pi();

  const V1 = gD(_u, {
    defaultStyle: tt,
    defaultLabelStyle: oh,
    defaultInterpolate: se.defaultInterpolate,
    confCurve: hp.curve
  }, yt);

  V1.id = fh;
  V1.classes = "flowchart-link " + js + " " + Hh;

  const n1 = await y(Xe, V1);

  const {source: r4, target: s6, sourceId: c4, targetId: b7} = h7(_u);

  ae.edges.push({
    ..._u,
    sources: [r4],
    targets: [s6],
    sourceId: c4,
    targetId: b7,
    labelEl: n1,
    labels: [{
      width: V1.width, height: V1.height,
      orgWidth: V1.width, orgHeight: V1.height,
      text: V1.label,
      layoutOptions: {
        "edgeLabels.inline": "true",
        "edgeLabels.placement": "CENTER"
      }
    }],
    edgeData: V1
  })
})), ae
```

**Классы:** `flowchart-link LS_<start> LE_<end>`.

**`LS_` / `LE_`** — используются в CSS для стилизации связей конкретного
узла:
```css
.LS_A path { stroke: red; }
```

**`edgeLabels.inline: "true"`** — метка внутри линии.
**`edgeLabels.placement: "CENTER"`** — по центру.

### `c6` — политика `INCLUDE_CHILDREN`

```js
function c6(Fn, ae) {
  const Mt = Ln[Fn];
  Mt && (
    Mt.layoutOptions === void 0 && (Mt.layoutOptions = {}),
    Mt.layoutOptions["elk.hierarchyHandling"] = "INCLUDE_CHILDREN",
    Mt.id !== ae && c6(Mt.parentId, ae)
  )
}
```

**Рекурсивно поднимается по предкам**, ставя `INCLUDE_CHILDREN` всем.

**Вызывается** когда ребро соединяет узлы **из разных** подграфов:
```js
if (Ln[Mt] && Ln[se] && Ln[Mt].parentId !== Ln[se].parentId) {
  const Xe = xoe(Mt, se, fp);      // LCA — lowest common ancestor
  c6(Mt, Xe);
  c6(se, Xe)
}
```

**`xoe`** — находит LCA двух узлов.

**Зачем:** если ребро идёт между подграфами, ELK должен раскладывать
их **вместе** (`INCLUDE_CHILDREN`), иначе линия оборвётся.

### Корневой граф

```js
let sh = {
  id: "root",
  layoutOptions: {
    "elk.hierarchyHandling": "INCLUDE_CHILDREN",
    "elk.algorithm": Lr,                        // "layered"
    "nodePlacement.strategy": st.config.elk?.nodePlacementStrategy,
    "elk.layered.mergeEdges": st.config.elk?.mergeEdges,
    "elk.direction": "DOWN",
    "spacing.baseValue": 40,
    "elk.layered.crossingMinimization.forceNodeModelOrder": ...,
    "elk.layered.considerModelOrder.strategy": ...,
    "elk.layered.unnecessaryBendpoints": !0,
    "elk.layered.cycleBreaking.strategy": ...,
    "elk.layered.wrapping.multiEdge.improveCuts": !0,
    "elk.layered.wrapping.multiEdge.improveWrappedEdges": !0,
    "elk.layered.edgeRouting.selfLoopDistribution": "EQUALLY",
    "elk.layered.mergeHierarchyEdges": !0
  },
  children: [],
  edges: []
};
```

### Разбор опций

| Опция | Значение | Что |
|---|---|---|
| `hierarchyHandling` | `INCLUDE_CHILDREN` | дети в том же layout |
| `algorithm` | `"layered"` | слоистый |
| `nodePlacement.strategy` | из конфига | где ставить узлы |
| `layered.mergeEdges` | из конфига | объединять рёбра |
| `direction` | `DOWN` (потом из `st.direction`) | направление |
| **`spacing.baseValue`** | **40** | базовый отступ |
| `crossingMinimization.forceNodeModelOrder` | из конфига | сохранять порядок |
| `considerModelOrder.strategy` | из конфига | учитывать порядок |
| `unnecessaryBendpoints` | `true` | убирать лишние изломы |
| `cycleBreaking.strategy` | из конфига | как рвать циклы |
| `wrapping.multiEdge.improveCuts` | `true` | оптимизация обёртки |
| `wrapping.multiEdge.improveWrappedEdges` | `true` | то же |
| `edgeRouting.selfLoopDistribution` | `"EQUALLY"` | петли |
| `mergeHierarchyEdges` | `true` | объединять рёбра иерархии |

**`spacing.baseValue: 40`** — корень. А в `$oe()` для подграфов — **30**.

### Вызов layout

```js
let u6;
try {
  u6 = await a7.layout(sh)
} catch (Fn) {
  throw je.error("APA01 ELK layout error:", Fn),
    je.error("APA01 elkGraph that caused error:", JSON.stringify(sh, null, 2)),
    Fn
}
```

**`a7 = new xNe`** — ELK-инстанс.

**При ошибке** — лог + `JSON.stringify` всего графа (для отладки).

**Затем:**
```js
await Zl(0, 0, u6.children, wt, ED, 0),     // нарисовать узлы
u6.edges?.map(Fn => { ... })                 // обработать рёбра
```

---

# ЧАСТЬ 7. `cutter2` — обрезка линий

## 7.1 `kD`

```js
const kD = Lc((Fn, ae, Mt) => {
  const se = F(Fn),       // границы источника
        Xe = F(ae);       // границы цели

  if (Mt.length === 0) return [];

  const at = [...Mt],                 // копия точек
        tt = at[0],                   // первая
        oh = at[at.length - 1];       // последняя

  // 1. найти первую точку ВНЕ границ источника
  let _u = -1;
  for (const [js, Hh] of at.entries())
    _u == -1 && aD(se, Hh) && (_u = js), aD(Xe, Hh);

  if (_u !== -1) {
    const js = at[_u], Hh = AG(Fn, se, js, tt);
    SG(at, "start", Hh)               // вставить/заменить первую
  }

  // 2. найти последнюю точку ВНЕ границ цели
  let ys = null, fh = -1;
  for (let js = at.length - 1; js >= 0; js--)
    if (aD(Xe, at[js])) { ys = at[js], fh = js; break }

  // 2a. если не нашли — взять предпоследнюю
  if (!ys && at.length > 1) (ys = at[at.length - 2], fh = at.length - 2);

  if (ys) {
    const js = AG(ae, Xe, ys, oh);
    SG(at, "end", js)
  }

  // 3. убрать слишком близкую точку
  if (at.length > 1) {
    const js = at[at.length - 1],
          Hh = at[at.length - 2],
          hp = Math.sqrt((js.x - Hh.x) ** 2 + (js.y - Hh.y) ** 2);

    hp < 2 && (at.pop())               // если < 2px — убрать
  }

  return at
}, "cutter2")
```

## 7.2 Что делает

**ELK возвращает точки ребра** — от центра источника до центра цели.
Но рисовать надо **от границы до границы** узла.

**`cutter2`** — находит пересечение линии с границей узла и **обрезает**.

### Шаг 1 — начало

```js
for (const [js, Hh] of at.entries())
  _u == -1 && aD(se, Hh) && (_u = js), aD(Xe, Hh);
```

**Найти первую точку ВНЕ границ источника.**

**`AG(Fn, se, js, tt)`** — точка пересечения отрезка `js→tt` с границей `se`.

**`SG(at, "start", Hh)`** — заменить начало.

### Шаг 2 — конец

**Поиск с конца** — последняя точка вне границ цели.

**Fallback:** если не нашли и точек > 1 — предпоследняя.

### Шаг 3 — тримминг

```js
hp < 2 && at.pop()
```

**Если последние две точки ближе 2 px** — убрать последнюю.

**Зачем:** ELK может дать почти совпадающие точки. Визуально — артефакт.

## 7.3 Вспомогательные

**`aD(границы, точка)`** — точка вне границ?
**`AG(узел, границы, точка1, точка2)`** — точка пересечения.
**`SG(массив, "start"|"end", точка)`** — вставить/заменить.

## 7.4 `mD` / `vD` / `GE` — упрощённые версии

```js
const mD = (Fn, ae, Mt) => {
  let se = -1;
  for (const [Xe, at] of Fn.entries()) if (aD(Mt, at)) { se = Xe; break }
  ...
}
```

**`mD`** — только начало.
**`vD`** — только конец.
**`GE`** — удалить auto-center точку.

---

# ЧАСТЬ 8. Стили рёбер

## 8.1 `HE` — карта маркеров

```js
const HE = {
  arrow_open:          ["arrow_open",  "arrow_open"],
  arrow_cross:         ["arrow_open",  "arrow_cross"],
  double_arrow_cross:  ["arrow_cross", "arrow_cross"],
  arrow_point:         ["arrow_open",  "arrow_point"],
  double_arrow_point:  ["arrow_point", "arrow_point"],
  arrow_circle:        ["arrow_open",  "arrow_circle"],
  double_arrow_circle: ["arrow_circle", "arrow_circle"]
};
```

**`[start, end]`** — маркеры с каждой стороны.

| Тип Mermaid | Start | End |
|---|---|---|
| `arrow_open` | open | open |
| `arrow_cross` | open | cross |
| `double_arrow_cross` | cross | cross |
| `arrow_point` | open | point |
| `double_arrow_point` | point | point |
| `arrow_circle` | open | circle |
| `double_arrow_circle` | circle | circle |

**`double_*`** — маркер с обеих сторон.

## 8.2 `bD` — `computeStroke`

```js
const bD = (Fn, ae, Mt) => {
  let se = "normal",        // толщина
      Xe = "solid",         // паттерн
      at = "",              // стиль
      tt = "";              // стиль метки

  return Fn === "dotted"
    ? (Xe = "dotted", at = "fill:none;stroke-width:2px;stroke-dasharray:3;")
    : Fn === "thick"
      ? (se = "thick", at = "stroke-width: 3.5px;fill:none;")
      : (at = ae ?? "fill:none;", Mt !== void 0 && (tt = Mt)),

    {thickness: se, pattern: Xe, style: at, labelStyle: tt}
}
```

**Три типа:**
- `"dotted"` → `stroke-dasharray: 3`, шириной 2px
- `"thick"` → ширина 3.5px
- иначе → `fill:none` (толщина по умолчанию)

## 8.3 `wD` — `getCurve`

```js
const wD = (Fn, ae, Mt) =>
  Fn !== void 0 ? qi(Fn, lbn)
  : ae !== void 0 ? qi(ae, lbn)
  : qi(Mt, lbn)
```

**Приоритет:** `edge.interpolate` → `defaultInterpolate` → `confCurve`.

**`qi`** — `interpolateToCurve` (из `transform-4I4J0RKg.js`).

## 8.4 `gD` — `buildEdgeData`

```js
const gD = (Fn, ae, Mt) => {
  const se = {style: "", labelStyle: ""};

  se.minlen = Fn.length || 1;
  se.text = Fn.label;
  se.arrowhead = Fn.type === "arrow_open" ? "none" : "normal";

  const Xe = HE[Fn.type] ?? HE.arrow_open;
  se.arrowTypeStart = Xe[0];
  se.arrowTypeEnd = Xe[1];

  se.startLabelRight = Fn.startLabelRight;
  se.endLabelLeft = Fn.endLabelLeft;

  const at = bD(Fn.stroke, ae.defaultStyle, ae.defaultLabelStyle);

  se.thickness = at.thickness;
  se.pattern = at.pattern;
  se.style = (se.style || "") + (at.style || "");
  se.labelStyle = (se.labelStyle || "") + (at.labelStyle || "");
  se.curve = wD(Fn.interpolate, ae.defaultInterpolate, ae.confCurve);

  return ((Fn?.text) ?? "") !== ""
    ? (se.arrowheadStyle = "fill: #333", se.labelpos = "c")
    : Fn.style !== void 0 && (se.arrowheadStyle = "fill: #333"),

    se.labelType = Fn.labelType,
    se.label = ((Fn?.text) ?? "").replace(Mt.lineBreakRegex, "\n"),
    Fn.style === void 0 && (se.style = se.style ?? "stroke: #333; stroke-width: 1.5px;fill:none;"),
    se.labelStyle = se.labelStyle.replace("color:", "fill:"),
    se
}
```

### Разбор

**`minlen: Fn.length || 1`** — минимальная длина (по умолчанию 1).

**`arrowhead: "none"` для `arrow_open`** — без заливки.

**`arrowheadStyle = "fill: #333"`** — если есть текст или явный стиль.

**`labelpos = "c"`** — метка по центру (если текст есть).

**`label.replace(lineBreakRegex, "\n")`** — нормализация переносов.

**Стиль по умолчанию:** `stroke: #333; stroke-width: 1.5px; fill: none`.

**`labelStyle.replace("color:", "fill:")`** — в SVG текст красится `fill`,
не `color`.

---

# ЧАСТЬ 9. Схема

```
render(graph, svg, callbacks, {algorithm: "layered"})
 │
 ├─ insertMarkers(g, markers, type, diagramId)
 │
 ├─ elkGraph = {
 │    id: "root",
 │    layoutOptions: {
 │      "elk.hierarchyHandling": "INCLUDE_CHILDREN",
 │      "elk.algorithm": "layered",
 │      "elk.direction": dir2ElkDirection(direction),
 │      "spacing.baseValue": 40,
 │      "elk.layered.unnecessaryBendpoints": true,
 │      "elk.layered.mergeHierarchyEdges": true,
 │      ...
 │    },
 │    children: [],
 │    edges: []
 │  }
 │
 ├─ addVertices(nodesGroup, nodes, elkGraph)
 │   └─ addVertex (для каждого):
 │       ├─ если isGroup:
 │       │    ├─ создать {children: []}
 │       │    ├─ rekursiv addVertices
 │       │    ├─ labelHelper() → bbox метки
 │       │    └─ labelData = {width, height, wrappingWidth, labelNode}
 │       └─ иначе:
 │            ├─ insertNode() → SVG-элемент
 │            ├─ getBBox() → размер  ← РЕАЛЬНЫЙ DOM
 │            └─ записать width/height
 │
 ├─ addEdges(graph, elkGraph, svg)
 │   ├─ buildEdgeData → стиль
 │   ├─ insertEdgeLabel → метка
 │   ├─ getEdgeStartEndPoint → source/target
 │   └─ labels: [{width, height, text, layoutOptions}]
 │
 ├─ для каждого узла с childrenById:
 │    ├─ labels = [labelData]
 │    ├─ width += 2 * padding
 │    ├─ layoutOptions = buildSubgraphLayoutOptions()
 │    └─ delete x, y, width, height     ← ELK вычислит
 │
 ├─ для каждого ребра между разными parent:
 │    ├─ LCA = xoe(source, target)
 │    └─ setIncludeChildrenPolicy(source, LCA), (target, LCA)
 │
 ├─ result = await ELK.layout(elkGraph)
 │
 ├─ drawNodes(0, 0, result.children, svg, subgraphsGroup, 0)
 │   └─ рекурсивно:
 │       ├─ offset = {posX: x+parentX, posY: y+parentY, depth}
 │       ├─ обычный: transform="translate(cx, cy)"
 │       └─ группа: insertCluster + рекурсия
 │
 └─ для каждого ребра result:
      ├─ cutter2(points) → обрезать по границам узлов
      ├─ trim точек < 2px
      └─ positionEdgeLabel()
```

---

# ЧАСТЬ 10. Слабые места

## 10.1 GWT-компиляция — 300+ КБ «мусора»

```js
function DT() { DT = F, x9 = new mt("org.eclipse.elk.labels.labelManager") }
function KLn() { KLn = F, hfn = new Kt("separateLayerConnections", (dA(), IH)) }
function vf() { vf = F, ug = new XX("REGULAR", 0), xa = new XX("CRITICAL", 1) }
```

**Сотни таких функций.** Ленивая инициализация статических полей Java.

**Каждый вызов проверяет флаг `X = F`.** При повторном — мгновенный возврат.

**Стоимость:** сотни функций в бандле, но вызываются один раз.

## 10.2 `getBBox()` требует реального DOM

```js
const _u = oh.node().getBBox();
tt.width = _u.width;
tt.height = _u.height
```

**Узлы сначала вставляются в SVG, потом измеряются.**

**Проблема:** если SVG скрыт (`visibility: hidden` в `render-mermaid-svg`),
`getBBox` всё равно работает (по спецификации), но **некоторые браузеры
могут вернуть нули**.

## 10.3 `JSON.stringify(sh, null, 2)` в обработке ошибок

```js
} catch (Fn) {
  throw je.error("APA01 ELK layout error:", Fn),
    je.error("APA01 elkGraph that caused error:", JSON.stringify(sh, null, 2)),
    Fn
}
```

**Сериализация всего графа при ошибке.** Для большого графа (100+ узлов) —
мегабайты в лог.

**Плюс:** может бросить (циклические ссылки в `domId`) → ошибка потеряется.

## 10.4 `cutter2` — эвристика

```js
if (!ys && at.length > 1) (ys = at[at.length - 2], fh = at.length - 2);
```

**Fallback:** если не нашли точку вне границ — взять предпоследнюю.

**Проблема:** для сложных рёбер (много изломов) обрезка может быть неточной.

## 10.5 `approxEq` с жёстким допуском

```js
const qE = (Fn, ae, Mt = 1e-6) => Math.abs(Fn - ae) < Mt
```

**`1e-6`** — очень строго. ELK возвращает координаты с плавающей точкой,
и разница может быть больше.

**Результат:** `isCenterApprox` может возвращать `false` там, где
центр совпадает.

## 10.6 `qq` в экспорте `c` из `client` — не связано

*(см. документ #16)*

## 10.7 `spacing.baseValue` разный

```js
$oe:  "spacing.baseValue": 30     // подграфы
root: "spacing.baseValue": 40     // корень
```

**Расхождение:** подграфы плотнее корня. Визуально может выглядеть
непоследовательно.

## 10.8 `edgeLabels.inline: "true"` всегда

```js
layoutOptions: {
  "edgeLabels.inline": "true",
  "edgeLabels.placement": "CENTER"
}
```

**Все метки — inline и по центру.** Нельзя настроить.

## 10.9 `minlen: Fn.length || 1`

**`Fn.length`** — длина массива? В Mermaid `edge.length` — это **число
сегментов** (сколько рангов пропустить).

**`|| 1`** — если `0` или `undefined` → `1`.

## 10.10 `default: return "DOWN"` в `dbn`

```js
default: return "DOWN"
```

**Неизвестное направление → DOWN.** Молча.

## 10.11 GWT-исключения в production

```js
ClassCastException
NullPointerException
```

**Java-исключения** в браузерном коде. Стек будет непонятным.

## 10.12 Worker для ELK — не используется в Mermaid-пути

**`render` создаёт `a7 = new xNe`** без `workerUrl`/`workerFactory`.

**Класс `rc` (ELK с worker) требует оба параметра.**

**Значит `xNe` — синхронный вариант.** Раскладка **блокирует main thread**.

**Для больших графов** — заметный лаг.

---

# ЧАСТЬ 11. Итог

## 11.1 Ключевые числа

| Что | Значение |
|---|---|
| Размер | 1 454 844 байта |
| Реальная логика (адаптер) | ~17 КБ (1.2%) |
| Алгоритмов ELK | 9 |
| Вхождений `org.eclipse.elk` | 358 |
| Вхождений `elk` | 376 |
| Вхождений `Node`/`Edge`/`Graph` | 187/214/186 |
| Направлений Mermaid | 5 (LR, RL, TB, TD, BT) |
| Типов маркеров | 7 |
| Типов линий | 3 (normal, dotted, thick) |
| `spacing.baseValue` (корень) | 40 |
| `spacing.baseValue` (подграф) | 30 |
| Порог тримминга | 2 px |
| Допуск `approxEq` | 1e-6 |

## 11.2 Главное

**1. Это ELK, скомпилированный через GWT из Java.**
`org.eclipse.elk.*` (358 вхождений), `ClassCastException`,
`NullPointerException`, паттерн `function X(){X=F, ...}` для статических
полей.

**2. 9 алгоритмов раскладки:** layered, stress, mrtree, radial, force,
disco, sporeOverlap, sporeCompaction, rectpacking.

**3. `render` — адаптер Mermaid → ELK, ~17 КБ логики.**

**4. Размеры узлов берутся из `getBBox()` — реальный DOM.**
Узлы сначала вставляются, потом измеряются, потом раскладываются.

**5. `dir2ElkDirection`: 5 направлений**, `TB`/`TD` → `DOWN`.

**6. `cutter2` — обрезка рёбер по границам узлов.**
ELK даёт точки от центра до центра, рисуем от границы до границы.

**7. `INCLUDE_CHILDREN` включается автоматически** при рёбрах между
подграфами (через LCA).

**8. `LS_<start>` / `LE_<end>`** — CSS-классы для стилизации связей
конкретного узла.

**9. `JSON.stringify(граф, null, 2)` при ошибке** — может выдать мегабайты.

**10. `xNe` — синхронный ELK без worker** — блокирует main thread.

**11. `approxEq` с допуском 1e-6 — слишком строго** для координат
с плавающей точкой.

**12. `spacing.baseValue` рассинхронизирован:** 40 для корня, 30 для
подграфов.
