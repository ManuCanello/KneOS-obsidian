---
tags:
  - portfolio/kneos
  - backend
---

# Módulo CarRace

⬅️ Volver a [[Backend]]

Log de carreras del minijuego de apuestas a autos ([[CarRace]]) — global (sin `pc_id`), crece con cada carrera corrida. Mismo criterio que [[Módulo FlipCoin]] (`flipcoin_results`): a diferencia de [[Módulo Hangman]]/[[Módulo Kdle]] (catálogo fijo `words`, solo lectura), este módulo tiene un endpoint de escritura.

> [!info] Renombrado a inglés (2026-08-13)
> Tabla/columnas/archivos de este módulo se renombraron de español a inglés el mismo día que se creó: `carreraauto_resultados`→`car_race_results`, `id_resultado`→`id_result`, `auto`→`car` (columna — el valor guardado sigue siendo el nombre en español del auto ganador, no se tradujo). `routes/ahorcadoRoutes.js`→`hangmanRoutes.js` no aplica acá, pero sí `routes/carreraautoRoutes.js`→`carRaceRoutes.js`, `controllers/carreraautoController.js`→`carRaceController.js`, `models/carreraautoModel.js`→`carRaceModel.js`. Ver [[Deuda Técnica#Nombres en español traducidos a inglés (2026-08-13)]].

## Endpoints (`routes/carRaceRoutes.js`, montado en `/carRaceRoutes`)

| Método | Path | Controller | Auth |
|---|---|---|---|
| POST | `/resultado` | `addResultado` | requiere token + `carreraautoLimiter` (15/min) |
| GET | `/estadisticas` | `getEstadisticasCarrera` | requiere token |

Todo requiere `requireAuth`, incluida la lectura — mismo criterio que [[Módulo FlipCoin]] (a diferencia de `GET /kfruitRoutes/score`, público). Ver [[Módulo Session]].

## Controllers (`controllers/carRaceController.js`)

- **`addResultado`**: `auto` del body → valida contra la lista fija `AUTOS` (`["rojo","azul","cian","blanco"]`, exportada desde el modelo, 400 si no matchea); si pasa, `insertResultado(auto)`, responde `{ success: true }`.
- **`getEstadisticasCarrera`**: sin params → `getEstadisticas()`, responde `{ victorias, total }` (conteo crudo por auto + total de carreras — el cálculo de porcentaje/cuota es del frontend, ver [[CarRace]], mismo reparto de responsabilidades que `getPorcentaje` en [[Módulo FlipCoin]]).

## Modelo (`models/carRaceModel.js`)

- **`AUTOS`**: constante exportada, los 4 nombres válidos en minúsculas (`rojo`/`azul`/`cian`/`blanco`) — reusada tanto por el controller (validación) como internamente por `getEstadisticas` para saber qué contar.
- **`insertResultado(auto)`**: `create` en `car_race_results` (`data: { car: auto }` — el parámetro/variable sigue llamándose `auto` en el código, solo la columna de la tabla es `car`).
- **`getEstadisticas()`**: un `count()` total + un `count({where:{car:auto}})` por cada uno de los 4 autos (4 queries adicionales, sin agrupar por `groupBy` — volumen bajo, no justifica la complejidad).

## Dominio de negocio

`auto`: string en minúsculas, uno de `rojo`/`azul`/`cian`/`blanco` — mismo mapeo que `grabar(nombreGanador().toLowerCase())` del Java original (que escribía ese string a una línea de `ganadores.txt`). La carrera en sí (posiciones, avance aleatorio, detección de ganador) se resuelve client-side en `CarRace.js`, igual que el resto de la lógica de juego del proyecto (reparto de BlackJack, física de Kfruit) — el backend acá es puramente persistencia del historial + fuente de los conteos para calcular cuotas, no autoridad del resultado de la carrera.

## Consumido por

Servicio frontend `CarRaceServices` (ver [[Frontend Model Services Utils#Services]]), usado por [[CarRace]].
