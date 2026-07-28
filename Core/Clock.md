---
tags:
  - portfolio/kneos
  - frontend
  - core
---

# Clock

⬅️ Volver a [[Frontend Core]]

`core/Clock.js` — reloj digital de la taskbar (formato `es-AR`), autoalineado al segundo real (evita drift de `setInterval` recalculando el próximo timeout), resincroniza al volver de background (`visibilitychange`), y agrega un mini-calendario desplegable al hacer click. Autocontenido: no interactúa con ninguna otra clase del sistema (solo DOM e `Intl`).

> [!note] Corrección sobre `ARCHITECTURE.md`
> El `ARCHITECTURE.md` del repo describe a `Clock` como "stub sin implementar" — desactualizado. Está completamente implementado (reloj + mini-calendario navegable).

## Constructor(barraId="barraDeTarea")

Crea formateadores `Intl.DateTimeFormat` para hora (`HH:mm:ss` 24h) y fecha (`dd/mm/yyyy` numérico). Inicializa referencias a elementos DOM en `null`.

## Métodos públicos

- **`iniciar()`**: crea `div.relojBarraTarea` con `div.clockTime` + `div.clockDate`; click → `_toggleCalendar()`; registra listeners globales (`visibilitychange`, `click`, `contextmenu` para cerrar el calendario); llama `_tick()`.
- **`detener()`**: limpia timeout, remueve listeners globales y el elemento del DOM.

## Métodos privados

- **`_tick()`**: actualiza hora/fecha; reprograma `setTimeout` para el milisegundo exacto en que empieza el próximo segundo.
- **`_toggleCalendar()` / `_openCalendar()` / `_closeCalendar()`**: abren/cierran `div.taskbarCalendar`.
- **`_renderCalendar()`**: reconstruye header + grid del mes actual (`_viewDate`).
- **`_buildCalendarHeader()`**: navegación `‹ Mes Año ›`.
- **`_buildCalendarGrid()`**: grilla de días L-D + celdas de relleno + celdas de día (marca el día de hoy).

Instanciado una vez en `KNEOS.js`.
