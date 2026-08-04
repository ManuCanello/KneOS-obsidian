---
tags:
  - portfolio/kneos
  - arquitectura
---

# Arquitectura

⬅️ Volver a [[KneOS Portfolio]]

## Stack tecnológico

**Backend**
- [Express 5](https://expressjs.com/) (ESM) como servidor HTTP y de archivos estáticos
- [Prisma](https://www.prisma.io/) + PostgreSQL como ORM/base de datos
- [Groq API](https://groq.com/) (modelo `llama-3.3-70b-versatile`) como proveedor de IA, consumido vía proxy propio para no exponer la API key al cliente
- `uuid` para generar el identificador de sesión (`pc_id`)
- `jsonwebtoken` para firmar/verificar el JWT de sesión (2026-07-28, ver [[Módulo Session]])
- `dotenv` para variables de entorno

**Frontend**
- JavaScript vanilla con ES Modules, sin bundler — se cargan directo en el navegador vía `<script type="module">` e import maps
- [Three.js](https://threejs.org/) (+ `OrbitControls`, `GLTFLoader`, `CSS3DRenderer`) para renderizar la PC en 3D y proyectar el escritorio HTML sobre su pantalla — ver [[Escena 3D]]
- [interact.js](https://interactjs.io/) para mover/redimensionar las ventanas del escritorio
- [js-dos](https://js-dos.com/) para emular DOOM dentro de una ventana ([[Doom]])
- **planck** (Box2D portado a JS) como motor de física 2D del juego Kfruit ([[Kfruit]])
- CSS plano, organizado por módulo

**Infraestructura/build**
- Sin paso de build: todo se sirve estático desde `public/`
- `node --watch` para desarrollo (`npm run dev`)

## Capas del backend

```
server.js → routes/ → controllers/ → models/ → db/prisma.js → PostgreSQL
```

- **`server.js`**: configura Express, sirve `public/` como estático y monta los routers.
- **`routes/`**: definición de endpoints HTTP, sin lógica de negocio.
- **`controllers/`**: parsean el request, llaman a los modelos y devuelven la respuesta JSON.
- **`models/`**: acceso a datos vía Prisma Client (`db/prisma.js`), una función por operación.
- **`prisma/schema.prisma`**: esquema de la base (`sessions`, `files` — ex `icons`, renombrada en Postgres el 2026-07-29, ver [[Módulo Icon]] —, `txt`, `kfruit_keybinds`, `kfruit_score`, `kneai_chats`, `kneai_messages`).

Ver detalle completo en [[Backend]].

## Capas del frontend (KneOS)

`public/index.html` monta la escena 3D (`public/js/main.js`, ver [[Escena 3D]]) y, dentro de ella, un `<iframe>` que carga `public/KneOS/index.html` — el "sistema operativo" propiamente dicho.

```
public/index.html (escena 3D)
  └─ <iframe> public/KneOS/index.html
        └─ KNEOS.js (bootstrap)
              ├─ core/     infraestructura del escritorio → [[Frontend Core]]
              ├─ apps/     una clase por app → [[Apps]]
              ├─ model/    datos/registros estáticos → [[Frontend Model Services Utils]]
              ├─ services/ clientes HTTP → [[Frontend Model Services Utils]]
              ├─ utils/    helpers → [[Frontend Model Services Utils]]
              └─ styles/   CSS modular (main.css importa todo)
```

## Modelo de sesión y autenticación (`pc_id` + JWT en cookie httpOnly)

> [!info] JWT de sesión anónima (2026-07-28 — reemplaza la decisión "sin auth real")
> Cada visitante sigue identificado por un `pc_id` (UUID v4), pero ahora ese `pc_id` viaja **firmado** dentro de un JWT (`middlewares/auth.js`) en vez de viajar suelto en cada body/URL. El middleware `requireAuth` verifica la firma y expone `req.pcId` — los controllers ya no leen ningún `pc_id` que venga del cliente, así que un `pc_id` ajeno en el body/URL simplemente se ignora: es imposible por construcción operar sobre datos de otra sesión sin poseer su token. Sigue sin haber cuentas/login/contraseña — es autenticación de sesión anónima, no de usuarios reales. Ver [[Deuda Técnica]], [[Módulo Session]].

> [!warning] El token viaja en cookie `httpOnly`, no en `localStorage` (2026-07-29)
> Versión inicial: el token se guardaba en `localStorage['kneos_token']` y viajaba como header `Authorization: Bearer`. El usuario pidió explícitamente que ni el token ni el `pc_id` fueran legibles desde la consola de JS del navegador — con `localStorage`, cualquier script de la página (o alguien tipeando en la consola) podía leerlo con `localStorage.getItem(...)`. Se cambió a una cookie `kneos_token` con `httpOnly: true` (invisible para `document.cookie` y para cualquier JS), `sameSite: 'strict'` (la mitigación de CSRF que reemplaza a lo que antes evitaba el header `Authorization` por no adjuntarse solo) y `secure: req.secure`. El servidor la pone (`Set-Cookie`) y la lee (`req.cookies`, vía `cookie-parser`); el cliente no vuelve a tocarla en ningún momento. Verificado con Playwright: `document.cookie` da `""` después de iniciar sesión.

Gestionado por `public/KneOS/js/services/session.js` (`startSession()`):

1. `POST /session/verificar` (la cookie, si existe, viaja sola) — revalida firma **y** que la fila siga en `sessions`, para autorepararse si la sesión fue borrada de la base. Si el server confirma, pone una cookie renovada.
2. Si falló (no había cookie o quedó inválida) → `POST /session/nueva`, que crea la fila en `sessions` y pone una cookie nueva.

El cliente nunca lee ni guarda nada — ambos endpoints solo devuelven `{ success }`, el efecto real es el `Set-Cookie`. Sin recursión (a diferencia del viejo `obtenerPcId`, que se reinvocaba a sí mismo sin guarda si `/session/verificar` respondía que ya no existía). Como mucho dos requests.

`req.pcId` (derivado de la cookie, nunca del cliente) reemplaza a `pc_id` en todos los endpoints scopeados por sesión; ya no aparece en ninguna URL ni body.

## Dominios expuestos por el backend

| Ruta                  | Propósito                                                        | Auth | Nota |
|-----------------------|--------------------------------------------------------------------|------|------|
| `/session`            | Emitir/renovar el token de sesión (JWT)                            | público | [[Módulo Session]] |
| `/iconRoutes`         | CRUD de íconos del escritorio (posición, nombre, jerarquía, alta/baja) | requiere token | [[Módulo Icon]] |
| `/kneAI`               | CRUD de chats y mensajes del asistente IA                          | requiere token | [[Módulo KneAI]] |
| `/groq`               | Proxy hacia la API de Groq (chat + generación de títulos)          | requiere token | [[Módulo Groq]] |
| `/txtRoutes`          | Persistencia del contenido de archivos de texto                    | requiere token | [[Módulo Txt]] |
| `/kfruitRoutes`       | Keybinds (requiere token) y leaderboard (`GET` público, `POST` requiere token) | mixto | [[Módulo Kfruit]] |
| `/folderStylesRoutes` | Vista/orden guardados por carpeta                                   | requiere token | [[Módulo Folder Styles]] |
| `/folderGroupByRoutes`, `/folderViewsRoutes` | Catálogos estáticos de criterios de vista/orden       | público | — |
