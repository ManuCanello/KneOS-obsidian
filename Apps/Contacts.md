---
tags:
  - portfolio/kneos
  - apps
---

# Contacts

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/Contacts.js` — extiende [[File]]. Extensión `"contacts"`, ícono `sources/appIcon/file.svg` (genérico — no tiene ícono propio todavía), `src = null`.

> [!abstract] Qué hace
> Ventana chica y fija (sobre [[Window y Taskbar#`ViewWindow` (`core/ViewWindow.js`, 2026-07-29, extiende `Window`)|ViewWindow]], 800×1020, sin resize/minimizar/maximizar ni entrada en la taskbar) con una **ficha estilo ID card retro** (look CRT/HUD), no una lista de links como en la versión original. Layout en columna (`.contactsApp`), de arriba a abajo:
> - **Header** (`.contactsHeader`, 5% alto): título "ケンエ KNE" a la izquierda, "SIGNAL LOW" a la derecha.
> - **Acceso** (`.contactsAccess`, 15% alto, fondo verde tenue `#00cc332e`): fila superior "Full Stack Developer" / "ACCESS", fila inferior nombre grande "CANELLO MANUEL" / id enmascarado (`***********`).
> - **Foto** (`.contactsPhotoRow`, alto automático — ver más abajo): a la izquierda un marco (`.contactsPhotoFrame`, 40% ancho) con el placeholder de foto `#img` (`.contactsPhotoSquare`) forzado a ser **cuadrado** vía `width:100%; height:0; padding-bottom:100%` (no `aspect-ratio`, para no depender de la altura del contenedor); a la derecha un panel lateral vacío (`.contactsPhotoSide`, 60% ancho).
> - **Autorización** (`.contactsAuth`): bloque con `box-shadow` inset a los lados (viñeta suave en `var(--primary-color)` vía `color-mix()`, 25% de mezcla).
> - **Footer** (`.contactsFooter`, 10% alto).

## Sistema de alturas: la fila de la foto manda

> [!note] `.contactsPhotoRow` no tiene alto fijo
> A diferencia del resto de las filas (que usan `%` fijo del alto total), `.contactsPhotoRow` no declara `height`: se dimensiona sola al contenido, que es el cuadrado `#img` (su alto sale de su propio ancho vía `padding-bottom:100%`, no del contenedor). `.contactsAuth` compensa con `flex: 1; min-height: 0;` — absorbe automáticamente el espacio vertical que quede libre, así no hay que recalcular porcentajes a mano cada vez que cambia el tamaño de la foto.

## Esquinas punteadas (`.contactsPanel` + `.contactsCornerDot`)

> [!tip] Decoración tipo HUD
> Cualquier panel con clase `.contactsPanel` (`position: relative`) puede recibir puntitos cuadrados de 6×6px en las esquinas vía el helper `_crearEsquinas(esquinas)` (`Contacts.js`), que devuelve `<span>` con clases `contactsCornerDot contactsCornerDot--{tl|tr|bl|br}` (color `currentColor`, heredan el color del borde). Solo se agregan esquinas donde el borde realmente "cierra" — ej. `filaAcceso` no tiene borde superior, así que solo lleva `bl`/`br`; `#img` tiene los 4 lados con borde, así que lleva las 4 esquinas.

## Todo en clases, no inline

> [!info] Refactor (2026-08-06)
> El layout completo pasó de `style.cssText` inline a clases definidas en `contacts.css` (`.contactsHeader`, `.contactsAccess`, `.contactsAccessRow`, `.contactsAccessLabel`(`--dim`), `.contactsAccessName`, `.contactsAccessIdWrap`, `.contactsAccessId`, `.contactsPhotoRow/Frame/Square/Side`, `.contactsAuth`, `.contactsFooter`). El viejo TODO que decía "estilos inline temporales" ya no aplica.

## Constructor(name)

`super(name, "contacts", null, "sources/appIcon/file.svg", FileType.OTHER, 0)`, igual que [[Maxwell]] se pisa `this.window` justo después con la `ViewWindow` chica en vez de la `Window` completa que arma `File` por defecto.

## `_crearContenido()` / `_crearEsquinas(esquinas)`

Arma la ficha completa a mano (`createElement` + `className`, sin loop de datos) y decora cada panel con `_crearEsquinas(["tl","tr","bl","br"])` (o el subset de esquinas que corresponda) para los puntos de las esquinas.

> [!warning] Código muerto: `CONTACT_LINKS` / `_crearLink` (2026-08-06)
> La constante `CONTACT_LINKS` (4 entradas con datos **falsos**: `usuario_falso`, `correo@ejemplo.com`) y el método `_crearLink(label, value, href)` siguen en el archivo pero **ya no se usan** — `_crearContenido()` no los llama más desde que la app pasó de "lista de links" a "ficha ID card". Quedan como base para si se vuelve a agregar una sección de contacto real, pero hoy son dead code. El ícono genérico `file.svg` y el nombre "CANELLO MANUEL"/"Full Stack Developer" hardcodeados también son placeholder a reemplazar cuando se defina el contenido final.

## Registro

Como [[Maxwell]]/[[RecycleBin]]/Calculator, es una app "fija": entrada en `iconSrc.js` (`contacts` → `Contacts`), en `defaultFiles.js` (`espacio3`, nombre "Contactos") y en `filesUndeletable.js` — no aparece en los menús "Nuevo" ni se puede borrar. Al igual que el resto de `defaultFiles`, solo aparece automáticamente en un escritorio **nuevo** (`IconServices.getIcons()` con cero filas); no aparece retroactivamente en escritorios ya persistidos.

## Persistencia

Ninguna propia — no interactúa con ningún módulo de [[Frontend Model Services Utils]] más allá de lo que ya hace `File`/`DesktopManager` para cualquier ícono.
