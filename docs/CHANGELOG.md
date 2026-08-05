# FotoFauna — Changelog (rolling)

## 2026-08-03 10:43 — FotoFauna
**Build PRE:** `3e2a4f3f` · **Origen:** sync automático (Chewie)

- Cambios en working tree pendientes de commit (sync automático).


## 2026-08-02 — Sesión intensiva: identificación, sesión, rendimiento y audit 9/9
**PRO:** `dca7fa21` · **PRE:** `dca7fa21` · **GitHub:** `ed503572`

### Identificación
- **Re-identify tras ubicación**: arreglado dedup que bloqueaba `force=true` cuando el ID ya estaba en cola del background worker. Fotos in-flight se re-encolan automáticamente.
- **MAX_CONCURRENT=2**: worker paralelo (antes secuencial). Con backoff para 429/red/timeout.
- **Manual ID preservado**: `_resetPhotoForReidentify` ya no borra `species.source === 'manual'` al recortar.
- **Geo priors en YF interactivo**: `_yolofauna_suggestion` ahora envía lat/lon/date al servicio YF.

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

### YOLOFauna
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

