# `run-widget-code-RkUxf2Ko.js` — полный разбор

**Размер:** 3 455 байт
**URL:** `https://web-sandbox.oaiusercontent.com/assets/run-widget-code-RkUxf2Ko.js`
**Роль:** точка входа для **App Blocks / виджетов ChatGPT**. Собирает глобалы,
подключает MCP Apps adapter, запускает HTML в iframe.

---

# ЧАСТЬ 1. Контекст: чем отличается от `runUserCode`

| | `runUserCode` | `runWidgetCode` |
|---|---|---|
| Кто вызывает | чат (сгенерированный код) | **App Block** (виджет из ChatGPT) |
| `isFirstParty` | жёстко `false` | **параметр** |
| Глобалы | `hn({isFirstParty: false})` | `hn({isFirstParty, includePrivate: false, hostHandlesFollowUpMessageAuthorization})` |
| MCP Apps | нет | **есть** (`initMcpAppsAdapter`) |
| `appBlockMessenger` | нет | **есть** |
| CSP | `applyCsp(csp)` | `applyCsp(csp)` + `resetPreview()` |
| `prepareWidgetDocument` | нет | **есть** (из приватного API) |
| Языки | react/recharts/python/html/threejs | **только HTML** |

**`runWidgetCode` — «взрослый» запуск. `runUserCode` — песочница для чата.**

---

# ЧАСТЬ 2. Импорты

```js
import {a as Q} from "./apply-csp-_9OvJz9C.js";           // applyCsp
import {S as G, a as U, b as X, d as H, g as Z, L as $,
        e as z, f as F, r as m, o as A, h as p} from "./main-BR1bOQyH.js";
import {initMcpAppsAdapter as ee} from "./adapter-CQcBVlDI.js";
```

| Локальное | Экспорт | Реальное имя | Что |
|---|---|---|---|
| `Q` | `a` | `N` | `applyCsp` |
| `G` | `S` | `Ne` | `"openai:set_globals"` (строка) |
| `U` | `a` | `to` | класс `SetGlobalsEvent` |
| `X` | `b` | `xr` | `resetPreview()` — убить все iframe `#root` |
| `H` | `d` | `X` | `hn({isFirstParty:true, includePrivate:true})` |
| `Z` | `g` | `hn` | публикация Host API |
| `$` | `L` | `Mn` | локаль (`Mt.get("locale") ?? ""`) |
| `z` | `e` | `Ms` | `widgetStateManager` |
| `F` | `f` | `Ps` | `viewParams` → `{displayMode, view}` |
| `m` | `r` | `Dr` | `runHTMLAsync` |
| `A` | `o` | `yi` | `autofillObserver` |
| `p` | `h` | `mi` | `getGlobalSurface()` → `Sr` |
| `ee` | — | `initMcpAppsAdapter` | MCP Apps |

**Критично `H` = `X()`** — это `hn({isFirstParty: true, includePrivate: true})`,
то есть **приватный** API. Здесь он доступен, потому что `runWidgetCode`
работает в контексте хоста.

---

# ЧАСТЬ 3. `te()` — фон surface

```js
function te({document: i, backgroundColor: v, viewWindow: g}) {
  const o = i.createElement("style");
  let b = !0;

  function u() {
    b && o.textContent !== "" && !o.isConnected && (i.head?.append(o))
  }

  function f(a) {
    o.textContent = a === "transparent"
      ? "html, body { background: transparent !important; }"
      : "";
    a === "transparent" ? u() : o.remove()
  }

  function d(a) {
    if (!(a instanceof U)) return;
    const C = a.detail.globals.surfaceBackgroundColor;
    typeof C === "string" && f(C)
  }

  f(v);

  const h = new MutationObserver(u);
  h.observe(i, {childList: !0, subtree: !0});
  queueMicrotask(u);

  g.addEventListener(G, d);

  return () => {
    b = !1;
    h.disconnect();
    o.remove();
    g.removeEventListener(G, d);
  };
}
```

## 3.1 Что делает

Управляет **прозрачностью фона** iframe. Если хост говорит
`surfaceBackgroundColor: "transparent"` — добавляется `<style>` с
`background: transparent !important`.

## 3.2 Пошагово

**`o = i.createElement("style")`** — создать `<style>`.

**`b = !0`** — флаг «жив».

**`u()` — вставка стиля:**
```js
b && o.textContent !== "" && !o.isConnected && (i.head?.append(o))
```
Три условия:
1. `b` — компонент ещё жив
2. `o.textContent` не пустой (есть что вставлять)
3. `!o.isConnected` — ещё не вставлен

**`f(a)` — установить цвет:**
```js
o.textContent = a === "transparent"
  ? "html, body { background: transparent !important; }"
  : "";
a === "transparent" ? u() : o.remove()
```

- `"transparent"` → CSS + вставить
- иначе → очистить текст + удалить из DOM

**`d(a)` — обработчик события:**
```js
if (!(a instanceof U)) return;
const C = a.detail.globals.surfaceBackgroundColor;
typeof C === "string" && f(C)
```
`U` = `to` = класс `openai:set_globals` CustomEvent. Проверка на
`instanceof` — защита от поддельных событий.

**`h = new MutationObserver(u)`** — если `<head>` пересоздали (React мог
сделать `innerHTML`), стиль будет вставлен заново.

**`queueMicrotask(u)`** — попытаться вставить сразу после текущего тика.

**`g.addEventListener(G, d)`** — слушать `openai:set_globals` на `viewWindow`
(= `iframe.contentWindow`).

## 3.3 Зачем MutationObserver

Пользовательский HTML может сделать `document.head.innerHTML = "..."`,
и наш `<style>` исчезнет. Observer вернёт его.

**Но:** если пользователь намеренно удаляет — стиль вернётся. Это может
использоваться для обхода прозрачности (не критично).

## 3.4 Утечка

```js
return () => {
  b = !1;
  h.disconnect();
  o.remove();
  g.removeEventListener(G, d);
};
```

Cleanup корректен. `b = false` перед `disconnect` — чтобы `u()` не сработал
в гонке.

---

# ЧАСТЬ 4. Сигнатура `runWidgetCode`

```js
async function* ie(i, {
  appBlockMessenger: v,
  html: g,
  isFirstParty: o,
  hostHandlesFollowUpMessageAuthorization: b,
  disableExternalNavigation: d,          // ← в выводе было обрезано
  measureHeight: C,
  maxWidth: R,
  toolInput: _,
  toolOutput: S,
  toolResponseMetadata: q,
  subjectId: T = null,
  appPackageName: W,
  supportsOAuthCompletion: x,
  supportsSourceCitationLinks: V,
  isSidebarOpen: j,
  displayMode: E,
  measureWidth: I,
  expectReadySignal: L = !1,
  expectedReadySignal: w,
  theme: k,
  userAgent: B,
  safeArea: D,
  isTombstone: N = !1,
  features: Y,
  screenshotVegaPreview: oe,
  csp: J,
  mcpApps: e
})
```

**25 параметров.** Для сравнения `runUserCode` — 12.

| Параметр | Назначение |
|---|---|
| `i` | `SandboxLogger` (первый, вне объекта) |
| `appBlockMessenger` | мост для App Block сообщений |
| `html` | готовый HTML виджета |
| `isFirstParty` | **ключевой флаг** |
| `hostHandlesFollowUpMessageAuthorization` | хост сам авторизует `sendFollowUpMessage` |
| `disableExternalNavigation` | запрет навигации |
| `measureHeight` | замер высоты |
| `maxWidth` | максимальная ширина |
| `toolInput` | входные данные инструмента (то, что вызвал модель) |
| `toolOutput` | выходные данные (что вернул инструмент) |
| `toolResponseMetadata` | метаданные ответа |
| `subjectId` | ID субъекта (для персонализации) |
| `appPackageName` | имя пакета приложения |
| `supportsOAuthCompletion` | может завершать OAuth |
| `supportsSourceCitationLinks` | может давать ссылки на цитаты |
| `isSidebarOpen` | боковая панель открыта |
| `displayMode` | `inline`/`fullscreen`/`pip` |
| `measureWidth` | замер ширины |
| `expectReadySignal` | ждать `data-ready` |
| `expectedReadySignal` | конкретное значение `data-ready` |
| `theme` | `light`/`dark` |
| `userAgent` | UA для инъекции |
| `safeArea` | отступы (чёлка, жесты) |
| `isTombstone` | виджет «мёртв» (заглушка) |
| `features` | фича-флаги |
| `screenshotVegaPreview` | скриншот Vega-графика |
| `csp` | CSP-конфиг |
| `mcpApps` | конфиг MCP Apps |

---

# ЧАСТЬ 5. Тело функции

## 5.1 CSP + сброс

```js
try {
  Q(J)
} catch (l) {
  throw i.error("Failed to apply CSP", l), l
}

(e || d) && X();
```

**`Q(J)`** — `applyCsp(csp)`. Ослабление → throw.

**`(e || d) && X()`** — если **есть MCP Apps** (`e`) **или**
`disableExternalNavigation` (`d`) → вызвать `X()` = `resetPreview()`.

**Что делает `resetPreview()` (`xr`):**
```js
xr = () => {
  document.querySelectorAll(`iframe#${Dd}`).forEach(an),   // удалить все iframes
  Yn()                                                      // сбросить color-scheme
};
```

`an`:
```js
an = t => {
  t.dispatchEvent(new Event("preview-cancel")),
  t.remove()
};
```

**Зачем при MCP Apps:** MCP Apps-виджеты изолированы — старые iframe
могут мешать. Сброс гарантирует чистый старт.

**Зачем при `disableExternalNavigation`:** та же логика — режим строгий,
нужен чистый preview.

## 5.2 Сборка глобалов

```js
let t, P;
try {
  P = H().prepareWidgetDocument,
  t = Z({
    isFirstParty: o,
    includePrivate: !1,
    hostHandlesFollowUpMessageAuthorization: b
  }),
  t.__appBlockMessenger = v,
  t.maxHeight = C,
  t.maxWidth = R,
  t.theme = k,
  t.locale = $,
  t.userAgent = B,
  t.appPackageName = o ? W : void 0,
  t.supportsOAuthCompletion = o && x === !0,
  t.supportsSourceCitationLinks = o && V === !0,
  (e?.hostInfo) != null && (t.hostInfo = e.hostInfo),
  t.isSidebarOpen = j ?? !1;

  t.widget = {
    state: h,
    props: S,
    setState: c => {
      console.warn("window.openai.widget is deprecated. It will be removed in future versions.");
      return H()?.updateWidgetState?.call(H(), f, c)
    }
  };

  const l = z({
    widgetId: f,
    globals: {toolInput: _, toolOutput: S, toolResponseMetadata: q, widgetState: h, subjectId: T},
    sandboxLogger: i
  });

  Object.assign(t, l, F({displayMode: E, viewParams: a, isTombstone: N}))
} catch {}
```

### `H().prepareWidgetDocument`

**`H()` = `X()`** = `hn({isFirstParty: true, includePrivate: true})`.

**`prepareWidgetDocument`** — приватный метод (отфильтровывается в
публичном API). Возвращает функцию, которая готовит документ iframe.

### `Z({isFirstParty: o, includePrivate: false, ...})`

**`Z` = `hn`.** Здесь `includePrivate: false` — то есть private-методы
**отфильтрованы**, но `isFirstParty` — какой передал хост.

**Важно:** если `o = true` (first-party виджет) — фильтр `!Jr.has(o)`
пропускает **все** методы, включая 31 из `Jr` и `prepareWidgetDocument`
(при `includePrivate: false` `prepareWidgetDocument` **всё равно отфильтрован**).

**Проверим фильтр:**
```js
.filter(o => (o !== "prepareWidgetDocument" || includePrivate)
          && (t || includePrivate || !Jr.has(o)))
```

- `t = isFirstParty`
- `e = includePrivate = false`

При `t = true, e = false`:
- `(o !== "prepareWidgetDocument" || false)` → `prepareWidgetDocument` **отсеян**
- `(true || ...)` → всегда `true` → остальные методы проходят

**Значит:** first-party виджет получает **все** методы **кроме**
`prepareWidgetDocument`. И `prepareWidgetDocument` берётся отдельно
через `H()` (`X()`) — с `includePrivate: true`.

### Присвоение полей

```js
t.__appBlockMessenger = v,
t.maxHeight = C,
t.maxWidth = R,
t.theme = k,
t.locale = $,
t.userAgent = B,
t.appPackageName = o ? W : void 0,          // ← только first-party
t.supportsOAuthCompletion = o && x === !0,   // ← только first-party
t.supportsSourceCitationLinks = o && V === !0,  // ← только first-party
(e?.hostInfo) != null && (t.hostInfo = e.hostInfo),
t.isSidebarOpen = j ?? !1;
```

**Только first-party получает:**
- `appPackageName`
- `supportsOAuthCompletion`
- `supportsSourceCitationLinks`

**Все получают:** `maxHeight`, `maxWidth`, `theme`, `locale`, `userAgent`,
`isSidebarOpen`, `__appBlockMessenger`.

**`e?.hostInfo`** — из MCP Apps-конфига.

### `window.openai.widget` — deprecated

```js
t.widget = {
  state: h,
  props: S,
  setState: c => {
    console.warn("window.openai.widget is deprecated. It will be removed in future versions.");
    return H()?.updateWidgetState?.call(H(), f, c)
  }
};
```

**Три поля:**
- `state` — текущее состояние
- `props` — `toolOutput`
- `setState(fn)` — **устаревший** способ, вызывает `updateWidgetState`

**`console.warn` при каждом вызове** — виджет засорит консоль.

### `Ms()` — widgetStateManager

```js
const l = z({
  widgetId: f,
  globals: {toolInput: _, toolOutput: S, toolResponseMetadata: q, widgetState: h, subjectId: T},
  sandboxLogger: i
});
```

**`z` = `Ms`** (из main.js):

```js
Ms = ({widgetId: t, globals: e, sandboxLogger: n}) => {
  const {toolInput: r, toolOutput: o, toolResponseMetadata: s,
         widgetState: i, subjectId: l = null} = e;

  let a = i;                    // локальная копия состояния

  const u = async (h, d) => {   // setState
    const g = typeof h == "function" ? h(a) : h;
    a = g;
    Xt({additionalGlobals: {widgetState: g}, sandboxLogger: n});   // инъекция
    const E = await X()?.updateWidgetState?.call(X(), t, g);        // хост
    ...
  };

  return {toolInput: r, toolOutput: o, toolResponseMetadata: s,
          widgetState: a, subjectId: l, setWidgetState: u};
};
```

**Что делает:**
1. Хранит **локальную** копию `widgetState`
2. `setWidgetState(fn|value)`:
   - вычисляет новое значение (функция или значение)
   - **инжектирует** в iframe через `Xt({additionalGlobals: {widgetState: g}})`
   - **отправляет** на хост через `updateWidgetState`

**Двойная синхронизация:** локально (для мгновенного отклика) + на хост
(для персистентности).

### `Ps()` — viewParams

```js
Ps = ({displayMode: t, viewParams: e, isTombstone: n = !1}) => ({
  displayMode: t,
  view: {params: e ?? null, mode: t, isTombstone: n}
});
```

**`window.openai.view`** — три поля: `params`, `mode`, `isTombstone`.

### `catch {}` — молча

Всё в одном `try/catch`. Если `hn()` бросит `"Host API not published"` —
`t` останется `undefined`, и дальше `t.__appBlockMessenger = v` бросит
`TypeError`, который тоже проглотится... нет, `catch` вокруг всего блока,
значит `t` останется `undefined` частично заполненным.

**Реально:** первая строка `t = Z(...)` бросит — все последующие
присвоения не выполнятся. `t = undefined`.

---

# ЧАСТЬ 6. MCP Apps

```js
const s = e?.requiresInitialization ? Promise.withResolvers() : void 0;
```

**`Promise.withResolvers()`** — новый API (TC39, ES2024). Возвращает
`{promise, resolve, reject}`.

**Если `mcpApps.requiresInitialization = true`** — создаётся обещание,
которое резолвится, когда адаптер инициализирован.

**Браузерная поддержка:** Chrome 119+, Firefox 121+, Safari 17.4+.
В старых — `TypeError: Promise.withResolvers is not a function`.

---

# ЧАСТЬ 7. Вызов `runHTMLAsync`

```js
yield* m({
  sandboxLogger: i,
  html: g,
  theme: k,
  features: Y,
  additionalGlobals: t,
  safeArea: D,
  transformPosition: null,
  expectReadySignal: L,
  expectedReadySignal: w,
  isFirstParty: o,
  forwardConsoleLogs: !e,             // ← выключается при MCP Apps
  measureHeight: !0,                  // ← ВСЕГДА true
  measureWidth: I,
  disableMutationObserver: !L && o && !I,
  disableExternalNavigation: d,
  allowPopups: !d,
  onIframeReady: (e || d) ? ({viewWindow: l, document: c}) => {...} : void 0,
  appInitialization: s?.promise,
  prepareDocument: P
});
```

## 7.1 Параметры

| Параметр | Значение | Откуда |
|---|---|---|
| `forwardConsoleLogs` | `!e` | **выключается** при MCP Apps |
| `measureHeight` | `true` | **всегда** |
| `disableMutationObserver` | `!L && o && !I` | расчётное |
| `allowPopups` | `!d` | **инверсия** `disableExternalNavigation` |
| `onIframeReady` | функция если `e \|\| d` | MCP Apps или запрет навигации |

### `forwardConsoleLogs: !e`

**При MCP Apps (`e` есть) — `false`.** Консоль виджета **не** пробрасывается
в главный поток.

**Почему:** MCP Apps — сторонние приложения, их `console.log` может
содержать чувствительные данные.

### `measureHeight: true` — всегда

В отличие от `runUserCode`, где `html`-ветка ставит `measureHeight: true`,
а `react` — тоже `true`. Но здесь **жёстко**, без опции.

### `disableMutationObserver: !L && o && !I`

```
!expectReadySignal && isFirstParty && !measureWidth
```

**MutationObserver отключается**, если:
1. Не ждём `data-ready` (`!L`)
2. First-party виджет (`o`)
3. Не меряем ширину (`!I`)

**Зачем:** `MutationObserver` на всём документе — дорого. Если виджет
не ждёт сигнала готовности и first-party — можно сэкономить.

**НО:** `MutationObserver` в `runHTMLAsync` также отслеживает
`data-fatal` (для завершения при ошибке). Отключение = **не будет
автоматического завершения по `data-fatal`**.

## 7.2 `onIframeReady`

```js
onIframeReady: (e || d) ? ({viewWindow: l, document: c}) => {
  const r = d ? A({document: c, viewWindow: l}) : void 0,   // autofill observer
        n = e?.globalSurface ? te({                              // surface background
          document: c,
          backgroundColor: e?.hostContext.styles?.variables?.["--color-background-primary"],
          viewWindow: l
        }) : void 0;

  let y = null;

  if (e) try {
    y = ee({                                                    // MCP Apps adapter
      onInitialized: s?.resolve,
      onInitializationError: s?.reject,
      viewWindow: l,
      forwardedRequestMethods: e.forwardedRequestMethods,
      hostContext: e.hostContext,
      hostCapabilities: e.hostCapabilities,
      hostInfo: e.hostInfo,
      getSetWidgetState: p,
      sandboxLogger: i
    }).destroy
  } catch (K) {
    throw r?.(), n?.(), K
  }

  return () => {
    r?.();
    n?.();
    y?.()
  }
} : void 0
```

### Три подписчика

**1. `A()` — autofill observer** (если `d`, то есть запрет навигации)

```js
yi = ({document: t, viewWindow: e}) => {
  let n = null;

  const r = () => {
    X()?.notifyAutofillCandidates?.(vs(t, e))    // собрать поля с autocomplete
  };

  const o = new MutationObserver(() => {
    n ??= e.requestAnimationFrame(() => { n = null; r() });
  });

  o.observe(t, {
    attributes: !0,
    attributeFilter: ["autocomplete", "disabled", "hidden", "readonly", "style", "type"],
    childList: !0,
    subtree: !0
  });

  return () => o.disconnect();
};
```

**Следит за атрибутами** `autocomplete`, `disabled`, `hidden`, `readonly`,
`style`, `type` на всех элементах. При изменении — отправляет хосту
список автозаполняемых полей.

**Зачем:** ChatGPT может подставить данные пользователя (имя, адрес,
карту) в формы виджета. Нужно знать, какие поля есть.

**Побочный эффект:** `notifyAutofillCandidates` — **приватный** метод
(`Jr`), вызывается через `X()` (`hn({includePrivate:true})`).

**2. `te()` — surface background** (если `e.globalSurface`)

Прозрачность фона из `hostContext.styles.variables["--color-background-primary"]`.

**3. `ee()` — MCP Apps adapter**

```js
y = ee({
  onInitialized: s?.resolve,
  onInitializationError: s?.reject,
  viewWindow: l,
  forwardedRequestMethods: e.forwardedRequestMethods,
  hostContext: e.hostContext,
  hostCapabilities: e.hostCapabilities,
  hostInfo: e.hostInfo,
  getSetWidgetState: p,
  sandboxLogger: i
}).destroy
```

**`initMcpAppsAdapter`** — из `adapter-CQcBVlDI.js`. Создаёт мост
MCP Apps-протокола (`2026-01-26`) между iframe и хостом.

**Возвращает объект с `.destroy`.**

**`getSetWidgetState: p`** — `p` = `mi()` = `Sr`. Это глобальная ссылка
на `setWidgetState`, установленная ранее.

**Cleanup:**
```js
return () => { r?.(); n?.(); y?.() }
```
Все три подписчика отключаются.

**При ошибке создания адаптера:** `r?.(), n?.(), K` — отключить
autofill и background, пробросить ошибку.

---

# ЧАСТЬ 8. Итоговая схема

```
runWidgetCode(logger, params)
 │
 ├─ applyCsp(csp)                      ─┬─ ослабление → throw
 │
 ├─ (mcpApps || disableExternalNavigation) && resetPreview()
 │    ├─ удалить все iframe#root
 │    └─ сбросить color-scheme
 │
 ├─ prepareWidgetDocument = hn({isFirstParty:true, includePrivate:true}).prepareWidgetDocument
 │
 ├─ globals = hn({isFirstParty: params.isFirstParty, includePrivate: false,
 │                 hostHandlesFollowUpMessageAuthorization})
 │    ├─ __appBlockMessenger
 │    ├─ maxHeight, maxWidth, theme, locale, userAgent
 │    ├─ appPackageName        ← только first-party
 │    ├─ supportsOAuthCompletion ← только first-party
 │    ├─ supportsSourceCitationLinks ← только first-party
 │    ├─ hostInfo             ← из mcpApps
 │    ├─ isSidebarOpen
 │    ├─ widget: {state, props, setState}  ← deprecated
 │    └─ Object.assign(..., widgetStateManager, viewParams)
 │
 ├─ appInitialization = mcpApps?.requiresInitialization
 │                       ? Promise.withResolvers()
 │                       : undefined
 │
 └─ yield* runHTMLAsync({
      html, additionalGlobals: globals,
      expectReadySignal, expectedReadySignal,
      isFirstParty,
      forwardConsoleLogs: !mcpApps,
      measureHeight: true,
      measureWidth,
      disableMutationObserver: !expectReadySignal && isFirstParty && !measureWidth,
      disableExternalNavigation,
      allowPopups: !disableExternalNavigation,
      onIframeReady:
        ├─ autofillObserver (если disableExternalNavigation)
        ├─ surfaceBackground (если mcpApps.globalSurface)
        └─ initMcpAppsAdapter (если mcpApps)
      appInitialization,
      prepareDocument
    })
```

---

# ЧАСТЬ 9. Приватный API — что доступно

## 9.1 Два способа получить

```js
X()  // = hn({isFirstParty: true, includePrivate: true}) — ВСЁ
Z()  // = hn({isFirstParty: o, includePrivate: false, ...}) — по флагу
```

## 9.2 31 метод из `Jr` (скрыт от `isFirstParty: false`)

```js
Jr = new Set([
  "scanCard",
  "sendInstrument",
  "notifySecurityPolicyViolation",
  "notifyEnvironmentError",
  "notifyIntrinsicHeight",
  "notifyBackgroundColor",
  "notifyNavigation",
  "notifyEscapeKey",
  "notifyAppBlockMessage",
  "notifyAutofillCandidates",
  "drawBoundingBoxes",
  "getDownloadURL",
  "openExternalForm",
  "openPromptInput",
  "openConversationOverlay",
  "openPulseOverlay",
  "openPulseShare",
  "openPlaidLink",
  "openAppleHealthSettings",
  "ensureAppleHealthOnboarding",
  "requestAppleHealthPermissions",
  "shouldRequestAppleHealthPermissions",
  "showToast",
  "triggerHaptic",
  "updateWidgetState",
  "callMcp",
  "initializeMcpApp",
  "requestConnectPlugin",
  "requestReconnectConnector",
  "requestDeclinePluginSuggestions",
  "requestOpenPlugin"
])
```

**Что это за методы:**

| Метод | Что делает |
|---|---|
| `scanCard` | сканировать карту камерой |
| `sendInstrument` | отправить метрику |
| `notifySecurityPolicyViolation` | сообщить о нарушении CSP |
| `notifyEnvironmentError` | ошибка окружения |
| `notifyIntrinsicHeight` | высота виджета |
| `notifyBackgroundColor` | цвет фона |
| `notifyNavigation` | навигация |
| `notifyEscapeKey` | нажатие Escape |
| `notifyAppBlockMessage` | сообщение App Block |
| `notifyAutofillCandidates` | поля автозаполнения |
| `drawBoundingBoxes` | рисовать рамки (для CV) |
| `getDownloadURL` | URL для скачивания |
| `openExternalForm` | внешняя форма |
| `openPromptInput` | поле ввода промпта |
| `openConversationOverlay` | оверлей разговора |
| `openPulseOverlay` | оверлей Pulse |
| `openPulseShare` | шаринг Pulse |
| `openPlaidLink` | Plaid (банковские счета) |
| `openAppleHealthSettings` | настройки Apple Health |
| `ensureAppleHealthOnboarding` | онбординг Apple Health |
| `requestAppleHealthPermissions` | разрешения Apple Health |
| `shouldRequestAppleHealthPermissions` | проверка разрешений |
| `showToast` | показать тост |
| `triggerHaptic` | вибрация |
| `updateWidgetState` | обновить состояние |
| `callMcp` | **вызвать MCP-инструмент** |
| `initializeMcpApp` | инициализировать MCP App |
| `requestConnectPlugin` | подключить плагин |
| `requestReconnectConnector` | переподключить коннектор |
| `requestDeclinePluginSuggestions` | отклонить предложения плагинов |
| `requestOpenPlugin` | открыть плагин |

## 9.3 8 методов из `ao` (требуют user gesture)

```js
ao = new Set([
  "requestDisplayMode",
  "sendFollowUpMessage",
  "openExternal",
  "openExternalForm",
  "openPlaidLink",
  "requestPrompt",
  "requestTargetedReply",
  "triggerHaptic"
])
```

**Для этих методов** — проверка `navigator.userActivation.isActive`:

```js
!t && ao.has(s) && !(s === "sendFollowUpMessage" && n) && !uo()
  ? console.warn(`Method \`${s}\` called without synchronous user event`)
  : ho(s, r, i)
```

- `!t` — не first-party
- `ao.has(s)` — метод требует жеста
- `!(s === "sendFollowUpMessage" && hostHandlesFollowUpMessageAuthorization)` — не исключение
- `!uo()` — нет активного user activation

**Тогда:** `console.warn` и **метод не вызывается** (возвращает `undefined`).

**`uo()`:**
```js
uo = () => "userActivation" in window.navigator
          ? window.navigator.userActivation.isActive
          : !0    // ← если API нет — считаем активным
```

**Fallback `true`** — в браузерах без `userActivation` проверка пропускается.

## 9.4 Что доступно виджету в итоге

**Обычный виджет (`isFirstParty: false`):**
- 8 методов из `ao` (с проверкой жеста)
- методы, не входящие ни в `Jr`, ни в `ao`
- **`notifyIntrinsicWidth`** (не в `Jr`!)
- поля: `toolInput`, `toolOutput`, `toolResponseMetadata`, `widgetState`,
  `subjectId`, `theme`, `locale`, `userAgent`, `maxHeight`, `maxWidth`,
  `isSidebarOpen`, `__appBlockMessenger`, `widget`, `displayMode`, `view`

**First-party виджет (`isFirstParty: true`):**
- **все** методы из `Jr` (31) — включая `callMcp`, `updateWidgetState`, `showToast`
- все 8 из `ao` **без** проверки жеста
- плюс `appPackageName`, `supportsOAuthCompletion`, `supportsSourceCitationLinks`
- плюс `hostInfo`

**`prepareWidgetDocument`** — **никогда** через `Z()`, только через `H()` (= `X()`).

---

# ЧАСТЬ 10. Слабые места

## 10.1 `catch {}` вокруг всей сборки глобалов

```js
try {
  P = H().prepareWidgetDocument,
  t = Z({...}),
  t.__appBlockMessenger = v,
  ...
} catch {}
```

**Если `Z()` бросит** (хост не инициализирован) — `t = undefined`, все
последующие присвоения не выполнятся, `catch` проглотит.

**Дальше:** `additionalGlobals: t` = `undefined` → `runHTMLAsync` не
инжектирует глобалы. Виджет останется без `window.openai`.

**Молча.**

## 10.2 `disableMutationObserver` ломает обработку `data-fatal`

```js
disableMutationObserver: !L && o && !I
```

Если `expectReadySignal = false` и first-party и не меряем ширину —
observer отключён.

**В `runHTMLAsync`** observer также ловит:
```js
if (Pt.hasAttribute("data-fatal")) { Ct(); return }
```

`data-fatal` ставится при ошибке в React-обёртке (`jGe`). **Без observer
ошибка не будет обнаружена**, виджет останется «висящим».

## 10.3 `widget.setState` — deprecated, но вызывается

```js
console.warn("window.openai.widget is deprecated. It will be removed in future versions.");
```

**При каждом вызове** — предупреждение в консоли. Если виджет вызывает
часто (например, на каждое движение мыши) — консоль засорится.

## 10.4 `Promise.withResolvers()` — не везде

```js
const s = e?.requiresInitialization ? Promise.withResolvers() : void 0;
```

**Chrome 119+, Firefox 121+, Safari 17.4+.** В более старых — `TypeError`.
Fallback нет.

**Дальше:** `s?.promise` в `runHTMLAsync`, `s?.resolve`/`s?.reject`
в адаптере. Если `withResolvers` недоступен — всё падает сразу.

## 10.5 `autofillObserver` работает только при `disableExternalNavigation`

```js
const r = d ? A({document: c, viewWindow: l}) : void 0
```

**`d` = `disableExternalNavigation`.** То есть автозаполнение
отслеживается **только** в строгом режиме.

**Если `disableExternalNavigation = false`** — `r = undefined`,
наблюдателя нет. ChatGPT не узнает о полях формы.

**Странная логика:** автозаполнение нужно как раз в обычном режиме
(форма заказа), а не в строгом.

## 10.6 `resetPreview()` при `disableExternalNavigation`

```js
(e || d) && X();
```

**Убивает все iframe `#root`** при каждом вызове с `disableExternalNavigation`.

**Проблема:** если в чате несколько виджетов и один из них запускается
строго — все остальные **снесутся**.

## 10.7 `measureHeight: true` жёстко

```js
measureHeight: !0,
```

**Нельзя отключить.** Даже если виджет fixed-height.

## 10.8 `forwardConsoleLogs: !e`

**При MCP Apps консоль не пробрасывается.** Отладка невозможна.

## 10.9 `onIframeReady` при ошибке адаптера

```js
} catch (K) {
  throw r?.(), n?.(), K
}
```

**`throw r?.(), n?.(), K`** — comma-оператор: сначала cleanup, потом throw.

**Но:** `r` и `n` могут быть `undefined` — `?.()` защищает.

**Проблема:** если `y` (адаптер) частично создан и бросил внутри —
его `destroy` не вызовется (переменная `y` не присвоена).

## 10.10 25 параметров — сложность

Огромная сигнатура. Трудно отследить, что откуда приходит.

---

# ЧАСТЬ 11. Сравнение с `runUserCode`

| Аспект | `runUserCode` | `runWidgetCode` |
|---|---|---|
| Языки | 7 | **только HTML** |
| `isFirstParty` | `false` жёстко | параметр |
| `prepareWidgetDocument` | ❌ | ✅ (через `X()`) |
| MCP Apps | ❌ | ✅ |
| `appBlockMessenger` | ❌ | ✅ |
| `forwardConsoleLogs` | всегда | `!mcpApps` |
| `measureHeight` | параметр | **всегда true** |
| `resetPreview` | ❌ | при `mcpApps \|\| disableNav` |
| Приватные методы | ❌ | ✅ при `isFirstParty` |
| `widgetState` синхронизация | ❌ | ✅ (`Ms`) |
| Autofill | ❌ | ✅ (при `disableNav`) |
| Surface background | ❌ | ✅ (при `mcpApps.globalSurface`) |
| Параметров | 12 | **25** |

---

# ЧАСТЬ 12. Итог

## 12.1 Ключевые числа

| Что | Значение |
|---|---|
| Размер | 3 455 байт |
| Параметров | 25 |
| Методов в `Jr` (скрытых) | 31 |
| Методов в `ao` (жест) | 8 |
| Уровней глобалов | 2 (`X()` всё, `Z()` по флагу) |
| Подписчиков в `onIframeReady` | 3 (autofill, surface, MCP) |

## 12.2 Главное

**1. `runWidgetCode` — «взрослый» путь.**
First-party виджеты получают **все** 31 метод из `Jr`, включая `callMcp`
и `updateWidgetState`.

**2. `prepareWidgetDocument` — только через `X()` (`includePrivate: true`).**
Через `Z()` (тот, что инжектируется в виджет) — недоступен всегда.

**3. `catch {}` вокруг сборки глобалов — молчаливое падение.**
Если хост не инициализирован, виджет останется без `window.openai`
и без сообщения.

**4. `disableMutationObserver` ломает `data-fatal`.**
При `!expectReadySignal && isFirstParty && !measureWidth` — observer отключён,
ошибка React не обнаружится.

**5. Autofill только при `disableExternalNavigation`.**
Странно: в обычном режиме (форма заказа) автозаполнение не отслеживается.

**6. `resetPreview()` при `disableExternalNavigation` сносит все виджеты.**
Если в чате несколько и один строгий — остальные умрут.

**7. `Promise.withResolvers()` без fallback.**
Safari < 17.4, Firefox < 121 → `TypeError`.

**8. `measureHeight: true` жёстко.**
Нельзя отключить.

**9. `widget.setState` deprecated, но с `console.warn` на каждый вызов.**

**10. `forwardConsoleLogs: !mcpApps`** — MCP Apps нельзя отлаживать.
