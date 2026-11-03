# `adapter-CQcBVlDI.js` — полный разбор (MCP Apps bridge)

**Размер:** 119 651 байт
**URL:** `https://web-sandbox.oaiusercontent.com/assets/adapter-CQcBVlDI.js`
**Роль:** мост между MCP Apps-виджетом (в iframe) и хостом ChatGPT.
Реализует протокол **MCP Apps `2026-01-26`**.

---

# ЧАСТЬ 1. Структура файла

## 1.1 Из чего состоит

| Байт | Что | Размер |
|---|---|---|
| 0 – 180 | преамбула (`So`, `R` — defineProperty) | 180 |
| 180 – 57 500 | **zod** (схемы валидации) | ~57 КБ |
| 57 500 – 88 000 | MCP SDK-схемы (tools, resources, prompts, tasks) | ~30 КБ |
| 88 000 – 106 000 | `AppBridge` (класс `ku`) и `App` (класс `ju`) | ~18 КБ |
| 106 000 – 114 700 | event-слоты, хендлеры | ~8.7 КБ |
| 114 769 – 119 651 | **`initMcpAppsAdapter`** — 4.9 КБ | 4.9 КБ |

**97% — библиотека.** Реальная логика Skybridge — последние 5 КБ.

## 1.2 Импорты

```js
import {n as dn, d as Ro} from "./main-BR1bOQyH.js";
```

| Локальное | Экспорт | Реальное имя | Что |
|---|---|---|---|
| `dn` | `n` | `pi` | `normalizeToolResult` — нормализация `tools/call` |
| `Ro` | `d` | `X` | `hn({isFirstParty: true, includePrivate: true})` |

**`Ro` — приватный API.** Именно через него доступен `callMcp`.

## 1.3 Экспорты

```js
export {
  el as initMcpAppsAdapter,
  Qu as notifyMcpAppsHostContext,
  Yu as notifyMcpAppsMcpNotification,
  Gu as notifyMcpAppsToolCancelled,
  Bu as notifyMcpAppsToolInput,
  Ku as notifyMcpAppsToolResult,
  Xu as requestMcpAppsResourceTeardown
};
```

| Экспорт | Что делает |
|---|---|
| `initMcpAppsAdapter` | создать мост (главное) |
| `notifyMcpAppsHostContext` | обновить hostContext |
| `notifyMcpAppsMcpNotification` | отправить MCP-уведомление |
| `notifyMcpAppsToolCancelled` | инструмент отменён |
| `notifyMcpAppsToolInput` | входные данные инструмента |
| `notifyMcpAppsToolResult` | результат инструмента |
| `requestMcpAppsResourceTeardown` | запрос на удаление ресурса |

---

# ЧАСТЬ 2. Глобальное состояние

```js
let M = null,     // текущий App-экземпляр
    Re,           // sandboxLogger
    ne = null,    // hostContext
    _e = !1,      // initialized
    ze = [];      // очередь до ui/notifications/initialized
```

**Эти переменные — на уровне модуля.** Один bridge на всё.

## 2.1 `He()` — очередь

```js
const He = e => {
  if (M) {
    if (!_e) { ze.push(e); return }   // ещё не инициализирован → в очередь
    e()                                // инициализирован → выполнить
  }
};
```

**Три состояния:**
1. `M === null` — bridge не создан → **молча отбросить**
2. `M !== null && !_e` — создан, но не инициализирован → **в очередь**
3. `M !== null && _e` — работает → **выполнить сразу**

## 2.2 `Vu()` — разгрузка очереди

```js
const Vu = () => {
  if (!ze.length) return;
  const e = ze;
  ze = [];              // очистить ДО выполнения
  for (const t of e) t();
};
```

**`ze = []` до цикла** — если во время выполнения добавят ещё, они пойдут
в новую очередь (не бесконечный цикл).

---

# ЧАСТЬ 3. Публичные обёртки

```js
const Bu = e => { He(() => { M?.sendToolInput(e) }) };
const Ku = e => { const t = dn(e); t != null && He(() => { M?.sendToolResult(t) }) };
const Gu = e => { He(() => { M?.sendToolCancelled(e) }) };
const Qu = e => { ne = {...ne ?? {}, ...e}, He(() => { M?.setHostContext(ne ?? e) }) };
const Yu = e => { He(() => { M?.transport?.send({...e, jsonrpc: "2.0"}) }) };
const Xu = ...   // requestMcpAppsResourceTeardown
```

**`Ku`** — `dn(e)` (нормализация). Если `null` — **не отправлять**.

**`Qu`** — мерж в `ne` **до** `He()`. То есть даже если bridge не создан,
`ne` обновится.

**`Yu`** — сырой `transport.send`. `jsonrpc: "2.0"` добавляется принудительно.
Если `e` уже содержит `jsonrpc` — перезапишется на `"2.0"`.

---

# ЧАСТЬ 4. `initMcpAppsAdapter` — главное

## 4.1 Полный код

```js
el = ({
  onInitialized: e,
  onInitializationError: t,
  viewWindow: n,
  forwardedRequestMethods: o = [],
  hostContext: r,
  hostCapabilities: s,
  hostInfo: i,
  getSetWidgetState: c,
  sandboxLogger: a
}) => {
  const p = Ro(),                    // приватный API
        f = new Set(o);              // форвардимые методы

  ne = r;                            // глобальный hostContext

  const g = new ju(                  // App-экземпляр
    null,
    {name: i?.name ?? "chatgpt", version: i?.version ?? "0.0.1"},
    s,
    {hostContext: r}
  );

  M = g;                             // глобальная ссылка
  Re = a;
  _e = !1;

  // ... (см. ниже)
};
```

## 4.2 Параметры

| Параметр | Что |
|---|---|
| `onInitialized` | колбэк при `ui/notifications/initialized` |
| `onInitializationError` | колбэк при ошибке `connect()` |
| `viewWindow` | `iframe.contentWindow` |
| `forwardedRequestMethods` | методы, которые надо проксировать в `callMcp` |
| `hostContext` | контекст хоста (тема, локаль, стили) |
| `hostCapabilities` | возможности хоста |
| `hostInfo` | `{name, version}` хоста |
| `getSetWidgetState` | ссылка на `setWidgetState` |
| `sandboxLogger` | логгер |

## 4.3 `Ro()` — приватный API

```js
const p = Ro();    // hn({isFirstParty: true, includePrivate: true})
```

**Что доступно:** все 31 метод из `Jr` + `prepareWidgetDocument` +
8 из `ao` + прочие.

**Используются в адаптере:**

| Метод | Где |
|---|---|
| `p.callMcp` | `b()` — прокси MCP |
| `p.openExternal` | `g.onopenlink` |
| `p.sendFollowUpMessage` | `ui/message` |
| `p.requestClose` | `g.onrequestteardown` |
| `p.requestDisplayMode` | `g.onrequestdisplaymode` |
| `p.notifyIntrinsicWidth` | `g.onsizechange` |
| `p.notifyIntrinsicHeight` | `g.onsizechange` |
| `p.initializeMcpApp` | перехват `ui/initialize` |

## 4.4 `v()` — ошибка «нет MCP-прокси»

```js
const v = () => {
  const m = new Error("MCP proxy not enabled");
  throw m.code = -32e3, m;             // -32000
};
```

**`-32e3` = `-32000`.** JSON-RPC-совместимый код ошибки.

**Когда:** `p.callMcp` отсутствует (не first-party или MCP не настроен).

## 4.5 `b()` — прокси в `callMcp`

```js
const b = async (m, y) => {
  if (!p.callMcp) return v();

  try {
    const P = await p.callMcp({method: m, params: y});
    return m === "tools/call" ? dn(P) ?? P : P;
  } catch (P) {
    const q = P != null && typeof P == "object" ? P : {},
          ie = "message" in q && typeof q.message == "string"
               ? q.message : "MCP proxy request failed",
          me = "code" in q && typeof q.code === "number" ? q.code : -32e3;

    throw Object.assign(new Error(ie, {cause: P}), {
      code: me,
      ..."data" in q ? {data: q.data} : {}
    });
  }
};
```

### Разбор

**`if (!p.callMcp) return v()`** — нет прокси → `-32000`.

**`p.callMcp({method, params})`** — вызов хоста. Хост делает реальный
MCP-запрос к серверу.

**`m === "tools/call" ? dn(P) ?? P : P`** — для `tools/call` результат
нормализуется через `dn` (см. 4.6). Если нормализация вернула `null` —
сырой результат.

**Обработка ошибки:**
- `message` → текст ошибки
- `code` → числовой код, дефолт `-32000`
- `data` → пробрасывается, если есть
- `cause: P` → исходная ошибка

**`Object.assign(new Error(ie, {cause: P}), {...})`** — `Error` с `cause`
(ES2022), плюс `code` и `data`.

## 4.6 `dn()` — нормализация `tools/call`

Из `main-BR1bOQyH.js` (`pi`):

```js
pi = t => {
  if (typeof t != "object" || t === null) return null;

  const e = t;
  if (!Array.isArray(e.content) || !e.content.every(Zr)) return null;

  const n = l => {
    if (Array.isArray(l)) return l.map(a => n(a));
    if (typeof l == "object" && l !== null) {
      const a = {};
      for (const [u, h] of Object.entries(l))
        h !== null && (a[u] = u === "_meta" ? h : n(h));
      return a
    }
    return l
  };

  const r = n(e.content),
        o = be(e.structuredContent) ? e.structuredContent : void 0;

  if (o === void 0 && e.structuredContent !== void 0) {
    const l = JSON.stringify(e.structuredContent);
    l !== void 0 && r.push({type: "text", text: l})
  }

  const s = typeof e.isError == "boolean" ? e.isError : void 0,
        i = be(e._meta) ? e._meta : void 0;

  return {
    content: r,
    ...o !== void 0 ? {structuredContent: o} : {},
    ...s !== void 0 ? {isError: s} : {},
    ...i !== void 0 ? {_meta: i} : {}
  }
};
```

### Что делает

1. **Проверка:** `content` — массив валидных блоков (`Zr`)
2. **Рекурсивная очистка:** убрать все `null`-значения, кроме `_meta`
3. **`structuredContent`:**
   - если объект — оставить
   - если не объект, но задан — сериализовать в JSON и добавить как
     `{type: "text", text: JSON.stringify(...)}` в `content`
4. **`isError`** — только если boolean
5. **`_meta`** — только если объект

**Зачем:** MCP `tools/call` может вернуть `structuredContent` любого типа.
MCP Apps-виджет ожидает либо объект, либо текст. Невалидный JSON
конвертируется в текстовый блок.

## 4.7 `k()` — ошибка валидации

```js
const k = m => {
  const y = new Error(m);
  throw y.code = -32602, y;
};
```

**`-32602` = Invalid params** (стандартный JSON-RPC).

## 4.8 `Z()` — валидация displayMode

```js
const Z = m => m === "inline" || m === "fullscreen" || m === "pip" ? m : "inline";
```

**Три допустимых режима.** Всё остальное → `"inline"`.

## 4.9 `se()` — собрать текст

```js
const se = m => {
  const y = [];
  for (const P of m)
    if (P.type === "text" && typeof P.text === "string") {
      const q = P.text.trim();
      q && y.push(q);
    }
  return y.join("\n")
};
```

**Только текстовые блоки.** Изображения, аудио — отбрасываются.

**`trim()` + проверка `q`** — пустые строки тоже отбрасываются.

---

# ЧАСТЬ 5. Хендлеры `App`

## 5.1 `g.onopenlink` — `ui/open-link`

```js
g.onopenlink = async ({url: m}) =>
  p.openExternal
    ? (p.openExternal({href: m}), {})
    : (a?.log("MCP Apps open-link ignored (host missing openExternal)"), {isError: !0});
```

**Если `openExternal` есть** → открыть URL, вернуть `{}`.

**Если нет** → лог + `{isError: true}`.

**Важно:** `p.openExternal({href: m})` — **без проверки схемы**. Если
виджет попросит `javascript:alert(1)` — хост получит это. Хост должен
сам валидировать.

## 5.2 `ui/message` — отправка сообщения в чат

```js
g.setRequestHandler(Uu, async ({params: m}) => {
  const y = Lu.safeParse(m);
  if (!y.success) return k("Invalid MCP message params");

  // (1) форвардим?
  if (f.has("ui/message")) return await b("ui/message", m);

  // (2) служебный метаданный openai/message?
  if (y.data._meta?.["openai/message"] != null) return {isError: !0};

  // (3) роль должна быть "user"
  const {role: P, content: q} = y.data;
  if (P !== "user") return a?.log("MCP Apps message ignored (unsupported role)", P), {isError: !0};

  // (4) sendFollowUpMessage есть?
  if (!p.sendFollowUpMessage) return a?.log("MCP Apps message ignored (host missing sendFollowUpMessage)"), {isError: !0};

  // (5) собрать текст
  const ie = se(q);
  return ie
    ? (p.sendFollowUpMessage({prompt: ie, isVisible: !0}), {})
    : (a?.log("MCP Apps message ignored (empty text content)"), {isError: !0});
});
```

### Пять последовательных проверок

**1. Валидация** — zod-схема `Lu` (MCP message + `_meta` из `F`).

**2. Форвардинг** — если `"ui/message"` в `forwardedRequestMethods` —
проксируется в `callMcp`. То есть хост сам решит.

**3. `_meta["openai/message"]`** — если задан, **отклонить**.
Это служебный маркер OpenAI — виджет не должен его подделывать.

**4. `role === "user"`** — только user-сообщения. `assistant` → отклонить.

**5. `sendFollowUpMessage`** — если хост его не дал → отклонить.

**6. Текст не пустой** — `se(q)` собирает только `text`-блоки.

**Результат:** `p.sendFollowUpMessage({prompt: ie, isVisible: true})`.

**`isVisible: true`** — сообщение показывается в чате (не скрытое).

## 5.3 `ui/update-model-context`

```js
g.setRequestHandler(Fu, async ({params: m}) => {
  if (!Ot.shape.params.safeParse(m).success)
    return k("Invalid MCP model context params");

  if (f.has("ui/update-model-context"))
    return await b("ui/update-model-context", m);

  const P = c?.();                      // getSetWidgetState()
  if (!P) throw Object.assign(new Error("Method not found"), {code: -32601});

  return await P(m, {waitForPersistence: !0}), {};
});
```

**Схема `Ot`:**
```js
{
  method: "ui/update-model-context",
  params: {
    content: ContentBlock[] | undefined,
    structuredContent: Record<string, unknown> | undefined
  }
}
```

**Если нет `getSetWidgetState`** → `-32601 Method not found`.

**`waitForPersistence: true`** — ждать, пока состояние сохранится на хосте
перед ответом. Виджет узнает, что контекст точно применён.

## 5.4 `ui/request-teardown` → `requestClose`

```js
g.onrequestteardown = () => p.requestClose?.();
```

**Возвращает результат `requestClose()`** — обычно промис.

## 5.5 `ui/request-display-mode`

```js
g.onrequestdisplaymode = async ({mode: m}) => {
  const y = await p.requestDisplayMode?.({mode: m}),
        P = y && typeof y === "object" && "mode" in y
            ? y.mode
            : (ne?.displayMode ?? r.displayMode ?? m);

  return {mode: Z(P)};
};
```

**Приоритет:**
1. `y.mode` — если хост вернул объект с `mode`
2. `ne.displayMode` — глобальный hostContext
3. `r.displayMode` — исходный hostContext
4. `m` — запрошенный

**`Z(P)`** — валидация: `inline`/`fullscreen`/`pip`, иначе `inline`.

**Зачем fallback:** если хост не вернул ничего — считаем, что режим
применился как запрошен.

## 5.6 `ui/notifications/size-changed`

```js
g.onsizechange = ({width: m, height: y}) => {
  m != null && p.notifyIntrinsicWidth?.(m);
  y != null && p.notifyIntrinsicHeight?.(y);
};
```

**`notifyIntrinsicWidth`** и **`notifyIntrinsicHeight`** — приватные методы.
Вызываются только если не `null`/`undefined`.

## 5.7 Логирование

```js
g.onloggingmessage = ({level: m, logger: y, data: P}) => {
  if (!a) return;
  const q = y ? `[mcp-app:${y}]` : "[mcp-app]";
  a.log(`${q} ${m}`, P);
};
```

**Формат:** `[mcp-app] level data` или `[mcp-app:loggerName] level data`.

**`a.log(...)`** → `sandboxLogger.log` → `forwardConsoleLogs` → хост.

**Но:** в `runWidgetCode` при MCP Apps `forwardConsoleLogs: false`
(см. документ #10). Значит **логи MCP-приложений не доходят до хоста**
в этом режиме. Идут только в консоль браузера.

## 5.8 `ui/notifications/initialized`

```js
g.oninitialized = () => {
  M === g && (_e = !0, Vu(), e?.());     // разгрузить очередь + колбэк
};
```

**`M === g`** — проверка, что это **текущий** экземпляр. Если bridge
пересоздали, старый `oninitialized` не должен срабатывать.

**`_e = true`** → очередь `ze` разгружается через `Vu()`.

---

# ЧАСТЬ 6. Tool и Resource хендлеры

## 6.1 `tools/call`

```js
const te = async m => {
  m?.name || k("Missing tool name");
  const y = {...m, arguments: m.arguments ?? {}};
  return await b("tools/call", y);
};
g.oncalltool = te;
```

**Обязательное поле `name`.** Иначе `-32602`.

**`arguments` по умолчанию `{}`** — если не переданы.

**`b("tools/call", y)`** → `p.callMcp({method: "tools/call", params: y})`
→ `dn(результат)`.

## 6.2 `resources/list`

```js
const re = async m => await b("resources/list", m && typeof m === "object" ? m : void 0);
g.onlistresources = re;
```

**Параметры опциональны.** Если не объект — `undefined`.

## 6.3 `resources/read`

```js
const L = async m => (m?.uri || k("Missing resource uri"), await b("resources/read", m));
g.onreadresource = L;
```

**Обязательный `uri`.**

## 6.4 `resources/templates/list`, `prompts/list`

```js
const J = async m => await b("resources/templates/list", m && typeof m === "object" ? m : void 0);
g.onlistresourcetemplates = J;

const oe = async m => await b("prompts/list", m && typeof m === "object" ? m : void 0);
g.onlistprompts = oe;
```

## 6.5 `ping`

```js
const Oe = async () => await b("ping");
g.onping = Oe;
```

**Пустой вызов.** Проверка живости хоста.

## 6.6 `fallbackRequestHandler`

```js
const D = async m => {
  if (f.has(m.method)) return await b(m.method, m.params);
  if (m.method === "tools/list") return await b("tools/list", m.params);

  const y = new Error("Method not found");
  throw y.code = -32601, y;
};
g.fallbackRequestHandler = D;
```

**Логика:**
1. Если метод в `forwardedRequestMethods` → проксировать
2. Если `tools/list` → проксировать **(без проверки `f`)**
3. Иначе → `-32601 Method not found`

**`tools/list` — особый случай:** всегда разрешён, даже если не в списке
форвардинга.

---

# ЧАСТЬ 7. Транспорт и перехват `ui/initialize`

```js
const Q = new class extends Hu {
  async start() {
    const m = this.onmessage;

    this.onmessage = async (y, P) => {
      const q = ko.safeParse(y);

      if (!q.success || !("id" in y) || !p.initializeMcpApp) {
        m?.(y, P);          // ← пропустить в стандартный обработчик
        return
      }

      try {
        const ie = await p.initializeMcpApp(q.data.params.appCapabilities);
        if (M !== g) return;

        ne = {...ne, ...ie};
        g.setHostContext(ne);
        m?.(y, P);
      } catch (ie) {
        if (a?.error("MCP Apps initialization failed", ie), M !== g) return;

        this.send({
          jsonrpc: "2.0",
          id: y.id,
          error: {code: -32603, message: "MCP app initialization failed"}
        }).catch(me => { a?.error("MCP Apps initialization reply failed", me) })
      }
    };

    await super.start()
  }

  async send(m) {
    !_e && "method" in m && m.method === "ui/notifications/host-context-changed"
      || await super.send(m)
  }
}(n, n);
```

## 7.1 Перехват `ui/initialize`

**`ko`** — схема:
```js
{
  method: "ui/initialize",
  params: {
    appInfo: {name, version},
    appCapabilities: {...},
    protocolVersion: string
  }
}
```

**Условие перехвата:** валидная схема + есть `id` + есть `p.initializeMcpApp`.

**Почему нужен `id`:** `ui/initialize` — запрос, на него ждут ответ.
Уведомления (`notifications/*`) идут без `id`.

**Что делает:**
1. `p.initializeMcpApp(q.data.params.appCapabilities)` — **хост** инициализирует
   MCP-приложение с возможностями виджета
2. Получает ответ `ie` (`{hostContext, ...}`)
3. **`M !== g` → выход** (bridge пересоздали)
4. `ne = {...ne, ...ie}` — смержить в hostContext
5. `g.setHostContext(ne)` — обновить у App
6. `m?.(y, P)` — **пропустить в стандартный обработчик** (AppBridge ответит сам)

**При ошибке:**
- `-32603 Internal error`
- `message: "MCP app initialization failed"`
- Отправить через `this.send(...)`

**Ключевое:** перехват **не заменяет** обработчик, а **дополняет**.
`super.onmessage` всё равно вызовется — AppBridge обработает `ui/initialize`
стандартно (`_oninitialize`).

## 7.2 `send` — блокировка host-context-changed

```js
async send(m) {
  !_e && "method" in m && m.method === "ui/notifications/host-context-changed"
    || await super.send(m)
}
```

**Разбор приоритета:**

```js
(!_e && "method" in m && m.method === "ui/notifications/host-context-changed") || await super.send(m)
```

**Если** `!_e` (не инициализирован) **и** это `host-context-changed`
→ левая часть `true`, `await super.send(m)` **не выполнится**.
**Иначе** → `super.send(m)`.

**То есть:** `ui/notifications/host-context-changed` блокируется до
`ui/notifications/initialized`.

**Зачем:** виджет ещё не готов принимать контекст. Если отправить раньше —
потеряется или вызовет ошибку.

**Проблема:** после `initialized` отправленный ранее `host-context-changed`
**не будет дослан автоматически**. Нужно явно вызвать
`notifyMcpAppsHostContext` ещё раз.

## 7.3 `Q.onerror`

```js
Q.onerror = m => { a?.error("MCP Apps transport error", m) };
```

Ошибки транспорта → в лог.

## 7.4 `g.connect(Q)`

```js
g.connect(Q).catch(m => {
  M === g && (t ? t(m) : a?.environmentError(m))
});
```

**При ошибке:**
- если `onInitializationError` задан → вызвать его
- иначе → `environmentError` в лог

**`M === g`** — только для текущего экземпляра.

---

# ЧАСТЬ 8. `destroy`

```js
return {
  destroy: () => {
    M === g && (M = null, Re = void 0, ne = null, _e = !1, ze = []);
    Q.close()
  }
};
```

**Если это текущий экземпляр** — сбросить всё глобальное состояние.

**`Q.close()`** — закрыть транспорт always (даже если `M !== g`).

**Проблема:** если создали два bridge подряд, и первый `destroy` вызывается
после второго `init` — `M !== g` для первого, но `Q.close()` всё равно
закроет **свой** транспорт. Правильно.

---

# ЧАСТЬ 9. Протокол — полный список

## 9.1 Версия

```js
var bo = "2026-01-26",
    Ou = [bo];       // поддерживаемые версии
```

**Только `2026-01-26`.**

**`_oninitialize`:**
```js
{
  protocolVersion: Ou.includes(o) ? o : bo,
  hostCapabilities: this.getCapabilities(),
  hostInfo: this._hostInfo,
  hostContext: this._hostContext
}
```

**Если клиент запросил другую версию** — вернётся `bo` (наша). Клиент
должен согласиться или разорвать.

## 9.2 17 UI-методов

### Запросы (нужен ответ)

| Метод | Направление | Обработка |
|---|---|---|
| `ui/initialize` | view → host | `initializeMcpApp` + AppBridge |
| `ui/message` | view → host | `sendFollowUpMessage` или `callMcp` |
| `ui/update-model-context` | view → host | `setWidgetState({waitForPersistence})` |
| `ui/open-link` | view → host | `openExternal({href})` |
| `ui/request-display-mode` | view → host | `requestDisplayMode({mode})` |
| `ui/download-file` | view → host | `_ondownloadfile` |
| `ui/resource-teardown` | view → host | `teardownResource` |

### Уведомления (без ответа)

| Метод | Направление |
|---|---|
| `ui/notifications/initialized` | view → host |
| `ui/notifications/host-context-changed` | host → view |
| `ui/notifications/size-changed` | view → host |
| `ui/notifications/tool-input` | host → view |
| `ui/notifications/tool-input-partial` | host → view |
| `ui/notifications/tool-result` | host → view |
| `ui/notifications/tool-cancelled` | host → view |
| `ui/notifications/sandbox-proxy-ready` | host → view |
| `ui/notifications/sandbox-resource-ready` | host → view |
| `ui/notifications/request-teardown` | host → view |

## 9.3 22 не-UI метода

### Через `callMcp`

| Метод | Где |
|---|---|
| `tools/call` | `oncalltool` |
| `tools/list` | `fallbackRequestHandler` |
| `resources/list` | `onlistresources` |
| `resources/read` | `onreadresource` |
| `resources/templates/list` | `onlistresourcetemplates` |
| `prompts/list` | `onlistprompts` |
| `ping` | `onping` |

### Только через `forwardedRequestMethods`

| Метод |
|---|
| `resources/subscribe` |
| `resources/unsubscribe` |
| `prompts/get` |
| `roots/list` |
| `elicitation/create` |
| `notifications/*` (9 типов) |

**Не форварднутые** → `-32601 Method not found`.

## 9.4 Схема CSP для sandbox (`Nt`)

```js
Nt = {
  connectDomains: string[] | undefined,
  resourceDomains: string[] | undefined,
  frameDomains: string[] | undefined,
  baseUriDomains: string[] | undefined
}
```

**Описания из схемы:**

> **connectDomains** — *Origins for network requests (fetch/XHR/WebSocket).
> Maps to CSP `connect-src`. Empty or omitted → no network connections
> (secure default)*
>
> **resourceDomains** — *Origins for static resources (images, scripts,
> stylesheets, fonts, media). Maps to CSP `img-src`, `script-src`,
> `style-src`, `font-src`, `media-src`. Wildcard subdomains supported:
> `https://*.example.com`. Empty or omitted → no network resources
> (secure default)*
>
> **frameDomains** — *Origins for nested iframes. Maps to CSP `frame-src`.
> Empty or omitted → no nested iframes allowed (`frame-src 'none'`)*
>
> **baseUriDomains** — *Allowed base URIs. Maps to CSP `base-uri`.
> Empty or omitted → only same origin (`base-uri 'self'`)*

**Это ровно те поля, что принимает `apply-csp`!** То есть MCP-приложение
объявляет свои домены, а хост применяет их через `runWidgetCode(csp: ...)`.

## 9.5 Политика разрешений (`At`)

```js
At = {
  camera: {} | undefined,          // Permission Policy: camera
  microphone: {} | undefined,      // microphone
  geolocation: {} | undefined,     // geolocation
  clipboardWrite: {} | undefined   // clipboard-write
}
```

**MCP-приложение может запросить:**
- камеру
- микрофон
- геолокацию
- запись в буфер обмена

**Это Permissions Policy в iframe** (атрибут `allow`).

---

# ЧАСТЬ 10. Поток инициализации

```
1. runWidgetCode({mcpApps: {...}})
     ├─ requiresInitialization? → Promise.withResolvers()
     └─ runHTMLAsync({appInitialization, onIframeReady})

2. iframe загружен → onIframeReady({viewWindow, document})
     └─ initMcpAppsAdapter({...})

3. В адаптере:
     ├─ p = Ro()                  ← приватный API
     ├─ g = new App(null, hostInfo, hostCapabilities, {hostContext})
     ├─ M = g                     ← глобальная ссылка
     ├─ Q = new Transport(viewWindow, viewWindow)
     ├─ g.connect(Q)
     └─ вернуть {destroy}

4. Виджет отправляет ui/initialize
     ├─ перехвачено: safeParse(ko)
     ├─ p.initializeMcpApp(appCapabilities)   ← ХОСТ
     ├─ ne = {...ne, ...результат}
     ├─ g.setHostContext(ne)
     └─ передать в супер-обработчик → _oninitialize()

5. _oninitialize отвечает:
     {protocolVersion: "2026-01-26", hostCapabilities, hostInfo, hostContext}

6. Виджет отправляет ui/notifications/initialized
     ├─ g.oninitialized()
     ├─ _e = true
     ├─ Vu() — разгрузить очередь ze
     └─ onInitialized() → s.resolve()

7. runHTMLAsync: appInitialization.promise резолвится

8. Теперь работают:
     ├─ notifyMcpAppsToolInput / ToolResult / ToolCancelled
     ├─ notifyMcpAppsHostContext
     ├─ ui/notifications/host-context-changed
     └─ tools/call, resources/*, prompts/*
```

---

# ЧАСТЬ 11. Слабые места

## 11.1 `openExternal` без проверки схемы

```js
g.onopenlink = async ({url: m}) => p.openExternal ? (p.openExternal({href: m}), {}) : ...
```

**`url` не валидируется.** Виджет может отправить `javascript:alert(1)`,
`file:///etc/passwd`, `intent://`, `tel:` и т.д.

**Ответственность на хосте** — он должен сам проверять. Но схема
`{url: string}` в zod не ограничивает протокол.

## 11.2 `ne = {...ne, ...e}` — мутация глобального контекста

```js
const Qu = e => { ne = {...ne ?? {}, ...e}, He(() => { M?.setHostContext(ne ?? e) }) };
```

**`ne` обновляется даже если bridge не создан.** При следующем `initMcpAppsAdapter`
```js
ne = r;    // ← ПЕРЕЗАПИСЫВАЕТСЯ
```

**То есть накопленный `ne` теряется при новом `init`.** Если хост
отправлял `hostContext` до создания bridge — данные пропадут.

## 11.3 `host-context-changed` до `initialized` — потерян навсегда

```js
async send(m) {
  !_e && "method" in m && m.method === "ui/notifications/host-context-changed"
    || await super.send(m)
}
```

**Блокируется**, но **не ставится в очередь**. После `initialized`
сообщение не досылается.

**Компенсация:** `ne` уже обновлён в `Qu()`, и при `ui/initialize`
`g.setHostContext(ne)` отправит полный контекст. **Но** если
`host-context-changed` пришёл **после** `ui/initialize` но **до**
`initialized` — потеряется.

## 11.4 `ze` очищается без проверки на ошибки

```js
const Vu = () => {
  if (!ze.length) return;
  const e = ze;
  ze = [];
  for (const t of e) t();
};
```

**Если `t()` бросит** — цикл прервётся, остальные в очереди **потеряются**
(хотя `ze` уже очищен).

## 11.5 `_e` не сбрасывается при ошибке `ui/initialize`

При `catch`:
```js
if (a?.error("MCP Apps initialization failed", ie), M !== g) return;
this.send({jsonrpc: "2.0", id: y.id, error: {code: -32603, ...}})
```

**`_e` остаётся `false`.** Очередь `ze` **никогда не разгрузится**.
Виджет не получит `tool-input`/`tool-result`.

## 11.6 `M === g` вместо `M === this`

```js
if (M !== g) return;
```

Везде сравнивается с **замыканием `g`**, а не с `this`. Это правильно
для методов адаптера, но в `Q.onmessage` сравнение с `g` — тоже норм.

**Проблема:** если создать **второй** адаптер, первый перестаёт работать
(`M` указывает на второй). Но `destroy()` первого проверит `M === g`
= `false` и **не сбросит ничего**. Хотя должен был.

## 11.7 `-32601 Method not found` для не-форварднутых методов

```js
const D = async m => {
  if (f.has(m.method)) return await b(m.method, m.params);
  if (m.method === "tools/list") return await b("tools/list", m.params);
  const y = new Error("Method not found");
  throw y.code = -32601, y;
};
```

**`elicitation/create`, `roots/list`, `resources/subscribe`, `prompts/get`**
— только через `forwardedRequestMethods`.

**По умолчанию `o = []`** → ни один из них не работает.

**То есть MCP-приложение не может:**
- запросить дополнительные данные у пользователя (`elicitation/create`)
- получить список корней (`roots/list`)
- подписаться на обновления (`resources/subscribe`)
- получить готовый промпт (`prompts/get`)

**Без явного разрешения хоста.**

## 11.8 `tools/list` в обход `forwardedRequestMethods`

```js
if (m.method === "tools/list") return await b("tools/list", m.params);
```

**Всегда разрешён.** MCP-приложение может перечислить все инструменты
сервера, даже если хост не форвардит.

**Это может быть утечкой:** список инструментов раскрывает возможности.

## 11.9 Логи MCP не доходят при `forwardConsoleLogs: false`

`g.onloggingmessage` → `a.log(...)` → `sandboxLogger.log`. Но в
`runWidgetCode` при MCP Apps `forwardConsoleLogs: !e` = `false`.

**Логи теряются.** Только консоль браузера.

## 11.10 `protocolVersion` — только одна версия

```js
Ou = [bo];   // ["2026-01-26"]
```

**Нет обратной совместимости.** Клиент со старой версией получит нашу,
и должен сам решить.

## 11.11 `ko.safeParse` без `id` → пропуск

```js
if (!q.success || !("id" in y) || !p.initializeMcpApp) { m?.(y, P); return }
```

**Если `ui/initialize` пришёл без `id`** (нарушение протокола) —
пропускается в стандартный обработчик. AppBridge обработает и, вероятно,
ответит ошибкой.

## 11.12 `Q.close()` всегда, даже при `M !== g`

```js
return { destroy: () => { M === g && (...); Q.close() } }
```

**Правильно** — транспорт принадлежит этому экземпляру.

---

# ЧАСТЬ 12. Итог

## 12.1 Ключевые числа

| Что | Значение |
|---|---|
| Размер файла | 119 651 байт |
| zod-часть | ~57 КБ (48%) |
| Реальная логика | 4.9 КБ (4%) |
| Версия протокола | `2026-01-26` |
| UI-методов | 17 |
| Не-UI методов | 22 |
| Всего методов | 39 |
| Глобальных переменных | 5 (`M`, `Re`, `ne`, `_e`, `ze`) |
| Приватных API-методов используется | 8 |

## 12.2 Главное

**1. `MCP Apps` = MCP-протокол + UI-расширения (`ui/*`).**
17 UI-методов для взаимодействия с ChatGPT, 22 — стандартный MCP через
`callMcp`.

**2. Весь канал — через `callMcp` (приватный метод).**
`p = Ro()` = `hn({isFirstParty: true, includePrivate: true})`.
Без `callMcp` — `-32000 "MCP proxy not enabled"`.

**3. `ui/initialize` перехватывается, но не заменяется.**
```js
p.initializeMcpApp(q.data.params.appCapabilities)
```
Хост получает возможности виджета, возвращает свой контекст, AppBridge
отвечает стандартно.

**4. `host-context-changed` до `initialized` — заблокирован, но не в очередь.**
После `initialized` не досылается. Компенсация — `setHostContext(ne)`
при `ui/initialize`.

**5. `openExternal` без проверки схемы.**
Виджет может отправить `javascript:`, `file://`, `intent://`.

**6. `tools/list` всегда разрешён.**
Даже если не в `forwardedRequestMethods`. Утечка списка инструментов.

**7. `elicitation/create`, `roots/list`, `resources/subscribe`, `prompts/get`
— только через `forwardedRequestMethods` (по умолчанию пусто).**

**8. `ui/message` — 5 проверок.**
Валидация → форвардинг → отказ на `_meta["openai/message"]` → роль `user`
→ `sendFollowUpMessage` есть → текст не пустой.

**9. `ui/update-model-context` → `setWidgetState({waitForPersistence: true})`.**
Ждёт сохранения перед ответом.

**10. CSP-схема (`Nt`) — 4 поля: `connectDomains`, `resourceDomains`,
`frameDomains`, `baseUriDomains`.** Ровно то, что принимает `apply-csp`.

**11. Permissions Policy: camera, microphone, geolocation, clipboardWrite.**
Объявляются в схеме, применяются к iframe.

**12. Очистка `ze` перед циклом — при ошибке в `t()` остаток теряется.**

**13. `_e` не сбрасывается при ошибке `ui/initialize` — очередь не разгрузится.**
