---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Kdle

⬅️ Volver a [[Backend]]

Catálogo de palabras del minijuego Kdle ([[Kdle]], renombrado de "Wordle" — este módulo reemplaza a la vieja `Módulo Wordle.md`, incluye el renombre de rutas/controller/model/tabla) — reemplaza al `palabras.txt` que leía `Palabra.java` en el original. Catálogo estático, mismo criterio que [[Módulo Hangman]] y que `folder_group_by`/`folder_views`: no tiene `pc_id`, no se consume ni se marca "usada" al jugarse, y por eso `prisma/reset.sql` (local, gitignored) la deja afuera del `TRUNCATE`.

> [!info] Tabla unificada con Ahorcado (2026-08-13)
> Kdle tenía su propia tabla `kdle_palabras` (`VarChar(5)`, 119 filas) separada de `ahorcado_palabras`. Se unificaron ambas en una sola tabla `palabras` (ver [[Módulo Hangman]] para el detalle de la migración y las ~712 palabras nuevas agregadas de paso). Como `palabras` ya no tiene el límite `VarChar(5)` — Ahorcado necesita cualquier largo — Kdle filtra en la query: `getPalabraAlAzar()` pasó de un `findMany` sin condiciones a `$queryRaw` con `WHERE LENGTH(palabra) = 5`, porque el query builder de Prisma no tiene forma de expresar un filtro por longitud de string. El pool de palabras jugables por Kdle pasó de 119 a 301.

> [!info] Validación de intentos contra un diccionario real (2026-08-13)
> `palabras` es solo el pool de *respuestas* posibles (chico, curado a mano) — no alcanza para decidir si lo que el jugador tipeó es una palabra real, así que se sumó un diccionario aparte: el paquete npm [`an-array-of-spanish-words`](https://www.npmjs.com/package/an-array-of-spanish-words) (~636k formas de palabras en español) se instaló como dependencia y se carga una sola vez al importar `models/kdleModel.js` (`createRequire` para importar el `.json` sin depender de qué versión de Node soporta import attributes), normalizado a un `Set` (minúscula, `.normalize('NFD')` + strip de marcas combinantes — de paso convierte "ñ" en "n") porque ni el teclado de Ahorcado ni el input de Kdle pueden tipear tildes/ñ. `existePalabra(palabra)` primero mira ese `Set`; si no está, cae a un `SELECT` contra `palabras` — así la palabra secreta de la partida (que siempre sale de esa tabla) nunca puede quedar "no reconocida" como intento, ni siquiera si es un préstamo que el corpus no tiene (p. ej. "koala").

## Endpoints (`routes/kdleRoutes.js`, montado en `/kdleRoutes`)

| Método | Path | Controller | Auth |
|---|---|---|---|
| GET | `/palabras/azar` | `getPalabraAzar` | requiere token |
| GET | `/palabras/existe/:palabra` | `getExistePalabra` | requiere token |

Solo lectura — no hay ningún endpoint que escriba en la tabla desde la app. Ver [[Módulo Session]].

## Controllers (`controllers/kdleController.js`)

- **`getPalabraAzar`**: sin params → `getPalabraAlAzar()`, responde `{ palabra }` (`null` si la tabla estuviera vacía — el frontend no muestra error en ese caso, ver [[feedback_no_error_message_just_stay]]).
- **`getExistePalabra`**: valida el param `:palabra` contra `/^[a-zA-Z]{5}$/` (`400` si no matchea) antes de llamar a `existePalabra()`, responde `{ existe }` (boolean).

## Modelo (`models/kdleModel.js`)

- **`getPalabraAlAzar()`**: `prisma.$queryRaw` con `SELECT palabra FROM palabras WHERE LENGTH(palabra) = 5`, elige una al azar en JS (`Math.random()`) — sin marcar nada. SQL crudo en vez del query builder porque Prisma no expone un filtro de longitud de string.
- **`existePalabra(palabra)`**: `Set` del diccionario normalizado (ver nota arriba) primero, `SELECT 1 FROM palabras WHERE palabra = $1 LIMIT 1` como respaldo si no está ahí.

## Dominio de negocio

Mismo perfil que [[Módulo Hangman]]: contenido fijo, la app solo lee — con la diferencia de que Kdle necesita que la grilla (5 columnas) coincida exactamente con el largo de la palabra, así que filtra en la query en vez de confiar en una restricción de columna (`VarChar(5)`) como hacía la vieja `kdle_palabras`, que ya no aplica ahora que la tabla es compartida.

> [!info] Seed original de las 119 palabras (2026-08-12)
> El `palabras.txt` original tiene 1000 líneas con muchísimos repetidos y al menos una palabra de largo distinto a 5 (`jirafa`, 6 letras — que `Palabra.java` leía igual y truncaba en silencio a los primeros 5 caracteres al armar `palabra_adivinar`, un bug del original que no se portó). El seed dedupe y filtra a las 119 palabras únicas de exactamente 5 letras, insertadas con un script puntual (`node` directo contra `@prisma/client`, no versionado) — mismo mecanismo que el seed original de `ahorcado_palabras`. Ver la nota de unificación arriba para cómo creció ese pool a 301.

## Consumido por

Servicio frontend `KdleServices` (ver [[Frontend Model Services Utils#Services]]), usado por [[Kdle]].
