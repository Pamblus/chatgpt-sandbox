# `index-BTTCYQt9.js` — полный разбор (@react-three/drei + MediaPipe)

**Размер:** 1 617 880 байт
**URL:** `https://web-sandbox.oaiusercontent.com/assets/index-BTTCYQt9.js`
**Роль:** **`@react-three/drei`** — 218 компонентов и хуков для R3F.
Плюс **FaceControls** с MediaPipe FaceLandmarker.

---

# ЧАСТЬ 1. Что это

## 1.1 Точная идентификация

**218 экспортов** — это характерно только для `@react-three/drei`.

**Признаки:**
- `dw3` — утилита (`maath`) для демпфирования
- `https://cdn.jsdelivr.net/gh/pmndrs/drei-assets@master` — **ассеты drei**
- 6 вхождений `drei`
- `zustand` — 3 вхождения (drei использует)
- `troika` — 32 вхождения (troika-three-text для `<Text>`)
- `fflate` — 4 (ZIP для GLTF)
- KTX2 — 6 (текстурный компрессор)
- MediaPipe JSX-обёртки — `<FaceLandmarker>`, `<FaceControls>`

## 1.2 Импорты

```js
import {r as g, R as Fs} from "./index-C0fqnZRQ.js";      // React
import {a as Jy} from "./client-BjybnHiv.js";             // react-dom
import {D as Ci, f as z, O as fi, ...} from "./three.module-C846Yxd8.js";  // Three.js
import {j as ic} from "./main-BR1bOQyH.js";               // Object
```

**Плюс динамический:**
```js
await import("./vision_bundle-CPRQl8Yc.js")   // MediaPipe (только для FaceLandmarker)
```

---

# ЧАСТЬ 2. 218 экспортов — категории

## 2.1 Простые фигуры (3D-примитивы)

```
Box, Circle, Cone, Cylinder, Dodecahedron, Extrude, Icosahedron,
Lathe, Octahedron, Plane, Polyhedron, Ring, Shape, Sphere,
Tetrahedron, Torus, TorusKnot, Tube
```

**18 фигур.** Обёртки над `THREE.Mesh` с геометрией.

## 2.2 Загрузчики ассетов

```
Fbx, Gltf, Ktx2, Text3D, useFBX, useGLTF, useKTX2, useTexture,
useTexture, useCubeTexture, useEnvironment, useFont, useMatcapTexture,
useNormalTexture, useVideoTexture, useSpriteLoader, useProgress
```

**17 загрузчиков.**

**`Gltf`** — модель GLTF.
**`Ktx2`** — сжатые текстуры (basis universal).
**`Text3D`** — 3D-текст (troika).

## 2.3 Постпроцессинг / эффекты

```
EffectComposer, Effects, Bloom, ChromaticAberration, Noise, Vignette,
DepthOfField, SSAO, Outline, Pixelation, ToneMapping
```

**12 эффектов.**

## 2.4 Тени

```
AccumulativeShadows, ContactShadows, BakeShadows, SoftShadows,
Shadow, ShadowAlpha
```

**6 компонентов теней.**

## 2.5 Окружение / фон

```
Environment, EnvironmentCube, EnvironmentMap, EnvironmentPortal,
Backdrop, Sky, Stars, Cloud, Clouds, CloudInstance, Sparkles,
GroundProjectedEnv
```

**12 компонентов окружения.**

## 2.6 Контролы

```
ArcballControls, CameraControls, CameraControlsImpl,
DeviceOrientationControls, DragControls, FirstPersonControls,
FlyControls, KeyboardControls, MapControls, OrbitControls,
PointerLockControls, TransformControls, PresentationControls
```

**13 контролов.**

**`CameraControls`** — самый мощный (на основе `camera-controls`).

## 2.7 Gizmo / отладка

```
GizmoHelper, GizmoViewcube, GizmoViewport, Grid, Helper,
Stats, StatsGl, Fps, Perf, Bvh, CycleRaycast
```

**11 отладочных.**

## 2.8 Текст

```
Text, Text3D, Html, Billboard, Hud, Svg, useFont, useTrail
```

**8 текстовых.**

**`Html`** — HTML-оверлей в 3D.
**`Text`** — troika-three-text (SDF-рендеринг).

## 2.9 MediaPipe / лицо

```
FaceControls, FaceLandmarker, FaceLandmarkerDefaults,
Facemesh, FacemeshDatas, FacemeshEye, FacemeshEyeDefaults,
useFaceControls, useFaceLandmarker
```

**9 компонентов для лицевого трекинга.**

## 2.10 Хуки R3F

```
useThree, useFrame, useCamera, useCubeCamera, useBounds,
useBoxProjectedEnv, useContextBridge, useCursor, useDepthBuffer,
useDetectGPU, useFBO, useHelper, useIntersect, useMask,
useMotion, usePerformanceMonitor, useScroll, useSelect,
useSurfaceSampler, useTrailTexture, useAnimations, useAspect,
useBVH, useGizmoContext
```

**24 хука.**

## 2.11 Инстансинг

```
Instance, InstancedAttribute, Instances, Merged, Points, Sprite,
SpriteAnimator, ComputeAttribute
```

**8 компонентов.**

---

# ЧАСТЬ 3. `FaceLandmarker` — обёртка MediaPipe

## 3.1 Дефолтные настройки

```js
const Qx = g.createContext({});

const jy = {
  basePath: "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.17/wasm",
  options: {
    baseOptions: {
      modelAssetPath: "https://storage.googleapis.com/mediapipe-models/face_landmarker/face_landmarker/float16/1/face_landmarker.task",
      delegate: "GPU"
    },
    runningMode: "VIDEO",
    outputFaceBlendshapes: !0,
    outputFacialTransformationMatrixes: !0
  }
};
```

### Разбор

**`basePath`** — CDN для WASM:
```
https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.17/wasm
```

**Версия 0.10.17.** **Внешний CDN**, не локальный.

**`modelAssetPath`** — модель **float16**:
```
https://storage.googleapis.com/mediapipe-models/face_landmarker/face_landmarker/float16/1/face_landmarker.task
```

**Google Cloud Storage.** Не наш origin.

**`delegate: "GPU"`** — GPU-делегат.

**`runningMode: "VIDEO"`** — видео-режим (не IMAGE).

**`outputFaceBlendshapes: true`** — **52 blendshape** (мимика).

**`outputFacialTransformationMatrixes: true`** — матрица 4×4 позы головы.

## 3.2 Компонент `Xk` — `FaceLandmarker`

```js
const Xk = g.forwardRef(({basePath: r = jy.basePath, options: e = jy.options, children: t}, i) => {
  const n = JSON.stringify(e),
        s = cr(async () => {
          const {FilesetResolver: o, FaceLandmarker: a} =
            await S2(async () => {
              const {FilesetResolver: c, FaceLandmarker: u} =
                await import("./vision_bundle-CPRQl8Yc.js");
              return {FilesetResolver: c, FaceLandmarker: u}
            }, []),
            l = await o.forVisionTasks(r);
          return a.createFromOptions(l, e)
        }, [r, n]);

  g.useEffect(() => () => {
    s?.close();
    th([r, n])
  }, [s, r, n]);

  g.useImperativeHandle(i, () => s, [s]);

  return g.createElement(Qx.Provider, {value: s}, t)
});
```

### Разбор

**`n = JSON.stringify(e)`** — **ключ кэша** по опциям.

**`cr(...)`** — React 19 `use()` на промис (или свой хук). Загружает
MediaPipe.

**Динамический `import("./vision_bundle-CPRQl8Yc.js")`** — загрузка
только когда компонент используется.

**`FilesetResolver.forVisionTasks(basePath)`** — загрузить WASM.

**`FaceLandmarker.createFromOptions(fileset, options)`** — создать задачу.

**Cleanup:**
```js
g.useEffect(() => () => { s?.close(); th([r, n]) }, [s, r, n]);
```
**`s?.close()`** — освободить нативный ресурс.

**`th([r, n])`** — очистить кэш (suspend-react).

**`useImperativeHandle(i, () => s, [s])`** — пробросить `ref` на задачу.

**`Qx.Provider value={s}`** — контекст с задачей.

## 3.3 `VL()` — `useFaceLandmarker`

```js
function VL() { return g.useContext(Qx) }
```

**Просто `useContext`.**

## 3.4 Вспомогательные функции

```js
function Xy(r, e) { return r.clone().add(e).multiplyScalar(.5) }   // середина
function $o(r, e, t) { const i = r.localToWorld(e); return t.worldToLocal(i) }  // преобразование
```

**`Xy(a, b)`** — `(a + b) / 2` (средняя точка).
**`$o(obj, point, other)`** — точка из `obj.local` в `other.local`.

---

# ЧАСТЬ 4. `FaceControls` — управление камерой лицом

## 4.1 Дефолты

```js
const qk = g.forwardRef(({
  camera: r,
  videoTexture: e = {start: !0},
  manualDetect: t = !1,
  faceLandmarkerResult: i,
  manualUpdate: n = !1,
  makeDefault: s,
  smoothTime: o = .25,
  offset: a = !0,
  offsetScalar: l = 80,
  eyes: c = !1,
  eyesAsOrigin: u = !0,
  depth: h = .15,
  debug: f = !1,
  facemesh: d
}, m) => { ... })
```

## 4.2 Параметры

| Параметр | По умолчанию | Что |
|---|---|---|
| `camera` | — | камера (или текущая) |
| `videoTexture` | `{start: true}` | текстура с камеры |
| `manualDetect` | `false` | не детектить автоматически |
| `faceLandmarkerResult` | — | внешний результат |
| `manualUpdate` | `false` | не обновлять автоматически |
| `makeDefault` | — | сделать контролом по умолчанию |
| **`smoothTime`** | **`0.25`** | сглаживание (сек) |
| `offset` | `true` | смещать |
| **`offsetScalar`** | **`80`** | множитель смещения |
| `eyes` | `false` | следить за глазами |
| `eyesAsOrigin` | `true` | глаза как origin |
| **`depth`** | **`0.15`** | глубина |
| `debug` | `false` | отладка |
| `facemesh` | — | меш лица |

## 4.3 `w()` — вычислить целевую позу камеры

```js
const w = g.useCallback(() => {
  _.parent = x.parent;
  const P = C.current;

  if (P) {
    const {outerRef: O, eyeRightRef: N, eyeLeftRef: Z} = P;

    if (N.current && Z.current) {
      const {irisDirRef: ee} = N.current,
            {irisDirRef: q} = Z.current;

      if (ee.current && q.current && O.current) {
        // 1. направления зрачков в мировые координаты
        S.copy($o(ee.current, new z(0, 0, 0), O.current)),   // правый
        b.copy($o(q.current, new z(0, 0, 0), O.current));    // левый

        // 2. позиция — середина между зрачками
        _.position.copy($o(O.current, Xy(S, b), x.parent || E));

        // 3. направление взгляда — середина направлений
        T.copy($o(ee.current, new z(0, 0, 1), O.current)),
        R.copy($o(q.current, new z(0, 0, 1), O.current));

        _.lookAt(O.current.localToWorld(Xy(T, R)))
      }
    } else if (O.current) {
      // fallback: по центру лица
      _.position.copy($o(O.current, new z(0, 0, 0), x.parent || E)),
      _.lookAt(O.current.localToWorld(new z(0, 0, 1)))
    }
  }

  return _
}, [x, b, R, S, T, E, _]);
```

### Логика

**1. Если есть оба глаза** — взять направления зрачков, вычислить
**середину** по позиции и направлению.

**2. Если только лицо** — центр лица + направление `(0,0,1)`.

**3. `$o(...)`** — преобразовать из локальной системы глаза в локальную
систему лица.

## 4.4 `D()` — применение с демпфированием

```js
const D = g.useCallback(function (P, O) {
  if (x) {
    O ??= w();

    o > 0
      ? (Rr.damp3(B.position, O.position, o, P, void 0, void 0, 1e-9),
         Rr.dampE(B.rotation, O.rotation, o, P, void 0, void 0, 1e-9))
      : (B.position.copy(O.position), B.rotation.copy(O.rotation));

    x.position.copy(B.position);
    x.rotation.copy(B.rotation)
  }
}, [x, w, o, B.position, B.rotation]);
```

### `Rr` = `maath/easing`

```js
Rr.damp3(current, target, smoothTime, delta, maxSpeed, easing, eps)
Rr.dampE(current, target, smoothTime, delta, ...)
```

**`damp3`** — сглаживание `Vector3` (экспоненциальное).
**`dampE`** — сглаживание `Euler` (углы).

**`smoothTime = 0.25`** — за 0.25 с догоняет.

**`1e-9`** — epsilon (порог остановки).

**При `smoothTime = 0`** — мгновенно (`copy`).

## 4.5 Применение каждый кадр

```js
Ve((P, O) => { n || D(O) });
```

**`Ve`** — `useFrame`. Если не `manualUpdate` — обновлять каждый кадр.

**`O`** — `delta` (время с прошлого кадра).

## 4.6 `U()` — обработка видео

```js
const U = g.useCallback((P, O) => {
  const N = F.current;
  if (!N) return;

  const Z = N.source.data,
        ee = W?.   // ← дальше не показано
```

**Берёт видео из текстуры и передаёт в FaceLandmarker.**

---

# ЧАСТЬ 5. `Facemesh` — 468 точек

## 5.1 `FacemeshDatas`

**Экспорт `FacemeshDatas`** — данные топологии.

**Из документа #18** (MediaPipe): 478 точек, сотни связей.

**Здесь — 468** (без зрачков).

## 5.2 `FacemeshEye`

**Отдельный компонент для глаз** — 468–477 (10 точек) или 473–477
(5 точек на глаз).

**`FacemeshEyeDefaults`** — дефолтные настройки.

---

# ЧАСТЬ 6. Загрузчики

## 6.1 `im()` — `useKTX2` (basis)

```js
const lE = "https://cdn.jsdelivr.net/gh/pmndrs/drei-assets@master";

function im(r, e = `${lE}/basis/`) {
  const t = le(n => n.gl),
        i = li(j0, Tl(r) ? Object.values(r) : r, n => {
          n.detectSupport(t),
          n.setTranscoderPath(e)
        });

  if (g.useEffect(() => {
        (Array.isArray(i) ? i : [i]).forEach(t.initTexture)
      }, [t, i]),
      Tl(r)) {
    const n = Object.keys(r), s = {};
    return n.forEach(o => Object.assign(s, {[o]: i[n.indexOf(o)]})), s
  } else return i
}
```

### Разбор

**`lE = "https://cdn.jsdelivr.net/gh/pmndrs/drei-assets@master"`** —
**CDN с ассетами drei.**

**`@master`** — **НЕ пинится версия!** Всегда последняя.

**`basis/`** — путь к transcoder'у KTX2.

**`n.detectSupport(t)`** — определить поддержку сжатия.
**`n.setTranscoderPath(e)`** — путь к transcoder'у.

**`im.preload = (r, e) => li.preload(...)`** — предзагрузка.

### Риск

**`@master` в URL CDN** — код обновляется без ведома проекта.

**Если репозиторий скомпрометирован** — произвольный JS/WASM в виджете.

**Правильно:** пиннить версию (`@1.0.0`).

## 6.2 `Gltf` — модели

**Использует `GLTFLoader` из three.**

**`fflate`** (4 вхождения) — распаковка ZIP (GLTF с текстурами в одном
файле).

## 6.3 `Ktx2` — сжатые текстуры

**6 вхождений `ktx2`.**

**Basis Universal** — GPU-сжатие. Требует transcoder (`.wasm`).

---

# ЧАСТЬ 7. Остальные заметные компоненты

## 7.1 `Html` — HTML в 3D

**Позиционирует DOM-элемент поверх canvas по 3D-координатам.**

**Использует `createPortal` из `react-dom`** (импорт `Jy` из
`client-BjybnHiv.js`).

## 7.2 `Text` — troika

**32 вхождения `troika`.** `troika-three-text` — SDF-рендеринг текста
в 3D.

**`troika-three-text`** содержит свой **worker** для шейпинга текста
(~300 КБ).

## 7.3 `Stats` / `StatsGl` / `Fps` / `Perf`

**Отладочные оверлеи.**

**`StatsGl`** — WebGL-статистика.
**`Perf`** — график производительности.

## 7.4 `Bvh`

**BVH-ускорение raycast** (`three-mesh-bvh`).

**Ускоряет** пересечения лучей с мешами в 10–100 раз.

## 7.5 `DetectGPU`

**`GPU tier`** — уровень видеокарты (0–3).

**Использует WebGL-рендеринг строки и сравнивает с базой.**

## 7.6 `Environment`

**HDR-окружение** для освещения.

**Загружает `.hdr` с CDN drei-assets.**

## 7.7 `OrbitControls` и другие контролы

**Из `three-stdlib`** (вшит).

**`CameraControls`** — самый мощный, на `camera-controls`.

---

# ЧАСТЬ 8. `zustand` внутри

```js
const Sy = r => {
  (Ux ? "production" : void 0) !== "production"
    && typeof r != "function"
    && console.warn("[DEPRECATED] Passing a vanilla store will be unsupported in a future version. Instead use `import { useStore } from 'zustand'`.");

  const e = typeof r == "function" ? tL(r) : r,
        t = (i, n) => SL(e, i, n);

  return Object.assign(t, e), t
};

const wy = typeof window < "u" && (
  window.document?.createElement || window.navigator?.product === "ReactNative"
) ? Fs.useLayoutEffect : Fs.useEffect;
```

**zustand для сторов drei.**

**`wy`** — `useIsomorphicLayoutEffect` (SSR-safe).

**Предупреждение `[DEPRECATED]`** — старая сигнатура.

---

# ЧАСТЬ 9. Схема

```
<Canvas>  (run-three / R3F)
 │
 └─ <FaceLandmarker options={...}>
     │
     ├─ dynamic import("./vision_bundle-CPRQl8Yc.js")
     ├─ FilesetResolver.forVisionTasks(
     │    "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.17/wasm")
     ├─ FaceLandmarker.createFromOptions(fileset, {
     │    baseOptions: {
     │      modelAssetPath: "https://storage.googleapis.com/mediapipe-models/
     │                     face_landmarker/face_landmarker/float16/1/face_landmarker.task",
     │      delegate: "GPU"
     │    },
     │    runningMode: "VIDEO",
     │    outputFaceBlendshapes: true,
     │    outputFacialTransformationMatrixes: true
     │  })
     └─ → Qx.Provider value={landmarker}
          │
          └─ <FaceControls smoothTime={0.25} eyes>
              ├─ useFrame((state, delta) => {
              │    const target = computeTargetPose();   // w()
              │    damp3(camera.position, target.position, 0.25, delta, ...);
              │    dampE(camera.rotation, target.rotation, 0.25, delta, ...);
              │  })
              └─ <Facemesh /> <FacemeshEye />
```

---

# ЧАСТЬ 10. Слабые места

## 10.1 `@master` в drei-assets CDN

```js
const lE = "https://cdn.jsdelivr.net/gh/pmndrs/drei-assets@master";
```

**`@master` — не пинится.** Всегда последний коммит.

**Если репозиторий скомпрометирован** — произвольные текстуры/транскодеры
в виджете.

## 10.2 MediaPipe WASM с внешнего CDN

```js
basePath: "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.17/wasm"
```

**Версия 0.10.17 пиннится**, но **CDN внешний** (jsdelivr).

**Плюс `vision_bundle-CPRQl8Yc.js` — локальный**, а WASM — с CDN.

**Расхождение:** JS-обёртка локальная, WASM с CDN. Если версии разойдутся —
сломается.

## 10.3 Модель с Google Cloud Storage

```js
modelAssetPath: "https://storage.googleapis.com/mediapipe-models/face_landmarker/face_landmarker/float16/1/face_landmarker.task"
```

**Внешний домен `storage.googleapis.com`.**

**CSP должен разрешать** — в `apply-csp` базовый список включает
`*.oaiusercontent.com`, 35 Azure Blob, jsdelivr, unpkg, esm.sh, pypi,
threejs.org.

**`storage.googleapis.com` — НЕТ в списке!**

**Значит:** если виджет использует `<FaceLandmarker>`, `applyCsp`
**заблокирует** загрузку модели.

**Проверить:** нужно передать `frameDomains`/`connectDomains` с
`storage.googleapis.com`.

## 10.4 `delegate: "GPU"` жёстко

```js
delegate: "GPU"
```

**Не проверяется поддержка.** Если GPU-делегат недоступен — ошибка
инициализации.

**Fallback на CPU отсутствует** в дефолтах.

## 10.5 `JSON.stringify(options)` как ключ кэша

```js
const n = JSON.stringify(e),
      s = cr(async () => { ... }, [r, n]);
```

**Порядок ключей** в объекте влияет на строку.

**Два одинаковых по смыслу объекта** с разным порядком → два экземпляра
MediaPipe.

**Плюс:** если `options` содержит функцию (например, колбэк) —
`JSON.stringify` её выбросит или сериализует как `undefined`.

## 10.6 `s?.close()` — только при размонтировании

```js
g.useEffect(() => () => { s?.close(); th([r, n]) }, [s, r, n]);
```

**Если компонент перерендерится с новыми `options`** — старый экземпляр
**не закрывается** до unmount.

**Утечка** при частой смене опций.

## 10.7 `useImperativeHandle(i, () => s, [s])`

**Пробрасывает промис, не значение.**

**Пользователь через `ref` получит `Promise`, а не задачу.**

**Должно быть** `useImperativeHandle(i, () => await s)` — но хук синхронный.

## 10.8 `wy` — `useLayoutEffect` в SSR

```js
const wy = typeof window < "u" && (...) ? Fs.useLayoutEffect : Fs.useEffect;
```

**Если `window.document.createElement` есть** — `useLayoutEffect`.

**В iframe** `window` есть → `useLayoutEffect`. **Предупреждений не будет.**

## 10.9 Огромный размер

**1.6 МБ** ради 218 компонентов.

**Реально используется** — 5–10.

**Проблема:** dead-code elimination (tree-shaking) должен срезать
неиспользуемое, но `__vite__mapDeps`-preload грузит **весь файл**.

**Из `prepare-environment`:**
```js
t(() => import("./index-BTTCYQt9.js"), __vite__mapDeps([9, 7, 1, 2, 8, 10, 11]))
```
**Загружает весь бандл** (1.6 МБ) независимо от того, что нужно.

## 10.10 `drei-assets` — зависимость от инфраструктуры

**Текстуры, HDR, basis-транскодер — всё на внешнем CDN.**

**Если CDN недоступен** — компоненты не работают.

**Если репозиторий удалён** — тоже.

---

# ЧАСТЬ 11. Итог

## 11.1 Ключевые числа

| Что | Значение |
|---|---|
| Размер | 1 617 880 байт |
| Экспортов | 218 |
| Фигур | 18 |
| Загрузчиков | 17 |
| Эффектов | 12 |
| Теней | 6 |
| Окружения | 12 |
| Контролов | 13 |
| Отладочных | 11 |
| Текстовых | 8 |
| MediaPipe | 9 |
| Хуков R3F | 24 |
| Инстансинга | 8 |
| MediaPipe tasks-vision | 0.10.17 |
| `smoothTime` FaceControls | 0.25 с |
| `offsetScalar` | 80 |
| `depth` | 0.15 |

## 11.2 Главное

**1. Это `@react-three/drei` — 218 компонентов для R3F.**

**2. MediaPipe встроен:** `<FaceLandmarker>` + `<FaceControls>` +
`<Facemesh>`.

**3. WASM MediaPipe — с jsdelivr (v0.10.17), модель — с
`storage.googleapis.com`.**

**`storage.googleapis.com` НЕТ в базовом CSP-списке** (`apply-csp`).
Значит `<FaceLandmarker>` **заблокируется**, если хост не добавит домен
явно.

**4. `delegate: "GPU"` жёстко, без fallback.**

**5. `drei-assets` с CDN — `@master` без пина версии.**

**6. `JSON.stringify(options)` как ключ кэша** — чувствителен к порядку
ключей, ломается на функциях.

**7. `useImperativeHandle` пробрасывает Promise, не значение.**

**8. `<FaceControls>` — управление камерой лицом:**
```js
damp3(camera.position, target.position, 0.25, delta, ...)
dampE(camera.rotation, target.rotation, 0.25, delta, ...)
```
Сглаживание 0.25 с.

**9. Лицо: 468 точек** (+ 10 глаз через `<FacemeshEye>`).

**10. `52 blendshapes`** (`outputFaceBlendshapes: true`) — мимика.

**11. Матрица 4×4 позы головы** (`outputFacialTransformationMatrixes`).

**12. 1.6 МБ загружается целиком** через `prepareEnvironment`,
независимо от того, какие 5–10 компонентов нужны.
