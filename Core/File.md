---
tags:
  - portfolio/kneos
  - frontend
  - core
---

# File

⬅️ Volver a [[Frontend Core]]

`core/File.js` — clase raíz de todo "archivo" del sistema (no extiende nada). [[Folder]] extiende `File`, y por lo tanto [[DesktopGrid y DesktopFolder|DesktopFolder]] también (indirectamente). Todas las [[Apps|apps de escritorio]] extienden `File`.

## Constructor

`File(nombre, extension, src, icono, direction)` — el 5º parámetro (`direction`) se recibe pero no se usa (dead param).

- `nombre`, `extension`, `icono` guardados tal cual.
- `desktopPlace = null`, `parentId = null`, `id = null`.
- `src = "KneOS/" + nombreParaRuta"` (sobreescrito luego si viene de BD/carpeta padre).
- Metadata inicial: `size=0`, `createdAt`, `updatedAt`, `lastOpenedAt=null`, `favorite=false` (sobreescritos con datos reales en `DesktopManager._aplicarMeta`).
- `window = new Window(_generarVentanaId(), nombreCompleto, icono, () => this._crearContenido())`.

## Getters/setters

- `nombreCompleto`: `"${nombre}.${extension}"`.
- `nombreParaRuta`: igual con espacios → `_` (override en `Folder`, que usa solo `nombre`).
- `ventanaId`: delega en `window.id`.

## Métodos públicos

- `abrirVentana()`: `window.abrir()`; si `id != null`, dispara (fire-and-forget vía `advertirSiFalla`) `iconServices.changeLastOpened(id)`.
- `cerrarVentana()`, `minimizar()`, `restaurar()`: delegan 1:1 en `window`.
- `toggleVentana()`: el gesto de click sobre el ícono (usado por `DesktopManager._crearContenedorIcono` y `DesktopFolder._renderIconoEscritorio`, **no** por `abrirVentana()` en sí). Si la ventana ya existe (`estaAbierta()` o `estaMinimizada()`) delega en `window.toggleMinimizar()` — minimiza si estaba activa al frente, o restaura/enfoca si estaba minimizada o tapada por otra ventana; si no existe, la abre. La navegación por breadcrumb/sidebar dentro de [[Folder]] sigue llamando `abrirVentana()` directo (sin toggle): ahí minimizar en vez de enfocar sería sorpresivo. El menú contextual "Abrir" (`ContextMenuManager`) también usa `abrirVentana()` directo por la misma razón.
- `actualizarSrc(srcPadre)`: recalcula `src`, actualiza `updatedAt`, persiste vía `changeSrc` (override en `Folder`, que además propaga el cambio a sus hijos).

## Métodos usados por subclases y otras clases del core

- `_refrescarColumnas()`: refresca columnas fecha/tamaño en `iconoElement` y en su clon en `desktopFolder._clones` si es un ítem raíz.
- `_propagarTamano(delta)`: recorre la cadena de carpetas ancestro (vía `parentId`) sumando `delta` a `size` de cada una — modela "el tamaño de una carpeta es la suma de su contenido" (espejo de `getTamanosAgregados` en [[Módulo Icon]]).
- `_crearContenido()`: por defecto un `div` vacío; **cada app la sobreescribe** para construir su UI real (ver [[Apps]]).
- `_generarVentanaId()`: `"ventana_" + nombre.replaceAll(" ","_")`.

## Dependencias

[[Window y Taskbar|Window]] (composición), `utils/avisos.js` (`advertirSiFalla`), `utils/formato.js`, y globales `window.iconServices`, `window.archivosAbiertos`, `window.desktopFolder` — ver [[Frontend Model Services Utils]].
