# `adapter-CQcBVlDI.js` — доскональный разбор

**Размер:** 119 651 байт
**URL:** `https://web-sandbox.oaiusercontent.com/assets/adapter-CQcBVlDI.js`
**Роль:** MCP Apps adapter — превращает виджет в MCP-приложение
**Прочитан целиком:** символы 0 – 119 651

---

## 0. Состав файла

| Диапазон | Что |
|---|---|
| 0 – 57 500 | **`zod`** — схема-валидатор (основная часть файла) |
| 57 500 – 68 000 | JSON-RPC схемы, константы протокола |
| 68 000 – 88 000 | Схемы MCP (tools, resources, prompts) |
| 88 000 – 97 500 | Схемы MCP Apps UI (display modes, CSP, permissions) |
| 97 500 – 103 000 | `HostContext`, `AppCapabilities`, `HostCapabilities` |
| 103 000 – 106 500 | `Hu` — транспорт (postMessage) |
| 106 500 – 116 000 | `ju` — `AppBridge` (главный класс) |
| 116 000 – 119 434 | Глобальное состояние + `initMcpAppsAdapter` (`el`) |
| 119 435 – 119 651 | Экспорты |

---

## 1. Версия протокола

```js
bo = "2026-01-26"        // текущая версия MCP Apps
Ou = [bo]                // поддерживаемые версии
```

При `ui/initialize` клиент присылает свою `protocolVersion`. Если она в `Ou` —
возвращается она же, иначе принудительно `"2026-01-26"`.

---

## 2. Транспорт — класс `Hu`

```js
class Hu {
  constructor(eventTarget = window.parent, eventSource) {
    this.eventTarget = eventTarget;      // window.parent
    this.eventSource = eventSource;      // ожидаемый source сообщений
    this.messageListener = o => {
      // если задан eventSource и o.source !== eventSource → игнор
      if (n && o.source !== this.eventSource) {
        console.debug("Ignoring message from unknown source", o);
        return;
      }
      const r = Ja.safeParse(o.data);    // zod-валидация входящего
      if (r.success) {
        this.onmessage?.(r.data);
      } else if (o.data?.jsonrpc !== "2.0") {
        console.debug("Ignoring non-JSON-RPC message", r.error.message, o);
      } else {
        console.error("Failed to parse message", r.error.message, o);
        this.onerror?.(Error("Invalid JSON-RPC message received: " + r.error.message));
      }
    };
  }

  async start() {
    window.addEventListener("message", this.messageListener);
  }

  async send(t, n) {
    if (t.method !== yu) console.debug("Sending message", t);
    this.eventTarget.postMessage(t, "*");       // ← targetOrigin "*"
  }

  async close() {
    window.removeEventListener("message", this.messageListener);
    this.onclose?.();
  }
}
```

**Внимание:** `postMessage(t, "*")` — **всегда звёздочка**, без проверки origin.

---

## 3. `AppBridge` — класс `ju`

Наследуется от MCP-сервера `ku`. Это **мост между виджетом и хостом**.

### Конструктор

```js
constructor(client, hostInfo, capabilities, options) {
  super(options);
  this._client = client;                    // MCP-клиент (тут null)
  this._hostInfo = hostInfo;                // {name, version}
  this._capabilities = capabilities;        // HostCapabilities
  this._hostContext = options?.hostContext || {};
  this._initializedReceived = false;

  // Обработчики-геттеры/сеттеры
  this.onping;
  this._onmessage;  this._onopenlink;  this._ondownloadfile;
  this._onrequestdisplaymode;  this._onupdatemodelcontext;
  this._oncalltool;  this._onlistresources;  this._onlistresourcetemplates;
  this._onreadresource;  this._onlistprompts;
  this.sendResourceTeardown = this.teardownResource;

  // События
  this.eventSchemas = {
    sizechange: Eu, sandboxready: Iu, initialized: Mu,
    requestteardown: Zu, loggingmessage: go
  };

  this.addEventListener("initialized", () => { this._initializedReceived = true; });
  this.setRequestHandler(ko, i => this._oninitialize(i));   // ui/initialize
  this.setRequestHandler(tt, (i, c) => { this.onping?.(i.params, c); return {}; });  // ping
  this.replaceRequestHandler(ln, i => ({mode: this._hostContext.displayMode ?? "inline"}));  // ui/request-display-mode
}
```

### Защита от вызова до рукопожатия

```js
this._baseReplaceRequestHandler = this.replaceRequestHandler;
this.replaceRequestHandler = (method, handler) => {
  this._baseReplaceRequestHandler(method, (request, extra) => {
    if (!this._initializedReceived) {
      console.warn(`[ext-apps] AppBridge received '${method.method}' before ui/notifications/initialized. The View is calling host methods before completing the handshake; it should await app.connect() first.`);
    }
    return handler(request, extra);
  });
};
```

Виджет не может вызвать хост, пока не прошёл `ui/initialize` → `ui/notifications/initialized`.

---

## 4. Свойства-обработчики

Каждое свойство при установке регистрирует request-handler:

```js
get onmessage() { return this._onmessage; }
set onmessage(n) {
  this.warnIfRequestHandlerReplaced("onmessage", this._onmessage, n);
  this._onmessage = n;
  this.replaceRequestHandler(Ht, async (o, r) => {          // ui/message
    if (!this._onmessage) throw Error("No onmessage handler set");
    return this._onmessage(o.params, r);
  });
}

get onopenlink() { return this._onopenlink; }
set onopenlink(n) {
  this._onopenlink = n;
  this.replaceRequestHandler(Tu, async (o, r) => { ... });  // ui/open-link
}

get ondownloadfile() { ... }                                 // ui/download-file
get onrequestdisplaymode() { ... }                           // ui/request-display-mode
get onupdatemodelcontext() { ... }                           // ui/update-model-context
get oncalltool() { ... }                                     // tools/call
get onlistresources() { ... }                                // resources/list
get onlistresourcetemplates() { ... }                        // resources/templates/list
get onreadresource() { ... }                                 // resources/read
get onlistprompts() { ... }                                  // prompts/list
```

---

## 5. Отправка уведомлений (host → app)

```js
sendToolListChanged(params = {})
  → notifications/tools/list_changed

sendResourceListChanged(params = {})
  → notifications/resources/list_changed

sendPromptListChanged(params = {})
  → notifications/prompts/list_changed

sendToolInput(input)
  → ui/notifications/tool-input

sendToolInputPartial(input)
  → ui/notifications/tool-input-partial

sendToolResult(result)
  → ui/notifications/tool-result

sendToolCancelled(params)
  → ui/notifications/tool-cancelled

sendSandboxResourceReady({html, sandbox, csp, permissions})
  → ui/notifications/sandbox-resource-ready

sendHostContextChange(changed)
  → ui/notifications/host-context-changed

setHostContext(newContext)
  → сравнивает с текущим (JSON.stringify), отправляет только изменения
```

---

## 6. Запросы (app → host)

```js
teardownResource(params, opts)   → ui/resource-teardown
callTool(params, opts)           → tools/call
listTools(params, opts)          → tools/list
```

---

## 7. `src/` блок: `connect()`

```js
async connect(n) {
  if (this.transport) throw Error("AppBridge is already connected. Call close() before connecting again.");
  this._initializedReceived = false;

  if (this._client) {
    const o = this._client.getServerCapabilities();
    if (!o) throw Error("Client server capabilities not available");

    // tools
    o.tools && (
      this.oncalltool = async (r, s) =>
        this._client.request({method: "tools/call", params: r}, Ee, {signal: s.signal}),
      o.tools.listChanged && this._client.setNotificationHandler(mo, r => this.sendToolListChanged(r.params))
    );

    // resources
    o.resources && (
      this.onlistresources = async (r, s) => this._client.request({method: "resources/list", params: r}, eo, {signal: s.signal}),
      this.onlistresourcetemplates = async (r, s) => this._client.request({method: "resources/templates/list", params: r}, no, {signal: s.signal}),
      this.onreadresource = async (r, s) => this._client.request({method: "resources/read", params: r}, ro, {signal: s.signal}),
      o.resources.listChanged && this._client.setNotificationHandler(so, r => this.sendResourceListChanged(r.params))
    );

    // prompts
    o.prompts && (
      this.onlistprompts = async (r, s) => this._client.request({method: "prompts/list", params: r}, ao, {signal: s.signal}),
      o.prompts.listChanged && this._client.setNotificationHandler(lo, r => this.sendPromptListChanged(r.params))
    );
  }
  return super.connect(n);
}
```

**В нашем случае `this._client === null`** — прокси идёт через `callMcp`, а не через прямой MCP-клиент.

---

## 8. Жизненный цикл `ui/initialize`

```js
async _oninitialize(n) {
  const o = n.params.protocolVersion;

  if (this._appInfo !== undefined)
    console.warn("[ext-apps] AppBridge received a second ui/initialize. The View may be double-mounting (e.g. React StrictMode in dev) without closing the previous App instance. Responding normally; the latest appInfo/appCapabilities replace the previous values.");

  this._appCapabilities = n.params.appCapabilities;
  this._appInfo = n.params.appInfo;

  return {
    protocolVersion: Ou.includes(o) ? o : bo,    // эхо или текущая
    hostCapabilities: this.getCapabilities(),
    hostInfo: this._hostInfo,
    hostContext: this._hostContext
  };
}
```

---

## 9. Глобальное состояние адаптера

```js
let M = null;      // текущий AppBridge
let Re;            // sandboxLogger
let ne = null;     // hostContext
let _e = false;    // initialized?
let ze = [];       // очередь вызовов до initialized
```

```js
const He = e => {
  if (!M) return;
  if (!_e) { ze.push(e); return; }    // до initialized — в очередь
  e();
};

const Vu = () => {
  if (!ze.length) return;
  const e = ze; ze = [];
  for (const t of e) t();
};
```

**Любое уведомление до `initialized` ставится в очередь** и выполняется после рукопожатия. Это защита от гонки.

---

## 10. Экспортируемые функции

```js
// host → app (уведомления)
notifyMcpAppsToolInput(input)      = e => He(() => M?.sendToolInput(e))
notifyMcpAppsToolResult(result)    = e => { const t = dn(e); t != null && He(() => M?.sendToolResult(t)); }
notifyMcpAppsToolCancelled(params) = e => He(() => M?.sendToolCancelled(e))
notifyMcpAppsMcpNotification(note) = e => He(() => M?.transport?.send({...e, jsonrpc: "2.0"}))
notifyMcpAppsHostContext(ctx)      = e => { ne = {...ne ?? {}, ...e}; He(() => M?.setHostContext(ne ?? e)); }

// teardown
requestMcpAppsResourceTeardown(timeout = 500)
  = async e => { if (M && _e) try { await M.teardownResource({}, {timeout: e}); } catch (t) { Re?.error("MCP Apps resource teardown failed", t); } }
```

---

## 11. `initMcpAppsAdapter()` — главная функция

```js
initMcpAppsAdapter({
  onInitialized,
  onInitializationError,
  viewWindow,
  forwardedRequestMethods = [],
  hostContext,
  hostCapabilities,
  hostInfo,
  getSetWidgetState,
  sandboxLogger
})
```

### Шаг 1. Получить Host API

```js
const p = Ro();                    // ← hn({isFirstParty:true, includePrivate:true})
const f = new Set(forwardedRequestMethods);
ne = hostContext;
```

**Ключевое:** `Ro()` = `X()` из main.js = `hn({isFirstParty: true, includePrivate: true})`.
То есть **адаптеру доступны ВСЕ 31 скрытых метода + prepareWidgetDocument**.

### Шаг 2. Создать AppBridge

```js
const g = new ju(null, {
  name: hostInfo?.name ?? "chatgpt",
  version: hostInfo?.version ?? "0.0.1"
}, hostCapabilities, {hostContext});

M = g;
Re = sandboxLogger;
_e = false;                        // initialized = false
```

### Шаг 3. Прокси `callMcp`

```js
const v = () => {
  const m = new Error("MCP proxy not enabled");
  m.code = -32000;
  throw m;
};

const b = async (method, params) => {
  if (!p.callMcp) return v();
  try {
    const P = await p.callMcp({method, params});
    return method === "tools/call" ? dn(P) ?? P : P;
  } catch (P) {
    const q = P != null && typeof P == "object" ? P : {};
    const msg = "message" in q && typeof q.message == "string" ? q.message : "MCP proxy request failed";
    const code = "code" in q && typeof q.code == "number" ? q.code : -32000;
    throw Object.assign(new Error(msg, {cause: P}), {code, ...("data" in q ? {data: q.data} : {})});
  }
};
```

### Шаг 4. Коды ошибок

```js
const k = m => { const y = new Error(m); y.code = -32602; throw y; };   // Invalid params
```

### Шаг 5. Нормализация display mode

```js
const Z = m => m === "inline" || m === "fullscreen" || m === "pip" ? m : "inline";
```

### Шаг 6. Извлечение текста из content-блоков

```js
const se = m => {
  const y = [];
  for (const P of m)
    if (P.type === "text" && typeof P.text === "string") {
      const q = P.text.trim();
      q && y.push(q);
    }
  return y.join("\n");
};
```

### Шаг 7. Обработчик `onopenlink`

```js
g.onopenlink = async ({url: m}) =>
  p.openExternal
    ? (p.openExternal({href: m}), {})
    : (sandboxLogger?.log("MCP Apps open-link ignored (host missing openExternal)"), {isError: true});
```

### Шаг 8. Обработчик `ui/message`

```js
g.setRequestHandler(Uu, async ({params: m}) => {
  const y = Lu.safeParse(m);
  if (!y.success) return k("Invalid MCP message params");

  // forward если разрешено
  if (f.has("ui/message")) return await b("ui/message", m);

  // спец-поле openai/message в _meta → отказ
  if (y.data._meta?.["openai/message"] != null) return {isError: true};

  const {role: P, content: q} = y.data;

  if (P !== "user") {
    sandboxLogger?.log("MCP Apps message ignored (unsupported role)", P);
    return {isError: true};
  }

  if (!p.sendFollowUpMessage) {
    sandboxLogger?.log("MCP Apps message ignored (host missing sendFollowUpMessage)");
    return {isError: true};
  }

  const ie = se(q);
  return ie
    ? (p.sendFollowUpMessage({prompt: ie, isVisible: true}), {})
    : (sandboxLogger?.log("MCP Apps message ignored (empty text content)"), {isError: true});
});
```

**Важно:** `ui/message` → `sendFollowUpMessage({prompt, isVisible: true})`.
То есть MCP-приложение может **отправить сообщение в чат от имени пользователя**.

**Защита:** `_meta["openai/message"]` блокируется — нельзя подделать внутреннее сообщение.

### Шаг 9. Обработчик `ui/update-model-context`

```js
g.setRequestHandler(Fu, async ({params: m}) => {
  if (!Ot.shape.params.safeParse(m).success) return k("Invalid MCP model context params");
  if (f.has("ui/update-model-context")) return await b("ui/update-model-context", m);

  const P = getSetWidgetState?.();
  if (!P) throw Object.assign(new Error("Method not found"), {code: -32601});

  return await P(m, {waitForPersistence: true}), {};
});
```

**`ui/update-model-context` позволяет приложению обновить контекст модели** —
то есть вставить свой текст в контекст следующего хода.

### Шаг 10. Остальные обработчики

```js
g.onrequestteardown = () => p.requestClose?.();

g.onrequestdisplaymode = async ({mode: m}) => {
  const y = await p.requestDisplayMode?.({mode: m});
  const P = y && typeof y === "object" && "mode" in y
    ? y.mode
    : (ne?.displayMode ?? hostContext.displayMode ?? m);
  return {mode: Z(P)};
};

g.onsizechange = ({width, height}) => {
  width != null && p.notifyIntrinsicWidth?.(width);
  height != null && p.notifyIntrinsicHeight?.(height);
};

g.onloggingmessage = ({level, logger, data}) => {
  if (!sandboxLogger) return;
  const q = logger ? `[mcp-app:${logger}]` : "[mcp-app]";
  sandboxLogger.log(`${q} ${level}`, data);
};

g.oninitialized = () => {
  M === g && (_e = true, Vu(), onInitialized?.());
};
```

### Шаг 11. MCP-методы через прокси

```js
const te = async m => {
  m?.name || k("Missing tool name");
  const y = {...m, arguments: m.arguments ?? {}};
  return await b("tools/call", y);
};
g.oncalltool = te;

g.onlistresources     = async m => await b("resources/list", m && typeof m == "object" ? m : undefined);
g.onreadresource      = async m => (m?.uri || k("Missing resource uri"), await b("resources/read", m));
g.onlistresourcetemplates = async m => await b("resources/templates/list", m && typeof m == "object" ? m : undefined);
g.onlistprompts       = async m => await b("prompts/list", m && typeof m == "object" ? m : undefined);
g.onping              = async () => await b("ping");

// fallback для forwardRequestMethods
g.fallbackRequestHandler = async m => {
  if (f.has(m.method)) return await b(m.method, m.params);
  if (m.method === "tools/list") return await b("tools/list", m.params);
  const y = new Error("Method not found");
  y.code = -32601;
  throw y;
};
```

### Шаг 12. Транспорт с перехватом `ui/initialize`

```js
const Q = new class extends Hu {
  async start() {
    const m = this.onmessage;
    this.onmessage = async (y, P) => {
      const q = ko.safeParse(y);       // ui/initialize

      // не ui/initialize или нет id или нет initializeMcpApp → пропустить
      if (!q.success || !("id" in y) || !p.initializeMcpApp) {
        m?.(y, P);
        return;
      }

      try {
        const ie = await p.initializeMcpApp(q.data.params.appCapabilities);
        if (M !== g) return;
        ne = {...ne, ...ie};
        g.setHostContext(ne);
        m?.(y, P);
      } catch (ie) {
        sandboxLogger?.error("MCP Apps initialization failed", ie);
        if (M !== g) return;
        this.send({
          jsonrpc: "2.0", id: y.id,
          error: {code: -32603, message: "MCP app initialization failed"}
        }).catch(me => sandboxLogger?.error("MCP Apps initialization reply failed", me));
      }
    };
    await super.start();
  }

  async send(m) {
    // не отправлять host-context-changed до initialized
    if (!_e && "method" in m && m.method === "ui/notifications/host-context-changed") return;
    await super.send(m);
  }
}(viewWindow, viewWindow);
```

**Зачем перехват:** перед ответом на `ui/initialize` вызывается
`host.initializeMcpApp(appCapabilities)` — хост решает, разрешать ли это приложение.

### Шаг 13. Подключение и возврат

```js
Q.onerror = m => sandboxLogger?.error("MCP Apps transport error", m);

g.connect(Q).catch(m => {
  M === g && (onInitializationError ? onInitializationError(m) : sandboxLogger?.environmentError(m));
});

return {
  destroy: () => {
    M === g && (M = null, Re = undefined, ne = null, _e = false, ze = []);
    Q.close();
  }
};
```

---

## 12. Схемы, которые виджет должен соблюдать

### `AppCapabilities` (`qu`) — что объявляет приложение

```js
{
  experimental: {},                          // TBD
  tools: {listChanged?: boolean},            // App поддерживает tools/list_changed
  availableDisplayModes: ["inline"|"fullscreen"|"pip"][]
}
```

### `HostCapabilities` (`xu`) — что объявляет хост

```js
{
  sandbox: {
    permissions?: ["camera"|"microphone"|"geolocation"],   // что разрешено
    csp?: {...}                                             // домены CSP
  },
  updateModelContext?: {},        // хост принимает ui/update-model-context
  message?: {},                   // хост принимает ui/message
  sampling: {tools?: {}}          // хост поддерживает LLM sampling
}
```

### `HostContext` (`wo`)

```js
{
  toolInfo: {id, tool},           // метаданные вызова инструмента
  theme: "light" | "dark",
  styles: {...},
  displayMode: "inline" | "fullscreen" | "pip",
  availableDisplayModes: [...],
  containerDimensions: {height | maxHeight, width | maxWidth},
  locale: "en-US",
  timeZone: "Asia/Yekaterinburg",    // IANA
  userAgent: "...",
  platform: "web" | "desktop" | "mobile",
  deviceCapabilities: {touch?: bool, hover?: bool},
  safeAreaInsets: {top, right, bottom, left}
}
```

### `ui/initialize` (app → host)

```js
{
  appInfo: {name, version},
  appCapabilities: {...},
  protocolVersion: "2025-11-21" | "2026-01-26"
}
```

### `ui/initialize` ответ (host → app)

```js
{
  protocolVersion: "<эхо или 2026-01-26>",
  hostInfo: {name, version},
  hostCapabilities: {...},
  hostContext: {...}
}
```

---

## 13. CSP и permissions в MCP-схеме

```js
{
  csp: ContentSecurityPolicyConfig,     // домены для connect/script/img/frame
  permissions: SandboxPermissions,      // camera, microphone, geolocation
  domain: string,                       // выделенный origin для view
  prefersBorder: boolean                // нужна ли рамка
}
```

**`domain`** — интересное поле:
```
"Useful when views need stable, dedicated origins for OAuth callbacks,
 CORS policies, or API key allowlists."
```
Документация прямо упоминает паттерны:
```
{hash}.claudemcpcontent.com
www-example-com.oaiusercontent.com
```

То есть **каждому MCP-приложению может выдаваться свой поддомен**.

---

## 14. Полный список методов

### App → Host (запросы)

| Метод | Что делает |
|---|---|
| `ui/initialize` | рукопожатие |
| `ui/message` | отправить сообщение в чат |
| `ui/open-link` | открыть ссылку |
| `ui/download-file` | скачать ресурс |
| `ui/request-display-mode` | сменить режим отображения |
| `ui/update-model-context` | обновить контекст модели |
| `ui/resource-teardown` | разрушить ресурс |
| `tools/call` | вызвать инструмент |
| `tools/list` | список инструментов |
| `resources/list` | список ресурсов |
| `resources/templates/list` | шаблоны ресурсов |
| `resources/read` | прочитать ресурс |
| `prompts/list` | список промптов |
| `ping` | проверка связи |

### Host → App (уведомления)

| Метод | Что несёт |
|---|---|
| `ui/notifications/initialized` | рукопожатие завершено |
| `ui/notifications/tool-input` | вход инструмента |
| `ui/notifications/tool-input-partial` | частичный вход |
| `ui/notifications/tool-result` | результат инструмента |
| `ui/notifications/tool-cancelled` | отмена |
| `ui/notifications/host-context-changed` | изменился контекст |
| `ui/notifications/sandbox-resource-ready` | sandbox готов |
| `notifications/tools/list_changed` | список инструментов изменился |
| `notifications/resources/list_changed` | список ресурсов |
| `notifications/prompts/list_changed` | список промптов |

---

## 15. Что может MCP-приложение — по факту

```js
✅ открыть ссылку                 → через onopenlink → host.openExternal
✅ отправить сообщение в чат      → ui/message → sendFollowUpMessage({prompt, isVisible:true})
✅ обновить контекст модели       → ui/update-model-context → setWidgetState(waitForPersistence)
✅ сменить display mode           → ui/request-display-mode → requestDisplayMode
✅ вызвать любой MCP-инструмент   → tools/call → host.callMcp
✅ получить список инструментов   → tools/list
✅ читать ресурсы                 → resources/read
✅ список промптов                → prompts/list
✅ сообщить размер                → onsizechange → notifyIntrinsicWidth/Height
✅ логировать                     → onloggingmessage
```

**Всё это требует `forwardedRequestMethods` или явных возможностей хоста.**

---

## 16. Слабые места

### 1. `postMessage(t, "*")` в транспорте

```js
async send(t, n) {
  this.eventTarget.postMessage(t, "*");
}
```

**Всегда `"*"`.** Проверка source есть (`o.source !== this.eventSource` → игнор),
но проверки origin **нет**.

### 2. `ui/message` → `sendFollowUpMessage`

```js
p.sendFollowUpMessage({prompt: ie, isVisible: true});
```

MCP-приложение может отправить сообщение **от имени пользователя**, с флагом
`isVisible: true`. Единственная защита — `_meta["openai/message"]` блокируется.

### 3. `ui/update-model-context`

```js
return await P(m, {waitForPersistence: true}), {};
```

Приложение вставляет свой контент в контекст модели на следующем ходу.
Это **влияние на последующие ответы модели**.

### 4. `initializeMcpApp(appCapabilities)` — без валидации appCapabilities

```js
const ie = await p.initializeMcpApp(q.data.params.appCapabilities);
```

Схема `qu` объявлена, но `ko.safeParse` проверяет только `appInfo`/`appCapabilities`/`protocolVersion`
как `d(...)` — то есть структура `appCapabilities` **не валидируется схемой `qu`**.

### 5. `domain` — поддомен для OAuth

Если MCP-приложение попросит `domain: "myapp"`, получит
`myapp.oaiusercontent.com`. Это **отдельный origin** с сохранением cookie/localStorage
между сессиями.

---

## 17. Что даёт это виджету

**Виджет может быть полноценным MCP-приложением:**

```js
// в коде виджета
window.parent.postMessage({
  jsonrpc: "2.0", id: 1, method: "ui/initialize",
  params: {
    appInfo: {name: "my-app", version: "1.0.0"},
    appCapabilities: {
      tools: {listChanged: true},
      availableDisplayModes: ["inline", "fullscreen"]
    },
    protocolVersion: "2026-01-26"
  }
}, "*");
```

После рукопожатия — полный доступ к `tools/call`, `resources/read`,
`ui/message`, `ui/update-model-context`.

**Но:** `callMcp` в основном коде — из `Jr` (только first-party). Значит
**MCP-вызовы из обычного виджета не пройдут** — вернётся
`{code: -32000, message: "MCP proxy not enabled"}`.

---

## 18. Итог

`adapter-CQcBVlDI.js` — **полная реализация MCP Apps протокола** (версия
`2026-01-26`), вшитая в песочницу.

```
zod (57 КБ)        — валидация всех входящих сообщений
Hu                 — транспорт postMessage (targetOrigin "*")
ju (AppBridge)     — мост app↔host с 14 методами
Hu + перехват      — ui/initialize проходит через host.initializeMcpApp
el()               — фабрика адаптера
```

**Ключевое ограничение:** адаптер получает `Ro()` = `hn({isFirstParty:true, includePrivate:true})`.
Он видит все скрытые методы — **но только внутри адаптера**. Виджет через адаптер
вызывает то, что адаптер ему разрешил: `openExternal`, `sendFollowUpMessage`,
`callMcp` (если хост дал), `requestDisplayMode`, `notifyIntrinsicWidth/Height`.

**Практическое применение для виджета:** MCP-протокол открывает
`ui/update-model-context` — то, чем можно влиять на контекст модели.
Но требуется `hostCapabilities.updateModelContext`.
