---
tags:
  - portfolio/kneos
  - frontend
  - core
---

# Frontend Core

⬅️ Volver a [[KneOS Portfolio]] · Ver [[Arquitectura]]

Infraestructura del "escritorio" simulado dentro de `public/KneOS/js/core/`. Todo modelado con clases ES6.

## Bootstrap — `KNEOS.js`

Script de entrada (`type="module"`, top-level `await`), cargado desde `public/KneOS/index.html` — desde 2026-08-13 vía el bundle local `js/dist/kneos.bundle.js` (ver [[Deuda Técnica#Bundling local de three.js/planck/interactjs (2026-08-13)]]), no directamente.

0. *(2026-08-13)* `import interact from "interactjs"` + `window.interact = interact` — antes `interact` llegaba como global desde un `<script>` de jsdelivr en `public/KneOS/index.html`; ahora que interactjs está bundleado con esbuild, `KNEOS.js` lo expone a mano como global para no tocar cada call-site que sigue llamando `interact(...)` sin import (`Window.js`, etc.).
1. Importa `Groq`, `DesktopManager`, `Clock`, `TaskbarContextMenu`, `startSession` (de `services/session.js`), `FolderGroupByServices`, `FolderViewsServices`.
2. Inicializa globales:
   - `window.archivosAbiertos = new Map()` — registro de toda instancia de archivo/carpeta abierta, indexado por `id` de ícono.
   - `window.groq = new Groq()` — cliente único del asistente IA.
3. `await startSession()` — bloquea el arranque hasta resolver la sesión ([[Módulo Session]]); carga `window.folderGroupByOptions`/`window.folderViewsOptions`.
4. Instancia `DesktopManager`, `Clock`, `TaskbarContextMenu`; llama `desktop.iniciar()` y `reloj.iniciar()`.
5. *(2026-08-13)* Saca (fade-out + remove) el `#loadingScreen` del `index.html` — ver [[Escena 3D#Placeholder de carga con `<img>`, no texto (2026-08-13)]].

## Clases del core

| Clase / módulo | Rol | Nota |
|---|---|---|
| `DesktopManager` | Orquesta el escritorio: crea/edita/borra íconos raíz, aplica metadata persistida | [[DesktopManager]] |
| `DesktopGrid` | Gestiona los "espacios" (slots) del escritorio y su drag&drop | [[DesktopGrid y DesktopFolder]] |
| `DesktopFolder` | Representa el "Escritorio" como carpeta navegable (extiende `Folder`) | [[DesktopGrid y DesktopFolder]] |
| `ContextMenu` | Motor genérico de menús contextuales (agnóstico de dominio) | [[Menús Contextuales]] |
| `ContextMenuManager` | Menú click-derecho del escritorio/íconos | [[Menús Contextuales]] |
| `TaskbarContextMenu` | Menú click-derecho de la taskbar (alineación) | [[Menús Contextuales]] |
| `Window` | Ventana arrastrable/redimensionable (interact.js) | [[Window y Taskbar]] |
| `ViewWindow` | Ventana "solo de vista" (extiende `Window`): tamaño fijo, solo Cerrar, sin taskbar (2026-07-29) | [[Window y Taskbar]] |
| `TaskBarManager` | Botones de la taskbar, agrupados por extensión + lupa de búsqueda | [[Window y Taskbar]] |
| `TaskbarSearch` | Ventana "Buscar" (sobre `ViewWindow`) abierta por la lupa — busca en todos los archivos del sistema (2026-07-30) | [[Window y Taskbar#`TaskbarSearch` (`core/TaskbarSearch.js`, 2026-07-30, nuevo)\|Window y Taskbar]] |
| `Home` | Ventana "Inicio" (sobre `ViewWindow`) abierta por un segundo botón de la taskbar — buscador que entrega a `TaskbarSearch`, columnas "Abierto recientemente"/"Todo" (por `FileType`), footer usuario+prender (2026-07-31) | [[Window y Taskbar#`Home` (`core/Home.js`, 2026-07-31, nuevo)\|Window y Taskbar]] |
| `File` | Clase base de todo ítem del escritorio | [[File]] |
| `Clock` | Reloj + mini-calendario de la taskbar | [[Clock]] |
| `dragGhost.js` | Ghost transparente + cableado de `dragstart` | [[Drag and Drop y Selección Múltiple]] |
| `multiSelect.js` | Selección múltiple (ctrl+click) y borrado con `Delete` (ex `seleccionMultiple.js`, renombrado y traducido al inglés 2026-07-29) | [[Drag and Drop y Selección Múltiple]] |
| `FileProperties` | Ventana "Propiedades" de un archivo (nombre editable, tipo, ubicación/tamaño, creado/modificado), sobre `ViewWindow` — ex `PropertiesApp`, reemplaza a `PropertiesDialog.js` (2026-07-29) | [[Menús Contextuales]] |

## Estilos globales — `styles/base.css`

Cargado primero por `main.css` (antes de todo lo modular en `core/`/`apps/`). Define `:root` (`--primary-color`, `--primary-glow`, `--primary-dim`, `--primary-background`, `--scanline-opacity`), `box-sizing`/`image-rendering` en `*`, la fuente `W95FA` en `body`, y estilos de `[contenteditable]`.

- **Scrollbars estilo mobile (2026-07-30)**: `*` con `scrollbar-width: thin` + `scrollbar-color` (Firefox) y `*::-webkit-scrollbar*` (Chromium/WebKit) — finas (6px), thumb redondeado en `--primary-dim` (se aclara a `--primary-color` en hover), track y corner transparentes. Es una regla única y global (no se duplica por componente) que cubre todo contenedor con `overflow: auto` en el proyecto (editor de texto, listado de `Folder`/`Kfruit`, chat de `KneAI`, panel de `FileProperties`, etc.).
- **`::placeholder` global (2026-07-30)**: `color: var(--primary-color); opacity: 0.5` para todo `input`/`textarea` del proyecto. Antes vivía duplicado solo en `.folderBuscador::placeholder` ([[Folder]]); se centralizó acá y se sacó esa regla específica al volverse redundante.

> [!info] Fuente `W95FA` reubicada a `styles/fonts/` (2026-07-31)
> El `@font-face` de `base.css` apuntaba a `../W95FA.woff2` (es decir `public/KneOS/W95FA.woff2`, suelto en la raíz de KneOS) — el archivo real nunca había estado versionado (el 404 se notó recién al abrir la consola del navegador; hasta entonces caía en silencio al fallback `monospace` de `font-family: 'W95FA', monospace`). Al agregar el `.woff2` real se aprovechó para ordenarlo en `styles/fonts/W95FA.woff2`, y el `src` pasó a `url('fonts/W95FA.woff2')` (relativo a `base.css`, que vive en `styles/`).

## `sources/` — reorganizado por categoría (2026-07-31)

Antes todo (íconos de archivo/app, íconos de acciones de menú, controles de ventana, taskbar) vivía junto en un único `sources/icon/`. Se dividió en tres carpetas hermanas bajo `sources/`, sin la capa `icon/` intermedia (ya no hace falta, las carpetas hijas son las que categorizan):

- **`sources/appIcon/`** — íconos que representan un tipo de archivo/app (el ícono de escritorio y de la barra de título de cada uno): `desktop.svg`, `doom.png`, `folder.svg`, `kneAi.png`, `terminal.svg`, `txt.svg`, `file.svg` (genérico/Maxwell/vistas de `Folder`), `trash.svg` ([[RecycleBin]]), más `defaultIcon.png` (referenciado en `iconSrc.js` como fallback pero el archivo en sí nunca existió — 404 preexistente, no introducido por esta reorganización) y tres archivos legacy sin ninguna referencia en el código (`folder.png`, `kmd.png`, `txtIcon.png` — se movieron igual, no se borraron, por si convenía conservarlos).
- **`sources/accions/`** — íconos de ítems de menú/acciones puntuales: `more.svg` ("Nuevo"/nuevo chat), `see.svg` ("Ver"), `group-by.svg` ("Ordenar por"), `list.svg` (vista Lista), `save.svg` (Guardar de [[TxtFile]]), `menu.svg` (colapsar de [[KneAI]]), y desde 2026-07-31 los 6 del menú contextual de ícono (ver [[Menús Contextuales#`ContextMenuManager` (`core/ContextMenuManager.js`)|ContextMenuManager]]): `open.svg` (Abrir), `edit.svg` (Renombrar), `link.svg` (Copiar ruta de acceso), `fav.svg` (favoritos), `attach.svg` (anclar a la taskbar), `config.svg` (Propiedades del archivo).
- **`sources/core/`** — chrome de sistema, no ligado a ningún archivo/app puntual: `cancel.svg`/`remove.svg`/`expand.svg`/`dismiss.svg` (botones de ventana, ver [[Window y Taskbar]]), `clock.svg` ([[Clock]]), `search.svg` (lupa de la taskbar/`TaskbarSearch`).

Cada referencia (`sources/icon/X` en JS/CSS) se reescribió a `sources/<categoría>/X` — mismo nombre de archivo, solo cambió el directorio contenedor; ningún ícono cambió de contenido.

## Relación entre las piezas nuevas (Kfruit-era)

> [!info] Cómo encajan `ContextMenu`, `DesktopFolder`, `dragGhost` y `multiSelect`
> - **`ContextMenu`** es el motor genérico (abrir/cerrar/submenú/ítem/separador). Tanto `ContextMenuManager`, `TaskbarContextMenu`, `TaskBarManager` como [[Folder]] instancian su propia `ContextMenu` — no hay una única instancia compartida.
> - **`DesktopFolder`** resuelve "ver el escritorio como una carpeta navegable" reutilizando casi toda la lógica de `Folder`, pero renderizando **clones visuales** en vez de los íconos reales (el contenido real vive en los espacios de `DesktopGrid`).
> - **`dragGhost.js`** + **`multiSelect.js`** implementan "arrastrar varios íconos a la vez": `multiSelect` gestiona qué está seleccionado y expone `idsToDrag`; `dragGhost.bindIconDragStart` es el punto único donde cualquier ícono se conecta a ese estado para decidir qué ids van en el `dataTransfer`.
