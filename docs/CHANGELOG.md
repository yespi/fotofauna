# FotoFauna — Changelog (rolling)

## 2026-10-06 — FotoFauna: PRE → PRO (orden de Gustavo)
- `deploy-to-pro.sh` ejecutado (rsync `public-pre/` → `public/`, bump de caché, parche de rutas `/pre/`→`/`, SEO: 401 páginas de especie + 6 temas + sitemap). Smoke PRO: OK (18 comprobaciones). Backup previo en `/mnt/docker/backups/fotofauna/` (rotación 3). Backend compartido sin reinicio (`fauna_api` se recrea solo a diario 00:30/01:00).
- Revertir: `unzip -o <último backup_fotofauna_*.zip> -d /mnt/docker/ecosistema-fauna/webapp/public/`.

## 2026-10-06 — AutoID / BioFauna (documentación)
- Regla de hermanas de AutoID retirada (4-oct); rescate 2 activo; recorte en pausa; género p≥0,95; `fauna_api` se recrea a las 00:30 y 01:00 (renew-inat-token).
- Índice BioFauna: promociones del 5-oct (combo limpio) y 6-oct (lotes 01+02); OOS panel 82,85 %.
- Sin cambios de FotoFauna en PRO; FF sigue en PRE a la espera de «súbelo a PRO».


## 2026-10-04 10:00 — FotoFauna
**Build PRE:** `13eff6ba` · **Origen:** sync automático (Chewie)

- Cambios en working tree pendientes de commit (sync automático).


## 2026-10-03 21:00 — FotoFauna
**Build PRE:** `c17252a9` · **Origen:** sync automático (Chewie)

- Cambios en working tree pendientes de commit (sync automático).


## 2026-10-03 20:30 — FotoFauna
**Build PRE:** `746db163` · **Origen:** sync automático (Chewie)

- Cambios en working tree pendientes de commit (sync automático).


## 2026-10-03 20:00 — FotoFauna
**Build PRE:** `9d179ce8` · **Origen:** sync automático (Chewie)

- Cambios en working tree pendientes de commit (sync automático).


## 2026-10-03 19:30 — FotoFauna
**Build PRE:** `811ae400` · **Origen:** sync automático (Chewie)

- Cambios en working tree pendientes de commit (sync automático).


## 2026-10-03 — FotoFauna: publicar no modifica la sesión, deshacer/cancelar subida, identificación con foto completa, log de sesión, reinicio rápido

**Código:** `ecosistema-fauna` (`webapp/public-pre` → `deploy-to-pro.sh`, backups en `/mnt/new/backups/ff_dup_aviso_20261003/` y `/mnt/docker/backups/fotofauna/backup_fotofauna_20261003_*.zip`) · `fauna_api` · `docker-compose.yml`.

**Incidente (21:41 CEST):** tras recargar la sesión y pulsar *Subir*, se publicaron 13 observaciones en Minka (883199–883211); varias llevaban una especie que el usuario había dejado sin identificar. Causa: `resolvePhotoSpeciesForPublish` (desde jun-2026) publicaba la sugerencia **pendiente** (sin confirmar) con cualquier confianza ≥ 15 %; mientras el servidor descartaba sugerencias < 0,83 el fallo no se veía, y al restaurar el piso 0,15 (`vision_routes.py`) afloró. Despublicadas por erróneas: 883201, 883204, 883207, 883208 (las demás se dejaron; 883202 y 883209 dudosas). Regla de Gustavo resultante: **publicar sube exactamente lo que hay en pantalla**.

- **Publicar no modifica la sesión** (`group-species.js`): ya no se usan sugerencias pendientes y `groupSpeciesPatchForPublish` ya no unifica ni cambia la especie de las fotos del grupo al publicar/simular (solo se guarda el estado de publicación). Sin especie → se publica sin especie.
- **Botones «Cancelar la subida» y «Deshacer lo subido (N)»** (`use-upload-platforms.js`, `PhotoGrid.js`): el primero detiene todo el proceso; el segundo borra de Minka y/o iNaturalist las observaciones subidas desde la sesión, con confirmación que dice cuántas se borrarán en cada plataforma. Backend nuevo: `POST /proxy/minka/unpublish` y `POST /proxy/inat/unpublish` (`routes_minka.py`, `routes_inat.py`): solo borran observaciones del propio usuario (verifican el login), purgan la caché de publicación (6 h) y el historial local para poder volver a subir. Borrado real en iNaturalist: sin probar de punta a punta.
- **Identificación con foto completa** (`use-vision-pipeline.js`): se identifica la foto completa salvo que el usuario haya recortado **a mano** esa foto. Antes se mandaba siempre `_locatedBbox` (recorte por atención de BF que activa `FAUNA_LOCATE_ATTN=1`, heredado por las copias) con prioridad sobre el recorte manual: las copias recortadas a otro organismo repetían la especie de la primera. Medido (120 fotos del eval + 31 de *Corallium rubrum*): recorte por atención 69,7 % vs foto completa 80,7 % (16 aciertos perdidos, 3 ganados). El servidor ya prueba recortes por su cuenta (rescate `crop_mix`).
- **Recorte automático por atención: solo se dibuja (decisión de Gustavo vía Robotin, 3-oct 22:48):** el bbox de `FAUNA_LOCATE_ATTN=1` (`edit_params.auto_cropped`) no puntúa al identificar (`identify-crop.js`; también `detectPhoto` → `/vision/detect`), ni se aplica al exportar/publicar (`use-photo-actions.js` `_renderExportBlob`, `MobileApp.js` `_getPhotoBlob`); solo cuenta el recorte manual. Probado con node (6 casos) y sobre lo servido en PRO.
- **Panel Administración de BioFauna:** tarjeta nueva con la fecha de la última medida/calibración, aviso de que el acierto (82,46 % / Tier 1 77,5 %) no cambió y la lista de cambios recientes (`_BF_CAMBIOS` en `admin/index.html`). El panel no leía datos viejos (`biofauna-stats.json` se regenera a diario); parecía mudo porque solo pintaba KPIs.
- **Log de sesión:** la réplica al servidor (`POST /usage/client-log`) no funcionaba en PRO (`API_BASE` vacío descartaba el envío en `use-logger.js`); corregido (`logs en /fauna-tmp/client-logs/user_<id>.log`). `/mnt/scripts/fauna/cleanup-fauna-tmp.sh` ya no borra `client-logs`.
- **Reinicio rápido de `fauna_api`:** el arranque (`docker-compose.yml`, servicio `api`) ya no reinstala libs de sistema ni dependencias e2e en cada restart (`dpkg -s` / flag `/usr/local/.e2e-deps-ready.flag`): ~4 s en vez de ~30 s de 502. Backup `docker-compose.yml.bak_pre_fast_restart`.
- **Miniaturas de especie:** generadas las 2.889 que faltaban (`scripts/make_missing_species_thumbs_20261003.py`, 75×75 desde la galería; antes 404 en el panel Identificar).
- **Aviso de validación duplicado (EXIF) — corregido (3-oct, noche):** si Minka e iNat fallan la misma validación se muestra un único aviso (`_mergeValidationParts`); si el detalle difiere, los dos como antes.
- **Autosave de sesión (3-oct, noche):** `flushAutosaveSync` (recargar/cerrar) escribe primero en IndexedDB; antes `localStorage.setItem` superaba la cuota (sesión de 5,58 MB), lanzaba la excepción y nunca llegaba a IDB, perdiendo las últimas ediciones al recargar. Backup `use-session.js.bak_pre_quota`.
- **AutoID (misma fecha):** publicación a **género** con `p_genus` ≥ 0,95 (tope 50/día, primeras 30 a revisión en `logs/autoid_genus_p95_review.jsonl`; familia OFF; `AUTOID_GENUS_P95=0` desactiva); `identify` expone `p_genus`/`p_family` en sombra (`BF_HIER_SHADOW`). Ver [`AUTOID_PIPELINE.md`](AUTOID_PIPELINE.md).

## 2026-10-01 — AutoID: fallback por recortes, formato nuevo de Minka, kNN en GPU, galería en archive, IA local

**Código:** `hansolo-dockers` main · `fauna_api` + `biofauna-id.service` · detalle en [`../../biofauna/INFORME_SESION_20260929.md`](../../biofauna/INFORME_SESION_20260929.md) §16–§17.

- **Fallback por recortes (AutoID y FF):** si la imagen completa no da ID publicable, `identify_service` prueba 4 cuadrantes, centro 17 %, saliencia y centro 40 %; rescata con confianza ≥ 0,93 y acuerdo de ≥ 2 pasos (guardia de género distinto ⇒ ≥ 3). 500 fotos: 35 → 48 publicables; lo añadido acierta 85,7 %. AutoID marca `BioFauna+recorte` (`autoid_history.source`), tope `AUTOID_CROP_MAX_DAY=30` y registra `rescue` en `logs/shadow_autoid.jsonl`. Apagar: borrar `crop_mix.conf` + reiniciar `biofauna-id`.
- **Fix publicación manual (503 "Servidor no disponible"):** Minka devuelve ahora `/observations/{id}` como `{"results":[obs]}`; `_get_observation_owner_login` devolvía "dueño desconocido" y la política fail-closed bloqueaba. `_unwrap_obs` acepta ambos formatos (4 puntos de `routes_minka_autoid.py`; evita además duplicar IDs en `_has_our_identification`).
- **Rendimiento:** búsqueda kNN en GPU (`BF_GPU_SEARCH=1`): `/identify` ≈ 0,2 s/llamada (antes 0,7 s). Geo prior vectorizado.
- **Galería:** `/mnt/gpu/fotofauna-images` → enlace a `/mnt/archive/gpu_migrated_20260930/fotofauna-images`; `fauna_api` reiniciado y probado. Descargas de BioFauna Fotos sin cambios.
- **Panel FF Administración:** estadísticas regeneradas (1-oct) y dos tarjetas nuevas: *Baseline comparable* (81,18 %) y *AutoID real (publicado)* (96,6 %); `gen_stats.py` escribe `kpi_referencia`.
- **Credenciales:** 16 ficheros dejaron de llevar la clave antigua de BD (usan `FAUNA_DB_DSN`/`TESLAMATE_DSN`); el bind mount `./backend:/app` hace que el cambio persista sin reconstruir imagen.
- **IA local:** Academy usa `qwen3:4b-instruct` vía Ollama (Groq retirado). Ver [`../../sistema/IA_LOCAL_OLLAMA_20261001.md`](../../sistema/IA_LOCAL_OLLAMA_20261001.md).

## 2026-09-29 — AutoID publicación: token Minka de sesión web + calibración + 2 hilos

**Código:** `hansolo-dockers` main (`c271a8d3f` + sync) · `fauna_api` + `biofauna-id.service`

- **🔴 Fix crítico publicación AutoID**: el JWT de `.api-keys` (MINKA_API_TOKEN) solo LEE (POST → 401).
  El token que publica es el `api_token` de la sesión web (`observe.minka-sdg.org/login` → `/session` →
  `users/api_token`). `_minka_login` ahora hace login web primero; JWT de .api-keys como fallback de lectura.
  TTL de caché 3.600 → 3.000 s. Verificado end-to-end (obs 881616 publicada).
- **🔴 Fix calibración jerárquica** (causa de 0 publicaciones): `calibration_hierarchical.json` con
  thresholds de relleno 0,895 (SHRINK_K=30) bloqueaba el rango 0,83-0,895. SHRINK_K → 1000 en
  `fit_calib_hierarchical.py` → thresholds ~0,83. Reiniciado `biofauna-id.service`.
- **AutoID 2 hilos**: `autoid_wave.py` con `asyncio.gather` WORKERS=2 → ritmo ~54-63 obs/min (4-5×).
- **Minka-API_test**: token PAT `ff_pat_...` creado por Gustavo para el equipo de Minka; verificado
  (`POST /vision/biofauna/identify` → Cratena 0,94 / Turdus 0,998). Fix `vision_routes.py` (import
  `BIOFAUNA_URL` estaba tras un `raise` → 503; movido).
- **YOLO retirado de AutoID** (modelos COCO no detectan fauna); attention crop evaluado (−5 pp) → default
  OFF (`FAUNA_LOCATE_ATTN=0`), endpoint `/locate` disponible.
- **Borrado físico de identificaciones Minka**: `DELETE observe/identifications/{id}?delete=true` + meta
  CSRF (la API solo retira). Doc: `MINKA_BORRADO_FISICO_20260929.md`.

## 2026-09-23 19:30 — FotoFauna
**Build PRE:** `4144f67d` · **Origen:** sync automático (Chewie)

- Cambios en working tree pendientes de commit (sync automático).


## 2026-09-23 08:00 — FotoFauna
**Build PRE:** `abd2ff25` · **Origen:** sync automático (Chewie)

- Cambios en working tree pendientes de commit (sync automático).


## 2026-09-23 07:30 — FotoFauna
**Build PRE:** `6bbe9a07` · **Origen:** sync automático (Chewie)

- Cambios en working tree pendientes de commit (sync automático).


## 2026-09-21 — Auth BF-Fotos: renovar JWT ante 401

**Código:** `hansolo-dockers` PR #40 (`cursor/ff-inat-publish-a866`) · PRE+PRO en HanSolo

- El iframe `#bf-fotos` (`biofauna_download.html`) usaba `fetch` sin el refresh-on-401 del Admin → «Token inválido o expirado» con la sesión del padre aún visible.
- Ante 401: pide token fresco al padre (`ff_request_auth_token`) y/o `POST /auth/refresh` (cookie `ff_refresh`) y reintenta.
- El padre refresca el access JWT antes del `postMessage` al cargar el iframe; sincroniza si el hijo renueva (`ff_auth_token_updated`).
- Cache: `PhotoGrid.js?v=auth-refresh-1`, iframe `v=20260921t`.

## 2026-09-21 — iNat: publicar sin especie (regresión 17-sep)

**Código:** mismo PR #40 · `routes_inat.py` + `use-upload-platforms.js`

- Minka e iNat admiten observaciones `needs_id` sin taxón. El 17-sep ya se había quitado el bloqueo; el restore `d62ef6efc` lo reintrodujo en cliente y backend.
- Cliente: no añade «falta especie» al validar; log de aviso si el lote va sin ID.
- Backend: sin `taxon_id` ni `species_guess` → crea obs `needs_id` (no HTTP 400).
- Dual Minka+iNat: el error de iNat ya no lo tapa el OK de Minka.
- Payload: taxón de confirmada → pending usable → organismos (`resolvePhotoSpeciesForPublish`).

## 2026-09-21 — Arrastrar foto sobre otra: agrupado fiable

- El drop se perdía si `dragend` iba antes que `drop`, si se arrastraba un hijo visible del grupo, o si el `img` nativo robaba el drag.
- PRE y PRO: siempre se agrupa el master; el id sobrevive al `dragend`; miniatura no es arrastrable.
- Rama aparte: `cursor/ff-group-drop-a866` (no mezclar con BioFauna en esa rama).

## 2026-09-21 — Identificar: flechas ya no saltan de 2 en 2

- Tras seleccionar fotos, el foco queda en la rejilla: ←/→ se manejaban dos veces (grid `@keydown` + `window.keydown`) y el panel Identificar avanzaba de dos en dos.
- PRE y PRO.

## 2026-09-21 — Filtro Sobreexp. = inverso de Subexp.

- El ☀ Sobreexp. ya no recorta luces a medias y mezcla de vuelta al original (por eso casi no se notaba).
- Misma lógica que 🌙 Subexp. invertida: gain hacia gris medio 0.42 y compresión de luces (LIME + máscara). Fotos oscuras/normales siguen casi sin cambio.
- El blanco recortado (255) no tiene detalle que recuperar.
- Test: `backend/tests/test_enhance_overexp.py`.

## 2026-09-20 — BioFauna Fotos en FF móvil

- Barra inferior móvil: 📦 **BF Fotos** (solo `is_admin`), abre `biofauna_download.html` en pestaña nueva con `auth_token` (mismo patrón que Auto-ID).
- Menú de usuario móvil: **BioFauna Fotos ↗**. Escritorio sin cambio (sigue el rail `#bf-fotos`).

## 2026-09-19 — BioFauna Fotos (cierre): árbol, ZIP plano/árbol, merge renombrado

**Backend:** `fauna_api` bind-mount · **Git:** `hansolo-dockers` · **Doc:** [`BIOFAUNA_FOTOS.md`](BIOFAUNA_FOTOS.md)

- Árbol de descarga con `iconic`/familia. Selector: árbol taxonómico o solo carpetas de especie.
- Merge: `DOBLE_CLIC_Unir_CSVs.bat` + `NO_ABRIR_motor_unir_CSVs.ps1`.
- Rail 📦 `#bf-fotos`, lupa, splitter `bf_split_pct`, KPI `biofauna_exports`.

## 2026-09-19 — BioFauna Fotos (descarga masiva admins) en PRE y PRO

- Rail 📦 bajo AutoID (`#bf-fotos`). ZIP ≤2 GB + CSV + README; Admin KPI `biofauna_exports`.

## 2026-09-18 — FF quick wins (locate + thumbs) + AutoID excluded

**Doc:** [`AUTOID_PIPELINE.md`](AUTOID_PIPELINE.md), [`FF_CROP_BF_GUIDED.md`](FF_CROP_BF_GUIDED.md)

- **`/vision/locate`:** solo YOLO26n; YOLO vacío o conf &lt;0,25 → full-frame a BF.
- **Thumbs taxón:** local-first; iNat live off por defecto.
- **AutoID:** fail-closed para observadores `excluded`.
- **Borde:** cutover HAProxy delante de NPM.

---

## 2026-09-01 — Identificar y AutoID: candado GPS + iNat + geo
**Backend:** bind-mount PRO/PRE · **Git:** `hansolo-dockers` rama `cursor/identify-safety-gates-aef2` · **Doc:** [`AUTOID_PIPELINE.md`](AUTOID_PIPELINE.md)

- Sin GPS no hay especie (Identificar y AutoID).
- BioFauna propone; iNaturalist CV corrobora el binomial (fail-closed si discrepa o no responde).
- Si la especie tiene `geo_priors` y el punto está a >500 km, se rechaza.
- PRE: toasts de bloqueo; changelog `20260901e`.

---

## 2026-08-27 22:20 — Filtro Marina: menos rojo (gray-world ×0,82)
**Deploy:** `docker restart fauna_api` · **Git:** `hansolo-dockers` `ae9450d7d` · **Público:** `yespi/fotofauna`

- **`_marine_correct`:** boost del canal rojo `** (strength × 0,82)` en lugar de `** strength`; G/B, CLAHE y slider intactos.
- Paper (EN/ES) §3.4 y docs de filtros sincronizados al repo público.

---

## 2026-08-16 — Auto-ID: pipeline BF+recorte, guardia, email alertas

**Backend:** bind-mount PRO/PRE · **Git:** `hansolo-dockers` (`505caf4ec` + rama `cursor/autoid-guard-email-4435`) · **Doc:** [`AUTOID_PIPELINE.md`](AUTOID_PIPELINE.md)

### Pipeline oleada (`routes_minka_autoid.py`)
- **Recorte previo** a identificar: YOLO26n → fallback bbox IA (`AUTOID_LOCATE_AI=1`).
- **BioFauna primero** sobre el recorte; publicación **sin iNat** si `p_species ≥ 0.75` y `WAVE_BIOFAUNA_REQUIRE_INAT=0`.
- Historial ampliado a **500** entradas FIFO.

### Guardia horaria (`autoid_guard.py`)
- Circuit breaker: analiza `autoid_history` (1h/24h); trip desactiva planificaciones.
- Umbrales **dinámicos** desde `autoid_schedules` (+25% headroom); subida masiva = warning, no falso positivo.
- API `GET/POST /admin/autoid/guard`; submit bloqueado con 503 si tripped.
- Tarea planificador `autoid_watchdog` (priority 50) + cron host `:55`.

### Alertas e investigación
- Email SMTP al admin al trip (métricas, alertas, enlace panel).
- Registro automático en `TAREAS_PENDIENTES.md` (checklist Cursor).
- Volumen docker-compose: `TAREAS_PENDIENTES.md` rw en `fauna_api`.
- **Sin** Cloud Agent automático (decisión 2026-08-16).

### Métricas iniciales (100 pub, 15–16 ago)
- Fuentes: 57% iNat CV · 23% BF+iNat · 19% BF solo · 1% Minka CV.
- Confianza media **92,6%** (68≥90%, 26 en 80–89%, 6 &lt;80%).
- Tras deploy recorte+BF (~14:55): oleada aún con pocos datos post-cambio; monitorizar próximas horas.

---

## 2026-08-16 — Panel Ajustes full-screen, tokens PAT, Admin sidebar
**Build PRO/PRE:** `3a767a18` · **Git:** `hansolo-dockers` · **Deploy:** `sync-pre-to-pro.sh` (bind-mount, sin restart)

### Panel Ajustes (FaunaApp.js + styles.css)
- Portal **a pantalla completa** con **barra lateral** (Preferencias + Información), paleta verde-pizarra (`--settings-*`), distinta del azul de la app y del rojo del admin.
- Pestañas: **General**, **Cuentas**, **Sesiones**, **API BQ** (admin o `api_token_enabled`), **Novedades**, **Ayuda**.
- **Novedades**: changelog integrado en timeline (sustituye modal).
- **Ayuda**: buscador, accesos rápidos y tarjetas colapsables (sustituye modal); todas las secciones **colapsadas al abrir**.
- Estética unificada en todas las pestañas (inputs, toggles, sesiones, botones) — sin mezcla con `--surface` azul de la app.
- Menú usuario: solo **Ajustes** (Novedades/Ayuda viven dentro). Botón `?` y **F1** → Ajustes → Ayuda. URL `/?settings=api|ayuda|novedades|…`.
- `api-tokens.html` redirige a `/?settings=api`.

### Tokens API (PAT) — backend + UI
- Tabla `user_api_tokens`; endpoints `GET/POST/DELETE /auth/api-tokens`.
- `resolve_user()` acepta `Authorization: Bearer ff_pat_…`.
- Gestión en **Ajustes → API BQ** (y pestaña homónima en Admin para referencia cruzada).
- Docs: [`API.md`](API.md), [`../bioquest/API.md`](../bioquest/API.md).

### Panel Admin (`admin/index.html`)
- Navegación **vertical** (sidebar), paleta rojiza de precaución.
- Fix **Usuarios**: `isSelf` definido en `renderUsersTable()` (PRE y PRO).
- **Estadísticas**: números KPI y pipeline en **ámbar** (`--stat-value: #fbbf24`), no en el acento rojizo.

### Fixes varios
- AutoID sibling álbum (backend `autoid_wave`, cola duplicados).
- Fix carga FF (`SyntaxError` template Vue en sesiones).

---

## 2026-08-16 10:23 — FotoFauna
**Build PRE:** `dc29f68c` · **Origen:** sync automático (Chewie)

- Cambios en working tree pendientes de commit (sync automático).


## 2026-08-15 11:xx — FotoFauna/BioQuest: cascadas LLM saneadas (modelos muertos + Gemini fuera)
**Deploy:** `docker restart fauna_api` (backend único, sirve PRE+PRO) · **Git:** `hansolo-dockers`

Sesión de mantenimiento nocturno en paralelo al reembed de BioFauna. Auditoría (`grep -rl`) de
modelos LLM caducados/retirados por los proveedores, disparada por un aviso oficial de Groq
(decommission `llama-3.3-70b-versatile`, 2026-08-16) y verificación real contra las APIs en vivo
(no de memoria) de Groq/OpenRouter/Gemini/Cerebras.

- **`vision_routes.py`/`vision_detect.py`** (cascada `/vision/locate`, localización del sujeto
  antes de recortar): eliminada la rama Gemini (decisión del usuario — cuota gratuita de Google
  de solo 20 peticiones/día por proyecto/modelo, compartida entre todas las apps del host, se
  agotaba casi de inmediato). Queda YOLO → OpenRouter (`google/gemma-4-31b-it:free` +
  `nvidia/nemotron-nano-12b-v2-vl:free`) → Groq (`qwen/qwen3.6-27b`).
- **`vision_identify.py`**: `_identify_with_gemini` (capa 3, ya código muerto sin llamadas) y sus
  imports eliminados. **`CEREBRAS_MODEL` corregido**: `llama3.3-70b` ya no existe en el catálogo
  de Cerebras (verificado en vivo, catálogo actual: `gpt-oss-120b`/`zai-glm-4.7`/`gemma-4-31b`) →
  nuevo default `gpt-oss-120b` (~10ms de latencia real medida). Este paso (capa 3c, refinado
  taxonómico anti-alucinación) llevaba tiempo fallando en silencio y devolviendo el resultado sin
  validar.
- **`routes_academy.py`** (backend de BioQuest Academy — BioQuest no tiene backend propio, lo
  sirve `fauna_api`): eliminada función `_ai_call_gemini` (con reintentos que desperdiciaban hasta
  30s por llamada contra una cuota ya agotada) y su rama en resumen narrado de especies + ficha
  rica de especie. `OPENROUTER_TEXT_MODEL` (`meta-llama/llama-3.3-70b-instruct:free`, retirado del
  catálogo) → `nvidia/nemotron-3-super-120b-a12b:free`.
- **`routes_search.py`**, **`academy/refresh_common_names.py`**: mismos modelos muertos de Groq/
  OpenRouter corregidos.
- **Verificación real** (no solo edición de config, pedido explícito por el usuario): llamadas
  directas a los 3 proveedores con los modelos nuevos (Groq y OpenRouter 200 OK; Gemini 429 cuota
  agotada, motivo real de por qué se quitó) + tráfico de producción real (ráfaga de ~46 llamadas
  del refresco de catálogo Academy tras el restart: OpenRouter respondió 200 OK el 100% de las
  veces) + sesión de navegador real con Playwright/Chrome del sistema (login, subida de foto,
  identificación — 0 errores de consola). Detalle completo en `BIOFAUNA_SESION_STATUS.md`
  (repo `hansolo-docs`).
- **Nota infra**: `tests/e2e_fotofauna_flow.py` (Playwright) no arrancaba en Ubuntu 26.04 con el
  Chromium que descarga por defecto — arreglado apuntando a `google-chrome-stable` del sistema
  (`channel="chrome"`). Sus selectores de login están desactualizados (UI cambió a
  `.ff-cross-link--guest`/`.bq-modal--auth`, el script busca `.fsh-modal`) — pendiente de
  actualizar si se quiere que el CI de cada 6h vuelva a pasar.


## 2026-08-03 10:43 — FotoFauna
**Build PRE:** `3e2a4f3f` · **Origen:** sync automático (Chewie)

- Cambios en working tree pendientes de commit (sync automático).


## 2026-08-02 — Sesión intensiva: identificación, sesión, rendimiento y audit 9/9
**PRO:** `dca7fa21` · **PRE:** `dca7fa21` · **GitHub:** `ed503572`

### Identificación
- **Re-identify tras ubicación**: arreglado dedup que bloqueaba `force=true` cuando el ID ya estaba en cola del background worker. Fotos in-flight se re-encolan automáticamente.
- **MAX_CONCURRENT=2**: worker paralelo (antes secuencial). Con backoff para 429/red/timeout.
- **Manual ID preservado**: `_resetPhotoForReidentify` ya no borra `species.source === 'manual'` al recortar.
- **Geo priors en YF interactivo**: `_biofauna_suggestion` ahora envía lat/lon/date al servicio YF.

### UI/UX
- **Ciclo 3 fotos especie**: botón ↻ en panel ID. 1558 especies con 3 miniaturas locales (dataset YF, 0 API calls). 4734 thumbnails totales.
- **`:alt` condicional**: no muestra nombres de fichero mientras el thumbnail no está listo.
- **Popups mapa**: blob URLs inválidas se ocultan con `onerror`.
- **Shift+arrows**: selecciona rango completo (antes solo celda destino).

### Sesión
- **Doble carga**: `addFiles` con guard `_addFilesBusy` anti-doble-llamada.
- **Sesión corrupta auto-detect**: si >70% sin match → auto-limpieza + reload.
- **Blob URLs pre-splice**: limpieza de `thumbUrl/sourceThumbUrl/cropThumbUrl` antes del `splice`.

### Rendimiento
- **Cold-load thumbs**: blob temporal de `p.file` antes de regenerar canvas. Sin frame negro.
- **`ensurePhotoThumbs`**: solo regenera ausentes, no toca blobs vivos.
- **`_qualityChecked`**: serializado en sesión (no re-analiza al restaurar).

### BioFauna
- **Geo priors build**: Minka-first, iNat ultra-light fallback (1 página, 5s timeout, sin reintentos). 775+/1369 especies (~80% cobertura).
- **build_geo_priors.py**: manejo 429 con Retry-After, sleep 2s, PER_PAGE=30.
- **build_species_thumbs.py**: 3 miniaturas por especie (dataset local YF, sin APIs).

### Infraestructura / multi-sitio
- **iNat calls**: reducción 99% (5000+/h → 10-20/h). Autocomplete Minka-first; fotos especie 100% locales (`/img/species_thumbs/`). Wave autoid 30/h → 10/h.
- **PRO/PRE**: separación restaurada; `deploy-to-pro.sh` con path rewriting; `deploy-fotofauna-pre.sh` con syntax check + version bump + smoke test.
- **Seguridad deploy**: `sed ?v=` excluye `vendor/` y `ff-vendor-loader.js`; `_V` usa `location.search`; `bump-version.sh` excluye `ff-vendor-loader.js`.

### Fixes del audit (`FOTOFAUNA_ANALISIS.md`) — 9/9 aplicados
| # | Sitio | Fix |
|---|-------|-----|
| T1 | FotoFauna | pokedex.html redirect respeta entorno (no hardcodea /pre/) |
| T2 | FotoFauna | Móvil usa vendor/ local (sin CDN unpkg/jsdelivr) |
| T3 | FotoFauna | SyntaxError index-mobile.html eliminado |
| T4 | Multi | Self-XSS ecosistema-search.js (`_esc(input.value)`) |
| T6 | BioQuest | Badge `PRE·20260525-A` eliminado de academy.html |
| T7 | FotoFauna | console.log de sesión gated con `window._FAUNA_DEBUG` |
| T8 | Todos | .bak files borrados |
| T10 | HanSolo Admin | admin/*.html gated con `current_user` (→ 401) |
| T12 | FotoFauna | /vision/health sin disk/apis para anónimos |

### Deuda técnica
- `useUbicacion` simplificado (parámetro `reidentifyPhoto` sin uso eliminado); debug logs fuera de `use-vision-pipeline.js` y `use-ubicacion.js`; comentarios actualizados en worker; `deploy-fotofauna-pre.sh` + `smoke-fotofauna.mjs` como herramientas permanentes.

## 2026-08-02 — Cold load: grid vacío sin sesión cacheada
**Build PRE/PRO:** `d047af61` · **Git:** `f1ac2453`

- **Bug**: al cargar un paquete de fotos **sin** sesión/IDB, se abría el panel de ubicación y el grid salía con celdas vacías (sin miniaturas).
- **Causa**: el fix de blobs del 2026-07-30 (`ensurePhotoThumbs`) anulaba/revocaba los `blob:` vivos de `makePhotoEntry` antes de tener data URLs; en cold load no hay thumbs durables de sesión → grid en blanco. `_safeRestoredThumb` también descartaba el preview vivo si la sesión no traía thumb.
- **Fix**: `ensurePhotoThumbs` solo regenera thumbs **ausentes** (no toca blob vivos); `_safeRestoredThumb` conserva el preview de `makePhotoEntry`; revoke tras paint; `onThumbError` regenera desde `file` si el blob falla.
- Deploy PRE→PRO + push `hansolo-dockers`.

## 2026-07-30 — Consola: 401s, Leaflet markers, blobs tras restore
- **401 `/usage/error`**: no POST sin token (`use-mobile-api`, `FaunaApp.trackError`); no reportar 401/403 como `http_error`.
- **401 `/api/location-points`**: no llamar sin sesión (`use-ubicacion`, `MobileApp`).
- **401 `/admin/errors`**: `loadErrors` / auto-refresh solo con admin autenticado.
- **Leaflet 404 `marker-*.png`**: PNGs en `vendor/images/` + `fixLeafletDefaultIcons` / mergeOptions (CDN mobile + sidebar + detalle).
- **ERR_FILE_NOT_FOUND UUID (blob)**: tras restore/autosave-idb, `ensurePhotoThumbs` limpia el modelo y espera un frame antes de `revokeObjectURL`; `makePhotoEntry` aplica data URL antes de revocar.
- **Nota 2026-08-02**: ese clear-before-revoke en cold load dejaba el grid vacío; ver entrada de arriba.

## 2026-07-01 — Docs, ayuda in-app y sync GitHub
- **Menú de ayuda (PRE)**: nueva sección «Buscador universal (⌘K)» en el panel ❓ — documenta atajo Ctrl/⌘+K y búsqueda cross-app FF+BQ.
- **Sync GitHub**: pull en HanSolo (`docs`, `scripts`, `cloudflare`, `multi-portal`); `hansolo-dockers` con cambios locales pendientes de merge.
- **Portal Yespi**: app Tesla renombrada a **CargaEV** (`/w/cargaev`); URL legacy `/w/kazamos-tesla` redirige.

## 2026-06-24 — SEO-03: páginas por especie + Search Console (PRE/PRO)
- **Search Console**: FotoFauna y BioQuest **verificadas** (fichero `google35607ecbe31a3370.html` en raíz — NO borrar) + dominio `yespi.es` por TXT DNS. Sitemaps enviados.
- **SEO-03 (pre-render por especie)**: `seo-species-build.mjs` parametrizado con `--app ff|bq` (mismo catálogo `academy_catalog_core`, distinto copy: FF=identificar/subir, BQ=Atlas/colección). Generadas **233 páginas estáticas por especie** (cada una indexable, con Schema.org Taxon, OG, canonical) en FF y BQ, PRE y PRO.
- **FotoFauna pasa de 5 a 239 URLs** en su sitemap (`seo-fotofauna-build.mjs` ahora incluye `_species-sitemap.json`); BioQuest a 245. Salto grande de superficie indexable → captación orgánica.
- **Automatismo**: `seo-rebuild-all.sh` (cron diario 4:30) regenera páginas + sitemaps de FF+BQ (PRE+PRO) desde el catálogo → las especies nuevas aparecen solas en <24h. Node del host (no migrable al planificador, que vive en el contenedor).

## 2026-06-24 — SCHED-04: + migraciones de crons + UX panel TAREAS (PRE/PRO)
- **Panel TAREAS arreglado y unificado**: el click en la pestaña recargaba Estadísticas porque `_ADMIN_TABS` no incluía `'sched'` (tenía la vieja `'styles'`). Corregido. Admin **PRE=PRO unificado** (PRO tenía routing divergente + MOCKUPS; ahora idénticos; ruta `/pre/img/` corregida a relativa-al-entorno).
- **Auto-refresco**: la pestaña TAREAS se recarga sola cada 5 s mientras está abierta (y solo si visible); **quitado el botón Recargar**.
- **4 crons migrados al planificador** (handlers in-process): `academy_sync_nightly`, `recompute_academy_cache`, `research_vessels_sync` (diarios), `iucn_update` (semanal, probado 222 s). Eran los 4 pasos de `bq-academy-nightly.sh` → ese cron se comentó y se desactivó su invocación dentro de `fauna-nightly.sh` (evita duplicación). Mapa completo en `docs/fotofauna/MAPA_CRONS_PLANIFICADOR.md`.
- **AutoID**: `MAX_SCAN` 300→~120 (no malgastar llamadas a iNat CV en obs que se descartan) + timeout de la tarea a 40 min. La lentitud real es el **rate-limit 429 de iNat** (esperas de hasta 80 s/imagen), no nuestro código; mitigado con tareas no bloqueantes (no frena el tick).
- **Aprendizaje operativo**: al cambiar handlers, reiniciar **ambos** backends (leader election: el líder podría ser el de código viejo → "handler desconocido").

## 2026-06-24 — FIX AutoID: duplicados de foto en la oleada automática (PRE/PRO)
- **Bug**: AutoID identificaba la misma foto re-subida varias veces (p. ej. 10× "Morena", 9× "Salpa" del mismo usuario). Causa: el control de duplicados por **pHash** existía pero SOLO se aplicaba en la cola interactiva del panel admin — la oleada automática (`autoid-wave.py`) nunca lo llamaba.
- **Fix**: la oleada aplica ahora el **pHash contra las imágenes ya identificadas en esa misma oleada**, reutilizando los bytes de foto que ya se descargan para identificar (coste cero, sin descargas extra). Umbral dist ≤ 6/64 = misma imagen. Validado: fotos distintas de la misma especie (dist 27-39) **se publican**; la misma foto re-subida (dist 0) **se salta**. **No frena fotos legítimamente diferentes** (un usuario que sube muchas fotos distintas de una morena las identifica todas).
- **Importante**: AutoID solo **AÑADE** una identificación (`POST /identifications`) — nunca modifica la observación del otro usuario. Las identificaciones ya publicadas en Minka **NO se han tocado** (orden de Gustavo).

## 2026-06-24 — Planificador SCHED-03: robustez + más migraciones (PRE/PRO)
- **Leader election** (idea Gustavo): el tick del planificador ya NO depende de un contenedor concreto. Todos los backends arrancan el loop, pero solo el que coge un `pg_try_advisory_lock` ejecuta el tick por ciclo; si el líder cae, otro toma el relevo solo. Adiós punto único de fallo.
- **Tareas no bloqueantes**: cada tarea vencida se lanza como background task (`asyncio.create_task`); una tarea LARGA (autoid, sync de Minka) ya no bloquea el tick ni el heartbeat ni retrasa las demás. Recovery `running→pending` al reiniciar el backend.
- **BQ puede gestionar tareas**: el admin embebido en bioquest.yespi.es ahora usa `/ff-api` para TODA la API admin (antes solo illustrations) → la pestaña 🗓️ TAREAS funciona igual desde BQ que desde FF.
- **Backend único = PRO** (decisión Gustavo): el split PRE/PRO es solo de frontend; el backend es uno. Fix del `_SpaHomeRedirectMiddleware` que redirigía `/admin/scheduler/*` a la home.
- **Crons migrados** (comentados en `/etc/cron.d/hansolo-fauna`, reversibles) → tareas recurring del planificador: `calc_badges` (1h), `autoid_wave` (1h), `sync_and_badges` (diario), `obs_taxon_audit` (diario dry-run) + `download_observations` (diario, nuevo). Sin migrar (2ª fase / no in-process): nightly compuestos, ai-warm/playwright (host), renew-inat-token (toca .env host), cleanup (root).

## 2026-06-24 — Planificador SCHED-02: primeras migraciones (PRE)
- **`calc_badges` migrado** al planificador: el recálculo de medallas globales (antes cron horario `calc_badges_cron.sh` vía docker exec) ahora es una tarea `recurring` (cada hora) con histórico y observabilidad. Handler in-process (`scheduler/jobs/calc_badges.py`, ejecutado en executor por ser psycopg2 síncrono). **Probado**: 52 medallas · 3 usuarios en 117ms. El cron viejo queda **comentado** (reversible) en `/etc/cron.d/hansolo-fauna`.
- **`download_observations` automatizado**: nueva tarea `recurring` diaria que descarga obs (iNat+Minka) de especies del catálogo con <5 obs locales (lote 15) → mantiene Atlas/FaunaDex al día sin intervención manual.
- Convivencia: ambas son idempotentes; el resto de crons sigue vivo hasta validar uno a uno. AutoID-wave será el último.

## 2026-06-24 — Planificador de tareas (SCHED-01) (PRE)
- **Nuevo ejecutor/planificador de tareas** con estado en Postgres y loop asyncio en el backend (`scheduler/` + `routes_scheduler.py`). Sustituye progresivamente los crons de FF/BQ y habilitará los toast "luego te cuento" del buscador IA. **Cero dependencias nuevas** (no Celery/Redis).
- **Tablas**: `scheduled_tasks` (once/recurring/count, intervalo, runs_left, prioridad, created_by, timeout), `task_runs` (histórico de ejecuciones con duración/resultado/error), `scheduler_heartbeat`.
- **Tick cada 2 min**: toma vencidas con `FOR UPDATE SKIP LOCKED`, ejecuta el handler con timeout, registra el run y reprograma según el tipo. Resistente: un handler que falla no tumba el tick ni las demás tareas. Al reiniciar el backend, las tareas 'running' vuelven a 'pending'.
- **Contingencia**: `scheduler-watchdog.sh` (cron cada 5 min) vigila el heartbeat; si lleva >10 min sin latir, reinicia el contenedor del backend para revivir el loop.
- **UI en panel admin** (pestaña 🗓️ TAREAS, sustituye la de MOCKUPS BQ): ver tareas (estado, handler, frecuencia, creada por, última ejecución, resultado), crear, pausar/reanudar, ejecutar ya, ver runs, archivar al histórico y recuperar/borrar.
- **Handlers migrados (fase 1)**: `download_observations` (descarga obs iNat+Minka de especies del catálogo) — **probado**: 3/3 especies · 36 obs. `noop` (prueba). Migración GRADUAL: los crons actuales siguen vivos; se migrarán tras validar; AutoID al final. Solo PRE.

## 2026-06-24 — SEO, analytics de negocio y mejoras móvil/admin (PRE)
- **SEO landing (PRE alineado con PRO)**: el `<body>` de la home incorpora el bloque indexable `<main id="ff-seo-content">` (h1/h2 + descripción real) que ya estaba en PRO, para que no se pierda al promover. Quitado el `<h1>` oculto duplicado.
- **ANALYTICS-EVENTS**: los eventos de negocio (subidas a Minka/iNat, detecciones, fotos cargadas) ahora se reenvían también a **GA4** vía `ffGaEvent`, no solo a `usage_sessions`. Nuevo evento `login` (OAuth) como señal de captación. Se mide aunque no haya sesión iniciada (embudo).
- **ANALYTICS-PANEL** (admin): `/admin/stats` añade `active_30d`, `new_7d`, `new_30d`; el panel muestra tarjeta «Activos 30 días» y nuevos registros (captación/retención).
- **SEO-04 / Core Web Vitals**: `preconnect`/`dns-prefetch` a `googletagmanager.com` para acelerar el handshake de GA4.
- **FF-AUTOID-MOBILE-ZOOM** (admin auto-ID): el comparador de duplicado ya no recorta (`object-fit:contain`), es más alto, responsive en móvil y cada imagen se abre a tamaño completo al pulsar.
- **UX-ICONS-INSHOT (visor)** *solo móvil*: el visor de foto (`index-mobile.html`/`MobileApp.js`) adopta el patrón InShot — botón de cierre flotante con `backdrop-filter`; tocar la imagen colapsa/expande los controles. Escritorio sin tocar.
- **Backend compartido (FaunaDex)**: el conteo de especies del FaunaDex pasa a contar **solo rank=species** (`hrank/lrank=species`) → ya no infla con subespecies; la columna «🏅 MED.» del panel admin cuenta el total real (trofeos + medallas + medallas de zona conseguidas). Cambios en `routes_pokedex.py`/`admin.py` (backend compartido FF/BQ). Detalle completo en el changelog de BioQuest.

## 2026-06-23 — Móvil: editor InShot, botones y celebración (PRE)
**Solo interfaz móvil** (`index-mobile.html`, `MobileApp.js`). Escritorio sin tocar.
- **Editor de recorte estilo InShot**: imagen mucho más grande (58vh → 82vh inmersivo), barra superior (navegación + relación de aspecto) flotando semitransparente con `backdrop-filter`, y botón ⤡/⤢ que colapsa los controles para ver la foto a casi pantalla completa. Cropper.js intacto.
- **Botones de acción del detalle**: tinte por color de marca (azul/verde/rojo/gris), icono en disco luminoso, sombras suaves y feedback `scale` al pulsar.
- **🎉 Celebración al publicar**: tras enviar a Minka/iNat con éxito, overlay efímero con confeti CSS (sin librerías) + resumen «N observaciones · M especies», vibración háptica; respeta `prefers-reduced-motion`.

## 2026-06-19 — UX identificación: acciones panel vs selección

**Panel identificar:** botones compactos 🔍 Re-detectar y ✗ Quitar ID (solo la observación visible, incl. hijo de grupo).

**Barra superior:** Re-detectar / Quitar ID solo con fotos seleccionadas (sin fallback a la foto del panel).

**Fix:** tras re-detectar, el panel sincroniza sugerencia/especie confirmada si cambió (`syncIdentifyUiFromPhoto` + watch de estado).

**Deploy PRO:** build `5c9fc6a0`.

## 2026-06-19 — CC Marina: archivo láminas IA

168 PNG (activos + `.rejected`) archivados en `/mnt/docs/bioquest/archive/cc-marina-ai-slides-20260619/`. Script `edu-archive-ai-slides.mjs`. Candidatos Commons en `cc-marina-gallery-candidates.json`.

## 2026-06-19 — Marathon QA continuo 10h

Script `/mnt/scripts/fauna/qa-marathon-continuous.sh` — bucle FF+BQ, tracker en `QA_MARATHON_BUGS.md` (retirado 2026-08-03; en git history).

## 2026-06-19 — QA Playwright recorte+filtros + fix BQ Academy

**Tarea:** E2E carga fotos, recorte 1:1, 9 filtros + antipartículas PRE/PRO; monitor consola/red.

**Fix FF:** hooks `applyCropByName`, `clickFilterInCrop`, `data-key` en botones, espera post-upload.

**Fix BQ:** `scheduleRenderDebounced is not a function` en AcademyView.

**Scripts:** `e2e_crop_filters_full.py`, `qa-all-interfaces-audit.sh` — ver `/mnt/docs/fotofauna/QA_CROP_FILTERS.md`

## 2026-06-19 — Fix auth/refresh 401 en modo anónimo (PRE `1c8663d7`, PRO `8efac762`)

**Problema:** consola mostraba `auth/refresh 401` al abrir PRE sin sesión válida (hint `ff_refresh_hint` o token caducado en `sessionStorage`).

**Fix:**
- Backend: `/auth/refresh` sin cookie/`ff_refresh` válida → `200 {"access_token": null, "guest": true}` (no 401).
- Frontend: tratar respuesta guest, limpiar hints; hint de sesión con TTL 7 días; excluir `/auth/refresh` del tracker de errores HTTP.

**Nota:** errores `region1.google-analytics.com … ERR_SSL_UNRECOGNIZED_NAME_ALERT` = bloqueo DNS (AdGuard), no afectan a FF.

**Deploy:** PRE sync + restart `fauna_api` + PRO, smoke OK.

## 2026-06-19 — Fix recorte PRE/PRO: 401 vision, RGB, blobs (`f37aa9dc`)

**Problemas:** filtros/identify devolvían 401 con token caducado; preview derecho negro al mover RGB; primer trazo de recorte fallaba; blobs rotos tras restaurar sesión; marker-icon 404 en mapa lateral.

**Fix:** Bearer inválido → invitado/cookie (backend + ffFetch); preview sin ocultar canvas; undo al confirmar recorte; purga blob thumbs en restore; divIcon en mapa sidebar.

**Deploy:** PRE→PRO HanSolo, smoke OK.

## 2026-06-19 00:24 — Deploy PRE→PRO (`977255ce`)

**Promoción:** `deploy-to-pro.sh` — rsync `public-pre/` → `public/`, bump caché, parche rutas `/pre/`→`/` en PRO (móvil incluido), SEO, smoke OK.

## 2026-06-19 00:20 — Fix móvil PRO rutas `/pre/` (`ff202606190020`)

**Problema:** `index-mobile.html` en PRO importaba assets desde `/pre/...` (MobileApp.js, iconos, manifest) → pantalla en blanco en móvil.

**Fix:** rutas raíz `/` en `public/index-mobile.html` (PRE sigue en `/pre/`).

**Infra:** DNS LAN corregido a HanSolo canónico 192.168.31.135 (ver changelog BioQuest).

## 2026-06-16 16:30 — Fix filtros IA 401 (Auto-enhance) (`ce77fa31`)

**Problema:** botón ✨ Auto (y resto de filtros) no aplicaba nada; consola `401 /vision/auto-enhance`.

**Causa:** endpoints de visión exigían Bearer JWT; usuarios sin token en `sessionStorage` (o solo cookie `ff_refresh` HttpOnly) recibían 401 sin reintento de refresh.

**Fix:**
- Backend: `current_user_or_guest` en `/vision/*` — sesión Bearer/cookie o modo invitado.
- `resolve_user`: Bearer → `ff_refresh` → SSO `yespi_access`.
- Frontend: `canAttemptFfSessionRefresh` siempre intenta refresh salvo logout explícito.
- Toast de error si el filtro falla; revierte toggle activo.

**Deploy PRE:** HanSolo.

## 2026-06-16 14:00 — Recortar: toast Auto + arreglo paréntesis en botón (`e7ed3037`)

**Problema:** al pulsar ✨ Auto, aparecían artefactos tipo paréntesis a los lados del botón; el toast «Mejora automática aplicada» tapaba la barra de filtros varios segundos.

**Fix:**
- Toast de filtros movido al **preview** (parte inferior de la imagen, encima de la barra de filtros).
- Duración del toast de filtros reducida a **2,2 s**.
- Animación `filtro-glow` sustituida por pulso de opacidad en botón/badge — sin halos laterales.
- `focus-visible` limpio en botones de filtro.

**Deploy PRE:** HanSolo.

## 2026-06-16 13:25 — Recortar: panel izquierdo instantáneo al cambiar foto (`0a2470fd`)

**Problema:** preview derecho rápido, pero el cropper (izquierda) tardaba en redibujar — spinner negro con fotos de 5 MP (p. ej. 4928×3264).

**Fix:**
- **Proxy editor** ≤2048 px para el cropper; export/preview siguen en resolución original (escala de coordenadas automática).
- Overlay se quita en cuanto la imagen es visible (no espera al `ready` del cropper).
- `cropper.replace()` con timeout corto + precarga del proxy en ±2 vecinas.
- Overlay semitransparente si aún hay espera residual.

**Deploy PRE:** HanSolo.

## 2026-06-16 13:05 — Recortar: navegación entre fotos instantánea (`66ce7ee2`)

**Bug:** al pasar de foto en foto, pantalla negra con tijeras durante varios segundos (regresión tras fix RGB).

**Causa:** los watchers de `file_original`/`file_filtered` reaccionaban al cambio de `crop2Idx` → doble init del cropper + invalidación de la caché de precarga de la foto destino.

**Fix:**
- Watchers solo actúan si cambia el archivo de la **misma** foto (no en navegación).
- Preview instantáneo desde imagen precargada (`_crop2DrawPreviewEarly`) mientras el cropper carga.
- `cropper.replace()` en lugar de destroy+new al cambiar de foto.
- Eliminado reset erróneo de `active_version` en prev/next.

**Deploy PRE:** HanSolo.

## 2026-06-16 12:25 — Recortar: fix preview negro con RGB (`b94a0697`)

**Bug:** preview derecho negro con sliders RGB activos (p. ej. R+47 G+64). El canvas 2D no soporta `filter: url(#ff-rgb-balance)` — `drawImage` fallaba y quedaba solo el fondo `#0e0e0e`.

**Fix:** en preview canvas, RGB solo vía `applyRgbBalanceToImageData`; filtros SVG reservados al cropper DOM. Dibujo en canvas temporal + blit.

**Deploy PRE:** HanSolo.

## 2026-06-16 12:15 — Recortar: preview al cargar + RGB más a la izquierda (`4f0446f3`)

**Fix preview negro al abrir:**
- Reintentos de dibujado cuando el panel derecho aún no tiene tamaño (layout flex).
- Dibujo directo desde `<img>` (sin `getCroppedCanvas`) si no hay rotación — más fiable.
- Versión filtrada: previsualiza el original mientras decodifica `file_filtered`.
- `ResizeObserver` antes de init cropper + redraw al terminar `crop2Initializing`.

**UI RGB:** sliders R/G/B pegados a la rejilla de iconos; separador solo entre RGB y Exp/Bril/Cont.

**Deploy PRE:** HanSolo.

## 2026-06-16 10:08 — Recortar PRE desplegado en HanSolo + cache bust (`f6defb4d`)

**Problema:** los cambios de Recortar (icono RGB triangular, sliders reubicados, fix preview negro) estaban solo en Chewie; `fotofauna.yespi.es/pre/` servía código antiguo.

**Fix:**
- Rsync `public-pre/` Chewie → HanSolo.
- Cache bust PRE: `APP_VERSION` embebido en `index.html` + auto-recarga al detectar `version.json` distinto (mismo patrón que BioQuest).
- `bump-version.sh` actualiza `APP_VERSION` automáticamente.

**Deploy PRE:** HanSolo (build `f6defb4d`).

## 2026-06-16 10:30 — Recortar: RGB en barra de iconos + fix preview (`ec202606161030`)

**UI Recortar:**
- Icono RGB (3 bolas en triángulo) junto a rotar/restablecer/papelera; eliminado botón descargar recorte.
- Sliders RGB a la izquierda de brillo/contraste al abrir el panel.

**Bug:** al abrir RGB desaparecía el preview del recorte (pantalla negra).

**Fix:** `v-show` en lugar de `v-if`, redibujado al togglear + `ResizeObserver` en el panel preview.

**Deploy PRE:** Chewie (pendiente HanSolo hasta este deploy).

## 2026-06-16 01:05 — Recortar: botón RGB + ∞ en sliders (`ec202606153110`)

**UI Recortar:**
- Botón RGB con tres bolas de color (R/G/B) en lugar del texto «RGB».
- Al llegar al máximo del rango normal (100 %) o en modo infinito, el valor muestra **∞** en amarillo brillante.

**Deploy PRO + PRE:** HanSolo.

## 2026-06-15 23:55 — Auth: sin parpadeo tras login (`ec202606153070`)

**Bug:** tras iniciar sesión el icono de cuenta parpadeaba (varios redibujados Acceder ↔ avatar).

**Causa:** hidrataciones OAuth + refresh en paralelo intercambiaban el bloque de nav varias veces.

**Fix:** estado `authHydrating` con placeholder «Conectando…»; `bootAuthOnce()` unifica refresh al arranque; placeholder `.ff-auth-pending` (cursor wait, sin clics).

**Deploy PRO + PRE:** HanSolo.

## 2026-06-15 23:30 — Google login sin popup (`ec202606153050`)

**Fix:** «Continuar con Google» ya no abre ventana flotante — refresh de sesión compartida o redirect en la misma pestaña (igual que BQ). OAuth return limpia `ff_logout_skip`.

**Deploy PRO + PRE:** HanSolo.

## 2026-06-15 22:40 — Modal Acceder compacto (`ec202606153010`)

**UI:** `.bq-modal--auth` — formulario más agrupado; «Crear cuenta» sin desbordar pantalla.

**Deploy PRO + PRE:** HanSolo.

## 2026-06-15 22:30 — Acceder unificado + fix logout (`ec202606153000`)

**Nav:** botón invitado **«Acceder ▾»** (antes «Cuenta»).

**Formulario:** modal login/registro al estilo BioQuest (Google arriba, tabs Entrar/Crear cuenta, campos `bq-login__*`).

**Logout:** `markFfLoggedOut()` — no re-hidrata sesión tras cerrar sesión.

**Deploy PRO + PRE:** `ec202606153000` — HanSolo.

## 2026-06-15 22:20 — Ayuda: acordeón por parejas (`ec202606152920`)

**Fix:** al pulsar cualquier sección se abre/cierra **toda la fila** (p.ej. «Agrupar» + «Identificar» a la vez). Solo una fila abierta; filas envueltas en `.fsh-help__row` con altura compartida.

**Deploy PRO + PRE:** `ec202606152920` — HanSolo.

## 2026-06-15 22:10 — Ayuda: cuadrícula alineada (`ec202606152910`)

**Fix:** modal «Cómo funciona» pasa de dos columnas apiladas a **CSS grid 2×N** (pares en fila: cargar/restaurar, agrupar/identificar…). Cada celda respeta el ancho de columna al expandir; «Atajos de teclado» ocupa ancho completo. Texto con `overflow-wrap` para no desbordar.

**Deploy PRO + PRE:** `ec202606152910` — HanSolo.

## 2026-06-15 22:00 — Menú: ayuda acordeón + topbar compacta (`ec202606152900`)

**Ayuda «Cómo funciona»:** acordeón exclusivo — al abrir una sección se cierran las demás (9 ítems en dos columnas).

**Topbar:** logo pegado al borde izquierdo (`pg-root` sin padding izquierdo; brand sin `min-width` fijo); caja de arrastre de fotos junto al logo (gap reducido, drop sin flex expansivo).

**Deploy PRO + PRE:** `ec202606152900` — HanSolo.

## 2026-06-13 — Filtros IA, UX recorte y Antipartículas (sesión completa)
**Build PRO:** `2e42ac2a` · Deploy: `deploy-to-pro.sh` + restart `fauna_api`

Sesión grande de filtros IA, UX de recorte, herramienta **Antipartículas** y correcciones backend. Todo validado en PRE y promovido a PRO.

### Antipartículas
| Cambio | Detalle |
|--------|---------|
| **Nombre** | Botón **💨 Antipartículas** en grupo **Cor** (junto a Nitidez, Enfocar, Bruma) |
| **Filtro global 💨** | Eliminado de la barra — solo herramienta local |
| **Algoritmo** | Inpaint **solo en motas detectadas** dentro del trazo (no emborrona el fondo) |
| **UX** | Al activar otro filtro se cierra antipartículas; undo global incluye filtros; pincel default ⌀ 56 |

### Filtros IA (backend `vision_image.py`)
| Filtro | Cambio |
|--------|--------|
| **🌙 Subexp. / ☀️ Sobreexp. / ◑ Contraste** | Pipelines dedicados (`mode=underexp`/`overexp`) con corrección **adaptativa**; defaults conservadores (68%/62%) |
| **🎯 Enfocar** | RL + detailEnhance + sharpen borde-aware; **sin CLAHE**; luminancia global anclada |
| **☁️ Bruma** | Dark-channel + corrección croma; **sin boost global** de brillo/contraste |
| **💨 Antipartículas** | Herramienta local; endpoint `/vision/remove-particles-stamp` |

Headers `X-Filter-Result` / `X-Filter-Detail` en respuestas de filtros.

### UI Recortar (desktop)
- Barra de filtros en una sola fila (`Exp` / `Cor` / `Col`); dial flotante de intensidad (anillo + slider vertical + ±); toast central y estados `tuning`/`applying` con debounce ~380 ms.
- Eliminado el dock «⚡ Filtros IA en lote»; navegación 2D real con flechas en grid; barra IA/import como overlay flotante sin reflow.

### Otros
- EXIF normalizado al importar (desktop) y en `file_original` (`exif-orientation.js`).
- Script regresión: `/mnt/scripts/fauna/ff-filter-regression.mjs`.
- Identificación: fix shift taxón +1, `status: detected`, scrollIntoView en grid.
- Composables nuevos: `use-filtro-dial.js`, `use-particles-stamp.js`, `exif-orientation.js`.
- Persistencia duplicados + antipartículas tras F5: 3 bugs de restore cerrados (detalle en `GRUPOS_IDENTIFICACION.md`); E2E Playwright 24 checks (timer cada 6 h).

Guía de uso de los filtros: [`FILTROS_ORGANIZACION_2026-06-13.md`](FILTROS_ORGANIZACION_2026-06-13.md).

---

Entradas manuales por sesión (las automáticas del sync Chewie se retiraron 2026-08-02; Chewie ya no es espejo desde 2026-07-21).


## 4-oct-2026 — Logs y auditoría de publicaciones (Gustavo)
- **Causa:** tras recargar el 3-oct a las 22:43 CEST no llegó al servidor NINGUNA línea del cliente; el código de `use-logger.js` vaciaba la cola antes de comprobar `ffFetch`/sesión, no reintentaba si el POST fallaba, y un POST `keepalive` con cuerpo > 64 KB falla siempre. Además los logs del contenedor `fauna_api` se pierden al recrearlo y no existía ninguna traza de publicaciones.
- **Backend (activo):** `backend/ff_audit.py` escribe una línea JSON por evento en `fauna-tmp/audit/ff-AAAAMMDD.jsonl` (persistente): `minka_publish`/`inat_publish` (entrada con resumen del payload —taxón, `species_guess`, fecha, coordenadas redondeadas—, salida con `observation_id`/uri, o error con status), `minka_unpublish`/`inat_unpublish` (ids) y `client_log.recv` (llegada de logs de cliente por sesión). `cleanup-fauna-tmp.sh` no borra `audit/`.
- **Frontend (PRO, build 1a957ab0, subido 4-oct 11:33 CEST):** `use-logger.js` — cola solo se vacía con confirmación del servidor, lotes ≤ 40 KB, reintento con espera creciente, aviso de fallo limitado a 1/min, `window.ffLogStats()` para diagnosticar desde la consola.

## 4-oct-2026 — Causa de "ID retirada que se publica" (PRO, subido 4-oct)
- **Causa raíz:** el payload de publicación (`minka_payload`, persistido en la sesión) conservaba `taxon` y `species_guess` del primer ID aunque el usuario retirara la ID: (1) `applyGroupSpeciesToMinkaPayload/Inat` devolvían el payload TAL CUAL cuando no había especie resuelta (se publicaba el taxón viejo); (2) con especie, heredaban el id de taxón del payload viejo aunque fuera de otra especie; (3) `buildMinkaPayload/buildInatPayload` y `mergeMinkaPayload` rellenaban nombre/guess/id con el payload anterior; (4) `_clearIdentificationPatch` no limpiaba `minka_payload`.
- **Arreglo:** el payload final refleja EXACTAMENTE la especie resuelta (o ninguna: `taxon` y `species_guess` a null); el id de taxón solo se hereda si el nombre es el mismo; al retirar la ID se limpia también `minka_payload`; cada publicación escribe un log `publish · taxón a publicar` (plataforma, taxón, id, origen, confianza). Probado con 8 casos en Node (sin especie, otra especie, misma, con id, builders).
- **Sigue vigente (sin cambios):** nunca se publican sugerencias pendientes; restaurar sesión usa el autosave más reciente (IDB/LS por `savedAt`); `groupSpecies` del master manda sobre las especies de los hijos (decisión de diseño).

## 4-oct-2026 — Resumen del incidente del 3-oct (cerrado)
Causa: sugerencia pendiente publicada desde el 15 % tras quitar el gate 0,83 del servidor (arreglo en PRO 22:55, tras las subidas de 21:41-22:30) y taxón de ID retirada que persistía en `minka_payload`. Arreglos en PRO (builds 1a957ab0 y 13eff6ba). Retiradas: Minka 30, iNat 66 (obs intactas); conservadas 127 y 14 respaldadas por curadores. Ver `TAREAS_FINALIZADAS.md`.
