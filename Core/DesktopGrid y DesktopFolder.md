---
tags:
  - portfolio/kneos
  - frontend
  - core
---

# DesktopGrid y DesktopFolder

⬅️ Volver a [[Frontend Core]]

## `DesktopGrid` (`core/DesktopGrid.js`)

Gestiona únicamente los "espacios" (slots) de íconos del escritorio — creación, búsqueda de espacio libre y drag&drop sobre ellos. No conoce nada de archivos/íconos: delega toda decisión de negocio al callback `onSoltar` recibido en el constructor.

**Constructor(contenedor, onSoltar)** — `onSoltar: (idIcono, espacioId) => void`.

**Métodos públicos:**
- `crear(cantidad=70)`: crea `cantidad` divs `.espacioParaShortCut` con id `espacioN`, habilita drop en cada uno.
- `buscarVacio(cantidad=70)`: recorre `espacio1..espacioN`, devuelve el `id` del primero sin hijos o `null`.

**Privado:** `_habilitarDrop(espacio)` — engancha `dragover`/`drop`; parsea el JSON de ids desde `dataTransfer` (puede haber más de un id si hay selección múltiple, ver [[Drag and Drop y Selección Múltiple]]); por cada ícono movido invoca `_onSoltar`; al final llama `limpiarSeleccion()`.

Instanciado y consumido por [[DesktopManager]] (que le pasa `_onIconoSoltado` como callback).

---

## `DesktopFolder` (`core/DesktopFolder.js`, extiende [[Folder]])

Representa el "Escritorio" como si fuera una carpeta más (mismo sidebar, breadcrumb, buscador y menú de vista de `Folder`), pero:
- No tiene contenido propio en BD: sus "archivos" son directamente las instancias ya vivas de `window.archivosAbiertos` con `parentId == null`.
- No admite drop (el escritorio es la raíz).
- No permite crear archivos nuevos desde su menú contextual (para eso está el menú del escritorio real, ver [[Menús Contextuales]]).

**Constructor()**: `super("Escritorio")`, fija `extension = "desktop"`.

**Overrides clave:**
- `get nombreCompleto()`: solo `nombre` (sin `.desktop`).
- `async _loadContent()`: en vez de pedir a la BD, toma `[...window.archivosAbiertos.values()]` filtrando `parentId == null`, y por cada uno renderiza un **clon visual** (`_renderIconoEscritorio`). Reinicia `_clones = new Map()` en cada apertura.
- `_elementoDeOrden(archivo)`: devuelve `_clones.get(archivo)` en vez de `archivo.iconoElement`.
- `_renderIconoEscritorio(archivo)`: crea un `div.icono` clon **sin id propio** — usa `container.dataset.iconoId = archivo.iconoElement.id` para que `seleccionMultiple.idDe` resuelva el id real. Guarda el clon en `_clones`.
- `_filtrarArchivos(texto)`: filtra ocultando/mostrando los **clones**, no `archivo.iconoElement`.
- `_rutaCarpetas()`: devuelve `[]` — es la raíz.
- `crearDropVentana()` / `crearDrop()`: no-op — anulan la capacidad de aceptar drops.
- `_permiteCrear()`: `false` — oculta la opción "Nuevo" del menú contextual.

Instanciado una única vez por [[DesktopManager]] (`window.desktopFolder`), referenciado por `Folder` (sidebar "Escritorio", breadcrumb raíz, navegación con Backspace) y por [[File]]`._refrescarColumnas` (para actualizar el clon si el archivo es de la raíz).
