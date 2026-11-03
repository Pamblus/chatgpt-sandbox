# `events-5a94e5eb.esm-DZJsuX-L.js` — полный разбор (React Three Fiber)

**Размер:** 159 828 байт
**URL:** `https://web-sandbox.oaiusercontent.com/assets/events-5a94e5eb.esm-DZJsuX-L.js`
**Роль:** **React Three Fiber v9.5.0** — React-рендерер для Three.js.
Несмотря на имя файла, это не «events», а весь R3F + его зависимости.

---

# ЧАСТЬ 1. Состав файла

## 1.1 Карта сегментов

| Байты | Что | Размер |
|---|---|---|
| 0 – 300 | преамбула, импорты | 300 |
| 300 – 1 330 | `use-sync-external-store-shim` | 1 КБ |
| 1 330 – 4 080 | `use-sync-external-store-shim/with-selector` | 2.7 КБ |
| 4 080 – 7 950 | **`scheduler.production.js`** | 3.9 КБ |
| 7 950 – 10 200 | `react-jsx-runtime.production.js` | 2.2 КБ |
| 10 200 – 21 400 | **R3F: core (utils, attach, store)** | 11 КБ |
| 21 400 – 25 130 | R3F: события, raycast | 3.7 КБ |
| 25 130 – 27 400 | `react-reconciler-constants` + package.json | 2.3 КБ |
| 27 400 – 147 900 | **`react-reconciler` (полный)** | **120 КБ** |
| 147 900 – 159 828 | **R3F: `createRoot`, Canvas, events** | 12 КБ |

**75% файла — `react-reconciler` (120 КБ).** Это ядро React, адаптированное
для рендеринга в не-DOM деревья.

## 1.2 Импорты

```js
import {r as J, R as vc} from "./index-C0fqnZRQ.js";        // React
import {L as tc, C as yp, R as Sp, U as kp, S as bc, W as Ep,
        a as yc, O as zp, P as wp, b as xp, c as Do, d as _p,
        e as Np, N as Cp, A as Tp, V as Sc, f as Ll, B as Ip,
        g as Pp, h as jp, i as Mp} from "./three.module-C846Yxd8.js";
import {j as kc} from "./main-BR1bOQyH.js";
```

| Источник | Что |
|---|---|
| `index-C0fqnZRQ.js` | React (8.5 КБ — только хуки) |
| `three.module-C846Yxd8.js` | Three.js (704 КБ) |
| `main-BR1bOQyH.js` | `j` = `D` = `Object` (UMD-хелпер) |

**Из Three.js импортируется 18 сущностей:**
`L` (Vector3?), `C`, `R`, `U`, `S` (ColorSpace), `W` (WebGLRenderer),
`a`, `O`, `P` (PerspectiveCamera), `b`, `c` (ShadowMap), `d`, `e`,
`N`, `A`, `V` (Raycaster-результат), `f` (**Vector3**), `B`, `g`, `h`,
`i` (Clock).

## 1.3 `Pm` — package.json внутри бандла

```js
var Pm = {
  name: "@react-three/fiber",
  version: "9.5.0",
  description: "A React renderer for Threejs",
  author: "Paul Henschel (https://github.com/drcmda)",
  license: "MIT",
  maintainers: ["Josh Ellis", "Cody Bennett", "Kris Baumgarter"],
  dependencies: {
    "@babel/runtime": "^7.17.8",
    "@types/webxr": "*",
    "base64-js": "^1.5.1",
    "buffer": "^6.0.3",
    "its-fine": "^2.0.0",
    "react-use-measure": "^2.1.7",
    "scheduler": "^0.27.0",
    "suspend-react": "^0.1.3",
    "use-sync-external-store": "^1.4.0",
    "zustand": "^5.0.3"
  },
  peerDependencies: {
    expo: ">=43.0",
    "expo-asset": ">=8.4",
    "expo-file-system": ">=11.0",
    "expo-gl": ">=11.0",
    react: ">=19 <19.3",
    "react-dom": ">=19 <19.3",
    "react-native": ">=0.78",
    three: ">=0.156"
  }
};
```

**React 19** (`>=19 <19.3`). **Three.js** `>=0.156`.

**НО:** в `transpile-Cld0n4VL.js` пиннинг `react: "18.3.1"` и
`three: "0.172.0"`. **R3F 9.5.0 требует React 19.**

**Это несовместимость:** если виджет импортирует `@react-three/fiber`,
он получит R3F 9.5.0 (из unpkg/jsdelivr), но React — 18.3.1 (пиннинг).
**R3F не заработает.**

---

# ЧАСТЬ 2. Зависимости изнутри

## 2.1 `use-sync-external-store-shim` (300 – 1 330)

```js
/** @license React
 * use-sync-external-store-shim.production.js */
var rl = J;
function Fp(i, o) {
  return i === o && (i !== 0 || 1/i === 1/o) || i !== i && o !== o
}
var Rp = typeof Object.is === "function" ? Object.is : Fp,
    Op = rl.useState,
    Lp = rl.useEffect,
    Dp = rl.useLayoutEffect,
    Up = rl.useDebugValue;
```

**`Fp`** — полифил `Object.is`:
- `i === o && (i !== 0 || 1/i === 1/o)` — строгое равенство + обработка `-0`
- `i !== i && o !== o` — оба `NaN`

**Зачем `1/i === 1/o`:** `Object.is(0, -0)` = `false`, а `0 === -0` = `true`.
`1/0 = Infinity`, `1/-0 = -Infinity` → различает.

**`useSyncExternalStore`** — API React 18 для подписки на внешние сторы
(zustand). Shim нужен для совместимости.

## 2.2 `with-selector` (1 330 – 4 080)

Версия `useSyncExternalStore` с селектором. Позволяет подписаться на
**часть** состояния:

```js
const value = useStore(state => state.foo);
```
Без селектора — ререндер на любое изменение. С селектором — только когда
`state.foo` меняется.

## 2.3 `scheduler.production.js` (4 080 – 7 950)

**Планировщик React.** Управляет приоритетами задач:

| Приоритет | Что |
|---|---|
| Immediate | синхронно |
| UserBlocking | ввод пользователя |
| Normal | обычные обновления |
| Low | низкий приоритет |
| Idle | когда свободно |

**`requestAnimationFrame` + `MessageChannel`** — механизм разбиения работы.

## 2.4 `react-jsx-runtime` (7 950 – 10 200)

**`jsx()` и `jsxs()`** — автоматический JSX (новый трансформ).

```js
ot.jsx(Ym, {store: v, children: S, onCreated: C, rootElement: i})
```

`ot` — объект `{jsx, jsxs, Fragment}`.

**`jsx` vs `jsxs`:** `jsxs` для **статических** наборов children
(массив), `jsx` — для одиночных.

## 2.5 `react-reconciler` (27 400 – 147 900, 120 КБ)

**Ядро React.** Алгоритм согласования (reconciliation), Fiber-дерево,
обновления.

**В R3F используется с кастомным `hostConfig`:**
```js
Al = Lm({
  isPrimaryRenderer: !1,
  warnsIfNotActing: !1,
  supportsMutation: !0,
  supportsPersistence: !1,
  supportsHydration: !1,
  ...
})
```

**`isPrimaryRenderer: false`** — этот рендерер не главный (есть `react-dom`).

**`supportsMutation: true`** — будет `appendChild`, `removeChild`,
`insertBefore` (для Three.js-объектов — `add`, `remove`).

**`supportsPersistence: false`** — не создаёт «снимки» дерева.

**Что даёт `react-reconciler`:**
- `createContainer` — создать Fiber-корень
- `updateContainer` — отрендерить элемент
- `flushSyncFromReconciler` — синхронный flush
- `createPortal` — портал в другой контейнер

---

# ЧАСТЬ 3. Утилиты сравнения — `ve`

```js
const ve = {
  obj: i => i === Object(i) && !ve.arr(i) && typeof i != "function",
  fun: i => typeof i == "function",
  str: i => typeof i == "string",
  num: i => typeof i == "number",
  boo: i => typeof i == "boolean",
  und: i => i === void 0,
  nul: i => i === null,
  arr: i => Array.isArray(i),

  equ(i, o, {arrays: s = "shallow", objects: d = "reference", strict: c = !0} = {}) {
    // 1. разные типы?
    if (typeof i != typeof o || !!i != !!o) return !1;

    // 2. примитивы
    if (ve.str(i) || ve.num(i) || ve.boo(i)) return i === o;

    const v = ve.obj(i);
    if (v && d === "reference") return i === o;

    const j = ve.arr(i);
    if (j && s === "reference") return i === o;

    // 3. одинаковые ссылки
    if ((j || v) && i === o) return !0;

    // 4. ключи i должны быть в o
    let C;
    for (C in i) if (!(C in o)) return !1;

    // 5. сравнение значений
    if (v && s === "shallow" && d === "shallow") {
      for (C in c ? o : i)
        if (!ve.equ(i[C], o[C], {strict: c, objects: "reference"})) return !1
    } else {
      for (C in c ? o : i) if (i[C] !== o[C]) return !1
    }

    // 6. проверка длины/пустоты
    if (ve.und(C)) {
      if (j && i.length === 0 && o.length === 0
          || v && Object.keys(i).length === 0 && Object.keys(o).length === 0) return !0;
      if (i !== o) return !1
    }

    return !0
  }
};
```

## 3.1 Что делает `equ`

**Гибкое сравнение с настройками:**
- `arrays: "shallow" | "reference"` — как сравнивать массивы
- `objects: "shallow" | "reference"` — как сравнивать объекты
- `strict: boolean` — сравнивать ли ключи в обе стороны

## 3.2 `r3f`-специфичные настройки

```js
const tl = {objects: "shallow", strict: !1};
```

**Объекты — поверхностно, строгость выключена.**

**Зачем:** Three.js-объекты (`props`) содержат ссылки на материалы,
текстуры. Глубокое сравнение невозможно (циклы) и бессмысленно.

## 3.3 Проблема

```js
for (C in c ? o : i) if (i[C] !== o[C]) return !1
```

**Если `strict: false`** — идёт по ключам `i`. Ключи, которые есть только
в `o`, **не проверяются**.

**То есть:** `{a: 1}` и `{a: 1, b: 2}` могут считаться равными.

---

# ЧАСТЬ 4. Обход дерева Three.js

## 4.1 `Uc()` — каталог объектов

```js
function Uc(i) {
  const o = {nodes: {}, materials: {}, meshes: {}};

  return i && i.traverse(s => {
    s.name && (o.nodes[s.name] = s),
    s.material && !o.materials[s.material.name] && (o.materials[s.material.name] = s.material),
    s.isMesh && !o.meshes[s.name] && (o.meshes[s.name] = s)
  }), o
}
```

**Обходит всё дерево и индексирует:**
- по имени → `nodes`
- материалы по имени → `materials`
- меши по имени → `meshes`

**Зачем:** доступ по строковому имени:
```js
useThree(state => state.scene.getObjectByName("cube"))
```
или через `nodes`.

**Проблема:** объекты **без имени** не индексируются. Безымянный меш —
недоступен.

## 4.2 `vm()` — dispose

```js
function vm(i) {
  i.type !== "Scene" && (i.dispose?.());

  for (const o in i) {
    const s = i[o];
    s?.type !== "Scene" && s?.dispose?.()
  }
}
```

**Освобождает ресурсы:** вызывает `dispose()` на объекте и всех его
свойствах.

**`type !== "Scene"`** — сцену не dispose-ит (она корневая).

**Проблема:** идёт по **всем** перечислимым свойствам, включая
не-объекты. `s?.dispose?.()` защищает, но проверка типа неполная.

## 4.3 `bm()` — очистка props

```js
const Ac = ["children", "key", "ref"];

function bm(i) {
  const o = {};
  for (const s in i) Ac.includes(s) || (o[s] = i[s]);
  return o
}
```

**Убирает React-специфичные props:** `children`, `key`, `ref`.

**Зачем:** эти props не должны попадать в Three.js-объекты.

---

# ЧАСТЬ 5. `__r3f` — метаданные объекта

## 5.1 `qr()` — создать/получить

```js
function qr(i, o, s, d) {
  const c = i;
  let v = c?.__r3f;

  return v || (
    v = {
      root: o,
      type: s,
      parent: null,
      children: [],
      props: bm(d),
      object: c,
      eventCount: 0,
      handlers: {},
      isHidden: !1
    },
    c && (c.__r3f = v)
  ), v
}
```

**Каждому Three.js-объекту создаётся `__r3f`:**
```js
{
  root,           // R3F-root
  type,           // тип элемента
  parent,         // родитель в R3F-дереве
  children,       // дети
  props,          // props без children/key/ref
  object,         // сам Three.js-объект
  eventCount,     // сколько обработчиков
  handlers,       // обработчики событий
  isHidden        // скрыт ли
}
```

## 5.2 `nl()` — получить store по объекту

```js
function nl(i) {
  return i.__r3f?.root.getState()
}
```

**Из любого Three.js-объекта можно получить Zustand-store корня.**

## 5.3 `Ul()` — резолв вложенного пути

```js
function Ul(i, o) {
  // 1. нет дефиса → прямой доступ
  if (!o.includes("-")) return {root: i, key: o, target: i[o]};

  // 2. есть как ключ целиком
  if (o in i) return {root: i, key: o, target: i[o]};

  // 3. разбить по дефисам
  let s = i;
  const d = o.split("-");

  for (const c of d) {
    if (typeof s != "object" || s === null) {
      if (s !== void 0) {
        const v = d.slice(d.indexOf(c)).join("-");
        return {root: s, key: v, target: void 0}
      }
      return {root: i, key: o, target: void 0}
    }
    o = c;
    i = s;
    s = s[o]
  }

  return {root: i, key: o, target: s}
}
```

**Что делает `attach="material-color"`:** проходит по пути
`object.material.color`.

**Три попытки:**
1. Прямое `obj["material-color"]`
2. `obj` целиком содержит такой ключ
3. Разбить по дефисам и идти вглубь

## 5.4 `Qr()` — attach

```js
const ac = /-\d+$/;

function Qr(i, o) {
  // 1. attach — строка
  if (ve.str(o.props.attach)) {
    // 1a. с числовым индексом в конце: "children-2"
    if (ac.test(o.props.attach)) {
      const c = o.props.attach.replace(ac, ""),
            {root: v, key: j} = Ul(i.object, c);
      Array.isArray(v[j]) || (v[j] = [])
    }

    // 1b. присвоить
    const {root: s, key: d} = Ul(i.object, o.props.attach);
    o.previousAttach = s[d];
    s[d] = o.object
  }
  // 2. attach — функция
  else ve.fun(o.props.attach) && (o.previousAttach = o.props.attach(i.object, o.object))
}
```

### Строковый `attach`

**`attach="material"`** → `parent.material = child`.

**`attach="material-color"`** → `parent.material.color = child`.

**`attach="children-2"`** → `parent.children[2] = child`
(с предварительным созданием массива).

**`previousAttach`** — сохраняет предыдущее значение для отката в `Hr()`.

### Функциональный `attach`

```js
attach={(parent, self) => { ... }}
```
Пользователь сам решает, куда прикрепить. `previousAttach` — возврат
функции (обычно отмена).

## 5.5 `Hr()` — detach

```js
function Hr(i, o) {
  if (ve.str(o.props.attach)) {
    const {root: s, key: d} = Ul(i.object, o.props.attach),
          c = o.previousAttach;
    c === void 0 ? delete s[d] : s[d] = c
  } else if (ve.fun(o.props.attach)) {
    o.previousAttach?.()
  }
}
```

**Откат при удалении элемента:**
- строковый → восстановить `previousAttach` или `delete`
- функциональный → вызвать возвращённую функцию

---

# ЧАСТЬ 6. Store — `xm()`

## 6.1 Полный код

```js
xm = (i, o) => {
  const s = _c((C, b) => {
    const I = new Ll, z = new Ll, S = new Ll;    // Vector3 ×3

    // вычисление viewport
    function _(F = b().camera, x = z, Y = b().size) {
      const {width: Z, height: K, top: V, left: ge} = Y,
            ke = Z / K;                             // aspect

      x.isVector3 ? S.copy(x) : S.set(...x);

      const oe = F.getWorldPosition(I).distanceTo(S);

      if (Oc(F))                                   // ортографическая?
        return {width: Z/F.zoom, height: K/F.zoom, top: V, left: ge,
                factor: 1, distance: oe, aspect: ke};

      const pe = F.fov * Math.PI / 180,             // fov → радианы
            ue = 2 * Math.tan(pe/2) * oe,           // высота в мире
            Qe = ue * (Z/K);                        // ширина в мире

      return {width: Qe, height: ue, top: V, left: ge,
              factor: Z/Qe, distance: oe, aspect: ke}
    }

    let w;                                          // таймер performance

    const P = F => C(x => ({performance: {...x.performance, current: F}})),
          le = new Sc;                              // Vector2 (pointer)

    return {
      set: C,
      get: b,
      gl: null,
      camera: null,
      raycaster: null,
      events: {priority: 1, enabled: !0, connected: !1},
      scene: null,
      xr: null,
      invalidate: (F = 1) => i(b(), F),
      advance: (F, x) => o(F, x, b()),
      legacy: !1,
      linear: !1,
      flat: !1,
      controls: null,
      clock: new Mp,                                // Clock
      pointer: le,
      mouse: le,                                    // ← ТО ЖЕ ОБЪЕКТ
      frameloop: "always",
      onPointerMissed: void 0,
      performance: {
        current: 1,
        min: .5,
        max: 1,
        debounce: 200,
        regress: () => { ... }
      },
      ...
    }
  })
}
```

## 6.2 Два объекта — `pointer` и `mouse`

```js
const le = new Sc;    // Vector2
...
pointer: le,
mouse: le,
```

**`pointer` и `mouse` — ОДИН объект.**

**Исторически** в старых версиях R3F `mouse` был отдельно. Сейчас —
алиас (deprecated).

**Последствие:** если пользователь делает `state.mouse.x = ...` — меняется
и `pointer`.

## 6.3 `_()` — вычисление viewport

**Для перспективной камеры:**
```
fov_rad = fov × π / 180
height = 2 × tan(fov_rad / 2) × distance
width  = height × (width_px / height_px)
```

**Классическая тригонометрия:** угол обзора → высота в мировых единицах
на расстоянии `distance`.

**Для ортографической:**
```
width  = width_px / zoom
height = height_px / zoom
```

**`factor: Z/Qe`** — для перспективы: `pixels / world_unit`. Полезно для
преобразования размеров.

**`distance`** — от камеры до целевой точки (по умолчанию `(0,0,0)`).

## 6.4 `performance.regress()` — адаптивное качество

```js
performance: {
  current: 1,
  min: .5,
  max: 1,
  debounce: 200,
  regress: () => {
    const F = b();
    w && clearTimeout(w);
    F.performance.current !== F.performance.min && ...
  }
}
```

**Механизм:** если кадры идут медленно — `current` снижается до `min: 0.5`.
Дебаунс 200 мс.

**Зачем:** адаптивное качество (снижать разрешение при лагах).

**`current`** — множитель для DPR.

## 6.5 `frameloop: "always"`

**Режимы:**
- `"always"` — рендер каждый кадр
- `"demand"` — только по `invalidate()`
- `"never"` — никогда

**`"always"` по умолчанию** — 60 FPS.

## 6.6 `invalidate` и `advance`

```js
invalidate: (F = 1) => i(b(), F),
advance: (F, x) => o(F, x, b()),
```

**`invalidate(frames)`** — запросить `frames` кадров.
**`advance(time, delta)`** — прокрутить вручную.

**`i` и `o`** — колбэки из `createRoot`.

---

# ЧАСТЬ 7. События

## 7.1 `Ho` — карта событий DOM

```js
const Ho = {
  onClick:              ["click",             !1],   // false = не passive
  onContextMenu:        ["contextmenu",       !1],
  onDoubleClick:        ["dblclick",          !1],
  onWheel:              ["wheel",             !0],   // passive
  onPointerDown:        ["pointerdown",       !0],
  onPointerUp:          ["pointerup",         !0],
  onPointerLeave:       ["pointerleave",      !0],
  onPointerMove:        ["pointermove",       !0],
  onPointerCancel:      ["pointercancel",     !0],
  onLostPointerCapture: ["lostpointercapture",!0]
};
```

**10 событий.**

**`passive: true`** — для `wheel` и всех pointer-событий. Значит
`preventDefault()` **не сработает**.

**`passive: false`** — только `click`, `contextmenu`, `dblclick`.

**Почему `onWheel` passive:** блокировка скролла ломает UX. R3F не пытается
её предотвратить.

## 7.2 `bh()` — стандартные события

```js
function bh(i) {
  const {handlePointer: o} = wm(i);

  return {
    priority: 1,
    enabled: !0,

    compute(s, d, c) {
      d.pointer.set(
        s.offsetX / d.size.width * 2 - 1,
        -(s.offsetY / d.size.height) * 2 + 1
      );
      d.raycaster.setFromCamera(d.pointer, d.camera)
    },

    connected: void 0,

    handlers: Object.keys(Ho).reduce((s, d) => (
      {...s, [d]: o(d)}
    ), {}),

    update: () => {
      const {events: d, internal: c} = i.getState();
      c.lastEvent?.current && d.handlers?.onPointerMove(c.lastEvent.current)
    },

    connect: s => {
      const {set: d, events: c} = i.getState();
      c.disconnect?.();
      d(v => ({events: {...v.events, connected: s}}));

      if (c.handlers)
        for (const v in c.handlers) {
          const j = c.handlers[v],
                [C, b] = Ho[v];
          s.addEventListener(C, j, {passive: b})
        }
    },

    disconnect: () => {
      const {set: s, events: d} = i.getState();
      if (d.connected) {
        if (d.handlers)
          for (const c in d.handlers) {
            const v = d.handlers[c],
                  [j] = Ho[c];
            d.connected.removeEventListener(j, v)
          }
        s(c => ({events: {...c.events, connected: void 0}}))
      }
    }
  }
}
```

### `compute()` — нормализация координат

```js
d.pointer.set(
  s.offsetX / d.size.width * 2 - 1,        // -1..1 по X
  -(s.offsetY / d.size.height) * 2 + 1     // 1..-1 по Y (инверсия)
);
```

**NDC (Normalized Device Coordinates):**
- X: `0 → -1`, `width → +1`
- Y: `0 → +1`, `height → -1` (**инверсия**, CSS Y вниз, WebGL Y вверх)

**`offsetX/offsetY`** — относительно **целевого элемента** (canvas).

**`raycaster.setFromCamera(pointer, camera)`** — луч из камеры через точку.

### `connect()` — навешивание

**Все 10 обработчиков** навешиваются на переданный элемент (`s`).

**`{passive: b}`** — из карты `Ho`.

### `disconnect()` — снятие

**По обратной карте.** `d.connected.removeEventListener(j, v)` —
тип из `Ho[c][0]`, обработчик из `d.handlers[c]`.

---

# ЧАСТЬ 8. Raycast и сортировка

## 8.1 Поиск пересечений (из 17 000)

```js
function d(b, I) {
  const z = i.getState(),
        S = new Set,                       // уже обработанные
        _ = [],                            // результат

        w = I ? I(z.internal.interaction) : z.internal.interaction;

  // 1. сбросить camera у всех raycaster
  for (let F = 0; F < w.length; F++) {
    const x = nl(w[F]);
    x && (x.raycaster.camera = void 0)
  }

  // 2. кастомный compute
  z.previousRoot || z.events.compute?.(b, z);

  // 3. пересечения по каждому root
  function P(F) {
    const x = nl(F);
    if (!x || !x.events.enabled || x.raycaster.camera === null) return [];

    if (x.raycaster.camera === void 0) {
      x.events.compute?.(b, x, x.previousRoot?.getState());
      x.raycaster.camera === void 0 && (x.raycaster.camera = null)
    }

    return x.raycaster.camera ? x.raycaster.intersectObject(F, true) : []
  }

  // 4. сортировка
  let le = w.flatMap(P).sort((F, x) => {
    const Y = nl(F.object),
          Z = nl(x.object);
    return !Y || !Z
      ? F.distance - x.distance
      : Z.events.priority - Y.events.priority || F.distance - x.distance
  })
  .filter(F => {
    const x = $r(F);                       // уникальный ID пересечения
    return S.has(x) ? !1 : (S.add(x), !0)
  });

  // 5. кастомный фильтр
  z.events.filter && (le = z.events.filter(le, z));

  // 6. подняться к eventCount-предкам
  for (const F of le) {
    let x = F.object;
    for (; x;) {
      x.__r3f?.eventCount && _.push({...F, eventObject: x}),
      x = x.parent
    }
  }

  // 7. захваченные pointer'ы
  if ("pointerId" in b && z.internal.capturedMap.has(b.pointerId))
    for (let F of z.internal.capturedMap.get(b.pointerId).values())
      S.has($r(F.intersection)) || _.push(F.intersection);

  return _
}
```

### Ключевые моменты

**1. `raycaster.camera = void 0`** — сброс. Каждый root сам заполнит.

**2. `x.raycaster.camera === null`** — «уже проверено, камеры нет» →
пропустить.

**3. Сортировка:**
```js
Z.events.priority - Y.events.priority || F.distance - x.distance
```
- Сначала по **приоритету** (убывание)
- При равенстве — по **расстоянию** (возрастание)

**4. Дедупликация через `Set`** по `$r(F)` — уникальный ключ пересечения.

**5. `eventCount`** — если у объекта есть обработчики, он попадает
в результат **вместе со всеми предками** с обработчиками.

**6. `capturedMap`** — pointer capture (если объект захватил указатель,
все события идут ему, даже вне пересечения).

## 8.2 Вызов обработчиков (из 18 000)

```js
function c(b, I, z, S) {
  if (b.length) {
    const _ = {stopped: !1};

    for (const w of b) {
      let P = nl(w.object);

      // 1. если нет __r3f — искать у предков
      P || w.object.traverseAncestors(le => {
        const ae = nl(le);
        if (ae) return P = ae, !1
      });

      if (P) {
        const {raycaster: le, pointer: ae, camera: F, internal: x} = P,
              Y = new Ll(ae.x, ae.y, 0).unproject(F),    // точка в мире
              Z = oe => x.capturedMap.get(oe)?.has(w.eventObject) ?? !1,
              K = oe => {                                 // setPointerCapture
                const pe = {intersection: w, target: I.target};
                x.capturedMap.has(oe)
                  ? x.capturedMap.get(oe).set(w.eventObject, pe)
                  : x.capturedMap.set(oe, new Map([[w.eventObject, pe]])),
                I.target.setPointerCapture(oe)
              },
              V = o => { ... };
        ...
      }
    }
  }
}
```

**`new Ll(ae.x, ae.y, 0).unproject(F)`** — обратное проецирование NDC
в мировые координаты (для `state.pointer` в 3D).

**`setPointerCapture`** — захват указателя. Все последующие события
идут захваченному объекту.

---

# ЧАСТЬ 9. `createRoot` — `dh()`

## 9.1 Полный код

```js
const _t = new Map;    // canvas → {fiber, store}

function dh(i) {
  const o = _t.get(i),
        s = o?.fiber,
        d = o?.store;

  o && console.warn("R3F.createRoot should only be called once!");

  const c = typeof reportError == "function" ? reportError : console.error,
        v = d || xm(Ko, gc),                       // store
        j = s || Al.createContainer(v, Nm, null, !1, null, "", c, c, c, null);

  o || _t.set(i, {fiber: j, store: v});

  let C, b, I = !1, z = null;                      // onCreated, camera, configured

  return {
    async configure(S = {}) { ... },
    render(S) { ... },
    unmount() { Xm(i) }
  }
}
```

### `_t` — глобальная карта

```js
const _t = new Map;   // canvas → {fiber, store}
```

**Один root на canvas.** Повторный `createRoot(canvas)` — предупреждение.

### `Al.createContainer`

```js
Al.createContainer(v, Nm, null, !1, null, "", c, c, c, null)
```

**`Nm = 1`** — `ConcurrentRoot`. Режим React 18+.

**`v`** — store. Он **и есть** Fiber-узел для reconciler? Нет — это
внешний store, к которому привязан root.

## 9.2 `configure()` — длинная функция

```js
async configure(S = {}) {
  let _;
  z = new Promise(se => _ = se);                   // промис готовности

  let {
    gl: w,
    size: P,
    scene: le,
    events: ae,
    onCreated: F,
    shadows: x = !1,
    linear: Y = !1,
    flat: Z = !1,
    legacy: K = !1,
    orthographic: V = !1,
    frameloop: ge = "always",
    dpr: ke = [1, 2],
    performance: oe,
    raycaster: pe,
    camera: ue,
    onPointerMissed: Qe
  } = S;

  const U = v.getState(),
        Ee = U.gl;

  // 1. WebGL-контекст
  if (!U.gl) {
    const se = {canvas: i, powerPreference: "high-performance",
                antialias: !0, alpha: !0},
          fe = typeof w == "function" ? await w(se) : w;
    ic(fe) ? Ee = fe : Ee = new Ep({...se, ...w});
    U.set({gl: Ee})
  }

  // 2. Raycaster
  let Ne = U.raycaster;
  Ne || U.set({raycaster: Ne = new yc});

  const {params: T, ...$} = pe || {};
  ve.equ($, Ne, tl) || at(Ne, {...$});
  ve.equ(T, Ne.params, tl) || at(Ne, {params: {...Ne.params, ...T}});

  // 3. Камера
  if (!U.camera || U.camera === b && !ve.equ(b, ue, tl)) {
    b = ue;
    const se = ue?.isCamera,
          fe = se ? ue : V ? new zp(0,0,0,0,.1,1e3) : new wp(75,0,.1,1e3);
    se || (fe.position.z = 5,
          ue && (at(fe, ue),
                fe.manual || ("aspect" in ue || "left" in ue || "right" in ue
                              || "bottom" in ue || "top" in ue)
                  && (fe.manual = !0, fe.updateProjectionMatrix())),
          !U.camera && !ue?.rotation && fe.lookAt(0,0,0));
    U.set({camera: fe});
    Ne.camera = fe
  }

  // 4. Сцена
  if (!U.scene) {
    let se;
    le?.isScene ? (se = le, qr(se, v, "", {}))
                : (se = new xp, qr(se, v, "", {}), le && at(se, le));
    U.set({scene: se})
  }

  // 5. События
  ae && !U.events.handlers && U.set({events: ae(v)});

  // 6. Размер
  const D = Wm(i, P);
  ve.equ(D, U.size, tl) || U.setSize(D.width, D.height, D.top, D.left);

  // 7. DPR
  ke && U.viewport.dpr !== Dc(ke) && U.setDpr(ke);

  // 8. Frameloop
  U.frameloop !== ge && U.setFrameloop(ge);

  // 9. onPointerMissed
  U.onPointerMissed || U.set({onPointerMissed: Qe});

  // 10. Performance
  oe && !ve.equ(oe, U.performance, tl)
    && U.set(se => ({performance: {...se.performance, ...oe}}));

  // 11. XR
  if (!U.xr) { ... }

  // 12. Тени
  if (Ee.shadowMap) {
    const se = Ee.shadowMap.enabled,
          fe = Ee.shadowMap.type;
    if (Ee.shadowMap.enabled = !!x, ve.boo(x)) Ee.shadowMap.type = Do;
    else if (ve.str(x)) {
      const sn = {basic: Ip, percentage: Pp, soft: Do, variance: jp};
      Ee.shadowMap.type = sn[x] ?? Do
    } else ve.obj(x) && Object.assign(Ee.shadowMap, x);
    (se !== Ee.shadowMap.enabled || fe !== Ee.shadowMap.type)
      && (Ee.shadowMap.needsUpdate = !0)
  }

  // 13. Цвет
  return _p.enabled = !K,
    I || (Ee.outputColorSpace = Y ? Np : bc,
          Ee.toneMapping = Z ? Cp : Tp),
    U.legacy !== K && U.set(() => ({legacy: K})),
    U.linear !== Y && U.set(() => ({linear: Y})),
    U.flat !== Z && U.set(() => ({flat: Z})),
    w && !ve.fun(w) && !ic(w) && !ve.equ(w, Ee, tl) && at(Ee, w),

    C = F, I = !0, _(), this
}
```

### Разбор ключевых частей

**WebGL-контекст по умолчанию:**
```js
{canvas, powerPreference: "high-performance", antialias: true, alpha: true}
```

**`powerPreference: "high-performance"`** — дискретная видеокарта
(на ноутбуках).

**`alpha: true`** — прозрачный фон. Критично для виджетов (прозрачный
фон контейнера).

**Камера по умолчанию:**
```js
new PerspectiveCamera(75, 0, 0.1, 1000)   // fov, aspect, near, far
camera.position.z = 5
camera.lookAt(0, 0, 0)
```

**`fov: 75`**, `near: 0.1`, `far: 1000`, `z: 5`.

**Ортографическая:**
```js
new OrthographicCamera(0, 0, 0, 0, 0.1, 1000)
```

**Все границы `0`** — потом обновятся по размеру.

**Тени:**
```js
const sn = {basic: Ip, percentage: Pp, soft: Do, variance: jp};
```
- `shadows={true}` → `Do` (PCFSoftShadowMap)
- `shadows="basic"` → `Ip`
- `shadows="percentage"` → `Pp`
- `shadows="soft"` → `Do`
- `shadows="variance"` → `jp`
- `shadows={object}` → `Object.assign`

**Цвет:**
```js
Ee.outputColorSpace = Y ? Np : bc      // linear ? LinearSRGB : SRGB
Ee.toneMapping = Z ? Cp : Tp           // flat ? NoToneMapping : ACESFilmic
```

**`linear: true`** → Linear SRGB (без гамма-коррекции).
**`flat: true`** → без tone mapping.

**`_p.enabled = !K`** — `colorManagement.enabled = !legacy`.

## 9.3 `render()`

```js
render(S) {
  return !I && !z && this.configure(),
    z.then(() => {
      Al.updateContainer(
        ot.jsx(Ym, {store: v, children: S, onCreated: C, rootElement: i}),
        j, null, () => {}
      )
    }),
    v
}
```

**Автоконфигурация:** если `configure()` не вызван — вызвать.

**`ot.jsx(Ym, {...})`** — рендер внутреннего компонента.

**Возвращает `v` (store)**, не `this`.

## 9.4 `Ym` — внутренний компонент

```js
function Ym({store: i, children: o, onCreated: s, rootElement: d}) {
  return Gr(() => {
    const c = i.getState();
    c.set(v => ({internal: {...v.internal, active: !0}}));
    s && s(c);
    i.getState().events.connected || c.events.connect?.(d)
  }, []), ot.jsx(ni.Provider, {value: i, children: o})
}
```

**`Gr(...)`** — `useIsomorphicLayoutEffect` (имена минифицированы).

**Что делает:**
1. `internal.active = true`
2. `onCreated(store.getState())` — колбэк
3. `events.connect(rootElement)` — если не подключено
4. Оборачивает в `Provider` (`ni` = R3F-контекст)

## 9.5 `Xm()` — unmount

```js
function Xm(i, o) {
  const s = _t.get(i),
        d = s?.fiber;

  if (d) {
    const c = s.store.getState();
    c && (c.internal.active = !1);

    Al.updateContainer(null, d, null, () => {
      c && setTimeout(() => {
        try {
          c.events.disconnect?.(),
          c.gl?.renderLists?.dispose?.(),
          c.gl?.forceContextLoss?.(),
          c.gl?.xr && c.xr.disconnect(),
          vm(c.scene),                       // dispose всех объектов
          _t.delete(i),
          o && o(i)
        } catch {}
      }, 500)                                 // ← 500 мс задержка
    })
  }
}
```

### Что делает

1. `internal.active = false`
2. `updateContainer(null, ...)` — размонтировать React-дерево
3. **Через 500 мс:**
   - `events.disconnect()`
   - `gl.renderLists.dispose()`
   - `gl.forceContextLoss()` — освободить WebGL-контекст
   - `xr.disconnect()`
   - `vm(scene)` — dispose всех объектов
   - `_t.delete(canvas)`

**`setTimeout(..., 500)`** — задержка. Зачем: дать React завершить
очистку, дать браузеру освободить ресурсы.

**`try {} catch {}`** — молча.

## 9.6 Проблема `unmount`

**`forceContextLoss()`** — **жёсткое** освобождение WebGL-контекста.

**Лимит браузера:** ~16 WebGL-контекстов на страницу. Если создавать/
уничтожать много — можно упереться.

**`forceContextLoss`** помогает восстановить лимит, но не всегда.

**Плюс 500 мс задержки:** если создать новый root на том же canvas
в течение 500 мс — `_t` ещё содержит старый, `createRoot` предупредит.

---

# ЧАСТЬ 10. `Canvas`-компонент — `ph()`

```js
function ph(i, o, s) {
  return ot.jsx(Gm, {children: i, container: o, state: s})
}
```

**`ph`** — экспорт `s`. Обёртка над `Gm`.

## 10.1 `Gm` — «Canvas» без canvas

```js
function Gm({state: i = {}, children: o, container: s}) {
  const {events: d, size: c, ...v} = i,
        j = ti(),                        // useThree()
        [C] = J.useState(() => new yc),   // Raycaster
        [b] = J.useState(() => new Sc),   // Vector2 (pointer)

  const I = Lc((S, _) => {
    let w;
    if (_.camera && c) {
      const P = _.camera;
      w = S.viewport.getCurrentViewport(P, new Ll, c);
      P !== S.camera && $c(P, c)
    }
    return {
      ...S,
      ..._,
      scene: s,
      raycaster: C,
      pointer: b,
      mouse: b,
      previousRoot: j,
      events: {...S.events, ..._.events, ...d},
      size: {...S.size, ...c},
      viewport: {...S.viewport, ...w},
      setEvents: P => _.set(le => ({...le, events: {...le.events, ...P}}))
    }
  });

  const z = J.useMemo(() => {
    const S = _c((w, P) => ({...v, set: w, get: P})),
          _ = w => S.setState(P => I.current(w, P));
    return _(j.getState()), j.subscribe(_), S
  }, [j, s]);

  return ot.jsx(ot.Fragment, {
    children: Al.createPortal(
      ot.jsx(ni.Provider, {value: z, children: o}),
      z, null
    )
  })
}
```

### Что делает

**Создаёт виртуальный R3F-контекст** — без `canvas`.

**`createPortal(children, z, null)`** — рендерит детей в портал
с новым store.

**Зачем:** позволяет вложенные R3F-сцены. Например, `<Canvas>` внутри
`<Canvas>` — каждая со своей сценой, но общим React-деревом.

**`previousRoot: j`** — ссылка на родительский store.

**`subscribe`** — подписка на родительский store → обновление.

## 10.2 Порядок параметров

**`ph(i, o, s)`** → `Gm({children: i, container: o, state: s})`.

**`container: o`** — это сцена (Three.js `Scene`).

**`state: s`** — начальный store.

---

# ЧАСТЬ 11. Hooks

## 11.1 `ti()` — `useThree`

```js
ti = () => J.useContext(ni)
```

**`useThree()`** — получить store R3F-контекста.

**`ni = J.createContext(null)`** — контекст.

**Без селектора** — весь store. С селектором — через `Lc` (useStore).

## 11.2 `Lc()` — `useStore`

```js
Lc = ((i, o) => ...)    // useStoreWithSelector
```

**`useStore(selector, equalityFn)`** — из zustand. Подписка на часть
состояния.

## 11.3 `hh` / `gh` / `vh` — подписка на фазы кадра

```js
const Xc = new Set,    // before
      Gc = new Set,    // after
      Zc = new Set;    // tail

const hh = i => ri(i, Xc),    // subscribers("before")
      gh = i => ri(i, Gc),    // subscribers("after")
      vh = i => ri(i, Zc);    // subscribers("tail")

function ri(i, o) {
  const s = {callback: i};
  return o.add(s), () => void o.delete(s)
}
```

**Три набора подписчиков** с фазами. `ri` возвращает функцию отписки.

## 11.4 `Bo()` / `Dl()` — вызов подписчиков

```js
function Bo(i, o) {
  if (i.size) for (const {callback: s} of i.values()) s(o)
}

function Dl(i, o) {
  switch (i) {
    case "before": return Bo(Xc, o);
    case "after":  return Bo(Gc, o);
    case "tail":   return Bo(Zc, o)
  }
}
```

**`Dl(phase, frame)`** — вызвать всех подписчиков фазы.

## 11.5 `Go()` — один кадр

```js
function Go(i, o, s) {
  let d = o.clock.getDelta();

  o.frameloop === "never" && typeof i == "number"
    && (d = i - o.clock.elapsedTime,
        o.clock.oldTime = o.clock.elapsedTime,
        o.clock.elapsedTime = i);

  Vo = o.internal.subscribers;

  for (let c = 0; c < Vo.length; c++) {
    qo = Vo[c];
    qo.ref.current(qo.store.getState(), d, s)
  }

  return !o.internal.priority && o.gl.render && o.gl.render(o.scene, o.camera),
    o.internal.frames = Math.max(0, o.internal.frames - 1),
    o.frameloop === "always" ? 1 : o.internal.frames
}
```

### Что делает

**1. `clock.getDelta()`** — время с прошлого кадра.

**2. `frameloop === "never"`** — время задаётся вручную (`i`).

**3. Подписчики `internal.subscribers`** — все `useFrame`-колбэки.
Вызываются с `(state, delta, frame)`.

**4. Рендер:**
```js
!o.internal.priority && o.gl.render && o.gl.render(o.scene, o.camera)
```

**`internal.priority`** — если у какого-то `useFrame` есть `priority`,
автоматический рендер **отключается** (пользователь рендерит сам).

**5. `frames`** — уменьшить. При `"always"` — вернуть 1 (продолжать).

## 11.6 `Kc()` — глобальный цикл

```js
let Wr = !1, Zo = !1, Qo, hc, ll;

function Kc(i) {
  hc = requestAnimationFrame(Kc);     // ← перезапуск сразу
  Wr = !0;
  Qo = 0;

  Dl("before", i);
  Zo = !0;

  for (const s of _t.values()) {       // все roots
    ll = s.store.getState();
    ll.internal.active
      && (ll.frameloop === "always" || ll.internal.frames > 0)
      && !ll.gl.xr?.isPresenting
      && (Qo += Go(i, ll))
  }

  if (Zo = !1, Dl("after", i), Qo === 0)
    return Dl("tail", i), Wr = !1, cancelAnimationFrame(hc)
}
```

### Разбор

**`hc = requestAnimationFrame(Kc)`** — **первым делом** планируется
следующий кадр. Даже если текущий ничего не сделает.

**`Dl("before", i)`** — подписчики «before» до всех roots.

**Цикл по всем roots** (`_t.values()`).

**Условие активности:**
- `internal.active`
- и (`frameloop === "always"` или `frames > 0`)
- и не в XR-презентации

**`Qo += Go(...)`** — суммарное количество кадров.

**Если `Qo === 0`** — все roots неактивны → остановить цикл:
`cancelAnimationFrame(hc)`, `Wr = false`.

## 11.7 `Ko()` — invalidate

```js
function Ko(i, o = 1) {
  if (!i) return _t.forEach(d => Ko(d.store.getState(), o));

  if (i.gl.xr?.isPresenting || !i.internal.active || i.frameloop === "never")
    return;

  o > 1 ? i.internal.frames = Math.min(60, i.internal.frames + o)
        : Zo ? i.internal.frames = 2 : i.internal.frames = 1;

  Wr || (Wr = !0, requestAnimationFrame(Kc))
}
```

### Что делает

**`frames` увеличивается:**
- `o > 1` → `min(60, frames + o)`
- `Zo` (внутри кадра) → `2`
- иначе → `1`

**`Zo = 2`** — если `invalidate` вызван **во время** кадра — нужен ещё
один кадр (текущий уже не успеет).

**`Wr ||`** — запустить цикл, если не запущен.

**Лимит 60** — защита от лавины.

## 11.8 `gc()` — `advance`

```js
function gc(i, o = !0, s, d) {
  if (o && Dl("before", i), s) Go(i, s, d);
  else for (const c of _t.values()) Go(i, c.store.getState());
  o && Dl("after", i)
}
```

**Ручной проход кадра.** `advance(timestamp)`.

---

# ЧАСТЬ 12. Экспорты — полная карта

```js
export {
  Al as A,   // react-reconciler (createContainer, updateContainer, createPortal, flushSync...)
  rh as B,   // ?
  sh as C,   // ?
  ch as D,   // ?
  ah as E,   // ?
  ih as F,   // ?
  qc as G,   // suspend-react (preload, clear)
  ti as H,   // useThree
  uh as I,   // ?
  em as J,   // useStore-something
  am as K,   // ?
  om as L,   // suspend-react preload
  im as M,   // suspend-react clear
  _t as _,   // Map canvas → {fiber, store}
  Lc as a,   // useStore (zustand)
  Gr as b,   // useIsomorphicLayoutEffect
  dh as c,   // createRoot
  Xm as d,   // unmountComponentAtNode
  fh as e,   // useFrame
  bh as f,   // createPointerEvents
  nh as g,   // ?
  gh as h,   // subscribe("after")
  th as i,   // useIsomorphicLayoutEffect2?
  ot as j,   // jsx-runtime {jsx, jsxs, Fragment}
  hh as k,   // subscribe("before")
  vh as l,   // subscribe("tail")
  fm as m,   // ?
  gc as n,   // advance
  at as o,   // applyProps (Object.assign-хелпер)
  Uc as p,   // buildObjectCatalog (nodes/materials/meshes)
  ni as q,   // R3F Context
  wm as r,   // createEventsHandler
  ph as s,   // Canvas (virtual)
  eh as t,   // useLoader?
  lh as u,   // ?
  vm as v,   // dispose
  Dl as w,   // flushSubscribers(phase)
  mh as x,   // flushSyncFromReconciler
  nl as y,   // getRootState
  Ko as z    // invalidate
};
```

---

# ЧАСТЬ 13. Схема

```
createRoot(canvas)
 │
 ├─ _t.get(canvas) — уже есть? warn
 ├─ store = xm(invalidate, advance)     ← zustand
 ├─ fiber = Al.createContainer(store, ConcurrentRoot, ...)
 ├─ _t.set(canvas, {fiber, store})
 │
 └─ → { configure, render, unmount }
      │
      ├─ configure({
      │    gl, size, scene, events, onCreated,
      │    shadows, linear, flat, legacy, orthographic,
      │    frameloop, dpr, performance, raycaster, camera, onPointerMissed
      │  })
      │   ├─ gl = new WebGLRenderer({canvas, high-performance, antialias, alpha})
      │   ├─ raycaster = new Raycaster
      │   ├─ camera = new PerspectiveCamera(75, 0, 0.1, 1000)  [или ortho]
      │   ├─ scene = new Scene
      │   ├─ events = createPointerEvents(store)
      │   ├─ size = из parent.getBoundingClientRect()
      │   ├─ dpr = clamp(1..2)
      │   ├─ frameloop
      │   ├─ XR-хуки
      │   ├─ shadowMap
      │   └─ outputColorSpace, toneMapping
      │
      ├─ render(children)
      │   └─ Al.updateContainer(
      │        jsx(Ym, {store, children, onCreated, rootElement}),
      │        fiber, null, () => {}
      │      )
      │      └─ Ym:
      │           ├─ internal.active = true
      │           ├─ onCreated(state)
      │           └─ events.connect(canvas)
      │
      └─ unmount()
          ├─ internal.active = false
          ├─ updateContainer(null, ...)
          └─ setTimeout(500):
              ├─ events.disconnect()
              ├─ gl.renderLists.dispose()
              ├─ gl.forceContextLoss()
              ├─ xr.disconnect()
              ├─ dispose(scene)
              └─ _t.delete(canvas)
```

## 13.1 Кадровый цикл

```
requestAnimationFrame(Kc)
 │
 ├─ Dl("before", time)          ← подписчики "before"
 ├─ for each root in _t:
 │    ├─ active && (always || frames>0) && !xr.isPresenting
 │    └─ Qo += Go(time, state)
 │         ├─ delta = clock.getDelta()
 │         ├─ for each useFrame subscriber:
 │         │    callback(state, delta, frame)
 │         ├─ if (!internal.priority) gl.render(scene, camera)
 │         └─ frames--
 ├─ Dl("after", time)           ← подписчики "after"
 └─ if (Qo === 0):
      ├─ Dl("tail", time)       ← подписчики "tail"
      └─ cancelAnimationFrame
```

---

# ЧАСТЬ 14. Слабые места

## 14.1 R3F 9.5.0 требует React 19, а пиннинг — 18.3.1

```js
peerDependencies: {react: ">=19 <19.3", "react-dom": ">=19 <19.3"}
```

**В `transpile-Cld0n4VL.js`:**
```js
yGe = {react: "18.3.1", "react-dom": "18.3.1", ...}
```

**`import {Canvas} from "@react-three/fiber"`** → R3F 9.5.0 с jsdelivr
+ React 18.3.1 → **несовместимость**.

**R3F 9 использует API React 19** (`use()`, `Activity`, новые хуки).
С React 18 упадёт.

**Обход:** в `yGe` **не** указан `@react-three/fiber`?.. указан:
```js
"@react-three/fiber": "8.17.7"
```

**Версия 8.17.7** в пиннинге — совместима с React 18.

**Но:** этот файл (R3F 9.5.0) — **вшит** в бандл Skybridge, а не грузится
с CDN.

**Значит:** если виджет использует вшитый R3F (через `prepareEnvironment`),
а React — 18.3.1 с CDN → несовместимость.

**Проверить:** как соотносятся `index-C0fqnZRQ.js` (React) и этот файл.

## 14.2 `pointer` и `mouse` — один объект

```js
const le = new Sc;
...
pointer: le,
mouse: le,
```

**Мутация одного меняет другой.** В старых версиях R3F они были разные.

## 14.3 `ve.equ` при `strict: false` не проверяет лишние ключи

```js
for (C in c ? o : i) if (i[C] !== o[C]) return !1
```

**`c = strict`.** При `strict: false` идёт по ключам `i`. Ключи только
в `o` — не проверяются.

**`{a: 1}` ≡ `{a: 1, b: 2}`** — может считаться равными.

## 14.4 `Uc` не индексирует безымянные объекты

```js
s.name && (o.nodes[s.name] = s)
```

**Объекты без `name` недоступны через `nodes`.**

## 14.5 `vm` (dispose) идёт по всем перечислимым свойствам

```js
for (const o in i) {
  const s = i[o];
  s?.type !== "Scene" && s?.dispose?.()
}
```

**Включая вложенные массивы, материалы, текстуры.** Может двойной
`dispose` (объект и его материал, потом материал отдельно).

## 14.6 `Kc` планирует rAF даже когда всё неактивно

```js
function Kc(i) {
  hc = requestAnimationFrame(Kc);    // ← всегда
  ...
  if (Qo === 0) return ... cancelAnimationFrame(hc)
}
```

**Каждый кадр заново планируется**, потом отменяется. Микро-накладные
расходы, но не критично.

## 14.7 `Ko` лимит 60 кадров

```js
o > 1 ? i.internal.frames = Math.min(60, i.internal.frames + o) : ...
```

**`invalidate(1000)` не даст 1000 кадров** — только 60.

## 14.8 `unmount` с задержкой 500 мс

```js
setTimeout(() => { ... }, 500)
```

**Если создать новый root на том же canvas в течение 500 мс** —
`_t` ещё содержит старый → предупреждение, утечка.

## 14.9 `forceContextLoss` — лимит WebGL-контекстов

**~16 контекстов на страницу.** При частом создании/удалении R3F-сцен
можно упереться, даже с `forceContextLoss`.

## 14.10 `powerPreference: "high-performance"`

```js
{canvas, powerPreference: "high-performance", antialias: true, alpha: true}
```

**На ноутбуках** — задействует дискретную видеокарту → **расход батареи**.

**Для виджета в чате** — избыточно.

## 14.11 `connect()` навешивает все 10 обработчиков

```js
for (const v in c.handlers) {
  const [C, b] = Ho[v];
  s.addEventListener(C, j, {passive: b})
}
```

**Даже если у виджета нет ни одного `onPointerDown`** — обработчик
навешан. Работает проверка `eventCount` при raycast, но listener
всё равно есть.

## 14.12 `events.compute` вызывается даже без roots

```js
z.previousRoot || z.events.compute?.(b, z);
```

**Если `previousRoot` не задан** — compute всё равно вызывается.

## 14.13 `gm` (createContainer) с `Nm = 1`

```js
const Nm = 1;    // ConcurrentRoot
```

**Concurrent-режим всегда.** Нет опции legacy-рендеринга.

## 14.14 `Xm` — `try {} catch {}` молча

```js
try { ... } catch {}
```

**Если `forceContextLoss` бросит** — не узнать.

---

# ЧАСТЬ 15. Итог

## 15.1 Ключевые числа

| Что | Значение |
|---|---|
| Размер | 159 828 байт |
| `react-reconciler` | 120 КБ (75%) |
| R3F core | 12 КБ |
| Версия R3F | 9.5.0 |
| Требует React | `>=19 <19.3` |
| Требует three | `>=0.156` |
| Событий DOM | 10 |
| passive-событий | 7 из 10 |
| Фаз кадра | 3 (before/after/tail) |
| Лимит `invalidate` | 60 кадров |
| Задержка unmount | 500 мс |
| dpr по умолчанию | `[1, 2]` |
| fov по умолчанию | 75 |
| near/far | 0.1 / 1000 |
| camera.position.z | 5 |

## 15.2 Главное

**1. R3F 9.5.0 требует React 19, пиннинг Skybridge — 18.3.1.**
Реальный конфликт, если виджет использует этот R3F с CDN-React.

**2. 75% файла — `react-reconciler` (120 КБ).**
Кастомный `hostConfig`: `isPrimaryRenderer: false`, `supportsMutation: true`.

**3. `pointer` и `mouse` — один `Vector2`.**

**4. `ve.equ` при `strict: false` не видит лишние ключи в `o`.**
`{a:1}` ≡ `{a:1, b:2}` возможно.

**5. `attach` — гибкий механизм:** `"material"`, `"material-color"`,
`"children-2"`, функция.

**6. Кадровый цикл — 3 фазы:** `before`, `after`, `tail`. Подписчики
через Sets.

**7. `internal.priority` отключает авто-рендер** (`gl.render` не вызывается).

**8. `Ko` лимит 60 кадров** — `invalidate(1000)` даст только 60.

**9. `unmount` — 500 мс задержки**, потом `forceContextLoss`.

**10. `powerPreference: "high-performance"`** — дискретная видеокарта,
расход батареи.

**11. `alpha: true`** — прозрачный фон (важно для виджетов).

**12. 10 DOM-обработчиков навешиваются всегда**, даже если не нужны.
