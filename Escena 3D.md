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
- **Luces**: `AmbientLight(0xffffff, 0.6)` + `DirectionalLight(0xffffff, 1)` en `(5,5,5)`.

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

### Resize

Listener de `window.resize` recalcula `camera.aspect`/`updateProjectionMatrix()` y el tamaño de ambos renderers.

### Animación del teclado 3D

`pressKey(key, down)` busca en el modelo `pc.getObjectByName(key)` y anima su posición Y (simula que las teclas físicas se hunden al escribir). Se dispara por:
- Eventos de teclado nativos del portfolio (fuera del iframe).
- Eventos `postMessage` recibidos desde el iframe (`e.data.type === "keydown"/"keyup"`) — ver [[DesktopManager]]`._bindEventosTeclado`, que reenvía el teclado del iframe al `window.parent`, ya que los eventos de teclado de un iframe no burbujean naturalmente al `window` padre.

## `public/KneOS/index.html`

Documento aislado que corre dentro del `<iframe>`. Import map propio con `planck` (necesario porque [[Kfruit]] se ejecuta en este contexto). Scripts globales clásicos (no módulos, para exponer variables globales): `js-dos-api.js` (`Dosbox`, usado por [[Doom]]) e `interactjs` (`interact`, usado por [[Window y Taskbar|Window]]). Entry point: `<script type="module" src="js/KNEOS.js">` (ver [[Frontend Core#Bootstrap — KNEOS.js]]).
