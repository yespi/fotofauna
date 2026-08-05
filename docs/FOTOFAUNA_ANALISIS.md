# FotoFauna — Análisis crítico y plan de mejoras

> **Fecha auditoría:** 2026-08-02 · **Auditor:** IA (Cursor) · **Sitio:** https://fotofauna.yespi.es/
> **Código:** `./docker/ecosistema-fauna/` · **Build en PRO durante la auditoría:** `7c364f54`
>
> Este documento es una **auditoría sincera y exhaustiva**, no una lista de elogios. Cada tarea
> está redactada para que **otra IA (Claude, Deepseek, etc.) pueda implementarla sin contexto
> adicional**: ruta de fichero, causa raíz, comportamiento esperado y criterios de aceptación.
>
> **NO se han aplicado fixes en esta auditoría** (salvo este documento). El entregable es el análisis.

> ⚠️ **Estado 2026-08-03 — leer antes de trabajar:** en la sesión del 2026-08-02 se aplicaron ya
> **T1, T2, T3, T4, T6, T7, T8, T10 y T12** (ver `CHANGELOG.md`, sección "audit 9/9"; T1/T2 verificados en código:
> `pokedex.html:1791` respeta entorno y `index-mobile.html` ya no carga unpkg/jsdelivr).
>
> **T5 también está cerrado** (verificado el 2026-08-03 en `vision_identify.py:525`: ya prefiere la
> `p_species` calibrada y solo cae a `similarity` si no hay calibración — commit `ae6b8485`, «Tarea F»
> del handoff de YOLOFauna). La ficha de §3 y la tabla de §1 conservan la redacción original de la
> auditoría; **no reabrir T5**.
>
> **Siguen abiertos:** **T9** (monolitos) y **T11** (higiene SQL f-strings), más los menores de §6.
> No re-implementar los cerrados.

---

## 0. Metodología y qué se ha verificado

- **Sitio en vivo** (Playwright headless, escritorio 1440×900 + móvil iPhone 390×844): home, `pokedex.html`,
  `academy.html`, `seo/especie/abubilla.html`. Screenshots, errores de consola, fallos de red, imágenes rotas,
  overflow horizontal. Suite ad-hoc en `/tmp/ff-audit/` (no versionada).
- **curl**: cabeceras HTTP, CSP, CORS preflight, gating de endpoints (`/vision/identify`, `/admin/*`, `/minka/*`),
  `version.json`, `vision/health`.
- **Contenedor**: `docker ps` (`fauna_api` healthy, `fotofauna-embed` up), `docker logs`, `docker inspect`
  (RestartCount=0, sin OOM).
- **Código**: lectura de `use-session.js`, `use-vision-pipeline.js`, `use-thumbnails.js`, `auto-id-confidence.js`,
  `ff-auth-session.js`, `index.html`, `index-mobile.html`, `vision_identify.py`, `main.py`, `auth.py`,
  `ratelimit_public.py`; grep de smells (catch vacíos, console.*, innerHTML, SQL f-string, claves).

**Lo que funciona bien (para calibrar el resto):** home escritorio limpia, sin errores de consola, carga ~1.6 s,
sin overflow. Auth con cookies HttpOnly + `secure` + `samesite=lax`, TTL access 1 h / refresh 30 d. CSP razonable con
`nosniff`, `Referrer-Policy`, `Permissions-Policy`. Rate-limit por IP para anónimos, usuarios propios exentos. Los
endpoints IA y admin están **gated** (401/302 sin sesión). El fix de miniaturas en cold-load (`f1ac2453`) está bien
razonado en `use-thumbnails.js`. La gestión de 401 con refresh silencioso (`FaunaApp.js`) resuelve el histórico de
"401 spam".

**Leyenda de categorías:** `BROKEN` = bug real · `RIESGO` = diseño frágil · `MEJORA` = mejora de calidad ·
`ESTÉTICA` = UX/visual. **Esfuerzo:** S ≤1 h · M ≤ medio día · L > medio día.

---

## 1. Resumen ejecutivo (top por severidad)

| # | Cat | Pri | Esf | Título |
|---|-----|-----|-----|--------|
| T1 | BROKEN | **P1** | S | `pokedex.html` en PRO redirige al invitado a `/pre/` (staging) |
| T2 | RIESGO | **P1** | M | Móvil (`index-mobile.html`) carga Vue/Cropper/Leaflet/JSZip desde **CDN** (unpkg/jsdelivr); escritorio ya es local |
| T3 | BROKEN | **P2** | S | `index-mobile.html:2828` `SyntaxError` en cada carga móvil (asignación sobre optional chaining) |
| T4 | SEC/RIESGO | **P2** | S | `ecosistema-search.js` inyecta `input.value` sin escapar en `innerHTML` (self-XSS) |
| T5 | RIESGO | **P2** | M | YOLOFauna interactivo filtra por `similarity` (sin calibrar), no por `p_species` — *(ya registrado)* |
| T6 | ESTÉTICA | **P2** | S | `academy.html` público "en construcción" y con badge `PRE·20260525-A` visible en PRO |
| T7 | MEJORA | P3 | S | `console.log` residuales + 109 `catch` vacíos que tragan errores |
| T8 | MEJORA | P3 | S | Ficheros muertos: `pokedex.html.bak`, `academy_precache.py.bak` |
| T9 | MEJORA | P3 | L | Ficheros gigantes: `PhotoGrid.js` (3558), `use-recortar.js` (2964), `FaunaApp.js` (2689), `styles.css` (6207/237 KB) |
| T10 | RIESGO | P3 | S | `admin/index.html` y `admin/autoid.html` descargables por cualquiera (API sí gated) |
| T11 | MEJORA | P3 | M | Backend SQL con f-strings interpolando nombres de columna/`WHERE` (inyección de columnas si cambian los mapas) |
| T12 | MEJORA | P3 | S | `/vision/health` expone métricas de disco del host sin auth |

Items de infra/SEO/nombres ya vivos en `TAREAS_PENDIENTES.md` se listan en §7 (no se reabren aquí).

---

## 2. BROKEN — bugs verificados

### T1 · `pokedex.html` en PRO redirige al invitado a `/pre/` (staging) — P1 · S
- **Fichero:** `./docker/ecosistema-fauna/webapp/public/pokedex.html` línea **1791** (y copia idéntica en `public-pre/pokedex.html`).
- **Código actual:**
  ```js
  onMounted(() => {
    if (!_token()) { window.location.href = '/pre/?next=/pre/pokedex.html'; return; }
  ```
- **Causa raíz:** la ruta `/pre/` está **hardcodeada**. En PRO (`fotofauna.yespi.es/pokedex.html`) un usuario sin
  token es enviado al entorno **PRE** (staging, con `noindex`). Se reprodujo en la auditoría: al navegar directamente a
  `pokedex.html` sin sesión, la página intentó cargar 24 módulos de `/pre/composables/*.js` (todos `ERR_ABORTED`).
  Normalmente la Pokédex se abre como iframe dentro de la app (`PhotoGrid.js:301`, `:src="isPre + '/pokedex.html'"`) con
  token presente, por eso no salta siempre; pero el acceso directo o el token ausente rompe.
- **Comportamiento esperado:** el destino debe respetar el entorno actual. Usar el mismo patrón que ya existe en el
  fichero (`const isPre = window.location.pathname.startsWith('/pre') ? '/pre' : ''` — ver `PhotoGrid.js:1741`):
  ```js
  const base = window.location.pathname.startsWith('/pre') ? '/pre' : '';
  if (!_token()) { window.location.href = `${base}/?next=${base}/pokedex.html`; return; }
  ```
- **Criterios de aceptación:**
  1. Sin sesión, `https://fotofauna.yespi.es/pokedex.html` redirige a `https://fotofauna.yespi.es/?next=/pokedex.html` (NO a `/pre/`).
  2. Sin sesión, `https://fotofauna.yespi.es/pre/pokedex.html` sigue redirigiendo a `/pre/?next=/pre/pokedex.html`.
  3. La Pokédex embebida (iframe con token) sigue funcionando en ambos entornos.
- **Nota deploy:** el fix debe editarse en `public-pre/pokedex.html` y promoverse a `public/` (flujo PRE→PRO habitual). Correr `bump-version.sh` antes de promover.

### T3 · `SyntaxError` en cada carga de la app móvil — P2 · S
- **Fichero:** `./docker/ecosistema-fauna/webapp/public/index-mobile.html` línea **2828** (idéntico en `public-pre`).
- **Código actual:**
  ```html
  <script>document.getElementById('ff-seo-crawl')?.style?.display='none';</script>
  ```
- **Causa raíz:** **no se puede asignar a una expresión con optional chaining**. `a?.b?.c = 'x'` es
  `SyntaxError: Invalid left-hand side in assignment` en tiempo de parseo. Se confirmó con Playwright móvil: cada carga
  de la home móvil emite un `pageerror`. Además `index-mobile.html` **no tiene** ningún elemento `#ff-seo-crawl` (ese
  bloque solo existe en el `index.html` de escritorio), por lo que el script es inútil aunque no fallara.
- **Comportamiento esperado:** eliminar la línea (no hay nada que ocultar en móvil) o, si se quiere conservar por
  simetría, reescribir sin asignación sobre optional chaining:
  ```html
  <script>var _c=document.getElementById('ff-seo-crawl'); if(_c) _c.style.display='none';</script>
  ```
- **Criterios de aceptación:** al cargar `https://fotofauna.yespi.es/` en móvil (o `index-mobile.html`) la consola no
  muestra `SyntaxError: Invalid left-hand side in assignment`. Editar en `public-pre` y promover.

---

## 3. Seguridad / RIESGO

### T2 · El móvil depende de CDN externos para toda su base — P1 · M
- **Fichero:** `./docker/ecosistema-fauna/webapp/public/index-mobile.html` líneas **21–38**.
- **Código actual:**
  ```html
  <script src="https://unpkg.com/vue@3.4.21/dist/vue.global.prod.js"></script>
  <link rel="stylesheet" href="https://unpkg.com/cropperjs@1.6.2/dist/cropper.min.css" />
  <script src="https://unpkg.com/cropperjs@1.6.2/dist/cropper.min.js"></script>
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/jszip@3.10.1/dist/jszip.min.js"></script>
  ```
- **Causa raíz:** el **escritorio ya se migró a `vendor/` local** (`index.html:75` carga `./vendor/vue.global.prod.js`),
  pero el **móvil quedó en unpkg/jsdelivr**. Esto es exactamente la clase de fallo del histórico "Leaflet 404s / CDN
  failures": si unpkg cae o rate-limita, **la app móvil entera no arranca** (Vue no carga). Es además un punto de fuga
  de privacidad (IP del usuario a terceros) y hace que la CSP tenga que mantener `script-src ... unpkg cdn.jsdelivr`.
- **Comportamiento esperado:** servir Vue, Cropper, Leaflet y JSZip desde `/vendor/` local, igual que el escritorio.
  Verificar que los ficheros existen ya en `webapp/public/vendor/` (Leaflet/Cropper/JSZip se cargan bajo demanda por
  `ff-vendor-loader.js` en escritorio) y referenciarlos con rutas relativas para que `/pre/` sirva PRE.
- **Criterios de aceptación:**
  1. `index-mobile.html` no contiene ninguna URL `unpkg.com` ni `cdn.jsdelivr.net`.
  2. Con el bloqueo de red a `unpkg.com`/`cdn.jsdelivr.net`, la app móvil arranca, el recorte (Cropper) y el mapa
     (Leaflet) funcionan.
  3. Tras verificar, se puede endurecer la CSP quitando `unpkg`/`jsdelivr` de `script-src`/`style-src`/`connect-src`
     en `main.py` (líneas ~330–340). (Opcional, en tarea aparte.)
- **Riesgo de la tarea:** medio — comprobar que las versiones locales existan y que `ff-vendor-loader.js` no dé por
  hecho el CDN. Probar recorte + mapa en móvil real.

### T4 · Self-XSS en el buscador global — P2 · S
- **Fichero:** `./docker/ecosistema-fauna/webapp/public/ecosistema-search.js` línea **178**.
- **Código actual:**
  ```js
  results.innerHTML = `<div class="eco-empty">Sin resultados para «${input.value}»</div>`;
  ```
- **Causa raíz:** `input.value` se interpola **sin escapar** en `innerHTML`. El resto del fichero sí usa el helper
  `_esc()` (línea 206) para labels/sublabels, pero este caso quedó fuera. Un `<img src=x onerror=...>` en el buscador
  ejecuta script. Es **self-XSS** (el usuario se ataca a sí mismo), impacto bajo, pero es una regla de higiene que ya
  se cumple en el resto del fichero y aquí se rompió.
- **Comportamiento esperado:** `` `...«${_esc(input.value)}»...` `` usando el helper existente.
- **Criterios de aceptación:** escribir `<b>x</b>` en el buscador y no obtener resultados muestra el texto literal
  `«<b>x</b>»`, no HTML renderizado.

### T5 · YOLOFauna interactivo filtra por `similarity`, no por `p_species` — P2 · M — *(ya registrado en TAREAS_PENDIENTES)*
- **Fichero:** `./docker/ecosistema-fauna/backend/vision_identify.py` líneas **125–126, 524**.
- **Código actual:**
  ```py
  YOLOFAUNA_MIN_CONFIDENCE = float(os.getenv("YOLOFAUNA_MIN_CONFIDENCE", "0.85"))
  ...
  local = await _identify_with_yolofauna(image_path, lat=lat, lon=lng, date=observed_on)
  if local and local.get("confidence", 0) >= YOLOFAUNA_MIN_CONFIDENCE:
      return local, "yolofauna"
  ```
  donde `confidence` = `round(float(top.get("similarity", 0)), 3)` (similitud coseno **sin calibrar**), mientras que
  la respuesta ya trae `p_species` calibrada cuando `pred.calibrated` (líneas 497–499).
- **Causa raíz:** el flujo **wave** ya migró a `p_species` (umbral 0.90, 96 % precisión real — ver
  `TAREAS_PENDIENTES.md` §YOLOFauna), pero el flujo **interactivo** (identificación en la app) sigue comparando
  `similarity` ≥ 0.85, que no es una probabilidad y no está calibrada. Esto puede auto-confirmar especies con
  similitud alta pero baja probabilidad real (congéneres indistinguibles).
- **Comportamiento esperado:** cuando `local.get("calibrated")` sea true, decidir por `p_species` ≥ umbral calibrado
  (p. ej. `YOLOFAUNA_MIN_P_SPECIES` reutilizando el env del wave, 0.90) y caer a `similarity` solo si no hay
  calibración. Al cambiar el criterio, **re-verificar** que la app sigue mostrando la especie en el grid
  (ver histórico `ac4b5c2c`: `isHighConfidenceSpecies` acepta `yolofauna`/`yf`).
- **Criterios de aceptación:** con una foto cuya `p_species` < 0.90 pero `similarity` ≥ 0.85, la identificación
  interactiva **no** auto-confirma (queda como sugerencia `_pendingIdSuggestion`). Con `p_species` ≥ 0.90 sí confirma.
- **Nota:** ya está anotado en `TAREAS_PENDIENTES.md` ("YOLOFAUNA_MIN_CONFIDENCE ... ojo: compara similitud, no
  p_species; migrar a p_species pendiente"). Se incluye aquí para trazabilidad; **no duplicar**.

### T10 · `admin/*.html` descargables por cualquiera — P3 · S
- **Rutas:** `GET /admin/index.html` y `GET /admin/autoid.html` devuelven 200 (121 KB / 107 KB) sin sesión.
- **Causa raíz:** los HTML del panel admin se sirven como estáticos. La **API** admin sí está protegida (probado:
  `/admin/api/stats` → 302 a login), así que no hay fuga de datos, pero el markup revela estructura, endpoints y
  nombres internos a cualquiera.
- **Comportamiento esperado:** servir `admin/*.html` solo con sesión admin (middleware/route guard en `main.py`), o
  como mínimo asumir el riesgo conscientemente y documentarlo.
- **Criterios de aceptación:** `curl https://fotofauna.yespi.es/admin/index.html` sin cookie → 302/401; con sesión
  admin → 200.
- **Severidad honesta:** baja (defense-in-depth); no es urgente porque la API está gated.

### T12 · `/vision/health` expone disco del host sin auth — P3 · S
- **Evidencia:** `curl https://fotofauna.yespi.es/vision/health` → `{"disk":{"total_gb":217.97,"used_gb":138.69,
  "free_gb":68.14,"used_pct":63.6}, "apis":{"groq":true,"inat":true,"gemini":true}, ...}`.
- **Causa raíz:** el health público informa espacio de disco del host y qué APIs están configuradas. Útil para
  monitorización, pero es información de infraestructura innecesaria para anónimos.
- **Comportamiento esperado:** health público reducido a `{status, db}`; mover disco/uptime/apis a un endpoint
  autenticado o solo interno (Docker healthcheck ya usa el suyo).
- **Criterios de aceptación:** `/vision/health` anónimo no incluye `disk` ni `apis`.

---

## 4. Lógica y backend (RIESGO / MEJORA)

### T11 · SQL con f-strings interpolando columnas / cláusulas WHERE — P3 · M
- **Ficheros/líneas:** `admin.py:369` (`{col}` en INSERT/UPDATE), `admin.py:486` (`{where_sql}`),
  `routes_academy.py:144,322,1233,1285,1342`, `routes_pokedex.py:399`.
- **Estado real:** los **valores** van parametrizados (`$1,$2,...`), lo cual es correcto. Lo que se interpola por
  f-string son **nombres de columna** (`col_map.get(body.event)` en `admin.py`, validado contra un mapa con `if not
  col: raise 400`) y fragmentos `WHERE` construidos de listas internas. Hoy **no es explotable** porque las columnas
  salen de mapas cerrados. Es un patrón frágil: si alguien añade una entrada al mapa desde input de usuario, se abre
  inyección.
- **Comportamiento esperado:** mantener whitelist explícita de columnas permitidas (ya existe en `admin.py`), y
  documentar el invariante "nunca interpolar identificadores derivados de input sin whitelist". No requiere cambio
  funcional inmediato; es deuda a marcar.
- **Criterios de aceptación:** comentario/aserción junto a cada f-string SQL indicando de dónde sale el identificador
  y que está en whitelist. (Tarea de higiene, no bug.)

### Observaciones de lógica que NO son bugs (verificadas, para no reabrirlas)
- **Autosave/sesión (`use-session.js`):** doble escritura localStorage + IndexedDB, guardado síncrono en
  `pagehide`, y detección de "sesión corrupta" (>70 % sin match → auto-limpieza + reload). Bien diseñado. El fix de
  ubicaciones (`scheduleAutosave` tras `applyLocation`, commit `5113c2f0`) está en `TAREAS`. **OK.**
- **Cold-load thumbs (`use-thumbnails.js`):** `ensurePhotoThumbs` ya NO revoca blobs vivos (fix `f1ac2453`). Revisado
  todo el fichero: no se ve otra ruta que revoque un blob todavía referenciado por Vue. `onThumbError` regenera desde
  `file`. **No se detectan regresiones similares latentes.**
- **Worker de identificación (`use-vision-pipeline.js`):** cola con `MAX_CONCURRENT=2`, backoff diferenciado
  (429/red/timeout), cancelación por `_suppressAutoId`. Robusto. Único matiz: `startIdentifyWorker` deja un
  `setInterval` cada 6 s (`_workerWatchTimer`); confirmar que `stopIdentifyWorker` se llama en `onBeforeUnmount`
  (MEJORA menor si no).

---

## 5. Calidad de código (MEJORA)

### T7 · `console.log` residuales + 109 `catch` vacíos — P3 · S/M
- **Evidencia:** 5 `console.log` "propios" + ~45 `console.*` en total en JS propio; destacan en
  `use-session.js` (líneas 497, 600, 636, 913, 959) que imprimen detalle de sesión en producción. **109** bloques
  `catch(_) {}` / `catch {}` vacíos en `*.js` + `composables/*.js`.
- **Causa raíz:** ruido de depuración enviado a producción y errores tragados silenciosamente (dificulta diagnóstico
  del histórico de bugs de sesión/thumbs).
- **Comportamiento esperado:** (1) quitar/gate los `console.log` de sesión tras `window._FAUNA_DEBUG`; (2) para los
  `catch` vacíos críticos (IDB, blobs, restauración), enrutar al menos a `ffLogger.debug` para tener rastro.
- **Criterios de aceptación:** en PRO, una restauración de sesión normal no imprime `[FF Session] ...` salvo con
  `?debug=1`/`window._FAUNA_DEBUG`. (No es necesario tocar los 109 de golpe; priorizar los de sesión/IDB.)

### T8 · Ficheros muertos — P3 · S
- **Ficheros:** `webapp/public/pokedex.html.bak` (80 KB), `backend/academy_precache.py.bak`.
- **Acción:** borrar (están versionados en git; no hacen falta como respaldo).
- **Criterio de aceptación:** `find webapp/public backend -name '*.bak'` no devuelve nada.

### T9 · Ficheros gigantes / monolitos — P3 · L
- **Ficheros:** `PhotoGrid.js` 3558 líneas, `use-recortar.js` 2964, `FaunaApp.js` 2689, `MobileApp.js` 2058,
  `index-mobile.html` 2830; `styles.css` 6207 líneas / 237 KB (un único CSS sin split ni minificar);
  backend: `routes_buceo.py` 2055, `routes_academy.py` 1651, `routes_minka_autoid.py` 1557.
- **Causa raíz:** crecimiento orgánico. `PhotoGrid.js` mezcla template Vue enorme + 84 funciones en `setup()`
  (recordar el bug `ac4b5c2c`: `sourceTitle` definida en `setup()` pero fuera del `return` → pantalla negra; los
  componentes tan grandes hacen ese tipo de fallo fácil y difícil de detectar).
- **Comportamiento esperado:** extraer bloques de `PhotoGrid.js` a composables (ya hay patrón `use-*.js`); considerar
  minificar `styles.css` en deploy (build step) o al menos dividir por dominio (grid, panel ubicación, recorte).
- **Criterios de aceptación:** ningún fichero JS propio > 1500 líneas (objetivo, iterable); `styles.css` servido
  minificado o partido. **Esfuerzo alto; hacer por partes, no en un PR.** No bloqueante.

---

## 6. Estética / UX

### T6 · `academy.html` público "en construcción" + badge `PRE·20260525-A` en PRO — P2 · S
- **Fichero:** `./docker/ecosistema-fauna/webapp/public/academy.html` líneas **179** (`<span>PRE·20260525-A</span>`),
  **221** y **290** ("En construcción" / "En construcción activa").
- **Evidencia:** screenshot de `https://fotofauna.yespi.es/academy.html` muestra la página casi vacía, mensaje "En
  construcción activa" y un **badge `PRE·20260525-A`** arriba a la izquierda **en producción**.
- **Causa raíz:** feature a medias accesible públicamente y con un marcador de build/entorno ("PRE...") hardcodeado que
  se ve en PRO. Da imagen de producto inacabado.
- **Comportamiento esperado:** (a) quitar el badge `PRE·...` o mostrarlo solo en PRE; (b) o bien gate de Academy hasta
  que tenga contenido, o bien un empty-state honesto sin la etiqueta de entorno.
- **Criterios de aceptación:** en PRO no aparece ningún texto tipo `PRE·<fecha>` en la UI de Academy.

### Otros (menores, ESTÉTICA — sin ficha completa)
- **Home móvil = pantalla de login directa** (screenshot): sin sesión, el móvil enseña el formulario de acceso a pantalla
  completa, sin el hero/onboarding que sí tiene el escritorio (los 3 pasos "Sube / IA detecta / Publica"). Coherencia:
  valorar un onboarding móvil equivalente o al menos el CTA "Escuchar/probar sin cuenta" que ya existe en otros
  productos del ecosistema. (MEJORA producto, P3.)
- **Banner de cookies** tapa parcialmente el CTA inferior en escritorio hasta que se acepta (screenshot home). Aceptable,
  pero valorar que no solape el botón "Arrastra o selecciona tus fotos". (ESTÉTICA, P3.)
- **Copy español**: correcto y consistente en las vistas revisadas (home, SEO especie, Academy). Sin errores
  ortográficos detectados. La ficha SEO de Abubilla está bien redactada (grupo, obs, especies relacionadas). **OK.**

---

## 7. SEO — estado y huecos (NO rehacer)

Trabajo reciente **verificado y correcto**, no tocar:
- **403 fichas** en `seo/especie/*.html`; `sitemap.xml` con **413 `<loc>`**; `robots.txt` 200.
- Ficha `seo/especie/abubilla.html`: `<title>` descriptivo, breadcrumb, datos (obs, grupo, taxon iNat), CTA a la app,
  especies relacionadas, rango de años. Contenido suficiente para no ser "thin".
- `index.html`: canonical, OG/Twitter, JSON-LD `SoftwareApplication`, `robots index,follow`, contenido crawlable
  `#ff-seo-crawl` con enlaces internos. PRE con `noindex` vía middleware (`_PreNoIndexMiddleware`). **OK.**

Huecos remanentes (menores, ya en seguimiento en `TAREAS_PENDIENTES.md` §SEO — *ya registrado*):
- Indexación Google sin auto-fix (inspección URL manual pendiente); GA4 SA Lector pendiente. **No reabrir aquí.**
- Enlazado interno entre fichas SEO existe (especies relacionadas) pero es intra-grupo; posible mejora futura de
  enlazado cruzado geográfico. Bajo impacto.

---

## 8. Regresiones recientes — verificación (área 7 del encargo)

- **Cold-load blob thumbs (`f1ac2453`):** revisado `use-thumbnails.js` completo. El fix está bien y **no se detecta
  otra ruta que revoque blobs vivos**. `makePhotoEntry` auto-promueve blob→dataURL y solo revoca tras `rAF`;
  `ensurePhotoThumbs` solo regenera ausentes. **Sin regresión latente similar.**
- **`sourceTitle` fuera del return (`ac4b5c2c`):** confirmado presente en el `return` actual de `PhotoGrid.js`
  (líneas ~2183, 3474 exponen el bloque de estado de fecha; `sourceTitle` usado en template). El riesgo estructural
  persiste por el tamaño del componente (ver T9), no por este símbolo concreto.
- **Ubicaciones no persistidas (`5113c2f0`) y GPS re-identificar:** cerrados en `TAREAS`. No reintroducidos en el
  código leído (`use-vision-pipeline.js` respeta GPS de sesión/prefs).

---

## 9. Items ya registrados en `TAREAS_PENDIENTES.md` (no reabrir — trazabilidad)

Estos ya están anotados; se listan para que otra IA no los duplique:
- **YOLOFauna interactivo `similarity`→`p_species`** (T5 arriba) — *ya registrado*.
- **Revisión diaria de nombres de especie** (cron/scheduler que revise sinónimos/reclasificaciones en Minka) — *ya registrado* (§FotoFauna 🔜).
- **Cron `hansolo-fauna`** requiere `sudo` en host (`/etc/cron.d`) — *ya registrado* (§Infra).
- **SEO**: indexación Google sin auto-fix, GA4 SA Lector, inspección URL manual — *ya registrado* (§SEO).
- **YOLOFauna Fase 2**: priors GPS+fecha (bloqueado por rate-limit iNat 429), fusión de sinónimos, fine-tuning BioCLIP,
  re-embed con crop — *ya registrado* (§YOLOFauna Fase 2 / `HANDOFF_YOLOFAUNA_FASE2.md`).

---

## 10. Orden de ataque sugerido

1. **T1** (pokedex→/pre/) y **T3** (SyntaxError móvil): S cada uno, fixes de una línea, alto valor. Editar en
   `public-pre`, `bump-version.sh`, promover a `public`, verificar con curl/Playwright, push a GitHub.
2. **T2** (móvil a `vendor/` local): mayor esfuerzo pero cierra el riesgo real de "app móvil no arranca si cae el CDN".
3. **T4** (escape buscador) y **T6** (badge PRE en Academy): S, higiene visible.
4. **T5** (p_species interactivo): coordinar con el equipo YOLOFauna; requiere recalibrar/verificar grid.
5. Resto (T7–T12): higiene de fondo, por lotes, sin prisa.

> **Recordatorio de flujo (CLAUDE.md):** se trabaja en local en HanSolo (`/mnt/`), se edita PRE, se corre
> `bump-version.sh`, se promueve a PRO, se verifica con `docker ps`/`curl`, y **push a GitHub**. Sin SSH, sin sync a Chewie.
