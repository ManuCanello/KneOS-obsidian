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
- La UI **core/sistema** (ventanas, menús contextuales, folders, taskbar) es estrictamente **monocromática verde**, basada en `var(--primary-color)` y `var(--primary-background)`. No introducir otros colores (ej. rojo para errores o advertencias). Las apps individuales (Kfruit, Doom, etc.) sí pueden tener su propia paleta.
- Si se necesitara señalar un estado especial (error, advertencia) en la UI core, debe expresarse con variaciones del verde existente (opacidad, brillo) — nunca con un color nuevo — y solo tras confirmarlo con el usuario.

## Tooltips y feedback de UI

- **Nunca usar el atributo `title`** de HTML (tooltips nativos del navegador) en ningún elemento de KneOS. No encaja con la estética CRT retro. Si se necesita dar una pista visual, usar una afordancia visual (badge, ícono, label) en vez de un tooltip.
- Cuando una acción del usuario falla en resolverse (ej. una ruta escrita en la barra de direcciones del Folder no existe), **no mostrar ningún indicador de error** — nada de mensajes, bordes rojos, animaciones de shake, etc. Simplemente dejar la UI/input tal cual, sin cambios, para que el usuario pueda corregir y reintentar.

## Documentación

- Cada vez que se actualice código de KneOS, **también debe actualizarse la nota correspondiente en este vault de Obsidian** para mantener la documentación sincronizada con el código.

## Contexto general

- KneOS es una interfaz de escritorio falsa estilo Windows 95 (taskbar, íconos arrastrables, ventanas, apps como Kfruit/Doom/editor de texto) embebida en el portfolio de Manuel. Tema visual: terminal CRT verde (`--primary-color: #00ff41`, animaciones de scanline/flicker en `desktop.css`, fuente pixel `W95FA` definida una vez en `body` en `base.css`).
- La app raíz (`server.js`, Express + Prisma/Postgres) es una capa más pesada y separada; probar localmente requiere variables de entorno de DB y setup de sesión (`obtenerPcId`), por lo que los cambios solo de UI bajo `public/KneOS` no siempre se pueden probar en navegador sin esa configuración.
- Nuevos módulos core siguen el patrón existente: clase ES6, `export default`, constructor recibe un id de DOM, un getter re-obtiene el elemento por id en vez de cachearlo. Nuevas clases core se registran en `js/KNEOS.js`.
