# `client-BjybnHiv.js` — полный разбор (react-dom)

**Размер:** 208 365 байт
**URL:** `https://web-sandbox.oaiusercontent.com/assets/client-BjybnHiv.js`
**Роль:** **`react-dom` v19.3.0-canary** — DOM-рендерер React.
Экспортирует два модуля: `a` = `react-dom`, `c` = namespace-обёртка.

---

# ЧАСТЬ 1. Что это и версия

## 1.1 Точная версия

```js
var R1 = K1.version;

if (R1 !== "19.3.0-canary-7dc903cd-20251203")
  throw Error(z(527, R1, "19.3.0-canary-7dc903cd-20251203"));

Ve.version = "19.3.0-canary-7dc903cd-20251203";
```

**`react-dom@19.3.0-canary-7dc903cd-20251203`**

**Canary-сборка** — не стабильный релиз. Дата коммита: **2025-12-03**.

**Проверка версии жёсткая:** если загруженный `react-dom` не этой версии —
`Error #527`.

## 1.2 Экспорты

```js
export {Lm as a, $2 as c};
```

| Экспорт | Что |
|---|---|
| `a` (`Lm`) | `react-dom` — объект с API |
| `c` (`$2`) | namespace-обёртка `{default: reactDom, ...}` |

**Используется в:**
- `run-react` → `jGe()` обёртка: `import {createRoot} from 'react-dom/client'`
- **НО:** здесь нет `react-dom/client`! Только `react-dom`.

## 1.3 Отсутствие `react-dom/client`

**В файле есть `createRoot` и `hydrateRoot`** — но на **самом** `react-dom`,
а не на подпути `/client`.

**В React 19** `createRoot` переехал в `react-dom/client`. Здесь он
экспортируется из основного модуля — потому что это **внутренняя** сборка
(`react-dom-client.production.js`, не публичный пакет).

---

# ЧАСТЬ 2. Состав файла

## 2.1 Три React-модуля

| Байты | Модуль | Что |
|---|---|---|
| 479 – 4 355 | `react-dom.production.js` | **shared** (`createPortal`, `flushSync`, `preload`, `preinit`) |
| 4 337 – 8 193 | `react-dom-client.production.js` (начало) | API-поверхность |
| 8 193 – 206 200 | `react-dom-client.production.js` (тело) | **весь DOM-рендерер** |
| 206 200 – 208 365 | финал: `createRoot`, `hydrateRoot` | точки входа |

**Плюс `scheduler.production.js`** — внутри `client` (байты ~479–4337).

## 2.2 Импорты

```js
import {j as Km} from "./main-BR1bOQyH.js";     // Object (UMD)
import {r as Q1} from "./index-C0fqnZRQ.js";    // React
```

---

# ЧАСТЬ 3. `react-dom` shared — публичный API

## 3.1 `__DOM_INTERNALS_DO_NOT_USE_OR_WARN_USERS_THEY_CANNOT_UPGRADE`

```js
Nl.__DOM_INTERNALS_DO_NOT_USE_OR_WARN_USERS_THEY_CANNOT_UPGRADE = _l;
```

**Имя говорит само за себя.** Это `_l` — внутренний объект с методами:
- `_l.d` — DOM-операции (`C`, `D`, `S`, `X`, `M`, `L`, `m`, `r`, `f`)
- `_l.p` — приоритет
- `_l.T` — диспетчер

**«Не использовать или предупреждать пользователей, что они не могут
обновиться»** — буквальное название. Внутренний API, не для публики.

## 3.2 `createPortal`

```js
Nl.createPortal = function (l, t) {
  var u = 2 < arguments.length && arguments[2] !== void 0 ? arguments[2] : null;

  if (!t || t.nodeType !== 1 && t.nodeType !== 9 && t.nodeType !== 11)
    throw Error(L1(299));

  return Fm(l, t, null, u)
};
```

**Валидация `container`:**
- `nodeType === 1` — Element
- `nodeType === 9` — Document
- `nodeType === 11` — DocumentFragment

**Иначе — `Error #299`** («Target container is not a DOM element»).

**`Fm(l, t, null, u)`** — внутренняя реализация (в `react-reconciler`).

## 3.3 `flushSync`

```js
Nl.flushSync = function (l) {
  var t = Va.T,          // текущий диспетчер
      u = _l.p;          // текущий приоритет

  try {
    if (Va.T = null, _l.p = 2, l)   // приоритет 2 = SyncLane
      return l()
  } finally {
    Va.T = t, _l.p = u, _l.d.f()    // восстановить + flush
  }
};
```

**Что делает:**
1. Сохранить текущий диспетчер и приоритет
2. Установить `T = null` (отключить хуки) и `p = 2` (синхронный приоритет)
3. Выполнить функцию
4. Восстановить + `_l.d.f()` — принудительный flush

**Зачем:** `flushSync` гарантирует, что обновления состояния применятся
**синхронно**, до возврата.

**Опасность:** блокирует UI. Нельзя использовать в lifecycle-методах.

## 3.4 `preconnect` / `prefetchDNS`

```js
Nl.preconnect = function (l, t) {
  typeof l === "string" && (
    t ? (t = t.crossOrigin,
         t = typeof t === "string" ? (t === "use-credentials" ? t : "") : void 0)
      : t = null,
    _l.d.C(l, t)
  )
};

Nl.prefetchDNS = function (l) {
  typeof l === "string" && _l.d.D(l)
};
```

**`__DOM_INTERNALS.preconnect(url, {crossOrigin})`** — `<link rel="preconnect">`.

**`prefetchDNS(url)`** — `<link rel="dns-prefetch">`.

**`crossOrigin`:** только `"use-credentials"` или `""` (анонимный CORS).
Любая другая строка → `""`.

## 3.5 `preinit`

```js
Nl.preinit = function (l, t) {
  if (typeof l === "string" && t && typeof t.as === "string") {
    var u = t.as,
        a = Le(u, t.crossOrigin),
        n = typeof t.integrity === "string" ? t.integrity : void 0,
        e = typeof t.fetchPriority === "string" ? t.fetchPriority : void 0;

    u === "style"
      ? _l.d.S(l, typeof t.precedence === "string" ? t.precedence : void 0,
                {crossOrigin: a, integrity: n, fetchPriority: e})
      : u === "script" && _l.d.X(l, {
          crossOrigin: a, integrity: n, fetchPriority: e,
          nonce: typeof t.nonce === "string" ? t.nonce : void 0
        })
  }
};
```

**`__DOM_INTERNALS.preinit(url, {as, crossOrigin, integrity, fetchPriority, precedence, nonce})`**

**Два типа:** `"style"` и `"script"`.

**`precedence`** — для стилей. React использует его для **дедупликации**
и порядка.

**`integrity`** — SRI! **Единственное место, где React поддерживает SRI**.

## 3.6 `preinitModule`

```js
Nl.preinitModule = function (l, t) {
  if (typeof l === "string")
    if (typeof t === "object" && t !== null) {
      if (t.as == null || t.as === "script") {
        var u = Le(t.as, t.crossOrigin);
        _l.d.M(l, {
          crossOrigin: u,
          integrity: typeof t.integrity === "string" ? t.integrity : void 0,
          nonce: typeof t.nonce === "string" ? t.nonce : void 0
        })
      }
    } else t == null && _l.d.M(l)
};
```

**`<link rel="modulepreload">`.** Только для `as === "script"` или
не указан.

## 3.7 `preload`

```js
Nl.preload = function (l, t) {
  if (typeof l === "string" && typeof t === "object" && t !== null
      && typeof t.as === "string") {
    var u = t.as,
        a = Le(u, t.crossOrigin);

    _l.d.L(l, u, {
      crossOrigin: a,
      integrity: typeof t.integrity === "string" ? t.integrity : void 0,
      nonce: typeof t.nonce === "string" ? t.nonce : void 0,
      type: typeof t.type === "string" ? t.type : void 0,
      fetchPriority: typeof t.fetchPriority === "string" ? t.fetchPriority : void 0,
      referrerPolicy: typeof t.referrerPolicy === "string" ? t.referrerPolicy : void 0,
      imageSrcSet: typeof t.imageSrcSet === "string" ? t.imageSrcSet : void 0,
      imageSizes: typeof t.imageSizes === "string" ? t.imageSizes : void 0,
      media: typeof t.media === "string" ? t.media : void 0
    })
  }
};
```

**Самое богатое — 9 опций:**
- `as` (обязательно)
- `crossOrigin`
- `integrity`
- `nonce`
- `type`
- `fetchPriority`
- `referrerPolicy`
- `imageSrcSet`
- `imageSizes`
- `media`

**`<link rel="preload">`.**

## 3.8 `preloadModule`

```js
Nl.preloadModule = function (l, t) {
  if (typeof l === "string")
    if (t) {
      var u = Le(t.as, t.crossOrigin);
      _l.d.m(l, {
        as: typeof t.as === "string" && t.as !== "script" ? t.as : void 0,
        crossOrigin: u,
        integrity: typeof t.integrity === "string" ? t.integrity : void 0
      })
    } else _l.d.m(l)
};
```

**Отличие от `preinitModule`:** `as` **не может** быть `"script"`.

**Порядок:** `preload` (загрузить) → `initModule` (инициализировать).

## 3.9 `requestFormReset`

```js
Nl.requestFormReset = function (l) {
  _l.d.r(l)
};
```

**Сброс формы из React.** Связано с `useActionState` и Server Actions.

## 3.10 `unstable_batchedUpdates`

```js
Nl.unstable_batchedUpdates = function (l, t) {
  return l(t)
};
```

**Полный no-op.** Просто вызывает функцию.

**Почему:** в React 18+ автоматическое батчинг всегда включён. Функция
осталась для совместимости.

**`l(t)`** — но `t` передаётся только первый аргумент. Если вызвать
`unstable_batchedUpdates(fn, arg)` — `fn(arg)`.

## 3.11 `useFormState` / `useFormStatus`

```js
Nl.useFormState = function (l, t, u) {
  return Va.H.useFormState(l, t, u)
};

Nl.useFormStatus = function () {
  return Va.H.useHostTransitionStatus()
};
```

**Делегирование в диспетчер хуков** `Va.H`.

**`useFormState`** — устаревшее имя (сейчас `useActionState`).
**`useFormStatus`** — статус формы.

## 3.12 `findDOMNode`

```js
Z.findDOMNode = function (l) {
  var t = l._reactInternals;

  if (t === void 0)
    throw typeof l.render === "function"
      ? Error(z(188))
      : (l = Object.keys(l).join(","), Error(z(268, l)));

  return l = km(t),
    l = l !== null ? $1(l) : null,
    l = l === null ? null : l.stateNode,
    l
};
```

**Устаревший API.** Находит DOM-узел для компонента.

**`_reactInternals`** — Fiber-ссылка, прикреплённая React'ом.

**`Error #188`** — «Unable to find node on an unmounted component».
**`Error #268`** — «Argument appears to not be a ReactComponent».

**`z(268, l)`** — включает список ключей для отладки.

**Удалён в React 19?** Нет — здесь присутствует. Но **deprecated**.

---

# ЧАСТЬ 4. `createRoot` — финал файла

```js
Ve.createRoot = function (l, t) {
  if (!J1(l)) throw Error(z(299));

  var u = !1,        // strictMode
      a = "",        // identifierPrefix
      n = Ey,        // onUncaughtError
      e = Oy,        // onCaughtError
      f = sy;        // onRecoverableError

  t != null && (
    t.unstable_strictMode === !0 && (u = !0),
    t.identifierPrefix !== void 0 && (a = t.identifierPrefix),
    t.onUncaughtError !== void 0 && (n = t.onUncaughtError),
    t.onCaughtError !== void 0 && (e = t.onCaughtError),
    t.onRecoverableError !== void 0 && (f = t.onRecoverableError)
  );

  t = qm(l, 1, !1, null, null, u, a, null, n, e, f, jm);

  l[Ea] = t.current;    // Ea = internal container key
  wc(l);
  return new Pc(t)
};
```

## 4.1 Параметры

| Параметр | По умолчанию | Что |
|---|---|---|
| `container` | — | DOM-элемент |
| `options.unstable_strictMode` | `false` | StrictMode |
| `options.identifierPrefix` | `""` | префикс ID (`:r0:` и т.п.) |
| `options.onUncaughtError` | `Ey` | ошибка без boundary |
| `options.onCaughtError` | `Oy` | ошибка в boundary |
| `options.onRecoverableError` | `sy` | восстанавливаемая |

## 4.2 `qm(...)` — создание контейнера

```js
qm(l, 1, !1, null, null, u, a, null, n, e, f, jm)
```

**Аргументы:**
- `l` — container
- `1` — **`ConcurrentRoot`**
- `!1` — **не hydrate**
- `null` — initialChildren
- `null` — `hydrationCallbacks`
- `u` — strictMode
- `a` — identifierPrefix
- `null` — `formState`
- `n, e, f` — обработчики ошибок
- `jm` — `transitionCallbacks`

## 4.3 Возврат

```js
return new Pc(t)
```

**`Pc`** — класс `ReactDOMRoot` с методами `render()`, `unmount()`.

## 4.4 `hydrateRoot`

```js
Ve.hydrateRoot = function (l, t, u) {
  if (!J1(l)) throw Error(z(299));

  var a = !1, n = "", e = Ey, f = Oy, i = sy, c = null;

  u != null && (
    u.unstable_strictMode === !0 && (a = !0),
    u.identifierPrefix !== void 0 && (n = u.identifierPrefix),
    u.onUncaughtError !== void 0 && (e = u.onUncaughtError),
    u.onCaughtError !== void 0 && (f = u.onCaughtError),
    u.onRecoverableError !== void 0 && (i = u.onRecoverableError),
    u.formState !== void 0 && (c = u.formState)
  );

  t = qm(l, 1, !0, t, u ?? null, a, n, c, e, f, i, jm);

  t.context = Gm(null);         // ← пустой legacy-context

  u = t.current;
  a = jl();
  a = nc(a);
  n = It(a);
  n.callback = null;
  Pt(u, n, a);
  u = a;
  t.current.lanes = u;
  bn(t, u);
  ot(t);

  l[Ea] = t.current;
  wc(l);
  return new ef(t)
};
```

### Отличия от `createRoot`

**1. `qm(l, 1, !0, t, ...)`** — третий аргумент `!0` = **hydrate**.

**2. `t.context = Gm(null)`** — legacy-context (устаревший API).

**3. Немедленное обновление:**
```js
u = a;
t.current.lanes = u;
bn(t, u);
ot(t);
```
То есть hydration **запускается сразу**, не ждёт `render()`.

**4. `t` как initialChildren** — четвертый аргумент `qm`.

**5. `u.formState`** — дополнительная опция.

**6. Возврат `new ef(t)`** — другой класс (HydratedRoot).

## 4.5 `Error #299`

```js
if (!J1(l)) throw Error(z(299))
```

**`J1`** — проверка «это валидный DOM-контейнер».

**`#299`** = «Target container is not a DOM element».

---

# ЧАСТЬ 5. Внутренности: React 19 фичи

## 5.1 `react.view_transition`

```js
var rf = Symbol.for("react.view_transition")
```

**View Transitions API** — новый механизм анимации переходов между
состояниями.

**React 19** поддерживает `<ViewTransition>`.

**`Symbol.for`** — глобальный символ (совместимость между бандлами).

## 5.2 `react.client.reference`

```js
var nd = Symbol.for("react.client.reference")
```

**Client References** — RSC (React Server Components). Объект, описывающий
ссылку на клиентский компонент в серверном рендеринге.

**В браузере** используется для резолва клиентских компонентов, пришедших
с сервера.

## 5.3 `hoistable` — 8 вхождений

**Hoistable elements** — элементы, которые React «поднимает» в `<head>`:
`<link>`, `<meta>`, `<title>`.

**React 19** умеет рендерить `<title>`, `<meta>`, `<link>` из любого места
дерева — и автоматически переносить в `<head>`.

**`precedence`** (25 вхождений) — ключ для дедупликации стилей:
```jsx
<link rel="stylesheet" href="/a.css" precedence="default" />
<link rel="stylesheet" href="/a.css" precedence="default" />
```
→ вставится **один раз**.

## 5.4 `dispatchEvent` — 5 вхождений

**Обработка событий.** React 19 использует корневой делегированный
обработчик:
```js
container.addEventListener(eventType, dispatchEvent)
```

**`EventPriority`** — определение приоритета по типу события.

## 5.5 `unstable_` — 46 вхождений

**46 функций с префиксом `unstable_`** — экспериментальный API.
Список включает Form Actions, `useFormState`, транзишены.

---

# ЧАСТЬ 6. `react-dom` vs `react-dom/client`

## 6.1 Что экспортируется где

| API | `react-dom` | `react-dom/client` | Здесь |
|---|---|---|---|
| `createPortal` | ✅ | ❌ | ✅ |
| `flushSync` | ✅ | ❌ | ✅ |
| `preload` | ✅ | ❌ | ✅ |
| `preinit` | ✅ | ❌ | ✅ |
| `findDOMNode` | ✅ (deprecated) | ❌ | ✅ |
| `createRoot` | ❌ | ✅ | ✅ |
| `hydrateRoot` | ❌ | ✅ | ✅ |
| `useFormState` | ✅ | ❌ | ✅ |
| `useFormStatus` | ✅ | ❌ | ✅ |

**Здесь — ОБА в одном модуле.**

**Почему:** это **внутренняя** сборка (`react-dom-client.production.js`),
которая включает shared-часть.

## 6.2 Проблема с `run-react`

**`jGe()` в `transpile-Cld0n4VL.js` генерирует:**
```js
import { createRoot } from 'react-dom/client';
```

**А `esbuild` с `external: ["react", "react-dom", ...]`** оставит
`react-dom/client` как есть.

**В `mJe` (resolve):**
```js
x.onResolve({filter: /^[^./]/}, ({path: D}) => {
  const g = Aee(D);
  return {path: RB(`${Eee}/npm/${g}/+esm`), namespace: "external"}
});
```
`"react-dom/client"` → `https://cdn.jsdelivr.net/npm/react-dom/client@18.3.1/+esm`

**`RB()` пиннит версию 18.3.1**, но **путь** `react-dom/client` не
является валидным npm-пакетом.

**Результат:** `https://cdn.jsdelivr.net/npm/react-dom/client@18.3.1/+esm`
→ **404** (jsdelivr интерпретирует `react-dom` как пакет, `client` как
подпуть... но с `@18.3.1` после `client` — синтаксически неверно).

**Правильный URL:** `https://cdn.jsdelivr.net/npm/react-dom@18.3.1/client/+esm`

**Значит:** `RB()` ломает subpath-импорты.

**Проверить:** `RB()` regex:
```js
`((?:^|/))((?:npm/)${v})(@[^/]+)?(?=/|$|\\?)`
```
Для `https://cdn.jsdelivr.net/npm/react-dom/client/+esm`:
- `v` = `react-dom` (escaped)
- matches `npm/react-dom` 
- `(@[^/]+)?` — нет версии
- `(?=/|$|\?)` — смотрит на `/client` → `/` — подходит
- Замена: `npm/react-dom@18.3.1`
- Результат: `https://cdn.jsdelivr.net/npm/react-dom@18.3.1/client/+esm`

**Работает!** Я ошибся. `RB()` правильно вставляет версию перед подпутём.

**НО:** `Aee(D)` вызывается **до** `RB()`:
```js
const g = Aee(D);              // react-dom/client
return {path: RB(`${Eee}/npm/${g}/+esm`), ...}
```
`Aee` обрабатывает только lucide. `react-dom/client` не изменится.

**Итог:** `https://cdn.jsdelivr.net/npm/react-dom@18.3.1/client/+esm` — валидно.

**НО:** это **другой** `react-dom`, чем этот файл. То есть subpath-импорт
тянет **отдельный** экземпляр React.

---

# ЧАСТЬ 7. Проверка версии — `Error #527`

```js
if (R1 !== "19.3.0-canary-7dc903cd-20251203")
  throw Error(z(527, R1, "19.3.0-canary-7dc903cd-20251203"));
```

**Что это защищает:** `react` и `react-dom` должны быть **одной версии**.

**`R1` = `K1.version`** — версия присланного `react`.

**Если разные** — `Error #527`:
> «Incompatible React versions: The "react" and "react-dom" packages must
> have the exact same version.»

**Следствие:** если виджет импортирует `react` с CDN (`18.3.1`), а
`react-dom` — вшитый (`19.3.0-canary`), **упадёт с #527**.

**Проверить:** в `jGe()` генерируется `import {createElement, Component} from 'react'`
и `import {createRoot} from 'react-dom/client'`. Оба external → оба с CDN.
Значит версии совпадут (`18.3.1`) → **этот вшитый `react-dom` не
используется**.

**Вывод:** файл `client-BjybnHiv.js` загружается только при использовании
`runComponent` / `prepareEnvironment` для threejs (где R3F требует React 19).

---

# ЧАСТЬ 8. `Le()` — нормализация crossOrigin

```js
function Le(u, t) {
  if (u === "font") return "";
  if (typeof t === "string") return t === "use-credentials" ? t : "";
}
```

**Три случая:**
1. `as === "font"` → `""` (пустой — анонимный)
2. строка → `"use-credentials"` или `""`
3. иначе → `undefined`

**Почему шрифты особые:** `crossorigin` на `<link rel=preload as=font>`
обязателен. `""` = анонимный CORS.

**Любое значение кроме `"use-credentials"`** → `""` (анонимный).
То есть `"anonymous"` → `""` (то же самое).

---

# ЧАСТЬ 9. Схема

```
react-dom (client-BjybnHiv.js)
 │
 ├─ react-dom.production.js (shared)
 │   ├─ createPortal(children, container, key)
 │   ├─ flushSync(fn)
 │   ├─ preconnect(url, {crossOrigin})
 │   ├─ prefetchDNS(url)
 │   ├─ preinit(url, {as: "style"|"script", ...})     ← SRI!
 │   ├─ preinitModule(url, {...})
 │   ├─ preload(url, {as, ...9 опций})                ← SRI!
 │   ├─ preloadModule(url, {...})
 │   ├─ requestFormReset(form)
 │   ├─ unstable_batchedUpdates(fn, arg)              ← no-op
 │   ├─ useFormState(action, initial, permalink)
 │   ├─ useFormStatus()
 │   ├─ findDOMNode(component)                        ← deprecated
 │   └─ version
 │
 ├─ react-dom-client.production.js (body)
 │   ├─ reconciler
 │   ├─ event system (dispatchEvent ×5)
 │   ├─ hoistable elements (×8)
 │   ├─ precedence (×25)
 │   ├─ view transitions
 │   └─ client references (RSC)
 │
 ├─ createRoot(container, options)
 │   ├─ options: unstable_strictMode, identifierPrefix,
 │   │           onUncaughtError, onCaughtError, onRecoverableError
 │   ├─ qm(container, 1, !1, ...)     ← ConcurrentRoot, не hydrate
 │   ├─ container[Ea] = fiber.current
 │   └─ → new ReactDOMRoot(fiber)
 │
 └─ hydrateRoot(container, element, options)
     ├─ options: + formState
     ├─ qm(container, 1, !0, element, ...)   ← hydrate
     ├─ context = Gm(null)
     ├─ немедленный flush
     └─ → new HydratedRoot(fiber)
```

---

# ЧАСТЬ 10. Слабые места

## 10.1 Canary-версия

```js
Ve.version = "19.3.0-canary-7dc903cd-20251203"
```

**Не стабильный релиз.** Может содержать неотлаженные фичи.

**Для продакшена** — риск.

## 10.2 Жёсткая проверка версии

```js
if (R1 !== "19.3.0-canary-7dc903cd-20251203") throw Error(z(527, R1, ...))
```

**Любая другая версия `react`** → падение.

**В Skybridge:** `jGe()` генерирует `import from 'react'` (external) →
React 18.3.1 с CDN. Если этот `react-dom` загрузится — **#527**.

**Значит:** этот файл **не может** использоваться с кодом из `jGe()`.
Только с `runComponent`/`prepareEnvironment`-threejs, где React — вшитый.

## 10.3 `unstable_batchedUpdates` — no-op

```js
Nl.unstable_batchedUpdates = function (l, t) { return l(t) };
```

**Не батчит.** В React 18+ батчинг автоматический. Функция оставлена
для совместимости, но **не делает того, что обещает имя**.

**Проблема для legacy-кода:** если кто-то ожидает явного батчинга —
не получит.

## 10.4 `findDOMNode` — deprecated, но работает

**Может вернуть `null`** для функциональных компонентов.

**`Error #188`** и **`#268`** — непонятные для пользователя.

## 10.5 `flushSync` блокирует

**Синхронный flush** — блокирует main thread.

**При тяжёлом дереве** — заметный лаг.

## 10.6 `preload`/`preinit` с `integrity`

**Единственное место в React, где поддерживается SRI.**

**НО:** в `Skybridge`-контексте эти API не используются для загрузки
`esbuild-wasm`/`Pyodide` — они грузятся напрямую с CDN без SRI.

## 10.7 `Le()` для шрифтов всегда `""`

```js
if (u === "font") return "";
```

**Игнорирует переданный `crossOrigin`.** Для шрифтов всегда анонимный.

**Если нужен `use-credentials`** — невозможно.

## 10.8 `preinit` только для `style`/`script`

```js
u === "style" ? ... : u === "script" && ...
```

**Другие `as`** (`image`, `font`, `fetch`) — **молча игнорируются**.
Нет ошибки, нет предупреждения.

**Для `font`** — используйте `preload`, не `preinit`.

## 10.9 `hydrateRoot` устанавливает `context`

```js
t.context = Gm(null);
```

**`Gm(null)`** — пустой legacy-context.

**Legacy Context API** устарел (используйте `createContext`). Но React
оставил для совместимости.

## 10.10 `container[Ea]` — прямое свойство DOM

```js
l[Ea] = t.current;    // Ea = Symbol или ключ
```

**React пишет в DOM-элемент.** Если `Ea` — строка, видно в
`Object.keys(container)`.

**`Ea`** — либо `Symbol.for(...)`, либо `"__reactContainer$<random>"`.

## 10.11 `__REACT_DEVTOOLS_GLOBAL_HOOK__`

```js
if (typeof __REACT_DEVTOOLS_GLOBAL_HOOK__ < "u") {
  var $n = __REACT_DEVTOOLS_GLOBAL_HOOK__;
  if (!$n.isDisabled && $n.supportsFiber)
    try { Tn = $n.inject(K2), Xl = $n } catch {}
}
```

**Inject в DevTools.** Если hook есть — React регистрируется.

**`try {} catch {}`** — молча.

**В iframe** DevTools hook обычно **нет** → эта ветка не выполняется.

## 10.12 `checkDCE`

```js
function Vm() {
  if (!(typeof __REACT_DEVTOOLS_GLOBAL_HOOK__ > "u"
        || typeof __REACT_DEVTOOLS_GLOBAL_HOOK__.checkDCE != "function"))
    try { __REACT_DEVTOOLS_GLOBAL_HOOK__.checkDCE(Vm) } catch (l) { console.error(l) }
}
Vm()
```

**`checkDCE`** — проверка «dead code elimination». В production-сборке
эта функция должна быть вырезана бандлером. Если нет — React
предупреждает.

**Вызов `Vm()` при загрузке** — проверка целостности сборки.

---

# ЧАСТЬ 11. Итог

## 11.1 Ключевые числа

| Что | Значение |
|---|---|
| Размер | 208 365 байт |
| Версия | `19.3.0-canary-7dc903cd-20251203` |
| Дата коммита | 2025-12-03 |
| Экспортов | 2 (`a`, `c`) |
| React-модулей внутри | 3 (shared + client + scheduler) |
| Функций `unstable_` | 46 |
| Вхождений `precedence` | 25 |
| Вхождений `hoistable` | 8 |
| Вхождений `dispatchEvent` | 5 |
| Опций `preload` | 9 |
| Опций `preinit` | 6 |
| Типов контейнера | 3 (Element, Document, Fragment) |

## 11.2 Публичный API

**23 функции** в `Nl` (namespace `react-dom`):

`createPortal`, `flushSync`, `preconnect`, `prefetchDNS`, `preinit`,
`preinitModule`, `preload`, `preloadModule`, `requestFormReset`,
`unstable_batchedUpdates`, `useFormState`, `useFormStatus`, `version`,
`findDOMNode`, `__DOM_INTERNALS_...` + внутренние.

**Точки входа:** `createRoot`, `hydrateRoot`.

## 11.3 Главное

**1. Canary-сборка React 19.3.0 от 2025-12-03.**

**2. Жёсткая проверка версии — `Error #527` при несовпадении с `react`.**
Значит этот файл несовместим с `jGe()`-кодом (React 18.3.1).

**3. `react-dom` и `react-dom/client` — в одном файле.**
Нестандартная сборка.

**4. `createRoot` и `hydrateRoot` — полный разбор.**
`qm(container, 1, !1, ...)` для root, `qm(container, 1, !0, ...)` для hydrate.

**5. 7 опций `createRoot`:**
`unstable_strictMode`, `identifierPrefix`, `onUncaughtError`,
`onCaughtError`, `onRecoverableError`. Плюс `formState` для hydrate.

**6. `preload`/`preinit` поддерживают `integrity` (SRI) — единственное
место в React.**

**7. `unstable_batchedUpdates` — полный no-op.**
```js
Nl.unstable_batchedUpdates = function (l, t) { return l(t) };
```

**8. `Le()` для шрифтов всегда `""`** — `crossOrigin` игнорируется.

**9. `preinit` только для `style`/`script`** — другие `as` молча игнорируются.

**10. `findDOMNode` deprecated, но присутствует.**
`Error #188` / `#268` при невалидном аргументе.

**11. React 19 фичи: view transitions, client references (RSC),
hoistable elements с `precedence`.**

**12. `container[Ea]` — React пишет в DOM напрямую.**
