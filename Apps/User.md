---
tags:
  - portfolio/kneos
  - apps
---

# User

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/User.js` (ex `Contacts.js`, renombrado 2026-08-07) — extiende [[File]]. Extensión `"contacts"` (sin renombrar — solo cambió el nombre de archivo/clase/clases CSS, no el identificador interno ni la entrada en `filesUndeletable.js`), ícono `sources/appIcon/user.svg` (2026-08-07, propio — antes usaba el genérico `file.svg`; el mismo cambio se hizo en `iconSrc.js`), `src = null`. En el escritorio aparece como **"Usuario"** (`defaultFiles.js`).

> [!abstract] Qué hace
> Ventana chica y fija (sobre [[Window y Taskbar#`ViewWindow` (`core/ViewWindow.js`, 2026-07-29, extiende `Window`)|ViewWindow]], 800×1020, sin resize/minimizar/maximizar ni entrada en la taskbar) con una **ficha estilo ID card retro** (look CRT/HUD). Layout en columna (`.userApp`), de arriba a abajo:
> - **Header** (`.userHeader`, 5% alto): título "ケンエ KNE" a la izquierda, "SIGNAL LOW" al medio, ícono de batería a la derecha (ver más abajo) — los tres con `justify-content: space-between`.
> - **Acceso** (`.userAccess`, 15% alto, fondo verde tenue `#00cc332e`): fila superior "Full Stack Developer" / "ACCESS", fila inferior nombre grande "CANELLO MANUEL" / id enmascarado (`***********`).
> - **Foto** (`.userPhotoRow`, alto automático — ver más abajo): a la izquierda un marco (`.userPhotoFrame`, 40% ancho) con la foto real `#img` (`.userPhotoSquare`) forzada a ser **cuadrada** vía `width:100%; height:0; padding-bottom:100%` (no `aspect-ratio`, para no depender de la altura del contenedor), con efecto holograma (ver más abajo); a la derecha un panel lateral (`.userPhotoSide`, 60% ancho, columna izquierda-alineada) con la ficha "REPLI¿HUMAN" + SEC CODE + datos de desarrollo (ver más abajo).
> - **Autorización** (`.userAuth`): centra (`display:flex; align-items:center; justify-content:center`) los **links reales a redes sociales** (ver más abajo), con `box-shadow` inset a los lados (viñeta suave en `var(--primary-color)` vía `color-mix()`, 25% de mezcla).
> - **Footer** (`.userFooter`, 10% alto, `justify-content: space-between`): "ケンエ" a cada extremo (glow, mismo patrón que el header) flanqueando un bloque centrado de dos líneas, "PROPERTY OF KNE SYSTEMS /" y "CYBER SYSTEMS UNIT" (`.userFooterText` columna centrada, `.userFooterLine` con `margin:0`).

## Refactor de nombres (2026-08-07): `contacts*` → `user*`, todo unificado

> [!info] Rename completo, sin duplicar reglas
> Todas las clases y `@keyframes` que antes llevaban el prefijo `contacts` (`.contactsApp`, `.contactsPanel`, `.contactsHeader`, `.contactsNeonText`, `contactsPowerOn`, etc.) pasaron a `user*` — coherente con el rename del archivo/clase. De paso se unificaron reglas que estaban duplicadas letra por letra:
> - **Glow de borde**: `.userApp` y `.userPanel` comparten una sola regla (`position: relative; box-shadow: 0 0 14px color-mix(...)`) en vez de que cada uno declare su propio `box-shadow` — y de paso `.userApp` ahora también lleva `border: 2px solid var(--primary-color)`, o sea el contenedor principal tiene el mismo glow de borde que cualquier panel interno.
> - **Glow neón**: `.userNeonText`, `.userLinkIcon`, `.userLinkLabel`, `.userFooterText` y `.userBatteryIcon` comparten una sola declaración de `filter: drop-shadow(...)` + `animation: userNeonFlicker 15s linear infinite` — antes cada uno repetía el mismo `filter`/`animation` por separado.

## Ícono de batería en el header (`_crearBateria()`, 2026-08-07)

> [!info] SVG inline armado a mano, no vía mask
> A diferencia de los demás íconos de la app (mask + `background-color: currentColor`), la batería (`sources/core/battery.svg`) se arma como SVG real (`document.createElementNS`) porque necesita **dos piezas independientes animadas por separado**: un `<path>` fijo con el marco completo (borde + terminal, sin las 3 barritas de carga originales del SVG) y un `<rect>` aparte para **solo la última barrita** (la más cercana al terminal) — las otras dos simplemente no se dibujan, como si ya se hubieran gastado. Esa única barrita tiene `.userBatteryBar { animation: userBatteryBlink 1.2s ease-in-out infinite; }` (opacity 1↔0.15) para leerse como "batería a punto de quedarse sin carga". El ícono entero (`.userBatteryIcon`) participa del glow neón compartido (ver arriba). Reemplaza al viejo `espacioHeader` (un `<p>` vacío que solo hacía de relleno para el `justify-content: space-between`).

## Efecto holograma sobre la foto (`.userHologram`, 2026-08-07)

> [!info] Capa aparte, no en `.userPhotoSquare` directo
> La foto (`sources/socialMedia/bitmap.png`, halftone verde — reemplazó el placeholder vacío) vive en un wrapper `.userHologram` (`position:absolute; inset:0`) dentro de `#img`, en vez de aplicarse directo sobre `.userPhotoSquare`. Motivo: el efecto necesita `overflow:hidden` para recortar el barrido, pero `.userPhotoSquare` también contiene los `<span>` de esquinas (`.userCornerDot`) que sobresalen a propósito por fuera del borde — si el `overflow:hidden` fuera del elemento que los contiene, se cortarían. Con la capa separada, `#img` mantiene sus esquinas visibles y solo `.userHologram` recorta su propio contenido.
> Tres piezas, todas dentro de la paleta monocromo verde (nada de cian/azul, que sería lo típico de un holograma real):
> - `::before` — scanlines reutilizando el keyframe `userScanlines`, con `mix-blend-mode: screen` (aclara en vez de teñir).
> - `::after` — una banda de luz que barre de arriba a abajo cada 4s (`userHologramSweep`), también con `screen`.
> - El wrapper mismo — parpadeo de opacidad **irregular** (`userHologramFlicker`, saltos tipo 45%→0.85, 50%→0.5, 52%→0.9...), a propósito no un fundido parejo: un holograma "tiembla", no se atenúa suave.

## Panel lateral de la foto (`.userPhotoSide`, 2026-08-07)

> [!info] Antes vacío, ahora con la ficha "REPLI¿HUMAN"
> `.userPhotoSide` pasó de estar vacío (solo esquinas) a columna izquierda-alineada (`flex-direction:column; align-items:flex-start; gap:1rem; font-size:32px`) con tres piezas, armadas por métodos dedicados en vez de mezclarse en `_crearContenido()`:
> - **`_crearFilaSuperiorLateral()`**: fila superior (`.userPhotoSideTop`, `flex-direction:row; justify-content:space-between; align-items:center; width:100%`) con la etiqueta "REPLI¿HUMAN" a la izquierda y un sello cuadrado vacío a la derecha (`.userPhotoSideSeal`, 2rem×2rem, `border:2px solid`, sin contenido — 2026-08-07).
> - **`_crearEtiquetaReplicante()`**: la línea "REPLI¿HUMAN", con el tramo "PLI" reemplazado por texto **fijo** (no animado) con marcas Unicode de combinación tipo tachado (`U+0335`/`U+0337`/`U+0338`) puestas directo en el `textContent` del span — `glitch.textContent = "P̸L̷I̵"`. "RE" y "¿HUMAN" quedan como texto plano a los lados. (Primera versión usaba un `@keyframes` de CSS ciclando el `content` de un `::before` por distintos glyphs — se descartó a pedido explícito: "que las letras glitcheadas sean fijas".)
> - **`_crearGrupoSecCode()`** / **`_crearGrupoDesarrollo()`**: cada uno un `<div class="userPhotoSideGroup">` (`display:flex; flex-direction:column; gap:0.5rem`) con los `<p class="userFooterLine">` de "SEC CODE"/"18752-D-2254" y "ソフトウェア"/"TEC. DESARROLLO SOFTWARE"/"開発者" respectivamente — reutiliza `.userFooterLine` (el `margin:0` del footer) en vez de una clase nueva.
> - Todas las líneas de texto (incluida la de REPLI¿HUMAN) pasan por `_aplicarNeon`/`.userNeonText` para el glow.

## Más delay entre animaciones (2026-08-07)

> [!info] De 4 variantes de delay a 6, y sin defaults compartidos
> Antes de este ajuste, ~14 elementos (los 4 links —ícono+label sin variante asignada—, `.userFooterText` y `.userBatteryIcon`, todos sin clase `--a/b/c/d`) titilaban en el instante exacto `0s`, exactamente en sync con el grupo `--a`. Se corrigió en dos frentes:
> - **`.userNeonText--a..d` → `--a..f`** (6 variantes en vez de 4, `user.css`), separadas 2.5s en vez de ~3.5-4s dentro del mismo ciclo de 15s de `userNeonFlicker` — y **todo** elemento con glow ahora recibe una variante explícita, nada queda en el delay por defecto (0s). `NEON_DELAYS` (array de 6 nombres de clase) en `User.js` se cicla por índice para los 4 links vía `_crearLink(..., delay)`.
> - **`.userCornerDot--tl/tr/bl/br`**: antes sin `animation-delay` (los 4 puntos de cada panel pulsaban a la vez); ahora 0s/0.6s/1.2s/1.8s por posición, así corren en secuencia alrededor de cada panel en vez de parpadear todos juntos.

## Sistema de alturas: la fila de la foto manda

> [!note] `.userPhotoRow` no tiene alto fijo
> A diferencia del resto de las filas (que usan `%` fijo del alto total), `.userPhotoRow` no declara `height`: se dimensiona sola al contenido, que es el cuadrado `#img` (su alto sale de su propio ancho vía `padding-bottom:100%`, no del contenedor). `.userAuth` compensa con `flex: 1; min-height: 0;` — absorbe automáticamente el espacio vertical que quede libre, así no hay que recalcular porcentajes a mano cada vez que cambia el tamaño de la foto.

## Esquinas punteadas (`.userPanel` + `.userCornerDot`)

> [!tip] Decoración tipo HUD
> Cualquier panel con clase `.userPanel` (`position: relative`) puede recibir puntitos cuadrados de 6×6px en las esquinas vía el helper `_crearEsquinas(esquinas)` (`User.js`), que devuelve `<span>` con clases `userCornerDot userCornerDot--{tl|tr|bl|br}` (color `currentColor`, heredan el color del borde). Solo se agregan esquinas donde el borde realmente "cierra" — ej. `filaAcceso` no tiene borde superior, así que solo lleva `bl`/`br`; `#img` tiene los 4 lados con borde, así que lleva las 4 esquinas.

## Todo en clases, no inline

> [!info] Refactor (2026-08-06)
> El layout completo pasó de `style.cssText` inline a clases definidas en `user.css` (ex `contacts.css`, renombrado junto con el archivo JS el 2026-08-07). Esto se mantuvo incluso cuando se pegó markup con estilos inline como referencia para pedir cambios (ej. el panel lateral de la foto) — siempre se tradujo a clases CSS nuevas o reutilizadas, nunca a `style="..."` literal.

## Constructor(name)

`super(name, "contacts", null, "sources/appIcon/user.svg", FileType.OTHER, 0)`, igual que [[Maxwell]] se pisa `this.window` justo después con la `ViewWindow` chica en vez de la `Window` completa que arma `File` por defecto.

## `_crearContenido()` / `_crearEsquinas(esquinas)`

Arma la ficha completa a mano (`createElement` + `className`, sin loop de datos salvo `CONTACT_LINKS`) y decora cada panel con `_crearEsquinas(["tl","tr","bl","br"])` (o el subset de esquinas que corresponda) para los puntos de las esquinas.

## `CONTACT_LINKS` / `_crearLink(label, value, href, icon)`

> [!info] Datos reales (2026-08-07)
> `CONTACT_LINKS` tiene los **4 links reales**: GitHub (`ManuCanello`), LinkedIn (`in/manuel-canello`), Instagram (`@manuelcanello`), X (`@canellomanuel`). Cada entrada trae un `icon` (ruta absoluta `/KneOS/sources/socialMedia/{github,linkedin,instagram,x}.svg` — no hay un módulo dedicado, son SVG estáticos servidos tal cual). `_crearLink` arma un `<a target="_blank" rel="noopener noreferrer">` con:
> - `.userLinkIcon`: no es un `<img>` — es un `<span>` con `background-color: currentColor` + `mask`/`-webkit-mask: var(--icono) center / contain no-repeat`, mismo patrón monocromo que `.btnVentana::before` en `core/window.css`. El SVG se pasa como custom property inline (`style.setProperty("--icono", ...)`) porque cada ícono es distinto; importante: el `url()` de una custom property se resuelve relativo a la **hoja de estilos que declara la regla `mask`** (`user.css`), no al JS que la setea — por eso las rutas son absolutas desde la raíz del sitio (`/KneOS/...`) y no relativas.
> - `.userSocialLinks` los arma en fila (`flex-wrap: wrap`) dentro de `.userAuth`, no en columna como en la versión original de lista vertical.
>
> El nombre "CANELLO MANUEL"/"Full Stack Developer" hardcodeados siguen siendo placeholder.

## Registro

Como [[Maxwell]]/[[RecycleBin]]/Calculator, es una app "fija": entrada en `iconSrc.js` (`contacts` → `User`, `css: url(sources/appIcon/user.svg)`), en `defaultFiles.js` (nombre "Usuario") y en `filesUndeletable.js` (sigue como `"contacts"`, no se tocó) — no aparece en los menús "Nuevo" ni se puede borrar. Al igual que el resto de `defaultFiles`, solo aparece automáticamente en un escritorio **nuevo** (`IconServices.getIcons()` con cero filas); no aparece retroactivamente en escritorios ya persistidos.

## Accesibilidad de movimiento

> [!note] `@media (prefers-reduced-motion: reduce)`
> Todas las animaciones de la app se desactivan ahí: `.userApp`/`.userApp::after` (power-on + glitch + scanlines), `.userNeonText`/`.userLinkLabel`/`.userLinkIcon`/`.userFooterText`/`.userBatteryIcon` (glow flicker), `.userBatteryBar` (parpadeo), `.userHologram` + sus dos pseudo-elementos (scanlines/barrido/temblor), `.userCornerDot` (pulso) y `.userLink:hover` (flicker de hover). El texto glitcheado de "REPLI¿HUMAN" no necesita entrar en esta lista porque ya es estático (no animado) desde el rediseño.

## Persistencia

Ninguna propia — no interactúa con ningún módulo de [[Frontend Model Services Utils]] más allá de lo que ya hace `File`/`DesktopManager` para cualquier ícono.
