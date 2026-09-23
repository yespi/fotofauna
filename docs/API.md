# API Ecosistema Fauna — FotoFauna + BioQuest

**Última actualización:** 2026-09-23

Backend compartido: contenedor `fauna_api` (FastAPI, puerto **3005** en HanSolo).

| Cliente | Base URL pública |
|---------|------------------|
| FotoFauna | `https://fotofauna.yespi.es/` (misma origin; rutas en la raíz) |
| BioQuest | `https://bioquest.yespi.es/ff-api/` (nginx reescribe `/ff-api/*` → backend) |

En BioQuest, antepone **`/ff-api`** a todas las rutas de esta documentación (salvo recursos estáticos de la propia web BQ).

Documentación BioQuest (consumo desde BQ): [`https://github.com/yespi/bioquest/blob/master/docs/API.md`](https://github.com/yespi/bioquest/blob/master/docs/API.md).  
Motor BioFauna (identificación local, token `X-API-Key`): [`https://github.com/yespi/biofauna/blob/master/docs/api.md`](https://github.com/yespi/biofauna/blob/master/docs/api.md).

---

## Índice

1. [Autenticación](#autenticación)
2. [Auth y perfil](#auth-y-perfil)
3. [Visión / FotoFauna](#visión--fotofauna-vision)
4. [BioFauna vía API](#biofauna-vía-api)
5. [Sesión de fotos (upload / list)](#sesión-de-fotos)
6. [BioQuest — Academy](#bioquest--academy-academy)
7. [FaunaDex, medallas y trofeos](#faunadex-medallas-y-trofeos)
8. [Explore / Buceo](#explore--buceo-buceo)
9. [Búsqueda](#búsqueda)
10. [Proxies Minka e iNaturalist](#proxies-externos)
11. [Sincronización y ubicaciones](#sincronización-y-ubicaciones)
12. [Admin y telemetría](#admin-y-telemetría)
13. [Códigos HTTP y límites](#códigos-http-y-límites)
14. [OpenAPI](#openapi)

---

## Autenticación

### Sesión web (JWT + cookie)

Flujo habitual del navegador:

1. `POST /auth/login` (email/contraseña) u OAuth Google → respuesta con `access_token` (JWT, ~1 h) + cookie `ff_refresh` (30 días, dominio `.yespi.es`).
2. Peticiones autenticadas: cabecera `Authorization: Bearer <access_token>`.
3. Renovación: `POST /auth/refresh` con cookie `ff_refresh` (sin body).

**Frontend:** `window.ffFetch` reintenta con refresh ante `401`. Paneles Admin y BioFauna Fotos hacen lo mismo; el padre puede empujar token por `postMessage` (`ff_auth_token` / `ff_request_auth_token`).

**SSO:** cookie `yespi_access` (multi/hansolo) o `GET /auth/sso-redirect?next=…` para apps `*.yespi.es`.

#### Ejemplo: login y perfil

```bash
BASE="https://fotofauna.yespi.es"

# Login (guarda cookies en un jar para refresh)
curl -sS -c /tmp/ff-cookies.txt -X POST "${BASE}/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"usuario@ejemplo.com","password":"…"}' | jq .

export FF_JWT="eyJ…"   # access_token de la respuesta

curl -sS -H "Authorization: Bearer ${FF_JWT}" "${BASE}/auth/me" | jq .
```

Desde BioQuest:

```bash
curl -sS -H "Authorization: Bearer ${FF_JWT}" \
  "https://bioquest.yespi.es/ff-api/auth/me"
```

### Tokens API personales (PAT) — `ff_pat_…`

Para automatizar contra la API de FotoFauna/BioQuest (Academy, Minka proxy, buceo, etc.) sin mantener cookies.

**Quién puede crearlos:** usuarios con `is_admin` **o** `api_token_enabled` (el administrador puede activar el flag por usuario).

**Dónde:** FotoFauna → **Ajustes → API BQ** (o Admin → API BQ). Legacy: `/api-tokens.html` → `/?settings=api`.

| Método | Ruta | Descripción |
|--------|------|-------------|
| `GET` | `/auth/api-tokens` | Lista tokens propios (sin secreto) |
| `POST` | `/auth/api-tokens` | Crea token. Body: `{"name": "…", "expires_days": 90}` (`expires_days` opcional, 1–365; omitir = sin caducidad) |
| `DELETE` | `/auth/api-tokens/{id}` | Revoca un token propio |

Respuesta al crear (el campo `token` **solo se muestra una vez**):

```json
{
  "id": 3,
  "name": "Script Atlas",
  "token_prefix": "ff_pat_xYz12",
  "token": "ff_pat_xYz12AbC…",
  "created_at": "2026-08-16T10:00:00+00:00",
  "expires_at": null
}
```

Uso:

```bash
export FF_PAT="ff_pat_…"

curl -sS -H "Authorization: Bearer $FF_PAT" \
  "https://bioquest.yespi.es/ff-api/academy/timeseries?taxon_id=47120"
```

- Prefijo fijo: `ff_pat_`.
- Máximo **10** tokens activos por usuario.
- Revocación inmediata vía panel o `DELETE`.

### Invitado (solo visión limitada)

`POST /auth/guest-vision` emite un JWT de corta duración para probar filtros/visión sin cuenta. Requiere origen FotoFauna permitido o cabecera `X-FF-Vision-App-Token` (config servidor). Alcance: `vision:guest` — **no** sustituye PAT ni token BioFauna.

Algunos endpoints de imagen usan `current_user_or_guest` (invitado `id: 0`).

### Token BioFauna (`X-API-Key`) — distinto del PAT

`POST /vision/biofauna/identify` acepta:

- `X-API-Key: <secreto>` emitido por el administrador (solicitud por correo), **o**
- Sesión FotoFauna (`Bearer` JWT de usuario real, no invitado).

Ver [`https://github.com/yespi/biofauna/blob/master/docs/api.md`](https://github.com/yespi/biofauna/blob/master/docs/api.md) y Ayuda en la app (sección «API BioFauna»).

---

## Auth y perfil

| Método | Ruta | Auth | Notas |
|--------|------|------|-------|
| `POST` | `/auth/register` | — | Registro email |
| `POST` | `/auth/login` | — | Login email |
| `POST` | `/auth/logout` | cookie | Cierra sesión |
| `POST` | `/auth/refresh` | cookie | Nuevo access token |
| `POST` | `/auth/guest-vision` | origen/app token | JWT invitado visión |
| `POST` | `/auth/forgot-password` | — | |
| `POST` | `/auth/reset-password` | — | |
| `GET` | `/auth/me` | sí | Usuario + prefs |
| `PATCH` | `/auth/prefs` | sí | Minka, iNat, filtros IA, GPS por defecto, etc. |
| `GET` | `/auth/minka-credentials` | sí | Indicador credenciales Minka |
| `GET` | `/auth/inat-credentials` | sí | Indicador credenciales iNat |
| `GET` | `/auth/google/login` | — | OAuth (redirect) |
| `GET` | `/auth/google/callback` | — | OAuth callback |

---

## Visión / FotoFauna (`/vision/*`)

Límite de subida habitual: **30 MB** por archivo.  
Rate limit en locate/identify/detect: variable `VISION_RATE_LIMIT_RPM` (default **600** peticiones/minuto por usuario).

| Método | Ruta | Auth | Descripción |
|--------|------|------|-------------|
| `POST` | `/vision/locate` | sesión | Detección YOLO para recorte (`organisms[]`, `bbox`, `exif`) |
| `POST` | `/vision/identify` | sesión | Identificación sobre imagen (BioFauna local) |
| `POST` | `/vision/detect` | sesión | Pipeline completo: detectar + identificar organismos |
| `POST` | `/vision/inat-score` | sesión | BioFauna primero; fallback iNat CV |
| `POST` | `/vision/biofauna/identify` | `X-API-Key` o sesión | Solo motor BioFauna (proxy a `:8090`) |
| `GET` | `/vision/health` | — | Estado del motor de visión |
| `GET` | `/vision/local-taxon-thumb/{slug}` | — | Miniatura local de especie BF |
| `POST` | `/vision/session/export` | sesión | Exportar sesión de trabajo |
| `POST` | `/vision/session/cleanup` | sesión | Limpiar temporales de sesión |
| `DELETE` | `/vision/admin/purge-all` | admin | Purga global (operación) |

### Filtros y mejora de imagen (`/vision/…`)

Requieren sesión o invitado según endpoint (`image_router`, prefijo `/vision`):

| Método | Ruta | Descripción |
|--------|------|-------------|
| `POST` | `/vision/detect-environment` | Clasificación entorno (marino/terrestre, etc.) |
| `POST` | `/vision/auto-enhance` | Mejora automática |
| `POST` | `/vision/dehaze` | Deshaze / partículas |
| `POST` | `/vision/dehaze/score` | Puntuación de bruma |
| `POST` | `/vision/dehaze-score` | Alias puntuación |
| `POST` | `/vision/marine-correct` | Corrección color marina |
| `POST` | `/vision/deblur` | Desenfoque |
| `POST` | `/vision/sharpen` | Nitidez |
| `POST` | `/vision/remove-particles` | Motas |
| `POST` | `/vision/remove-particles-stamp` | Motas con sello |
| `POST` | `/vision/reduce-reds` | Compensar rojos submarinos |

### Ejemplo: localizar y identificar

```bash
BASE="https://fotofauna.yespi.es"
HDR="Authorization: Bearer ${FF_JWT}"

# 1) Localizar fauna (YOLO)
curl -sS -X POST "${BASE}/vision/locate" -H "${HDR}" \
  -F "file=@foto.jpg" | jq '.organisms[0]'

# 2) Identificar (recorte o imagen completa)
curl -sS -X POST "${BASE}/vision/identify" -H "${HDR}" \
  -F "file=@recorte.jpg" \
  -F "lat=38.345" -F "lng=-0.412" -F "observed_on=2026-09-20" | jq .
```

### Ejemplo: pipeline `detect`

```bash
curl -sS -X POST "${BASE}/vision/detect" -H "${HDR}" \
  -F "file=@foto.jpg" \
  -F "lat=38.3" -F "lng=-0.5" \
  -F "observed_on=2026-09-20" | jq '.organisms'
```

Respuesta típica de `identify` / `detect` (especie):

```json
{
  "species": {
    "name": "Octopus vulgaris",
    "common_name": "Pulpo común",
    "taxon_id": 47120,
    "confidence": 0.82,
    "source": "biofauna"
  },
  "source": "biofauna",
  "gps_default": false
}
```

---

## BioFauna vía API

| Aspecto | Detalle |
|---------|---------|
| URL pública | `POST https://fotofauna.yespi.es/vision/biofauna/identify` |
| Auth | `X-API-Key` (solicitar a [gustavo.zafra@gmail.com](mailto:gustavo.zafra@gmail.com)) o JWT sesión FF |
| Cuerpo | `multipart/form-data`, campo `file` |
| Query | `topk` (default 5) |

```bash
curl -sS -X POST "https://fotofauna.yespi.es/vision/biofauna/identify?topk=5" \
  -H "X-API-Key: ${BF_API_KEY}" \
  -F "file=@foto.jpg" | jq '.prediction'
```

Contrato JSON completo (`p_species`, `rank`, `threshold`, …): [`https://github.com/yespi/biofauna/blob/master/docs/api.md`](https://github.com/yespi/biofauna/blob/master/docs/api.md).

El servicio interno en `:8090` **no** está expuesto; lo usa `fauna_api` por red Docker.

---

## Sesión de fotos

| Método | Ruta | Auth | Descripción |
|--------|------|------|-------------|
| `POST` | `/upload` | sesión | Subir imagen a sesión temporal |
| `GET` | `/list` | sesión | Listar ficheros de sesión |
| `GET` | `/files/{filename}` | sesión | Descargar fichero temporal |

---

## BioQuest — Academy (`/academy/*`)

Prefijo en BQ: `/ff-api/academy/…`

| Método | Ruta | Descripción |
|--------|------|-------------|
| `GET` | `/academy/timeseries` | Serie temporal por taxón |
| `GET` | `/academy/timeseries-group` | Serie agrupada |
| `GET` | `/academy/multi-species-trend` | Tendencia multi-especie |
| `GET` | `/academy/source-trend` | Tendencia por fuente |
| `GET` | `/academy/heatmap-frame` | Heatmap observaciones |
| `GET` | `/academy/species-doc` | Ficha científica (IA + metadatos) |
| `GET` | `/academy/species-rich` | Riqueza por zona |
| `GET` | `/academy/species-photos` | Fotos públicas de especie |
| `GET` | `/academy/photo-proxy/{photo_id}/{filename}` | Proxy imagen iNat |
| `GET` | `/academy/thumbs/{taxon_inat_id}/{filename}` | Miniatura |
| `GET` | `/academy/wiki-exists` | Comprueba Wikipedia ES |
| `GET` | `/academy/effort-stats` | Esfuerzo de muestreo |
| `GET` | `/academy/migration-pattern` | Patrones migración |
| `GET` | `/academy/chart-help` | Ayuda contextual gráficos |
| `GET` | `/academy/prefs` · `PUT` · `DELETE` | Preferencias Academy |
| `POST` | `/academy/feedback` | Feedback de ficha |

Subrouters bajo `/academy/`:

- `/conservation/*` — WDPA, mapas conservación  
- `/species/*` — catálogo, invasive, IUCN, ranking  
- `/tts/*` — text-to-speech fichas  
- `/critico/*`, `/amenazadas/*` — listados temáticos  

### Ejemplo Academy

```bash
curl -sS -H "Authorization: Bearer $FF_PAT" \
  "https://bioquest.yespi.es/ff-api/academy/species-doc?taxon_id=47120&lang=es" | jq '.title'
```

### Academy Edu (`/academy/edu/*`)

| Método | Ruta | Descripción |
|--------|------|-------------|
| `GET` | `/academy/edu/courses` | Listado cursos |
| `GET` | `/academy/edu/courses/{id}` | Detalle curso |
| `GET` | `/academy/edu/courses/{id}/day/{n}` | Día del curso |
| `GET` | `/academy/edu/media/search` | Biblioteca multimedia |
| `GET` | `/academy/edu/papers/topics` | Temas papers |

Ver documentación Academy Edu en el repositorio BioQuest.

---

## FaunaDex, medallas y trofeos

| Prefijo | Descripción |
|---------|-------------|
| `/pokedex/*` | Especies del usuario, observaciones, carrusel fotos |
| `/badges/*` | Medallas por grupo/zona |
| `/trophies/*` | Trofeos exclusivos |
| `/awards/*` | Premios globales calculados |

Ejemplo:

```bash
curl -sS -H "Authorization: Bearer $FF_JWT" \
  "https://fotofauna.yespi.es/pokedex/species/mine?limit=20"
```

---

## Explore / Buceo (`/buceo/*`)

Meteo, spots, sitios de buceo, webcams, boyas, grid Open-Meteo.

```bash
curl -sS -H "Authorization: Bearer $FF_PAT" \
  "https://bioquest.yespi.es/ff-api/buceo/spots?lat=38.3&lng=-0.5" | jq '.spots[:3]'

curl -sS -H "Authorization: Bearer $FF_PAT" \
  "https://bioquest.yespi.es/ff-api/buceo/forecast?lat=38.3&lng=-0.5"
```

Rutas principales: `/buceo/weather`, `/weather-batch`, `/forecast`, `/spots`, `/spot/{id}`, `/dive-sites`, `/search`, `/wind-arrows`, `/meteo-grid`, `/buoys`, `/webcams`.

---

## Búsqueda

| Método | Ruta | Descripción |
|--------|------|-------------|
| `GET` | `/search` | Búsqueda unificada taxa/especies |
| `GET` | `/search/ai` | Búsqueda asistida por IA |

```bash
curl -sS "https://fotofauna.yespi.es/search?q=octopus&limit=5" \
  -H "Authorization: Bearer $FF_JWT"
```

---

## Proxies externos

Requieren credenciales del usuario en prefs (salvo endpoints públicos documentados en iNat).

| Prefijo | Destino |
|---------|---------|
| `/proxy/minka/*` | API Minka (obs, publish, species…) |
| `/proxy/inat/*` | API iNaturalist (incl. `POST /proxy/inat/publish`) |
| `/proxy/taxa/autocomplete` | Autocompletado taxa |
| `/proxy/taxa/photos/{taxon_id}` | Fotos de taxón |

También montados bajo `/vision/proxy/…` (mismos handlers).

### Minka (ejemplos)

| Método | Ruta | Uso |
|--------|------|-----|
| `POST` | `/proxy/minka/validate` | Validar obs antes de publicar |
| `POST` | `/proxy/minka/publish` | Publicar observación |
| `GET` | `/proxy/minka/species` | Buscar especies |
| `GET` | `/proxy/minka/obs` | Listar observaciones |

```bash
curl -sS -X POST "https://fotofauna.yespi.es/proxy/minka/validate" \
  -H "Authorization: Bearer $FF_JWT" \
  -H "Content-Type: application/json" \
  -d '{"observations":[…]}'
```

### iNaturalist

| Método | Ruta | Notas |
|--------|------|-------|
| `POST` | `/proxy/inat/publish` | Sin `taxon_id` → `needs_id` |
| `GET` | `/proxy/inat/observations` | Listado |
| `GET` | `/proxy/inat/species_counts` | Conteos |

---

## Sincronización y ubicaciones

| Método | Ruta | Descripción |
|--------|------|-------------|
| `POST` | `/sync/me` | Sincronizar prefs/sesión usuario |
| `POST` | `/sync/full/me` | Sincronización completa |
| `GET` | `/sync/status/me` | Estado sync |
| `GET` | `/api/location-points` | Puntos guardados |
| `POST` | `/api/location-points` | Crear punto |
| `DELETE` | `/api/location-points/{id}` | Borrar punto |

(También bajo `/vision/api/location-points`.)

---

## Admin y telemetría

Prefijo `/admin/*` — solo administradores (JWT o PAT con `is_admin`).

| Área | Rutas |
|------|--------|
| Usuarios | `/admin/users`, `PATCH …/role`, `…/autoid`, `…/api-tokens-perm` |
| Estadísticas | `/admin/stats`, `/admin/pipeline-stats`, `/admin/errors` |
| Academy admin | `/admin/academy/catalog`, `/admin/academy/edu/sites` |
| Auto-ID Minka | `/admin/autoid/*` (cola, analyze, submit, schedules, …) |
| BioFauna Fotos | `/admin/biofauna/species`, `/admin/biofauna/export/*` |
| Uso cliente | `POST /usage/ping`, `/usage/session-start`, `/usage/client-log`, `/usage/error` |

Descarga masiva BF: ver [`BIOFAUNA_FOTOS.md`](BIOFAUNA_FOTOS.md).

---

## Códigos HTTP y límites

| Código | Significado |
|--------|-------------|
| `401` | Sin sesión o token inválido/caducado |
| `403` | Autenticado pero sin permiso (p. ej. no admin, PAT no habilitado) |
| `413` | Cuerpo demasiado grande |
| `429` | Rate limit visión (`VISION_RATE_LIMIT_RPM`) |
| `503` | BD no disponible o motor IA caído |

BioQuest reintenta automáticamente `502`/`503`/`504` (`bq-api-fetch.js`).

---

## OpenAPI

Con el backend en marcha (red interna / túnel): `GET /docs` (Swagger UI) y `GET /openapi.json` en el puerto 3005. No se expone públicamente en producción; esta guía y el código en `ecosistema-fauna/backend/` son la referencia estable.

---

## Changelog de esta guía

- **2026-09-23:** Documentación ampliada (ejemplos curl/Python, BioFauna `X-API-Key`, PAT `api_token_enabled`, rate limit, rutas visión/filtros). Publicado en repos `hansolo-docs`, `fotofauna`, `biofauna`.
