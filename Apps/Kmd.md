---
tags:
  - portfolio/kneos
  - apps
---

# Kmd

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/Kmd.js` — extiende [[File]]. Extensión `"kmd"`, ícono `sources/appIcon/terminal.svg`, `src = null`, tamaño declarado `96_000` bytes (no persiste contenido propio en BD, ver `File`).

> [!success] Implementada (2026-07-31)
> Terminal estilo CMD de Windows que opera sobre el **sistema de archivos real de KneOS** (la tabla `files`, vía `IconServices`/`DesktopManager`/`Folder`) — no es un FS simulado aparte. Un `mkdir` desde la terminal crea una carpeta real y persistida; un `del` la manda a la Papelera; un `cd` navega la misma jerarquía que muestra [[Folder]].

## Estado — se reinicia en cada apertura (2026-07-31, cambiado)

`_crearContenido()` se re-ejecuta desde cero cada vez que se abre la ventana — la primera vez, o después de haberla cerrado (no en un simple minimizar/restaurar, que no reconstruye contenido, ver [[Window y Taskbar]]). A diferencia del diseño original (todo el estado sobrevivía al cierre, como en [[Folder]]/[[RecycleBin]]), acá se decidió lo contrario: **cada apertura se comporta como una consola nueva**, sin el scrollback ni el historial de la vez anterior — mismo comportamiento que abrir una ventana de `cmd.exe` nueva.

`_crearContenido()` llama a `_resetSession()`, que:
- Resetea `_cwd` a `null` (vuelve a la raíz del escritorio).
- Vacía `_history`/`_historyIndex` (historial de comandos, flechas ↑/↓) y `_aiHistory` (contexto de `kneai` — de por sí solo vive en memoria, nunca se persiste en BD ni crea chats en `kneai_chats`).
- Limpia `_output` (`replaceChildren()`) e imprime el banner de nuevo.
- Reconstruye el prompt (`_buildPromptLine()`).

El nodo `_output` en sí (`<div class="kmdOutput">`) se sigue creando una sola vez, en el constructor, y se reutiliza (mismo elemento) en cada apertura — lo único que cambia es que ahora se le vacía el contenido en vez de preservarlo.

## DOM

```
div.app.kmdApp
└── div.kmdOutput
    ├── div.kmdLine          (comandos ya ejecutados + su salida, texto plano)
    └── div.kmdPromptLine    (siempre el último hijo mientras no hay un comando corriendo)
        ├── span.kmdPrompt   "C:\Escritorio>"
        └── input.kmdInput
```

Se usa un `<input>` real (no `contenteditable`) porque [[Folder]] instala un handler global de Backspace que navega a la carpeta padre y solo hace `return` temprano si el target es `INPUT` o `contentEditable` — un input real evita chocar con eso y el manejo manual de caret.

Mientras un comando async corre (`_submit()`), el `<input>` queda `disabled` para no pisar dos comandos a la vez; se reengancha al final con el prompt recalculado (por si `cd` cambió el `_cwd`).

## Resolución de rutas — `js/utils/kmdPath.js` (archivo nuevo)

Dos helpers puros, sin estado, separados de la clase:

- `tokenize(line)`: separa en tokens respetando comillas dobles (los nombres de KneOS admiten espacios: `cd "Mis Documentos"`).
- `resolveEntry(pathStr, cwd)`: resuelve una ruta relativa o absoluta (separador `\`, soporta `.`/`..`) a la instancia `File`/`Folder` correspondiente, cargando cada carpeta intermedia que todavía no se haya abierto esta sesión (`await folder._loadContent()`, idempotente por el guard `first_open` — mismo truco que `utils/resolveArchivo.js`, compartido con `TaskbarSearch`/`Home`). Compara nombres sin distinguir mayúsculas, contra `nombre` y `nombreCompleto` (los hijos de una carpeta se muestran con extensión, ver `DesktopManager._crearContenedorIcono`). Devuelve `null` si algún segmento no existe o se intenta bajar dentro de algo que no es una carpeta.

Detectar "¿esto es una carpeta?" se hace por duck-typing (`typeof archivo._loadContent === "function"`) en vez de `instanceof Folder`, para no tener que importar `Folder.js` en `Kmd.js` — no haría falta por ciclo (`Folder.js` no importa `Kmd.js` de vuelta), pero mantiene `kmdPath.js` desacoplado de una clase concreta.

## Comandos

| Comando | Notas de implementación |
|---|---|
| `dir` | `folder._loadContent()` + `folder.files`, 5 columnas (fecha, hora, `<DIR>` si es carpeta, tamaño si no lo es, nombre) alineadas con **CSS grid** (`_printDirGrid`, ver más abajo) en vez de rellenar con espacios a mano. |
| `cd [ruta]` / `cd ..` / `cd \` | Todo pasa por `resolveEntry` (incluido `..` y `\`) — sin lógica especial duplicada en el comando. |
| `cls` | `_output.replaceChildren(this._promptLine)`. |
| `tree` | Una sola llamada a `iconServices.getIcons()` (todo el árbol plano) agrupada en memoria por `parent_id`, en vez de N fetches recursivos de `getIconsByParent`. Solo carpetas (como el `tree` real sin `/F`). |
| `mkdir` / `md` | Raíz → `desktopManager.crearIcono(..., edit=false)`; dentro de una carpeta → `desktopManager.crearIconoEnCarpeta(..., edit=false)` (ver cambio en `DesktopManager` abajo). |
| `touch` (2026-07-31, nuevo) | Crea un `.txt` vacío — idéntico a `mkdir` pero `tipo="txt"` y el ícono de `txt.svg`; mismo criterio de "ya existe" (compara contra archivos, no carpetas). Es el comando "crear archivo" que faltaba en la tabla original. |
| `rmdir` / `rd`, `del` | `desktopManager.eliminarIconos([id])` → soft delete a la Papelera, en cascada. Ambos rechazan `filesUndeletable` (`kmd`, `exe`, `ai`, `kfruit`, `maxwell`, `recyclebin`, `desktop`) con "Acceso denegado."; `del` además rechaza carpetas (hay que usar `rmdir`). |
| `move` | Destino carpeta → mismo patrón que `Folder._moverArchivo` (`animateRemoval()` → `appendChild` en el nuevo contenedor → `_assignParent`, que ya persiste `parent_id`/`desktop_place`/`src` y propaga tamaños). Destino `\` → `desktopManager.moverAlEscritorio(archivo)`. |
| `ren` | `archivo.renombrar(nombreSinExtension)` — único punto de verdad de renombrado en `File`, ya actualiza label, título de ventana, `src`, columnas y BD. |
| `type` | Solo `.txt`. `TxtServices.getContent(id)` devuelve HTML de un `contenteditable`; se convierte a texto plano con un helper de módulo (`htmlToPlainText`) que pasa `<br>`/cierre de `<div>`/`<p>` a `"\n"` antes de sacar el resto de las etiquetas. |
| `echo` | Imprime el texto crudo de la línea (no el tokenizado, para no perder comillas/espacios múltiples); sin args imprime `ECO está activado.`. |
| `exit` | `this.cerrarVentana()` (idéntico a cualquier otro archivo, ver `File.cerrarVentana`). |
| `kneai` | `window.groq.ask(contexto, texto, this._aiHistory)` (mismo `Groq`/`/groq/chat` global que usa [[KneAI]] y [[TxtFile]]). `null` → mensaje de fallo, sin romper (mismo convenio que esas apps); ojo con el rate limit de 10 req/min por sesión (`groqLimiter`, ver [[Módulo Groq]]). |
| `curl [-i\|-I] <url>` (2026-07-31, nuevo) | Ver sección propia más abajo. |
| `help` | Tabla de texto fija (no derivada de la tabla de despacho interna), para no perder el formato exacto pedido originalmente por el usuario. |

Errores de comando (ruta/archivo inexistente, comando no reconocido, "Acceso denegado.") se muestran como **texto CMD real**, en el verde monocromático de siempre — no contradice la regla de "no mostrar ningún indicador de error" de [[Reglas]]: en una terminal el texto **es** la interfaz, no un adorno de error agregado encima; no hay rojo, ni bordes, ni shake.

## `tree`, `help` y `dir` alineados con CSS grid/bloque (2026-07-31)

Tres helpers en `Kmd.js` arman su salida como un solo nodo en vez de imprimir línea por línea con `_printLine` + `padEnd`/`padStart`: un padding manual con espacios se desalinea si la fuente no es perfectamente monoespaciada, así que en su lugar se deja que el navegador calcule el ancho real de cada columna a partir del contenido.

- **`_printCenteredBlock(lineas)`** (usado por `tree`): un único nodo `white-space: pre` con todas las líneas del árbol unidas por `"\n"`. El navegador calcula el ancho intrínseco del nodo a partir de su línea más larga, así todas las líneas quedan alineadas entre sí (los conectores `├──`/`└──` no se desalinean). `_cmdTree` arma un array de líneas en vez de imprimir con `_printLine` una por una.
- **`_printCenteredGrid(filas)`** (usado por `help`): CSS grid de 2 columnas (`.kmdHelpGrid`, `grid-template-columns: max-content 1fr`) — la columna de comando se ajusta sola al texto más largo, la de descripción ocupa el resto y envuelve el texto largo si hace falta. Retoma la tabla de 2 columnas (Comando/Descripción) tal como el usuario la dio originalmente.
- **`_printDirGrid(filas)`** (usado por `dir`, agregado después): CSS grid de 5 columnas (`.kmdDirGrid`, `grid-template-columns: max-content max-content max-content max-content 1fr`) para fecha/hora/`<DIR>`/tamaño/nombre; la columna de tamaño se alinea a la derecha con `:nth-child(5n+4)` sobre la lista plana de `<span>` (cada fila ocupa 5).

Los tres se envuelven en `.kmdLeftWrap` (`text-align: left`) — el nombre quedó de un cambio posterior del usuario: al principio estos tres bloques se centraban horizontalmente (`text-align: center`), pero se volvió a alinear todo a la izquierda, como el resto de la terminal (estilo CMD real, flush-left).

## `curl` (2026-07-31, nuevo; `-H`/`-X`/`-d` agregados el mismo día)

`curl [-i|-I] [-H "Clave: Valor"] [-X MÉTODO] [-d datos] <url>` — `fetch()` del propio navegador, **sin proxy de backend**. Decisión deliberada: un proxy server-side que reciba una URL arbitraria del usuario y la vaya a buscar sería un riesgo real de SSRF (el servidor podría terminar pegándole a su propia red interna, a `169.254.169.254`, etc.); haciéndolo client-side, la petición sale del navegador del propio usuario y queda sujeta a las mismas reglas de CORS que cualquier `fetch()` tipeado a mano en la consola — no es una capacidad nueva, es la misma que ya tiene cualquiera con DevTools abierto.

- Sin esquema (`curl example.com`) asume `http://`, igual que el curl real.
- `-I`: `HEAD`, imprime solo el status line + encabezados.
- `-i`: `GET` (o el método que corresponda), imprime status line + encabezados + línea en blanco + cuerpo.
- Sin flags: `GET`, imprime solo el cuerpo (fiel al curl real, que no muestra encabezados salvo `-i`/`-I`/`-v`).
- **`-H "Clave: Valor"`** (repetible): agrega un encabezado al request (`fetch(..., { headers })`). Parseo consciente de flags — el parser recorre `args` con un índice manual y, al encontrar `-H`/`-X`/`-d`, consume el **token siguiente** como su valor, en vez de tratar cualquier token sin `-` como candidato a URL. Bug corregido el mismo día: antes `-H` no existía como flag reconocida, así que el valor del header (ej. `"x-api-key: xxx"`, que no empieza con `-`) se colaba como si fuera la URL — de ahí el error visto `curl: (1) Protocolo no soportado: x-api-key:` (el parser de `URL` interpreta cualquier `algo:` como un esquema custom válido, así que ni siquiera tiraba una excepción al construir el `URL`).
- **`-X MÉTODO`**: fuerza el método HTTP (`GET`/`POST`/`PUT`/etc.).
- **`-d datos`**: cuerpo del request; si no se pasó `-X`, implica `POST` — mismo comportamiento que el curl real.
- Un `fetch()` fallido (red caída, el servidor no manda `Access-Control-Allow-Origin`, o rechaza un encabezado custom en el preflight de CORS) no se puede distinguir desde JS — todos tiran el mismo `TypeError`, así que el mensaje de error (`curl: (7) ...`) cubre las tres posibilidades a la vez.
- El cuerpo se recorta a `CURL_LIMITE_CHARS` (4000) y `CURL_LIMITE_LINEAS` (200 líneas): sin este límite, una página grande generaría miles de `<div class="kmdLine">` (uno por línea, ver `_printLine`) y podría trabar la ventana.
- **JSON reindentado** (agregado después): si el cuerpo parsea como JSON válido (`JSON.parse` en un `try/catch`), se reemplaza por `JSON.stringify(parsed, null, 2)` antes de imprimirlo — la mayoría de las APIs devuelven JSON minificado en una sola línea, imposible de leer en el scrollback. Si no es JSON (HTML, texto plano, etc.) se muestra crudo, sin tocar.

## Cambio en `DesktopManager` (2026-07-31)

`crearIconoEnCarpeta(iconoCss, tipo, name, carpeta, edit = true)` ganó un 5º parámetro opcional: antes llamaba incondicionalmente a `_editarTextoIcono()`, que pone el label en modo edición inline y le roba el foco a quien haya llamado — `mkdir` necesita crear la carpeta sin eso. Retrocompatible (el único llamador previo, el menú "Nuevo" de `Folder`, no pasa el parámetro). También ahora devuelve la instancia creada (antes `undefined`). Se agregó `DesktopManager.buscarEspacioVacio()`, wrapper público sobre `_grid.buscarVacio()` (privado), para que `mkdir` en la raíz del escritorio pueda encontrar un espacio libre sin tocar `_grid` directamente.

## Estilos

`styles/apps/kmd.css` (nuevo) + `@import` agregado a `main.css`. Sin `font-family` propia (se hereda de `body`, ver [[Reglas]]), sin `text-shadow`, solo `var(--primary-color)`/`var(--primary-background)`.
