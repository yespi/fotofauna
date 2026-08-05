# Separación de backend PRE / PRO (FF y BQ)

**Estado:** diseño + implementación (2026-06-20) · **Prioridad:** P0 infraestructura
**Decisión Gustavo 2026-06-20:** 2 contenedores; misma DB; separación por env `FF_ENV`; mismos recursos.

## Motivación

El backend de FotoFauna (`fauna_api`) es **único** y sirve PRE y PRO desde el mismo proceso
(`./backend:/app`, sirve `/pre/*` desde `web-pre/` y `/*` desde `web/`). Cualquier cambio de
backend toca producción al reiniciar → rompe la regla PRE→PRO. BioQuest **no tiene backend
propio**: su API la provee `fauna_api`, así que separar FF separa también BQ.

## Arquitectura objetivo

| Contenedor | Env | Puerto | IP yespi-net | Sirve |
|------------|-----|--------|--------------|-------|
| `fauna_api_pro` (ex `fauna_api`) | `FF_ENV=pro` | 3005 | 172.19.0.5 | PRO (`web/`) — FF + BQ |
| `fauna_api_pre` (nuevo) | `FF_ENV=pre` | 3006 | 172.19.0.15 | PRE (`web-pre/`) — FF + BQ |

- **Mismo código backend** (`./backend`) montado en ambos. `FF_ENV` decide comportamiento.
- **Misma DB** (`FAUNA_DB_DSN` idéntico). Datos compartidos PRE/PRO.
- **Recursos iguales** (4G límite / 2G reserva cada uno). HanSolo: 30G RAM, ~20G libre → sin OOM.

## Routing nginx (nginx-proxy-manager)

- `fotofauna.yespi.es` (`6.conf`) → location `/pre/` → `fauna_api_pre:3006`; location `/` → `fauna_api_pro:3005`.
- BQ (`bioquest-webapp` nginx interno) ya separa `/pre/` y `/` por volumen; las llamadas API
  de BQ que van a fauna_api deben enrutar `/pre/*` → pre.

## Fases de implementación

1. **F-SPLIT-1** Añadir `fauna_api_pre` al compose FF (puerto 3006, IP .6, `FF_ENV=pre`). Renombrar
   `fauna_api` → `fauna_api_pro` (alias para no romper refs). Levantar sin tocar routing.
2. **F-SPLIT-2** nginx: enrutar `/pre/*` → `fauna_api_pre`. Verificar PRO intacto.
3. **F-SPLIT-3** Ajustar scripts de despliegue: `--restart` por entorno (`fauna_api_pre` para PRE,
   `fauna_api_pro` solo en paso a PRO).
4. **F-SPLIT-4** (opcional) Backend honra `FF_ENV` para no servir el frontend del otro entorno.

## Estado de implementación (2026-06-20)

- ✅ **F-SPLIT-1** — `api-pre`/`fauna_api_pre` añadido al compose FF (puerto 3006, IP `172.19.0.15`,
  `FF_ENV=pre`). Up healthy. `fauna_api` (PRO) **no se tocó** (mismo contenedor, solo se le añadió
  `FF_ENV=pro`, env inocua). IP `.6` estaba ocupada por crowdsec → se usó `.15`.
- ✅ **F-SPLIT-2** — nginx: `location /pre/ → http://fauna_api_pre:3006` en `proxy_host/6.conf`
  (`fotofauna.yespi.es`). Verificado con SNI correcto saltando caché Cloudflare: PRE parado → `/pre/`
  da 502 y `/` da 200 (PRO intacto); PRE arriba → `/pre/` da 200.
  - **NOTA NPM:** el `advanced_config` del host 6 está en la DB (respaldo), pero NPM **no regenera**
    `6.conf` al reiniciar (solo al guardar por su API/UI). El `location /pre/` se insertó a mano en
    `6.conf`. Si NPM regenera ese host desde la UI, **revisar que el bloque /pre/ siga presente**.
  - **Verificación correcta:** `curl` desde HanSolo a `fotofauna.yespi.es` va a **Cloudflare** (caché),
    no al NPM local. Para probar routing local usar `--resolve fotofauna.yespi.es:443:127.0.0.1`
    dentro del contenedor `nginx-proxy-manager`.
- ✅ **F-SPLIT-3** — `ff-pre-sync-hansolo.sh` y `bq-pre-sync-hansolo.sh` reinician `fauna_api_pre`
  (PRE), NUNCA `fauna_api` (PRO). `deploy-from-chewie.sh --restart`: para PRE usar `fauna_api_pre`.
- ⏳ **F-SPLIT-4** — backend honra `FF_ENV` (no servir el frontend del otro entorno). Opcional.

### Despliegue tras este split

- **Editar frontend PRE** (`public-pre/`) → `bump-version.sh` → `deploy-from-chewie.sh --restart "fauna_api_pre"`.
- **Editar backend** (`./backend`, compartido) → PRE: `--restart "fauna_api_pre"`; paso a PRO: `--restart "fauna_api"` (con permiso explícito).

## Rollback

`fauna_api_pre` es **aditivo**: para revertir, `docker rm -f fauna_api_pre`, quitar el servicio
`api-pre` del compose y el `location /pre/` del `6.conf`, y recargar nginx. `fauna_api` (PRO)
nunca se modificó, así que producción no corre riesgo. Backup DB NPM:
`./docker/nginx-proxy-manager/data/database.sqlite.bak-presplit-*`.
