---
tags:
  - reglas
---

# Reglas del proyecto KneOS

Convenciones y decisiones de estilo/arquitectura acordadas con el usuario para el desarrollo de KneOS. Ver también [[KneOS Portfolio]].

## Nomenclatura

- El código existente de KneOS usa nombres en español (`agregar`, `eliminar`, `iniciar`, `barraId`, etc.) y comentarios en español. **No renombrar** estos identificadores.
- Todo **código nuevo** (funciones, métodos, variables) debe nombrarse en **inglés**, aunque el resto del archivo esté en español.
- Los comentarios siguen la convención ya existente en el archivo, salvo que se indique lo contrario.

## CSS y estilos

- No redeclarar una propiedad CSS en un componente si ya está definida en un ancestro (`body`, `:root`) y se heredaría de forma natural (ej. `font-family`). Si se necesita un valor distinto al heredado, **preguntar antes** de sobreescribirlo.
- **Nunca usar `text-shadow`** en ninguna hoja de estilos de KneOS, sin excepciones (aunque el look CRT retro podría sugerirlo).
- La UI **core/sistema** (ventanas, menús contextuales, folders, taskbar) es estrictamente **monocromática**, basada en `var(--primary-color)`/`var(--primary-glow)`/`var(--primary-dim)` y `var(--primary-background)`. No introducir otros colores (ej. rojo para errores o advertencias) directo en CSS/hex — usar siempre esas variables. Las apps individuales (Kfruit, Doom, etc.) sí pueden tener su propia paleta.
- Si se necesitara señalar un estado especial (error, advertencia) en la UI core, debe expresarse con variaciones del color existente (opacidad, brillo) — nunca con un color nuevo — y solo tras confirmarlo con el usuario.
- **(2026-08-19)** El color ya no es fijo: [[Config]] deja elegir `--primary-color`/`--primary-glow`/`--primary-dim` entre 10 opciones (verde de fábrica + 9 más, ver [[Frontend Model Services Utils#Model|themeColors.js]]), persistido por sesión. "Monocromática" sigue queriendo decir *un solo* color a la vez en todo el chrome del sistema — no que ese color tenga que ser verde. La única excepción admitida a "nunca hex/color literal fuera de las variables" es la propia grilla de swatches de Config: cada muestra necesita mostrar el color real de esa opción para poder elegirla; el resto de esa misma ventana (bordes, texto, foco) sigue usando `--primary-*` como cualquier otra.

## Tooltips y feedback de UI

- **Nunca usar el atributo `title`** de HTML (tooltips nativos del navegador) en ningún elemento de KneOS. No encaja con la estética CRT retro. Si se necesita dar una pista visual, usar una afordancia visual (badge, ícono, label) en vez de un tooltip.
- **Nunca usar `window.confirm()`/`alert()`/`prompt()`** nativos — mismo motivo, rompen la estética. Para confirmar una acción destructiva (ej. "Vaciar papelera", ver [[RecycleBin]]), usar un `ContextMenu` más con las opciones (ej. "Sí, vaciar" / "Cancelar") en vez de un diálogo del navegador — mismo idioma visual que ya usa el resto del sistema para cualquier decisión del usuario (2026-07-31).
- Cuando una acción del usuario falla en resolverse (ej. una ruta escrita en la barra de direcciones del Folder no existe), **no mostrar ningún indicador de error** — nada de mensajes, bordes rojos, animaciones de shake, etc. Simplemente dejar la UI/input tal cual, sin cambios, para que el usuario pueda corregir y reintentar.

## Documentación

- Cada vez que se actualice código de KneOS, **también debe actualizarse la nota correspondiente en este vault de Obsidian** para mantener la documentación sincronizada con el código.

## Contexto general

- KneOS es una interfaz de escritorio falsa estilo Windows 95 (taskbar, íconos arrastrables, ventanas, apps como Kfruit/Doom/editor de texto) embebida en el portfolio de Manuel. Tema visual: terminal CRT (`--primary-color: #00ff41` de fábrica, elegible desde 2026-08-19 vía [[Config]] — ver más arriba, animaciones de scanline/flicker en `desktop.css`, fuente pixel `W95FA` definida una vez en `body` en `base.css`).
- La app raíz (`server.js`, Express + Prisma/Postgres) es una capa más pesada y separada; probar localmente requiere variables de entorno de DB y setup de sesión (`obtenerPcId`), por lo que los cambios solo de UI bajo `public/KneOS` no siempre se pueden probar en navegador sin esa configuración.
- Nuevos módulos core siguen el patrón existente: clase ES6, `export default`, constructor recibe un id de DOM, un getter re-obtiene el elemento por id en vez de cachearlo. Nuevas clases core se registran en `js/KNEOS.js`.
