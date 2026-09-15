---
tags:
  - portfolio/kneos
  - backend
---

# Módulo CameraView

⬅️ Volver a [[Backend]]

Persiste la última posición/target de `OrbitControls` de la escena 3D **principal** (`public/js/main.js`, la landing fuera de KneOS — el escritorio/pc con el que arranca el sitio) más el modo de foco activo (`"pc"`, `"guestbook"` o `null` = vista libre) y, si hay un foco activo, la distancia de dolly (`dist`, el zoom con scroll dentro de ese foco), para que un visitante que vuelve encuentre la cámara exactamente donde la dejó — incluido el zoom — en vez de siempre en el encuadre inicial hardcodeado (`INITIAL_CAMERA_POSITION`/`INITIAL_CAMERA_ROTATION` en `main.js`). Agregado 2026-09-15 (posición/modo), `dist` sumado el mismo día a pedido explícito ("también si se está haciendo focus en la pc, y el último estado del focus"). Mismo molde upsert-por-`pc_id` que [[Módulo Theme]] (una fila por sesión), pero sin default en `getState` — si no hay fila, el frontend simplemente cae a sus constantes hardcodeadas en vez de que el backend le devuelva un valor inventado.

Distinto de `camera_photos` ([[Módulo CameraPhoto]]): esa tabla es el contenido de las fotos que saca la app Camera de KneOS (bytes de gris), esta es la posición de la cámara 3D de Three.js — nombres deliberadamente separados (`cameraViewRoutes`/`camera_view_state`, no `cameraRoutes`) para no confundirlos.

## Endpoints (`routes/cameraViewRoutes.js`, montado en `/cameraViewRoutes`, `requireAuth` en todo el router)

| Método | Path | Controller |
|---|---|---|
| GET | `/state` | `getCameraViewState` |
| PATCH | `/state` | `editCameraViewState` |

## Controllers (`controllers/cameraViewController.js`)

- **`getCameraViewState`**: `getState(req.pcId)` → `{ state }` (`state` es `null` si la sesión nunca guardó nada — sin upsert-con-default, a diferencia de `getUserColor` de [[Módulo Theme]]).
- **`editCameraViewState`**: valida `x/y/z/target_x/target_y/target_z` con `isFiniteNumber` (nuevo en `utils/validation.js`), `modo` con `isValidCameraViewModo` (`null`, `"pc"` o `"guestbook"` — `400` si no) y `dist` con `isFiniteNumberOrNull` (número finito o `null`, nunca `undefined` implícito) antes de llamar `setState(req.pcId, {...})` → `{ success: true }`.

## Modelo (`models/cameraViewModel.js`)

- **`getState(pc_id)`**: `camera_view_state.findUnique({where:{pc_id}})` — simple, no upsert (no hay default razonable que insertar para una posición de cámara).
- **`setState(pc_id, state)`**: `camera_view_state.upsert({where:{pc_id}, update:state, create:{pc_id,...state}})`.

## Validación (`utils/validation.js`)

- **`isFiniteNumber(value)`**: `typeof value === "number" && Number.isFinite(value)` — genérico, no específico de este módulo.
- **`isFiniteNumberOrNull(value)`**: `value === null || isFiniteNumber(value)` — para `dist`, que viaja `null` en vista libre.
- **`isValidCameraViewModo(value)`**: `null` o una de `CAMERA_VIEW_MODOS` (`Set` con `"pc"`/`"guestbook"`) — mismos ids que `FOCUS_IDS` en `public/js/main.js` (el backend no comparte módulos con el frontend, mismo patrón que `THEME_COLOR_KEYS`, ver [[Módulo Theme#Validación (utils/validation.js)]]).

## Tabla `camera_view_state` (`prisma/schema.prisma`, 2026-09-15)

```prisma
model camera_view_state {
  id_state Int      @id @default(autoincrement())
  pc_id    String   @unique
  x        Float
  y        Float
  z        Float
  target_x Float
  target_y Float
  target_z Float
  modo     String?
  dist     Float?
  sessions sessions @relation(fields: [pc_id], references: [pc_id], onDelete: NoAction, onUpdate: NoAction)
}
```

`pc_id` `@unique` (no `@id` directo) — mismo criterio que `theme_settings`: PK `id_state` autoincremental propia, separada de la FK.

Se guarda **posición + target** (lo que `OrbitControls` usa nativamente para reconstruir la orientación vía `lookAt`), nunca la rotación — evita depender del `order` del `Euler` y es más robusto ante cambios de versión de three.js.

## Frontend

`public/js/CameraViewServices.js` (`getCameraViewState()`/`setCameraViewState(state)`, fetch directo — fuera de `KneOS/js/services`, no usa `apiFetch.js`) consumido por `main.js`:

- **Guardado**: `saveCameraState(position, target, modo, dist)` — sin debounce propio, cada call-site ya es un evento puntual:
  - `controls.addEventListener('end', ...)` (vista libre, `modo: null, dist: null`) — **no `'change'`**: se probó primero con `'change'` + debounce de 800ms, pero con `enableDamping` ese evento sigue disparando cuadro a cuadro varios segundos después de soltar el mouse mientras la inercia decae, así que el debounce nunca terminaba de asentarse y terminaba guardando una posición intermedia al azar (encontrado con un script de Playwright que orbitaba y comparaba la posición justo guardada contra la que realmente quedaba en pantalla). `'end'` marca el fin real del gesto (pointerup/wheel) sin esperar el asentamiento.
  - `enterFocus`/`exitFocus` — guardan el punto de **vista libre** de donde se entró/al que se vuelve (nunca el encuadre ya resuelto del foco), junto con `modo` y, en `enterFocus`, la `dist` (distancia de dolly) vigente en ese momento — así al restaurar se puede re-entrar al mismo foco, con el mismo zoom, en vez de reproducir a mano la posición exacta de la cámara enfocada.
  - El listener `'wheel'` de `cssDiv` (zoom con scroll mientras hay un foco activo) llama `scheduleFocusDistSave(dist)` — ese sí tiene debounce propio (400ms, dispara muchos eventos por gesto de scroll).
  > [!bug] `'end'` disparaba igual aunque el mismo click acabara de entrar en foco
  > El `pointerdown` que dispara `enterFocus` (raycaster en `cssDiv`) deja el listener interno de `pointerup` de `OrbitControls` ya enganchado desde antes de que `controls.enabled` pasara a `false` — así que ese mismo click, al soltarse, igual emitía `'end'` y pisaba con `modo:null` el guardado que `enterFocus` acababa de hacer. Se corrigió agregando un guard (`if (focusActive || camTransition) return;`) al listener `'end'`.
- **Restauración**: `restoreCameraState()`, fire-and-forget desde el setup diferido (no bloquea el primer render, que ya arranca en `INITIAL_CAMERA_POSITION`/`INITIAL_CAMERA_ROTATION`) — si hay estado guardado, hace *snap* de `camera.position`/`controls.target` (sin transición animada) y, si `modo` no es `null`, guarda `modo`/`dist` en `pendingFocusModo`/`pendingFocusDist` y llama `tryEnterPendingFocus(pc)`/`tryEnterPendingFocus(guestbook?.getObject())`.
  > [!bug] Carrera entre el gltf de `pc`/`book` y el propio fetch de `restoreCameraState`
  > `pc.glb` es un archivo estático local (casi instantáneo); `ensureSession()` + `getCameraViewState()` son 2-3 round trips de red seguidos. En la práctica `pc.glb` casi siempre termina de cargar **antes** de que `restoreCameraState()` resuelva — así que el chequeo original ("si `pendingFocusModo` ya está seteado cuando `pc` carga, entrar en foco") nunca se cumplía: `pendingFocusModo` todavía era `null` en ese momento, y no había nada que reintentara después. Se corrigió con un helper compartido, `tryEnterPendingFocus(obj)` (chequea `FOCUS_IDS.get(obj) === pendingFocusModo`, clampea `pendingFocusDist` contra el `min`/`max` del `FOCUS_CONFIG` del objeto y llama `enterFocus`), invocado desde **3** lugares: el callback de carga de `pc.glb`, el `registerFocusTarget` pasado a `initGuestbook` (para el libro) y el propio final de `restoreCameraState()` — cubre las dos direcciones de la carrera (objeto carga antes o después del restore).
  > [!bug] Paneo visible al restaurar un foco guardado
  > `tryEnterPendingFocus` llamaba `enterFocus(obj)` tal cual, que siempre dispara `startCameraTransition` (650ms, ease) — un visitante que recargaba con `modo:"pc"` guardado veía la cámara arrancar en la vista libre y "viajar" de golpe hasta la pc, un paneo raro apenas carga la página. Se agregó un segundo parámetro a `enterFocus(obj, { instant = false })`: con `instant:true` (solo usado por `tryEnterPendingFocus`) salta directo a `toPos`/`focusTarget` vía `camera.position.copy()` + `camera.lookAt()`, sin pasar por `startCameraTransition` ni volver a llamar `saveCameraState` (no hace falta re-guardar lo que ya estaba guardado). El click/tap real del usuario (`enterFocus(obj)` sin opciones) sigue animado como siempre.
  > [!bug] Flash de la vista por defecto antes de la restaurada
  > `hideLoadingScreen()` (fondo negro opaco fijo, `#loadingScreen` en `index.html`) se llamaba apenas terminaba de cargar `pc.glb` — un archivo local, casi instantáneo — sin esperar a `restoreCameraState()` (2-3 round trips de red: `ensureSession()` + `getCameraViewState()`). Un visitante con algo guardado veía la pantalla de carga desaparecer, el encuadre **default** un instante, y recién después el salto/paneo a la posición o foco guardado — justo el efecto que se había corregido en el bug anterior, pero ahora expuesto ANTES de que `tryEnterPendingFocus`/el snap de posición llegaran a correr. Se agregó un segundo gate: `cameraStateSettled` (true una vez que `restoreCameraState()` terminó su fetch, haya o no encontrado algo) — `maybeHideLoadingScreen()` reemplaza la llamada directa y solo oculta la pantalla cuando **ambas** condiciones se cumplen (`pc` cargado Y `cameraStateSettled`), llamada desde el callback de `pc.glb` y desde el final de `restoreCameraState()` (en las dos ramas, con y sin estado guardado) para cubrir cualquier orden de llegada. Sin timeout de por medio a propósito — el fetch es same-origin contra el propio server, no debería colgarse en la práctica; si algún día hiciera falta un tope, agregarlo ahí.
- **Sesión propia**: `ensureSession()` (exportada desde `guestbook.js` el mismo día — antes vivía dentro de `initGuestbook`, usada solo al firmar) se llama también desde `main.js` ni bien cargan los controles, para que un visitante que nunca abre el libro de firmas igual tenga cookie de sesión y la persistencia de cámara funcione.

## Consumido por

Solo `public/js/main.js` (la escena principal) — no tiene nada que ver con [[Maxwell]] (el visor 3D dentro de KneOS, cámara fija, sin persistencia) ni con [[Módulo CameraPhoto]].
