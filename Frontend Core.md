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

Script de entrada (`type="module"`, top-level `await`), cargado desde `public/KneOS/index.html`.

1. Importa `Groq`, `DesktopManager`, `Clock`, `TaskbarContextMenu` y `obtenerPcId` (de `services/session.js`).
2. Inicializa globales:
   - `window.archivosAbiertos = new Map()` — registro de toda instancia de archivo/carpeta abierta, indexado por `id` de ícono.
   - `window.groq = new Groq()` — cliente único del asistente IA.
3. `window.pcId = await obtenerPcId()` — bloquea el arranque hasta resolver la sesión ([[Módulo Session]]).
4. Instancia `DesktopManager`, `Clock`, `TaskbarContextMenu`; llama `desktop.iniciar()` y `reloj.iniciar()`.

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
| `TaskBarManager` | Botones de la taskbar, agrupados por extensión | [[Window y Taskbar]] |
| `File` | Clase base de todo ítem del escritorio | [[File]] |
| `Clock` | Reloj + mini-calendario de la taskbar | [[Clock]] |
| `dragGhost.js` | Ghost transparente + cableado de `dragstart` | [[Drag and Drop y Selección Múltiple]] |
| `seleccionMultiple.js` | Selección múltiple (ctrl+click) y borrado con `Delete` | [[Drag and Drop y Selección Múltiple]] |

## Relación entre las piezas nuevas (Kfruit-era)

> [!info] Cómo encajan `ContextMenu`, `DesktopFolder`, `dragGhost` y `seleccionMultiple`
> - **`ContextMenu`** es el motor genérico (abrir/cerrar/submenú/ítem/separador). Tanto `ContextMenuManager`, `TaskbarContextMenu`, `TaskBarManager` como [[Folder]] instancian su propia `ContextMenu` — no hay una única instancia compartida.
> - **`DesktopFolder`** resuelve "ver el escritorio como una carpeta navegable" reutilizando casi toda la lógica de `Folder`, pero renderizando **clones visuales** en vez de los íconos reales (el contenido real vive en los espacios de `DesktopGrid`).
> - **`dragGhost.js`** + **`seleccionMultiple.js`** implementan "arrastrar varios íconos a la vez": `seleccionMultiple` gestiona qué está seleccionado y expone `idsParaArrastrar`; `dragGhost.bindIconDragStart` es el punto único donde cualquier ícono se conecta a ese estado para decidir qué ids van en el `dataTransfer`.
