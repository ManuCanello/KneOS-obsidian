---
tags:
  - portfolio/kneos
  - apps
---

# Apps

⬅️ Volver a [[KneOS Portfolio]]

Las 10 apps de escritorio de KneOS (`public/KneOS/js/apps/`). Todas extienden [[File]] y sobreescriben `_crearContenido()`; su tipo/extensión se resuelve vía `iconSrc.js` (ver [[Frontend Model Services Utils#Model]]).

| App | Extensión | Qué hace | Librería externa |
|---|---|---|---|
| [[TxtFile]] | `txt` | Editor de texto enriquecido (negrita/cursiva/subrayado, guardado manual o Ctrl+G) | ninguna (`contentEditable` nativo) |
| [[Folder]] | `fld` | Explorador de archivos estilo Windows (sidebar, breadcrumb, buscador, vistas, orden, drag&drop) | ninguna |
| [[KneAI]] | `ai` | Chat con IA estilo ChatGPT, historial de conversaciones, auto-titulado | ninguna (delega el LLM al backend, ver [[Módulo Groq]]) |
| [[Doom]] | `exe` | Emulador de DOOM vía DOSBox compilado a WASM | **js-dos** |
| [[Kmd]] | `kmd` | Terminal estilo CMD sobre el sistema de archivos real (`dir`/`cd`/`mkdir`/`touch`/`rmdir`/`move`/`ren`/`del`/`type`/`echo`/`tree`/`kneai`/`curl`/etc.) | ninguna |
| [[Kfruit]] | `kfruit` | Juego de fusión de frutas estilo Suika Game, con física real | **planck** (Box2D portado a JS) |
| [[Maxwell]] | `maxwell` | Visor 3D: carga un `.glb` y lo muestra girando sobre su eje | **Three.js** (import map propio, ver nota en [[Escena 3D]]) |
| [[RecycleBin]] | `recyclebin` | Papelera de reciclaje: lista lo mandado a la papelera (soft delete), con Restaurar/Eliminar definitivamente/Vaciar papelera | ninguna |
| Calculator (`apps/Calculator.js`, sin nota propia todavía) | `calc` | Calculadora simple, sin precedencia de operadores (evalúa de izquierda a derecha) | ninguna |
| [[Contacts]] | `contacts` | Links directos a redes/email (Instagram, LinkedIn, X, Email), cada uno abre en pestaña nueva del navegador | ninguna |

[[DesktopGrid y DesktopFolder|DesktopFolder]] (el "Escritorio" navegable) extiende `Folder`, pero se documenta junto al resto del core por ser infraestructura, no una app de usuario.

> [!info] Únicas apps con ícono de escritorio por defecto (`defaultFiles.js`)
> No las 10 aparecen automáticamente en un escritorio nuevo: `defaultFiles.js` solo sembraba Doom/Terminal/Kfruit/Kne, después Maxwell (2026-07-30), RecycleBin (2026-07-31), Calculator y ahora también Contacts (2026-08-04) — `TxtFile`/`Folder` no están ahí porque se crean a demanda desde el menú "Nuevo" (ver [[Folder#Funciones clave|Folder._abrirSubMenuCrear]] y el equivalente en `ContextMenuManager`). Igual que las otras apps "fijas" (Doom/Kmd/Kfruit/KneAI/Maxwell/RecycleBin/Calculator), Contacts **no** está en esos menús "Nuevo" y además está en `filesUndeletable` (no tendría sentido poder eliminarla ni recrearla). `defaultFiles` solo se usa si `IconServices.getIcons()` devuelve **cero** filas (escritorio nunca inicializado) — en un escritorio con datos ya persistidos, agregar una entrada nueva ahí no hace aparecer el ícono retroactivamente.
