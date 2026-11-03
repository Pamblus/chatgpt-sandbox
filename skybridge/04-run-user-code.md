# `run-user-code-BcXlYlun.js` — полный разбор

**Размер:** 2 590 байт
**URL:** `https://web-sandbox.oaiusercontent.com/assets/run-user-code-BcXlYlun.js`
**Роль:** диспетчер языков. Принимает код + язык, выбирает раннер, собирает глобалы.

---

# Часть 1. Контекст: что нужно понять до чтения

## 1.1 Где этот файл в архитектуре

```
Хост (ChatGPT)
   │  вызывает через фасад wt (из main.js)
   ▼
main.js → $s({ runUserCode })          ← фасад
   │  _t(() => import("run-user-code"), deps)   ← ленивая загрузка
   ▼
run-user-code.js   ◄── ЭТОТ ФАЙЛ
   │
   ├── react      → runReact     → транспиляция → <script type="module">
   ├── recharts   → runRecharts  → транспиляция → <script type="module">
   ├── python     → runPythonInWorker → Pyodide в Worker
   ├── threejs    → runThree     → WebGL в iframe
   ├── html       → runHTMLAsync → document.write
   └── javascript → (ничего)
```

## 1.2 Что такое `__vite__mapDeps`

Первая строка файла:

```js
const __vite__mapDeps = (i, m = __vite__mapDeps, d = (m.f || (m.f = [
  "assets/run-react-vAjeCFay.js",              // [0]
  "assets/main-BR1bOQyH.js",                   // [1]
  "assets/main-foMHhJws.css",                  // [2]
  "assets/transpile-Cld0n4VL.js",              // [3]
  "assets/get-html-D9pLuxCy.js",               // [4]
  "assets/parse-transpile-error-CaCAx2Mb.js",  // [5]
  "assets/run-recharts-CQOvWM4G.js",           // [6]
  "assets/run-three-BJvPLrbj.js"               // [7]
]))) => i.map(i => d[i]);
```

**Построчно:**

1. `i` — массив индексов, например `[0, 1, 2, 3, 4, 5]`
2. `m = __vite__mapDeps` — ссылка на саму функцию
3. `d = (m.f || (m.f = [...]))` — **ленивый кэш**: при первом вызове создаётся
   массив путей и сохраняется в `m.f`. При повторных — берётся из кэша
4. `i.map(i => d[i])` — превращает индексы в пути

**Результат:** `__vite__mapDeps([0,1,2])` → `["assets/run-react-vAjeCFay.js", "assets/main-BR1bOQyH.js", "assets/main-foMHhJws.css"]`

**Зачем:** Vite генерирует такие массивы, чтобы при preload передавать
**имена файлов**, а не хардкодить их в каждом вызове.

## 1.3 Что такое `_t` — функция предзагрузки

Импортируется как `_` (в файле `n`). Полный код из `main.js`:

```js
_t = function (e, n, r) {
  // e — функция загрузки (async () => import(...))
  // n — массив путей для preload
  // r — context (не используется)
  let o = Promise.resolve();

  if (n && n.length > 0) {
    document.getElementsByTagName("link");   // ← пустой вызов, мусор от сборщика

    // ищем nonce для CSP
    const i = document.querySelector("meta[property=csp-nonce]");
    const l = i?.nonce || i?.getAttribute("nonce");

    // для каждого пути — создать <link rel=modulepreload>
    o = Promise.allSettled(n.map(a => {
      a = Br(a);                    // "assets/x.js" → "/assets/x.js"
      if (a in An) return;          // уже грузили — пропустить
      An[a] = true;                 // отметить в кэше

      const u = a.endsWith(".css");
      const h = u ? '[rel="stylesheet"]' : "";

      // если уже в DOM — пропустить
      if (document.querySelector(`link[href="${a}"]${h}`)) return;

      const d = document.createElement("link");
      d.rel = u ? "stylesheet" : "modulepreload";
      if (!u) d.as = "script";
      d.crossOrigin = "";           // анонимный CORS
      d.href = a;
      l && d.setAttribute("nonce", l);
      document.head.appendChild(d);

      // для CSS — ждать загрузки
      if (u) return new Promise((g, E) => {
        d.addEventListener("load", g);
        d.addEventListener("error", () => E(new Error(`Unable to preload CSS for ${a}`)));
      });
    }));
  }

  // обработчик ошибок preload
  function s(i) {
    const l = new Event("vite:preloadError", {cancelable: true});
    l.payload = i;
    window.dispatchEvent(l);
    if (!l.defaultPrevented) throw i;
  }

  // дождаться preload → выполнить загрузку
  return o.then(i => {
    for (const l of i || [])
      l.status === "rejected" && s(l.reason);
    return e().catch(s);
  });
};
```

### Разбор по частям

**`Br(a)`** — префикс пути:
```js
Br = function (t) { return "/" + t; };
```

**`An`** — глобальный кэш уже запрошенных preload:
```js
An = {};
```
При повторном вызове с теми же путями `<link>` не создаётся заново.

**`allSettled`, а не `all`** — не падает, если один файл не загрузился.

**CSS ждётся, JS — нет.** `<link rel=modulepreload>` не даёт события `load`
в нужном смысле, поэтому для JS возвращается `undefined` (промис сразу
resolved). Для CSS — явный `Promise`, ждущий `load`/`error`.

**`vite:preloadError`** — кастомное событие, можно отловить:
```js
window.addEventListener("vite:preloadError", e => {
  console.log("Не удалось предзагрузить:", e.payload);
  e.preventDefault();   // не пробрасывать исключение
});
```

**`meta[property=csp-nonce]`** — если CSP требует nonce для `<link>`,
он будет подставлен.

## 1.4 Что такое `T` — обёртка callback → async generator

Импортируется как `c` (в файле `T`). Полный код из `main.js`:

```js
function ni(t) {
  let e = false;         // done
  const n = [];          // очередь ожидающих next()
  const r = [];          // буфер полученных значений

  // отправить значение или в буфер
  function o(a) {
    n.length > 0 ? n.shift()(a) : r.push(a);
  }

  // emit
  function s(a) {
    e || o({value: a, done: false});
  }

  // завершить
  function i() {
    e || (e = true,
      n.forEach(a => a({value: void 0, done: true})),
      n.length = 0);
  }

  // ошибка
  function l(a) {
    e || (o(Promise.reject(a)), i());
  }

  // запустить пользовательскую функцию
  try { t(s, i, l); } catch (a) { l(a); }

  return {
    next() {
      return new Promise(a => {
        r.length > 0 ? a(r.shift())            // есть буфер → отдать
          : e ? a({value: void 0, done: true}) // завершён
              : n.push(a);                     // ждать
      });
    },
    [Symbol.asyncIterator]() { return this; }
  };
}
```

### Как это работает

**Проблема:** есть callback-API (Python-воркер шлёт через `onmessage`), но
нужен `yield*` в async generator.

**Решение:** `ni()` создаёт мост:

```js
yield* T((emit, done, fail) => {
  runPythonInWorker(code, logger, msg => {
    emit(msg);                              // → yield
    if (msg.type === RUN_COMPLETE) done();  // → return
  });
});
```

**Поток значений:**
1. `runPythonInWorker` вызывает `emit(msg)` → `s(msg)` → `o({value: msg, done: false})`
2. Если кто-то ждёт (`next()` уже вызван) — отдаётся сразу
3. Если нет — кладётся в буфер `r`
4. `yield*` получает значение

**Проблема буфера:** если вызывающий перестал вызывать `next()`, значения
копятся в `r` **без ограничения** — утечка памяти при долгом Python-скрипте.

---

# Часть 2. Сам файл — построчный разбор

## 2.1 Импорты

```js
import {M as f, g as R, _ as n, c as T} from "./main-BR1bOQyH.js";
import {a as A} from "./apply-csp-_9OvJz9C.js";
```

| Локальное | Откуда | Что это |
|---|---|---|
| `f` | `M` из main | `bt` — типы событий (`RUN_START`, `RUN_COMPLETE`, ...) |
| `R` | `g` из main | `hn()` — публикация Host API |
| `n` | `_` из main | `_t()` — предзагрузка + динамический импорт |
| `T` | `c` из main | `ni()` — обёртка callback → async generator |
| `A` | `a` из apply-csp | `N()` — применить CSP, вернуть `true` если ужесточилась |

## 2.2 Сигнатура

```js
async function* v(e, {
  code: a, csp: o, language: P, id: _,
  additionalGlobals: l, expectReadySignal: k,
  ownerColorScheme: m, frameColorScheme: w,
  enableExternalNavigationGating: c = false,
  disableExternalNavigation: d = false,
  allowPopups: y = true
})
```

| Параметр | Тип | Назначение |
|---|---|---|
| `e` | `SandboxLogger` | логгер (первый аргумент, вне объекта) |
| `code` | string | исходный код |
| `csp` | object \| undefined | конфиг CSP |
| `language` | string | см. таблицу языков |
| `id` | string | runId (для событий) |
| `additionalGlobals` | object | что домешать в `window.openai` |
| `expectReadySignal` | bool | ждать `data-ready` в HTML-ветке |
| `ownerColorScheme` | `"light"`\|`"dark"` | тема владельца |
| `frameColorScheme` | `"light"`\|`"dark"` | тема фрейма (приоритетнее) |
| `enableExternalNavigationGating` | bool | перехватывать переходы |
| `disableExternalNavigation` | bool | запретить переходы наружу |
| `allowPopups` | bool | разрешить `window.open` |

## 2.3 Шаг 1 — CSP

```js
let p = false;
try {
  p = A(csp);
} catch (t) {
  throw (e.error("Failed to apply CSP", t), t);
}
```

**Что делает `A(csp)`:** строит `<meta http-equiv="Content-Security-Policy">`,
сравнивает с текущей, и:

- новая **слабее** → `throw new Error("CSP is being loosened")`
- **такая же** → `return false`
- **строже** → применяет, `return true`

**`p`** (`true` только если политика изменилась) — **используется один раз**,
в Python-ветке.

**`e.error(...)`** — через `SandboxLogger.error()` отправляет
`sendInstrument({type: "error", ...})` хосту.

**`throw (expr, t)`** — comma-оператор: выполнить `e.error(...)`, потом бросить `t`.

## 2.4 Шаг 2 — событие старта

```js
yield {
  type: f.RUN_START,          // "run_start"
  runId: _,                   // переданный id
  id: crypto.randomUUID(),    // новый UUID
  timestamp: Date.now()
};
```

**`crypto.randomUUID()`** требует **secure context** (`https:` или `localhost`).
В `https://web-sandbox...` — работает.

**Если недоступен:** `TypeError: crypto.randomUUID is not a function`.
**Fallback отсутствует** — генератор упадёт на первом `yield`.

**`runId` vs `id`:** `runId` пришёл извне (идентифицирует запуск),
`id` — новый UUID (идентифицирует именно этот вызов генератора).

## 2.5 Шаг 3 — сборка глобалов

```js
let s;
try {
  s = R({isFirstParty: false, includePrivate: false});
  if (l) for (const [t, r] of Object.entries(l)) s[t] = r;
} catch {}
```

**Что делает `R({isFirstParty: false, includePrivate: false})`** = `hn(...)`:

```js
hn = ({isFirstParty, includePrivate = false, ...}) => {
  if (!Je) throw new Error("Host API not published");
  return Object.fromEntries(
    Object.keys(Je)
      .filter(o => (o !== "prepareWidgetDocument" || includePrivate)
                && (isFirstParty || includePrivate || !Jr.has(o)))
      .map(o => [o, (...i) =>
        !isFirstParty && ao.has(o) && !navigator.userActivation.isActive
          ? console.warn(`Method \`${o}\` called without synchronous user event`)
          : ho(o, Je, i)
      ])
  );
};
```

**С `isFirstParty: false`:**
- `prepareWidgetDocument` — **отфильтрован** (нужен `includePrivate`)
- 31 метод из `Jr` — **отфильтрованы**
- 8 методов из `ao` — **остаются**, но с проверкой `userActivation.isActive`

**Результат `s`:**
```js
{
  requestDisplayMode, sendFollowUpMessage, openExternal, openExternalForm,
  openPlaidLink, requestPrompt, requestTargetedReply, triggerHaptic,
  // + методы хоста, не входящие в Jr (например notifyIntrinsicWidth)
}
```

**`catch {}`** — молча. Если `R()` бросит `"Host API not published"`
(хост не инициализирован), `s` останется `undefined`.

**Последствие:** раннер получит `additionalGlobals: undefined` →
`window.openai` не будет создан. **Без единого сообщения об ошибке.**

## 2.6 Ветка `react`

```js
case "react": {
  const {runReact: t} = await n(
    async () => {
      const {runReact: r} = await import("./run-react-vAjeCFay.js");
      return {runReact: r};
    },
    __vite__mapDeps([0, 1, 2, 3, 4, 5])
  );
  yield* t({
    id: _, code: a, sandboxLogger: e, additionalGlobals: s,
    enableExternalNavigationGating: c, allowPopups: y, worker: true
  });
  break;
}
```

**Пошагово:**

1. `n(...)` = `_t(...)` — preload 6 файлов:
   ```
   [0] run-react-vAjeCFay.js      ← раннер
   [1] main-BR1bOQyH.js           ← ядро
   [2] main-foMHhJws.css          ← стили
   [3] transpile-Cld0n4VL.js      ← транспилер (3.5 МБ)
   [4] get-html-D9pLuxCy.js       ← шаблон
   [5] parse-transpile-error...   ← парсер ошибок
   ```
2. `import("./run-react-vAjeCFay.js")` — загрузка модуля
3. `yield* t({...})` — делегирование генератору `runReact`

**`worker: true`** — транспиляция в Web Worker, не блокирует UI.

**Зачем preload:** когда пользователь жмёт «запустить», всё уже в кэше.
Первый запуск не ждёт 3.5 МБ.

## 2.7 Ветка `recharts`

```js
case "recharts": {
  const {runRecharts: t} = await n(
    async () => {
      const {runRecharts: r} = await import("./run-recharts-CQOvWM4G.js");
      return {runRecharts: r};
    },
    __vite__mapDeps([6, 1, 2, 0, 3, 4, 5])
  );
  yield* t({
    id: _, code: a, sandboxLogger: e, additionalGlobals: s,
    enableExternalNavigationGating: c, allowPopups: y, worker: true
  });
  break;
}
```

**Preload 7 файлов, другой порядок:**
```
[6] run-recharts-CQOvWM4G.js   ← речартс
[1] main-BR1bOQyH.js
[2] main-foMHhJws.css
[0] run-react-vAjeCFay.js      ← React (recharts строится на нём)
[3] transpile-Cld0n4VL.js
[4] get-html-D9pLuxCy.js
[5] parse-transpile-error...
```

**Почему `[0]`:** Recharts — React-библиотека. Без React-раннера не отрисуется.

## 2.8 Ветка `python`

```js
case "python": {
  const {runPythonInWorker: t, terminatePythonWorker: r} = await n(
    async () => {
      const {runPythonInWorker: i, terminatePythonWorker: h} =
        await import("./main-BR1bOQyH.js").then(u => u.m);
      return {runPythonInWorker: i, terminatePythonWorker: h};
    },
    __vite__mapDeps([1, 2])
  );

  p && r();

  yield* T((i, h) => t(a, e, u => {
    i(u);
    u.type === f.RUN_COMPLETE && h();
  }));

  break;
}
```

**Пошагово:**

1. Preload 2 файла: main + css
2. `import(...).then(u => u.m)` — берётся **экспорт `m`**:
   ```js
   bi = Object.freeze({runPythonInWorker: Gs, terminatePythonWorker: Rr})
   ```
3. `p && r()` — **если CSP ужесточилась** → убить старый воркер
4. `yield* T(...)` — обернуть callback в генератор

**`runPythonInWorker`** (из main.js):

```js
Gs = async (code, logger, onMessage) => {
  try { Lt ??= await js(); }                 // ← ОДИН воркер на всё
  catch (r) { logger.error("failed to create python worker", r); }

  return new Promise((resolve, reject) => {
    const s = Lt;
    if (!s) return reject(new Error("Python worker is unavailable"));

    let i = false;
    const onDone = () => { i || (i = true, logger.timeEnd("python.evaluation"), resolve()) };

    const h = g => {
      if (g.data.type === "security_policy_violation") {
        X().notifySecurityPolicyViolation(g.data.payload);
        return;                                  // ← не идёт дальше
      }
      onMessage(g.data);
      g.data.type === "run_complete" && (cleanup(), onDone());
    };

    s.onmessage = h;                             // ← перезаписывает!
    s.postMessage({code});
  });
};
```

**Ключевые моменты:**

- `Lt ??= await js()` — **один воркер** на всё. Переменные Python сохраняются.
- `s.onmessage = h` — **перезаписывает** обработчик. Два параллельных запуска
  → второй перезапишет первый.
- `security_policy_violation` — отдельный тип, идёт в хост.

**Почему `p && r()`:** старый воркер создан под прежнюю CSP. Если политика
ужесточилась — воркер всё равно имеет старые права. Убийство + пересоздание.

**Что теряется:** все переменные Python (импорты, данные).

## 2.9 Ветки `javascript` / `typescript`

```js
case "javascript":
case "typescript":
  break;
```

**Полный разбор:** ничего не происходит.

- Нет `import()`
- Нет preload
- Нет `yield*`
- Просто `break`

**Поток событий для `language: "javascript"`:**
1. `yield RUN_START`
2. `break` → функция завершается

**Что видит вызывающий:** только `RUN_START`, потом конец. Ни `RUN_COMPLETE`,
ни `ERROR`.

**Если язык неизвестен** (`"rust"`): `switch` не найдёт ветку, тот же результат —
только `RUN_START`.

**`default` отсутствует.** Нет обработки неизвестного языка.

## 2.10 Ветка `threejs`

```js
case "threejs": {
  const {runThree: t} = await n(
    async () => {
      const {runThree: r} = await import("./run-three-BJvPLrbj.js");
      return {runThree: r};
    },
    __vite__mapDeps([7, 1, 2])
  );
  yield* t({
    code: a, sandboxLogger: e,
    enableExternalNavigationGating: c, allowPopups: y
  });
  break;
}
```

**Preload 3 файла:** раннер, main, css.

**Что НЕ передаётся:**

| Параметр | react | threejs |
|---|---|---|
| `id` | ✅ | ❌ |
| `additionalGlobals` | ✅ | ❌ |
| `worker` | ✅ | ❌ |

**Последствие:** в threejs-виджете **нет `window.openai`** через этот путь.

## 2.11 Ветка `html`

```js
case "html": {
  const {runHTMLAsync: t} = await n(
    async () => {
      const {runHTMLAsync: r} = await import("./main-BR1bOQyH.js").then(i => i.p);
      return {runHTMLAsync: r};
    },
    __vite__mapDeps([1, 2])
  );

  yield* t({
    html: a,
    sandboxLogger: e,
    transformPosition: null,
    measureHeight: true,
    measureWidth: false,
    additionalGlobals: s,
    expectReadySignal: k,
    theme: m,
    ownerColorScheme: w ?? m,
    enableExternalNavigationGating: c,
    disableExternalNavigation: d || (o?.disableNetworkRequests),
    allowPopups: y && !d
  });

  break;
}
```

**`import(...).then(i => i.p)`** — экспорт `p` = `_i`:
```js
_i = Object.freeze({runHTMLAsync: Dr})
```
То есть `runHTMLAsync` = `Dr()` из `main.js` — создаёт iframe и делает
`document.write(html)`.

### Разбор каждого параметра

**`html: a`** — HTML пишется напрямую, без транспиляции.

**`transformPosition: null`** — маппинг позиций для sourcemap отключён.
Для HTML нет JSX.

**`measureHeight: true`** — после рендера измерить высоту → `notifyIntrinsicHeight`.

**`measureWidth: false`** — ширину не мерить, растягивается по контейнеру.

**`additionalGlobals: s`** — здесь **есть** глобалы (в отличие от threejs).
Значит HTML-виджет получит `window.openai`.

**`expectReadySignal: k`** — `true` → ждать `data-ready`; `false` → `DOMContentLoaded`.

**`theme: m`** — `ownerColorScheme`.

**`ownerColorScheme: w ?? m`** — `frameColorScheme ?? ownerColorScheme`:
```
frameColorScheme задан  → использовать его
иначе                   → ownerColorScheme
```
`??` срабатывает только на `null`/`undefined`. Если `frameColorScheme === "light"`,
возьмётся `"light"`, даже если `ownerColorScheme === "dark"`.

**Почему `w ?? m`, а не `m ?? w`:** тема фрейма приоритетнее темы владельца.

**`disableExternalNavigation: d || (o?.disableNetworkRequests)`** — **автоматика**:
```js
d === true                              → true
d === undefined, csp.disableNetworkRequests === true  → true
d === undefined, csp.disableNetworkRequests === undefined → undefined
```

**`allowPopups: y && !d`** — попапы только если запрошены и навигация не
запрещена явно:
```js
y=true,  d=false     → true
y=true,  d=true      → false
y=false, d=false     → false
y=true,  d=undefined → true   (т.к. !undefined === true)
```

**Важный нюанс:** `d || (o?.disableNetworkRequests)` при `d=undefined` и
`csp=undefined` даёт `undefined`, а не `false`. То есть параметр остаётся
`undefined` — «не задан».

---

# Часть 3. Полная схема вызовов

## 3.1 react

```
runUserCode(logger, {language: "react", code, ...})
 ├─ A(csp)                         → применить CSP
 ├─ yield RUN_START
 ├─ R({isFirstParty:false})        → 8 методов + whitelist
 ├─ _t(import run-react, [0,1,2,3,4,5])
 │   ├─ <link modulepreload> × 5
 │   ├─ <link stylesheet> × 1
 │   └─ await import()             → {runReact}
 └─ yield* runReact({...})
     ├─ transpileReact(worker)     → esbuild-wasm
     ├─ getHtml()                  → HTML-строка
     ├─ Dr({html, ...})
     │   ├─ создать iframe
     │   ├─ document.open()
     │   ├─ Xt() — глобалы
     │   ├─ document.write(html)
     │   ├─ MutationObserver → data-ready
     │   └─ yield ENVIRONMENT_STATUS.RUNNING_CODE
     └─ yield RUN_COMPLETE
```

## 3.2 python

```
runUserCode(logger, {language: "python", code, ...})
 ├─ A(csp) → p
 ├─ yield RUN_START
 ├─ R(...)
 ├─ _t(import main.m, [1,2])
 ├─ p && terminatePythonWorker()     ← если CSP изменилась
 └─ yield* ni((emit, done, fail) => {
      runPythonInWorker(code, logger, msg => {
        emit(msg)
        msg.type === "run_complete" && done()
      })
    })
     ├─ Lt ??= new Worker(blob)
     ├─ worker.postMessage({code})
     ├─ loadPyodide(0.26.4)          ← cdn.jsdelivr.net
     ├─ loadPackagesFromImports()
     ├─ micropip.install()           ← PyPI
     ├─ runPython
     └─ postMessage({type: "run_complete"})
```

## 3.3 html

```
runUserCode(logger, {language: "html", code, ...})
 ├─ A(csp)
 ├─ yield RUN_START
 ├─ R(...)
 ├─ _t(import main.p, [1,2])
 └─ yield* Dr({html: code, ...})
     ├─ создать iframe (sandbox по allowPopups)
     ├─ document.open()
     ├─ Xt() — глобалы
     ├─ document.write(code)
     ├─ ResizeObserver → notifyIntrinsicHeight
     ├─ MutationObserver → data-ready
     └─ yield ENVIRONMENT_STATUS.RUNNING_CODE
```

---

# Часть 4. Что виджет получает

## 4.1 react / recharts / html

```js
window.openai = {
  // 8 клик-методов (проверка userActivation)
  requestDisplayMode, sendFollowUpMessage, openExternal, openExternalForm,
  openPlaidLink, requestPrompt, requestTargetedReply, triggerHaptic,

  // методы хоста вне Jr
  notifyIntrinsicWidth,

  // + additionalGlobals из runWidgetCode
  toolInput, toolOutput, toolResponseMetadata,
  widgetState, subjectId, setWidgetState,
  displayMode, view, maxHeight, maxWidth, theme, locale, userAgent,
};
```

## 4.2 python

**Python-воркер не имеет DOM.** Глобалы основного контекста в Worker
не проникают. Вызовы `openai.*` из Python невозможны.

## 4.3 threejs

```js
window.openai = ???    // additionalGlobals НЕ передан
```

Виджет получит только то, что успел проставить `runWidgetCode`.

---

# Часть 5. Примеры

## 5.1 React

```jsx
import { useState } from "react";
import { Button } from "@/components/ui/button";

export default function App() {
  const [n, setN] = useState(0);
  return (
    <div style={{padding: 24, textAlign: "center"}}>
      <h1 style={{fontSize: 32}}>{n}</h1>
      <Button onClick={() => setN(n + 1)}>Плюс один</Button>
      <Button variant="outline"
        onClick={() => openai.sendFollowUpMessage(`Число: ${n}`)}>
        Отправить в чат
      </Button>
    </div>
  );
}
```

**Что произойдёт:**
1. JSX → esbuild-wasm в Worker
2. `import {Button}` → вшитый компонент (248 в `transpile`)
3. `getHtml()` → HTML
4. `Dr()` → iframe → `document.write()`
5. `window.openai.sendFollowUpMessage` доступен

## 5.2 Python с графиком

```python
import matplotlib.pyplot as plt
import numpy as np

x = np.linspace(0, 2*np.pi, 100)
plt.figure(figsize=(6, 3))
plt.plot(x, np.sin(x), label="sin")
plt.plot(x, np.cos(x), label="cos")
plt.legend()
plt.title("Тригонометрия")
plt.show()
```

**Что произойдёт:**
1. `loadPackagesFromImports` увидит `matplotlib`, `numpy`
2. `micropip.install("matplotlib")`, `micropip.install("numpy")` — с PyPI
3. `plt.show()` → перехвачено → PNG через `savefig`
4. PNG → base64 → `data:image/png;base64,...`
5. Хост получает `output` с картинкой

## 5.3 HTML с интерактивом

```html
<div style="font-family:system-ui;padding:24px">
  <button id="b" style="padding:12px 24px;font-size:18px">0</button>
  <script>
    let n = 0;
    document.getElementById('b').onclick = () => {
      document.getElementById('b').textContent = ++n;
    };
  </script>
</div>
```

**Что произойдёт:**
1. `document.write(html)` — включая `<script>`
2. CSP `'unsafe-inline'` — скрипт выполнится
3. `onclick` сработает
4. Высота замеряется → хосту

---

# Часть 6. Что ломается

## 6.1 `crypto.randomUUID()` без secure context

```
TypeError: crypto.randomUUID is not a function
```

**Условие:** `window.isSecureContext === false`.

**Последствие:** падение на первом `yield`. Ни `RUN_START`, ни `RUN_COMPLETE`.

**Fallback отсутствует.**

## 6.2 `catch {}` при сборке глобалов

```js
try { s = R({...}); } catch {}
```

**Если `R()` бросит:**
- `Je === null` → `"Host API not published"`
- Причина: хост не вызвал `lo(t)` (публикацию API)

**Последствие:** `s = undefined` → нет `window.openai`. Молча.

## 6.3 Параллельный запуск Python

```js
s.onmessage = h;   // ← перезаписывает
```

**Два вызова одновременно:**
1. Первый ставит `onmessage = h1`
2. Второй — `h2` (перезаписал)
3. Результат первого → обрабатывается `h2` → **первый resolve не случится**

**Последствие:** первый промис висит вечно.

## 6.4 `javascript` / `typescript` молча ничего не делают

```
runUserCode({language: "javascript", code: "..."})
→ yield RUN_START
→ return
```

**Ни ошибки, ни предупреждения.**

## 6.5 `threejs` без глобалов

```js
yield* runThree({code: a, sandboxLogger: e, ...});
//                                    ↑ нет additionalGlobals
```

**Виджет не получит `window.openai` через этот путь.**

## 6.6 Python-воркер убивается при любом изменении CSP

```js
p && terminatePythonWorker();
```

`p` = «CSP изменилась», а не «стала строже по сети». Даже если изменился
только `img-src` — воркер убивается, переменные Python теряются.

---

# Часть 7. Итоговая таблица

| Язык | Раннер | Preload | worker | Глобалы | Измерения | Что ломается |
|---|---|---|---|---|---|---|
| `react` | `runReact` | 6 | ✅ | ✅ | h+w | — |
| `recharts` | `runRecharts` | 7 | ✅ | ✅ | — | — |
| `python` | `runPythonInWorker` | 2 | — | ✅ (но не видно в Worker) | — | параллельный запуск |
| `javascript` | — | — | — | — | — | **ничего не делает** |
| `typescript` | — | — | — | — | — | **ничего не делает** |
| `threejs` | `runThree` | 3 | — | ❌ | — | нет `window.openai` |
| `html` | `runHTMLAsync` | 2 | — | ✅ | h | — |

---

# Часть 8. Сводка по безопасности

| Что | Состояние |
|---|---|
| `isFirstParty` | жёстко `false` — приватный API недоступен |
| `prepareWidgetDocument` | отфильтрован |
| 31 метод `Jr` | отфильтрованы |
| 8 методов `ao` | доступны, с проверкой `userActivation` |
| `notifyIntrinsicWidth` | **не в `Jr`** → доступен всем |
| CSP | применяется до всего, ослабить нельзя |
| `disableNetworkRequests` | автоматически тянет `disableExternalNavigation` |
| Python-воркер | общий, переиспользуется, переменные сохраняются |
| Молчаливые провалы | `catch {}`, пустые ветки JS/TS |
