# `main-BR1bOQyH.js` — полный разбор (ядро Skybridge)

**Размер:** 89 639 байт
**URL:** `https://web-sandbox.oaiusercontent.com/assets/main-BR1bOQyH.js`
**Роль:** ядро. Фасад Host API, iframe-раннер, глобалы, события,
защита навигации, все утилиты.

---

# ЧАСТЬ 1. Экспорты — публичная карта

```js
export {
  qn as E,      // EnvironmentStatus
  Mn as L,      // locale
  bt as M,      // RunEventType
  Ne as S,      // "openai:set_globals"
  _t as _,      // preload + dynamic import
  to as a,      // SetGlobalsEvent
  xr as b,      // resetPreview
  ni as c,      // callback → async generator
  X  as d,      // hn({isFirstParty:true, includePrivate:true})
  Ms as e,      // widgetStateManager
  Ps as f,      // viewParams
  hn as g,      // publishHostApi
  mi as h,      // getGlobalSurface
  fi as i,      // trusted CDN list
  gi as j,      // (не разобран — маркеры)
  D  as k,      // Object (UMD-хелпер)
  nn as l,      // LRU-кэш (yn)
  bi as m,      // python module {runPythonInWorker, terminatePythonWorker}
  pi as n,      // normalizeToolResult
  yi as o,      // autofillObserver
  _i as p,      // {runHTMLAsync}
  Dr as r       // runHTMLAsync
};
```

**22 экспорта.** Всё остальное — приватное.

---

# ЧАСТЬ 2. Константы и типы

## 2.1 `bt` — `RunEventType`

```js
var bt = (t => (
  t.LOG = "log",
  t.RUN_START = "run_start",
  t.ENVIRONMENT_STATUS = "environment_status",
  t.OUTPUT = "output",
  t.ERROR = "error",
  t.RUN_COMPLETE = "run_complete",
  t
))(bt || {});
```

**6 типов событий:**
- `log`
- `run_start`
- `environment_status`
- `output`
- `error`
- `run_complete`

## 2.2 `qn` — `EnvironmentStatus`

```js
var qn = (t => (
  t[t.INITIALIZING = 0] = "INITIALIZING",
  t[t.INSTALLING_PACKAGES = 1] = "INSTALLING_PACKAGES",
  t[t.RUNNING_CODE = 2] = "RUNNING_CODE",
  t
))(qn || {});
```

**Числовое перечисление** (reverse-mapping): `INITIALIZING = 0`,
`INSTALLING_PACKAGES = 1`, `RUNNING_CODE = 2`.

**Почему числа, а не строки:** передаются через `postMessage` — числа
компактнее.

## 2.3 `fi` — доверенные CDN

```js
fi = [
  "https://cdn.jsdelivr.net",
  "https://cdn.tailwindcss.com",
  "https://esm.sh",
  "https://unpkg.com",
  "https://pypi.org",
  "https://files.pythonhosted.org"
];
```

**Используется в `apply-csp`** как база `script-src`.

## 2.4 `Jr` — скрытые методы (31)

```js
Jr = new Set([
  "scanCard", "sendInstrument", "notifySecurityPolicyViolation",
  "notifyEnvironmentError", "notifyIntrinsicHeight", "notifyBackgroundColor",
  "notifyNavigation", "notifyEscapeKey", "notifyAppBlockMessage",
  "notifyAutofillCandidates", "drawBoundingBoxes", "getDownloadURL",
  "openExternalForm", "openPromptInput", "openConversationOverlay",
  "openPulseOverlay", "openPulseShare", "openPlaidLink",
  "openAppleHealthSettings", "ensureAppleHealthOnboarding",
  "requestAppleHealthPermissions", "shouldRequestAppleHealthPermissions",
  "showToast", "triggerHaptic", "updateWidgetState", "callMcp",
  "initializeMcpApp", "requestConnectPlugin", "requestReconnectConnector",
  "requestDeclinePluginSuggestions", "requestOpenPlugin"
]);
```

## 2.5 `ao` — методы с user gesture (8)

```js
ao = new Set([
  "requestDisplayMode", "sendFollowUpMessage", "openExternal",
  "openExternalForm", "openPlaidLink", "requestPrompt",
  "requestTargetedReply", "triggerHaptic"
]);
```

---

# ЧАСТЬ 3. Валидация значений — `ze()`

```js
const Fe = /^[A-Za-z][A-Za-z0-9_]{0,63}$/,
      Ur = 32,        // максимум bindings+callbacks
      Wr = 8,         // максимальная глубина вложенности
      Hr = 256,       // максимальное число значений
      Rn = 4096,      // максимальная длина строки/ключа
      $r = 16384,     // максимальный размер JSON (bindings+callbacks)
      Vr = 16384,     // максимальный размер callback
      Xr = Function.prototype.toString.call(Object);

const Cn = t => t != null && typeof t == "object"
            && !Array.isArray(t)
            && Object.getPrototypeOf(t) === Object.prototype;

function ze(t, e = 0, n = {value: Hr}) {
  if (n.value -= 1, n.value < 0) return false;      // лимит значений
  if (t == null) return t !== void 0;               // null ok, undefined нет
  if (typeof t == "boolean") return true;
  if (typeof t == "number") return Number.isFinite(t);
  if (typeof t == "string") return t.length <= Rn;
  if (e >= Wr || typeof t != "object") return false; // глубина

  if (Array.isArray(t)) return t.every(o => ze(o, e + 1, n));

  const r = Object.getPrototypeOf(t);
  return r != null && (!Object.prototype.hasOwnProperty.call(r, "constructor")
    || typeof r.constructor != "function"
    || Function.prototype.toString.call(r.constructor) !== Xr) ? false
    : Object.entries(t).every(([o, s]) => o.length <= Rn && ze(s, e + 1, n))
}
```

## 3.1 Что валидируется

| Тип | Правило |
|---|---|
| `null` | ✅ ok |
| `undefined` | ❌ не ok |
| `boolean` | ✅ |
| `number` | только `isFinite` (`NaN`/`Infinity` — нет) |
| `string` | длина ≤ 4096 |
| массив | каждый элемент рекурсивно |
| объект | только plain (`Object.prototype`), ключи ≤ 4096, значения рекурсивно |
| глубина | ≤ 8 |
| всего значений | ≤ 256 |

## 3.2 Защита от прототипных цепочек

```js
r != null && (!Object.prototype.hasOwnProperty.call(r, "constructor")
  || typeof r.constructor != "function"
  || Function.prototype.toString.call(r.constructor) !== Xr) ? false : ...
```

**`Xr = Function.prototype.toString.call(Object)`** — строка вида
`"function Object() { [native code] }"`.

**Логика:** для объекта берётся его прототип `r`. Если у `r`:
- нет своего `constructor` **или**
- `constructor` не функция **или**
- `constructor` не нативный `Object`

→ **вернуть `false`**.

**Что это блокирует:**
- `class Foo {}` → `new Foo()` — прототип `Foo.prototype`, `constructor === Foo`, `Foo.toString() !== Xr` → отсеян
- `Object.create({})` → прототип не `Object.prototype`, но `constructor` унаследован от `Object` → пройдёт (и это ок)
- объект с подменённым `constructor` → отсеян

**Почему так, а не `Object.getPrototypeOf(t) === Object.prototype`:** работает
для **plain-объектов с цепочкой** — например `{a: Object.create(Object.prototype)}`.
Хотя по сути это эквивалентно.

**`Cn(t)` — строже:** `Object.getPrototypeOf(t) === Object.prototype` —
только буквально plain.

## 3.3 `Ze()` — проверка размера JSON

```js
function Ze(t, e = $r) {
  try { return JSON.stringify(t).length <= e }
  catch { return false }
}
```

**`JSON.stringify` в `try`** — на объектах с циклом бросит `TypeError`
→ `false`.

**Лимит по умолчанию 16 384 байта.**

**Стоимость:** `JSON.stringify` на большом объекте дорогой. Вызывается
много раз в `Kr()`.

---

# ЧАСТЬ 4. App Block Messenger — `Kr()`

```js
function Kr(t) {
  if (!Cn(t)
      || t.protocol !== "app-block-messenger"
      || t.version !== 1
      || typeof t.bridgeId !== "string"
      || t.bridgeId.length === 0
      || t.bridgeId.length > 128
      || !Cn(t.bindings)
      || !Array.isArray(t.callbacks)
      || !Ze({bindings: t.bindings, callbacks: t.callbacks}, Vr))
    return null;

  const e = Object.entries(t.bindings),
        n = t.bindings,
        r = t.callbacks;

  return e.length + r.length > Ur
      || e.some(([o, s]) => !Fe.test(o) || !ze(s) || !Ze(s))
      || r.some(o => typeof o !== "string" || !Fe.test(o) || Object.hasOwn(n, o))
      || new Set(r).size !== r.length
    ? null
    : {
        protocol: "app-block-messenger",
        version: 1,
        bridgeId: t.bridgeId,
        bindings: Object.fromEntries(e.map(([o, s]) => [o, structuredClone(s)])),
        callbacks: [...r]
      }
}
```

## 4.1 Требования к `appBlockMessenger`

| Поле | Требование |
|---|---|
| `protocol` | ровно `"app-block-messenger"` |
| `version` | ровно `1` |
| `bridgeId` | строка, 1–128 символов |
| `bindings` | plain-объект |
| `callbacks` | массив |
| Общий размер | ≤ 16 384 байта |
| **`bindings` + `callbacks`** | **≤ 32 элементов** |
| Имя в `bindings` | `/^[A-Za-z][A-Za-z0-9_]{0,63}$/` |
| Значение в `bindings` | `ze()` + `Ze()` |
| Имя в `callbacks` | то же + **не должно быть в `bindings`** |
| Дубликаты в `callbacks` | ❌ |

## 4.2 Что делает при успехе

```js
{
  protocol: "app-block-messenger",
  version: 1,
  bridgeId: t.bridgeId,
  bindings: Object.fromEntries(e.map(([o, s]) => [o, structuredClone(s)])),
  callbacks: [...r]
}
```

**`structuredClone(s)`** — **глубокая копия** каждого значения.

**Зачем:** защита от последующей мутации исходного объекта. Если хост
передал объект и потом изменил его — мост не сломается.

**`structuredClone` не клонирует функции** — значит `bindings` не могут
содержать функции. Только данные.

**`[...r]`** — копия массива callbacks.

## 4.3 Зачем `bindings` и `callbacks`

**`bindings`** — значения, доступные виджету (объекты/примитивы).
В `runWidgetCode`: `t.__appBlockMessenger = v`.

**`callbacks`** — имена функций, которые виджет может вызвать. Сам код
функций передаётся отдельно (или резолвится по имени на стороне хоста).

**Пересечение запрещено:**
```js
Object.hasOwn(n, o)
```
Нельзя, чтобы имя было и в `bindings`, и в `callbacks`.

---

# ЧАСТЬ 5. Валидация контента MCP

## 5.1 `be()`, `Tt()`, `Yr()`

```js
const be = t => typeof t === "object" && t !== null && !Array.isArray(t);
const Tt = t => typeof t === "string";
const Yr = (t, e) => t[e] == null || Tt(t[e]);
```

`be` — «это объект (не массив)».
`Tt` — «это строка».
`Yr` — «поле отсутствует или строка».

## 5.2 `qr()` — ресурс

```js
const qr = t => {
  if (!be(t) || !Tt(t.uri)) return false;
  const e = Tt(t.text), n = Tt(t.blob);
  return e !== n && Yr(t, "mimeType") && (t._meta == null || be(t._meta))
};
```

**Требования:**
- объект
- `uri` — строка
- **ровно одно из** `text` или `blob` (`e !== n` — XOR)
- `mimeType` — отсутствует или строка
- `_meta` — отсутствует или объект

**`e !== n`** — XOR для булевых:
- оба `false` → `false` (нет ни text, ни blob)
- оба `true` → `false` (нельзя оба)
- один `true` → `true`

## 5.3 `Zr()` — блок контента

```js
const Zr = t => {
  if (!be(t) || !Tt(t.type)) return false;
  switch (t.type) {
    case "text": return Tt(t.text);
    case "image":
    case "audio": return Tt(t.data) && Tt(t.mimeType);
    case "resource_link": return Tt(t.name) && Tt(t.uri);
    case "resource": return qr(t.resource);
    default: return false
  }
};
```

**Пять типов:**

| `type` | Требования |
|---|---|
| `text` | `text` — строка |
| `image` | `data` — строка + `mimeType` — строка |
| `audio` | `data` + `mimeType` |
| `resource_link` | `name` + `uri` |
| `resource` | вложенный `resource`, проходящий `qr()` |

**Других типов нет.** `video`, `file` — отклоняются.

---

# ЧАСТЬ 6. `hn()` — публикация Host API

```js
hn = ({isFirstParty: t, includePrivate: e = !1, hostHandlesFollowUpMessageAuthorization: n = !1}) => {
  if (!Je) throw new Error("Host API not published");

  const r = Je;

  return Object.fromEntries(
    Object.keys(r)
      .filter(o => (o !== "prepareWidgetDocument" || e)
                && (t || e || !Jr.has(o)))
      .map(o => {
        const s = o;
        return [o, (...i) =>
          !t && ao.has(s)
            && !(s === "sendFollowUpMessage" && n)
            && !uo()
              ? console.warn(`Method \`${s}\` called without synchronous user event`)
              : ho(s, r, i)
        ]
      })
  )
};
```

## 6.1 Три условия фильтра

```js
(o !== "prepareWidgetDocument" || e)      // 1. prepareWidgetDocument только с includePrivate
&& (t || e || !Jr.has(o))                 // 2. Jr скрыт без firstParty/includePrivate
```

**Таблица результатов:**

| `isFirstParty` | `includePrivate` | `prepareWidgetDocument` | 31 метод `Jr` | 8 из `ao` | Прочие |
|---|---|---|---|---|---|
| `false` | `false` | ❌ | ❌ | ✅ (с проверкой) | ✅ |
| `true` | `false` | ❌ | ✅ | ✅ (без проверки) | ✅ |
| `false` | `true` | ✅ | ✅ | ✅ (с проверкой) | ✅ |
| `true` | `true` | ✅ | ✅ | ✅ (без проверки) | ✅ |

## 6.2 Проверка user activation

```js
!t && ao.has(s) && !(s === "sendFollowUpMessage" && n) && !uo()
  ? console.warn(`Method \`${s}\` called without synchronous user event`)
  : ho(s, r, i)
```

**Четыре условия для блокировки:**
1. `!t` — не first-party
2. `ao.has(s)` — метод требует жеста
3. `!(s === "sendFollowUpMessage" && n)` — не исключение
   (`hostHandlesFollowUpMessageAuthorization`)
4. `!uo()` — нет активного user activation

**Тогда:** `console.warn` и **возврат `undefined`** (метод не вызывается).

**`uo()`:**
```js
uo = () => "userActivation" in window.navigator
          ? window.navigator.userActivation.isActive
          : !0    // нет API → считаем активным
```

**Fallback `true`** — в браузерах без `userActivation` проверка пропускается.

**`ho()` возвращает результат метода или `undefined`.**

## 6.3 `ho()` — диспетчер вызова

```js
ho = (t, e, n) => {
  // 1. requestDropdown — с координатами
  if (t === "requestDropdown") {
    const u = In()?.getBoundingClientRect();
    return e.requestDropdown?.call(e, {
      clientX: u?.left ?? 0,
      clientY: u?.top ?? 0,
      ...n[0]
    })
  }

  // 2. requestPrompt — тоже координаты
  if (!ln && t === "requestPrompt") {
    const u = In()?.getBoundingClientRect();
    return e.openPromptInput?.call(e, {
      ...n[0],
      clientX: u?.left ?? 0,
      clientY: u?.top ?? 0
    })
  }

  // 3. callTool — нормализация _meta → meta
  if (t === "callTool") {
    const [u, h] = n;
    return e.callTool?.call(e, u, h).then(d =>
      d.meta == null && d._meta != null ? {...d, meta: d._meta} : d
    )
  }

  // 4. всё остальное
  return e[t]?.call(e, ...n)
};
```

### Три особых случая

**1. `requestDropdown`** — добавляет `clientX`/`clientY` из позиции
`ye` (последний target pointerdown).

**2. `requestPrompt`** — то же, но **только не на мобильном Skybridge**
(`!ln`).

**3. `callTool`** — **мутирует ответ**: если есть `_meta` но нет `meta` —
копирует `_meta` в `meta`. MCP-спецификация использует `_meta`,
но у ChatGPT `meta`.

### `In()` — последний target

```js
In = () => {
  const t = nt()?.contentWindow;
  return !ye || !t || !(ye instanceof t.HTMLElement) ? null : ye;
};
```

`ye` устанавливается в `oo()` на `pointerdown`:
```js
E = y => { ye = y.target };
o.addEventListener("pointerdown", E, {passive: !0});
```

**То есть координаты берутся от элемента, на который нажали** — не от
самого виджета.

---

# ЧАСТЬ 7. События глобалов

```js
vn = "webplus:set_globals";
class Qr extends CustomEvent {
  constructor(n) { super(vn, {detail: {globals: n}}); C(this, "type", vn) }
}

Ne = "openai:set_globals";
class to extends CustomEvent {
  constructor(n) { super(Ne, {detail: {globals: n}}); C(this, "type", Ne) }
}
```

**Два события:**

| Имя | Класс | Когда |
|---|---|---|
| `webplus:set_globals` | `Qr` | legacy |
| `openai:set_globals` | `to` | актуальное |

**`C(this, "type", vn)`** — `defineProperty` для `type`. В `CustomEvent`
`type` уже есть, но read-only. Здесь переопределяется с `configurable: true`.

**Зачем:**
```js
if (!(a instanceof U)) return;    // U = to
```
Проверка `instanceof` — защита от поддельных событий. Виджет не может
создать свой `openai:set_globals` с тем же именем (класс недоступен).

## 7.1 `Is()` — отправка

```js
Is = (t, e) => {
  t.dispatchEvent(new to(e));
  t.dispatchEvent(new Qr(e));
};
```

**Два события на каждое обновление** — совместимость.

---

# ЧАСТЬ 8. `Xt()` — инъекция глобалов

```js
Xt = ({root: t = nt(), additionalGlobals: e, sandboxLogger: n, isInit: r = !1}) => {
  if (t?.contentWindow) {
    if (r) {                              // ← ВЕТКА 1: инициализация
      t.contentWindow.oai = e;
      t.contentWindow.openai = e;
      t.contentWindow.webplus = e;
      return
    }

    // ВЕТКА 2: обновление
    t.contentWindow.oai ??= {};
    t.contentWindow.openai ??= {};
    t.contentWindow.webplus ??= {};

    for (const [l, a] of Object.entries(e))
      try {
        t.contentWindow.oai[l] = a;
        t.contentWindow.openai[l] = a;
        t.contentWindow.webplus[l] = a
      } catch (u) { n.environmentError(u, !1) }

    Is(t.contentWindow, e)
  }
};
```

## 8.1 Две ветки

**`isInit: true`** — **полная замена** ссылки:
```js
t.contentWindow.oai = e;
t.contentWindow.openai = e;
t.contentWindow.webplus = e;
```

**Три глобала указывают на ОДИН объект.**

**`isInit: false`** — **поэлементное обновление**:
```js
t.contentWindow.oai[l] = a;
t.contentWindow.openai[l] = a;
t.contentWindow.webplus[l] = a;
```

## 8.2 Проблема ветки обновления

**`try/catch` внутри цикла:**
```js
try {
  t.contentWindow.oai[l] = a;
  ...
} catch (u) { n.environmentError(u, !1) }
```

**Если `oai[l] = a` бросит** (например, `oai` заморожен виджетом
через `Object.freeze`) — то `openai[l]` и `webplus[l]` **не выполнятся**.
Глобалы рассинхронизируются.

**Виджет может это использовать:**
```js
Object.freeze(window.openai);
```
→ при следующем обновлении `openai` не получит поле, но `oai` не тоже
(throw на первом присвоении). Все три останутся со старым.

**`n.environmentError(u, false)`** — второй аргумент `false` означает
«не фатально».

## 8.3 Три имени глобала

| Имя | Статус |
|---|---|
| `window.oai` | актуальное |
| `window.openai` | история (старое) |
| `window.webplus` | legacy (внутреннее) |

**Все три — один объект.** Изменение через любое видно во всех.

---

# ЧАСТЬ 9. Защита навигации — `oo()`

Самая большая функция файла. Разберём по частям.

## 9.1 Сигнатура

```js
oo = ({
  doc: t,                              // iframe element
  isFirstParty: e,
  enableExternalNavigationGating: n = !1,
  disableExternalNavigation: r = !1
}) => { ... }
```

## 9.2 Подготовка

```js
const o = t.contentWindow;
if (!o) return;

const s = hn({isFirstParty: e, includePrivate: e || n});
ro(s);

const i = r ? Reflect.get(o, "navigation") : void 0;
eo(i) && i.addEventListener("navigate", y => {
  if (!y.cancelable) return;
  new URL(y.destination.url, o.location.href).origin !== o.location.origin
    && y.preventDefault()
});
```

### `s = hn({isFirstParty: e, includePrivate: e || n})`

**`includePrivate: e || n`** — приватные методы доступны если виджет
first-party **или** включена навигационная защита.

**Зачем:** для перехвата навигации нужны `openExternal`,
`openExternalForm`, `sendInstrument` — часть из них в `Jr`.

### `ro(s)` — Escape-key

```js
ro = t => {
  ee = t;
  !Ln && (Ln = !0, window.addEventListener("keydown", e => {
    e.key === "Escape" && !e.defaultPrevented && ee?.notifyEscapeKey?.()
  }, {passive: !0}))
};
```

**Один глобальный слушатель** на `window` (не на iframe!).
`Ln` — защита от двойной регистрации.

**`ee`** — текущий API-объект. Перезаписывается при каждом `ro(s)`.

### Navigation API (только при `disableExternalNavigation`)

```js
const i = r ? Reflect.get(o, "navigation") : void 0;
```

**`window.navigation`** — Navigation API (Chrome 102+). Позволяет
перехватывать навигацию.

```js
eo(i) && i.addEventListener("navigate", y => {
  if (!y.cancelable) return;
  new URL(y.destination.url, o.location.href).origin !== o.location.origin
    && y.preventDefault()
});
```

**Блокирует любую кросс-origin навигацию.**

**`eo(t)`** — проверка «есть `addEventListener`»:
```js
eo = t => typeof t == "object" && t != null
       && typeof Reflect.get(t, "addEventListener") == "function";
```

## 9.3 `fe()` — обработка внешнего URL

```js
fe = (t, e, n, r) => {
  try {
    const o = new URL(n, e.href);

    // 1. Тот же документ (только hash) → не внешний
    if (o.origin === e.origin
        && o.pathname === e.pathname
        && o.search === e.search) return !1;

    // 2. Другой хост/протокол или forceHostOpen → openExternal
    if ((o.hostname !== e.hostname
         || o.protocol !== e.protocol
         || r?.forceHostOpen) && t.openExternal) {
      const i = {href: o.href};
      r?.openMode != null && (i.openMode = r.openMode);
      t.openExternal(i);
      return !0
    }
  } catch { return }
};
```

**Возвращает:**
- `false` — это не внешний переход (можно продолжить навигацию)
- `true` — обработано (нужно `preventDefault`)
- `undefined` — ошибка парсинга

### Три условия «внешний»

1. **Тот же origin + path + search** → `false` (это якорь `#section`)
2. **Другой `hostname`** → внешний
3. **Другой `protocol`** → внешний
4. **`forceHostOpen: true`** → принудительно внешний

**Заметь:** проверка `hostname`, а не `origin`. Значит `https://example.com:8080`
считается **тем же** хостом, что `https://example.com`. Порт не учитывается.

**Это баг:** переход с `:443` на `:22` (или `:8080`) не будет перехвачен.

## 9.4 `a()` — обработка формы

```js
const a = y => {
  try {
    const w = y.action || o.location.href,
          O = new URL(w, o.location.href);

    if (!n) return !1;                    // ← гейтинг отключён

    const m = (y.getAttribute("target") ?? "").trim(),
          _ = (y.getAttribute("method") ?? "get").trim().toLowerCase(),
          f = (y.getAttribute("enctype") ?? "").trim() || void 0;

    // случай 1: openExternalForm есть
    if (s.openExternalForm) {
      const p = new o.FormData(y),
            A = [];
      let L = !1;
      for (const [T, I] of p.entries())
        typeof I === "string" ? A.push({name: T, value: I}) : L = !0;

      return s.openExternalForm({
        action: O.href, method: _, target: m, enctype: f,
        entries: A, hasFiles: L
      }), !0
    }

    // случай 2: GET — в URL
    if (_ === "get") {
      const p = new URL(O.href),
            A = new o.FormData(y);
      for (const [L, T] of A.entries())
        typeof T === "string" && p.searchParams.append(L, T);
      return !!fe(s, o.location, p, {forceHostOpen: !0})
    }

    // случай 3: POST — только action
    return !!fe(s, o.location, O.href, {forceHostOpen: !0})
  } catch { return !1 }
};
```

### Три ветки

**1. `openExternalForm` есть** — передать **всю форму** хосту:
```js
{
  action, method, target, enctype,
  entries: [{name, value}],   // только текстовые
  hasFiles: bool              // есть ли файлы
}
```

**2. GET** — параметры формы в query-строку, потом `fe()`.

**3. POST** — только action (данные не передаются).

**Файлы:** при `openExternalForm` — только флаг `hasFiles`, сами файлы
**не передаются**.

## 9.5 `u()` — обработчик клика

```js
const u = y => {
  // 1. только user-initiated
  if (!y.isTrusted
      || (s.sendInstrument?.call(s, {type: "interaction", label: "click"}),
          y.defaultPrevented
          || !(y instanceof o.MouseEvent))) return;

  // 2. найти ближайший <a>
  const w = y.target;
  if (!(w instanceof o.Element)) return;
  const O = w.closest("a");
  if (!O) return;

  // 3. получить href (обычный или SVG xlink)
  const m = O.getAttribute("href")
         ?? O.getAttributeNS("http://www.w3.org/1999/xlink", "href"),
        _ = O.namespaceURI === "http://www.w3.org/2000/svg"
            ? (O.href?.animVal ?? m) : m;
  if (!_) return;

  // 4. абсолютный URL
  let f;
  if (O instanceof o.HTMLAnchorElement) f = O.href;
  else try { f = new URL(_, O.baseURI) } catch { return }

  // 5. modifier keys → новый таб
  const {location: p} = o,
        A = e && (y.altKey || y.ctrlKey || y.metaKey || y.shiftKey);

  fe(s, p, f, {forceHostOpen: n, openMode: A ? "tab" : void 0})
    && y.preventDefault()
};
```

### Что делает

1. **`!y.isTrusted`** → выйти. Только настоящие клики.
2. **`sendInstrument({type: "interaction", label: "click"})`** — метрика.
3. **`y.defaultPrevented`** → выйти (кто-то уже обработал).
4. **`y instanceof o.MouseEvent`** — только MouseEvent (не PointerEvent).
5. **`closest("a")`** — подняться к ссылке.
6. **`getAttributeNS("http://www.w3.org/1999/xlink", "href")`** — SVG-ссылки.
7. **Modifier keys** — `ctrl`/`meta`/`alt`/`shift` открывают **в новом табе**
   (только для first-party).

**`A = e && (...)`** — `openMode: "tab"` только если first-party **и**
нажат модификатор.

## 9.6 `h()` — копирование

```js
const h = () => {
  const y = o.getSelection();
  !y || y.toString().trim().length === 0
    || s.sendInstrument?.call(s, {type: "interaction", label: "copy"})
};
```

**Метрика копирования.** Отправляется, только если выделение не пустое.

## 9.7 `d()` — скролл (`throttled`)

```js
const d = () => {
  l || (l = o.setTimeout(() => { l = void 0 }, 1500),
        s.sendInstrument?.call(s, {type: "interaction", label: "scroll"}))
};
```

**`no = 1500`** — 1.5 секунды throttle.

**`l`** — таймер. Пока не истёк — не отправлять.

**`o.setTimeout`** — таймер **в iframe** (не в главном окне).

## 9.8 `g()` / `E()` — Escape и pointerdown

```js
const g = y => {
  ye = y.target;
  y.key === "Escape" && s.notifyEscapeKey?.()
};

const E = y => { ye = y.target };
```

**`ye`** — последний target. Используется в `In()` для координат.

## 9.9 Регистрация слушателей

```js
o.addEventListener("pointerdown", E, {passive: !0});
o.addEventListener("keydown", g, {passive: !0});
o.addEventListener("click", u, {capture: n});   // ← capture только при гейтинге
```

**`capture: n`** — при `enableExternalNavigationGating` клик перехватывается
**до** обработчиков виджета.

**Без гейтинга** — `capture: false` (bubble), клик обрабатывается после.

## 9.10 Патч `HTMLAnchorElement.prototype.click`

```js
if (n) {
  const y = o.HTMLAnchorElement.prototype.click;
  o.HTMLAnchorElement.prototype.click = function () {
    const m = [], _ = f => m.push(f);
    this.addEventListener("click", _, {capture: !0});
    try { y.call(this) }
    finally { this.removeEventListener("click", _, {capture: !0}) }

    m.length === 0 || m[0].defaultPrevented
      || this.target.toLowerCase() !== "_blank"
      || this.hasAttribute("download")
      || (this.protocol !== "http:" && this.protocol !== "https:")
      || fe(s, o.location, this.href, {forceHostOpen: !0})
  }
}
```

### Что делает

Перехватывает **программный** клик по ссылке (`a.click()` из JS).

1. Временно добавляет capture-слушатель `_`
2. Вызывает оригинальный `click()`
3. Убирает слушатель
4. Если сработал — проверить условия:
   - `m[0].defaultPrevented` — уже отменён → ничего
   - `this.target.toLowerCase() !== "_blank"` — не новый таб → ничего
   - `this.hasAttribute("download")` — скачивание → ничего
   - `protocol !== "http:" && !== "https:"` — не http(s) → ничего
5. Иначе → `fe(..., {forceHostOpen: true})`

**То есть:** `a.click()` с `target="_blank"` на http(s)-ссылке без
`download` → открывается через хост.

**Проблема:** `this.protocol` для относительной ссылки будет
`location.protocol` — обычно `https:`. Значит относительная ссылка
с `_blank` тоже уйдёт в `fe()`.

**Второй `!==` вместо `&&`:**
```js
(this.protocol !== "http:" && this.protocol !== "https:")
```
Правильно: «не http И не https» → пропустить. Логика верна.

## 9.11 Патч `submit`

```js
const w = m => {
  if (m.defaultPrevented) return;
  const _ = m.target;
  _ instanceof o.HTMLFormElement && a(_) && m.preventDefault()
};
o.addEventListener("submit", w);

const O = o.HTMLFormElement.prototype.submit;
o.HTMLFormElement.prototype.submit = function (...m) {
  if (!a(this)) return O.apply(this, m)
};
```

**Два перехвата:**
1. Событие `submit` — предотвратить, если `a()` вернул `true`
2. `form.submit()` — не вызывать оригинал, если `a()` вернул `true`

**Разница с `click()`:** здесь **нет** проверки `_blank`/`download`/протокола.
Любая форма уходит в `a()`.

## 9.12 Патч `window.open`

```js
o.open = y => {
  if (!y) return null;
  const {location: w} = o;
  fe(s, w, y, {forceHostOpen: n, openMode: e ? "tab" : void 0});
  return null
};
```

**`window.open` внутри iframe заменяется полностью.** Всегда возвращает `null`.

**`openMode: "tab"`** — только first-party.

**Это и есть причина `"window.open returned null; likely blocked or disallowed"`**
из ваших логов — `window.open` в iframe всегда `null`.

## 9.13 Метрики

```js
o.addEventListener("copy", h, {passive: !0});
o.addEventListener("scroll", d, {passive: !0});
o.document.addEventListener("scroll", d, {capture: !0, passive: !0});
```

**Два scroll-слушателя:**
- на `window` — bubble
- на `document` — capture (ловит скролл внутри элементов)

**Throttle общий** (`l`), так что двойной скролл → одна метрика.

---

# ЧАСТЬ 10. Параметры URL

```js
Mt = new URLSearchParams(window.location.search);
Mt.get("isStorybook");

so = Mt.get("sessionId") ?? "";
Et = Mt.get("app") ?? "";
io = Mt.get("clientPlatform") ?? "";
Zn = Mt.get("deviceType") === "mobile";

ln = Zn && Et === "skybridge";                          // mobile skybridge
Jn = !Zn && Et === "skybridge";                          // desktop skybridge
Qn = Et === "skybridge" && io === "android";             // android
un = Et === "skybridge" && Mt.get("backgroundMode") === "transparent";

dn = Et === "chatgpt" || Et === "chatgpt-tt" || Et === "chatgpt-staging";
tr = dn && Mt.get("darkModeType") === "increased";

Mn = Mt.get("locale") ?? "";
co = Mt.get("unsafeSkipTargetOriginCheck") === "true";
```

## 10.1 Таблица режимов

| URL-параметр | Переменная | Что |
|---|---|---|
| `?app=skybridge&deviceType=mobile` | `ln` | мобильный Skybridge |
| `?app=skybridge` (без mobile) | `Jn` | десктоп Skybridge |
| `?app=skybridge&clientPlatform=android` | `Qn` | Android |
| `?app=skybridge&backgroundMode=transparent` | `un` | прозрачный фон |
| `?app=chatgpt` / `chatgpt-tt` / `chatgpt-staging` | `dn` | режим ChatGPT |
| `?app=chatgpt&darkModeType=increased` | `tr` | OLED-тема |
| `?locale=ru-RU` | `Mn` | локаль |
| `?unsafeSkipTargetOriginCheck=true` | `co` | **отключить проверку origin** |

## 10.2 CSS-классы

```js
er = "mobileSkybridge",
nr = "desktopSkybridge",
rr = "androidSkybridge",
or = "transparentSkybridgeBackground",
sr = "chatgpt",
ir = "chatgptOled";

dn && document.documentElement.classList.add(sr);
tr && document.documentElement.classList.add(ir);
Qn && document.documentElement.classList.add(rr);
un && document.documentElement.classList.add(or);
ln ? document.documentElement.classList.add(er)
   : Jn && document.documentElement.classList.add(nr);
```

**6 классов** на `<html>`:
- `chatgpt` — при `app=chatgpt*`
- `chatgptOled` — при `darkModeType=increased`
- `androidSkybridge` — при `clientPlatform=android`
- `transparentSkybridgeBackground` — при `backgroundMode=transparent`
- `mobileSkybridge` / `desktopSkybridge` — взаимоисключающие

## 10.3 `co` — `unsafeSkipTargetOriginCheck`

**Единственный способ получить `targetOrigin: "*"`.** Из URL-параметра.
То есть требует контроля над URL загрузки iframe.

---

# ЧАСТЬ 11. `Hs()` — выбор targetOrigin

Из предыдущего разбора:

```js
Hs = () => {
  const co = Mt.get("unsafeSkipTargetOriginCheck") === "true";
  const targetOrigin = co ? "*"
    : Et === "chatgpt" ? "https://chatgpt.com"
    : Et === "chatgpt-tt" ? "https://tt.chatgpt.com"
    : Et === "feather" ? "https://feather.openai.com"
    : Et === "skybridge" ? "https://skybridge.oaistatic.com"
    : "*";
  return targetOrigin
};
```

**Приоритет:** `co` > `app`-параметр > `"*"`.

---

# ЧАСТЬ 12. `runHTMLAsync` — `Dr()`

## 12.1 Сигнатура

```js
async function* Dr({
  sandboxLogger: t,
  features: e,
  html: n,
  theme: r,
  safeArea: o,
  transformPosition: s,
  additionalGlobals: i,
  expectReadySignal: l,
  expectedReadySignal: a,
  appInitialization: u,
  ownerColorScheme: h,
  isFirstParty: d = !1,
  forwardConsoleLogs: g = !0,
  measureWidth: E = !1,
  measureHeight: y = !1,
  disableMutationObserver: w = !1,
  onIframeReady: O,
  prepareDocument: m,
  enableExternalNavigationGating: _ = !1,
  disableExternalNavigation: f = !1,
  allowPopups: p = !0
})
```

**21 параметр.**

## 12.2 Создание iframe — `jr()`

```js
jr = (t, e) => {
  const n = nt();                                    // текущий #root
  document.querySelectorAll(`iframe#${De}`).forEach(o => {
    (o !== n || !o.hasAttribute("data-preview-ready")) && an(o)   // удалить старые
  });

  const r = document.createElement("iframe");
  r.sandbox = e?.allowPopups === false
    ? "allow-scripts allow-same-origin"
    : "allow-scripts allow-same-origin allow-popups allow-popups-to-escape-sandbox allow-forms";
  r.id = De;                                          // "root"
  r.style.visibility = "hidden";
  nt() || Yn(e?.ownerColorScheme);
  e?.ownerColorScheme != null && (r.style.colorScheme = e.ownerColorScheme);
  ...
};
```

### Атрибут `sandbox`

| `allowPopups` | Значение |
|---|---|
| `false` | `allow-scripts allow-same-origin` |
| `true` (дефолт) | `allow-scripts allow-same-origin allow-popups allow-popups-to-escape-sandbox allow-forms` |

**`allow-scripts` + `allow-same-origin`** — **песочница неэффективна**
(известная проблема: скрипты могут снять `sandbox`).

**`allow-popups-to-escape-sandbox`** — позволяет попапам выходить из
песочницы.

**`allow-forms`** — формы работают (утечка через `<form>` возможна).

**Без `allow-modals`, `allow-top-navigation`, `allow-downloads`,
`allow-pointer-lock`, `allow-presentation`, `allow-storage-access-by-user-activation`.**

### `visibility: hidden`

**iframe скрыт до готовности.** `data-preview-ready` появляется при
`Ge()`, тогда `visibility` снимается.

### `id = "root"` — `De`

**Все iframe-ы имеют `id="root"`.** Отсюда `document.querySelectorAll('iframe#root')`.

## 12.3 Подготовка документа — `Fr()`

```js
const Fr = async () => {
  if (et || !F.contentDocument || !F.contentWindow) return;

  const P = F.contentDocument,
        wn = F.contentWindow;

  P.open();

  // 1. app block messenger
  if (i != null && Object.hasOwn(i, "__appBlockMessenger")) {
    const $ = X().notifyAppBlockMessage;
    H = li({
      viewWindow: wn,
      additionalGlobals: i,
      notify: $ == null ? void 0 : Z => {
        F.isConnected && F.contentDocument === P && $(Z)
      }
    })
  }

  // 2. onIframeReady
  if (O) {
    const $ = O({root: F, viewWindow: wn, document: P});
    z = typeof $ === "function" ? $ : null
  }

  // 3. инъекция глобалов
  if ((i || o) && Xt({
    root: F,
    additionalGlobals: Object.assign(i ?? {}, {safeArea: o}),
    sandboxLogger: t,
    isInit: !0
  }))

  // 4. prepareDocument с таймаутом 10 с
  if (m != null) {
    try {
      await Promise.race([
        new Promise(($, Z) => {
          de = $;
          xe = setTimeout(() => Z(new Error("Document preparation timed out")), 1e4)
        }),
        m()
      ])
    } finally { de = void 0, clearTimeout(xe) }

    if (et || !F.isConnected || F.contentDocument !== P) { Ct(!0); return }
  }

  // 5. защита навигации
  oo({doc: F, isFirstParty: d, enableExternalNavigationGating: _, disableExternalNavigation: f});

  // 6. подписчики
  Mr({doc: F});                                          // ?
  ei({doc: F, countDiagnosticOnce: he});                 // ?
  Qs({doc: F});                                          // ?
  ti({doc: F, sandboxLogger: t, transformPosition: s, onMessage: bn, forwardConsoleLogs: g});
  Js({doc: F});                                          // ?

  // 7. запись HTML
  P.write(n);
  if (et) return;

  // 8. observers
  const Pt = P.documentElement;

  const Tn = () => l && Pt.hasAttribute("data-ready")
                  && (a == null || Pt.getAttribute("data-ready") === a);

  ai(P, {measureHeight: y, measureWidth: E});

  L = new ResizeObserver(() => { Ue(), We() });
  A = new MutationObserver($ => {
    // data-fatal?
    if (Pt.hasAttribute("data-fatal")) { Ct(); return }

    for (const Z of $) if (Z.target === Pt) {
      if (Z.attributeName === "data-ready" && Tn()) { Ge(); break }
    } else if (u == null
               && Z.addedNodes.length === 0
               && Z.removedNodes.length > 0
               && Z.target.parentElement === P.body
               && Z.target.childNodes.length === 0
               && Array.from(P.body.childNodes).every(Dt => Dt === Z.target || ui(Dt))) {
      Ct();   // ← body опустошён → завершить
      break
    }

    Ue(); We();
  });

  Mn && (Pt.lang = Mn);
  r && Or({theme: r, doc: F});
  o && Tr({doc: F, safeArea: o});

  L.observe(Pt, {box: "border-box"});
  w || A.observe(Pt, {childList: !0, attributes: !0, subtree: !0});

  l || P.addEventListener("DOMContentLoaded", Ge, {once: !0});

  P.close();

  Pt.hasAttribute("data-fatal") ? Ct() : Tn() && Ge();
};
```

### Детектор пустого body

```js
else if (u == null
         && Z.addedNodes.length === 0
         && Z.removedNodes.length > 0
         && Z.target.parentElement === P.body
         && Z.target.childNodes.length === 0
         && Array.from(P.body.childNodes).every(Dt => Dt === Z.target || ui(Dt)))
```

**Условия:**
1. `u == null` — нет MCP Apps инициализации
2. Ничего не добавлено
3. Что-то удалено
4. Родитель — `body`
5. У цели 0 дочерних узлов
6. Остальные узлы в `body` — пустые/комментарии (`ui()`)

**Тогда:** `Ct()` — завершить запуск.

**Зачем:** React-приложение могло размонтироваться (`root.unmount()`),
оставив пустой `body`. Это сигнал «виджет умер».

**`ui()`:**
```js
function ui(t) {
  return t.nodeType === Node.COMMENT_NODE ? !0
       : t.nodeType === Node.TEXT_NODE ? !(t.textContent?.trim())
       : t.nodeType === Node.ELEMENT_NODE ? t.childNodes.length === 0
       : !0
}
```

**Комментарии** — пустые. **Текстовые** — пустые если только пробелы.
**Элементы** — пустые если без детей.

### Таймаут `prepareDocument`

```js
await Promise.race([
  new Promise(($, Z) => {
    de = $;
    xe = setTimeout(() => Z(new Error("Document preparation timed out")), 1e4)
  }),
  m()
])
```

**10 секунд.** `de` — резолвер (для отмены), `xe` — таймер.

**Если истёк** — `catch` в вызывающем коде, `Ct()`.

### `data-ready`

```js
const Tn = () => l && Pt.hasAttribute("data-ready")
                && (a == null || Pt.getAttribute("data-ready") === a);
```

**Два условия:**
1. `expectReadySignal` (l) — ждём сигнал
2. `data-ready` присутствует
3. **Если `expectedReadySignal` задан** — значение должно совпасть

**Проверка атрибута** — в MutationObserver (`attributeName === "data-ready"`)
и сразу после `P.close()`.

## 12.4 `Ct()` — завершение

```js
const Ct = (P = !1) => {
  if (et) return;
  et = !0;
  de?.();
  clearTimeout(xe);
  clearTimeout(je);
  A?.disconnect();
  L?.disconnect();
  P || he(G ? "html.preview_runtime_error" : "html.preview_startup_error");
  G || F.remove();
  T({type: bt.RUN_COMPLETE, duration: null, wasCancelled: P, wasFatalError: !P});
  z?.();
  H?.();
  I()
};
```

**Параметр `P` = «отменено»:**
- `Ct()` → `wasCancelled: false, wasFatalError: true`
- `Ct(true)` → `wasCancelled: true, wasFatalError: false`

**Метрики:** `html.preview_runtime_error` если `G` (запустился),
иначе `html.preview_startup_error`.

**`G || F.remove()`** — если виджет уже запустился, iframe **остаётся**.
Иначе — удаляется.

**`I()`** — завершить `ni()`-генератор (`done()`).

## 12.5 `Ge()` — готовность

```js
const Ge = () => {
  if (et || G || !F.isConnected) return;
  G = !0;
  F.setAttribute("data-preview-ready", "");
  F.style.removeProperty("visibility");
  l && d && !E && A?.disconnect();     // ← first-party: отключить observer
  En && an(En);
  F.ownerDocument.documentElement.style.colorScheme = F.style.colorScheme;
  Ue(); We();
  t.timeEnd("html.evaluation");
  he("html.preview_ready");
  bn({type: bt.ENVIRONMENT_STATUS, status: qn.RUNNING_CODE})
};
```

### Что происходит при готовности

1. `G = true` — «запустился»
2. `data-preview-ready` на iframe
3. `visibility` снят → **iframe виден**
4. **`l && d && !E && A?.disconnect()`** — если ждали сигнал, first-party,
   и не меряем ширину → **MutationObserver отключается**
5. `an(En)` — удалить старый iframe (`En` = `nt()` до создания нового)
6. Синхронизировать `color-scheme`
7. Замеры (`Ue` — height, `We` — width)
8. Событие `RUNNING_CODE`

**Пункт 4 — важно:** после `data-ready` observer больше не нужен
(если не меряем ширину). Экономия.

**Но:** отключается и обработка `data-fatal`. Если ошибка произойдёт
**после** готовности — не будет поймана через observer.

## 12.6 `Ue()` / `We()` — замеры

```js
const Ue = () => {
  if (!y || !G || et) return;                        // measureHeight
  const $ = Xs(P);
  $ && $ !== _n && (X()?.notifyIntrinsicHeight?.call(X(), $), _n = $)
};

const We = () => {
  if (!E || !G || et) return;                        // measureWidth
  const $ = Ks(P);
  $ && $ !== Sn && (X()?.notifyIntrinsicWidth?.call(X(), $), Sn = $)
};
```

**`Xs(P)` / `Ks(P)`** — измерить высоту/ширину документа.

**`_n` / `Sn`** — последнее отправленное значение. Если не изменилось —
не отправлять.

**`X()`** — приватный API (`notifyIntrinsicHeight`/`Width` в `Jr`).

## 12.7 `appInitialization`

```js
u != null && (je = setTimeout(() => {
  t.error("MCP app initialization timed out", null);
  Ct()
}, 3e4), u.then(() => { clearTimeout(je) }, P => {
  et || (t.error("MCP app initialization failed", P), Ct())
}));
```

**Таймаут 30 секунд.** Если MCP-приложение не инициализировалось —
`Ct()` (завершить с ошибкой).

**При ошибке** — тоже `Ct()`.

**`je`** — таймер (очищается в `Ct()`).

## 12.8 `F.src = "about:blank"`

```js
F.addEventListener("load", () => setTimeout(() => {
  Fr().catch(P => {
    et || (t.error("Failed to prepare preview document", P), Ct())
  })
}, 0), {once: !0});

F.src = "about:blank";
```

**`src = "about:blank"`** — запускает загрузку пустого документа. Событие
`load` → через `setTimeout(0)` → `Fr()`.

**Зачем `about:blank` вместо `srcdoc`:** контроль над моментом.
`srcdoc` начал бы исполнять код сразу.

**`setTimeout(0)`** — следующий тик, после полной инициализации iframe.

---

# ЧАСТЬ 13. Подписчики документа

Пять функций вызываются в `Fr()`. Их содержимое не показано целиком,
но по именам:

| Функция | Что |
|---|---|
| `Mr({doc})` | ? (вероятно, защита от clipboard-инъекций) |
| `ei({doc, countDiagnosticOnce})` | диагностика |
| `Qs({doc})` | ? |
| `ti({doc, sandboxLogger, transformPosition, onMessage, forwardConsoleLogs})` | **консоль и ошибки** |
| `Js({doc})` | ? (перехват `HTMLImageElement.src` — `sediment://`, `file-service://`) |

## 13.1 `Js()` — перехват `img.src`

```js
const qs = ["sediment://", "file-service://"],
      Zs = "/blank.svg";

Js = ({doc: t}) => {
  const e = t.contentWindow;
  if (!e) return;

  const n = Object.getOwnPropertyDescriptor(e.HTMLImageElement.prototype, "src");
  n && Object.defineProperty(e.HTMLImageElement.prototype, "src", {
    set(r) {
      ...
    }
  });
};
```

**Переопределяет setter `img.src`** — перехватывает схемы `sediment://`
и `file-service://` (внутренние OpenAI).

**`/blank.svg`** — заглушка для заблокированных.

**Зачем:** изображения из внутренних сервисов ChatGPT нельзя грузить
напрямую из iframe. Подменяются на заглушку (или проксируются).

## 13.2 `ti()` — консоль

Перехватывает `console.*` в iframe, передаёт через `onMessage`, применяет
`transformPosition` к стектрейсам.

**`forwardConsoleLogs`** — если `false`, не передавать.

---

# ЧАСТЬ 14. `Mr()`, `ei()`, `Qs()`, `ti()`

Не разобраны полностью — но по сигнатурам:

- **`Mr({doc})`** — подписчик без дополнительных параметров
- **`ei({doc, countDiagnosticOnce})`** — `he()` — отправка диагностики один раз
- **`Qs({doc})`** — подписчик
- **`ti({...})`** — консоль/ошибки с `transformPosition`
- **`Js({doc})`** — `img.src` (см. 13.1)

---

# ЧАСТЬ 15. Утилиты

## 15.1 `_t()` — preload + import

Разобран в документе #4. Создаёт `<link rel=modulepreload>`, ждёт CSS,
ловит `vite:preloadError`.

## 15.2 `ni()` — callback → generator

Разобран в документе #4.

## 15.3 `nt()` — текущий iframe

```js
const De = "root", nt = () => document.getElementById(De);
```

## 15.4 `an()` — удалить iframe

```js
an = t => {
  t.dispatchEvent(new Event("preview-cancel")),
  t.remove()
};
```

**Событие `preview-cancel`** → `runHTMLAsync` ловит через
`F.addEventListener("preview-cancel", () => Ct(!0), {once: !0})`
→ завершение с `wasCancelled: true`.

## 15.5 `Yn()` — сброс color-scheme

```js
Yn = t => {
  t ? document.documentElement.style.colorScheme = t
    : document.documentElement.style.removeProperty("color-scheme")
};
```

## 15.6 `xr()` — resetPreview

```js
xr = () => {
  document.querySelectorAll(`iframe#${De}`).forEach(an);
  Yn()
};
```

**Удаляет все iframe `#root` + сбрасывает color-scheme.**

## 15.7 `fo()` — проверка null

```js
function fo(t, e = "<unknown>") {
  if (t == null) throw new Error(`${e}: unexpected nullish value: ${JSON.stringify(t)}`);
  return t
}
```

## 15.8 `Mt` — URLSearchParams

```js
Mt = new URLSearchParams(window.location.search);
```

---

# ЧАСТЬ 16. Модуль `bi` — Python

```js
bi = Object.freeze(Object.defineProperty({
  __proto__: null,
  runPythonInWorker: Gs,
  terminatePythonWorker: Rr
}, Symbol.toStringTag, {value: "Module"}))
```

**Экспорт `m`.** Импортируется в `run-user-code.js`:
```js
await import("./main-BR1bOQyH.js").then(u => u.m)
```

## 16.1 `Gs()` — `runPythonInWorker`

```js
Gs = async (code, logger, onMessage) => {
  try { Lt ??= await js(); }                    // ← ОДИН воркер
  catch (r) { logger.error("failed to create python worker", r) }

  return new Promise((resolve, reject) => {
    const s = Lt;
    if (!s) return reject(new Error("Python worker is unavailable"));

    let i = false;
    const onDone = () => {
      i || (i = true, logger.timeEnd("python.evaluation"), resolve())
    };

    const h = g => {
      if (g.data.type === "security_policy_violation") {
        X().notifySecurityPolicyViolation(g.data.payload);
        return    // ← не пропускает в onMessage
      }
      onMessage(g.data);
      g.data.type === "run_complete" && (cleanup(), onDone())
    };

    s.onmessage = h;                             // ← ПЕРЕЗАПИСЫВАЕТ
    s.postMessage({code});
  })
};
```

**Проблемы:**
1. `Lt ??=` — один воркер на всё
2. `s.onmessage = h` — перезапись, параллельные запуски ломаются
3. `security_policy_violation` не идёт в `onMessage`

## 16.2 `Rr()` — `terminatePythonWorker`

```js
Rr = () => {
  try {
    Lt?.dispatchEvent(new Event("terminate")),
    Lt?.terminate()
  } finally { Lt = null }
};
```

**`dispatchEvent(new Event("terminate"))`** → в воркере
`self.addEventListener("terminate", d)` → `Ct()`.

**`Lt.terminate()`** — жёсткое убийство.

**`Lt = null`** — следующий запуск создаст новый.

---

# ЧАСТЬ 17. Схема взаимодействия

```
┌───────────────────────────────────────────────────────────────┐
│ Главный документ: web-sandbox.oaiusercontent.com              │
│                                                                │
│  main.js                                                       │
│   ├─ Mt = URLSearchParams          ← app, deviceType, locale   │
│   ├─ классы на <html>              ← chatgpt, androidSkybridge │
│   ├─ Je = null                     ← публикуемый API           │
│   ├─ Lt = null                     ← Python worker            │
│   ├─ F = iframe #root              ← (внутри Dr)              │
│   └─ $s({runUserCode, buildUserCode, prepareEnvironment,      │
│           runWidgetCode, runComponent, validate})             │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ iframe#root  sandbox="allow-scripts allow-same-origin"   │  │
│  │              +allow-popups +allow-popups-to-escape      │  │
│  │              +allow-forms                               │  │
│  │                                                          │  │
│  │  window.oai = window.openai = window.webplus = {...}   │  │
│  │                                                          │  │
│  │  document.write(html)                                    │  │
│  │  <script type="module">...                                │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  oo() — защита навигации на contentWindow                      │
│   ├─ pointerdown → ye (последний target)                       │
│   ├─ keydown → Escape → notifyEscapeKey                        │
│   ├─ click (capture?) → ближайший <a> → fe()                   │
│   ├─ submit → a() → openExternalForm / fe()                    │
│   ├─ HTMLAnchorElement.click пропатчен                         │
│   ├─ HTMLFormElement.submit пропатчен                          │
│   ├─ window.open пропатчен → всегда null                       │
│   ├─ copy → sendInstrument(label: copy)                        │
│   └─ scroll → sendInstrument(label: scroll) [throttle 1.5s]    │
│                                                                │
│  Xt() — инъекция глобалов                                      │
│   ├─ isInit → полная замена (3 ссылки на один объект)          │
│   └─ update → поэлементно + dispatchEvent(set_globals)         │
└───────────────────────────────────────────────────────────────┘

Хост (ChatGPT) ←─ postMessage ─→ фрейм
  ├─ ui/message / sendFollowUpMessage
  ├─ ui/open-link / openExternal
  ├─ ui/request-display-mode / requestDisplayMode
  ├─ tools/call / callMcp
  └─ ... 39 методов MCP Apps
```

---

# ЧАСТЬ 18. Слабые места

## 18.1 `allow-scripts` + `allow-same-origin`

```js
r.sandbox = e?.allowPopups === false
  ? "allow-scripts allow-same-origin"
  : "allow-scripts allow-same-origin allow-popups allow-popups-to-escape-sandbox allow-forms";
```

**Известная проблема:** с `allow-same-origin` скрипт может
`parent.document` → снять `sandbox` → получить полный доступ.

**Проверить:** `runHTMLAsync` записывает в `document` **того же origin**
(`web-sandbox.oaiusercontent.com`). Значит iframe и родитель — **один
origin**. `allow-same-origin` + `allow-scripts` = **песочница не работает**.

**Что это даёт виджету:**
- `parent.document` — доступ к главному документу
- `parent.localStorage` — тот же origin
- `parent.fetch` — с токенами
- `parent.window.openai` — **тот же `hn()` API**, но с `isFirstParty` родителя

**Единственная защита** — `CSP` и `disableExternalNavigation`.
`apply-csp` не может запретить `parent`-доступ.

**Проверить:** в `main.js` есть ли проверка `parentSameOrigin`? В дампе
пользователя:
```json
"frame": {
  "isTop": false,
  "parentSameOrigin": true,
  "topSameOrigin": {"error": "SecurityError..."}
}
```

**`parentSameOrigin: true`** — значит iframe может читать `parent`.
`topSameOrigin` — ошибка, потому что `top` — `chatgpt.com` (другой origin).

**То есть:**
- iframe → parent (`web-sandbox.oaiusercontent.com`): ✅ полный доступ
- iframe → top (`chatgpt.com`): ❌ cross-origin

**parent — это `web-sandbox`, не ChatGPT.** Но в parent есть
`window.openai` (публичный API того же origin), `window.fetch`,
`localStorage`.

**Это не эскалация до ChatGPT**, но: виджет может читать `localStorage`
`web-sandbox.oaiusercontent.com` (там могут быть токены сессии), и
вызывать `hn()` API **без проверки user gesture** (потому что
`isFirstParty` определяется в parent-контексте).

**Проверить:** в `main.js` `Je` — это published API. Если виджет вызовет
`hn({isFirstParty: true, includePrivate: true})` из `parent`-контекста —
получит **все 31 метод**.

**Доступ к `hn`:** функция экспортируется как `hn as g` из `main.js`.
Но в iframe её нет — только `window.openai` (уже опубликованный `Je`).
Однако виджет может сделать:
```js
parent.document.querySelector('script[src*="main"]').__proto__  // нет
```
Или динамически импортировать:
```js
const m = await import("https://web-sandbox.oaiusercontent.com/assets/main-BR1bOQyH.js");
```
**CSP `script-src` включает `'self'`** → импорт с того же origin разрешён!

**Значит:** виджет может импортировать `main.js` и вызвать `hn({isFirstParty: true, includePrivate: true})`.

**Проверить на практике:** `import()` внутри iframe. `script-src` включает
`https://web-sandbox.oaiusercontent.com/assets/` (при `disableNetworkRequests`)
или `'self'`. В обоих случаях — разрешено.

## 18.2 `fe()` не учитывает порт

```js
if ((o.hostname !== e.hostname || o.protocol !== e.protocol || r?.forceHostOpen) && t.openExternal)
```

**`hostname` без порта.** `https://example.com:8080` = `https://example.com`.

**Не перехватит переход на другой порт того же хоста.**

## 18.3 `window.open` всегда `null`

```js
o.open = y => { ...; return null };
```

**Возврат `null`** — виджет не может получить ссылку на попап.
Попап **не создаётся** (только `openExternal` хоста).

**Из ваших логов:**
```json
{"name": "open-result", "value": {"opened": false, "reason": "window.open returned null; likely blocked or disallowed"}}
```

**Это ожидаемое поведение** — не блокировка браузера, а патч.

## 18.4 `HTMLAnchorElement.click` патч — неправильная логика

```js
m.length === 0 || m[0].defaultPrevented
  || this.target.toLowerCase() !== "_blank"
  || this.hasAttribute("download")
  || (this.protocol !== "http:" && this.protocol !== "https:")
  || fe(s, o.location, this.href, {forceHostOpen: !0})
```

**Логическая ошибка:** последнее условие
```js
(this.protocol !== "http:" && this.protocol !== "https:")
```
возвращает `true` для **не-http** (например `mailto:`) → `||` замыкается
на `true` → `fe()` **не вызывается**.

**То есть:** для не-http ссылок — ничего не делается.
Для http(s) с `_blank` без `download` — `fe()`.

**НО:** цель — проверить, что это http(s). Логика:
- `protocol === "http:"` → `(false && true)` = `false` → идём дальше → `fe()`
- `protocol === "https:"` → `(true && false)` = `false` → `fe()`
- `protocol === "mailto:"` → `(true && true)` = `true` → **стоп**

**Всё правильно.** `mailto:` не идёт в `fe()`.

**Но:** `protocol` для относительной ссылки = `location.protocol` = `https:`.
Значит относительная ссылка с `_blank` → `fe()`. И `fe()` вернёт `false`
(тот же hostname) → ничего не произойдёт, но `preventDefault` не
вызовется. Ссылка откроется в новом табе браузера.

**Проверить:** `fe` возвращает `false` при том же origin/path/search —
значит `_blank` на относительную ссылку **откроется в новом табе**.

## 18.5 `try/catch` в цикле `Xt()` — рассинхронизация

```js
for (const [l, a] of Object.entries(e))
  try {
    t.contentWindow.oai[l] = a;
    t.contentWindow.openai[l] = a;
    t.contentWindow.webplus[l] = a
  } catch (u) { n.environmentError(u, !1) }
```

**Если `oai[l] = a` бросит** (заморожен) — `openai[l]` и `webplus[l]`
не получат обновление. Глобалы разойдутся.

## 18.6 `sendInstrument` до проверки `isTrusted` в `u()`

```js
if (!y.isTrusted
    || (s.sendInstrument?.call(s, {type: "interaction", label: "click"}),
        y.defaultPrevented
        || !(y instanceof o.MouseEvent))) return;
```

**Разбор:** `!y.isTrusted || (...)` — если не trusted, весь блок
пропускается. `sendInstrument` вызывается только для trusted.

**НО:** `s.sendInstrument` вызывается **до** проверки `defaultPrevented`.
То есть метрика отправляется даже если клик отменён.

**Не проблема** — просто неточность.

## 18.7 `Ct()` с `wasFatalError: !P`

```js
T({type: bt.RUN_COMPLETE, duration: null, wasCancelled: P, wasFatalError: !P})
```

**При обычном завершении** (`Ct()`, `P=false`): `wasCancelled: false`,
`wasFatalError: true`.

**При отмене** (`Ct(true)`): `wasCancelled: true`, `wasFatalError: false`.

**То есть:** любое завершение **не по готовности** трактуется как
фатальная ошибка, даже если это просто «body опустел».

## 18.8 `data-fatal` не ловится после готовности

```js
l && d && !E && A?.disconnect();
```

После `Ge()` (готовность), if `expectReadySignal && isFirstParty && !measureWidth`
→ observer отключён.

**`data-fatal` больше не ловится.** Если React-приложение упадёт
**после** рендера — не будет `Ct()`.

## 18.9 `Mt` — параметры из URL

```js
Mt = new URLSearchParams(window.location.search);
```

**`unsafeSkipTargetOriginCheck=true` в URL** → `targetOrigin: "*"`.

**Требует контроля над URL загрузки.** Если хост формирует URL из
пользовательских данных — возможна инъекция.

## 18.10 Escape-слушатель на `window`, не на iframe

```js
window.addEventListener("keydown", e => {
  e.key === "Escape" && !e.defaultPrevented && ee?.notifyEscapeKey?.()
}, {passive: !0})
```

**`window` главного документа.** Если пользователь нажмёт Escape в
**любом** месте `web-sandbox` — уйдёт `notifyEscapeKey`.

**`ee`** — последний `ro(s)`, то есть API **последнего** iframe.
При нескольких виджетах — только последний.

## 18.11 `ye` — глобальный «последний target»

```js
let ye = null;
const E = y => { ye = y.target };
```

**Один на весь модуль.** При нескольких iframe — перезаписывается.
`In()` проверяет `ye instanceof t.HTMLElement` где `t = nt().contentWindow`
— то есть `ye` должен принадлежать **текущему** iframe. Если последний
`pointerdown` был в другом iframe — `In()` вернёт `null`.

---

# ЧАСТЬ 19. Итог

## 19.1 Ключевые числа

| Что | Значение |
|---|---|
| Размер | 89 639 байт |
| Экспортов | 22 |
| Типов событий | 6 |
| Статусов | 3 |
| Скрытых методов (`Jr`) | 31 |
| Методов с жестом (`ao`) | 8 |
| Доверенных CDN | 6 |
| Параметров `runHTMLAsync` | 21 |
| Лимит bindings+callbacks | 32 |
| Лимит глубины | 8 |
| Лимит длины строки | 4096 |
| Лимит JSON | 16 384 |
| Таймаут `prepareDocument` | 10 с |
| Таймаут MCP init | 30 с |
| Throttle скролла | 1.5 с |

## 19.2 Главное

**1. `allow-scripts` + `allow-same-origin` = песочница неэффективна.**
iframe того же origin, что и родитель. Виджет может читать `parent`.

**2. Виджет может импортировать `main.js` и вызвать `hn({isFirstParty:true, includePrivate:true})`.**
CSP `script-src` включает `'self'` или `https://origin/assets/` — оба
разрешают импорт. Это даёт **все 31 приватный метод**.

**3. `window.open` пропатчен — всегда `null`.**
Ваши логи `"window.open returned null"` — это ожидаемо.

**4. `fe()` не учитывает порт — переход на другой порт того же хоста не перехватывается.**

**5. `try/catch` в цикле `Xt()` — при заморозке `window.openai` глобалы рассинхронизируются.**

**6. `data-fatal` не ловится после готовности** (при first-party + `expectReadySignal` + не меряем ширину).

**7. Escape-слушатель на `window` главного документа — при нескольких виджетах только последний получает.**

**8. `ye` — глобальный «последний target», при нескольких iframe перезаписывается.**

**9. `Kr()` клонирует `bindings` через `structuredClone` — функции не пройдут.**

**10. `ze()` блокирует классы через сравнение `constructor.toString()` с `Object`.**

**11. `_meta` → `meta` в `callTool`** — мост между MCP-спецификацией и ChatGPT.

**12. Три глобала (`oai`, `openai`, `webplus`) — один объект.**
