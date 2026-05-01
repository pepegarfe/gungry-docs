# CLAUDE.md — Contexto del Proyecto Gungry

Documento generado el 30 de abril de 2026 para continuar el trabajo de reactivación de la app Gungry.

---

## 1. Descripción del Proyecto

**Gungry** es una app de directorio de lugares (tipo Yelp/NearMe) comprada en CodeCanyon (Nearme 5 de Quanlabs). Stack original: Parse Server + Express (backend), Angular + Fuse (admin portal), Ionic/Angular (app móvil PWA).

### URLs de Producción

| Componente | URL |
|---|---|
| Backend (Parse Server 7) | https://gungry-backend-production.up.railway.app/api/1 |
| Parse Dashboard | https://gungry-backend-production.up.railway.app/dashboard |
| Admin Portal | https://manager-gungry.netlify.app / https://manager.gungry.com |
| App Móvil PWA | https://gungry.netlify.app / https://app.gungry.com (SSL pendiente) |
| Base de datos | MongoDB Atlas — cluster GungryDB, db: nearmedb |

### Repositorios GitHub

- Backend: https://github.com/pepegarfe/gungry-backend
- Admin Portal: https://github.com/pepegarfe/manager-gungry
- App Móvil: https://github.com/pepegarfe/gungry-app

---

## 2. Variables de Entorno (Railway)

Las credenciales reales se encuentran en **Railway → proyecto gungry-backend → Variables**. No se incluyen aquí por seguridad.

| Variable | Descripción | Dónde encontrarla |
|---|---|---|
| `MONGODB_URI` | Connection string de MongoDB Atlas | Railway → Variables |
| `APP_ID` | `gungry-app` | Railway → Variables |
| `MASTER_KEY` | Clave maestra de Parse Server | Railway → Variables |
| `PUBLIC_SERVER_URL` | `https://gungry-backend-production.up.railway.app/api/1` | Railway → Variables |
| `PARSE_DASHBOARD_USER` | Usuario del dashboard | Railway → Variables |
| `PARSE_DASHBOARD_PASS` | Password del dashboard | Railway → Variables |
| `NODE_ENV` | `production` | Railway → Variables |
| `PORT` | `8080` | Railway → Variables |
| `NODE_VERSION` | `18` | Railway → Variables |
| `NODE_TLS_REJECT_UNAUTHORIZED` | `0` | Railway → Variables |
| `GOOGLE_MAPS_API_KEY` | API Key de Google Maps | Google Cloud Console → Credenciales |
| `CLOUDINARY_CLOUD_NAME` | `dupxenpm1` | Cloudinary Dashboard |
| `CLOUDINARY_API_KEY` | API Key de Cloudinary | Cloudinary Dashboard → API Keys |
| `CLOUDINARY_API_SECRET` | API Secret de Cloudinary | Cloudinary Dashboard → API Keys |

> **NOTA:** `NODE_TLS_REJECT_UNAUTHORIZED=0` es necesario para que MongoDB Atlas funcione con el certificado TLS. Es inseguro pero es el workaround actual hasta que se configure correctamente el certificado.

---

## 3. Archivos Modificados y Propósito

### gungry-backend

| Archivo | Cambios |
|---|---|
| `index.js` | Parse Server config: `allowClientClassCreation: true`, `masterKeyIps: ['0.0.0.0/0']`, GridFSBucketAdapter para archivos, CORS con orígenes permitidos, headers CORS para `/api/1/files/*`, middleware Parse Dashboard para autenticación text/plain |
| `cloud/main.js` | Todas las Cloud Functions (ver sección 5) |
| `Dockerfile` | Node 18 slim, variables de entorno Cloudinary |
| `.env.example` | Documentación de variables |
| `package.json` | Dependencias actualizadas |

**Cambios recientes en `cloud/main.js`:**
- `getUsers`: eliminado `.toJSON()` — devuelve Parse Objects nativos para que el SDK del cliente hidrate correctamente `ParseUser` (getters `name`, `username`, `email`, `photo`, `authData`). Agrega filtro por parámetro `type`: `'admin'` → `containedIn(['admin', 'super_admin'])`, `'customer'` → `equalTo('customer')`, sin type → devuelve todos.

### manager-gungry (Admin Portal)

| Archivo | Cambios |
|---|---|
| `src/environments/environment.prod.ts` | `appId: 'gungry-app'`, `serverURL` apuntando a Railway |
| `src/app/modules/admin/category/list/list.component.html` | `imageThumb._url` → `image?._url` |
| `src/app/modules/admin/slider-image/list/list.component.html` | `imageThumb._url` → `image?._url` |
| `src/app/modules/admin/place/list/list.component.html` | `imageThumb._url` → `image?._url` |
| `src/app/modules/admin/post/list/list.component.html` | `imageThumb._url` → `image?._url` |
| `src/app/modules/admin/slide-intro/list/list.component.html` | `imageThumb._url` → `image?._url` |
| `src/app/modules/admin/notification/list/list.component.html` | `imageThumb._url` → `image?._url` |

### gungry-app (App Móvil)

| Archivo | Cambios |
|---|---|
| `src/environments/environment.prod.ts` | `appId: 'gungry-app'`, `serverURL` apuntando a Railway |
| `.browserslistrc` | `Safari >= 14`, `iOS >= 14` (en lugar de "last 2 major versions" que incluía versiones con formato inválido `18.5-18.7`) |
| `netlify.toml` | `npm install && ng build --configuration production`, SPA redirects `/* → /index.html` |
| `package.json` | `@angular-devkit/build-angular: ~13.3.11`, `ngx-bar-rating: ^3.0.0`, `overrides: { esbuild: "0.14.54" }` |
| `tsconfig.json` | `skipLibCheck: true` |
| `src/global.scss` | Comentado `@import "~ngx-bar-rating/themes/br-stars-theme"` (ruta cambia en v3) |
| `src/app/app.component.ts` | Import de `onesignal-cordova-plugin/types/Notification` comentado, reemplazado con `any` |
| `src/types.d.ts` | Nuevo — declare module para `NotificationReceivedEvent` y `OpenedEvent` |
| `src/app/pages/map/map.ts` | `buttonClose` → `bottomClose`, fallback a coordenadas de Guadalajara cuando no hay geolocalización |
| `src/app/components/info-window/info-window.html` | `place.imageThumb?.url()` → `place.image?._url`; categorías ocultas con `*ngIf="false"` (llegan como pointers, no strings) |
| `src/app/components/place-card/place-card.component.ts` | Agregado `@Input() routerLink: any[]` |
| `src/app/components/place-card/place-card.component.html` | `<ion-card button [routerLink]="routerLink">` — el routerLink ahora vive dentro del ion-card para que el click no sea consumido por el host |
| `src/app/pages/favorite-list/favorite-list.html` | routerLink cambiado a ruta absoluta `['/1/home/places/' + place.id]` (favorites se monta bajo `/1/likes` y `/1/profile/likes`, la ruta relativa no resolvía) |

---

## 4. Decisiones de Arquitectura

### Almacenamiento de Archivos
Se usa **GridFSBucketAdapter** (incluido en parse-server) que guarda archivos directamente en MongoDB Atlas. Los archivos se guardan en las colecciones `fs.files` y `fs.chunks`. Es persistente — no se pierde en redeploys de Railway.

**Import correcto:**
```javascript
const { GridFSBucketAdapter } = require('parse-server/lib/Adapters/Files/GridFSBucketAdapter');
```

### Cloud Functions vs Queries Directas
La app móvil usa Cloud Functions para casi todo. El Admin Portal usa una mezcla: queries directas para Category, SliderImage, Post, y Cloud Functions para Place, Review, UserPackage.

### Serialización de Datos (CRÍTICO)
Este es el punto más delicado del proyecto. Hay dos tipos de respuesta:

- **Con `.toJSON()`** — devuelve objeto plano JSON. El campo `id` se convierte en `objectId`. Los archivos se convierten en `{ __type: "File", name, url }`. Se accede a la URL como `objeto._url`.
- **Sin `.toJSON()`** — devuelve Parse Objects nativos. Tienen `.id`, métodos como `.get()`, `.url()` en archivos. Se accede a la URL como `objeto.url()`.

**Regla por componente:**
| Componente | Tipo de objeto esperado |
|---|---|
| App móvil — home (categories, slides) | Parse Objects nativos (sin toJSON) |
| App móvil — map (places) | Parse Objects nativos — `getPlaces` hace `query.find()` sin toJSON; el SDK los hidrata como instancias de `Place` |
| App móvil — place service load() | Array directo (sin wrapper `{results, total}`) |
| App móvil — favorites | Parse Objects nativos — `loadFavorites` usa query directa |
| Manager — listados de categorías, places, etc. | JSON plano (con toJSON), pero necesita `id` explícito |
| Manager — getUsers | Parse Objects nativos — el SDK los hidrata como `ParseUser`; los getters (`name`, `username`, `email`, `photo`) funcionan correctamente |

### Soft Delete
El manager usa soft delete en todos los modelos — NO llama a `.destroy()`, sino que guarda `deletedAt: new Date`. Por eso todas las queries del backend deben incluir:
```javascript
query.doesNotExist('deletedAt');
```

### Status con Mayúscula
Los registros en la BD tienen `status: "Active"` (con mayúscula A), no `"active"`. Los filtros deben usar regex case-insensitive:
```javascript
query.matches('status', /^active$/i);
```

---

## 5. Cloud Functions en cloud/main.js

| Función | Descripción | Notas |
|---|---|---|
| `getAppConfig` | Devuelve `{}` | App lo llama al iniciar |
| `getUsers` | Query a `_User`, devuelve `{ users, total }` | Parse Objects nativos. Acepta `type`: `'admin'` → filtra `['admin','super_admin']`; `'customer'` → filtra `'customer'`; sin type → todos |
| `getCollectionsCount` | Counts de Category, Place, Post | Para dashboard del manager |
| `getPlaces` | Query a `Place`, devuelve array directo | Sin wrapper, Parse Objects nativos (sin toJSON) |
| `getPlacesWithUser` | Query a `Place` con include user | Para manager, sin toJSON |
| `getPlaceWithUser` | Un Place por objectId con include user | Para manager, sin toJSON |
| `getReviewsWithUser` | Query a `Review` con include user y place | Para manager |
| `getReviewWithUser` | Un Review por objectId | Para manager |
| `getUserPackagesWithUser` | Query a `UserPackage` con include user y package | Para manager |
| `getCollections` | Counts de varias clases | Para manager dashboard |
| `getHomePageData` | Devuelve `{ newPlaces, featuredPlaces, nearbyPlaces, categories, slides }` | Sin toJSON — objetos nativos |
| `getPlacesWithUser` | Places con usuario para manager | Sin toJSON |
| `loginInCloud` | Parse.User.logIn(username, password) | Para login en app móvil |
| `signUpInCloud` | Crea usuario nuevo con signUp() | Para registro en app móvil |

---

## 6. CORS — Orígenes Permitidos

En `index.js`, la lista de orígenes CORS es:
```javascript
[
  'https://manager-gungry.netlify.app',
  'https://gungry.netlify.app',
  'https://manager.gungry.com',
  'https://app.gungry.com',
]
```

Los archivos en `/api/1/files/*` tienen headers adicionales:
```javascript
res.header('Access-Control-Allow-Origin', '*');
res.header('Cross-Origin-Resource-Policy', 'cross-origin');
```

---

## 7. DNS y Dominios

| Subdominio | Tipo | Apunta a |
|---|---|---|
| `manager.gungry.com` | NETLIFY (en Netlify DNS) | `manager-gungry.netlify.app` |
| `app.gungry.com` | NETLIFY (en Netlify DNS) | `gungry.netlify.app` |

**IMPORTANTE:** Los registros DNS están en **Netlify DNS** (no en Hostinger). El dominio `gungry.com` usa los nameservers de Netlify. Si se agregan registros en Hostinger para estos subdominios, habrá conflicto. El CNAME de `app` que se creó en Hostinger fue eliminado.

**SSL de `app.gungry.com`:** Pendiente. Netlify reporta que el dominio no parece estar servido por Netlify. Posible causa: conflicto de DNS o propagación incompleta.

---

## 8. Problemas Conocidos Pendientes

1. **SSL `app.gungry.com`** — Netlify no puede renovar el certificado. Verificar que no haya registros conflictivos en Hostinger.

2. **Categorías no aparecen en home (sección principal)** — Las categorías aparecen al hacer "Ver más" pero no en el grid del home. El backend devuelve `categories: []` cuando usa filtro por status. Resuelto parcialmente con regex case-insensitive, pero puede seguir fallando si hay registros con `deletedAt`.

3. ~~**Lista de usuarios en Manager en blanco**~~ — **RESUELTO**: `getUsers` devolvía objetos JSON planos (`.toJSON()`); el manager espera instancias de `ParseUser` con getters de `Parse.User`. Eliminado `.toJSON()` y agregado filtro por `type`. El SDK del cliente ahora hidrata los objetos correctamente.

4. **Login en app móvil** — `loginInCloud` existe pero aún devuelve 400 en algunos casos. Verificar que Railway ya tenga el último deploy con esa función.

5. **Categorías en sección principal del Home** — El HTML en `home.html` itera `*ngFor="let category of categories"` sin `*ngIf`, pero la sección está dentro de `*ngIf="isContentViewVisible"`. Los objetos `category` deben ser Parse Objects nativos con `.id`, `.title` y `.image?.url()`.

---

## 9. Advertencias — NO Tocar Sin Cuidado

- **NO cambiar `allowClientClassCreation`** a `false` sin antes crear todas las clases necesarias en Parse Dashboard. La app móvil necesita crear clases al registrar usuarios.

- **NO cambiar el `.browserslistrc`** de vuelta a `"last 2 Safari major versions"`. iOS 18 tiene versiones con formato `18.5-18.7` que esbuild 0.14.x no puede parsear.

- **NO agregar `.toJSON()` a `getHomePageData`**. La app móvil necesita Parse Objects nativos para acceder a `category.image?.url()` y `category.id`.

- **`getPlaces` no usa `.toJSON()`** — devuelve Parse Objects nativos. El mapa accede a `place.location.latitude` a través del getter `location` que retorna un `Parse.GeoPoint` (que tiene `.latitude` y `.longitude`). No agregar `.toJSON()` porque rompería los getters del modelo `Place` en el cliente.

- **NO cambiar el import de GridFSBucketAdapter**. El import correcto es desde la ruta interna:
  ```javascript
  const { GridFSBucketAdapter } = require('parse-server/lib/Adapters/Files/GridFSBucketAdapter');
  ```

- **NO tocar los registros DNS de Netlify** para `manager.gungry.com` o `app.gungry.com` sin entender que están gestionados desde el panel DNS de la organización Gungry en Netlify, no desde Hostinger.

- **NO usar `npm ci`** en el `netlify.toml` de gungry-app. El `package-lock.json` fue generado en Windows y causa problemas en Linux. Usar `npm install`.

---

## 10. Estructura de Rutas (tabs.router.module.ts)

Todas las rutas viven bajo el prefijo `/1/` (el TabsPage). Contextos principales:

| Tab | Ruta base | place-detail |
|---|---|---|
| Home | `/1/home` | `/1/home/places/:id` |
| Explore | `/1/explore` | `/1/explore/places/:id` |
| Map | `/1/map` | `/1/map/places/:id` |
| Favorites | `/1/likes` o `/1/profile/likes` | No tiene `places/:id` propio — usar ruta absoluta `/1/home/places/:id` |
| Profile | `/1/profile` | `/1/profile/places/:id` |

**Importante:** La ruta `places/:id/:slug` existe en todos los contextos pero el `slug` es opcional. Usar solo `place.id` es más robusto.

### place-card y routerLink

`<app-place-card>` tiene `@Input() routerLink: any[]`. El routerLink se pasa al `<ion-card button [routerLink]="routerLink">` interno. **NO poner `routerLink` como directiva en el host `<app-place-card>`** — `ion-card button` consumiría el click sin propagarlo.

Uso correcto:
```html
<app-place-card [routerLink]="['/1/home/places/' + place.id]" [place]="place">
```

---

## 11. Documentación Externa Consultada

- **Nearme 5 Docs** (https://v5-5-0--nearmeapp.netlify.app): Documentación original del producto. Explica los modelos de datos (Place, Category, SliderImage, Post, etc.), estructura del Admin Portal y configuración de Parse Server. Los campos `imageThumb` mencionados en los templates originalmente esperaban que el backend generara miniaturas automáticamente, funcionalidad que no está implementada — se usa `image` directamente.

- **Parse Server 7 Docs**: `filesAdapter` con GridFSBucketAdapter se importa desde ruta interna. `masterKeyIps` debe incluir `'0.0.0.0/0'` para que el Parse Dashboard pueda autenticarse. `allowClientClassCreation: true` necesario para que la app pueda crear clases nuevas.

- **Angular 13 / Ionic**: `@angular-devkit/build-angular ~13.3.11` es la última versión de Angular 13 que soporta browserslist moderno. `ngx-bar-rating ^3.0.0` es compatible con Angular 13 (v4+ requiere Angular 14+).

---

## 12. Estructura de Modelos Parse (Clases en MongoDB)

| Clase | Campos Clave |
|---|---|
| `Place` | title, description, address, location (GeoPoint), image, images[], categories[], status, isFeatured, priceRange, user, deletedAt |
| `Category` | title, image, status, order, isFeatured, deletedAt |
| `SliderImage` | description, image, isActive, sort, position, type, place, post, category, url |
| `Post` | title, image, place, status, deletedAt |
| `Review` | rating, comment, place, user |
| `UserPackage` | user, package, status |
| `SlideIntro` | image, title, description |
| `Notification` | title, message, image |
| `_User` | name, username, email, photo, authData, type (`'admin'` \| `'super_admin'` \| `'customer'`) |

**Soft Delete:** `Place`, `Category`, `Post` usan `deletedAt` para borrado lógico. Siempre filtrar con `query.doesNotExist('deletedAt')`.

**Status values:** `"Active"` / `"Inactive"` (con mayúscula). Para places: `"Approved"` / `"Pending"`.
