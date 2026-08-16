# API Ecosistema Fauna — FotoFauna + BioQuest

**Última actualización:** 2026-08-16

Backend compartido: contenedor `fauna_api` (FastAPI, puerto **3005**).

| Cliente | Base URL pública |
|---------|------------------|
| FotoFauna | `https://fotofauna.yespi.es/` (misma origin; rutas en raíz) |
| BioQuest | `https://bioquest.yespi.es/ff-api/` (nginx reescribe `/ff-api/*` → backend) |

Documentación orientada a consumo desde BioQuest: [`../webs/bioquest/API.md`](../webs/bioquest/API.md).

---

## Autenticación

### Sesión web (JWT + cookie)

Flujo habitual del navegador:

1. `POST /auth/login` o OAuth Google → respuesta con `access_token` (JWT, ~1 h) + cookie `ff_refresh` (30 días, dominio `.yespi.es`).
2. Peticiones autenticadas: cabecera `Authorization: Bearer <access_token>`.
3. Renovación: `POST /auth/refresh` con cookie (sin body).

SSO: cookie `yespi_access` (multi/hansolo) o `GET /auth/sso-redirect?next=…` para apps `*.yespi.es`.

### Tokens API personales (PAT) — administradores

Desde el panel **FotoFauna → Admin → pestaña «API BQ»** (solo usuarios con `is_admin`):

| Método | Ruta | Descripción |
|--------|------|-------------|
| `GET` | `/auth/api-tokens` | Lista tokens propios (sin secreto) |
| `POST` | `/auth/api-tokens` | Crea token. Body: `{"name": "…", "expires_days": 90}` (`expires_days` opcional, 1–365; omitir = sin caducidad) |
| `DELETE` | `/auth/api-tokens/{id}` | Revoca un token propio |

Respuesta al crear (el campo `token` **solo se muestra una vez**):

```json
{
  "id": 3,
  "name": "Prueba script Atlas",
  "token_prefix": "ff_pat_xYz12",
  "token": "ff_pat_xYz12AbC…",
  "created_at": "2026-08-16T10:00:00+00:00",
  "expires_at": null
}
```

Uso en scripts:

```bash
export FF_PAT="ff_pat_…"
curl -sS -H "Authorization: Bearer $FF_PAT" \
  https://bioquest.yespi.es/ff-api/auth/me
```

- Prefijo fijo: `ff_pat_`.
- Máximo **10** tokens activos por admin.
- Solo usuarios **admin** pueden crear y usar PAT (se revalida `is_admin` en BD en cada petición).
- Revocación inmediata vía panel o `DELETE`.

### Invitados

Algunos endpoints de visión aceptan usuario invitado (`id: 0`) si no hay sesión.

---

## Auth y perfil

| Método | Ruta | Auth | Notas |
|--------|------|------|-------|
| `POST` | `/auth/register` | — | Registro email |
| `POST` | `/auth/login` | — | Login email |
| `POST` | `/auth/logout` | cookie | Cierra sesión |
| `POST` | `/auth/refresh` | cookie | Nuevo access token |
| `GET` | `/auth/me` | sí | Usuario + prefs |
| `PATCH` | `/auth/prefs` | sí | Preferencias (Minka, iNat, filtros IA…) |
| `GET` | `/auth/minka-credentials` | sí | Indicador credenciales Minka |
| `GET` | `/auth/inat-credentials` | sí | Indicador credenciales iNat |

---

## Visión / FotoFauna (`/vision/*`)

| Método | Ruta | Descripción |
|--------|------|-------------|
| `POST` | `/vision/detect` | Localizar fauna (YOLO) |
| `POST` | `/vision/identify` | Identificar especie (iNat + LLM + BioFauna) |
| `POST` | `/vision/dehaze` | Deshaze / partículas |
| `POST` | `/vision/marine-correct` | Corrección marina |
| `GET` | `/vision/health` | Estado del motor |

Subida de archivos: `POST /upload`, listado `GET /list`, ficheros `GET /files/{session}/{name}`.

---

## BioQuest — Academy (`/academy/*`)

Prefijo en BQ: `/ff-api/academy/…`

| Método | Ruta | Descripción |
|--------|------|-------------|
| `GET` | `/academy/timeseries` | Serie temporal por taxón |
| `GET` | `/academy/heatmap-frame` | Heatmap observaciones |
| `GET` | `/academy/species-doc` | Ficha científica (texto IA + metadatos) |
| `GET` | `/academy/species-rich` | Riqueza por zona |
| `GET` | `/academy/species-photos` | Fotos públicas de especie |
| `GET` | `/academy/photo-proxy/{photo_id}/{filename}` | Proxy imagen iNat |
| `GET` | `/academy/thumbs/{taxon_inat_id}/{filename}` | Miniatura |
| `GET` | `/academy/wiki-exists` | Comprueba Wikipedia ES |
| `GET` | `/academy/effort-stats` | Esfuerzo de muestreo |
| `GET` | `/academy/migration-pattern` | Patrones migración |
| `GET` | `/academy/chart-help` | Ayuda contextual gráficos |
| `GET` | `/academy/prefs` · `PUT` · `DELETE` | Preferencias Academy por usuario |
| `POST` | `/academy/feedback` | Feedback de ficha |

Subrouters (mismo prefijo `/academy/`):

- `/conservation/*` — áreas WDPA, mapas conservación
- `/species/*` — catálogo, invasive, IUCN, ranking
- `/tts/*` — text-to-speech fichas
- `/critico/*`, `/amenazadas/*` — listados temáticos

---

## BioQuest — Academy Edu (`/academy/edu/*`)

| Método | Ruta | Descripción |
|--------|------|-------------|
| `GET` | `/academy/edu/courses` | Listado cursos |
| `GET` | `/academy/edu/courses/{id}` | Detalle curso |
| `GET` | `/academy/edu/courses/{id}/day/{n}` | Día del curso |
| `GET` | `/academy/edu/media/search` | Biblioteca multimedia |
| `GET` | `/academy/edu/papers/topics` | Temas papers |

Ver también [`../webs/bioquest/SCHEMA_ACADEMY_EDU.md`](../webs/bioquest/SCHEMA_ACADEMY_EDU.md).

---

## FaunaDex / Medallas

| Prefijo | Descripción |
|---------|-------------|
| `/pokedex/*` | Especies del usuario, observaciones, fotos carrusel |
| `/badges/*` | Medallas por grupo/zona |
| `/trophies/*` | Trofeos exclusivos |
| `/awards/*` | Premios globales calculados |

---

## Explore / Buceo (`/buceo/*`)

Meteo, spots, sitios de buceo, webcams, boyas, grid Open-Meteo. Ejemplo:

```bash
curl -H "Authorization: Bearer $FF_PAT" \
  "https://bioquest.yespi.es/ff-api/buceo/spots?lat=38.3&lng=-0.5"
```

---

## Búsqueda

| Método | Ruta | Descripción |
|--------|------|-------------|
| `GET` | `/search` | Búsqueda unificada taxa/especies |
| `GET` | `/search/ai` | Búsqueda asistida por IA |

---

## Proxies externos

| Prefijo | Destino |
|---------|---------|
| `/proxy/minka/*` | API Minka (obs, publish, species…) |
| `/proxy/inat/*` | API iNaturalist |
| `/proxy/taxa/autocomplete` | Autocompletado taxa |

Requieren credenciales del usuario en prefs (excepto endpoints públicos documentados).

---

## Admin (`/admin/*`)

Solo administradores (JWT o PAT admin):

- `/admin/users`, `/admin/stats`, `/admin/errors`
- `/admin/academy/*` — catálogo, edu sites
- `/admin/autoid/*` — cola Auto-ID Minka

---

## BioFauna (servicio aparte)

Identificador local BioCLIP: contenedor `biofauna-id`, proxy interno `:8090`.

Documentación: [`../biofauna/`](../biofauna/) · endpoint `POST /identify` en [`../webs/fotofauna/handoff/03_API.md`](../webs/fotofauna/handoff/03_API.md).

No pasa por `/ff-api`; lo consume el backend FF vía red Docker.

---

## Códigos HTTP habituales

| Código | Significado |
|--------|-------------|
| `401` | Sin sesión o token inválido/caducado |
| `403` | Autenticado pero sin permiso (p. ej. no admin) |
| `503` | BD no disponible (reintentar) |

BioQuest reintenta automáticamente 502/503/504 (`bq-api-fetch.js`).

---

## OpenAPI

Con el backend en marcha: `GET /docs` (Swagger UI) y `GET /openapi.json` en el puerto 3005 (solo red interna / túnel; no expuesto públicamente en producción).
