# `vision_bundle-CPRQl8Yc.js` — полный разбор (MediaPipe Tasks Vision)

**Размер:** 131 944 байта
**URL:** `https://web-sandbox.oaiusercontent.com/assets/vision_bundle-CPRQl8Yc.js`
**Роль:** **MediaPipe Tasks Vision** — 11 компьютерных-зрительных задач
(лицо, руки, поза, сегментация, классификация, эмбеддинги).

---

# ЧАСТЬ 1. Что это

## 1.1 Назначение

**MediaPipe** — фреймворк Google для ML-задач реального времени в браузере.
Здесь — **`@mediapipe/tasks-vision`**: обёртка над WebAssembly-графами.

**11 задач экспортируется:**

| Экспорт | Класс | Что делает |
|---|---|---|
| `FaceDetector` | `lt` | детекция лиц |
| `FaceLandmarker` | `R` | 478 точек лица |
| `FaceStylizer` | `St` | стилизация лица |
| `GestureRecognizer` | `et` | жесты рук |
| `HandLandmarker` | `nt` | 21 точка кисти |
| `HolisticLandmarker` | `P` | лицо + руки + поза вместе |
| `ImageClassifier` | `ft` | классификация изображений |
| `ImageEmbedder` | `rt` | эмбеддинги |
| `ImageSegmenter` | `q` | сегментация |
| `InteractiveSegmenter` | `Lt` | сегментация по клику |
| `ObjectDetector` | `dt` | детекция объектов |
| `PoseLandmarker` | `st` | 33 точки скелета |

**Плюс:**
- `FilesetResolver` (`he`) — загрузчик WASM
- `MPImage` (`z`) — обёртка изображения
- `MPMask` (`W`) — маска
- `VisionTaskRunner` (`J`) — базовый класс
- `ImageSegmenterResult`, `InteractiveSegmenterResult`

## 1.2 Зависимости

**Внутри — 4 слоя:**

| Слой | Что |
|---|---|
| `protobuf.js` | сериализация (Message Types) |
| **WASM-бандл** | C++ графы MediaPipe |
| **Tasks API** | JS-обёртки (`FaceDetector`, `HandLandmarker`, ...) |
| **Утилиты** | TextEncoder, UTF-8, canvas |

**Внешние WASM загружаются отдельно** — этот файл только JS-обёртка.

**Размер WASM:** `vision_wasm_internal.wasm` ~10 МБ (не в этом файле).

---

# ЧАСТЬ 2. WASM-загрузчик

## 2.1 `pa()` — проверка SIMD

```js
async function pa() {
  if (Ze === void 0)
    try {
      await WebAssembly.instantiate(J1);
      Ze = !0
    } catch {
      Ze = !1
    }
  return Ze
}
```

**`J1`** — тестовый WASM-модуль с SIMD-инструкциями.

**Результат:** `Ze` = `true` (SIMD поддерживается) / `false` / `undefined`.

**Кэш:** проверка один раз.

## 2.2 `Oe()` — выбор варианта WASM

```js
async function Oe(t, e = "") {
  const n = await pa() ? "wasm_internal" : "wasm_nosimd_internal";

  return {
    wasmLoaderPath: `${e}/${t}_${n}.js`,
    wasmBinaryPath: `${e}/${t}_${n}.wasm`
  }
}
```

**Два варианта:**
- `wasm_internal` — с SIMD (быстрее, ~10 МБ)
- `wasm_nosimd_internal` — без SIMD (совместимость, ~15 МБ)

**Автовыбор по поддержке.**

## 2.3 `ga()` — выбор mode

```js
function ga() {
  var t = navigator;
  return typeof OffscreenCanvas < "u" && (
    !function (e = navigator) {
      return (e = e.userAgent).includes("Safari")
          && !e.includes("Chrome")
    }(t)
    || !!((t = t.userAgent.match(/Version\/([\d]+).*Safari/))
           && t.length >= 1 && Number(t[1]) >= 17)
  )
}
```

**Проверка «можно ли в Worker»:**
1. `OffscreenCanvas` доступен
2. **и** (не Safari **или** Safari ≥ 17)

**Safari < 17** не поддерживает `OffscreenCanvas` в Worker.

---

# ЧАСТЬ 3. WebGL / GPU

## 3.1 `ui()` — загрузка текстуры

```js
function ui(t, e, n) {
  if (!t.i.canvas) throw Error("No OpenGL canvas configured.");

  n ? t.i._bindTextureToStream(n) : t.i._bindTextureToCanvas();

  if (!(n = t.i.canvas.getContext("webgl2") || t.i.canvas.getContext("webgl")))
    throw Error("Failed to obtain WebGL context from the provided canvas. `getContext()` should only be invoked with `webgl` or `webgl2`.");

  t.i.gpuOriginForWebTexturesIsBottomLeft
    && n.pixelStorei(n.UNPACK_FLIP_Y_WEBGL, !0);

  n.texImage2D(n.TEXTURE_2D, 0, n.RGBA, n.RGBA, n.UNSIGNED_BYTE, e);

  t.i.gpuOriginForWebTexturesIsBottomLeft
    && n.pixelStorei(n.UNPACK_FLIP_Y_WEBGL, !1);

  const [r, s] = ma(e);

  return !t.l || r === t.i.canvas.width && s === t.i.canvas.height
    || (t.i.canvas.width = r, t.i.canvas.height = s),
    [r, s]
}
```

### Разбор

**`getContext("webgl2") || getContext("webgl")`** — предпочитает WebGL2.

**`UNPACK_FLIP_Y_WEBGL`** — если `gpuOriginForWebTexturesIsBottomLeft`.
WebGL-текстуры имеют origin внизу-слева, а изображения — вверху-слева.

**`texImage2D(...)`** — загрузка пикселей в текстуру.

**Ресайз canvas** — под размер изображения.

## 3.2 `li()` — мультипотоковая передача

```js
function li(t, e, n) {
  t.m || console.error("No wasm multistream support detected: ensure dependency inclusion of :gl_graph_runner_internal_multi_input target");

  const r = new Uint32Array(e.length);
  ...
}
```

**`t.m`** — поддержка multistream.

**Если нет** — `console.error` с подсказкой про target:
`:gl_graph_runner_internal_multi_input`.

**Это отладочное сообщение** — говорит, какой build-таргет нужен в C++.

## 3.3 `we()` — создание изображения

```js
function we(t, e, n, r) {
  var s = e.data;
  const i = e.width, o = i * (e = e.height);

  if ((s instanceof Uint8Array || s instanceof Float32Array) && s.length !== o)
    throw Error("Unsupported channel count: " + s.length / o);

  return t = new W([s], n, !1, t.g.i.canvas, t.R, i, e),
    r ? t.clone() : t
}
```

**Валидация:** длина данных = `width × height`.

**Если не совпадает** — «Unsupported channel count: N» (N = каналов).

**`new W([s], ...)`** — создать `MPImage`.

**`t.clone()`** — если `r = true` (клонировать).

---

# ЧАСТЬ 4. Базовый класс `on`

## 4.1 Конструктор

```js
var on = class {
  constructor(t) {
    this.g = t;              // graph runner
    this.H = [];             // callbacks
    this.B = 0;              // timestamp counter
    this.g.setAutoRenderToScreen(!1)
  }

  l(t, e = !0) { ... }       // setOptions
}
```

**`setAutoRenderToScreen(false)`** — отключить авто-рендер.

**Зачем:** задачи возвращают результаты, а не рисуют на canvas.

## 4.2 `l()` — `setOptions`

```js
l(t, e = !0) {
  var n, r, s, i, o, h;

  if (e) {
    const a = t.baseOptions || {};

    // 1. нельзя оба
    if (a?.modelAssetBuffer && a?.modelAssetPath)
      throw Error("Cannot set both baseOptions.modelAssetPath and baseOptions.modelAssetBuffer");

    // 2. нужен хотя бы один
    if (!(w(this.baseOptions, wn, 1)?.g()
          || w(this.baseOptions, wn, 1)?.h()
          || a?.modelAssetBuffer
          || a?.modelAssetPath))
      throw Error("Either baseOptions.modelAssetPath or baseOptions.modelAssetBuffer must be set");

    // 3. delegate
    (function (c, u) {
      let g = w(c.baseOptions, ii, 3);
      if (!g) {
        var v = g = new ii, ut = new Qs;
        Re(v, 4, sn, ut)
      }
      "delegate" in u && (
        u.delegate === "GPU"
          ? (u = g, v = new E1, Re(u, 2, sn, v))
          : (u = g, v = new Qs, Re(u, 4, sn, v))
      ),
      m(c.baseOptions, 0, 3, g)
    })(this, a);

    // 4. загрузка модели
    if (a.modelAssetPath)
      return fetch(a.modelAssetPath.toString()).then(c => {
        if (c.ok) return c.arrayBuffer();
        throw Error(`Failed to fetch model: ${a.modelAssetPath} (${c.status})`)
      }).then(c => {
        try { this.g.i.FS_unlink("/model.dat") } catch {}
        this.g.i.FS_createDataFile("/", "model.dat", ...);
        ...
      })
  }
}
```

### Разбор

**Валидация:**
- Оба `modelAssetPath` **и** `modelAssetBuffer` → ошибка
- Ни один → ошибка

**`delegate`:**
- `"GPU"` → `E1` (GPU-делегат)
- иначе → `Qs` (CPU)

**Загрузка модели:**
```js
fetch(modelAssetPath) → arrayBuffer → FS_createDataFile("/model.dat", ...)
```

**`FS_unlink`** — удалить старый файл (если был).

**`FS_createDataFile`** — поместить в виртуальную ФС Emscripten.

**Значит: модель — файл в памяти WASM.**

## 4.3 Проверка режима

```js
function vt(t, e, n) {
  if (t.baseOptions?.g())
    throw Error("Task is not initialized with image mode. 'runningMode' must be set to 'IMAGE'.");
  ba(t, e, n, t.B + 1)
}

function Ot(t, e, n, r) {
  if (!t.baseOptions?.g())
    throw Error("Task is not initialized with video mode. 'runningMode' must be set to 'VIDEO'.");
  ba(t, e, n, r)
}
```

**`vt`** — `detect(image)` — только для `IMAGE`-режима.
**`Ot`** — `detectForVideo(video, ts)` — только для `VIDEO`.

**`baseOptions.g()`** — проверка режима (внутренний флаг).

**Несовпадение → ошибка** с чётким сообщением.

---

# ЧАСТЬ 5. `FaceDetector` — построчно

## 5.1 Класс

```js
var lt = class extends J {
  constructor(t, e) {
    super(new _t(t, e), "image_in", "norm_rect_in", !1),
    this.j = {detections: []},
    m(t = this.h = new Vn, 0, 1, e = new O),
    d(this.h, 2, .5),      // minDetectionConfidence = 0.5
    d(this.h, 3, .3)       // minSuppressionThreshold = 0.3
  }

  get baseOptions() { return w(this.h, O, 1) }
  set baseOptions(t) { m(this.h, 0, 1, t) }

  o(t) {                   // setOptions
    "minDetectionConfidence" in t && d(this.h, 2, t.minDetectionConfidence ?? .5);
    "minSuppressionThreshold" in t && d(this.h, 3, t.minSuppressionThreshold ?? .3);
    return this.l(t)
  }

  D(t, e) {                // detect
    return this.j = {detections: []}, vt(this, t, e), this.j
  }

  F(t, e, n) {             // detectForVideo
    return this.j = {detections: []}, Ot(this, t, n, e), this.j
  }

  m() {                    // build graph
    var t = new tt;
    M(t, "image_in");
    M(t, "norm_rect_in");
    b(t, "detections");

    const e = new at;
    At(e, D1, this.h);

    const n = new $;
    ht(n, "mediapipe.tasks.vision.face_detector.FaceDetectorGraph");
    L(n, "IMAGE:image_in");
    L(n, "NORM_RECT:norm_rect_in");
    _(n, "DETECTIONS:detections");
    n.o(e);
    ct(t, n);

    this.g.attachProtoVectorListener("detections", (r, s) => {
      for (const i of r) r = Co(i), this.j.detections.push(da(r));
      l(this, s)
    });
    this.g.attachEmptyPacketListener("detections", r => { l(this, r) });

    t = t.g();
    this.setGraph(new Uint8Array(t), !0)
  }
}
```

## 5.2 Разбор

**`super(new _t(t, e), "image_in", "norm_rect_in", !1)`**:
- `_t` — `GraphRunner`
- `"image_in"` — входной поток изображения
- `"norm_rect_in"` — входной поток нормализованного прямоугольника
- `!1` — не синхронный

**`m(this.h, 0, 1, e)`** — установить `baseOptions`.
**`d(this.h, 2, .5)`** — `minDetectionConfidence = 0.5`.
**`d(this.h, 3, .3)`** — `minSuppressionThreshold = 0.3`.

**`w`** — getter поля protobuf. **`m`** — setter.
**`d`** — setter для double.

## 5.3 `m()` — построение графа

```js
var t = new tt;              // CalculatorGraphConfig
M(t, "image_in");            // input_stream
M(t, "norm_rect_in");        // input_stream
b(t, "detections");          // output_stream

const e = new at;            // FaceDetectorOptions
At(e, D1, this.h);           // + baseOptions

const n = new $;             // CalculatorGraphConfig.Node
ht(n, "mediapipe.tasks.vision.face_detector.FaceDetectorGraph");   // calculator
L(n, "IMAGE:image_in");      // input_stream: "IMAGE:image_in"
L(n, "NORM_RECT:norm_rect_in");
_(n, "DETECTIONS:detections");  // output_stream
n.o(e);                      // node_options
ct(t, n);                    // добавить node в config

this.g.attachProtoVectorListener("detections", (r, s) => {
  for (const i of r) r = Co(i), this.j.detections.push(da(r));
  l(this, s)                 // callback
});
this.g.attachEmptyPacketListener("detections", r => { l(this, r) });

t = t.g();
this.setGraph(new Uint8Array(t), !0)   // ← protobuf → Uint8Array
```

### Имя графа

```
mediapipe.tasks.vision.face_detector.FaceDetectorGraph
```

**Полное имя C++-класса** (calculator). По нему WASM находит
зарегистрированный граф.

### Потоки

```
IMAGE:image_in              (вход)
NORM_RECT:norm_rect_in      (вход)
DETECTIONS:detections       (выход)
```

**Формат:** `"TAG:stream_name"` — MediaPipe-style.

### Protobuf → бинарь

```js
t = t.g();
this.setGraph(new Uint8Array(t), !0)
```

**`t.g()`** — сериализация protobuf в `Uint8Array`.

**`setGraph(binary, true)`** — передать в WASM.

## 5.4 Статические фабрики

```js
lt.prototype.detectForVideo = lt.prototype.F;
lt.prototype.detect = lt.prototype.D;
lt.prototype.setOptions = lt.prototype.o;

lt.createFromModelPath = async function (t, e) {
  return y(lt, t, {baseOptions: {modelAssetPath: e}})
};

lt.createFromModelBuffer = function (t, e) {
  return y(lt, t, {baseOptions: {modelAssetBuffer: e}})
};

lt.createFromOptions = function (t, e) {
  return y(lt, t, e)
};
```

**Три способа создать:**

| Метод | Модель |
|---|---|
| `createFromModelPath(fileset, path)` | по URL |
| `createFromModelBuffer(fileset, buffer)` | из ArrayBuffer |
| `createFromOptions(fileset, options)` | полный контроль |

**Алиасы прототипа** — чтобы `detect` работал как метод.

**Зачем:** минификация сохраняет короткие имена, а API — длинные.

---

# ЧАСТЬ 6. 11 задач — сводка

## 6.1 Имена графов

| Задача | Граф |
|---|---|
| FaceDetector | `mediapipe.tasks.vision.face_detector.FaceDetectorGraph` |
| FaceLandmarker | `mediapipe.tasks.vision.face_landmarker.FaceLandmarkerGraph` |
| FaceStylizer | `mediapipe.tasks.vision.face_stylizer.FaceStylizerGraph` |
| GestureRecognizer | `mediapipe.tasks.vision.gesture_recognizer.GestureRecognizerGraph` |
| HandLandmarker | `mediapipe.tasks.vision.hand_landmarker.HandLandmarkerGraph` |
| HolisticLandmarker | `mediapipe.tasks.vision.holistic_landmarker.HolisticLandmarkerGraph` |
| ImageClassifier | `mediapipe.tasks.vision.image_classifier.ImageClassifierGraph` |
| ImageEmbedder | `mediapipe.tasks.vision.image_embedder.ImageEmbedderGraph` |
| ImageSegmenter | `mediapipe.tasks.vision.image_segmenter.ImageSegmenterGraph` |
| InteractiveSegmenter | `mediapipe.tasks.vision.interactive_segmenter.InteractiveSegmenterGraph` |
| PoseLandmarker | `mediapipe.tasks.vision.pose_landmarker.PoseLandmarkerGraph` |
| ObjectDetector | `mediapipe.tasks.vision.object_detector.ObjectDetectorGraph` |

**12 имён** (ObjectDetector тоже есть).

## 6.2 Топология лица

```js
var _s = kt([61,146],[146,91],[91,181],[181,84],[84,17],[17,314],[314,405],
            [405,321],[321,375],[375,291],[61,185],[185,40],[40,39],[39,37],
            [37,0],[0,267],[267,269],[269,270],[270,409],[409,291],
            [78,95],[95,88],[88,178],[178,87],[87,14],[14,317],[317,402],
            [402,318],[318,324],[324,308],[78,191],[191,80],[80,81],
            [81,82],[82,13],[13,312],[312,311],[311,310],[310,415],[415,308]);
```

**40 связей** для **правой брови** (индексы 61–291) и **левой** (78–308).

**`kt(...)`** — создать массив `[from, to]` пар (connectors).

**Это только часть.** Полный граф лица — **478 точек**, сотни связей.

```js
var vs = kt([263,249],[249,390],[390,373],...);
var ws = kt([276,283],[283,282],...);
```

**`vs`** — правый глаз, **`ws`** — левый глаз (начало).

## 6.3 Индексы точек лица

**Стандарт MediaPipe FaceMesh:**
- **0–467** — базовая сетка
- **468–472** — зрачки правого глаза
- **473–477** — зрачки левого глаза

**Итого 478 точек.**

## 6.4 Pose — 33 точки

**Стандарт BlazePose:**
```
0  — нос
1-6  — глаза, уши
7-10 — рот
11-12 — плечи
13-14 — локти
15-16 — запястья
17-22 — кисти
23-24 — бёдра
25-26 — колени
27-28 — лодыжки
29-32 — стопы
```

## 6.5 Hand — 21 точка

```
0  — запястье
1-4  — большой палец
5-8  — указательный
9-12 — средний
13-16 — безымянный
17-20 — мизинец
```

---

# ЧАСТЬ 7. Конвертация результатов

## 7.1 `Co()` / `da()`

```js
this.g.attachProtoVectorListener("detections", (r, s) => {
  for (const i of r) r = Co(i), this.j.detections.push(da(r));
  l(this, s)
});
```

**`Co(i)`** — protobuf → JS-объект (deserialize).
**`da(r)`** — нормализация (плоская структура).

**`this.j`** — результат, мутируется.

**`l(this, s)`** — callback `onResults` с timestamp `s`.

## 7.2 `attachEmptyPacketListener`

```js
this.g.attachEmptyPacketListener("detections", r => { l(this, r) });
```

**Если поток опустошён** (нет детекций) — вызвать callback с пустым
результатом.

**Зачем:** в видео-режиме каждый кадр должен дать результат, даже если
ничего не найдено.

---

# ЧАСТЬ 8. `MPImage` и `MPMask`

## 8.1 `z` (`MPImage`)

**Обёртка изображения:**
```js
new W([s], n, !1, t.g.i.canvas, t.R, i, e)
```
- `[s]` — данные (Uint8Array / Float32Array / ImageData)
- `n` — тип
- `false` — не клонировать
- `canvas` — WebGL-канвас
- `R` — ImageSource
- `i`, `e` — width, height

**Источники:**
- `ImageData`
- `HTMLImageElement`
- `HTMLVideoElement`
- `HTMLCanvasElement`
- `ImageBitmap`
- `OffscreenCanvas`

## 8.2 `W` (`MPMask`)

**Маска сегментации** — `Float32Array` вероятностей.

**Методы (обычно):**
- `getAsFloat32Array()`
- `getAsUint8Array()`
- `getAsWebGLTexture()`
- `getAsFloat32Array()`
- `hasUint8Array()`

## 8.3 `Sr` / `Lr` — результаты сегментации

```js
export {..., Sr as ImageSegmenterResult, ..., Lr as InteractiveSegmenterResult, ...}
```

**`ImageSegmenterResult`:**
```js
{
  confidenceMasks: MPMask[] | undefined,
  categoryMask: MPMask | undefined,
  close(): void
}
```

**`InteractiveSegmenterResult`:**
```js
{
  confidenceMasks: MPMask[] | undefined,
  close(): void
}
```

---

# ЧАСТЬ 9. `FilesetResolver` — `he`

```js
var he = class {}
```

**Пустой класс!** Методы добавляются позже (статически или в прототип).

**Обычно:**
```js
FilesetResolver.forVisionTasks(wasmBasePath) → Promise<WasmFileset>
FilesetResolver.forAudioTasks(wasmBasePath)
FilesetResolver.forTextTasks(wasmBasePath)
FilesetResolver.forGenAiTasks(wasmBasePath)
```

**Загружает:**
- `vision_wasm_internal.js` (loader)
- `vision_wasm_internal.wasm` (бинарь)

**Возвращает** `{wasmLoaderPath, wasmBinaryPath}` — от `Oe()`.

**В этом файле — только объявление.** Реализация, вероятно, в
прототипе через `he.forVisionTasks = ...` где-то ещё.

---

# ЧАСТЬ 10. UTF-8 / текст

## 10.1 `Ni()` — кодирование

```js
function Ni(t) {
  if (Pa) t = (Ma || (Ma = new TextEncoder)).encode(t);
  else {
    let n = 0;
    const r = new Uint8Array(3 * t.length);
    for (let s = 0; s < t.length; s++) {
      var e = t.charCodeAt(s);
      if (e < 128) r[n++] = e;
      else {
        if (e < 2048) r[n++] = e >> 6 | 192;
        else {
          if (e >= 55296 && e <= 57343) {       // суррогатная пара
            if (e <= 56319 && s < t.length) {
              const i = t.charCodeAt(++s);
              if (i >= 56320 && i <= 57343) {
                e = 1024 * (e - 55296) + i - 56320 + 65536;
                r[n++] = e >> 18 | 240;
                r[n++] = e >> 12 & 63 | 128;
                r[n++] = e >> 6 & 63 | 128;
                r[n++] = 63 & e | 128;
                continue
              }
              s--
            }
            e = 65533                          // replacement char
          }
          r[n++] = e >> 12 | 224;
          r[n++] = e >> 6 & 63 | 128
        }
        r[n++] = 63 & e | 128
      }
    }
    t = n === r.length ? r : r.subarray(0, n)
  }
  return t
}
```

**Ручной UTF-8 энкодер** на случай отсутствия `TextEncoder`.

**Обработка суррогатных пар:**
```js
e = 1024 * (e - 55296) + i - 56320 + 65536;   // high<<10 + low + 0x10000
```
**4-байтовое кодирование** для эмодзи.

**Невалидная пара** → `65533` (U+FFFD REPLACEMENT CHARACTER).

## 10.2 `Jt()` — ошибка

```js
function Jt() { throw Error("Invalid UTF8") }
```

**Декодер при невалидном UTF-8.**

## 10.3 `Ls()` — конкатенация

```js
function Ls(t, e) {
  return e = String.fromCharCode.apply(null, e), t == null ? e : t + e
}
```

**`String.fromCharCode.apply(null, e)`** — проблема с большим массивом
(переполнение стека), как в `modern-screenshot`.

**`t == null ? e : t + e`** — конкатенация.

---

# ЧАСТЬ 11. Схема

```
FilesetResolver.forVisionTasks(basePath)
 │
 ├─ pa() → проверка SIMD
 ├─ Oe("vision", basePath) → {wasmLoaderPath, wasmBinaryPath}
 └─ → WasmFileset

FaceDetector.createFromOptions(fileset, options)
 │
 ├─ new lt(fileset, options)
 │   ├─ super(new GraphRunner(...), "image_in", "norm_rect_in", false)
 │   ├─ this.h = new FaceDetectorOptions()
 │   │    ├─ baseOptions (model path/buffer, delegate GPU/CPU)
 │   │    ├─ minDetectionConfidence = 0.5
 │   │    └─ minSuppressionThreshold = 0.3
 │   └─ m() → buildGraph()
 │        ├─ CalculatorGraphConfig
 │        ├─ input_stream: image_in, norm_rect_in
 │        ├─ output_stream: detections
 │        ├─ node: "mediapipe.tasks.vision.face_detector.FaceDetectorGraph"
 │        ├─ protobuf → Uint8Array
 │        └─ setGraph(binary, true)
 │
 ├─ l(options) → setOptions()
 │   ├─ валидация: не оба path+buffer, хотя бы один
 │   ├─ delegate: GPU → E1, CPU → Qs
 │   ├─ fetch(modelAssetPath) → arrayBuffer
 │   └─ FS_createDataFile("/model.dat", ...)
 │
 └─ detect(image) / detectForVideo(video, ts)
     ├─ vt/Ot — проверка режима
     ├─ ba(...) → передать в WASM
     ├─ attachProtoVectorListener("detections", ...)
     │    └─ Co(proto) → da(obj) → this.j.detections
     └─ l(this, ts) → callback onResults
```

---

# ЧАСТЬ 12. Слабые места

## 12.1 `Ls()` — переполнение стека

```js
String.fromCharCode.apply(null, e)
```

**Классическая проблема.** При `e.length > 65535` → `RangeError`.

**То же, что в `modern-screenshot`.** ELK/MediaPipe копируют этот паттерн.

## 12.2 `he` (`FilesetResolver`) — пустой класс

```js
var he = class {}
```

**Никаких методов в определении.** Реализация — где-то в прототипе
(или отсутствует).

**Если отсутствует** — `FilesetResolver.forVisionTasks()` бросит
`TypeError: not a function`.

## 12.3 `console.error` вместо throw

```js
function li(t, e, n) {
  t.m || console.error("No wasm multistream support detected: ...")
  ...
}
```

**Продолжает выполнение после ошибки.** Потом может упасть глубже
с непонятной ошибкой.

## 12.4 `modelAssetPath`/`modelAssetBuffer` — валидация только на «оба»

```js
if (a?.modelAssetBuffer && a?.modelAssetPath)
  throw Error("Cannot set both ...");

if (!(w(this.baseOptions, wn, 1)?.g() || w(...)?.h()
      || a?.modelAssetBuffer || a?.modelAssetPath))
  throw Error("Either ... must be set");
```

**Но:** если `modelAssetPath` — не строка (число, объект) —
`fetch(a.modelAssetPath.toString())` вызовет `fetch("123")` → ошибка сети.

**Нет валидации типа.**

## 12.5 `FS_unlink` в `try/catch` молча

```js
try { this.g.i.FS_unlink("/model.dat") } catch {}
```

**Если файла нет** — ошибка проглочена. Ок, но неинформативно.

## 12.6 `delegate: "GPU"` без проверки поддержки

```js
u.delegate === "GPU" ? (u = g, v = new E1, Re(u, 2, sn, v)) : ...
```

**Не проверяется:**
- есть ли WebGL2
- есть ли GPU-делегат в WASM

**При отсутствии** — ошибка при инициализации графа (глубже).

## 12.7 `this.j` — мутируемый результат

```js
this.j = {detections: []};
...
this.j.detections.push(da(r))
```

**Один объект на инстанс.** При параллельных вызовах — гонка.

**`D()` сбрасывает:**
```js
D(t, e) { return this.j = {detections: []}, vt(this, t, e), this.j }
```

**Но `attachProtoVectorListener` вызывается асинхронно** — может
писать в **новый** `this.j` или в **старый** (если `this.j` заменили).

**Проблема:** если вызвать `detect()` дважды подряд — второй вызов
сбросит `this.j`, а первый ещё не заполнился.

## 12.8 `Number(t[1]) >= 17` — только для Safari

```js
function ga() {
  return typeof OffscreenCanvas < "u" && (
    !(e.userAgent.includes("Safari") && !e.userAgent.includes("Chrome"))
    || !!((t = e.userAgent.match(/Version\/([\d]+).*Safari/))
           && t.length >= 1 && Number(t[1]) >= 17)
  )
}
```

**Проверка только Safari ≥ 17.** Другие браузеры — просто `OffscreenCanvas`.

**Проблема:** Firefox/старый Chrome могут иметь `OffscreenCanvas` без
полной поддержки в Worker.

## 12.9 `pa()` кэширует `false` навсегда

```js
if (Ze === void 0) try { ...; Ze = !0 } catch { Ze = !1 }
return Ze
```

**Если WASM-инстанс упал** (например, CSP заблокировала) — `Ze = false`
навсегда, даже если причина устранена.

## 12.10 `gpuOriginForWebTexturesIsBottomLeft`

```js
t.i.gpuOriginForWebTexturesIsBottomLeft
  && n.pixelStorei(n.UNPACK_FLIP_Y_WEBGL, !0);
```

**Флаг из WASM.** Если WASM считает, что origin внизу-слева — текстура
переворачивается.

**Проблема:** разные версии WASM могут иметь разный флаг → изображение
перевёрнуто.

## 12.11 `ma(e)` — размеры

```js
const [r, s] = ma(e);
return !t.l || r === t.i.canvas.width && s === t.i.canvas.height
  || (t.i.canvas.width = r, t.i.canvas.height = s),
  [r, s]
```

**Не показана.** Скорее всего — `[imageData.width, imageData.height]`
или измерение через canvas.

**`t.l`** — флаг «не ресайзить».

## 12.12 `M(t, "image_in")` — строковые потоки

```js
M(t, "image_in");
M(t, "norm_rect_in");
b(t, "detections");
```

**Строковые имена** — если опечатка, WASM не найдёт поток и упадёт
с неинформативной ошибкой.

---

# ЧАСТЬ 13. Итог

## 13.1 Ключевые числа

| Что | Значение |
|---|---|
| Размер | 131 944 байта |
| Задач (классов) | 12 |
| Графов MediaPipe | 12 |
| Точек лица (FaceMesh) | 478 |
| Точек скелета (BlazePose) | 33 |
| Точек кисти | 21 |
| Вариантов WASM | 2 (SIMD / noSIMD) |
| Делегатов | 2 (GPU / CPU) |
| Режимов | 2 (IMAGE / VIDEO) |
| Точек в `_s` (брови) | 40 |
| `minDetectionConfidence` | 0.5 |
| `minSuppressionThreshold` | 0.3 |

## 13.2 Главное

**1. MediaPipe Tasks Vision — 12 задач:** FaceDetector, FaceLandmarker,
FaceStylizer, GestureRecognizer, HandLandmarker, HolisticLandmarker,
ImageClassifier, ImageEmbedder, ImageSegmenter, InteractiveSegmenter,
ObjectDetector, PoseLandmarker.

**2. WASM грузится отдельно.** Этот файл — только JS-обёртка.
`vision_wasm_internal.wasm` ~10 МБ (SIMD) или ~15 МБ (noSIMD).

**3. Автовыбор WASM по SIMD:**
```js
const n = await pa() ? "wasm_internal" : "wasm_nosimd_internal";
```

**4. Граф-калькуляторы по полному имени:**
```
mediapipe.tasks.vision.face_detector.FaceDetectorGraph
```
Protobuf-конфиг → `Uint8Array` → `setGraph(binary, true)`.

**5. Модель — файл в виртуальной ФС Emscripten:**
```js
fetch(modelAssetPath) → arrayBuffer → FS_createDataFile("/model.dat", ...)
```

**6. Два режима строго:**
- `IMAGE` → `detect(image)`
- `VIDEO` → `detectForVideo(video, timestamp)`
- Несовпадение → ошибка

**7. Два делегата:** `GPU` (`E1`) и `CPU` (`Qs`). Без проверки поддержки.

**8. `this.j` — мутируемый результат, гонка при параллельных вызовах.**

**9. `Ls()` — `String.fromCharCode.apply` — переполнение стека** при
массиве > 65535.

**10. `FilesetResolver` — пустой класс** в определении, методы
добавляются извне.

**11. `pa()` кэширует `false` навсегда** — нет восстановления после
временной ошибки.

**12. Топология лица — сотни пар `[from, to]`.** В файле видны фрагменты
(`_s` — 40 пар для бровей).
