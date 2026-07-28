---
tags:
  - portfolio/kneos
  - deuda-tecnica
---

# Deuda Técnica

⬅️ Volver a [[KneOS Portfolio]]

Hallazgos transversales de la exploración del código, útiles para priorizar mejoras.

> [!note] Apps sin terminar
> Solo [[Kmd]] (terminal) sigue siendo un stub vacío (confirmado 2026-07-27, sin cambios).

> [!info] `lampara.glb` sigue huérfano (confirmado 2026-07-27)
> `public/models/lampara.glb` existe pero no se carga en ninguna parte de [[Escena 3D]] ni en ningún `.js` del proyecto. El usuario decidió dejarlo así (tenía pensado usarlo más adelante) — no remover sin preguntar de nuevo.

## Decisiones cerradas (no son bugs a resolver)

> [!warning] Autenticación real — decisión revertida (2026-07-28)
> La decisión de 2026-07-27 de abajo (tachada) quedó obsoleta: el usuario pidió explícitamente agregar auth real después de confirmar que cualquiera con las URLs podía insertar/editar/borrar datos de otra sesión con solo mandar el `pc_id` ajeno en el body — el riesgo se reevaluó como no tan bajo como se pensaba. Se implementó JWT de sesión (`middlewares/auth.js`): `POST /session/nueva` firma un token (`{ pcId }`, ~1 año), el cliente lo manda como `Authorization: Bearer` en cada request, y `req.pcId` (derivado del token, verificado por el middleware `requireAuth`) reemplaza por completo al `pc_id` que antes mandaba el cliente en el body/URL — ya no se puede operar a nombre de otra sesión. De paso se cerraron dos agujeros de *ownership* que el JWT solo no resolvía: `changeChatName`/`saveMessage` (`models/kneAI.js`) no filtraban por `pc_id`, y todo el módulo `folder_styles` no manejaba `pc_id` en absoluto (ownership derivado ahora vía el join `folder_styles.folder_id → icons.pc_id`). Sigue sin haber cuentas/login — es token de sesión anónima, no autenticación de usuarios reales. Ver [[Módulo Session]], [[Módulo Icon]], [[Módulo Kfruit]], [[Módulo KneAI]], [[Backend]].
>
> ~~Se evaluó el hallazgo "sin autenticación real" y se decidió explícitamente no construir un sistema de auth real (cuentas, login, sesiones firmadas): es un portfolio personal, no un producto con datos sensibles ni usuarios reales, y el `pc_id` (UUID en `localStorage`) ya da un aislamiento razonable para el caso de uso — el riesgo práctico de que alguien adivine el `pc_id` de otra persona es bajo, y el costo de implementar auth real no se justifica. No proponer esto de nuevo salvo que cambie el propósito del proyecto (ej. pasar a tener usuarios reales).~~ (2026-07-27, revertida el 2026-07-28)

## Resueltos

> [!success] Validación de entrada agregada a todos los controllers restantes (2026-07-27)
> Se agregó `utils/validation.js` (helper compartido: `isNonEmptyString`, `isString`, `isValidId`, `isBoolean`) y se aplicó validación de tipo/formato en `iconController.js` (10 endpoints), `txtController.js` (2), `kneAI.js` (5, incluyendo chequeo del enum `role_type`) y `kfruitController.js` (`editKeybinds`/`getUserKeybinds`) — todos devuelven `400` ante datos mal formados en vez de dejarlos pasar a Prisma. Probado end-to-end contra el servidor real: casos válidos e inválidos en los 4 módulos. Ver [[Módulo Icon]], [[Módulo Txt]], [[Módulo KneAI]], [[Módulo Kfruit]], [[Backend]].

> [!success] `ARCHITECTURE.md` corregido: Clock no es un stub (2026-07-27)
> El repo decía "Clock: stub sin implementar" en dos lugares (descripción de `core/` y árbol de archivos) — se corrigió a la descripción real (reloj + calendario, completo). Kmd sigue correctamente marcado como sin implementar.

> [!success] `html2canvas` removido (2026-07-27)
> Confirmado que no había ninguna llamada a esa librería en todo el proyecto (`grep` sin resultados fuera del propio `<script>`). Se sacó la etiqueta de `public/index.html`. `lampara.glb` se dejó tal cual (decisión del usuario, tenía pensado usarlo más adelante) — ver nota arriba.

> [!success] Mensaje de error copiado en `newIcon` — corregido (2026-07-27)
> Respondía `"Error al guardar mensaje. " + err` (texto de otro controller, con el objeto `err` completo concatenado). Ahora responde `{ error: "Error al crear el icono" }`, igual que el resto de los controllers del archivo. Confirmado que el frontend (`IconServices`) no depende del texto del mensaje. Ver [[Módulo Icon]].

> [!success] `Groq.js` sin try/catch — corregido, más un bug encontrado al revisar (2026-07-27)
> Se agregó `try/catch` + chequeo de `response.ok` a `ask()`/`getTitle()`, devolviendo `null` en error (mismo patrón que el resto de los servicios). Al revisar el caller se encontró algo peor que lo documentado: en `KneAI.js`, si `getTitle()` fallaba, el mensaje del usuario **nunca se enviaba** (el bloque abortaba antes de mostrarlo). Se corrigió: ahora un fallo de `getTitle` solo salta el renombrado, el mensaje se sigue enviando igual. Un fallo de `ask()` sigue sin mostrar nada (ya era el comportamiento esperado, ahora es explícito con un `if (!message) return`). Ver [[Frontend Model Services Utils#Services]], [[KneAI]].

> [!success] Condición de carrera en `getKeybinds` cerrada (2026-07-27)
> Se agregó `UNIQUE (pc_id)` en `kfruit_keybinds` (tabla estaba vacía) y se reescribió el patrón get-or-create manual como `upsert` atómico de Prisma. Probado con 10 requests concurrentes al mismo `pc_id` nuevo: una sola fila creada. Ver [[Módulo Kfruit]].

> [!success] Condición de carrera en `saveFolderStyle` cerrada (2026-07-28)
> Encontrado al tocar `folder_styles` para agregarle ownership vía JWT: `folder_id` no tenía `UNIQUE`, y el patrón manual `findFirst` + `create`/`update` dejaba una ventana para que dos requests concurrentes sobre la misma carpeta nueva crearan dos filas. Mismo arreglo que ya se había aplicado a `kfruit_keybinds` (ver arriba): se agregó `UNIQUE (folder_id)` en Postgres (`folder_styles_folder_id_unique`, tabla estaba vacía), se reintrospeccionó el schema y se reescribió `saveFolderStyle` como `prisma.folder_styles.upsert({where:{folder_id}, update, create})`. Probado con 10 llamadas concurrentes a la misma carpeta nueva: una sola fila. Ver [[Módulo Folder Styles]].

> [!success] `kfruit_score` sin PK — corregido (2026-07-27)
> Se confirmó contra `pg_constraint` que la tabla no tenía ninguna PRIMARY KEY (solo un `UNIQUE`). Se agregó `PRIMARY KEY (id_score)` en Postgres (dropeando el unique redundante) y se re-introspeccionó el schema (`@unique` → `@id`). No era intencional. Ver [[Backend]].

> [!success] Borrado en cascada de carpetas en el backend (2026-07-27)
> `deleteIcon` (backend) ahora borra recursivamente todo el subárbol (`deleteIconRecursivo`, hijos antes que el padre) en vez de depender únicamente de que el frontend borre hijo por hijo. Antes, pegarle al endpoint directamente sobre una carpeta con contenido fallaba con 500 por la FK `icons_icons_fk` (`onDelete: NoAction`). Probado con un árbol de 3 niveles y contra el endpoint HTTP real. Ver [[Módulo Icon]], [[Menús Contextuales]].

> [!success] `connect-pg-simple` removida (2026-07-27)
> Confirmado que no había ninguna referencia en el código (`grep` sin resultados). `npm uninstall connect-pg-simple`. Ver [[Backend]].

> [!success] Inconsistencia `succes`/`success` unificada (2026-07-27)
> Se cambiaron los 9 lugares que respondían `{ succes: true }` (typo) a `{ success: true }`: `iconController.js` (7×: editDesktopPlace/editParent/editName/editSrc/editLastOpened/editFav/removeIcon), `txtController.js` (saveContent), `kfruitController.js` (editKeybinds) y `kneAI.js` (editChatName). Se actualizó también el único consumidor que leía el campo, `IconServices.deleteIcon()` (`data.succes` → `data.success`). Probado end-to-end contra el endpoint real de borrado de íconos. Ver [[Módulo Icon]], [[Módulo Txt]], [[Módulo Kfruit]], [[Módulo KneAI]].

> [!success] Leaderboard de Kfruit validado + rate limit (2026-07-27, endurecido 2026-07-28)
> `POST /kfruitRoutes/score` valida `name` (string, 1–3 caracteres) y `score` (entero 0–999999) en el controller, y tiene rate limiting (`express-rate-limit`, 5 req/min por IP) vía `middlewares/rateLimiters.js`. Desde el JWT de sesión (2026-07-28) también requiere `requireAuth` — hace falta un token de sesión válido para postear un puntaje, ya no es un endpoint totalmente anónimo. Sigue siendo posible mandar puntajes falsos "legítimos" dentro de esos rangos con una sesión propia; eso es aceptado, solo se acotó el abuso burdo. `GET /kfruitRoutes/score` (leer el leaderboard) sigue público, sin token. Ver [[Módulo Kfruit]].

> [!success] `getMessagesContext` — eliminado, no corregido (2026-07-27)
> Se confirmó el bug (`result[0]` en vez del array de `getChatContext`), pero también que el endpoint `GET /kneAI/chat/:chat_id/context` era **código muerto**: nadie del frontend lo llamaba. El contexto real del LLM lo arma `KneAI.js` en el cliente y se pasa directo a `Groq.ask(...)`. Se eliminaron la ruta, el controller y el modelo en vez de arreglar el bug. Ver [[Módulo KneAI]].
