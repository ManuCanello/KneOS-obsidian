---
tags:
  - portfolio/kneos
  - apps
---

# Apps

⬅️ Volver a [[KneOS Portfolio]]

Las 6 apps de escritorio de KneOS (`public/KneOS/js/apps/`). Todas extienden [[File]] y sobreescriben `_crearContenido()`; su tipo/extensión se resuelve vía `iconSrc.js` (ver [[Frontend Model Services Utils#Model]]).

| App | Extensión | Qué hace | Librería externa |
|---|---|---|---|
| [[TxtFile]] | `txt` | Editor de texto enriquecido (negrita/cursiva/subrayado, guardado manual o Ctrl+G) | ninguna (`contentEditable` nativo) |
| [[Folder]] | `fld` | Explorador de archivos estilo Windows (sidebar, breadcrumb, buscador, vistas, orden, drag&drop) | ninguna |
| [[KneAI]] | `ai` | Chat con IA estilo ChatGPT, historial de conversaciones, auto-titulado | ninguna (delega el LLM al backend, ver [[Módulo Groq]]) |
| [[Doom]] | `exe` | Emulador de DOOM vía DOSBox compilado a WASM | **js-dos** |
| [[Kmd]] | `kmd` | Terminal — stub sin implementar | ninguna |
| [[Kfruit]] | `kfruit` | Juego de fusión de frutas estilo Suika Game, con física real | **planck** (Box2D portado a JS) |

[[DesktopGrid y DesktopFolder|DesktopFolder]] (el "Escritorio" navegable) extiende `Folder`, pero se documenta junto al resto del core por ser infraestructura, no una app de usuario.
