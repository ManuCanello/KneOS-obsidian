---
tags:
  - portfolio/kneos
  - frontend
---

# Escena 3D

⬅️ Volver a [[KneOS Portfolio]] · Ver [[Arquitectura]]

## `public/index.html`

Import map con `three` (0.158.0) + `OrbitControls`/`GLTFLoader`/`CSS3DRenderer`, y `planck` (redundante a nivel raíz, salvo que se comparta contexto — [[Kfruit]] corre dentro del iframe, que tiene su propio import map). Carga `html2canvas` (CDN, uso no confirmado en el código leído). Único script: `public/js/main.js`.

## `public/js/main.js`

Dos renderers superpuestos sobre `document.body`, ambos absolutos a pantalla completa:

- **`cssDiv`** (`pointer-events: auto`) — contiene el `CSS3DRenderer`, que renderiza el `<iframe>` de KneOS como objeto 3D.
- **`webglDiv`** (`pointer-events: none`, para no bloquear clics sobre el iframe) — contiene el `WebGLRenderer` (`alpha:true`, `clearColor` transparente), para que se vea el CSS3D detrás (efecto: el modelo 3D "flota" sobre el iframe).

### Escena

- `scene = new THREE.Scene()`.
- `camera`: `PerspectiveCamera(fov 50)`, posición inicial `(0, 1.6, 3.5)`.
- `controls = new OrbitControls(camera, cssDiv)` — el target del listener es `cssDiv` (para no interferir con el iframe interactivo montado encima). `enableDamping=true`, `target.set(0, 1.6, 0)`.
- **Luces** (2026-07-30, setup de 3 puntos — antes solo `AmbientLight(0.6)` + una `DirectionalLight`): `HemisphereLight(0xffffff, 0x444444, 0.5)` como base ambiental + `DirectionalLight` **key** (`intensity 1.2`, `(5,5,5)`, frontal) + `DirectionalLight` **fill** (`0.4`, `(-5,2,3)`, opuesta y más tenue) + `DirectionalLight` **rim** (`0.6`, `(0,3,-5)`, atrás) — le da volumen a la PC en vez de la iluminación plana anterior.

### Proyección del escritorio HTML sobre la pantalla del PC 3D

Se crea un `<iframe src="./KneOS/index.html">`, tamaño fijo `1433px × 1139px`, sin borde. Se envuelve en `CSS3DObject(iframe)`, posicionado en `(0, 1.5870, 0.7716)`, rotado en X (inclinación de la pantalla del monitor 3D) y **escalado a `0.001`** en los 3 ejes — técnica estándar para insertar `CSS3DObject` (unidades en píxeles de pantalla) dentro de una escena medida en unidades pequeñas. Se agrega directamente a `scene` (hermano, no hijo del PC), posicionado para coincidir con la pantalla del modelo.

### Carga de modelos 3D

`GLTFLoader` carga `./models/Pc.glb`:
- `pc = gltf.scene`, agregado a la escena.
- Busca el objeto `"screen_iframe"` y lo oculta (`visible=false`) — placeholder del propio `.glb`, reemplazado por el iframe CSS3D real.
- Busca `"screen"` y reemplaza su material por un `MeshPhysicalMaterial` de vidrio (transmission 1.0, roughness 0, opacity 0.15, `DoubleSide`) — efecto de reflejo semitransparente sobre el iframe.

> [!warning] `lampara.glb` sin integrar
> El asset existe en `public/models/lampara.glb` pero **no se carga** en ningún lugar del código leído. Ver [[Deuda Técnica]].

### Loop de render (`animate()`)

`requestAnimationFrame` recursivo: `controls.update()` → `rendererCSS.render(scene, camera)` → `renderer.clear()` → `renderer.render(scene, camera)`. El `renderer.clear()` manual asegura que el canvas WebGL transparente se limpie a `alpha 0` antes de dibujar de nuevo (con `autoClear=false` en la config, evita un clear automático redundante).

> [!success] Render bajo demanda (`needsRender`) — antes renderizaba sin parar (2026-08-03)
> Hasta esta fecha `animate()` llamaba a los dos `render()` en **cada** frame, para siempre, aunque la cámara estuviera quieta y no hubiera pasado nada — costo de GPU constante confirmado con trazas de Performance (ver [[Deuda Técnica]], proceso GPU al 99.46%). Ahora hay un flag `needsRender` (arranca en `true`) que gatea ambos `render()`; `controls.update()` se sigue llamando siempre (liviano en reposo, necesario para que el damping decante), pero solo se dibuja si `needsRender` está en `true`. Se pone en `true` desde: el evento `change` de `OrbitControls` (three.js solo lo emite cuando la cámara realmente se movió — compara contra el frame anterior con un epsilon, así que no hay falsos positivos con la cámara inmóvil), `pressKey()` (una tecla anima el teclado 3D aunque la cámara esté quieta), la carga exitosa de `Pc.glb`, y el listener de `resize`. `updateIframeOcclusion()` se movió adentro del gate (solo hace falta recalcularla cuando la cámara se movió, que es justo cuando `needsRender` se activa vía el evento `change`).
>
> `renderer.setPixelRatio` pasó de `window.devicePixelRatio` a `Math.min(window.devicePixelRatio, 2)` — en pantallas 3-4x el fill-rate escala al cuadrado del pixelRatio, así que el cap es -55%+ de píxeles a rasterizar por frame con pérdida de nitidez casi imperceptible.

### Resize

Listener de `window.resize` recalcula `camera.aspect`/`updateProjectionMatrix()` y el tamaño de ambos renderers.

### Animación del teclado 3D

`pressKey(key, down)` resuelve `key` (un `KeyboardEvent.key`) a uno o más nodos del modelo vía `KEY_TO_NODES` y anima la posición Y de cada uno (simula que las teclas físicas se hunden al escribir). Se dispara por:
- Eventos de teclado nativos del portfolio (fuera del iframe).
- Eventos `postMessage` recibidos desde el iframe (`e.data.type === "keydown"/"keyup"`) — ver [[DesktopManager]]`._bindEventosTeclado`, que reenvía el teclado del iframe al `window.parent`, ya que los eventos de teclado de un iframe no burbujean naturalmente al `window` padre.

> [!success] `Pc.glb` renombrado a un layout real de Razer Huntsman Mini (2026-07-30)
> El modelo modela un teclado 60% (61 teclas, sin fila de F ni flechas dedicadas — igual que el Huntsman Mini real). Se renombraron 8 nodos para que coincidan con las teclas físicas reales en vez de placeholders: `undo→backspace`, `shift_izq/der→shift_left/right`, `ctr_izq→ctrl_left`, `ctrl_der→ctrl_right`, `alt→alt_left`, `alt.001→alt_right`, `page→menu` (edición directa del chunk JSON del `.glb` vía script Node, preservando el chunk binario intacto).
>
> **`KEY_TO_NODES`** (nuevo diccionario en `pressKey`) traduce cada `KeyboardEvent.key` a los nodos a animar — letras/dígitos caen por default (`key.toLowerCase()`, ya coincide con el nombre del nodo), el resto tiene mapeo explícito (`Escape→esc`, `Backspace→backspace`, `' '→space`, puntuación sin shift `-→'-_'` etc.). Como el Huntsman Mini real no tiene teclas físicas de F1-F12 ni flechas — se acceden con **Fn + tecla secundaria**, impresa en el costado de la tecla — el diccionario mapea esas teclas lógicas a **dos nodos a la vez**: `F1..F10→['1'..'0','fn']`, `F11→['-_','fn']`, `F12→['=+','fn']`, y flechas `ArrowUp/Left/Down/Right→['i'/'j'/'k'/'l','fn']` (patrón en T invertida sobre I/J/K/L, confirmado contra el layout Fn real del Huntsman Mini — Fn+J/K son izquierda/abajo). `pressKey` anima todos los nodos de la lista a la vez.

## `public/KneOS/index.html`

Documento aislado que corre dentro del `<iframe>` — **no hereda** el import map de `public/index.html` (documento distinto). Import map propio: `planck` (necesario porque [[Kfruit]] se ejecuta en este contexto) y, desde 2026-07-30, también `three` + `three/examples/jsm/loaders/GLTFLoader.js` (mismo `unpkg.com/three@0.158.0` que la escena externa) — agregados para [[Maxwell]], la primera app que corre Three.js **dentro** de KneOS en vez de en la escena que la contiene. Scripts globales clásicos (no módulos, para exponer variables globales): `js-dos-api.js` (`Dosbox`, usado por [[Doom]]) e `interactjs` (`interact`, usado por [[Window y Taskbar|Window]]). Entry point: `<script type="module" src="js/KNEOS.js">` (ver [[Frontend Core#Bootstrap — KNEOS.js]]).
