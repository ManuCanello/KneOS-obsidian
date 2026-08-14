---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Hangman

⬅️ Volver a [[Backend]]

Catálogo fijo de palabras del minijuego de ahorcado ([[Hangman]]) — reemplaza al array de 256 palabras que antes vivía hardcodeado en `Ahorcado.js` (mismo texto original, `palabras.txt`). Es un catálogo estático, mismo criterio que `folder_group_by`/`folder_views` (ver [[Backend]]): no tiene `pc_id`, no se consume ni se marca "usada" al jugarse, y por eso `prisma/reset.sql` (local, gitignored) la deja afuera del `TRUNCATE`.

> [!info] Tabla unificada con Kdle (2026-08-13), renombrada a inglés el mismo día
> Este módulo compartía el mismo perfil que [[Módulo Kdle]] (catálogo fijo, solo lectura, sin `pc_id`) salvo por una cosa: Kdle solo podía jugar palabras de 5 letras. Se unificaron `ahorcado_palabras` y `kdle_palabras` (con 12 palabras duplicadas entre ambas) en una sola tabla `palabras`, y de paso se sumaron ~712 palabras nuevas (validadas contra el paquete npm `an-array-of-spanish-words` para descartar typos, y excluyendo cualquier palabra con `ñ` o acentos porque ni el teclado de Ahorcado ni el input de Kdle pueden tipear esos caracteres). Ahorcado sigue sin filtrar por largo — cualquier fila de la tabla sirve; Kdle filtra a las de 5 letras al elegir (`WHERE LENGTH(word) = 5`, ver [[Módulo Kdle]]). Total resultante: 1075 palabras (301 de ellas de 5 letras). Más tarde, el mismo 2026-08-13, la tabla se renombró de `palabras`/`palabra`/`id_palabra` a `words`/`word`/`id_word` — solo el esquema (nombres de tabla/columna), el contenido sigue siendo palabras reales en español, no se tradujo (ver [[Deuda Técnica#Nombres en español traducidos a inglés (2026-08-13)]]).

## Endpoints (`routes/hangmanRoutes.js`, montado en `/hangmanRoutes`)

| Método | Path | Controller | Auth |
|---|---|---|---|
| GET | `/palabras/azar` | `getPalabraAzar` | requiere token |

Solo lectura — no hay ningún endpoint que escriba en la tabla desde la app. Ver [[Módulo Session]].

## Controllers (`controllers/hangmanController.js`)

- **`getPalabraAzar`**: sin params → `getPalabraAlAzar()`, responde `{ palabra }` (`null` si la tabla estuviera vacía — el frontend no muestra error en ese caso, ver [[feedback_no_error_message_just_stay]]).

## Modelo (`models/hangmanModel.js`)

- **`getPalabraAlAzar()`**: `findMany` de todas las filas de `words` (`select: { word: true }`, sin filtrar por largo) y elige una al azar en JS (`Math.random()`) — sin marcar nada. Es la única función del modelo — no hay ningún `insertPalabra` ni endpoint de escritura expuesto a la app; los seeds (256 originales + ~712 de la unificación) se hicieron con scripts puntuales (`node` directo contra `@prisma/client`, no versionados).

## Dominio de negocio

A diferencia de [[Módulo Kfruit]] (leaderboard que sí crece con cada partida), acá la tabla es contenido fijo: se pobló y después se amplió, y la app solo lee. "PALABRA AL AZAR" en el frontend hace un `SELECT` al azar entre las 1075 filas de `words` (cualquier largo); "ESCRIBIR PALABRA" (la otra opción del menú) ni siquiera pega contra este módulo — juega la palabra tipeada directo, sin persistirla.

> [!warning] Diseño descartado en el camino (2026-08-12, misma sesión, dos vueltas)
> **Vuelta 1**: una primera versión de este módulo tenía un flag `usada` (Boolean) en la tabla, un endpoint `POST /palabras` para cargar palabras a mano, y `GET /palabras/azar` marcaba `usada: true` al sacar una — la idea era que las palabras se "tacharan" a medida que se jugaban, como una bolsa que se consume. El usuario corrigió: el menú debía quedar igual al original (azar / escribir), la BD solo tenía que reemplazar el array estático, y el tachado real es sobre las **letras** dentro de la partida (ver [[Hangman]]), no sobre las palabras. Se sacó la columna `usada`, el endpoint de carga y `palabraLimiter`.
> **Vuelta 2**: interpretando mal un mensaje posterior del usuario ("volver al modelo de antes todas las palabras"), este módulo entero se borró — tabla dropeada, `routes/ahorcadoRoutes.js`/`controllers/ahorcadoController.js`/`models/ahorcadoModel.js`/`AhorcadoServices.js` eliminados, `Ahorcado.js` vuelto a un array hardcodeado de las 256 palabras. El usuario aclaró que el pedido era sobre las letras, no las palabras ("la palabras mostrarlas en la bd"), así que se restauró todo el módulo tal como estaba (tabla recreada + re-seed de las 256 palabras).

## Consumido por

Servicio frontend `HangmanServices` (ex `AhorcadoServices`, renombrado 2026-08-13 — ver [[Frontend Model Services Utils#Services]]), usado por [[Hangman]].
