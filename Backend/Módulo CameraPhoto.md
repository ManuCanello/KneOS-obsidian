---
tags:
  - portfolio/kneos
  - backend
---

# Módulo CameraPhoto

⬅️ Volver a [[Backend]]

Persistencia del contenido de una foto sacada con [[Camera]]: un array de 4096 (64×64) bytes de gris (0-255), uno por píxel ya ditherizado a 4 niveles — **sin color**, el color se resuelve siempre client-side al dibujar (ver [[Camera#Otras apps reactivas al cambio de color|Camera]] e [[ImgFile]]). Acoplada al sistema de íconos ([[Módulo Icon]]) exactamente igual que [[Módulo Txt]] — mismo molde (tabla 1:1 con `files` por `id_icon`, upsert + actualización de `size` en una transacción), agregado 2026-08-19 al crear [[ImgFile]].

> [!info] Formato cambiado el mismo día: de color final a gris (2026-08-19)
> La primera versión guardaba el color ya mezclado (`"#rrggbb"`, ver `isColorArray` más abajo) — una foto quedaba congelada para siempre en el color de [[Config]] que estuviera vigente al sacarla. Se cambió a guardar el byte de gris crudo (uno de `GB_NIVELES = [0, 85, 170, 255]`, ver [[Camera]]) para que [[ImgFile]] pueda repintarla con el color **actual** cada vez que se abre (y en caliente si el color cambia con la ventana abierta). Sin migración de datos: se decidió con el usuario descartar las fotos ya guardadas con el formato viejo (proyecto en desarrollo local, sin datos reales en juego) en vez de sumar una rama de compatibilidad — `prisma/reset.sql` las limpió junto con el resto de datos de sesión de la sesión de desarrollo. La columna sigue siendo `Json`, no hizo falta ninguna migración de esquema.

## Endpoints (`routes/cameraPhotoRoutes.js`, montado en `/cameraPhotoRoutes`, `requireAuth` en todo el router)

| Método | Path | Controller |
|---|---|---|
| PATCH | `/photo` | `savePhoto` |
| GET | `/photo/:id_icon` | `getPhoto` |

## Controllers (`controllers/cameraPhotoController.js`)

- **`savePhoto`**: recibe `id_icon, pixels` del body + `req.pcId` del token. Valida `id_icon` (`isValidId`) y `pixels` con `isGrayscaleArray(value, length)` (`utils/validation.js`, 2026-08-19 — ex `isColorArray`) — array de exactamente `PHOTO_PIXEL_COUNT = 64*64 = 4096` enteros `0 ≤ v ≤ 255` (el mismo `PHOTO_SIZE=64` que `public/KneOS/js/model/cameraPhoto.js`, duplicado como constante en el controller porque el backend no comparte módulos con el frontend). Llama `saveCameraPhoto(...)`; `null` (ícono no existe o no es del `pc_id`) → `404`; éxito → `{ success: true }`. Mismo shape que `saveContent` de [[Módulo Txt]].
- **`getPhoto`**: `req.params.id_icon` + `req.pcId` → `getCameraPhoto(id_icon, req.pcId)`, devuelve `{ pixels }` (array vacío si no hay fila o el ownership no coincide — siempre `200`).

## Modelo (`models/cameraPhotoModel.js`)

- **`saveCameraPhoto(id_icon, pc_id, pixels)`**: verifica ownership (`findFirst` en `files`), calcula `size = Buffer.byteLength(JSON.stringify(pixels), 'utf8')`, y en una `$transaction` hace `camera_photos.upsert` (contenido) + `files.update` (nuevo `size`). Devuelve la fila `camera_photos` resultante o `null` si el ícono no existía.
- **`getCameraPhoto(id_icon, pc_id)`**: `findFirst` en `camera_photos` filtrando por `id_icon` y relación anidada `files: {pc_id}` (a diferencia de `txt`, acá el campo de relación sí se llama `files` — no quedó ningún alias `icons` legado porque el modelo se creó después del rename de la tabla), `select: {pixels: true}`.

## Tabla `camera_photos` (`prisma/schema.prisma`, 2026-08-19)

```prisma
model camera_photos {
  id_icon Int   @id
  pixels  Json
  files   files @relation(fields: [id_icon], references: [id_icon], onDelete: NoAction, onUpdate: NoAction)
}
```

`pixels` es `Json` — un array plano de bytes de gris (0-255), no una referencia a la paleta de 4 tonos que usa Camera al capturar/mostrar (esa paleta se recalcula en runtime a partir de las CSS vars del momento, ver [[Camera]]). Guardar el gris crudo en vez del color ya resuelto es justamente lo que permite que una foto se vea con el color vigente del tema en cualquier momento, en vez de quedar fija en el color con que se sacó.

> [!info] Acoplamiento con Icon
> Igual que `txt`: no se puede crear una fila en `camera_photos` sin un `icon` `img` preexistente (`Camera._guardarFoto` crea el ícono primero vía `DesktopManager.crearIcono`, recién después persiste el contenido), y borrar el icono borra en cascada manual su `camera_photos` (`deleteIconRecursivo`, ver [[Módulo Icon]]).

## Consumido por

Servicio frontend `CameraPhotoServices` (`public/KneOS/js/services/CameraPhotoServices.js`), usado tanto por [[Camera]] (`savePhoto`, al guardar) como por [[ImgFile]] (`getPhoto`, al abrir la foto).
