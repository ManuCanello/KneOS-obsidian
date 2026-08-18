---
tags:
  - portfolio/kneos
  - frontend
---

# Escena 3D

⬅️ Volver a [[KneOS Portfolio]] · Ver [[Arquitectura]]

## `public/index.html`

*(2026-08-13)* `<meta name="viewport">` y `<meta name="description">` agregados — faltaban, bajaban SEO (90) y el audit "Optimize viewport for mobile" de Performance. Ver [[Deuda Técnica#SEO y Accessibility a 100/100 + CSS bundleado (2026-08-13, mismo día que el pase de performance de abajo)]].

Único script: `js/dist/main.bundle.js` — bundle local generado por esbuild a partir de `public/js/main.js`, **con code-splitting** (ver más abajo) (ver [[Deuda Técnica#Bundling local de three.js/planck/interactjs (2026-08-13)]]). Hasta 2026-08-13 cargaba `three`/`planck` vía import map apuntando a `unpkg.com`; ahora `three`+`planck` (npm) se resuelven en build time (`npm run build`, corre también en `postinstall`) y no hay ningún import map ni request a CDN en runtime para esta página.

*(2026-08-13)* Además de `js/dist/main.bundle.js`, el `<head>` tiene `<link rel="preload" href="./models/pc.glb" as="fetch" crossorigin>` — el navegador arranca a descargar el modelo en paralelo al JS en vez de recién cuando `GLTFLoader.load()` corre (antes no era discoverable hasta que el JS se ejecutaba). Y el `<body>` arranca con un `#loadingScreen` (texto "KNEOS" centrado, estilos inline en el `<head>`, sin depender de ningún CSS externo) que pinta antes que cualquier JS corra — ver [[Deuda Técnica#Nivel 1 completo: performance de `/` y `/KneOS/` a ~78-99% (2026-08-13)]] para el porqué (ancla el LCP de Lighthouse a algo que pinta al toque en vez de a los ~7s que tarda la escena 3D real).

## `public/js/main.js`

Dos renderers superpuestos sobre `document.body`, ambos absolutos a pantalla completa:

- **`cssDiv`** (`pointer-events: auto`) — contiene el `CSS3DRenderer`, que renderiza el `<iframe>` de KneOS como objeto 3D.
- **`webglDiv`** (`pointer-events: none`, para no bloquear clics sobre el iframe) — contiene el `WebGLRenderer` (`alpha:true`, `clearColor` transparente), para que se vea el CSS3D detrás (efecto: el modelo 3D "flota" sobre el iframe).

### Escena

- `scene = new THREE.Scene()`.
- `camera`: `PerspectiveCamera(fov 50)`, posición inicial `(0, 1.6, 3.5)`.
- **Luces** (2026-07-30, setup de 3 puntos — antes solo `AmbientLight(0.6)` + una `DirectionalLight`): `HemisphereLight(0xffffff, 0x444444, 0.5)` como base ambiental + `DirectionalLight` **key** (`intensity 1.2`, `(5,5,5)`, frontal) + `DirectionalLight` **fill** (`0.4`, `(-5,2,3)`, opuesta y más tenue) + `DirectionalLight` **rim** (`0.6`, `(0,3,-5)`, atrás) — le da volumen a la PC en vez de la iluminación plana anterior.

> [!success] Bootstrap crítico vs. diferido (2026-08-13)
> `main.js` se separó en dos partes para reducir el Total Blocking Time. **Crítico (síncrono, corre de entrada):** `webglDiv`/`WebGLRenderer`, `scene`, `camera`, `hemiLight`+`keyLight` únicamente, kickoff de `GLTFLoader.load()`, arranque del loop `animate()`. **Diferido** (`setTimeout(0)`, un tick después): `import()` dinámico de `OrbitControls` y `CSS3DRenderer`/`CSS3DObject` (antes imports estáticos arriba del archivo — ver nota de code-splitting más abajo), `fillLight`+`rimLight`, la creación del `<iframe>` + `CSS3DObject` (ver sección de abajo), y toda la animación de teclado (`KEY_TO_NODES`, `pressKey`, listeners). `controls`, `rendererCSS` e `iframeCSS` son `let` a nivel de módulo, `undefined` hasta que el bloque diferido corre — `animate()` los usa con optional chaining (`controls?.update()`, `rendererCSS?.render(...)`) para no romper si todavía no existen.
>
> **Code-splitting real**: el build de `main.js` pasó de `--outfile` a `--bundle --splitting --format=esm --outdir=public/js/dist --entry-names=main.bundle` (mismo patrón que ya tenía el bundle de KneOS) — sin `--splitting`, esbuild igual mete el código de un `import()` dinámico en el mismo archivo de salida, solo que envuelto en una promesa que resuelve al toque; no alcanza con cambiar `import` estático por `import()` si el build sigue siendo un único `--outfile`. Con splitting, `OrbitControls`/`CSS3DRenderer` quedan en chunks aparte (`public/js/dist/chunks/`) que el navegador pide recién cuando el bloque diferido corre. El entry `main.bundle.js` bajó de ~546KB a ~47KB (el resto sigue siendo three.js core + `GLTFLoader`, que sí hacen falta para el primer frame — ~485KB en un chunk compartido).

### Proyección del escritorio HTML sobre la pantalla del PC 3D

Se crea un `<iframe title="Escritorio KneOS">` (2026-08-13: `title` agregado — sin él, Accessibility marcaba "`<frame>` o `<iframe>` elements do not have a title"; es la única excepción a la regla de no usar `title` en KneOS, ver nota de arriba) — tamaño fijo `1433px × 1139px`, sin borde — envuelto en `CSS3DObject(iframe)`, posicionado en `(0, 1.5870, 0.7716)`, rotado en X (inclinación de la pantalla del monitor 3D) y **escalado a `0.001`** en los 3 ejes — técnica estándar para insertar `CSS3DObject` (unidades en píxeles de pantalla) dentro de una escena medida en unidades pequeñas. Se agrega directamente a `scene` (hermano, no hijo del PC), posicionado para coincidir con la pantalla del modelo. *(2026-08-13)* Toda esta creación vive ahora dentro del bloque diferido descrito arriba, no en el bootstrap crítico.

> [!success] Carga del iframe diferida (2026-08-13, ajustado el mismo día)
> Hasta esta fecha `iframe.src = './KneOS/index.html'` se asignaba en el mismo tick que se creaba el elemento — o sea, la escena raíz arrancaba a descargar *todo* KneOS (su propio bundle three.js+planck+interactjs, más `js-dos-api.js`) antes de que el usuario viera nada, duplicando la carga de three.js/planck (una vez para la habitación, otra para el escritorio embebido) y disparando el LCP a ~7s (medido con Unlighthouse). Primer intento: `requestIdleCallback` (fallback `setTimeout(500)`) — la escena de la habitación consigue su primer render sin competir por red/CPU con el arranque de la app completa que el usuario todavía no vio.
>
> **Ajustado a `setTimeout(loadKneOS, 50)`** tras confirmar con el score real que `requestIdleCallback` salía contraproducente: bajo el throttling de CPU que usa Lighthouse (4x más lento), three.js satura el hilo principal casi todo el tiempo, así que el browser rara vez encuentra un hueco "idle" real — el iframe terminaba arrancando mucho más tarde que antes del fix, y el LCP de `/` empeoraba (7.3s, score 4/100) en vez de mejorar. Un delay fijo y corto arranca la carga casi enseguida (sigue siendo un tick async, no bloquea el script inicial) sin depender de que haya tiempo libre de verdad. Confirmado con 3 corridas seguidas de Unlighthouse sin tocar nada más: performance de `/` pasó de ~0.56-0.76 (con `requestIdleCallback`, mucho ruido) a un consistente ~0.71-0.73 (con `setTimeout(50)`).

> [!warning] Scroll nativo roto detrás del `matrix3d()` del CSS3DObject (2026-08-18)
> Dentro del iframe embebido acá, el scroll por rueda del mouse no disparaba en ninguna ventana con contenido scrolleable (`Folder`, listas, etc.) — funcionaba perfecto abriendo `KneOS/index.html` standalone. Diagnóstico (Playwright, instrumentando `wheel` con listeners en ambos documentos): el evento SÍ llega al documento del iframe, con el `target` y el `deltaY` correctos y sin `preventDefault` — pero Chromium no ejecuta la acción de scroll nativa por defecto cuando el iframe vive detrás de un ancestro con `transform: matrix3d(...)` en la página padre (el hit-testing del compositor que decide qué ancestro scrollear se pierde detrás de esa capa 3D). No es un problema de coordenadas del mouse ni de algún listener con `preventDefault` compitiendo — el evento aterriza exactamente donde debería y aun así el navegador no scrollea solo. Fix: [[Frontend Core#WheelScroll (2026-08-18)]], un polyfill de scroll manual (`el.scrollTop += e.deltaY` sobre el ancestro `overflow-y:auto/scroll` más cercano al `target`) que reemplaza la acción nativa en los dos escenarios (embebido y standalone), para no depender de que el navegador la resuelva bien en ninguno de los dos.

### Carga de modelos 3D

`GLTFLoader` carga `./models/pc.glb` (2026-08-13: nombre normalizado a minúscula — git lo trackeaba como `Pc.glb`, el archivo en disco ya estaba en minúscula por fuera de git; se corrigió con `git mv` para que coincidan, mismo criterio que `maxwell.glb`/`lampara.glb`):
- `pc = gltf.scene`, agregado a la escena; `needsRender = true`; se saca `#loadingScreen` (ver arriba) — este es el primer paint "real" de la página.
- *(2026-08-13, diferido)* En un `setTimeout(0)` aparte (no bloquea el paint de arriba): busca `"screen_iframe"` y lo oculta (`visible=false`, placeholder del propio `.glb`) + busca `"screen"` y reemplaza su material por un `MeshPhysicalMaterial` de vidrio (transmission 1.0, roughness 0, opacity 0.15, `DoubleSide`) — efecto de reflejo semitransparente sobre el iframe. Es cosmético, no hace falta para el primer frame.

> [!success] `pc.glb` comprimido con gltf-transform (2026-08-13)
> `weld`+`quantize`+`prune` (sin Draco — la quantización la lee nativamente `GLTFLoader` vía `KHR_mesh_quantization`, sin decoder aparte): 182KB→98KB (-46%). **Cuidado real encontrado en el camino**: el preset `optimize` de gltf-transform por default corre también `join`/`flatten`/`simplify`, que fusionaron las 64 mallas nombradas del modelo (cada tecla del teclado es un mesh separado, ver más abajo) en **una sola** — habría roto por completo la animación de teclas, que depende de `pc.getObjectByName('esc')` etc. encontrando cada tecla como nodo independiente. Se corrió de nuevo con `--join false --flatten false --simplify false` y se verificaron los 64 nombres de nodo antes de reemplazar el archivo.

> [!warning] `lampara.glb` sin integrar
> El asset existe en `public/models/lampara.glb` pero **no se carga** en ningún lugar del código leído. Ver [[Deuda Técnica]].

### Loop de render (`animate()`)

`requestAnimationFrame` recursivo: `controls?.update()` → `rendererCSS?.render(scene, camera)` → `renderer.clear()` → `renderer.render(scene, camera)` (2026-08-13: optional chaining porque `controls`/`rendererCSS` no existen hasta que corre el bloque diferido, ver arriba). El `renderer.clear()` manual asegura que el canvas WebGL transparente se limpie a `alpha 0` antes de dibujar de nuevo (con `autoClear=false` en la config, evita un clear automático redundante).

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

> [!success] `pc.glb` renombrado a un layout real de Razer Huntsman Mini (2026-07-30)
> El modelo modela un teclado 60% (61 teclas, sin fila de F ni flechas dedicadas — igual que el Huntsman Mini real). Se renombraron 8 nodos para que coincidan con las teclas físicas reales en vez de placeholders: `undo→backspace`, `shift_izq/der→shift_left/right`, `ctr_izq→ctrl_left`, `ctrl_der→ctrl_right`, `alt→alt_left`, `alt.001→alt_right`, `page→menu` (edición directa del chunk JSON del `.glb` vía script Node, preservando el chunk binario intacto).
>
> **`KEY_TO_NODES`** (nuevo diccionario en `pressKey`) traduce cada `KeyboardEvent.key` a los nodos a animar — letras/dígitos caen por default (`key.toLowerCase()`, ya coincide con el nombre del nodo), el resto tiene mapeo explícito (`Escape→esc`, `Backspace→backspace`, `' '→space`, puntuación sin shift `-→'-_'` etc.). Como el Huntsman Mini real no tiene teclas físicas de F1-F12 ni flechas — se acceden con **Fn + tecla secundaria**, impresa en el costado de la tecla — el diccionario mapea esas teclas lógicas a **dos nodos a la vez**: `F1..F10→['1'..'0','fn']`, `F11→['-_','fn']`, `F12→['=+','fn']`, y flechas `ArrowUp/Left/Down/Right→['i'/'j'/'k'/'l','fn']` (patrón en T invertida sobre I/J/K/L, confirmado contra el layout Fn real del Huntsman Mini — Fn+J/K son izquierda/abajo). `pressKey` anima todos los nodos de la lista a la vez.

## `public/KneOS/index.html`

Documento aislado que corre dentro del `<iframe>` — documento distinto de `public/index.html`, con su propio entry point: `<script type="module" src="js/dist/kneos.bundle.js">` (ver [[Frontend Core#Bootstrap — KNEOS.js]]).

*(2026-08-13)* Ya no hay import map: `three` (usado por [[Maxwell]]) y `planck` (usado por [[Kfruit]]) pasaron de `unpkg.com/three@0.158.0` a paquetes npm resueltos en build time por el mismo bundle esbuild que genera `js/dist/kneos.bundle.js` (ver [[Deuda Técnica#Bundling local de three.js/planck/interactjs (2026-08-13)]]). `interactjs` (`interact`, usado por [[Window y Taskbar|Window]]) también se bundleó — dejó de cargarse como `<script>` clásico de jsdelivr; `KNEOS.js` ahora hace `import interact from "interactjs"` y lo expone como `window.interact` a mano para no tocar los call-sites que lo siguen usando como global (ver [[Frontend Core#Bootstrap — KNEOS.js]]). Sigue quedando **un** `<script>` clásico externo: `js-dos-api.js` (`Dosbox`, usado por [[Doom]]) — no es un paquete npm bundleable (runtime emscripten grande), se dejó en el CDN de js-dos.com pero con `defer` agregado.

> [!success] Placeholder de carga con `<img>`, no texto (2026-08-13)
> `#loadingScreen` (mismo patrón que en `public/index.html`, ver arriba) se probó primero con texto chico ("KNEOS") y **no funcionó**: `lcp-discovery-insight` de Lighthouse seguía marcando el ícono de Doom (`.iconoImagen`, 80×80px, `background-image`) como el LCP real, 6s tarde. Causa: la caja que Lighthouse mide para un nodo de texto es la de sus glifos renderizados, que con ese tamaño de fuente terminaba siendo *más chica* que 80×80px — cualquier ícono real del escritorio la superaba apenas aparecía y se convertía en el nuevo LCP candidate. Fix: reemplazar el texto por una `<img src="sources/appIcon/kneAi.png">` de 200×200px (con "KNEOS" como caption chico debajo) — un elemento no-texto (imagen o `background-image`) mide su caja completa, así que 200×200 garantiza ganarle a cualquier ícono de 80×80. Resultado: `/KneOS/` pasó de performance 0.77 a **0.98-0.99** (antes de esto, el placeholder de texto no había movido el número en absoluto). Se saca (fade-out + remove) desde `KNEOS.js`, justo después de `desktop.iniciar()`. Ver [[Deuda Técnica#Nivel 1 completo: performance de `/` y `/KneOS/` a ~78-99% (2026-08-13)]].
