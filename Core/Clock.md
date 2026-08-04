---
tags:
  - portfolio/kneos
  - frontend
  - core
---

# Clock

⬅️ Volver a [[Frontend Core]]

`core/Clock.js` — reloj digital de la taskbar (formato `es-AR`), autoalineado al segundo real (evita drift de `setInterval` recalculando el próximo timeout), resincroniza al volver de background (`visibilitychange`), y agrega un mini-calendario desplegable al hacer click.

> [!note] Corrección sobre `ARCHITECTURE.md`
> El `ARCHITECTURE.md` del repo describe a `Clock` como "stub sin implementar" — desactualizado. Está completamente implementado (reloj + mini-calendario navegable).

> [!bug] Calendario que no se cerraba al abrir una ventana (2026-07-29)
> El calendario se cierra con un listener de `click`/`contextmenu` en `document` (`_onDocumentClick`), pero el click en un ícono que abre/enfoca una ventana (`DesktopManager._crearContenedorIcono`) hace `stopPropagation()` — nunca le llegaba al listener y el calendario quedaba abierto. Mismo patrón de bug que ya existía con `ContextMenu` (ver [[Menús Contextuales]]). Solución: `Clock` ahora mantiene un registro **a nivel de módulo** `_calendariosAbiertos` (Set, mismo patrón que `ContextMenu._menusAbiertos`) y expone `static closeAll()`; `Window.traerAlFrente()` (ver [[Window y Taskbar]]) la llama junto con `ContextMenu.closeAll()` cada vez que se abre/enfoca una ventana. Esto rompe la independencia total que tenía antes: `Window.js` ahora importa `Clock.js`.

## Constructor(barraId="barraDeTarea")

Crea formateadores `Intl.DateTimeFormat` para hora (`HH:mm:ss` 24h) y fecha (`dd/mm/yyyy` numérico). Inicializa referencias a elementos DOM en `null`.

## Métodos públicos

- **`iniciar()`**: crea `div.relojBarraTarea` con `div.clockIcon` (ver ícono animado abajo) + `div.clockTextWrapper` (`div.clockTime` + `div.clockDate`); click → `_toggleCalendar()`; registra listeners globales (`visibilitychange`, `click`, `contextmenu` para cerrar el calendario); llama `_tick()`.
- **`detener()`**: limpia timeout, remueve listeners globales y el elemento del DOM.

## Métodos privados

- **`_tick()`**: actualiza hora/fecha; llama `_updateHands(ahora)`; reprograma `setTimeout` para el milisegundo exacto en que empieza el próximo segundo.
- **`_toggleCalendar()` / `_openCalendar()` / `_closeCalendar()`**: abren/cierran `div.taskbarCalendar`.
- **`_renderCalendar()`**: reconstruye header + grid del mes actual (`_viewDate`).
- **`_buildCalendarHeader()`**: navegación `‹ Mes Año ›`.
- **`_buildCalendarGrid()`**: grilla de días L-D + celdas de relleno + celdas de día (marca el día de hoy).

> [!info] Ícono con manijas animadas (2026-07-29)
> `_loadIcon()` (async, llamado desde `iniciar()`) hace `fetch("sources/core/clock.svg")` y lo inyecta inline (`innerHTML`) dentro de `div.clockIcon` — a diferencia del resto de los íconos del sistema, que se pintan como `background`/`mask` (ver [[Frontend Model Services Utils]]), este necesita ser SVG real en el DOM para poder mover sus manijas por separado. El SVG tiene los paths `#hour-hand` y `#minute-hand` (más `#clock-center`, estático) con `fill="currentColor"`, así hereda el verde de `.clockIcon`.
> - **`_updateHands(ahora)`**: calcula `hourAngle = (horas%12 + minutos/60) * 30` y `minuteAngle = (minutos + segundos/60) * 6`, y los aplica como atributo SVG `transform="rotate(ángulo, 12, 12)"` (rotación nativa alrededor del centro del `viewBox 0 0 24 24`, no CSS `transform-origin`, para no depender de soporte de `transform-box` en SVG). El horario descansa apuntando a las 12 (arriba) y el minutero a las 3 (derecha), así que el minutero resta 90° (`minuteAngle - 90`) para compensar ese offset de reposo.
> - Si el `fetch` falla, `_loadIcon()` lo traga en un `catch` vacío: el reloj digital sigue funcionando igual, solo sin ícono.
> - `.clockIcon` mide 42px (mismo tamaño que `.iconoBarraTareaImg`, el resto de los íconos de la taskbar).

Instanciado una vez en `KNEOS.js`.
