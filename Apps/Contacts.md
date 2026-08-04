---
tags:
  - portfolio/kneos
  - apps
---

# Contacts

⬅️ Volver a [[Apps]]

`public/KneOS/js/apps/Contacts.js` — extiende [[File]]. Extensión `"contacts"`, ícono `sources/appIcon/file.svg` (genérico — no tiene ícono propio todavía), `src = null`.

> [!abstract] Qué hace
> Ventana chica y fija (sobre [[Window y Taskbar#`ViewWindow` (`core/ViewWindow.js`, 2026-07-29, extiende `Window`)|ViewWindow]], 300×280, sin resize/minimizar/maximizar ni entrada en la taskbar — mismo patrón que `apps/Calculator.js`, sin nota propia en el vault todavía) con una lista de 4 links de contacto (Instagram, LinkedIn, X, Email). Cada uno es un `<a target="_blank" rel="noopener noreferrer">` real — abre en una pestaña nueva del navegador, no dentro de KneOS, porque el destino siempre es externo al sistema falso.

## Constructor(name)

`super(name, "contacts", null, "sources/appIcon/file.svg", FileType.OTHER, 0)`, igual que [[Maxwell]] se pisa `this.window` justo después con la `ViewWindow` chica en vez de la `Window` completa que arma `File` por defecto.

## `_crearContenido()` / `_crearLink(label, value, href)`

Itera un array constante `CONTACT_LINKS` (`{label, value, href}`) y arma un `<a>` por cada uno, con dos `<span>` internos (label grande + value chico y atenuado, ver `contacts.css`).

> [!warning] Datos placeholder (2026-08-04)
> `CONTACT_LINKS` tiene 4 entradas con datos **falsos** (`usuario_falso`, `correo@ejemplo.com`) a pedido explícito del usuario — los reemplaza él mismo más adelante con sus redes/email reales, directo en el array al inicio de `Contacts.js`. El ícono también es el genérico `file.svg` a propósito, mismo motivo.

## Registro

Como [[Maxwell]]/[[RecycleBin]]/Calculator, es una app "fija": entrada en `iconSrc.js` (`contacts` → `Contacts`), en `defaultFiles.js` (`espacio3`, nombre "Contactos") y en `filesUndeletable.js` — no aparece en los menús "Nuevo" ni se puede borrar. Al igual que el resto de `defaultFiles`, solo aparece automáticamente en un escritorio **nuevo** (`IconServices.getIcons()` con cero filas); no aparece retroactivamente en escritorios ya persistidos.

## Persistencia

Ninguna propia — no interactúa con ningún módulo de [[Frontend Model Services Utils]] más allá de lo que ya hace `File`/`DesktopManager` para cualquier ícono.
