# `transform-4I4J0RKg.js` — полный разбор (d3 subset)

**Размер:** 37 571 байт
**URL:** `https://web-sandbox.oaiusercontent.com/assets/transform-4I4J0RKg.js`
**Роль:** **подмножество D3.js** — 8 модулей: dispatch, selection,
transition, color, interpolate, timer, path, zoom/transform.

---

# ЧАСТЬ 1. Что внутри

## 1.1 Карта модулей

| Байты | Модуль d3 | Что |
|---|---|---|
| 0 – 3 000 | **d3-dispatch** | шина событий (`dispatch.on`) |
| 3 000 – 12 000 | **d3-selection** | выборки DOM (`select`, `selectAll`, `data`, `join`) |
| 12 000 – 14 500 | d3-selection: события (`on`, `dispatch`) | |
| 14 500 – 20 400 | **d3-color** | парсинг/fормат цветов (148 имён, rgb, hsl, hex) |
| 20 400 – 22 500 | **d3-interpolate** | интерполяция (`rgb`, `hsl`, числа) |
| 22 500 – 26 000 | **d3-timer** | таймеры (`setTimeout`/`setInterval` с кадрами) |
| 26 000 – 34 000 | **d3-transition** | анимации (`transition`, `interrupt`) |
| 34 000 – 37 571 | **d3-path** (line) + **d3-zoom** (transform) | |

**Никаких `d3-shape` (curves), `d3-scale`, `d3-axis`, `d3-force`.**
Только то, что нужно Mermaid для линий и интерполяции кривых.

## 1.2 Экспорты

```js
export {
  V as C,    // Color (базовый класс)
  g as R,    // rgb
  y as S,    // Selection (класс)
  wt as a,   // constant (() => value)
  P as b,    // ?
  Ci as c,   // line (d3-path: context → line)
  It as d,   // interpolateRgb
  cr as e,   // interpolateNumber
  gt as f,   // inherit (proto)
  Qt as g,   // objectCreate (proto, props)
  Si as h,   // interpolateHue
  A as i,    // interpolateLinear
  Ke as j,   // [null] (parents)
  tn as n,   // interpolateNumber (без gamma)
  ir as r    // interpolateZoom (X — Transform)
};
```

**14 экспортов.** Используются в `render-mermaid-svg` как `lbn`.

---

# ЧАСТЬ 2. `d3-dispatch` — шина событий

## 2.1 `Pt()` — dispatch

```js
function Pt() {
  for (var t = 0, n = arguments.length, e = {}, r; t < n; ++t) {
    if (!(r = arguments[t] + "") || r in e || /[\s.]/.test(r))
      throw new Error("illegal type: " + r);
    e[r] = []
  }
  return new K(e)
}
```

**Создаёт объект с типами событий.**

**Валидация:** имя не пустое, не дубликат, без пробелов и точек.

**`/[\s.]/`** — почему точка запрещена: `on("click.foo")` в d3 использует
точку для namespace. Здесь — базовый dispatch без namespace.

## 2.2 `K` — класс

```js
function K(t) { this._ = t }
K.prototype = Pt.prototype = { constructor: K, on: function (t, n) { ... } }
```

**`this._`** — объект `{тип: [callbacks]}`.

## 2.3 `on()` — подписка

```js
on: function (t, n) {
  var e = this._, r = _n(t + "", e), i, o = -1, s = r.length;

  // 1. геттер
  if (arguments.length < 2) {
    for (;++o < s;)
      if ((i = (t = r[o]).type) && (i = dn(e[i], t.name))) return i;
    return
  }

  // 2. валидация callback
  if (n != null && typeof n != "function")
    throw new Error("invalid callback: " + n);

  // 3. сеттер
  for (;++o < s;) {
    if (i = (t = r[o]).type) e[i] = Nt(e[i], t.name, n);
    else if (n == null) for (i in e) e[i] = Nt(e[i], t.name, n)
  }
  return this
}
```

### Три режима

**1. `dispatch.on("click")`** — получить текущий callback.
**2. `dispatch.on("click", fn)`** — подписать.
**3. `dispatch.on("click", null)`** — отписать.

**`_n(t, e)`** — парсинг типа:
```js
function _n(t, n) {
  return t.trim().split(/^|\s+/).map(function (e) {
    var r = "", i = e.indexOf(".");
    if (i >= 0 && (r = e.slice(i + 1), e = e.slice(0, i)), e && !n.hasOwnProperty(e))
      throw new Error("unknown type: " + e);
    return {type: e, name: r}
  })
}
```

**Можно подписаться на несколько:**
```js
dispatch.on("click keydown", handler)
```
`split(/^|\s+/)` → `["click", "keydown"]`.

**Namespace через точку:**
```js
dispatch.on("click.foo", handler)
```
`{type: "click", name: "foo"}` — можно снять только `foo`:
```js
dispatch.on("click.foo", null)
```

**`unknown type`** — если подписаться на несуществующий тип.

## 2.4 `Nt()` — управление списком

```js
function Nt(t, n, e) {
  // ... (не показано)
}
```

**Скорее всего:**
- `e == null` → удалить callback с именем `n`
- callback уже есть с тем же именем → заменить
- иначе → добавить

---

# ЧАСТЬ 3. `d3-selection` — выборки

## 3.1 `y` — Selection

```js
var Ke = [null];

function y(t, n) { this._groups = t; this._parents = n }

function O() { return new y([[document.documentElement]], Ke) }

y.prototype = O.prototype = {
  constructor: y,
  select: xn,
  selectAll: Nn,
  selectChild: En,
  selectChildren: Rn,
  filter: Mn,
  data: Dn,
  enter: In,
  exit: Ln,
  join: On,
  merge: Vn,
  selection: Ge,
  order: Yn,
  sort: Bn,
  call: Un,
  nodes: Kn,
  node: Gn,
  size: Wn,
  empty: Jn,
  each: Qn,
  attr: ie,
  style: ue,
  property: he,
  classed: ge,
  text: ve,
  html: Ae,
  raise: ke,
  lower: Se,
  append: Ce,
  insert: Re,
  remove: Ie,
  clone: Xe,
  datum: qe,
  on: Ve,
  dispatch: ze,
  [Symbol.iterator]: Ue
};
```

### 31 метод

**Структура:**
- `_groups` — массив массивов элементов (`[[e1, e2], [e3]]`)
- `_parents` — массив родителей

**Итератор:**
```js
function* Ue() {
  for (var t = this._groups, n = 0, e = t.length; n < e; ++n)
    for (var r = t[n], i = 0, o = r.length, s; i < o; ++i)
      (s = r[i]) && (yield s)
}
```

**Yield только не-null** элементы.

## 3.2 `Ve()` — `on()`

```js
function Ve(t, n, e) {
  var r = Pe(t + ""),      // парсинг типа (с namespace)
        i, o = r.length, s;

  // 1. геттер
  if (arguments.length < 2) {
    var a = this.node().__on;
    if (a) {
      for (var u = 0, f = a.length, l; u < f; ++u)
        for (i = 0, l = a[u]; i < o; ++i)
          if ((s = r[i]).type === l.type && s.name === l.name) return l.value;
      return
    }
    return
  }

  // 2. сеттер
  for (a = n ? Oe : Le, i = 0; i < o; ++i)
    this.each(a(r[i], n, e));
  return this
}
```

**Обработчики хранятся в `element.__on`** — массив `{type, name, value, listener, options}`.

**`__on`** — прямое свойство DOM-элемента.

## 3.3 `Jt()` — dispatch event

```js
function Jt(t, n, e) {
  var r = zt(t),
        i = r.CustomEvent;

  typeof i == "function"
    ? i = new i(n, e)
    : (i = r.document.createEvent("Event"),
       e ? (i.initEvent(n, e.bubbles, e.cancelable), i.detail = e.detail)
         : i.initEvent(n, !1, !1)),

    t.dispatchEvent(i)
}
```

**Два пути:**
1. `CustomEvent` есть → создать
2. Fallback → `createEvent("Event")` + `initEvent`

**`zt(t)`** — получить ownerDocument окна.

---

# ЧАСТЬ 4. `d3-color` — цвета

## 4.1 Regex

```js
var q = .7,
    Q = 1 / q,
    C = "\\s*([+-]?\\d+)\\s*",                                        // целое
    D = "\\s*([+-]?(?:\\d*\\.)?\\d+(?:[eE][+-]?\\d+)?)\\s*",          // число
    v = "\\s*([+-]?(?:\\d*\\.)?\\d+(?:[eE][+-]?\\d+)?)%\\s*",         // процент
    We = /^#([0-9a-f]{3,8})$/,                                        // hex
    Je = new RegExp(`^rgb\\(${C},${C},${C}\\)$`),                     // rgb(int)
    Qe = new RegExp(`^rgb\\(${v},${v},${v}\\)$`),                     // rgb(%)
    Ze = new RegExp(`^rgba\\(${C},${C},${C},${D}\\)$`),               // rgba(int)
    je = new RegExp(`^rgba\\(${v},${v},${v},${D}\\)$`),               // rgba(%)
    tr = new RegExp(`^hsl\\(${D},${v},${v}\\)$`),                     // hsl
    nr = new RegExp(`^hsla\\(${D},${v},${v},${D}\\)$`);               // hsla
```

**8 форматов:**
1. `#rgb`, `#rgba`, `#rrggbb`, `#rrggbbaa` (3–8 символов)
2. `rgb(1, 2, 3)` — целые
3. `rgb(1%, 2%, 3%)` — проценты
4. `rgba(1, 2, 3, 0.5)`
5. `rgba(1%, 2%, 3%, 0.5)`
6. `hsl(120, 50%, 50%)`
7. `hsla(120, 50%, 50%, 0.5)`

**`C`** — `\d+` (без минуса?.. нет, с `[+-]?`).
**`D`** — float с экспонентой.
**`v`** — float + `%`.

**`We = /^#([0-9a-f]{3,8})$/`** — 3..8 hex-символов. Не проверяет
кратность (3, 4, 6, 8). `#abcde` (5) — пройдёт regex, но упадёт при разборе.

## 4.2 `$t` — 148 именованных цветов

```js
$t = {
  aliceblue: 15792383,
  antiquewhite: 16444375,
  aqua: 65535,
  aquamarine: 8388564,
  azure: 15794175,
  ...
  rebeccapurple: 6697881,
  red: 16711680,
  rosybrown: 12357519,
  royalblue: 4286945,
  ...
}
```

**Полный набор CSS4 named colors** в виде `0xRRGGBB` десятичного.

**Пример:**
- `red = 16711680` = `0xFF0000`
- `aqua = 65535` = `0x00FFFF`
- `rebeccapurple = 6697881` = `0x663399`

## 4.3 `Mt()` / `z()` — нормализация

```js
function Mt(t) { return t = (t || 0) % 360, t < 0 ? t + 360 : t }
function z(t) { return Math.max(0, Math.min(1, t || 0)) }
```

**`Mt`** — hue в `[0, 360)`. Отрицательный hue → `+360`.
**`z`** — clamp `[0, 1]`.

## 4.4 `ot()` — HSL → RGB

```js
function ot(t, n, e) {
  return (t < 60 ? n + (e - n) * t / 60
       : t < 180 ? e
       : t < 240 ? n + (e - n) * (240 - t) / 60
       : n) * 255
}
```

**Классический алгоритм HSL→RGB** для одного канала:

| Условие | Значение |
|---|---|
| `t < 60` | линейная интерполяция `n → e` |
| `t < 180` | `e` (максимум) |
| `t < 240` | линейная интерполяция `e → n` |
| иначе | `n` (минимум) |

**`× 255`** — в байты.

## 4.5 `formatHsl()`

```js
formatHsl() {
  const t = Z(this.opacity);
  return `${t === 1 ? "hsl(" : "hsla("}${Mt(this.h)}, ${z(this.s) * 100}%, ${z(this.l) * 100}%${t === 1 ? ")" : `, ${t})`}`
}
```

**`hsl(...)` при `opacity === 1`**, иначе `hsla(...)`.

**`Z(t)`** — точность (обычно `1 - 1e-6` для округления).

## 4.6 `rgb()` — конструктор

```js
function g(t, n, e) { ... }   // экспорт R
```

**Из `formatRgb()`:**
```js
formatRgb() {
  const t = Z(this.opacity);
  return `${t === 1 ? "rgb(" : "rgba("}${Math.max(0, Math.min(255, Math.round(this.r) || 0))}, ${...}${t === 1 ? ")" : `, ${t})`}`
}
```

## 4.7 `V` — базовый класс Color

```js
function V() {}   // экспорт C
```

**Абстрактный.** Содержит:
- `displayable()` — проверка валидности
- `toString()` — в hex/rgb
- `formatHex()`, `formatRgb()`, `formatHsl()`

**`displayable()`:**
```js
displayable() {
  return 0<=this.r&&this.r<=255 && 0<=this.g&&this.g<=255
      && 0<=this.b&&this.b<=255 && 0<=this.opacity&&this.opacity<=1
}
```

---

# ЧАСТЬ 5. `d3-interpolate`

## 5.1 `wt()` — константа

```js
const wt = t => () => t;
```

**Возвращает функцию, возвращающую значение.**

## 5.2 `jt()` — линейная

```js
function jt(t, n) {
  return function (e) { return t + e * n }
}
```

**`t + e×n`** — `t` = start, `n` = delta.

## 5.3 `ar()` — гамма

```js
function ar(t, n, e) {
  return t = Math.pow(t, e),
    n = Math.pow(n, e) - t,
    e = 1 / e,
    function (r) { return Math.pow(t + r * n, e) }
}
```

**Гамма-интерполяция:**
```
f(r) = (start^γ + r × (end^γ - start^γ))^(1/γ)
```

**`γ = 1`** — линейная. `γ = 2.2` — sRGB-корректная.

## 5.4 `Si()` — hue

```js
function Si(t, n) {
  var e = n - t;
  return e
    ? jt(t, e > 180 || e < -180 ? e - 360 * Math.round(e / 360) : e)
    : wt(isNaN(t) ? n : t)
}
```

**Короткий путь по кругу:** если разница > 180° — идти в обратную сторону.

**Пример:** `350° → 10°` — разница `-340°` → нормализуется до `+20°`.

**`isNaN(t) ? n : t`** — если `t` не число, взять `n`.

## 5.5 `ur()` / `tn()` — числа

```js
function ur(t) {
  return (t = +t) == 1 ? tn : function (n, e) {
    return e - n ? ar(n, e, t) : wt(isNaN(n) ? e : n)
  }
}

function tn(t, n) {
  var e = n - t;
  return e ? jt(t, e) : wt(isNaN(t) ? n : t)
}
```

**`ur(gamma)`** — конструктор с гаммой.
**`tn`** — простой `interpolateNumber` (gamma = 1).

## 5.6 `It` — `interpolateRgb`

```js
const It = function t(n) {
  var e = ur(n);

  function r(i, o) {
    var s = e((i = ut(i)).r, (o = ut(o)).r),
        a = e(i.g, o.g),
        u = e(i.b, o.b),
        f = tn(i.opacity, o.opacity);

    return function (l) {
      return i.r = s(l),
        i.g = a(l),
        i.b = u(l),
        i.opacity = f(l),
        i + ""
    }
  }

  return r.gamma = t, r
}(1);
```

### Разбор

**`ut(t)`** — парсинг цвета (d3.color).

**Интерполируются 4 канала:** `r`, `g`, `b`, `opacity`.

**Все через `e(...)`** — гамма-интерполяция.

**Возврат — функция `(l) => "rgb(...)"`.**

**`r.gamma = t`** — можно указать гамму:
```js
interpolateRgb.gamma(2.2)(color1, color2)
```

**По умолчанию `gamma = 1`** (линейная).

## 5.7 `A` — `interpolateLinear`

```js
function A(t, n) {
  return t = +t, n = +n,
    function (e) { return t * (1 - e) + n * e }
}
```

**`start×(1-t) + end×t`** — стандартная линейная.

## 5.8 `ft` / `st` — number regex

```js
var ft = /[-+]?(?:\d+\.?\d*|\.?\d+)(?:[eE][-+]?\d+)?/g,
    st = new RegExp(ft.source, "g");
```

**`ft`** — глобальный: все числа в строке.
**`st`** — то же, но отдельный экземпляр (для `lastIndex`).

**Зачем два:** `RegExp` с `g` хранит `lastIndex`. Общий экземпляр
при параллельных вызовах ломается.

## 5.9 `fr` — `interpolateString`

```js
function fr(t) { ... }
```

**Интерполирует строки:** находит все числа и интерполирует их,
сохраняя текст.

---

# ЧАСТЬ 6. `d3-timer`

## 6.1 `Xt()` — transition timer

```js
function Xt(t, n, e) {
  var r = new nt;
  return n = n == null ? 0 : +n,
    r.restart(i => { r.stop(), t(i + n) }, n, e),
    r
}
```

**Создать таймер, который сразу перезапускается.** `t(i + n)` —
с задержкой.

## 6.2 Эффективный таймер

```js
// (из фрагмента 25000)
n > 24
  ? (t < 1/0 && (F = setTimeout(Ht, t - L.now() - rt)), I && (I = clearInterval(I)))
  : (I || (tt = L.now(), I = setInterval(yr, rn)), R = 1, on(Ht))
```

**Два режима:**
- **интервал > 24 мс** → `setTimeout` (точнее)
- **интервал ≤ 24 мс** → `setInterval` (эффективнее для 60 FPS)

**`24 мс`** — порог. 60 FPS ≈ 16.7 мс, но с запасом.

**`L.now()`** — `performance.now()`.

---

# ЧАСТЬ 7. `d3-transition`

## 7.1 Константы состояний

```js
var vr = Pt("start", "end", "cancel", "interrupt"),
    mr = [],
    an = 0,   // NONE
    qt = 1,   // SCHEDULED
    ht = 2,   // STARTING
    G  = 3,   // STARTED
    Dt = 4,   // RUNNING
    pt = 5,   // ENDING
    W  = 6;   // ENDED
```

**7 состояний transition.**

**`vr`** — dispatch с 4 типами: `start`, `end`, `cancel`, `interrupt`.

## 7.2 `it()` — создать transition

```js
function it(t, n, e, r, i, o) {
  var s = t.__transition;

  if (!s) t.__transition = {};
  else if (e in s) return;    // ← уже есть

  br(t, e, {
    name: n,
    index: r,
    group: i,
    on: vr,
    tween: mr,
    time: o.time,
    delay: o.delay,
    duration: o.duration,
    ease: o.ease,
    timer: null,
    state: an
  })
}
```

**`__transition`** — прямое свойство DOM-элемента.

**Защита от дубликата:** `else if (e in s) return`.

## 7.3 `br()` — запуск

```js
function br(t, n, e) {
  var r = t.__transition, i;

  r[n] = e;
  e.timer = sn(o, 0, e.time);

  function o(f) {
    e.state = qt;                       // SCHEDULED
    e.timer.restart(s, e.delay, e.time);
    e.delay <= f && s(f - e.delay)
  }

  function s(f) {
    var l, c, h, p;
    if (e.state !== qt) return u();

    for (l in r)
      if (p = r[l], p.name === e.name) {
        if (p.state === G) return Xt(s);           // перезапуск
        p.state === Dt
          ? (p.state = W,                          // ENDED
             p.timer.stop(),
             p.on.call("interrupt", t, t.__data__, p.index, p.group),
             delete r[l])
          : +l < n && (p.state = W, p.timer.stop(), delete r[l])
      }
    ...
  }
}
```

### Логика

**1. Переход в `SCHEDULED`.**
**2. Перезапуск таймера.**
**3. Если `delay <= f`** — сразу старт.

**При старте (`s`):**

**Конфликт с другими transitions того же имени:**
- Если другой `STARTED` → перезапустить этот (рекурсивно `Xt(s)`)
- Если другой `RUNNING` → прервать его (`interrupt`), удалить
- Если индекс меньше — тоже прервать

**Это семантика d3:** новый transition того же имени отменяет старые.

## 7.4 `xt()` / `m()` / `x()` — получение

```js
function xt(t, n) {
  var e = x(t, n);
  if (e.state > an) throw new Error("too late; already scheduled");
  return e
}

function m(t, n) {
  var e = x(t, n);
  if (e.state > G) throw new Error("too late; already running");
  return e
}

function x(t, n) {
  var e = t.__transition;
  if (!e || !(e = e[n])) throw new Error("transition not found");
  return e
}
```

**Три уровня строгости:**
- `xt` — можно только до `SCHEDULED`
- `m` — можно до `STARTED`
- `x` — просто найти

---

# ЧАСТЬ 8. `d3-path` — line

## 8.1 `cn` — линейный контекст

```js
function cn(t) { this._context = t }

cn.prototype = {
  areaStart: function () { this._line = 0 },
  areaEnd: function () { this._line = NaN },
  lineStart: function () { this._point = 0 },

  lineEnd: function () {
    (this._line || this._line !== 0 && this._point === 1)
      && this._context.closePath(),
    this._line = 1 - this._line
  },

  point: function (t, n) {
    switch (t = +t, n = +n, this._point) {
      case 0:
        this._point = 1;
        this._line ? this._context.lineTo(t, n) : this._context.moveTo(t, n);
        break;
      case 1:
        this._point = 2;
      default:
        this._context.lineTo(t, n);
        break
    }
  }
};
```

### Разбор

**`_line`** — флаг «мы в area» (0/1) или вне (NaN).

**`_point`** — счётчик точек.

**`lineEnd`:**
```js
(this._line || this._line !== 0 && this._point === 1) && closePath()
```

**Условие закрытия:**
- `_line` truthy (мы в area) **или**
- `_line === 0` **и** одна точка

**То есть:** если линия в area-контексте — закрыть. Если одна точка
в line-контексте — тоже закрыть (для круга).

**`this._line = 1 - this._line`** — переключение area/line.

**`point`:**
- Первая точка: `moveTo` (или `lineTo` если в area)
- Вторая и далее: `lineTo`

**`Ci`** — фабрика (`c` в экспорте):
```js
function Ci(t) { return new cn(t) }
```

---

# ЧАСТЬ 9. `d3-zoom` — Transform

## 9.1 `X` — класс

```js
function X(t, n, e) { this.k = t; this.x = n; this.y = e }

X.prototype = {
  constructor: X,

  scale: function (t) {
    return t === 1 ? this : new X(this.k * t, this.x, this.y)
  },

  translate: function (t, n) {
    return t === 0 & n === 0 ? this : new X(this.k, this.x + this.k * t, this.y + this.k * n)
  },

  apply: function (t) {
    return [t[0] * this.k + this.x, t[1] * this.k + this.y]
  },

  applyX: function (t) { return t * this.k + this.x },
  applyY: function (t) { return t * this.k + this.y },

  invert: function (t) {
    return [(t[0] - this.x) / this.k, (t[1] - this.y) / this.k]
  },

  invertX: function (t) { return (t - this.x) / this.k },
  invertY: function (t) { return (t - this.y) / this.k },

  rescaleX: function (t) {
    return t.copy().domain(t.range().map(this.invertX, this).map(t.invert, t))
  },

  rescaleY: function (t) {
    return t.copy().domain(t.range().map(this.invertY, this).map(t.invert, t))
  },

  toString: function () {
    return "translate(" + this.x + "," + this.y + ") scale(" + this.k + ")"
  }
};
```

## 9.2 Математика

**Transform:** `(x, y) → (x × k + tx, y × k + ty)`

**`scale(s)`** — умножить `k`, оставить `x`/`y`.
**`translate(dx, dy)`** — добавить `k × dx` к `x` (масштабированный сдвиг!).

**Пример:**
```js
new X(2, 10, 20).translate(5, 0)
// → X(2, 10 + 2×5, 20) = X(2, 20, 20)
```

**`apply([x, y])`** — преобразовать точку.
**`invert([px, py])`** — обратное преобразование.

**`rescaleX(scale)`** — пересчитать домен шкалы после zoom:
```js
t.copy().domain(t.range().map(invertX).map(t.invert))
```
**Смысл:** взять range шкалы, обратить zoom, обратить scale → новый домен.

**`toString()`** — SVG-трансформ:
```
translate(10,20) scale(2)
```

## 9.3 Баг с `&`

```js
translate: function (t, n) {
  return t === 0 & n === 0 ? this : ...
}
```

**`&` (побитовое И) вместо `&&` (логическое)!**

**`t === 0 & n === 0`** — `boolean & boolean` → `0` или `1`.
Работает случайно (0 — falsy, 1 — truthy), но это **ошибка**.

**Если `t === 0` и `n === 0`** → `1 & 1 = 1` → `true` → вернуть `this`. Ок.

**Если `t === 0`, `n !== 0`** → `1 & 0 = 0` → `false`. Ок.

**Работает, но неправильно по типу.** В TypeScript была бы ошибка.

---

# ЧАСТЬ 10. Схема

```
Mermaid → render-mermaid-svg
            │
            ├─ interpolateToCurve(interpolate, lbn)
            │    └─ lbn = transform-4I4J0RKg.js
            │         ├─ interpolateRgb(c1, c2) → цвет
            │         ├─ interpolateNumber(a, b) → число
            │         ├─ interpolateHue(h1, h2) → угол
            │         └─ line(context) → SVG-путь
            │
            ├─ selection.append/attr/style
            │    └─ D3 Selection API
            │
            └─ transition().duration().ease()
                 └─ D3 Transition API
```

---

# ЧАСТЬ 11. Слабые места

## 11.1 `&` вместо `&&` в `translate`

```js
return t === 0 & n === 0 ? this : ...
```

**Побитовое И** на булевых. Работает случайно.

## 11.2 `We = /^#([0-9a-f]{3,8})$/`

**Не проверяет кратность.** `#abcde` (5 символов) пройдёт regex, но
при разборе упадёт или даст мусор.

**Правильно:** `/^#([0-9a-f]{3}|[0-9a-f]{4}|[0-9a-f]{6}|[0-9a-f]{8})$/`.

## 11.3 `_n()` требует `hasOwnProperty` на объекте типов

```js
e && !n.hasOwnProperty(e) && throw new Error("unknown type: " + e)
```

**`n`** — `this._` (объект с типами).

**Если `n` создан через `Object.create(null)`** — `hasOwnProperty` нет →
`TypeError`.

**Но `Pt()` создаёт `{}` (с прототипом)**, так что ок.

## 11.4 Обработчики в `__on` — прямое свойство DOM

```js
var a = this.node().__on;
```

**Конфликт:** если два разных D3-экземпляра на одном элементе —
перезапишут `__on`.

**Плюс:** `__on` видно в `Object.keys(element)`.

## 11.5 `__transition` — тоже прямое свойство

```js
var s = t.__transition;
if (!s) t.__transition = {};
```

**Та же проблема** — коллизия имён.

## 11.6 `it()` при дубликате — молча `return`

```js
else if (e in s) return;
```

**Второй transition с тем же `id` молча не создаётся.**

**Хост не узнает.**

## 11.7 `Xt()` — таймер может зациклиться

```js
function Xt(t, n, e) {
  var r = new nt;
  return n = n == null ? 0 : +n,
    r.restart(i => { r.stop(), t(i + n) }, n, e),
    r
}
```

**`r.restart(i => { r.stop(); t(i + n) })`** — сразу стоп и запуск `t`.

**Если `t` вызовет `Xt` снова** — бесконечная рекурсия.

**В `br()`:**
```js
if (p.state === G) return Xt(s);
```
**`s` вызывает `Xt(s)`** — если `p.state` так и останется `G` — цикл.

## 11.8 `L` (state) — глобальное

```js
an = 0, qt = 1, ht = 2, G = 3, Dt = 4, pt = 5, W = 6
```

**Числа — магические.** Легко перепутать.

## 11.9 `ot()` не обрабатывает hue ≥ 240

```js
return (t < 60 ? n + (e - n) * t / 60
     : t < 180 ? e
     : t < 240 ? n + (e - n) * (240 - t) / 60
     : n) * 255
```

**Для `t >= 240`** — вернуть `n`.

**Проверить:** стандарт HSL→RGB для `h` в шестом секторе (300–360°) →
возврат `n`. **Правильно.**

## 11.10 `formatHsl()` не clamp-ит

```js
`${Mt(this.h)}, ${z(this.s) * 100}%, ${z(this.l) * 100}%`
```

**`z()` clamp-ит `[0,1]`.** Но если `s` = `2` (невалидно) — вернёт `1`
молча, вместо ошибки.

## 11.11 `displayable()` не проверяет NaN

```js
displayable() {
  return 0<=this.r&&this.r<=255 && ...
}
```

**`NaN <= 255`** = `false` → вернёт `false`. Ок.

**Но `0 <= NaN`** = `false` → тоже `false`. Короткое замыкание.

**Работает.**

---

# ЧАСТЬ 12. Итог

## 12.1 Ключевые числа

| Что | Значение |
|---|---|
| Размер | 37 571 байт |
| Модулей d3 | 8 |
| Методов Selection | 31 |
| Именованных цветов | 148 |
| Форматов цвета | 8 |
| Состояний transition | 7 |
| Типов transition-событий | 4 |
| Экспортов | 14 |
| Порог таймера (setTimeout/setInterval) | 24 мс |

## 12.2 Главное

**1. Это подмножество D3, а не весь D3.** Только 8 модулей:
dispatch, selection, transition, color, interpolate, timer, path, zoom.

**Нет:** `d3-shape` (curves), `d3-scale`, `d3-axis`, `d3-force`,
`d3-hierarchy`, `d3-array`.

**2. `interpolateRgb` — основная функция** для кривых Mermaid
(`lbn` в `render-X3XFXER2`).

**3. 148 именованных цветов CSS4** в виде десятичных `0xRRGGBB`.

**4. `d3-path` — только `line` (без curves)** — `cn` класс.
Простейший: `moveTo` + `lineTo`.

**5. `X` (Transform) — zoom-математика:**
```js
translate(t, n) → new X(k, x + k*t, y + k*n)   // масштабированный сдвиг
toString() → "translate(x,y) scale(k)"
```

**6. Баг:** `t === 0 & n === 0` вместо `&&`.

**7. Regex hex не проверяет кратность** — `#abcde` пройдёт.

**8. `__transition` и `__on` — прямое свойство DOM**, коллизии возможны.

**9. `Xt()` может зациклиться** при рестарте.

**10. Таймер: `24 мс` — порог** между `setTimeout` и `setInterval`.
