---
tags:
  - portfolio/kneos
  - backend
---

# Módulo VROOM

⬅️ Volver a [[Backend]]

Log de carreras del minijuego de apuestas a autos ([[VROOM]], ex CarRace) — global (sin `pc_id`), crece con cada carrera corrida. Mismo criterio que [[Módulo FlipCoin]] (`flipcoin_results`): a diferencia de [[Módulo Hangman]]/[[Módulo Kdle]] (catálogo fijo `words`, solo lectura), este módulo tiene un endpoint de escritura.

> [!info] Renombrado de CarRace a VROOM (2026-09-18, a pedido del usuario)
> Todo el stack se renombró: `routes/carRaceRoutes.js`→`vroomRoutes.js`, `controllers/carRaceController.js`→`vroomController.js`, `models/carRaceModel.js`→`vroomModel.js`, ruta montada `/carRaceRoutes`→`/vroomRoutes`, `middlewares/rateLimiters.js` export `carreraautoLimiter`→`vroomLimiter`. La tabla se renombró con `ALTER TABLE car_race_results RENAME TO vroom_results` (preserva los datos, no un drop+create) — ver [[Deuda Técnica#Nombres en español traducidos a inglés (2026-08-13)]] para el rename anterior (español→inglés, distinto de este).
>
> Mismo día se agregó la columna `premio` (Int, default 0, ver abajo).

## Endpoints (`routes/vroomRoutes.js`, montado en `/vroomRoutes`)

| Método | Path | Controller | Auth |
|---|---|---|---|
| POST | `/resultado` | `addResultado` | requiere token + `vroomLimiter` (15/min) |
| GET | `/estadisticas` | `getEstadisticasCarrera` | requiere token |

Todo requiere `requireAuth`, incluida la lectura — mismo criterio que [[Módulo FlipCoin]] (a diferencia de `GET /kfruitRoutes/score`, público). Ver [[Módulo Session]].

## Controllers (`controllers/vroomController.js`)

- **`addResultado`**: `auto` y `premio` del body → `auto` valida contra la lista fija `AUTOS` (`["rojo","azul","cian","blanco"]`, exportada desde el modelo, 400 si no matchea); `premio` se sanitiza a un entero `≥ 0` (`Number(premio) > 0 ? Math.round(Number(premio)) : 0` — nunca confía en lo que mande el cliente sin normalizar). `insertResultado(auto, premio)`, responde `{ success: true }`.
- **`getEstadisticasCarrera`**: sin params → `getEstadisticas()`, responde `{ victorias, total, plataGanada }` (conteo crudo por auto + total de carreras + plata ganada acumulada por auto — el cálculo de porcentaje/cuota sigue siendo del frontend, ver [[VROOM]]).

## Modelo (`models/vroomModel.js`)

- **`AUTOS`**: constante exportada, los 4 nombres válidos en minúsculas (`rojo`/`azul`/`cian`/`blanco` — el `id` interno no cambió con el rename de nombres visibles a MAX/LEWIS/FRANCO/LANDO, ver [[VROOM]]) — reusada tanto por el controller (validación) como internamente por `getEstadisticas` para saber qué contar.
- **`insertResultado(auto, premio = 0)`**: `create` en `vroom_results` (`data: { car: auto, premio }`).
- **`getEstadisticas()`**: un `count()` total + por cada uno de los 4 autos, un `count({where:{car:auto}})` (victorias) y un `aggregate({where:{car:auto}, _sum:{premio:true}})` (plata ganada) — 9 queries adicionales sin `groupBy`, mismo criterio de "volumen bajo, no justifica la complejidad" que ya tenía el conteo de victorias.

## Columna `premio` (2026-09-18)

`vroom_results.premio` (Int, default 0) — lo que pagó esa fila a quien apostó a `car` (el auto ganador de esa carrera) **y ganó**: siempre `monto apostado × cuota del auto ganador` en el momento de esa carrera, sin importar si quien corrió la carrera había apostado a ese auto o a otro (a pedido explícito del usuario: "plata ganada" de un auto se suma en cada carrera que ese auto gana, la haya apostado o no quien la corrió). `getEstadisticas().plataGanada` es `SUM(premio)` agrupado por `car`, expuesto en la pantalla "VER PORCENTAJES" de [[VROOM]] como columna "PLATA GANADA".

Nota: esto **no** es el pozo total apostado por auto (no se guarda cuánto se apostó a un auto que no ganó) — es la suma de lo efectivamente pagado, calculada con la cuota de cada carrera individual.

## Dominio de negocio

`auto`: string en minúsculas, uno de `rojo`/`azul`/`cian`/`blanco` — mismo mapeo que `grabar(nombreGanador().toLowerCase())` del Java original. La carrera en sí (posiciones, avance aleatorio, detección de ganador) se resuelve client-side en `VROOM.js`, igual que el resto de la lógica de juego del proyecto — el backend acá es puramente persistencia del historial + fuente de los conteos/sumas para calcular cuotas y "plata ganada", no autoridad del resultado de la carrera.

## Consumido por

Servicio frontend `VROOMServices` (ex `CarRaceServices` — ver [[Frontend Model Services Utils#Services]]), usado por [[VROOM]].
