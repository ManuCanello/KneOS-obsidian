---
tags:
  - portfolio/kneos
  - apps
---

# Maxwell

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/Maxwell.js` — extiende [[File]]. Extensión `"maxwell"`, ícono `sources/appIcon/file.svg` (genérico — no tiene ícono propio todavía), `src = null`.

> [!abstract] Qué hace
> Visor 3D: carga `public/models/maxwell.glb` con Three.js y lo muestra girando sobre su propio eje (`rotation.y += 0.01` por frame) dentro de la ventana.

## Constructor(name)

`super(name, "maxwell", null, "sources/appIcon/file.svg", FileType.OTHER, size)` — `size` es el tamaño real actual de `maxwell.glb` en bytes (mismo patrón que Doom/Kfruit, ver la nota de `size` en [[File]]); hay que actualizarlo a mano si el `.glb` se vuelve a reemplazar. `FileType.OTHER` (2026-07-31, ver [[File]]) porque un visor 3D de demo no encaja en ninguna otra categoría.

## Three.js dentro de KneOS (import map propio)

`public/KneOS/index.html` corre en su propio documento (dentro del `<iframe>` de `public/js/main.js`) y por lo tanto tiene su **propio import map** — no hereda el de la página externa. Hasta esta app, ese import map solo tenía `planck` (para [[Kfruit]]); ahora también tiene `three` y `three/examples/jsm/loaders/GLTFLoader.js`, apuntando al mismo `unpkg.com/three@0.158.0` que ya usaba `public/index.html` (ver [[Escena 3D]]). Es la primera vez que Three.js corre **dentro** de KneOS, no solo en la escena que la contiene.

## `_crearContenido()` / `_iniciarEscena(container)`

- `Scene` + `PerspectiveCamera` (fov 50) + `WebGLRenderer` (`antialias`, `alpha:true`, `clearColor` transparente, `setPixelRatio`) — mismo patrón mínimo que `public/js/main.js` (ver [[Escena 3D]]), pero sin `OrbitControls`: no se pidió interacción, solo rotación automática.
- **Cámara con posición inicial fija** (`(0,1,4)`, `lookAt(0,0,0)`) antes de cargar nada — sin esto la cámara queda en el origen `(0,0,0)` (mirando a la nada) hasta que el modelo termina de cargar.
- **Placeholder wireframe** (`BoxGeometry` verde `#00ff41`, `MeshBasicMaterial({wireframe:true})`) agregado a la escena de entrada, antes de que `GLTFLoader` termine — confirma que el visor funciona aunque el modelo tarde en cargar, falle, o (como pasó en la práctica, ver debugging abajo) esté vacío.
- `GLTFLoader().load(MODEL_PATH, onSuccess, undefined, onError)`: en éxito, calcula `Box3().setFromObject(gltf.scene)` — **si `box.isEmpty()`, no reemplaza el placeholder** (se queda el wireframe) en vez de encuadrar la cámara con un tamaño inválido. `onError` loguea a consola en vez de fallar en silencio.
- **`_encuadrarCamara(modelo, box, camera)`**: el tamaño real del modelo se desconoce de antemano (a diferencia del visor fijo de `main.js`, tuneado a mano para el `Pc.glb`) — recentra el modelo en el origen (`position.sub(box.getCenter())`) y aleja la cámara en proporción a `box.getSize().length()`.
- **Resize**: `ResizeObserver` sobre el `container` (mismo patrón que el `<canvas>` de [[Kfruit]]).
- **Loop de render** (`requestAnimationFrame`): si `!container.isConnected` (la ventana se cerró y el contenido salió del documento), corta el loop en vez de seguir renderizando, ya que `_crearContenido()` (y por lo tanto un `WebGLRenderer` nuevo) se vuelve a ejecutar entero cada vez que la ventana se recrea (ver `Window.abrir()` en [[Window y Taskbar]] — `_crearContenido()` solo corre la primera vez o después de un `cerrar()`).
- **Pausa cuando está minimizada** (2026-08-03): si `container.offsetParent === null` (`display:none` en algún ancestro, pero sigue conectada al DOM), salta rotación y render de ese frame pero sigue pidiendo el próximo — para reanudar solo en cuanto se restaure. Encontrado investigando el lag reportado en Chrome (ver [[Deuda Técnica]]): antes de esto, minimizar la ventana no paraba nada, seguía girando y renderizando GPU para una ventana invisible.
- **`_liberarEscena(scene, renderer)`** (2026-08-03, nuevo): al cortar el loop por ventana cerrada, además de desconectar el `ResizeObserver` (ya existía) ahora recorre la escena liberando geometrías/materiales/texturas (`.dispose()`) y llama `renderer.dispose()` + `renderer.forceContextLoss()`. Antes de esto el contexto WebGL quedaba vivo hasta que el GC decidiera juntarlo — sin garantía de cuándo, y mientras tanto contaba como contexto activo (Chrome tiene un límite por pestaña). Abrir/cerrar la ventana varias veces sin esto acumulaba contextos y memoria de GPU sin liberar.
- **`setPixelRatio`**: capeado a `Math.min(window.devicePixelRatio, 2)` (antes `window.devicePixelRatio` sin tope) — mismo motivo que en `public/js/main.js`, ver [[Escena 3D]].

> [!bug] Debugging real: cámara en `Infinity` con el `.glb` vacío (2026-07-30, resuelto)
> `maxwell.glb` llegó como un glTF válido pero **vacío** (132 bytes, `{"scene":0,"scenes":[{"name":"Scene"}]}`, sin `nodes`/`meshes`/chunk binario — exportado desde Blender sin el objeto seleccionado). `Box3.setFromObject()` sobre una escena sin geometría devuelve la caja "vacía" por defecto de Three.js (`min=(+Inf,+Inf,+Inf)`, `max=(-Inf,-Inf,-Inf)`); su `getSize().length()` da `Infinity` (no `NaN`), así que `camera.position.set(0, size*0.2, size||3)` terminaba en `(0, Infinity, Infinity)` — cámara rota, nada visible, sin ningún error en consola. Se agregó el guard `box.isEmpty()` (arriba) y la posición inicial fija de la cámara para que el síntoma nunca vuelva a ser "ventana negra sin pistas" — ahora ese mismo caso muestra el wireframe placeholder. El usuario re-exportó `maxwell.glb` con geometría real (123.716 bytes, 1 mesh, 5 nodes) y confirmó que el visor lo carga y gira correctamente.

## Persistencia

Ninguna propia — no interactúa con ningún módulo de [[Frontend Model Services Utils]] más allá de lo que ya hace `File`/`DesktopManager` para cualquier ícono.
