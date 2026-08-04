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

**Constructor(contenedor, onSoltar)** — `onSoltar: (idIcono, destino, icono) => void|Promise`.

> [!info] `onSoltar` pasó a ser dueño del movimiento real (2026-07-30)
> Antes `DesktopGrid` hacía `destino.appendChild(icono)` él mismo, sincrónico, y recién después invocaba `onSoltar(idIcono, espacioId)` (`espacioId` como *string*). Ahora `DesktopGrid` **no mueve nada** — le pasa a `onSoltar` el elemento `destino` (no su id) y el `icono` arrastrado, y es el callback (`DesktopManager._onIconoSoltado`, ver [[DesktopManager]]) quien decide si animar la desaparición en el origen antes de mover (`File.animateRemoval()`, ver [[File]]) y recién ahí hacer el `appendChild`. Necesario para que un archivo que viene de **adentro de una carpeta** (ventana abierta) se vea desaparecer ahí antes de reaparecer en el escritorio, en vez de teletransportarse al instante.

**Métodos públicos:**
- `crear(cantidad=70)`: crea `cantidad` divs `.espacioParaShortCut` con id `espacioN`, habilita drop en cada uno.
- `buscarVacio(cantidad=70)`: recorre `espacio1..espacioN` **por índice**, devuelve el `id` del primero sin hijos o `null`. Usado para colocación simple (nuevo ícono, `DesktopManager.moverAlEscritorio`, ver [[DesktopManager]]) donde no importa la cercanía a ningún punto.

**Privado:**
- `_habilitarDrop(espacio)` — engancha `dragover`/`drop`; parsea el JSON de ids desde `dataTransfer` (puede haber más de un id si hay selección múltiple, ver [[Drag and Drop y Selección Múltiple]]).
  - **Rechazo si el espacio está ocupado por otro (2026-07-29):** si `espacio` ya tiene un hijo que **no** es ninguno de los íconos arrastrados (típicamente otro `File`, que a diferencia de `Folder` no tiene su propio drop de "meter adentro" — ver `DesktopManager._finalizarIcono`, solo `Folder` recibe `crearDrop`), el drop completo se cancela (`return` antes de mover nada): todos los íconos arrastrados quedan en su posición original. Antes de este fix, soltar sobre otro `File` mandaba el ícono arrastrado a un espacio vacío cualquiera elegido por índice (`buscarVacio()`), es decir lo teletransportaba a otra parte del escritorio en vez de dejarlo donde estaba.
  - **Colocación en grupo (2026-07-29):** si se soltaron varios íconos juntos (selección múltiple), el primero va al espacio soltado y el resto usa `_buscarVacioCercano(espacio, reservados)` — antes usaba `buscarVacio()` (primer libre por índice), que podía mandar al segundo ícono al otro extremo del escritorio. Por cada ícono movido invoca `_onSoltar`; al final llama `clearSelection()`.
  - **`reservados` (`Set`, 2026-07-30):** desde que el movimiento real quedó pendiente de una animación (ver arriba), `espacio.children.length` ya no se actualiza sincrónicamente dentro del `forEach` — sin este set, dos íconos sueltos juntos (viniendo de una carpeta) podrían elegir el mismo espacio libre, porque el primero todavía no llegó a "ocuparlo" de verdad cuando el segundo pregunta. Cada destino elegido se agrega a `reservados` **antes** de invocar `_onSoltar` (no después, no hace falta esperar la animación).
- `_buscarVacioCercano(referencia, excluir=new Set())` (2026-07-29, firma extendida 2026-07-30): recorre todos los `.espacioParaShortCut` vacíos **y no incluidos en `excluir`**, devuelve el de menor distancia euclídea (al cuadrado, sin `sqrt`) entre su centro y el de `referencia` (`getBoundingClientRect`). Hace que un drop de varios íconos los agrupe visualmente alrededor del punto soltado en vez de dispersarlos por índice.

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
- `_renderIconoEscritorio(archivo)`: crea un `div.icono` clon **sin id propio** — usa `container.dataset.iconoId = archivo.iconoElement.id` para que `multiSelect.idOf` resuelva el id real. Guarda el clon en `_clones`.
- `_filtrarArchivos(texto)`: filtra ocultando/mostrando los **clones**, no `archivo.iconoElement`.
- `_rutaCarpetas()`: devuelve `[]` — es la raíz.
- `crearDropVentana()` / `crearDrop()`: no-op — anulan la capacidad de aceptar drops.
- `_permiteCrear()`: `false` — oculta la opción "Nuevo" del menú contextual.

Instanciado una única vez por [[DesktopManager]] (`window.desktopFolder`), referenciado por `Folder` (sidebar "Escritorio", breadcrumb raíz, navegación con Backspace) y por [[File]]`._refrescarColumnas` (para actualizar el clon si el archivo es de la raíz).
