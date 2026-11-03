# `main-BR1bOQyH.js` — доскональный разбор

**Размер:** 89 639 байт
**URL:** `https://web-sandbox.oaiusercontent.com/assets/main-BR1bOQyH.js`
**Роль:** ядро Skybridge. Всё остальное — подключаемые модули.
**Прочитан целиком:** строки 1–86 000 (весь файл).

---

## 0. Состав файла

Файл — это один модуль, внутри которого лежат **вшитые библиотеки**:

| Диапазон (символы) | Что |
|---|---|
| 0 – 3 700 | Vite-преамбула: `__vite__mapDeps`, modulepreload polyfill |
| 3 700 – 5 200 | Константы протокола, валидаторы `ze`, `Kr`, `pi` |
| 5 200 – 7 600 | Классы событий, списки `Jr` и `ao`, параметры URL |
| 7 600 – 19 000 | **`culori`** — цветовые пространства (LAB, LCH, OKLab, P3, Rec2020, AdobeRGB, ProPhoto) |
| 19 000 – 31 700 | `culori`-утилиты: `alpha`, `darken`, `lighten`, `blend`, `getLuminance` |
| 31 700 – 34 000 | Измерение элементов, выделение компонентов (`As`), автозаполнение |
| 34 000 – 44 000 | `rt` (SandboxLogger), python-worker, deep clone (`gn`, `Us`) |
| 44 000 – 46 000 | RPC (`St`, `xn`), `Hs()` (targetOrigin), `$s()` (мост) |
| 46 000 – 60 000 | **`lru-cache`** целиком + `AbortController` polyfill |
| 60 000 – 77 700 | Вспомогательные: замеры высоты/ширины, стек |
| 77 707 – 82 400 | **`li()`** — публикация messenger/`__bindings`/`__effect` |
| 82 400 – 84 300 | **`Dr()`** — `runHTMLAsync`, главный рендер документа |
| 84 300 – 86 500 | Фасад `wt` — публичный API песочницы |
| 86 500 – конец | Экспорты |

---

## 1. Константы

```js
Fe = /^[A-Za-z][A-Za-z0-9_]{0,63}$/    // имя binding
Ur = 32                                  // максимум bindings+callbacks
Wr = 8                                   // максимальная глубина объекта
Hr = 256                                 // бюджет узлов в ze()
Rn = 4096                                // максимум длины строки/ключа
$r = 16384                               // максимум JSON для значения
Vr = 16384                               // максимум JSON для handshake
Xr = Function.prototype.toString.call(Object)  // эталонная строка Object
no = 1500                                // throttle для scroll-событий (мс)
```

---

## 2. Перечисления

### `bt` — типы событий раннера

```js
bt = {
  LOG:                 "log",
  RUN_START:           "run_start",
  ENVIRONMENT_STATUS:  "environment_status",
  OUTPUT:              "output",
  ERROR:               "error",
  RUN_COMPLETE:        "run_complete"
}
```

### `qn` — статусы окружения

```js
qn = {
  INITIALIZING:        0,
  INSTALLING_PACKAGES: 1,
  RUNNING_CODE:        2
}
```

---

## 3. Валидаторы

### `ze(t, e=0, n={value:256})` — валидатор значения

Проверяет, можно ли значение безопасно передать через `postMessage`.

```js
function ze(t, e = 0, n = {value: Hr}) {
  if (n.value -= 1, n.value < 0) return false;       // бюджет исчерпан
  if (t == null) return t !== undefined;             // null → false, undefined → false
  if (typeof t == "boolean") return true;
  if (typeof t == "number") return Number.isFinite(t);
  if (typeof t == "string") return t.length <= Rn;   // ≤ 4096
  if (e >= Wr || typeof t != "object") return false; // глубина > 8 или не объект

  if (Array.isArray(t)) return t.every(o => ze(o, e+1, n));

  const r = Object.getPrototypeOf(t);
  // прототип должен быть Object.prototype ИЛИ конструктор не переопределён
  return r != null && (
      !Object.prototype.hasOwnProperty.call(r, "constructor") ||
      typeof r.constructor != "function" ||
      Function.prototype.toString.call(r.constructor) !== Xr
    ) ? false
    : Object.entries(t).every(([o, s]) => o.length <= Rn && ze(s, e+1, n));
}
```

**Что отсекает:**
- `null`, `undefined`
- `NaN`, `Infinity`
- строки > 4096
- глубину > 8
- массивы с вложенными «плохими» значениями
- объекты с нестандартным конструктором (например, `class X {}` — отклонит)
- объекты с ключами > 4096
- вложенность > 256 узлов суммарно

**Чего НЕ отсекает:**
- `Symbol` — `typeof === "symbol"` не проверяется явно, но `typeof t != "object"` → return false
- функции — тоже `typeof === "function"` → return false
- `BigInt` — `typeof === "bigint"` → не попадает ни в одну ветку, `return false`

### `Ze(t, e=16384)` — проверка размера JSON

```js
function Ze(t, e = 16384) {
  try { return JSON.stringify(t).length <= e; }
  catch { return false; }
}
```
Отсекает циклические структуры (JSON.stringify бросит) и слишком большие.

### `Cn(t)` — «чистый объект»

```js
Cn = t => t != null && typeof t == "object" &&
          !Array.isArray(t) &&
          Object.getPrototypeOf(t) === Object.prototype;
```
Строже чем `ze` — требует **точный** `Object.prototype`.

### `Kr(t)` — валидатор рукопожатия моста

```js
function Kr(t) {
  if (!Cn(t)
      || t.protocol !== "app-block-messenger"
      || t.version !== 1
      || typeof t.bridgeId != "string" || t.bridgeId.length === 0 || t.bridgeId.length > 128
      || !Cn(t.bindings)
      || !Array.isArray(t.callbacks)
      || !Ze({bindings: t.bindings, callbacks: t.callbacks}, Vr)) return null;

  const e = Object.entries(t.bindings), n = t.bindings, r = t.callbacks;

  return e.length + r.length > Ur
      || e.some(([o, s]) => !Fe.test(o) || !ze(s) || !Ze(s))
      || r.some(o => typeof o != "string" || !Fe.test(o) || Object.hasOwn(n, o))
      || new Set(r).size !== r.length
    ? null
    : {
        protocol: "app-block-messenger",
        version: 1,
        bridgeId: t.bridgeId,
        bindings: Object.fromEntries(e.map(([o, s]) => [o, structuredClone(s)])),
        callbacks: [...r]
      };
}
```

**Проверки:**
1. `protocol === "app-block-messenger"`
2. `version === 1`
3. `bridgeId` — строка 1..128
4. `bindings` — «чистый» объект
5. `callbacks` — массив
6. JSON ≤ 16384
7. `bindings + callbacks ≤ 32`
8. каждое имя `bindings` — по `Fe`
9. каждое значение `bindings` — проходит `ze` и `Ze`
10. каждый `callback` — строка по `Fe`, не пересекается с `bindings`
11. `callbacks` — без дублей

**Возврат:** нормализованный объект с `structuredClone` значений, или `null`.

---

## 4. Валидаторы MCP-контента

### `pi(t)` — нормализация MCP tool result

```js
pi(result) {
  if (typeof result != "object" || result === null) return null;
  const e = result;
  if (!Array.isArray(e.content) || !e.content.every(Zr)) return null;

  const n = l => {                      // рекурсивная очистка от null
    if (Array.isArray(l)) return l.map(a => n(a));
    if (typeof l == "object" && l !== null) {
      const a = {};
      for (const [u, h] of Object.entries(l))
        h !== null && (a[u] = u === "_meta" ? h : n(h));
      return a;
    }
    return l;
  };

  const r = n(e.content);
  const o = be(e.structuredContent) ? e.structuredContent : undefined;

  // если structuredContent есть, но не объект — превратить в text-блок
  if (o === undefined && e.structuredContent !== undefined) {
    const l = JSON.stringify(e.structuredContent);
    l !== undefined && r.push({type: "text", text: l});
  }

  const s = typeof e.isError == "boolean" ? e.isError : undefined;
  const i = be(e._meta) ? e._meta : undefined;

  return {content: r, ...(o !== undefined ? {structuredContent: o} : {}),
          ...(s !== undefined ? {isError: s} : {}),
          ...(i !== undefined ? {_meta: i} : {})};
}
```

### `Zr(t)` — валидатор MCP content-блока

```js
type: "text"           → требует string text
type: "image"|"audio"  → требует string data и string mimeType
type: "resource_link"  → требует string name и string uri
type: "resource"       → требует qr(resource)
default                → false
```

### `qr(t)` — валидатор resource

```js
uri обязателен (string)
text и blob — взаимоисключающие (ровно один)
mimeType — если есть, строка
_meta — если есть, объект
```

---

## 5. Списки методов

### `Jr` — 31 скрытый метод

```js
new Set([
  "scanCard", "sendInstrument",
  "notifySecurityPolicyViolation", "notifyEnvironmentError",
  "notifyIntrinsicHeight", "notifyBackgroundColor",
  "notifyNavigation", "notifyEscapeKey", "notifyAppBlockMessage",
  "notifyAutofillCandidates", "drawBoundingBoxes", "getDownloadURL",
  "openExternalForm", "openPromptInput",
  "openConversationOverlay", "openPulseOverlay", "openPulseShare",
  "openPlaidLink",
  "openAppleHealthSettings", "ensureAppleHealthOnboarding",
  "requestAppleHealthPermissions", "shouldRequestAppleHealthPermissions",
  "showToast", "triggerHaptic", "updateWidgetState",
  "callMcp", "initializeMcpApp",
  "requestConnectPlugin", "requestReconnectConnector",
  "requestDeclinePluginSuggestions", "requestOpenPlugin"
])
```

Фильтр: `t || e || !Jr.has(o)` — если **не** first-party и **не** includePrivate,
метод с таким именем **не попадёт** в публикуемый API.

**Важно:** `notifyIntrinsicWidth` **отсутствует** в `Jr`, но используется
в `We()` → то есть width-уведомление доступно всем.

### `ao` — 8 методов с проверкой жеста

```js
new Set([
  "requestDisplayMode", "sendFollowUpMessage",
  "openExternal", "openExternalForm", "openPlaidLink",
  "requestPrompt", "requestTargetedReply", "triggerHaptic"
])
```

---

## 6. Публикация Host API — `hn()`

```js
hn = ({isFirstParty, includePrivate = false,
       hostHandlesFollowUpMessageAuthorization = false}) => {
  if (!Je) throw new Error("Host API not published");
  const r = Je;                       // текущий хост

  return Object.fromEntries(
    Object.keys(r)
      .filter(o => (o !== "prepareWidgetDocument" || e) &&   // prepareWidgetDocument только приватно
                   (t || e || !Jr.has(o)))                   // скрытые методы только first-party
      .map(o => {
        const s = o;
        return [o, (...i) =>
          !t && ao.has(s)
            && !(s === "sendFollowUpMessage" && n)            // исключение: followUp в first-party
            && !uo()                                          // нет userActivation
            ? console.warn(`Method \`${s}\` called without synchronous user event`)
            : ho(s, r, i)                                     // вызов
        ];
      })
  );
};
```

### `uo()` — проверка жеста

```js
uo = () => "userActivation" in window.navigator
  ? window.navigator.userActivation.isActive
  : true;
```

Если API нет — **пропускает** (старые браузеры).

### `ho(method, host, args)` — роутер

```js
ho = (t, e, n) => {
  // 1. requestDropdown — добавляет координаты центра элемента
  if (t === "requestDropdown") {
    const u = In()?.getBoundingClientRect();
    return e.requestDropdown?.({
      clientX: u?.left ?? 0, clientY: u?.top ?? 0, ...n[0]
    });
  }

  // 2. requestPrompt — добавляет координаты (кроме mobile-skybridge)
  if (!ln && t === "requestPrompt") {
    const u = In()?.getBoundingClientRect();
    return e.openPromptInput?.({...n[0], clientX: u?.left ?? 0, clientY: u?.top ?? 0});
  }

  // 3. callTool — конвертирует _meta → meta
  if (t === "callTool") {
    const [u, h] = n;
    return e.callTool?.(u, h).then(d =>
      d.meta == null && d._meta != null ? {...d, meta: d._meta} : d);
  }

  // 4. всё остальное — прямой вызов
  return e[t]?.call(e, ...n);
};
```

**`In()`** возвращает последний `Element`, по которому кликнули/навели:

```js
In = () => {
  const t = nt()?.contentWindow;
  return !ye || !t || !(ye instanceof t.HTMLElement) ? null : ye;
};
```
`ye` обновляется на `pointerdown`.

---

## 7. События

### Классы

```js
class Qr extends CustomEvent {          // "webplus:set_globals"
  constructor(n) { super("webplus:set_globals", {detail: {globals: n}}); this.type = vn; }
}
class to extends CustomEvent {          // "openai:set_globals"
  constructor(n) { super("openai:set_globals", {detail: {globals: n}}); this.type = Ne; }
}
```

### `Xt()` — инъекция глобалов

```js
Xt = ({root = nt(), additionalGlobals, sandboxLogger, isInit = false}) => {
  if (!root?.contentWindow) return;

  if (isInit) {                          // первый раз — присвоить целиком
    root.contentWindow.oai     = additionalGlobals;
    root.contentWindow.openai  = additionalGlobals;
    root.contentWindow.webplus = additionalGlobals;
    return;
  }

  // дальше — домешивать по одному
  root.contentWindow.oai     ??= {};
  root.contentWindow.openai  ??= {};
  root.contentWindow.webplus ??= {};

  for (const [l, a] of Object.entries(additionalGlobals))
    try {
      root.contentWindow.oai[l] = a;
      root.contentWindow.openai[l] = a;
      root.contentWindow.webplus[l] = a;
    } catch (u) { sandboxLogger.environmentError(u, false); }

  Is(root.contentWindow, additionalGlobals);      // dispatch оба события
};

Is = (t, e) => {
  t.dispatchEvent(new to(e));    // openai:set_globals
  t.dispatchEvent(new Qr(e));    // webplus:set_globals
};
```

**Три имени — один объект.** Любой доступ через `oai.X`, `openai.X`, `webplus.X`
даёт одно и то же значение.

---

## 8. Параметры URL

```js
Mt = new URLSearchParams(window.location.search)

so = Mt.get("sessionId") ?? ""
Et = Mt.get("app") ?? ""
io = Mt.get("clientPlatform") ?? ""

Zn = Mt.get("deviceType") === "mobile"
ln = Zn && Et === "skybridge"                        // mobile skybridge
Jn = !Zn && Et === "skybridge"                       // desktop skybridge
Qn = Et === "skybridge" && io === "android"
un = Et === "skybridge" && Mt.get("backgroundMode") === "transparent"
dn = Et === "chatgpt" || Et === "chatgpt-tt" || Et === "chatgpt-staging"
tr = dn && Mt.get("darkModeType") === "increased"
Mn = Mt.get("locale") ?? ""
co = Mt.get("unsafeSkipTargetOriginCheck") === "true"   // ← снимает origin-проверку
```

**CSS-классы на `<html>`:**

```js
dn && add("chatgpt")
tr && add("chatgptOled")
Qn && add("androidSkybridge")
un && add("transparentSkybridgeBackground")
ln ? add("mobileSkybridge") : Jn && add("desktopSkybridge")
```

---

## 9. `Hs()` — targetOrigin

```js
Hs = () =>
  co                                  ? "*" :
  Et === "chatgpt"                    ? "https://chatgpt.com" :
  Et === "chatgpt-tt"                 ? "https://tt.chatgpt.com" :
  Et === "feather"                    ? "https://feather.openai.com" :
  Et === "skybridge"                  ? "https://skybridge.oaistatic.com" :
                                        "*";    // ← неизвестный app → звёздочка
```

**Два пути к `"*"`:**
1. `?unsafeSkipTargetOriginCheck=true`
2. `app` не из списка

---

## 10. Мост — `$s()`

```js
$s = async (handlers, expectOther = []) => {
  // 1. снять ServiceWorker если есть
  if ("serviceWorker" in navigator) {
    const i = navigator.serviceWorker;
    let l;
    try { l = await i.getRegistration(); }
    catch (a) {
      if (!(a instanceof DOMException &&
            (a.name === "SecurityError" ||
             a.name === "NotSupportedError" &&
             a.message.endsWith("The user denied permission to use Service Worker."))) ||
          i.controller != null) throw a;
    }
    if (l) { await l.unregister(); window.location.reload(); return null; }
  }

  // 2. на каждый handler — свой MessagePort
  const ports = Object.fromEntries(
    Object.entries(handlers).map(([i, l]) => {
      const {port1: a, port2: u} = new MessageChannel;
      l && (l.constructor?.name === "AsyncGeneratorFunction"
        ? xn(a, l)      // генератор
        : St(a, l));    // обычная функция
      return [i, u];
    })
  );

  // 3. replyPort для обратного канала
  const r = new MessageChannel;

  window.parent.postMessage(
    {type: "init", ports, replyPort: r.port2},
    Hs(),
    [...Object.values(ports), r.port2]
  );

  // 4. ждём ответ
  const o = await new Promise(i => {
    r.port1.onmessage = l => i(l.data);
    r.port1.start();
  });

  // 5. собранный обратный API
  const s = new Set(expectOther);
  return Object.fromEntries(
    Object.entries(o).map(([i, l]) => [i, s.has(i) ? xn(l) : St(l)])
  );
};
```

---

## 11. RPC — `St()` и `xn()`

### Константы протокола

```js
zn = "ABORT"
Nn = "CALL"
kn = "REJECT"
Ye = "RESOLVE"
Bn = "GENERATOR_GENERATE"
```

### `St(port, handler)` — обычный RPC

```js
function St(t, e) {
  const n = s => {
    const i = s.data;
    if (!Array.isArray(i) || i[0] !== Nn) return;
    s.stopImmediatePropagation();
    const [l] = s.ports;
    if (!l) throw new Error("RPCCallMessage must contains a port.");

    e ? (async function () {
      const a = new AbortController;
      try {
        l.onmessage = ({data: u}) =>
          Array.isArray(u) && u[0] === zn && a.abort();

        l.postMessage([Ye, await e.call({signal: a.signal}, ...i.slice(1))]);
      } catch (u) {
        if (u instanceof Error && u.name === "DataCloneError") {
          try { l.postMessage([Ye, await e.call({signal: a.signal}, ...Us(i.slice(1)))]); return; }
          catch {}
        }
        l.postMessage([kn, u]);
      } finally { l.close(); }
    })()
    : (l.postMessage([kn, new Error("No function was registered on this RPC. This is probably calling a client which do not implement the function.")]),
       l.close());
  };

  t.addEventListener("message", n);
  t.start();

  // вызыватель
  const r = s => (...i) => new Promise((l, a) => {
    const {port1: u, port2: h} = new MessageChannel;
    u.onmessage = g => { const E = g.data; E[0] === Ye ? l(E[1]) : a(E[1]); u.close(); };
    s?.signal?.addEventListener("abort", () => {
      u.postMessage([zn]); u.close(); a(new Error("Aborted."));
    });
    t.postMessage([Nn, ...i], [h, ...s.transfer || []]);
  });

  const o = r({});
  o.withOptions = r;
  return o;
}
```

**Fallback при `DataCloneError`:** пробует заново с `Us()` — глубокой ручной
клонировкой, где функции превращаются в строки.

### `xn(port, handler)` — генераторный RPC

Создаёт **4 канала**: `next`, `return`, `throw`, `asyncDispose`.

```js
function xn(t, e) {
  const n = s => {
    const i = s.data;
    if (!Array.isArray(i) || i[0] !== Bn) return;
    if (s.stopImmediatePropagation(), !e)
      throw new Error("No function was registered on this RPC. ...");

    const [l, a, ...u] = i;       // l = объект с портами, a = ???
    const h = e(...u);            // вызываем фабрику генератора

    St(a.next,          h.next.bind(h));
    St(a.return,        d => h.return?.call(h, d) || {done: true});
    St(a.throw,         d => h.throw?.call(h, d) || Promise.reject(d));
    St(a.asyncDispose,  async () => {
      const d = Symbol.asyncDispose || Symbol.for("Symbol.asyncDispose");
      const g = Symbol.dispose || Symbol.for("Symbol.dispose");
      d in h ? await h[d]?.call(h) : g in h && h[g]?.call(h);
    });
  };

  t.addEventListener("message", n);
  t.start();
  // ... вызыватель с 4 портами
}
```

---

## 12. `li()` — публикация messenger

**Самая важная функция для виджета.**

```js
function li({viewWindow: t, additionalGlobals: e, notify: n}) {
  const r = Hn(e);                              // извлечь __appBlockMessenger
  if (!$n(e) || n == null) return null;         // нет мессенджера — выход

  let o = r;                                    // текущий handshake
  let s = r?.bridgeId ?? null;                  // текущий bridgeId
  let i = new Set(r?.callbacks ?? []);          // текущие callbacks
  let l = false;                                // destroyed

  const u = new Set;                            // активные effects
  const h = new Set;                            // запланированные effects

  // ── планировщик эффектов ────────────────────────────────
  const d = _ => {
    for (const f of u)
      !f.dependencies.has(_) || h.has(f) ||
        (h.add(f), queueMicrotask(() => { h.delete(f) && u.has(f) && f.run(); }));
  };

  const g = new Map;                            // подписчики на имена
  const E = new Map;                            // кэш функций-вызовов

  // ── messenger ───────────────────────────────────────────
  const y = Object.freeze({
    bindings: Object.freeze({
      get: _ => {
        if (!Fe.test(_)) return;                // имя не по шаблону → undefined
        a?.dependencies.add(_);                 // трекинг в текущем effect

        if (i.has(_)) {                         // это callback → вернуть функцию
          let p = E.get(_);
          if (p == null) {
            const A = o?.bridgeId;
            p = (...L) => {
              l || A == null || o?.bridgeId !== A || !i.has(_) || !ze(L) || !Ze(L)
                || n({
                    protocol: "app-block-messenger", version: 1,
                    bridgeId: A, type: "binding:call",
                    name: _, args: structuredClone(L)
                  });
            };
            E.set(_, p);
          }
          return p;
        }

        // иначе — значение из bindings
        if (o == null || !Object.hasOwn(o.bindings, _)) return;
        const f = o?.bindings[_];
        return f === undefined ? undefined : structuredClone(f);
      },

      subscribe: (_, f) => {
        if (!Fe.test(_) || typeof f != "function") return () => {};
        const p = g.get(_) ?? new Set;
        p.add(f); g.set(_, p);
        const A = o?.bindings[_];
        f(A === undefined ? undefined : structuredClone(A));    // сразу текущее значение
        return () => { p.delete(f); p.size === 0 && g.delete(_); };
      }
    })
  });

  Object.defineProperty(t, "messenger", {
    configurable: true, enumerable: false, writable: false, value: y
  });

  // ── __bindings — read-only Proxy ────────────────────────
  Object.defineProperty(t, "__bindings", {
    configurable: true,
    value: new Proxy(Object.create(null), {
      get: (_, f) => typeof f == "string" ? y.bindings.get(f) : undefined,
      set: () => { throw new TypeError("AppBlock bindings are read-only. Call an exposed binding function instead."); }
    })
  });

  // ── __effect — реактивный рантайм ───────────────────────
  Object.defineProperty(t, "__effect", {
    configurable: true,
    value: _ => {
      const f = {
        dependencies: new Set,
        run: () => {
          const p = a;                          // сохранить предыдущий
          a = f; f.dependencies.clear();
          try { _(); }
          catch (A) { t.dispatchEvent(new ErrorEvent("error", {error: A})); }
          finally { a = p; }
        }
      };
      return l || (u.add(f), f.run()), () => { u.delete(f); h.delete(f); };
    }
  });

  // ── обновление при set_globals ──────────────────────────
  const O = _ => {
    const f = _.detail?.globals;
    if (!$n(f)) return;

    const p = Hn(f);
    p != null && p.bridgeId !== s && E.clear();
    s = p?.bridgeId ?? s;

    const A = o?.bindings ?? {};
    const L = i;
    o = p;
    i = new Set(p?.callbacks ?? []);

    // перезапустить все затронутые effects
    for (const I of new Set([...Object.keys(A), ...Object.keys(p?.bindings ?? {}), ...L, ...i]))
      d(I);

    // разослать подписчикам
    for (const [I, z] of g) {
      const H = p?.bindings[I];
      for (const G of z) G(H === undefined ? undefined : structuredClone(H));
    }
  };

  t.addEventListener(Ne, O);                    // "openai:set_globals"

  // ── cleanup ─────────────────────────────────────────────
  const m = () => {
    l = true;
    t.removeEventListener(Ne, O);
    t.removeEventListener("pagehide", m);
    g.clear(); u.clear(); h.clear(); E.clear();
    delete t.messenger; delete t.__bindings; delete t.__effect;
  };

  t.addEventListener("pagehide", m, {once: true});
  return m;
}
```

---

## 13. `Dr()` — `runHTMLAsync`

Главный рендер документа виджета.

```js
async function* Dr({
  sandboxLogger, features, html, theme, safeArea,
  transformPosition, additionalGlobals,
  expectReadySignal, expectedReadySignal,
  appInitialization, ownerColorScheme,
  isFirstParty = false,
  forwardConsoleLogs = true,
  measureWidth = false,
  measureHeight = false,
  disableMutationObserver = false,
  onIframeReady,
  prepareDocument,
  enableExternalNavigationGating = false,
  disableExternalNavigation = false,
  allowPopups = true
})
```

### Шаги

```
1. Создать iframe через jr(features, {allowPopups, ownerColorScheme})
2. Слушать "preview-cancel"
3. На load:
   a. document.open()
   b. Если есть __appBlockMessenger → li() (публикация messenger)
   c. onIframeReady() — колбэк
   d. Xt() — инъекция глобалов + safeArea
   e. prepareDocument() — с таймаутом 10 сек
   f. oo()  — перехват навигации и форм
   g. Mr()  — (политика)
   h. ei()  — диагностика
   i. Qs()  — (?)
   j. ti()  — консоль и ошибки
   k. Js()  — (?)
   l. document.write(html)
   m. Настроить ResizeObserver + MutationObserver
   n. document.close()
   o. Если data-fatal → отмена
   p. Если data-ready → ready
```

### Ожидание готовности

```js
// два режима
expectReadySignal === false  → ждать DOMContentLoaded
expectReadySignal === true   → ждать data-ready на <html>
                               (и совпадения с expectedReadySignal если задан)
```

### Обнаружение «мёртвого» документа

```js
A = new MutationObserver($ => {
  if (Pt.hasAttribute("data-fatal")) { Ct(); return; }

  for (const Z of $) {
    if (Z.target === Pt) {
      if (Z.attributeName === "data-ready" && Tn()) { Ge(); break; }
    } else if (u == null
               && Z.addedNodes.length === 0
               && Z.removedNodes.length > 0
               && Z.target.parentElement === P.body
               && Z.target.childNodes.length === 0
               && Array.from(P.body.childNodes).every(Dt => Dt === Z.target || ui(Dt))) {
      Ct(); break;      // body опустел → ошибка стартапа
    }
    Ue(); We();
  }
});
```

### Таймауты

```js
MCP init:           30 000 мс → "MCP app initialization timed out"
prepareDocument:    10 000 мс → "Document preparation timed out"
```

### Метрики

```js
he(name) — инкрементальная метрика (один раз)
"html.preview_started"
"html.preview_ready"
"html.preview_runtime_error"
"html.preview_startup_error"
"html.preview_cdnjs_blocked"
"html.preview_script_blocked"
```

---

## 14. Перехват навигации — `oo()`

```js
oo = ({doc, isFirstParty, enableExternalNavigationGating = false,
       disableExternalNavigation = false}) => {
  const o = doc.contentWindow;
  if (!o) return;

  const s = hn({isFirstParty, includePrivate: isFirstParty || enableExternalNavigationGating});
  ro(s);                        // Escape → notifyEscapeKey

  // ── 1. Navigation API ──────────────────────────────────
  const i = disableExternalNavigation ? Reflect.get(o, "navigation") : undefined;
  eo(i) && i.addEventListener("navigate", y => {
    if (!y.cancelable) return;
    new URL(y.destination.url, o.location.href).origin !== o.location.origin && y.preventDefault();
  });

  // ── 2. Формы ───────────────────────────────────────────
  const a = y => {
    const w = y.action || o.location.href;
    const O = new URL(w, o.location.href);
    if (!n) return false;

    const m = (y.getAttribute("target") ?? "").trim();
    const _ = (y.getAttribute("method") ?? "get").trim().toLowerCase();
    const f = (y.getAttribute("enctype") ?? "").trim() || undefined;

    if (s.openExternalForm) {
      const p = new o.FormData(y), A = [];
      let L = false;
      for (const [T, I] of p.entries())
        typeof I == "string" ? A.push({name: T, value: I}) : L = true;
      s.openExternalForm({action: O.href, method: _, target: m, enctype: f, entries: A, hasFiles: L});
      return true;
    }
    if (_ === "get") {
      const p = new URL(O.href), A = new o.FormData(y);
      for (const [L, T] of A.entries()) typeof T == "string" && p.searchParams.append(L, T);
      return !!fe(s, o.location, p, {forceHostOpen: true});
    }
    return !!fe(s, o.location, O.href, {forceHostOpen: true});
  };

  // ── 3. Клики по ссылкам ─────────────────────────────────
  const u = y => {
    if (!y.isTrusted ||
        (s.sendInstrument?.({type: "interaction", label: "click"}),
         y.defaultPrevented || !(y instanceof o.MouseEvent))) return;

    const w = y.target;
    if (!(w instanceof o.Element)) return;

    const O = w.closest("a");
    if (!O) return;

    const m = O.getAttribute("href") ?? O.getAttributeNS("http://www.w3.org/1999/xlink", "href");
    const _ = O.namespaceURI === "http://www.w3.org/2000/svg" ? (O.href?.animVal ?? m) : m;
    if (!_) return;

    let f;
    if (O instanceof o.HTMLAnchorElement) f = O.href;
    else try { f = new URL(_, O.baseURI); } catch { return; }

    const {location: p} = o;
    const A = isFirstParty && (y.altKey || y.ctrlKey || y.metaKey || y.shiftKey);
    fe(s, p, f, {forceHostOpen: enableExternalNavigationGating, openMode: A ? "tab" : undefined})
      && y.preventDefault();
  };

  // ── 4. Копирование ──────────────────────────────────────
  const h = () => {
    const y = o.getSelection();
    !y || y.toString().trim().length === 0
      || s.sendInstrument?.({type: "interaction", label: "copy"});
  };

  // ── 5. Скролл (throttle 1500 мс) ────────────────────────
  const d = () => {
    l || (l = o.setTimeout(() => { l = undefined; }, no),
          s.sendInstrument?.({type: "interaction", label: "scroll"}));
  };

  // ── 6. Escape ───────────────────────────────────────────
  const g = y => { ye = y.target; y.key === "Escape" && s.notifyEscapeKey?.(); };
  const E = y => { ye = y.target; };

  o.addEventListener("pointerdown", E, {passive: true});
  o.addEventListener("keydown", g, {passive: true});
  o.addEventListener("click", u, {capture: enableExternalNavigationGating});

  // ── 7. Подмена HTMLAnchorElement.click ──────────────────
  if (enableExternalNavigationGating) {
    const y = o.HTMLAnchorElement.prototype.click;
    o.HTMLAnchorElement.prototype.click = function() {
      const m = [], _ = f => m.push(f);
      this.addEventListener("click", _, {capture: true});
      try { y.call(this); }
      finally { this.removeEventListener("click", _, {capture: true}); }

      m.length === 0 || m[0].defaultPrevented
        || this.target.toLowerCase() !== "_blank"
        || this.hasAttribute("download")
        || (this.protocol !== "http:" && this.protocol !== "https:")
        || fe(s, o.location, this.href, {forceHostOpen: true});
    };

    // ── 8. Подмена HTMLFormElement.submit ─────────────────
    const w = m => {
      if (m.defaultPrevented) return;
      const _ = m.target;
      _ instanceof o.HTMLFormElement && a(_) && m.preventDefault();
    };
    o.addEventListener("submit", w);

    const O = o.HTMLFormElement.prototype.submit;
    o.HTMLFormElement.prototype.submit = function(...m) {
      if (!a(this)) return O.apply(this, m);
    };
  }

  // ── 9. Подмена window.open ──────────────────────────────
  o.addEventListener("copy", h, {passive: true});
  o.addEventListener("scroll", d, {passive: true});
  o.document.addEventListener("scroll", d, {capture: true, passive: true});

  o.open = y => {
    if (!y) return null;
    const {location: w} = o;
    fe(s, w, y, {forceHostOpen: enableExternalNavigationGating,
                 openMode: isFirstParty ? "tab" : undefined});
    return null;                       // ← всегда null
  };
};
```

### `fe()` — решение «открыть наружу?»

```js
fe = (host, base, url, opts = {}) => {
  try {
    const u = new URL(url, base.href);

    // тот же origin + тот же path + тот же query → не внешняя
    if (u.origin === base.origin && u.pathname === base.pathname && u.search === base.search)
      return false;

    // другой host или другой protocol → наружу
    if ((u.hostname !== base.hostname ||
         u.protocol !== base.protocol ||
         opts.forceHostOpen) && host.openExternal) {
      const p = {href: u.href};
      opts.openMode != null && (p.openMode = opts.openMode);
      host.openExternal(p);
      return true;
    }
  } catch { return; }
};
```

**Итог:** `window.open` в песочнице всегда возвращает `null`, потому что
`fe()` перехватывает и вызывает `openExternal` (если доступно).

---

## 15. Автозаполнение — `yi()`

```js
Rs = new Set(["name", "given-name", "family-name", "email", "tel"]);
pn = new Map;      // id → input

// какие поля подходят
Cs = input =>
  input.disabled || input.readOnly || input.hidden || input.value
    || !["text", "email", "tel"].includes(input.type)
    ? null
    : input.type === "email" || input.type === "tel"
      ? input.type
      : input.autocomplete.toLowerCase().split(/\s+/).find(n => Rs.has(n)) ?? null;

// сканирование
vs = (doc, win) => {
  const m = new Map, r = [];
  doc.querySelectorAll("input").forEach((o, s) => {
    const i = Cs(o); if (!i) return;
    const l = win.getComputedStyle(o), a = o.getBoundingClientRect();
    if (l.display === "none" || l.visibility === "hidden" || a.width === 0 || a.height === 0) return;
    const u = {id: `${i}-${s}`, fieldType: i};
    m.set(u.id, o);
    r.push(u);
  });
  pn = m;
  return r;
};

// наблюдение
yi = ({document, viewWindow}) => {
  let n = null;
  const r = () => X().notifyAutofillCandidates?.(vs(document, viewWindow));
  const o = new MutationObserver(() => {
    n == null && (n = viewWindow.requestAnimationFrame(() => { n = null; r(); }));
  });
  o.observe(document, {
    attributes: true,
    attributeFilter: ["autocomplete", "disabled", "hidden", "readonly", "style", "type"],
    childList: true, subtree: true
  });
  r();
  return () => {
    o.disconnect();
    n != null && viewWindow.cancelAnimationFrame(n);
    pn.clear();
    X().notifyAutofillCandidates?.([]);
  };
};

// применение значения — БЕЗ события input от пользователя
Ls = ({id, value}) => {
  const n = pn.get(id);
  n && (
    Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, "value")?.set?.call(n, value),
    n.dispatchEvent(new InputEvent("input", {bubbles: true, data: value})),
    n.dispatchEvent(new Event("change", {bubbles: true}))
  );
};
```

**`Ls` вызывается через `applyAutofillCandidate` в фасаде.** То есть
**хост может заполнить форму виджета без участия пользователя** и сымитировать
`input`/`change`.

---

## 16. Выделение компонентов — `As`

```js
class As {
  // наведение
  handleMouseOver(e) {
    const n = Fn(e);               // найти компонент по composedPath
    this.hoveredComponent = n;
    this.measure();
  }

  // клик
  handleClick(e) {
    e.stopPropagation(); e.preventDefault();
    const n = Fn(e);
    this.selectedComponent = n;
    this.measure();
  }

  // измерение и отправка рамок хосту
  measure = () => {
    this.frame = requestAnimationFrame(() => {
      X().drawBoundingBoxes?.(this.getBoundingBoxes());
    });
  };

  getBoundingBoxes = () => ({hovered, selected});
}
```

`Fn(e)` ищет в `composedPath()` элемент с `data-source-name` и `data-source-pos`:

```js
Os = t => {
  const {sourceName, sourcePos} = t.dataset;
  if (!sourceName || !sourcePos) return null;
  const [r, o] = sourcePos.split("-");
  const [s, i] = r.split(":");
  const [l, a] = o.split(":");
  return {
    id: `${sourceName}:${s}:${i}`,
    name: sourceName,
    element: t,
    location: {
      start: {line: Number(s), col: Number(i)},
      end:   {line: Number(l), col: Number(a)}
    }
  };
};
```

**Вот как работает «клик по элементу → модель видит, какой код за него отвечает».**
Транспайлер вставляет `data-source-pos` в JSX.

### `Dn(element)` — вычисление рамки

```js
Dn = t => {
  const {borderWidth, borderRadius, backgroundColor,
         paddingLeft, paddingRight, paddingTop, paddingBottom} = getComputedStyle(t);
  const a = t.getBoundingClientRect();
  const u = color.getAlpha(color.parse(backgroundColor)) === 0;   // прозрачный фон
  const h = kt(borderWidth) > 0;
  const d = kt(borderRadius);
  const g = d ? d + 4 : 8;

  return u && !h
    ? {x: a.x + kt(paddingLeft) - 4,  y: a.y + kt(paddingTop) - 4,
       width: a.width - kt(paddingLeft) - kt(paddingRight) + 8,
       height: a.height - kt(paddingTop) - kt(paddingBottom) + 8,
       borderRadius: g}
    : {x: a.x - 4, y: a.y - 4,
       width: a.width + 8, height: a.height + 8,
       borderRadius: g};
};
```

Если у элемента **прозрачный фон и нет рамки** — рамка рисуется по padding-box.
Иначе — по border-box с отступом 4px.

---

## 17. Widget state — `Ms()`

```js
Ms = ({widgetId, globals, sandboxLogger}) => {
  const {toolInput, toolOutput, toolResponseMetadata, widgetState, subjectId = null} = globals;
  let a = widgetState;

  const u = async (h, d) => {
    const g = typeof h == "function" ? h(a) : h;
    a = g;
    Xt({additionalGlobals: {widgetState: g}, sandboxLogger});
    const E = await X().updateWidgetState?.(widgetId, g, d);

    if (E?.ok === false || (d?.waitForPersistence === true && E == null)) {
      const O = new Error("The host could not persist the widget state update");
      O.code = -32000;
      O.data = {
        code: "widget_state_persistence_failed",
        retryable: E?.ok === false ? E.retryable : false
      };
      throw O;
    }
  };

  Sr = u;             // глобальная ссылка
  return {toolInput, toolOutput, toolResponseMetadata, widgetState, subjectId, setWidgetState: u};
};
```

**`setWidgetState(v)`** — принимает значение или функцию-апдейтер.
После записи обновляет глобалы и шлёт в хост через `updateWidgetState`.

Ошибка `widget_state_persistence_failed` (код `-32000`) — если хост не смог
сохранить.

---

## 18. Python-worker

```js
const ks = "/assets/python-worker-Y5Z0ImDG.js";
Ar = new URL(ks, import.meta.url);
Ar.searchParams.set("sessionId", so);

xs = () => {
  const t = new Blob([`await import(${JSON.stringify(Ar.href)});`], {type: "text/javascript"});
  const e = URL.createObjectURL(t);
  try { return new Worker(e, {type: "module"}); }
  finally { URL.revokeObjectURL(e); }
};

js = () => {
  const t = xs();
  return new Promise(e => {
    t.onmessage = n => { n.data.type === "ready" && e(t); };
  });
};

Gs = async (code, logger, onMessage) => {
  try { Lt ??= await js(); }
  catch (r) { logger.error("failed to create python worker", r); }

  return new Promise((resolve, reject) => {
    const s = Lt;
    if (!s) return reject(new Error("Python worker is unavailable"));
    let i = false;
    const done = () => {...};

    const h = g => {
      if (g.data.type === "security_policy_violation") {
        X().notifySecurityPolicyViolation(g.data.payload);
        return;
      }
      onMessage(g.data);
      g.data.type === bt.RUN_COMPLETE && (done(), resolve(g.data));
    };

    s.onmessage = h;
    s.addEventListener("terminate", d);
    s.postMessage({code});
  });
};

Rr = () => {
  try { Lt?.dispatchEvent(new Event("terminate")); Lt?.terminate(); }
  finally { Lt = null; }
};
```

**Один воркер на всё.** Повторно используется через `Lt ??=`.

---

## 19. Замеры размеров

### Высота — `Xs()`

```js
pe = t => {                             // один элемент
  if (!t) return null;
  const e = t.scrollHeight ?? 0;
  const {height: n} = t.getBoundingClientRect();
  const r = Math.max(e, n);
  return Number.isFinite(r) && r > 0 ? r : null;
};

Vs = t => {                             // дети body
  const e = Array.from(t.children).filter(i => {
    const l = i.tagName;
    return l !== "SCRIPT" && l !== "STYLE";
  });
  if (!e.length) return null;
  if (e.length === 1) return pe(e[0]);

  const n = t.getBoundingClientRect?.();
  let r = 0;
  for (const i of e) {
    const l = i.getBoundingClientRect?.();
    if (!l) continue;
    const a = n ? n.top : 0;
    const u = l.bottom - a;
    r = Math.max(r, u);
  }
  return r > 0 ? r : null;
};

Xs = t => {                             // итог
  if (!t) return null;
  const e = pe(t.getElementById("root"));
  if (e) return e;
  const n = t.body;
  if (n) {
    const r = Vs(n); if (r) return r;
    const o = pe(n); if (o) return o;
  }
  return pe(t.documentElement) ?? pe(t.scrollingElement);
};
```

### Ширина — `Ks()`

```js
qe = t => {
  if (!t) return null;
  const e = t.scrollWidth ?? 0;
  const {width: n} = t.getBoundingClientRect();
  const r = Math.max(e, n);
  return Number.isFinite(r) && r > 0 ? r : null;
};

Ks = t => {
  if (!t) return null;
  const e = qe(t.getElementById("root"));
  return e || (qe(t.documentElement) ?? qe(t.scrollingElement));
};
```

---

## 20. Глубокое клонирование — `gn()` / `Us()`

`Us(t)` — попробовать `structuredClone`, при провале — `gn(t, new WeakMap)`.

`gn()` вручную обходит:
- примитивы → как есть
- `symbol`/`function`/`undefined` → `String(t)`
- циклические → `WeakMap`
- `Date`, `RegExp`, `ArrayBuffer`, `DataView`, `TypedArray`
- `Map`, `Set`
- `Blob`, `File`
- `URL`, `URLSearchParams`
- `Error` (со `stack` и `cause`)
- `SharedArrayBuffer`/`WeakMap`/`WeakSet` → `String(t)`
- массивы, объекты

**Используется** в `St()` как fallback при `DataCloneError`.

---

## 21. Логирование — `rt` (SandboxLogger)

```js
class rt {
  pendingTimers = new Map;
  previewDiagnostics;

  log(msg, ...args) { /* no-op */ }

  error(msg, err) {
    console.error("[sandbox]", msg);
    X().sendInstrument({type: "error", message: msg, error: err});
  }

  environmentError(err, fatal) {
    console.error("[sandbox]", err);
    if (err instanceof Error)
      return X().notifyEnvironmentError({name: err.name, message: err.message, stack: err.stack, fatal});

    const {stack} = new Error;
    try { X().notifyEnvironmentError({name: "Unknown environment error", fatal, message: JSON.stringify(err), stack}); }
    catch { X().notifyEnvironmentError({name: "Unknown environment error", fatal, message: "Failed to stringify error", stack}); }
  }

  count(label, n, tags) {
    X().sendInstrument({type: "count", previewDiagnostics: this.previewDiagnostics, label, tags, count: n});
  }

  hist(label, n, tags) {
    X().sendInstrument({type: "hist", label, tags, value: n});
  }

  time(name)    { this.pendingTimers.set(name, performance.now()); }
  timeEnd(name) {
    if (!this.pendingTimers.has(name)) return;
    const r = performance.now() - this.pendingTimers.get(name);
    this.pendingTimers.delete(name);
    this.hist(name, r);
  }
}
```

---

## 22. Публичный фасад — `wt`

**Вот главное API песочницы.** Инициализируется асинхронно через `$s()`:

```js
$s({
  // ──────────── ВЫЗЫВАЕТ ХОСТ ────────────
  async* runUserCode(t)        // запустить код
  async buildUserCode(t)       // собрать HTML из кода
  async prepareEnvironment(t)  // прогреть окружение
  async* runWidgetCode(t)      // отрисовать виджет
  async* runComponent(t)       // отрисовать компонент
  async validate(code, id)     // проверить React-код
  async renderMermaidSvg(t)    // отрисовать диаграмму
  stop()                       // остановить всё
  setEditMode(bool)            // режим выделения элементов
  applyAutofillCandidate({id, value})   // заполнить поле
  async screenshot()           // PNG-скриншот
  async setAdditionalGlobals({additionalGlobals})
  async setWidgetData({widgetId, widgetState, toolInput, toolOutput,
                       toolResponseMetadata, subjectId})
  async setWidgetView({displayMode, viewParams, isTombstone})
  async setTheme({theme})
  async setSafeArea({safeArea})
  notifyMcpAppsToolInput(t)           // MCP-уведомления
  notifyMcpAppsToolResult(t)
  notifyMcpAppsToolCancelled(t)
  notifyMcpAppsHostContext({hostContext})
  notifyMcpAppsMcpNotification(t)
  async requestMcpAppsResourceTeardown({timeoutMs})
  async navigate({delta | popToRoot})
  getCurrentPath()
}).then(t => { t && lo(t); });      // lo() → Je = t (публикация Host API)
```

### `screenshot()`

```js
async screenshot() {
  const t = nt();                                   // iframe
  if (!t) return null;
  const e = t.contentDocument?.documentElement;
  if (!e) return null;

  const n = e.scrollLeft || 0;
  const r = e.scrollTop || 0;
  const {domToDataUrl} = await import("./index-hL9okClV.js");
  return {
    imageBase64: await domToDataUrl(e, {backgroundColor: "transparent"}),
    scrollX: n,
    scrollY: r
  };
}
```

### `setWidgetData()`

```js
async setWidgetData({widgetId, widgetState, toolInput, toolOutput,
                     toolResponseMetadata, subjectId = null}) {
  Xt({
    additionalGlobals: {
      widget: {
        state: widgetState,
        props: toolResponseMetadata,
        setState: l => X().updateWidgetState?.(widgetId, l)
      },
      ...Ms({widgetId, globals: {toolInput, toolOutput, toolResponseMetadata,
                                 widgetState, subjectId}, sandboxLogger})
    },
    sandboxLogger
  });
}
```

Даёт виджету:
```js
window.openai.widget.state          — текущее состояние
window.openai.widget.props          — метаданные ответа инструмента
window.openai.widget.setState(v)    — установить состояние
window.openai.toolInput             — вход инструмента
window.openai.toolOutput            — выход инструмента
window.openai.setWidgetState(v)     — установить состояние (альтернатива)
```

### `navigate()`

```js
async navigate(t) {
  const {navigation} = await import(...);
  if ("delta" in t)   { navigation.go(t.delta);   return; }
  if ("popToRoot" in t) { navigation.popToRoot(); return; }
}
getCurrentPath() {
  const t = nt()?.contentWindow?.location;
  return t ? `${t.pathname}${t.search}${t.hash}` : null;
}
```

---

## 23. Как виджет вызывает возможности

### Что доступно в `window` песочницы

```js
window.openai = {
  // ── из setWidgetData ───────────────────────────────
  toolInput, toolOutput, toolResponseMetadata, widgetState, subjectId,
  setWidgetState(fn_or_value),
  widget: {state, props, setState},

  // ── из setWidgetView ───────────────────────────────
  displayMode, view: {params, mode, isTombstone},

  // ── из setTheme / setSafeArea ──────────────────────
  theme, safeArea,

  // ── Host API (8 клик-методов видны всем) ───────────
  requestDisplayMode(mode), sendFollowUpMessage(txt),
  openExternal({href}), openPromptInput(opts),
  requestTargetedReply(opts), triggerHaptic(opts),

  // ── все 31 скрытых метода (только first-party) ────
  scanCard, sendInstrument, callMcp, ...

  // ── messenger ──────────────────────────────────────
  // window.messenger.bindings.get / subscribe
  // window.__bindings.<name>
  // window.__effect(fn)
};
window.oai     === window.openai;     // то же
window.webplus === window.openai;     // то же
```

### Примеры вызова

```js
// открыть ссылку (нужен клик)
button.addEventListener("click", () => {
  openai.openExternal({href: "https://example.com"});
});

// отправить сообщение в чат
button.addEventListener("click", () => {
  openai.sendFollowUpMessage("Расскажи подробнее");
});

// запросить fullscreen
openai.requestDisplayMode("fullscreen");

// вибрация
openai.triggerHaptic({type: "light"});

// состояние
openai.setWidgetState(prev => ({...prev, count: prev.count + 1}));

// скрытые (только first-party, иначе undefined)
openai.callMcp({method: "tools/call", params: {...}});
openai.scanCard();
```

---

## 24. Что можно, а что нет — по факту

| Возможность | Доступно | Условие |
|---|---|---|
| `openExternal` | ✅ | клик пользователя |
| `openExternalForm` | ✅ | клик |
| `sendFollowUpMessage` | ✅ | клик |
| `requestDisplayMode` | ✅ | клик |
| `triggerHaptic` | ✅ | клик |
| `requestPrompt` | ✅ | клик |
| `requestTargetedReply` | ✅ | клик |
| `openPlaidLink` | ✅ | клик |
| `setWidgetState` | ✅ | без условий |
| `fetch` / `WebSocket` | ✅ | CSP `connect-src 'self'` |
| `window.open` | ❌ | всегда `null` |
| `scanCard` | ❌ | только first-party |
| `sendInstrument` | ❌ | только first-party |
| `callMcp` | ❌ | только first-party |
| `getDownloadURL` | ❌ | только first-party |
| `openConversationOverlay` | ❌ | только first-party |
| `notifyAutofillCandidates` | ❌ | только first-party |
| `drawBoundingBoxes` | ❌ | только first-party |
| `notifyIntrinsicWidth` | ⚠️ | **не в Jr** → доступно всем |
| `notifyEscapeKey` | ❌ | только first-party |
| `prepareWidgetDocument` | ❌ | только с `includePrivate` |

---

## 25. Итог по файлу

`main-BR1bOQyH.js` — это **весь механизм песочницы**:

```
1. Создание iframe с sandbox-атрибутами
2. Публикация Host API с двумя уровнями доступа
3. Инъекция глобалов в трёх именах (oai/openai/webplus)
4. Протокол моста через MessageChannel
5. Перехват всей навигации, форм, ссылок, window.open
6. Автозаполнение полей без участия пользователя
7. Режим выделения компонентов (клик → строка кода)
8. Управление widget state с проверкой персистентности
9. Запуск Python-воркера через blob URL
10. Замеры размеров и скриншоты
```

**Слабые места, которые видны в этом файле:**

1. `Hs()` возвращает `"*"` при `unsafeSkipTargetOriginCheck=true` **или**
   при неизвестном `app` — то есть дефолт для незнакомого окружения открыт
2. `notifyIntrinsicWidth` отсутствует в `Jr` — утечка имени метода
3. `Ls()` (applyAutofillCandidate) заполняет поле и **диспатчит события**
   от имени пользователя — любой, кто имеет доступ к фасаду, может имитировать
   ввод
4. `window.open` подменяется на `null` — но `fe()` всё равно зовёт `openExternal`,
   если он есть

**Сильные стороны:**

1. `ze()` — грамотный валидатор значений (глубина, бюджет, прототипы)
2. `Kr()` — строгая валидация рукопожатия
3. Два уровня API (`Jr` скрыт, `ao` требует клика)
4. `oo()` перехватывает **все** пути навигации: Navigation API, формы,
   клики по `<a>`, `HTMLAnchorElement.click`, `HTMLFormElement.submit`,
   `window.open`
5. `li()` корректно чистит за собой на `pagehide`
