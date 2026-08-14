---
tags:
  - portfolio/kneos
  - backend
---

# Backend

⬅️ Volver a [[KneOS Portfolio]] · Ver [[Arquitectura]]

Node/Express 5 (ESM) + Prisma 5 sobre PostgreSQL. Capas clásicas: `server.js → routes/ → controllers/ → models/ → db/prisma.js → PostgreSQL`.

## `server.js` — entry point

Configura Express:
1. `import 'dotenv/config'` (primer import, 2026-07-28 — antes era `import dotenv from 'dotenv'` + `dotenv.config()` a mitad de archivo. ESM resuelve todos los `import` antes de ejecutar el cuerpo del módulo, así que con el orden viejo cualquier módulo importado más arriba que leyera `process.env` a nivel de módulo —como el `JWT_SECRET` de `middlewares/auth.js`— lo veía `undefined`). Carga `.env` (`GROQ_API_KEY`, `JWT_SECRET`, `DATABASE_URL`, `PORT`).
2. Middlewares globales: `express.json()`, `cookieParser()` (2026-07-29, para leer la cookie de sesión — ver [[Módulo Session]]), `express.static(public/)`.
3. Monta 14 routers: `/groq`, `/session`, `/kneAI`, `/iconRoutes`, `/kfruitRoutes`, `/hangmanRoutes` (2026-08-12, ex `/ahorcadoRoutes`, renombrada 2026-08-13), `/flipcoinRoutes` (2026-08-12), `/kdleRoutes` (2026-08-12, ex `/wordleRoutes`), `/carRaceRoutes` (2026-08-13, ex `/carreraautoRoutes`, renombrada el mismo día), `/tetrisRoutes` (2026-08-13), `/txtRoutes`, `/folderGroupByRoutes`, `/folderStylesRoutes`, `/folderViewsRoutes`.
4. `GET /` sirve `public/index.html`.
5. `app.listen(PORT)`.

No hay middleware de manejo de errores central en Express; cada controller captura sus propios errores (ver [[Deuda Técnica]]).

## `utils/validation.js` (2026-07-27)

Helper compartido de validación de entrada, usado por todos los controllers de escritura/lectura scopeada (íconos, txt, kneAI, kfruit): `isNonEmptyString(value, maxLength?)`, `isString(value)` (permite vacío, para contenido de txt), `isValidId(value)` (entero positivo), `isBoolean(value)`. Devuelven `400` antes de llegar a Prisma en vez de dejar pasar datos mal formados. Es el equivalente backend de `public/KneOS/js/utils/formato.js` — helpers puros, sin dependencias.

## `middlewares/`

- **`auth.js`** (2026-07-28, cookie `httpOnly` desde 2026-07-29): `signSessionToken(pcId)` firma un JWT (`{ pcId }`, `expiresIn: '365d'`) con `JWT_SECRET`. `setSessionCookie(req, res, token)` la pone en una cookie `kneos_token` (`httpOnly`, `sameSite: 'strict'`, `secure: req.secure`) — antes viajaba en el header `Authorization` y el cliente la guardaba en `localStorage`, legible por cualquier JS de la página; se cambió a cookie para que ni el token ni el `pcId` sean accesibles desde la consola del navegador. `readSessionToken(cookies)` lee `cookies.kneos_token`, verifica firma/vencimiento y devuelve el `pcId` o `null` (no toca la base). `requireAuth(req, res, next)` es el middleware que se monta en cada router protegido: si `readSessionToken` no devuelve nada, `401 { error: "No autorizado" }`; si devuelve, setea `req.pcId`. A nivel de módulo hay un guard `if (!JWT_SECRET) throw` — como todo router protegido importa este archivo, faltar el secreto aborta el arranque en vez de firmar tokens con `undefined`. Ver [[Módulo Session]] y [[Deuda Técnica]].
- **`rateLimiters.js`**: `scoreLimiter` (`express-rate-limit`, 5 req/min por IP, `POST /kfruitRoutes/score` y, desde 2026-08-13, también `POST /tetrisRoutes/score` — mismo shape `{name, score}`, se reusa en vez de sumar un limiter por juego, 2026-07-27); `sessionLimiter` (20 req/hora por IP, `POST /session/nueva` — es la única escritura pública sin token, 2026-07-28); `groqLimiter` (10 req/min, `keyGenerator: req.pcId` en vez de IP porque la ruta ya requiere auth, `/groq/*`, 2026-07-28); `flipcoinLimiter` (30 req/min por IP, `POST /flipcoinRoutes/resultado`, 2026-08-12, más generoso que `scoreLimiter` porque tirar la moneda es la acción principal del juego); `carreraautoLimiter` (15 req/min por IP, `POST /carRaceRoutes/resultado`, 2026-08-13, más bajo que `flipcoinLimiter` porque una carrera completa con animación tarda mucho más que tirar una moneda — el nombre del export no se tradujo, ver [[Deuda Técnica#Nombres en español traducidos a inglés (2026-08-13)]]). Ver [[Módulo Kfruit]], [[Módulo Hangman]], [[Módulo FlipCoin]], [[Módulo Kdle]], [[Módulo CarRace]], [[Módulo Tetris]], [[Módulo Session]], [[Módulo Groq]]. `/hangmanRoutes` y `/kdleRoutes` no tienen rate limiter propio — son de solo lectura, sin ningún endpoint que escriba.

## `db/prisma.js`

Singleton del cliente Prisma (`const prisma = new PrismaClient()`), reexportado y compartido por todos los `models/*.js` para evitar múltiples conexiones al pool.

## Esquema de base de datos (`prisma/schema.prisma`)

| Tabla              | PK              | Campos clave                                                                 | Relaciones |
|--------------------|-----------------|-------------------------------------------------------------------------------|------------|
| `sessions`         | `pc_id`         | `creation_date`                                                                | 1–N con `kneai_chats`, `kneai_messages`; 1–1 con `kfruit_keybinds`, `tetris_keybinds` |
| `files` (ex `icons`, 2026-07-29) | `id_icon`       | `name`, `ext`, `src`, `desktop_place`, `pc_id`, `parent_id`, `size`, `fav`, timestamps | self-relation vía `parent_id` (jerarquía carpetas); 1–1 con `txt` |
| `txt`              | `id_icon`       | `txtcontent`                                                                   | 1–1 con `files` (comparte PK) |
| `kfruit_keybinds`  | `id_keybinds`   | `pc_id`, `moveleft` (def. `ArrowLeft`), `moveright` (def. `ArrowRight`), `drop` (def. `ArrowDown;Space`) | N–1 con `sessions` |
| `kfruit_score`     | `id_score`      | `name`, `score`                                                                | ninguna — leaderboard global anónimo |
| `words` (2026-08-13, unificación de `ahorcado_palabras` + `kdle_palabras`, 2026-08-12 cada una; tabla `palabras` resultante renombrada a `words` más tarde el mismo día — solo el esquema, el contenido sigue en español) | `id_word` | `word`, `created_at` | ninguna — catálogo fijo compartido por Ahorcado y Kdle, sin `pc_id`, mismo criterio que `folder_group_by`/`folder_views` (no `kfruit_score`: no crece, es de solo lectura). Ahorcado usa cualquier largo; Kdle filtra a las de 5 letras al elegir al azar (`WHERE LENGTH(word) = 5` en SQL crudo, ver [[Módulo Kdle]]) — la vieja `kdle_palabras` tenía esa restricción como `VarChar(5)` en el schema, acá ya no aplica porque la tabla es compartida |
| `flipcoin_results` (2026-08-12, ex `flipcoin_resultados`) | `id_result` | `result` (Int, 0=cara/1=cruz), `created_at` | ninguna — log global sin `pc_id`, crece con cada tirada, mismo criterio que `kfruit_score` |
| `car_race_results` (2026-08-13, ex `carreraauto_resultados`) | `id_result` | `car` (VarChar, `rojo`/`azul`/`cian`/`blanco` — el valor no se tradujo, solo la columna), `created_at` | ninguna — log global sin `pc_id`, crece con cada carrera, mismo criterio que `flipcoin_results` |
| `tetris_keybinds` (2026-08-13) | `id_keybinds` | `pc_id`, `moveleft` (def. `KeyA;ArrowLeft`), `moveright` (def. `KeyD;ArrowRight`), `rotate` (def. `KeyC;ArrowUp`), `drop` (def. `Space`) | N–1 con `sessions` — calco de `kfruit_keybinds` con una columna más |
| `tetris_score` (2026-08-13) | `id_score` | `name`, `score` | ninguna — leaderboard global anónimo, calco de `kfruit_score` |
| `kneai_chats`      | `chat_id`       | `pc_id`, `chat_name`, `created_at`                                             | 1–N con `kneai_messages` |
| `kneai_messages`   | `message_id`    | `chat_id`, `role` (enum `user`\|`system`), `message`, `pc_id?`                 | N–1 con `kneai_chats` |
| `folder_styles`    | `folder_style_id` | `folder_id` (**`@unique`**, 2026-07-28), `folder_view` (Int, FK), `folder_group_by` (Int, FK), `folder_group_order` (**VarChar**, "asc"/"desc") | N–1 con `files` (vía `folder_id`); N–1 con `folder_group_by` (FK `folder_styles_folder_group_by_fk`); N–1 con `folder_views` (FK `folder_styles_folder_views_fk`) |
| `folder_group_by`  | `folder_group_by_id` | `folder_group_by_desc`                                                 | 1–N con `folder_styles` |
| `folder_views`     | `folder_view_id` | `folder_view_desc`                                                         | 1–N con `folder_styles` — catálogo de "Íconos grandes"/"Íconos pequeños"/"Lista" |

> [!info] `folder_group_by`, `folder_views` y `folder_styles` ya tienen código (actualizado 2026-07-27)
> Las tres tablas aparecieron en sucesivos `prisma db pull` sin código backend inicialmente. `folder_group_by` y `folder_views` son catálogos de solo lectura (ver [[Módulo Folder Group By]] y [[Módulo Folder Views]]) que alimentan los submenús "Ordenar por" y "Ver" de [[Folder]], respectivamente. `folder_styles` persiste el estilo completo por carpeta (vista + criterio + dirección de orden) — ver [[Módulo Folder Styles]].
>
> Entre pulls sucesivos se fueron agregando en la BD las FKs `folder_styles_folder_group_by_fk` y `folder_styles_folder_views_fk` (antes esas columnas eran `Int` sueltos, sin relación declarada).
>
> **Cambio de columna (2026-07-27):** `folder_group_order` se alteró manualmente de `Int` a `character varying` (`ALTER TABLE ... ALTER COLUMN folder_group_order TYPE character varying`, tabla estaba vacía) para poder guardar literalmente `"asc"`/`"desc"` en vez de inventar una codificación numérica. Decisión explícita del usuario ante la ambigüedad del tipo original.
>
> **`folder_id` ahora es `@unique` (2026-07-28):** `ALTER TABLE folder_styles ADD CONSTRAINT folder_styles_folder_id_unique UNIQUE (folder_id)` (tabla estaba vacía), para cerrar una condición de carrera en `saveFolderStyle` — ver [[Módulo Folder Styles]] y [[Deuda Técnica]].

> [!success] `kfruit_score.id_score` ahora tiene PK real (2026-07-27)
> Se confirmó vía `pg_constraint` que la tabla no tenía ninguna PRIMARY KEY, solo el `UNIQUE` (`kfruit_score_unique`). Se hizo `ALTER TABLE` para dropear el unique y agregar `PRIMARY KEY (id_score)`, y se re-introspeccionó — el schema ahora dice `@id` en vez de `@unique`. Ver [[Deuda Técnica]].

`files.parent_id` modela archivos/carpetas anidados con `onDelete: NoAction` a nivel de FK — el borrado en cascada real se resuelve a mano en `deleteIcon`/`deleteIconRecursivo` (ver [[Módulo Icon]], resuelto 2026-07-27).

> [!info] Tabla `icons` renombrada a `files` (2026-07-29)
> Ver la nota completa en [[Módulo Icon]] — rutas/archivos/funciones del backend conservan el nombre viejo ("icon"), solo cambió el nombre de tabla/modelo Prisma.

`kfruit_keybinds.drop` admite múltiples teclas separadas por `;` (p. ej. `"ArrowDown;Space"`) — mismo esquema en las 4 columnas de `tetris_keybinds`.

## Dominios de negocio

Cada dominio tiene su propia nota con el detalle de rutas, controllers y modelos:

- [[Módulo Session]] — alta/verificación del `pc_id`
- [[Módulo Icon]] — CRUD de íconos/carpetas, cálculo recursivo de tamaños
- [[Módulo KneAI]] — chats y mensajes del asistente IA
- [[Módulo Groq]] — proxy hacia la API de Groq (LLM)
- [[Módulo Txt]] — contenido de archivos de texto
- [[Módulo Kfruit]] — keybinds y leaderboard del minijuego
- [[Módulo Hangman]] — banco de palabras global del minijuego
- [[Módulo FlipCoin]] — log de tiradas de cara/cruz
- [[Módulo Kdle]] — banco de palabras de 5 letras del minijuego
- [[Módulo CarRace]] — log de carreras y cuotas del minijuego de apuestas a autos
- [[Módulo Tetris]] — keybinds y leaderboard del minijuego de bloques
- [[Módulo Folder Group By]] — catálogo de criterios de "Ordenar por" de Folder
- [[Módulo Folder Views]] — catálogo de estilos de "Ver" de Folder
- [[Módulo Folder Styles]] — persistencia del estilo (vista/orden) por carpeta

## Variables de entorno

`GROQ_API_KEY`, `JWT_SECRET` (2026-07-28 — firma/verifica el JWT de sesión, ver [[Módulo Session]]), `DATABASE_URL`, `PORT`. Existen además `DB_HOST/PORT/NAME/USER/PASSWORD` en `.env` que no se referencian directamente en el código JS (probablemente vestigio de una config anterior o usadas para componer `DATABASE_URL` a mano).

## Dependencias (`package.json`)

`@prisma/client`, `prisma`, `pg`, `express`, `uuid`, `jsonwebtoken` (agregada 2026-07-28, ver [[Módulo Session]]), `cookie-parser` (agregada 2026-07-29, ídem), `dotenv`, `express-rate-limit` (agregada 2026-07-27, ver [[Módulo Kfruit]]), `an-array-of-spanish-words` (agregada 2026-08-13, diccionario de ~636k palabras usado solo por Kdle para validar intentos, ver [[Módulo Kdle]]).

> [!info] `connect-pg-simple` removida (2026-07-27)
> Estaba declarada pero no se importaba en ningún archivo — vestigio de un enfoque de sesión anterior (store de sesiones Express sobre Postgres) reemplazado por el sistema propio de `pc_id`. Se confirmó que no había ninguna referencia y se hizo `npm uninstall`. Ver [[Deuda Técnica]].
