---
tags:
  - portfolio/kneos
  - backend
---

# Módulo Theme

⬅️ Volver a [[Backend]]

Persiste el color del sistema elegido en [[Config]]: una clave fija (`"verde"`, `"rojo"`, ...), no un hex — el mapeo clave→hex vive solo en el frontend (`model/themeColors.js`), así un futuro repaint de paleta no requiere tocar filas ya guardadas. Agregado 2026-08-19. Mismo molde upsert-por-`pc_id` que [[Módulo Kfruit]]/`tetris_keybinds` (una fila por sesión, creada con el default al primer `GET`), no el molde 1:1-con-`files` de [[Módulo Txt]]/[[Módulo CameraPhoto]] — el color del sistema no cuelga de ningún ícono.

## Endpoints (`routes/themeRoutes.js`, montado en `/themeRoutes`, `requireAuth` en todo el router)

| Método | Path | Controller |
|---|---|---|
| GET | `/color` | `getUserColor` |
| PATCH | `/color` | `editColor` |

## Controllers (`controllers/themeController.js`)

- **`getUserColor`**: `getColor(req.pcId)` → `{ color }`.
- **`editColor`**: valida `color` del body con `isValidThemeColor` (`utils/validation.js`) — `400 { error: "Color inválido" }` si no es una de las 9 claves. Llama `setColor(req.pcId, color)` → `{ success: true }`.

## Modelo (`models/themeModel.js`)

- **`getColor(pc_id)`**: `theme_settings.upsert({where:{pc_id}, update:{}, create:{pc_id}})` — mismo patrón que `kfruitModel.getKeybinds`: si la sesión nunca eligió color, la fila se crea en el acto con el default de schema (`"verde"`) y se devuelve tal cual, sin un branch aparte para "todavía no existe". Devuelve `settings.color`.
- **`setColor(pc_id, color)`**: mismo `upsert`, pero `update: {color}` / `create: {pc_id, color}` — a diferencia de `getColor`, si la fila no existía la crea ya con el color elegido (no con el default, para no pisarlo un instante después con un segundo write).

## Validación (`utils/validation.js`)

`isValidThemeColor(value)` — `THEME_COLOR_KEYS` es un `Set` con las mismas 9 claves que `THEME_COLORS` en `public/KneOS/js/model/themeColors.js` (el backend no comparte módulos con el frontend, mismo patrón que `PHOTO_PIXEL_COUNT` duplicado en [[Módulo CameraPhoto]] — ver ese módulo para la nota completa de por qué se duplica la constante en vez de acoplar los dos lados).

## Tabla `theme_settings` (`prisma/schema.prisma`, 2026-08-19)

```prisma
model theme_settings {
  id_setting Int      @id @default(autoincrement())
  pc_id      String   @unique
  color      String   @default("verde")
  sessions   sessions @relation(fields: [pc_id], references: [pc_id], onDelete: NoAction, onUpdate: NoAction)
}
```

`pc_id` `@unique` (no `@id` directo) — mismo criterio que `kfruit_keybinds`/`tetris_keybinds`: una PK `id_setting` autoincremental propia, separada de la FK, en vez de reusar `pc_id` como PK compuesta.

## Arranque del frontend

`KNEOS.js` llama `ThemeServices.getColor()` y `applyThemeColor(...)` **antes** de crear `DesktopManager`, todavía detrás de `#loadingScreen` — ver [[Config#Arranque (KNEOS.js)]]. `reset.sql` incluye `theme_settings` en su `TRUNCATE ... CASCADE` (datos de sesión, se limpian junto con `sessions`).

## Consumido por

Servicio frontend `ThemeServices` (`public/KneOS/js/services/ThemeServices.js`) — usado por `KNEOS.js` (`getColor`, al arrancar) y [[Config]] (`setColor`, al elegir un color).
