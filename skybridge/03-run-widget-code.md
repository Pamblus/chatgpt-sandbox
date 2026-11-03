# `run-widget-code-RkUxf2Ko.js` — доскональный разбор

**Размер:** 3 455 байт
**URL:** `https://web-sandbox.oaiusercontent.com/assets/run-widget-code-RkUxf2Ko.js`
**Роль:** рендер виджета в iframe. Связывает `main.js` + `adapter` + глобалы.

---

## 1. Импорты

```js
import {a as Q} from "./apply-csp-_9OvJz9C.js";          // applyCsp
import {
  S as G,   // "openai:set_globals"
  a as U,   // класс события to (openai:set_globals)
  b as X,   // xr() — очистить все iframe#root
  d as H,   // X() — публикация Host API (first-party)
  g as Z,   // hn() — публикация Host API с параметрами
  L as $,   // Mn — locale
  e as z,   // Ms() — widget state factory
  f as F,   // Ps() — setWidgetView factory
  r as m,   // Dr() — runHTMLAsync
  o as A,   // yi() — автозаполнение
  h as p    // mi() — получить setWidgetState
} from "./main-BR1bOQyH.js";
import {initMcpAppsAdapter as ee} from "./adapter-CQcBVlDI.js";
```

---

## 2. Функция `te()` — управление фоном

```js
function te({document: i, backgroundColor: v, viewWindow: g}) {
  const o = i.createElement("style");
  let b = true;                              // alive

  // переустановить style, если его удалили
  function u() {
    b && o.textContent !== "" && !o.isConnected && i.head?.append(o);
  }

  // установить фон
  function f(a) {
    o.textContent = a === "transparent"
      ? "html, body { background: transparent !important; }"
      : "";
    a === "transparent" ? u() : o.remove();
  }

  // реагировать на смену surfaceBackgroundColor
  function d(a) {
    if (!(a instanceof U)) return;             // только "openai:set_globals"
    const C = a.detail.globals.surfaceBackgroundColor;
    typeof C === "string" && f(C);
  }

  f(v);                                        // применить сразу

  // MutationObserver: если head очистили — вернуть style
  const h = new MutationObserver(u);
  h.observe(i, {childList: true, subtree: true});
  queueMicrotask(u);

  g.addEventListener(G, d);                    // слушать openai:set_globals

  return () => {
    b = false;
    h.disconnect();
    o.remove();
    g.removeEventListener(G, d);
  };
}
```

**Зачем:** виджет может очистить `<head>` (например, `document.head.innerHTML = ""`).
Тогда `<style>` с прозрачным фоном исчезнет и появится белая/чёрная подложка.
`te()` возвращает его обратно.

---

## 3. `runWidgetCode()` — главная функция

### Полная сигнатура (36 параметров)

```js
async function* runWidgetCode(sandboxLogger, {
  // ── мост ────────────────────────────────────────────────
  appBlockMessenger,
  hostHandlesFollowUpMessageAuthorization,

  // ── контент ─────────────────────────────────────────────
  html,
  csp,

  // ── режим ───────────────────────────────────────────────
  isFirstParty,
  disableExternalNavigation = false,
  allowPopups,                       // вычисляется: !disableExternalNavigation

  // ── виджет ──────────────────────────────────────────────
  widgetId,
  enableAutofill = false,
  widgetState,
  viewParams,
  isTombstone = false,

  // ── данные инструмента ──────────────────────────────────
  toolInput,
  toolOutput,
  toolResponseMetadata,
  subjectId = null,

  // ── метаданные приложения ───────────────────────────────
  appPackageName,
  supportsOAuthCompletion,
  supportsSourceCitationLinks,

  // ── UI ──────────────────────────────────────────────────
  maxHeight,
  maxWidth,
  displayMode,
  isSidebarOpen,
  theme,
  userAgent,
  safeArea,
  measureWidth,

  // ── готовность ──────────────────────────────────────────
  expectReadySignal = false,
  expectedReadySignal,

  // ── прочее ──────────────────────────────────────────────
  features,
  screenshotVegaPreview,
  mcpApps
})
```

### Шаг 1. CSP

```js
try { Q(csp); }                                    // applyCsp
catch (l) { throw sandboxLogger.error("Failed to apply CSP", l), l; }
```

CSP применяется **до** всего остального. Если политика ослабляется — исключение.

### Шаг 2. Очистка старых iframe

```js
(mcpApps || enableAutofill) && X();                // xr(): убрать все iframe#root
```

Перед новым рендером убираются старые — если это MCP-приложение или включено
автозаполнение.

### Шаг 3. Сборка глобалов

```js
let t, P;

try {
  P = H().prepareWidgetDocument;                   // приватный метод (только first-party)

  t = Z({
    isFirstParty,
    includePrivate: false,
    hostHandlesFollowUpMessageAuthorization
  });                                              // ← Host API

  t.displayMode = displayMode;
  appBlockMessenger !== void 0 && (t.__appBlockMessenger = appBlockMessenger);

  t.maxHeight  = maxHeight;
  t.maxWidth   = maxWidth;
  t.theme      = theme;
  t.locale     = $;                                // Mn из URL
  t.userAgent  = userAgent;
  t.appPackageName = isFirstParty ? appPackageName : void 0;
  t.supportsOAuthCompletion      = isFirstParty && supportsOAuthCompletion === true;
  t.supportsSourceCitationLinks  = isFirstParty && supportsSourceCitationLinks === true;
  mcpApps?.hostInfo != null && (t.hostInfo = mcpApps.hostInfo);
  t.isSidebarOpen = isSidebarOpen ?? false;

  // устаревший widget API
  t.widget = {
    state: widgetState,
    props: toolResponseMetadata,
    setState: c => {
      console.warn("window.openai.widget is deprecated. It will be removed in future versions");
      return H()?.updateWidgetState?.call(H(), widgetId, c);
    }
  };

  // widget state (toolInput/toolOutput/setWidgetState)
  const l = z({
    widgetId,
    globals: {toolInput, toolOutput, toolResponseMetadata, widgetState, subjectId},
    sandboxLogger
  });

  // макс. приоритет
  Object.assign(t, l, F({displayMode, viewParams, isTombstone}));
} catch {}
```

**Что получается в `window.openai`:**

```js
window.openai = {
  // Host API (8 клик-методов, если не first-party)
  requestDisplayMode, sendFollowUpMessage, openExternal, openExternalForm,
  openPlaidLink, requestPrompt, requestTargetedReply, triggerHaptic,
  // + все 31 скрытых, если isFirstParty

  // из runWidgetCode
  displayMode, maxHeight, maxWidth, theme, locale, userAgent,
  isSidebarOpen, appPackageName*, supportsOAuthCompletion*,
  supportsSourceCitationLinks*, hostInfo*,
  // * — только first-party

  __appBlockMessenger,          // если передан

  // устаревший
  widget: {state, props, setState},

  // из Ms()
  toolInput, toolOutput, toolResponseMetadata, widgetState, subjectId,
  setWidgetState,

  // из Ps()
  displayMode, view: {params, mode, isTombstone}
};
```

### Шаг 4. Инициализация MCP

```js
const s = mcpApps?.requiresInitialization ? Promise.withResolvers() : void 0;
```

Если MCP-приложение требует инициализации — создаётся promise, который
разрешится после `ui/initialized`.

### Шаг 5. Запуск рендера

```js
yield* m({                                    // Dr() — runHTMLAsync
  sandboxLogger,
  html,
  theme,
  features,
  additionalGlobals: t,
  safeArea,
  transformPosition: null,

  expectReadySignal,
  expectedReadySignal,
  isFirstParty,

  forwardConsoleLogs: !mcpApps,               // для MCP — выключено
  measureHeight: true,
  measureWidth,
  disableMutationObserver: !expectReadySignal && isFirstParty && !measureWidth,
  disableExternalNavigation,
  allowPopups: !disableExternalNavigation,

  onIframeReady: mcpApps || enableAutofill
    ? ({viewWindow: l, document: c}) => {
        // автозаполнение
        const r = enableAutofill ? A({document: c, viewWindow: l}) : void 0;

        // стиль фона для MCP с globalSurface
        const n = mcpApps?.globalSurface
          ? te({
              document: c,
              backgroundColor: mcpApps?.hostContext.styles?.variables?.["--color-background-primary"],
              viewWindow: l
            })
          : void 0;

        let y = null;

        // MCP adapter
        if (mcpApps) {
          try {
            y = ee({
              onInitialized: s?.resolve,
              onInitializationError: s?.reject,
              viewWindow: l,
              forwardedRequestMethods: mcpApps.forwardedRequestMethods,
              hostContext: mcpApps.hostContext,
              hostCapabilities: mcpApps.hostCapabilities,
              hostInfo: mcpApps.hostInfo,
              getSetWidgetState: p,          // mi() → Sr
              sandboxLogger
            }).destroy;
          } catch (K) {
            throw (r?.(), n?.(), K);
          }
        }

        return () => { r?.(); n?.(); y?.(); };
      }
    : void 0,

  appInitialization: s?.promise,
  prepareDocument: P
});
```

---

## 4. Что включается флагами

| Флаг | Что |
|---|---|
| `enableAutofill` | сканирование `<input>` и отправка кандидатов хосту (`yi()`) |
| `mcpApps` | полноценный MCP-протокол через adapter |
| `mcpApps.globalSurface` | прозрачный/кастомный фон синхронизируется с хостом |
| `mcpApps.requiresInitialization` | ждать `ui/initialized` перед рендером |
| `disableExternalNavigation` | `allowPopups = false`, CSP без внешних доменов |
| `expectReadySignal` | ждать `data-ready` на `<html>` |

---

## 5. Связь с `Dr()` (runHTMLAsync)

Все параметры `runWidgetCode` транслируются в `Dr()`:

```
sandboxLogger             → sandboxLogger
html                      → html
theme                     → theme
features                  → features
t (глобалы)               → additionalGlobals
safeArea                  → safeArea
transformPosition         → null (никогда)
expectReadySignal         → expectReadySignal
expectedReadySignal       → expectedReadySignal
isFirstParty              → isFirstParty
!mcpApps                  → forwardConsoleLogs
true                      → measureHeight
measureWidth              → measureWidth
!expectReadySignal && isFirstParty && !measureWidth
                          → disableMutationObserver
disableExternalNavigation → disableExternalNavigation
!disableExternalNavigation → allowPopups
callback                  → onIframeReady
s?.promise                → appInitialization
P (prepareWidgetDocument) → prepareDocument
```

---

## 6. `prepareWidgetDocument`

**Это приватный метод Host API.** Доступен только через
`H()` = `X()` = `hn({isFirstParty: true, includePrivate: true})`.

Передаётся в `Dr()` как `prepareDocument`. В `Dr()`:

```js
if (prepareDocument != null) {
  try {
    await Promise.race([
      new Promise((resolve, reject) => {
        de = resolve;
        xe = setTimeout(() => reject(new Error("Document preparation timed out")), 10000);
      }),
      prepareDocument()
    ]);
  } finally {
    de = undefined;
    clearTimeout(xe);
  }

  if (cancelled || !iframe.isConnected || iframe.contentDocument !== P) {
    Ct(true);      // отмена
    return;
  }
}
```

**Таймаут — 10 секунд.** Если `prepareWidgetDocument` не завершится — рендер
отменяется.

**Назначение:** подготовить документ до `document.write(html)`. Скорее всего —
зарегистрировать ServiceWorker, установить заголовки или подгрузить ресурсы.

---

## 7. Устаревший `window.openai.widget`

```js
t.widget = {
  state: widgetState,
  props: toolResponseMetadata,
  setState: c => {
    console.warn("window.openai.widget is deprecated. It will be removed in future versions");
    return H()?.updateWidgetState?.call(H(), widgetId, c);
  }
};
```

**Устарело, но работает.** Заменяется на `setWidgetState`.

---

## 8. Метаданные только для first-party

```js
t.appPackageName              = isFirstParty ? appPackageName : void 0;
t.supportsOAuthCompletion     = isFirstParty && supportsOAuthCompletion === true;
t.supportsSourceCitationLinks = isFirstParty && supportsSourceCitationLinks === true;
```

**Три поля обнуляются для обычных виджетов:**
- `appPackageName` — имя пакета приложения (Android package)
- `supportsOAuthCompletion` — умеет ли завершать OAuth-поток
- `supportsSourceCitationLinks` — умеет ли показывать ссылки на источники

Значит **обычный виджет не может участвовать в OAuth-потоке** и не может
показывать цитаты.

---

## 9. Порядок инициализации (полный)

```
1. Q(csp)                          — применить CSP, ослабить нельзя
2. (mcpApps || enableAutofill) && X()   — очистить старые iframe
3. P = prepareWidgetDocument       — приватный метод (first-party)
4. t = Z({isFirstParty, includePrivate: false, ...})   — Host API
5. t.__appBlockMessenger = ...     — мост, если есть
6. Обогатить t: displayMode, maxHeight, theme, locale, userAgent, ...
7. t.widget = {...}                — устаревший API
8. l = Ms({widgetId, globals, ...})     — widget state
9. Object.assign(t, l, Ps({displayMode, viewParams, isTombstone}))
10. s = mcpApps?.requiresInitialization ? Promise.withResolvers() : undefined
11. yield* Dr({... onIframeReady, appInitialization: s.promise, prepareDocument: P})
```

Внутри `Dr()` → `onIframeReady`:
```
12. enableAutofill → A({document, viewWindow})  — сканер автозаполнения
13. mcpApps.globalSurface → te({...})           — синхронизация фона
14. mcpApps → ee({...})                          — MCP adapter
15. return cleanup
```

---

## 10. Что виджет получает на выходе

```js
window.openai = {
  // ── данные ────────────────────────────────────
  toolInput, toolOutput, toolResponseMetadata,
  widgetState, subjectId, setWidgetState,

  // ── UI ────────────────────────────────────────
  displayMode,
  view: {params, mode, isTombstone},
  maxHeight, maxWidth,
  theme, locale, userAgent,
  isSidebarOpen,

  // ── мост ──────────────────────────────────────
  __appBlockMessenger,          // если есть

  // ── Host API ──────────────────────────────────
  // 8 клик-методов + 31 скрытый (только first-party)

  // ── устаревшее ────────────────────────────────
  widget: {state, props, setState}
};

window.oai     = window.openai;
window.webplus = window.openai;

window.messenger = {bindings: {get, subscribe}};
window.__bindings = Proxy (read-only);
window.__effect = fn => cleanup;
```

---

## 11. Слабые места в этом файле

### 1. `t.widget.setState` вызывает `H()` заново

```js
setState: c => H()?.updateWidgetState?.call(H(), widgetId, c)
```

`H()` = `hn({isFirstParty: true, includePrivate: true})` — **возвращает Host API
полностью приватным**. То есть даже обычный виджет через устаревший
`widget.setState` получает **приватный** `updateWidgetState`.

**Практически:** это тот же метод, что и публичный (`updateWidgetState` есть
в `Jr`, но вызывается через `t.widget.setState` — то есть обход фильтра).

### 2. `catch {}` проглатывает ошибки сборки глобалов

```js
try {
  ...
} catch {}
```

Если сборка `t` упала — виджет получит **пустой** `additionalGlobals`, и
`window.openai` останется неинициализированным. Без единого сообщения.

### 3. `forwardConsoleLogs: !mcpApps`

Для MCP-приложений консоль не форвардится. То есть из MCP-виджета хост
не увидит `console.log`.

### 4. `disableMutationObserver: !expectReadySignal && isFirstParty && !measureWidth`

Только для first-party с `expectReadySignal` — MutationObserver отключается.
Обычный виджет всегда с наблюдателем.

---

## 12. Итог

`run-widget-code` — это **клей**:

```
apply-csp       → применить политику
main.js         → Host API, runHTMLAsync, widget state, автозаполнение
adapter         → MCP-протокол
```

**36 параметров** превращают плоский HTML-файл в полноценное приложение
с доступом к хосту.

**Ключевое, что даёт этот файл:** глобалы `window.openai` собираются здесь,
и именно здесь решается, что попадёт в виджет — с учётом `isFirstParty`.
