---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Guestbook

⬅️ Volver a [[Backend]]

Libro de firmas 3D sobre el escritorio de la escena raíz (ver [[Escena 3D#Libro de firmas 3D (`guestbook.js`)]]) — no es una app de [[KneOS Portfolio|KneOS]], vive en `public/js/guestbook.js`, fuera del iframe. Agregado 2026-08-31. Cada visitante puede dejar **una** firma (mensaje + trazos dibujados a mano), editable/borrable por su propia sesión; el resto de los visitantes solo la leen.

> [!success] Trazos vectoriales en `Json`, nunca un PNG
> La firma dibujada se guarda como un array de polilíneas con puntos `[x, y]` normalizados 0..1 (`[[[0.1,0.5],[0.12,0.44]], [...]]`), no como blob/bytea. Mismo criterio que `kpaint_files.pixels`/`camera_photos.pixels`: el frontend redibuja los trazos sobre un `<canvas>` con `moveTo`/`lineTo` — nítido a cualquier zoom, sin depender de la resolución de un raster fijo, y sin introducir la primera columna binaria del proyecto (no hay `multer` ni carpeta de uploads en el repo). Contrapartida aceptada: no hay un link a un PNG para compartir — si hiciera falta, se genera client-side con `canvas.toDataURL()` a partir de los mismos trazos.

## Endpoints (`routes/guestbookRoutes.js`, montado en `/guestbookRoutes`)

| Método | Path | Auth | Controller |
|---|---|---|---|
| GET | `/signatures` | público | `getAll` |
| PUT | `/signature` | `requireAuth` + `guestbookLimiter` | `save` |
| DELETE | `/signature` | `requireAuth` + `guestbookLimiter` | `remove` |

`GET` queda público (como `GET /kfruitRoutes/score`) para que cualquiera pueda hojear el libro sin sesión — mismo motivo por el que "una firma por sesión"/"editar la propia" sí exigen `requireAuth` en las escrituras: ambas reglas necesitan `pc_id`.

## Controllers (`controllers/guestbookController.js`)

- **`getAll`**: público — lee `readSessionToken(req.cookies)` (mismo helper que `verificarSesion`, sin exigirlo) para calcular `mine` (el `signature_id` propio, o `null`), y devuelve `{ signatures, mine }`. `pc_id` **nunca** viaja en la respuesta, ni el propio ni el ajeno.
- **`save`**: valida `isNonEmptyString(message, 280)` + `isStrokeArray(strokes, 64, 4096)` (`utils/validation.js`) — `400 { error: "Datos inválidos" }` si no. Llama `saveSignature(req.pcId, message.trim(), strokes)` (upsert) → `{ success: true }`.
- **`remove`**: `deleteSignature(req.pcId)` — `404` si la sesión no tenía firma.

## Modelo (`models/guestbookModel.js`)

- **`getSignatures()`**: `findMany({orderBy:{signature_id:'asc'}})` — el orden de aparición en el libro es por `signature_id` ascendente, así **editar una firma no la mueve de página**.
- **`saveSignature(pc_id, message, strokes)`**: `guestbook_signatures.upsert({where:{pc_id}, update:{message,strokes}, create:{pc_id,message,strokes}})` — mismo patrón que `themeModel.setColor`: firmar de nuevo reemplaza la firma anterior de la misma sesión en vez de crear una segunda (lo garantiza `pc_id @unique` a nivel de schema, no solo a nivel de aplicación).
- **`deleteSignature(pc_id)`**: `deleteMany({where:{pc_id}})`, devuelve `count > 0` — no tira si la sesión no tenía firma, lo resuelve el controller con un `404`.

## Validación (`utils/validation.js`)

`isStrokeArray(value, maxStrokes, maxPoints)` — array de polilíneas, cada punto `[x, y]` con `x`/`y` finitos en `[0,1]`; tope de 64 trazos / 4096 puntos totales para no dejar crecer el body sin control (`express.json()` en `server.js` no tiene `limit` propio, default 100kb — un trazo bien dentro de ese tope pesa ~2-8KB). Mismo estilo que `isIndexArray`.

## Tabla `guestbook_signatures` (`prisma/schema.prisma`, 2026-08-31)

```prisma
model guestbook_signatures {
  signature_id Int       @id @default(autoincrement())
  pc_id        String    @unique @db.VarChar
  message      String    @db.VarChar(280)
  strokes      Json
  created_at   DateTime? @default(now())
  updated_at   DateTime? @default(now()) @updatedAt
  sessions     sessions  @relation(fields: [pc_id], references: [pc_id], onDelete: NoAction, onUpdate: NoAction)
}
```

`pc_id` `@unique` (no `@id` directo) — mismo criterio que `theme_settings`/`kfruit_keybinds`: PK autoincremental propia, separada de la FK. `reset.sql` (gitignored) incluye `guestbook_signatures` en su `TRUNCATE ... CASCADE`, junto con el resto de datos de sesión.

## Consumido por

`public/js/guestbook.js` (frontend de la escena raíz, no un `services/*.js` de KneOS) — `fetch` directo a `/guestbookRoutes/*` y a `/session/verificar`+`/session/nueva` (asegura que exista sesión antes de firmar, ya que normalmente la crea KneOS *dentro* del iframe con delay). Ver [[Escena 3D#Libro de firmas 3D (`guestbook.js`)]] para el render de páginas y el modal de firma.
