# ⚠️ ACTUALIZACIÓN 2026-08-04 — sesión intensiva de mejora del acierto

> **TL;DR:** Triplet loss es la técnica ganadora (mejora 99% de especies). Crop no aporta.
> Fine-tuning de BioCLIP bloqueado por VRAM (9,6/12 GB solo con los pesos) + LoRA/open_clip
> incompatible con `peft`. Imágenes migradas a SSD (`/mnt/gpu/fotofauna-images/`).
> 975 spp, 520k fotos, calibración 63.9% especie, 95.4% precisión p≥0.98.
>
> ### 👉 **El plan de acción operativo vive ahora en [`YOLOFAUNA_PLAN_MAESTRO.md`](YOLOFAUNA_PLAN_MAESTRO.md)**
> Lo de aquí abajo (PLAN DE MEJORA / 1bis / 1ter) es el *research log* que sustenta ese plan
> — para saber "qué hago ahora, en qué orden" leer el plan maestro, no esto.

---

## 📋 PLAN DE MEJORA — próxima sesión (act. 2026-08-04, no ejecutado aún)

Objetivo: seguir subiendo el % de acierto de especie y llegar a un modelo que **falle poco
y sepa cuándo no sabe** (abstención + calibración fiable), no solo maximizar in-sample.
Orden por impacto esperado / esfuerzo:

### 1. 🔴 Desbloquear el fine-tuning real del encoder BioCLIP — la única palanca que rompe el techo
El triplet loss ya aplicado (sección arriba) entrena una **transformación sobre los
embeddings ya extraídos**, no toca los pesos de BioCLIP. Mover el propio espacio de
embeddings (metric learning sobre el encoder) es lo que de verdad separa pares crípticos,
y sigue bloqueado:
- **LoRA vía PEFT**: el intento manual chocó con incompatibilidad open_clip/BioCLIP.
  Probar la librería oficial `peft` (HuggingFace) en vez de una implementación manual de
  LoRA — es la causa más probable del bloqueo actual. Si `peft` tampoco soporta la
  arquitectura open_clip ViT-L/14 de BioCLIP-2, documentarlo como bloqueador real (no
  solo "no probado") y pasar a la alternativa siguiente.
- **Alternativa sin LoRA (más simple, probarla primero)**: descongelar solo el/los
  **último(s) bloque(s) transformer** de la ViT-L y fine-tunear directamente (sin
  adaptadores) con triplet/ArcFace sobre pares crípticos. Es la "etapa 1" que ya
  recomendaba la revisión externa antes de saltar al fine-tune completo — menor riesgo,
  menos VRAM, no depende de que LoRA funcione con open_clip.
- **Dataset de pares crípticos** (gratis, ya identificado el método): sacar de
  `identifications[].category=="maverick"` / `identifications_most_disagree` de Minka y/o
  de la matriz de confusión de `scripts/analyze_acc.py`. Guardar `dataset/crypto_pairs.json`.
- Son **GPU-días**: trocear en checkpoints reanudables (`dataset/ft_ckpts/`), una sesión
  no tiene que terminarlo.
- **Al desplegar cualquier checkpoint**: regenerar TODOS los embeddings con el encoder
  nuevo y **recalibrar** (`harvest_calib.py` + `fit_calib.py`) — no negociable, cambia el
  espacio de scoring.
- Criterio de aceptación: sube el acierto en el held-out de pares crípticos **sin** bajar
  el acierto global medido con `harvest_calib.py` (nunca `eval_field.py`, que mide con fuga).

### 2. 🟡 Completar los geo priors (GPS + fecha)
281 de 1369 especies siguen sin coordenadas (Minka-first pendiente; `build_geo_priors.py`
se corrompe al escribir — depurar ese bug, no reescribir desde cero). Una vez completo,
**recosechar la calibración** con los geo priors ya activos en el 100% del catálogo (hoy
solo están parcialmente activos). Impacto esperado moderado, esfuerzo bajo (es terminar
algo ya empezado, no una técnica nueva).

### 3. 🟢 Fusión de sinónimos — barrido exhaustivo
Solo se ha confirmado un caso (`ambigolimax_valentianus` ≡ `lehmannia_valentiana`). Correr
`scripts/analyze_acc.py` buscando sistemáticamente pares con confusión mutua alta y mismo
taxón real en Minka (que es el árbitro del nombre canónico, sección C). Gratis, pero
impacto pequeño (puñado de casos) — hacerlo en paralelo a 1/2, no como bloque dedicado.

### 4. 🟢 VLM re-ranker sobre el top-3 crítico
Solo para los casos en zona de incertidumbre (top-1/top-2 muy cerca, congéneres): pasar el
top-3 candidato a un VLM (qwen local o similar) pidiendo que verifique rasgos diagnósticos
textuales de la foto contra las 2-3 especies candidatas. Barato porque solo se invoca en la
minoría de fotos dudosas, no en todo el flujo. Explorar después de 1, cuando ya haya menos
pares realmente indistinguibles por embedding.

### 5. 🔵 Reactivar el wave de auto-publicación en Minka — condicionado, no antes de 1-2
Sigue **pausado** (publicaba erróneas en especies crípticas). Solo reactivar cuando el
acierto out-of-sample suba de forma medible tras 1 y/o 2. Empezar conservador
(`p_species ≥ 0.90`, ~96% precisión medida hoy) con la salvaguarda `WAVE_YOLOFAUNA_REQUIRE_INAT=1`.

### 1bis. 🔍 Pistas externas investigadas (2026-08-04, no probadas aún)
Búsqueda puntual sobre novedades de YOLO26/BioCLIP/alternativas que puedan mover 1-4:
- **Vía de escape para el bloqueo de LoRA (punto 1)**: la librería genérica `peft` de
  HuggingFace tiene un *issue* abierto sin resolver desde 2023 pidiendo soporte OpenCLIP
  — no es tan fiable como se anotó ayer. Alternativa más prometedora: **`clipora`**
  (github.com/awilliamson10/clipora), toolkit hecho específicamente para LoRA sobre
  **OpenCLIP** (la base real de BioCLIP-2). Está en desarrollo activo (le falta merge de
  adapters), probarlo antes de asumir que hace falta ir directo a "descongelar el último
  bloque sin LoRA".
- **YOLOE-26** (extensión open-vocabulary de YOLO26, ya usado como detector): da
  **segmentación por texto** ("nudibranch", "marine animal") en vez de solo bbox. Es un
  mecanismo distinto al crop por caja ya descartado — quita el fondo píxel a píxel. Para
  heterobranquios camuflados sobre roca/coral del mismo color podría ayudar donde el
  bbox no ayudó; **no confundir con la conclusión ya cerrada de que el crop no aporta**
  (esa era con bbox, no con máscara). También serviría para sustituir los fallbacks a
  Gemini/Groq del backend FotoFauna cuando YOLO no detecta nada.
- **DINOv3** (Meta, self-supervised): viene en tamaños ViT-S (21M) a ViT-7B; **ViT-S/ViT-B
  (21M/86M) caben de sobra en la 3060**, licencia comercial permisiva. Evaluado
  específicamente en iNaturalist 2018/2021 con resultados fuertes en fine-grained.
  Embeddings densos a nivel de píxel que codifican estructura de segmentación **sin
  cabeza dedicada** — candidato a experimento paralelo para separar organismo/fondo sin
  depender de un detector de cajas. No sustituye a BioCLIP-2 (más específico para
  taxonomía), sería un encoder complementario a probar en pares crípticos.
- **BioCLIP-2 sigue siendo el estado del arte** (no hay BioCLIP-3). Su propio repo
  (`Imageomics/bioclip-2`) **NO** tiene guía de fine-tuning ligero (LoRA/adapters/freeze
  parcial): solo entrenamiento completo distribuido (SLURM) con *experience replay* sobre
  LAION-2B para evitar olvido catastrófico. Esto confirma que la vía realista en local
  **no es replicar su receta oficial**, sino fine-tuning parcial (bloque final / LLRD, ver
  abajo) — no hay atajo oficial que nos hayamos saltado por error.
- Existe un fine-tune comunitario de BioCLIP para arrecife de coral
  (`ReefNet/finetuned-bioclip` en HF, dataset **ReefNet** — arxiv 2510.16822, "A
  Large-Scale Dataset and Benchmark for Fine-Grained Coral Reef Recognition"). El scraping
  automático no consiguió sacar el método exacto (LoRA/full/linear-probe) del paper — si se
  quiere replicar de verdad, hay que **leer el PDF completo a mano**, no solo el abstract.
  Precedente académico de que fine-tunear BioCLIP en un dominio marino nicho funciona.
- **BioVITA** (2026, audio+imagen) y **CLIBD** (imagen+ADN+texto) — descartados: no
  aplican (no hay audio ni secuencias de ADN en el dataset).

### 1ter. 🎯 Papers de clasificación taxonómica marina — técnicas concretas y aplicables
Búsqueda dirigida a "underwater cryptic species classification 2026" encontró trabajo
directamente en el dominio (peces/fauna marina con jerarquía taxonómica), con técnicas
trasplantables sin cambiar de arquitectura base:

- **Inferencia de mínimo riesgo taxonómico** (paper "Taxonomy-aware deep learning for
  hierarchical marine species classification", arxiv 2606.25989): en vez de decidir por
  argmax/similitud máxima, usan una **regla de decisión bayesiana** que elige la predicción
  que **minimiza la distancia taxonómica esperada** (usando una matriz de distancias entre
  taxones), no la de mayor probabilidad puntual. **Esto es una generalización directa de la
  abstención por margen que ya tenéis** (`YOLOFAUNA_FAMILY_MARGIN=0.06`, sección F): en vez
  de un margen fijo especie→género→familia, se pondera por *cuánto cuesta* equivocarse en
  la jerarquía real de Minka. Encaja con vuestro pipeline kNN sin rehacer el encoder — es
  cambiar la regla de decisión final, no reentrenar nada. Candidato de bajo riesgo/esfuerzo
  a probar **antes** que el fine-tuning del encoder (puede subir el género/familia útil sin
  tocar embeddings).
- **Layer-wise Learning Rate Decay (LLRD)** para el fine-tuning del backbone (mismo paper,
  comparado contra fine-tune ingenuo): +11,3% sobre fine-tune naive descongelando todo con
  la misma tasa de aprendizaje. Aplica tasas de aprendizaje decrecientes por bloque desde la
  salida hacia la entrada, en vez de la disyuntiva binaria "solo el último bloque sí/no".
  **Refina el punto 1 del plan**: cuando se llegue al fine-tuning del encoder, usar LLRD en
  vez de congelar todo menos el último bloque — mismo espíritu (bajo riesgo, no todo el
  backbone se mueve igual) pero más flexible.
- **MATANet** (arxiv 2601.03729, FathomNet 2025 / FishCLEF2015): mejora clasificación
  fine-grained con (a) atención multi-contexto que combina el recorte del organismo **con
  su entorno a varias escalas** en vez de solo el bbox aislado, y (b) clasificadores
  auxiliares por rango taxonómico durante el entrenamiento (sin tocar la predicción final).
  **Matiz importante para la sección E/"el crop no aporta"**: esa conclusión fue con
  bbox-crop puro (organismo aislado, sin contexto). MATANet sugiere que **crop + contexto
  multi-escala** es un mecanismo distinto — no está refutado por vuestro experimento previo.
  Código prometido en `github.com/C2A2-at-Florida-Atlantic-University/fathomnet-taxonomy`
  "tras publicación" (aún no confirmado si ya está subido).
- **FathomNet** (MBARI, 400k+ imágenes, API pública): fuente de datos potencial adicional,
  pero **ojo**: es mayormente fauna de aguas profundas/ROV (Pacífico), solapamiento incierto
  con heterobranquios mediterráneos de aguas someras — comprobar solapamiento de especies
  antes de invertir tiempo descargando. No asumir que aporta cobertura útil sin verificarlo.
- Línea de fondo (no producto, formaliza lo que ya hacéis): embeddings jerárquicos con
  geometría hiperbólica para jerarquías taxonómicas. Confirma que la abstención ya
  implementada va en la dirección correcta; no urgente, vigilar.

**Prioridad sugerida entre las pistas nuevas**: 1) inferencia de mínimo riesgo taxonómico
(barata, no toca el encoder, mejora directa de la abstención) · 2) `clipora` para desbloquear
LoRA · 3) LLRD como receta del fine-tuning del punto 1 del plan · 4) crop+contexto
multi-escala (matiza "el crop no aporta") · 5) DINOv3 como encoder complementario ·
6) FathomNet, solo si se confirma solapamiento de especies.

### Descartado — no repetir (ya medido y es peor que triplet)
MLP/cabeza lineal (69.4%, no bate al kNN a escala completa) · ProjHead (50.9% val) ·
ArcFace standalone · ponderar por confianza de curador (+1.8%±2.9%, no significativo: el
error es visual, no de etiqueta) · crop sin triplet (idéntico a original+triplet, no aporta
por sí solo) · añadir especies bajando `MIN_IMGS` (empeoró ligeramente el global).

---

# ⚠️ ESTADO REAL DE LA IMPLEMENTACIÓN — LEER ESTO PRIMERO (act. 2026-07-30)

> El plan histórico de 2 etapas (detector YOLO + clasificador YOLO-cls) está **más
> abajo, sin borrar**, como registro. **NO se implementó tal cual.** Tras analizarlo se
> construyó un motor **BioCLIP-2 + kNN**, que para fauna marina críptica es superior a
> entrenar un YOLO-cls desde cero. Esta sección es la **fuente de verdad** para continuar.
>
> - **Código y doc operativa detallada:** `./docker/fotofauna-yolo/README.md`
> - **Memoria persistente clave:** `yolofauna-gpu-qwen-contention` (GPU vs qwen).
> - **Todo es LOCAL en HanSolo** (`/mnt`), sin ssh. El contenedor `fotofauna-embed`
>   corre 24/7 (RestartPolicy unless-stopped) y se relanza solo vía `scripts/run_all.sh`.

## A. OBJETIVO REVISADO (2026-07-30)
Meta a largo plazo: que este motor sirva como **IA de identificación de especies para
Minka**. Por eso **NO se recorta el catálogo** — se busca cobertura amplia. El foco
*inmediato* es la fauna marina mediterránea y **en especial los heterobranquios** (lo que
el usuario fotografía), pero el pipeline debe escalar a todo el catálogo Minka.

## B. QUÉ SE CONSTRUYÓ REALMENTE (arquitectura as-built)
- **Embeddings BioCLIP-2** (`hf-hub:imageomics/bioclip-2`, ViT-L, **768-dim**) en GPU
  (RTX 3060). Por especie: `dataset/patterns/<slug>/embeddings.npy` + `prototype.npy`.
- **Decisión por kNN** (k=25 ponderado) sobre todos los embeddings (mejor que
  prototipo-media en crípticas). Fallback a prototipo si <25 muestras.
- **Descarga** `download_species.py`: **Minka PRIMERO** (labels más fiables) + iNat con JWT
  para completar volumen. Resumible (`.done`), manifest de procedencia.
- **Driver** `download_targets.py` (loop `while true`, MIN_OBS=10): recorre
  `dataset/target_species.json`, **ordenado por tier** (ver sección D).
- **Auto-embed 24/7**: hilo `_bg_loop` en `identify_service.py` embebe lo descargado
  (hoy en orden **alfabético**; ver TODO en sección E).
- **API estilo iNaturalist** (pública, vía fotofauna):
  `POST https://fotofauna.yespi.es/vision/yolofauna/identify -F file=@foto.jpg`.
  Interna (contenedor :8090): `/identify`, `/health`, `/species`, `/reload`.
- **Integración con FotoFauna**: `_identify_with_yolofauna` consulta PRIMERO nuestra BBDD
  (umbral `YOLOFAUNA_MIN_CONFIDENCE`=0.90); si supera, usa nuestra ID y arrastra los IDs.
- **Disco**: `archive_processed.py` mueve fotos ya embebidas a `/mnt/archive` dejando una
  muestra de 80 en SSD. Panel admin `fotofauna.yespi.es/admin` → 🦑 YOLOFAUNA (refresco 60s).

## C. IDENTIDAD DE ESPECIE — REGLA CANÓNICA: **MINKA MANDA** (2026-07-30)
Cada especie tiene **3 IDs** que NO coinciden: `slug local ↔ minka_taxon ↔ inat_taxon`.
Directriz del usuario: **el nombre/ID de Minka es el autoritativo.** Reglas:
1. El **nombre científico** de referencia y el **ID canónico** de salida = los de **Minka**
   (`minka_taxon`). El `inat_taxon` se mantiene solo como puente/enriquecimiento.
2. Si Minka e iNat coinciden en el nombre, **gana Minka** como ID.
3. La API debe devolver SIEMPRE `minka_taxon` como identificador principal, e `inat_taxon`
   como secundario. (Pendiente: revisar que `identify_service.py` etiquete Minka como
   principal en la respuesta — ver roadmap.)

## D. PRIORIZACIÓN POR NIVELES (tiers) — decisión 2026-07-30
La lista objetivo creció a **2.989 especies**. Se decidió **priorizar por niveles sin
recortar** (el objetivo Minka exige cobertura amplia). Cada especie de
`target_species.json` lleva un campo **`tier`**:
- **tier 0** = heterobranquios (636) — núcleo, interés directo del usuario.
- **tier 1** = resto fauna marina mediterránea (1.527) — iconic ∈ {Mollusca,
  Actinopterygii, Cnidaria, Malacostraca, Porifera, Annelida, Bryozoa, Echinodermata,
  Elasmobranchii, Ctenophora, Crustacea, Pycnogonida, Animalia, Platyhelminthes}.
- **tier 2** = terrestre / cola larga (826) — Plantae, Aves, Insecta, Fungi, etc.
El driver ordena por `(tier, -max_obs)` → descarga marino y frecuente primero, sin
excluir nada. **Estado hoy: los 636 heterobranquios ya están 100% descargados.**

## E. PRECISIÓN — DIAGNÓSTICO HONESTO (informe de confusión 2026-07-30)
Con 312 especies con patrón, el 55% quedan <80% de auto-consistencia (prototipo argmax).
**No es falta de imágenes** (las peores tienen 500-740 fotos). Causas reales:
1. **Especies crípticas / congéneres casi idénticos** (68/171 se confunden con su MISMO
   género): géneros *Doto*, *Haminoea*, *Favorinus*, *Elysia*, *Aplysia*. *Doto* es un
   sumidero (varias caen en `doto_millbayana`).
2. **Duplicados taxonómicos (sinónimos)**: p.ej. `ambigolimax_valentianus` ≡
   `lehmannia_valentiana` (misma especie con dos nombres) → confusión mutua garantizada.
   **Fusionar sinónimos recupera acierto gratis.** Buscar más casos.
3. **Contaminación terrestre**: caracoles/babosas terrestres (xerosecta, cernuella,
   solatopupa, parthenina, turbonilla) colados e inflando el <80%. Los tiers los relegan.
Herramienta: `scripts/analyze_acc.py` (correr con `sed 's#/work#./docker/fotofauna-yolo#'`
desde code-server; numpy local disponible). `test_knn.py` = versión kNN.

## F. ABSTENCIÓN JERÁRQUICA A **FAMILIA** — regla de producto (2026-07-30)
Directriz del usuario: **si no podemos distinguir con seguridad la especie, devolver la
FAMILIA** (que suele ser común entre las confundibles) en vez de arriesgar una especie mal.
Diseño previsto:
- Si el top-1 y top-2 del kNN son de la **misma familia** con scores próximos (margen
  bajo), la respuesta baja a nivel **familia** con alta confianza + top-3 especies como
  candidatas (estilo iNat "estamos bastante seguros de que es la familia X").
- **Requisito de datos**: hoy `target_species.json` NO tiene `family`. Hay que enriquecer
  cada especie con su **familia (y género) desde la taxonomía de Minka** (ancestros del
  `minka_taxon`), guardarla en el manifest/target, y usarla en `identify_service.py` para
  la abstención. Género ya es derivable del nombre; **familia requiere la llamada a Minka**.

## ✅ IMPLEMENTADO 2026-07-31
- **Enriquecimiento taxonómico** (`enrich_taxonomy.py`): familia+género desde Minka en
  `target_species.json` (2983/2989). Sección C/F desbloqueadas.
- **Minka manda**: `/identify` devuelve `prediction` con `id=minka_taxon`, `id_source="minka"`.
- **Abstención en 3 NIVELES** (especie→género→familia): si top-1/top-2 son congéneres y
  margen<0.06 devuelve el **género** (más informativo); si solo comparten familia, la familia.
  `YOLOFAUNA_FAMILY_MARGIN`=0.06. Validado out-of-sample (Doto koenneckeri no visto → género Doto). Antiguo: - **Abstención a familia** (sección F): activa. `YOLOFAUNA_FAMILY_MARGIN`=0.06. Validado
  out-of-sample (Doto koenneckeri no visto → antes *millbayana* errónea, ahora **Dotidae** OK).
- **Tiers** (sección D): driver ordena `(tier,-max_obs)`; heterobranquios 100% descargados.
- **Benchmark OUT-OF-SAMPLE real** (`eval_field.py`, fotos recientes de Minka NO vistas):
  **especie ~68-75%**, **género ~81%**, **familia ~83%** (abstención ~20-27%, según muestra).
  Por tier: heterobranquios (0) ~71%/82%, marino no-hetero (1) ~89%. Confirma: el acierto de
  campo < in-sample (77%); la abstención a género/familia eleva el resultado usable a ~81-83%.
  Los heterobranquios son los más crípticos.
- PENDIENTE aún: ordenar el AUTO-EMBED por tier (hoy alfabético; requiere reinicio cuando
  entren no-hetero); fusión de sinónimos; MLP/fine-tuning con BBDD estable; priors GPS+fecha.

## ✅ INTEGRACIÓN EN FOTOFAUNA — YOLOFauna-first en desktop (2026-07-31)
**Problema medido** (logs `fauna_api` del 30-jul): la cola de identificación de **desktop**
usaba SOLO iNat CV (`/vision/inat-score`) e ignoraba YOLOFauna. **931/1225 (76%)** de las
llamadas a iNat `score_image` fueron **429 (rate-limit)** → reintentos con backoff (hasta 8)
× worker en serie → "10 fotos tardan un minuto" y toasts sin parar.

**Arreglo** (`backend/vision_routes.py`, montado → editar+reiniciar `fauna_api`):
- `/vision/inat-score` prueba **YOLOFauna local PRIMERO** (rápido, sin rate-limit, fuera del
  semáforo iNat); si acierta ESPECIE con conf ≥ `YOLOFAUNA_MIN_CONFIDENCE` (0.90), devuelve
  esa ID con forma iNat-CV (`results[0].taxon`) + `meta.source="yolofauna"` y **no llama a iNat**.
- Abstención (género/familia) en desktop → cae a iNat CV (no forzamos especie dudosa).
- **fail-fast iNat**: 2 intentos, espera corta (antes 3 con backoff 4/8/16 s).
- Frontend (`use-vision-pipeline.js`, ambos public y public-pre; `/composables/` es no-cache):
  `_submitInatScore` lee `meta.source` → atribuye `source='yolofauna'` + `minka_taxon_id`.

**Resultado (Playwright E2E, `scripts/pw_yf_bench.js`, pila pública, fotos NO vistas):**
- Latencia mediana **334 ms** (p90 1.4 s) vs iNat (segundos + 429).
- Acierto especie **77%**; **win-rate 51%** (mitad de fotos resueltas en local → mitad de
  llamadas a iNat eliminadas). En crípticos puros: 60% especie, win 33%.
- **Duelo vs iNat CV** (`scripts/duel_inat.py`, 24 crípticas mediterráneas no vistas):
  **YOLOFauna 83.3% vs iNat 70.8%** (95.8% a género). Ganamos en nuestro terreno.

**Pendiente/palancas:** bajar `YOLOFAUNA_MIN_CONFIDENCE` a ~0.85 subiría el win-rate (ya
ganamos a iNat, así que ser más agresivos es defendible); usar la abstención a género para
resolver más en local sin iNat; `deploy-to-pro.sh` falla en code-server (paso SEO llama a
`docker` CLI inexistente) → los `/composables/` se copiaron a mano (no-cache = live).

## M. SESIÓN 2026-08-02 — Geo priors, recalibración, optimizaciones

### Geo priors (PARCIAL)
- **1088/1369 especies** con coordenadas (70185 pts). Fuente: backup iNat.
- Minka-first para las 281 restantes → build script inestable (se corrompe al escribir).
- Archivo: `dataset/geo_priors.json` (backup en `geo_priors_inat_backup.json`).
- `identify_service.py` ya carga y usa geo priors (`_geo_prior`, haversine, GEO_BOOST=2.0, GEO_SIGMA_KM=200).

### Recalibración (2026-08-02 ~22:30)
- `fit_calib.py` re-ejecutado con 2078 muestras (`calib_raw.jsonl`).
- Modelo: logistic species (full features). p>=0.90 → **95.6% precision** (320 muestras).
- Umbrales: p90=0.68, p95=0.90, p98=N/A.

### GPS en identificacion interactiva
- `_yolofauna_suggestion` ahora envia lat/lon/date a YF `/identify`.
- Antes solo el wave autoid usaba geo priors.

### Optimizaciones API (iNat -99%)
- Autocomplete taxa: YF-local (2989 spp, 0 API) + Minka-first, iNat solo fallback.
- Fotos especie: 100% locales (4734 thumbnails, sin iNat).
- Wave autoid: 30/h -> 10/h.
- Geo priors build: Minka-first, iNat ultra-light fallback (1 pagina, 5s timeout).

### Miniaturas locales
- 1558 especies con 3 miniaturas (dataset YF, 0 APIs).
- 4734 thumbnails totales en `/img/species_thumbs/`.

### Pendiente
- Completar geo priors con Minka (281 spp) — script `build_geo_priors.py` necesita debug.
- Fine-tuning encoder BioCLIP (GPU-dias, rompe el techo del ~67%).
- Re-cosechar calibracion con geo priors activos.


## G. ROADMAP DE PRECISION (orden por impacto real) — ACTUALIZADO 2026-08-02
1. ✅ **Fusion de sinonimos** (gratis): detectar y unir especies duplicadas. Caso: `ambigolimax_valentianus` ≡ `lehmannia_valentiana` fusionado.
2. ✅ **Abstencion a familia/genero** (seccion F): convierte fallos en aciertos utiles. Activa con `YOLOFAUNA_FAMILY_MARGIN=0.06`.
3. 🔄 **Priors contextuales** (GPS+fecha): PARCIAL. 1088/1369 especies con coordenadas. Geo scoring activo en desktop y wave. 281 spp pendientes con Minka.
4. 🔜 **Cabeza MLP** sobre embeddings: medido (+3.4 pt con 300-440 spp), NO bate al kNN a escala completa (69.4% con 1257 spp). Descartado como mejora independiente. El encoder afinado sí puede combinarse con MLP luego.
5. 🔜 **Fine-tuning del encoder** BioCLIP (metric learning, GPU-dias). Mayor techo de mejora para criticas.
6. 🔜 **VLM re-ranker** solo sobre el top-3 critico (rasgos diagnosticos).
NO ayudan: COCO (fauna marina→"pizza"), YOLO como clasificador (solo sirve para recortar).

## H. CÓMO CONTINUAR (checklist para otra IA)
1. Leer este bloque + `./docker/fotofauna-yolo/README.md` + memoria
   `yolofauna-gpu-qwen-contention`. Trabajar **local**, sin ssh.
2. Verificar que el contenedor vive y el auto-embed avanza:
   `curl -s --unix-socket /var/run/docker.sock http://localhost/containers/fotofauna-embed/json | python3 -c 'import sys,json;print(json.load(sys.stdin)["State"]["Status"])'`
   y `ls dataset/patterns/*/prototype.npy | wc -l` (debe subir con el tiempo).
3. **NO reiniciar el servicio sin motivo** — el auto-embed 24/7 se corta. Si hay que tocar
   `identify_service.py`, conservar la línea `threading.Thread(target=_bg_loop...).start()`
   (se perdió una vez y costó horas).
4. GPU: la 3060 (12GB) NO admite qwen (~11GB) + BioCLIP a la vez. El servicio suprime qwen
   mientras embebe. Ver memoria.
5. Tareas abiertas y su estado: `/mnt/docs/TAREAS_PENDIENTES.md`.
6. **TODO pendientes concretos**:
   - Ordenar el auto-embed por `tier` (hoy alfabético) — requiere reinicio; hacerlo cuando
     empiecen a entrar no-heterobranquios en la cola.
   - Enriquecer `target_species.json` con `family`/`genus` desde Minka (sección F).
   - Etiquetar Minka como ID principal en la respuesta de `/identify` (sección C).
   - Fusionar sinónimos (sección E.2).

---

# PLAN DE IMPLEMENTACIÓN TÉCNICA: PIPELINE Y TRAINING DE DETECCIÓN Y CLASIFICACIÓN DE ESPECIES CON YOLO (FOTOFAUNA)

## 1. OBJETIVO DEL PROYECTO

**Objetivo final**: Entrenar un modelo de clasificación de especies (YOLO-cls / YOLO11-cls) que haga de **motor de identificación propio de FotoFauna**, reemplazando la dependencia actual de iNaturalist CV (API externa).

Esto permitirá:
- Identificar especies **sin depender de internet ni de APIs externas**
- Enfocar el modelo en las **~1.000 especies de Cataluña** presentes en Minka-sdg
- Escalar progresivamente hasta **~2.000 especies** según espacio en disco y tiempo de entrenamiento
- Tener control total sobre la precisión, los datos de entrenamiento y las actualizaciones

**Arquitectura**: Pipeline de **2 etapas**:
1. **Detector generalista** (YOLO11s, 1 clase: `animal`) — localiza el animal en la imagen
2. **Clasificador de especie** (YOLO11-cls, ~1.000 clases) — identifica la especie exacta sobre el recorte

**Fuente de datos principal**: Minka-sdg (observaciones de Cataluña, datos curados, calidad research). iNaturalist como complemento para especies con pocas imágenes en Minka.

El sistema debe ser optimizado para ejecutarse en un entorno local con GPU NVIDIA RTX 3060 (12 GB VRAM) y 32 GB de RAM.

---

## 2. ESPECIFICACIONES Y RESTRICCIONES TÉCNICAS (HARDWARE & SOFTWARE)
- **Entorno GPU:** NVIDIA RTX 3060 (12 GB VRAM)
- **RAM Sistema:** 32 GB DDR4/DDR5
- **Almacenamiento Local/Servidor:** SSD/HDD secundario con al menos 250 GB libres para descarga e intermediación de datos.
- **Formato de Salida en Producción:** **PyTorch nativo (`.pt`)** en primera versión. Exportar a ONNX/TensorRT solo si el volumen supera 1.000 fotos/día o la latencia es >1s (ver sección 7.6). El formato `.pt` es directo, fácil de depurar en `fauna_api` y para el volumen actual (~200 fotos/día, ~200ms inferencia) es más que suficiente.
- **Limitación Clave:** Evitar el Olvido Catastrófico (*Catastrophic Forgetting*) mediante un diseño modular o el mantenimiento de un buffer de retención (*Core-set*).

---

## 3. ARQUITECTURA PROPUESTA: PIPELINE DE 2 ETAPAS (RECOMENDADO)
Para facilitar la escalabilidad futura (ej. añadir 50 especies nuevas sin tener que reentrenar el detector completo), se establece una arquitectura desacoplada:

1. **Etapa 1: Detección Generalista (Bounding Box)**
   - **Módulo:** YOLO11m / YOLOv8m o MegaDetector v5.
   - **Función:** Detectar la presencia de un "animal" en la imagen, recortar el área (Crop) y pasar la región de interés (RoI) a la Etapa 2.
   - **Clases del Detector:** 1 única clase general (`animal`).

2. **Etapa 2: Clasificación de Especie (Especie exacta)**
   - **Módulo:** YOLO-cls (YOLO11-cls / EfficientNet / ConvNeXt).
   - **Función:** Clasificar el recobre (Crop) en una de las ~1.000 especies objetivo.
   - **Ventaja:** Permite reentrenar o extender fácilmente el catálogo de especies con un coste computacional mucho menor y menor riesgo de degradar la localización.

*(Nota: En caso de optar por un pipeline de 1 solo paso con YOLO detector multiclase directo, ver Sección 6 para la gestión del buffer de retención).*

---

## 4. FASES DE IMPLEMENTACIÓN

### FASE 1: DESCARGA Y FILTRADO DE DATOS (Minka / iNaturalist)
- **Script:** `01_download_data.py`
- **Requisitos:**
  - Consumir la API pública de iNaturalist/GBIF o procesar el export de Minka-sdg.
  - Filtros obligatorios: `quality_grade = research`, taxonomía acotada (por ejemplo, Aves, Mamíferos, Reptiles de la región de interés).
  - Límite de imágenes: Seleccionar hasta **1.000 imágenes por especie**.
  - **Optimización de Almacenamiento Instantánea:** Redimensionar la imagen descargada a una resolución máxima de **640x640 px** en memoria antes de guardar en disco en formato JPEG (calidad 85-90). Esto reduce el tamaño total del dataset a ~200 KB por foto (~200 GB para 1M de fotos).

### FASE 2: PREPROCESAMIENTO Y GENERACIÓN DE ANOTACIONES
- **Script:** `02_prepare_dataset.py`
- **Requisitos:**
  - Ejecutar auto-etiquetado sobre las imágenes descargadas usando MegaDetector v5 o YOLO preentrenado para generar automáticamente los Bounding Boxes.
  - Generar la estructura estándar de Ultralytics:
    ```
    dataset/
      ├── dataset.yaml
      ├── images/
      │     ├── train/
      │     └── val/
      └── labels/
            ├── train/
            └── val/
    ```
  - Split balanceado: 80% Train / 20% Val.
  - Generar el archivo `dataset.yaml` mapeando los IDs de clase a los nombres científicos de las especies.

### FASE 3: PIPELINE DE ENTRENAMIENTO OPTIMIZADO (RTX 3060)
- **Script:** `03_train_model.py`
- **Configuración de Entrenamiento (PyTorch / Ultralytics):**
  - **Modelo base:** `yolo11m.pt` o `yolo11m-cls.pt`.
  - **Resolución (`imgsz`):** 640
  - **Batch Size (`batch`):** 16 o 32 (Ajustar dinámicamente según VRAM).
  - **Precision Mixta (`half`):** `True` (FP16 para minimizar uso de VRAM y acelerar cómputo).
  - **Workers (`workers`):** 8 (aprovechando los 32 GB de RAM para acelerar la carga de lotes).
  - **Patience:** 15 (Early stopping para evitar sobreajuste).
  - **Guardado de Checkpoints:** Almacenar checkpoints finales en el directorio especificado del servidor (`/servidor/models/fotofauna`).

### FASE 4: OPTIMIZACIÓN, EXPORTACIÓN Y LIMPIEZA
- **Script:** `04_export_and_cleanup.py`
- **Pasos:**
  1. Validar el modelo en el conjunto de test/val y registrar métricas (mAP50, mAP50-95, Top-1 / Top-5 Accuracy).
  2. Exportar el modelo entrenado a formato **ONNX** y/o **TensorRT** (`half=True`).
  3. **Generación del Buffer de Retención (Core-set):**
     - Seleccionar y conservar un subconjunto de **100 imágenes representativas por especie** (~20 GB total) junto con sus archivos `.txt` de etiquetas.
     - Este buffer se almacenará de forma permanente en el servidor para futuros reentrenamientos escalables (prevención de olvido catastrófico al agregar nuevas especies).
  4. **Limpieza:** Eliminar el resto de imágenes masivas descargadas (liberando ~180 GB de almacenamiento) manteniendo únicamente el modelo exportado (~50-90 MB) y el buffer de retención.

### FASE 5: INTEGRACIÓN EN EL BACKEND DE FOTOFAUNA (FastAPI)
- **Script / Módulo:** `app/services/detector.py`
- **Requisitos:**
  - Carga diferida (*lazy loading*) o al inicio de la aplicación del modelo ONNX/TensorRT en la GPU.
  - Endpoint REST API en FastAPI que reciba una imagen procesada por Fotofauna.
  - Ejecución de la inferencia, post-procesamiento (NMS - Non-Maximum Suppression) y retorno de respuesta estructurada en JSON con: `especie`, `confianza`, `bbox` [x, y, w, h] y `taxon_id`.

---

## 5. INSTRUCCIONES DE EJECUCIÓN PARA EL AGENTE DE CÓDIGO
Por favor, implementa los scripts descritos (`01_download_data.py`, `02_prepare_dataset.py`, `03_train_model.py`, `04_export_and_cleanup.py` y el servicio de inferencia para FastAPI) siguiendo buenas prácticas de programación en Python 3.10+:
- Utilizar `pathlib` para manejo de rutas.
- Manejar excepciones y reintentos en descargas de red.
- Incluir barras de progreso con `tqdm`.
- Utilizar la librería oficial `ultralytics` para el entrenamiento y exportación del modelo.

---

## 6. UBICACIÓN Y GESTIÓN DE DATOS (DECISIÓN DE ARQUITECTURA)

### 6.1. Almacenamiento durante el entrenamiento

- **Ubicación de descarga**: `./docker/fotofauna-yolo/dataset/` (SSD rápido, 279 GB libres)
- **Fuente de datos**: **Minka-sdg** (prioritario) + iNaturalist como complemento. Minka-sdg tiene datos más limpios y curados para la región de interés.
- **Volumen inicial**: 500 imágenes por especie × 500 especies ≈ 250.000 imágenes
- **Compresión en descarga**: Redimensionar a 640×640 px y guardar en JPEG calidad 85 inmediatamente al bajar cada foto (nunca guardar el RAW original). Esto reduce el peso medio a ~200 KB/imagen.
- **Tamaño total estimado del dataset**: 250.000 imágenes × 200 KB ≈ 50 GB

### 6.2. Estrategia de conservación y movimiento

Una vez que una especie ha sido procesada (imágenes descargadas, auto-etiquetadas, validadas), se pueden mover sus fotos a almacenamiento más lento pero más abundante:

1. **Durante el entrenamiento activo**: las imágenes de la especie actual residen en `./docker/fotofauna-yolo/dataset/` (SSD rápido, para acelerar la carga en training).
2. **Especie completada**: mover imágenes + etiquetas a `/mnt/archive/fotofauna-yolo/dataset/` (HDD 593 GB libres). Esto libera espacio en SSD para la siguiente tanda de especies.
3. **Buffer de retención (Core-set)**: Conservar permanentemente **100 imágenes representativas por especie** (~100 MB por especie, ~50 GB para 500 especies) en el SSD para futuros reentrenamientos incrementales.

### 6.3. Estimación de espacio

| Fase | SSD `/mnt/docker` | HDD `/mnt/archive` |
|------|-------------------|-------------------|
| Descarga activa (10 especies simultáneas) | ~1 GB | - |
| Dataset completo antes de mover | 50 GB temporal | - |
| Después de mover a archive | - | 50 GB |
| Buffer de retención permanente (SSD) | 5-10 GB | - |
| Modelos exportados | 200 MB | - |
| **Total necesario** | **~10 GB permanentes + 50 GB temporales** | **50 GB** |
| **Disponible** | **279 GB ✅** | **593 GB ✅** |

### 6.4. Flujo de trabajo por lotes

```
1. Descargar 10 especies → ./docker/fotofauna-yolo/dataset/
2. Auto-etiquetar con MegaDetector
3. Entrenar/incluir en el modelo
4. Mover a /mnt/archive/fotofauna-yolo/dataset/
5. Conservar 100 imágenes representativas en SSD (buffer)
6. Repetir con las siguientes 10 especies
```

Este flujo por lotes garantiza que nunca se ocupa más de ~5 GB del SSD simultáneamente.

---

## 7. APUNTES DEL AGENTE — REVISIÓN OBJETIVA TRAS ANALIZAR EL CÓDIGO EXISTENTE

> ⚠️ **SECCIÓN HISTÓRICA (julio 2026), NO es el estado actual.** Se conserva como registro
> del análisis que llevó a descartar el pipeline YOLO-cls. Las menciones a YOLO11* de abajo
> están **desfasadas**: ver la nota 7.0. El estado real está en las secciones A–K de arriba.

### 7.0. ACLARACIÓN DE VERSIONES (act. 2026-08-01)

Dos cosas distintas que el resto de esta sección mezcla:

1. **YOLOFauna (identificación de especie) NO usa YOLO.** Es **BioCLIP-2 (ViT-L, 768-dim) +
   kNN**. El nombre del proyecto es histórico. Ningún modelo YOLO interviene en `/identify`.
2. **El detector de bounding boxes de FotoFauna sí es YOLO, y ya está en YOLO26.**
   `backend/vision_detect.py` carga `os.getenv("FAUNA_YOLO_MODEL", "yolo26n.pt")` con
   `yolo11n.pt` solo como **fallback** si el primero no está. Ambos `.pt` viven en
   `./docker/ecosistema-fauna/backend/`. YOLO26 es NMS-free → `.fuse()` es best-effort.
   (Existe además `dataset/yolo26n-seg.pt` en `fotofauna-yolo/`, **no referenciado por
   ningún script** — candidato a borrar o a usar para recorte por máscara.)

Donde abajo ponga "YOLO11n en producción", léase **YOLO26n**, y solo para *detectar/recortar*.

### 7.1. Ya hay un detector YOLO funcionando en producción ~~(YOLO11n)~~ → hoy YOLO26n

El backend de FotoFauna ya ejecuta un detector YOLO en CPU para detección de fauna, integrado
en el pipeline `POST /vision/detect`. En julio era `yolo11n.pt` (5.4 MB); **hoy es
`yolo26n.pt`**, con `yolo11n.pt` como fallback.

**Implicación**: No partimos de cero. Cualquier modelo nuevo debe:
- Coexistir con el actual (código ya existe en `vision_detect.py`)
- Mantener la misma interfaz de entrada/salida
- Ser sustituible sin cambiar el pipeline de identificación (iNaturalist CV, Gemini, etc.)

### 6.2. GPU NO disponible en el contenedor de fauna_api

El contenedor `fauna_api` corre con `runtime: runc`, no `nvidia`. `torch.cuda.is_available()` devuelve `False`.

**Para entrenar**: Hay que lanzar el training desde otro lugar (ej. script en el host, o contenedor ad-hoc con runtime nvidia). El contenedor de karaoke (`karaoke-worker-1`) ya tiene runtime nvidia y GPU libre cuando no está procesando canciones.

**Para inferencia**: YOLO11n en CPU tarda ~200ms por imagen. Un modelo más grande (YOLO11m) en CPU sería 3-5x más lento. Para usar GPU en inferencia, hay que añadir `runtime: nvidia` al docker-compose de fauna_api.

### 6.3. El almacenamiento disponible NO es 250 GB

```
/mnt/archive: 409 GB totales, ~110 GB ocupados por backups, ~35 GB fotos, ~14 GB música
/mnt/fast: 16 GB (SSD rápido, para datos temporales)
```

**No hay 250 GB libres en ningún sitio.** El dataset de 1M de fotos (~200 GB) no cabe cómodamente. **Alternativa realista:**
- Limitar a **200 imágenes por especie** (~200 MB por especie → ~200 MB × 1.000 = 200 GB, justo en el límite)
- O reducir a **500 especies prioritarias** (las más fotografiadas en la región)
- Usar `/mnt/fast/tmp/` como almacenamiento temporal durante el entrenamiento (solo 16 GB, así que procesar en lotes)

### 6.4. Ya existe BioCLIP como clasificador zero-shot

El microservicio `bioclip/` (basado en `imageomics/bioclip-2`) ya clasifica especies mediante zero-shot learning. Soporta ~1.000 especies y tiene su propio build de taxa desde iNaturalist España.

**El pipeline de 2 etapas propuesto (detector + clasificador YOLO) COMPITE directamente con BioCLIP.** BioCLIP ya hace clasificación zero-shot sin necesidad de entrenar. YOLO-cls requeriría entrenamiento, datos etiquetados y mantenimiento.

**Recomendación:**
- Usar BioCLIP como clasificador de especie (ya existe, ya funciona)
- Entrenar SOLO el detector generalista (1 clase: `animal`) como mejora del YOLO11n actual
- El clasificador YOLO-cls solo tendría sentido si BioCLIP no funciona bien (hay que medirlo)

### 6.5. YOLO11m no cabe en 12 GB VRAM para entrenar

| Modelo | VRAM training batch 16 | VRAM training batch 8 | VRAM training batch 4 | VRAM inference |
|--------|----------------------|---------------------|---------------------|----------------|
| YOLO11n | ~4 GB | ~2.5 GB | ~2 GB | ~1 GB |
| YOLO11s | ~6 GB | ~4 GB | ~3 GB | ~1.5 GB |
| YOLO11m | **~12 GB** | ~8 GB | ~6 GB | ~2.5 GB |
| YOLO11l | ~18 GB | ~12 GB | ~9 GB | ~4 GB |

**Con RTX 3060 (12 GB):** YOLO11m solo entrenaría con batch=4 y recortando a 512px. **Opción recomendada: YOLO11s** (balance velocidad/calidad).

### 6.6. El formato ONNX/TensorRT no es necesario para el volumen actual

FotoFauna procesa ~100-200 fotos/día. Con 200ms por inferencia en CPU, YOLO11n tarda ~40 segundos acumulados al día. No hay cuello de botella.

**ONNX/TensorRT solo tendría sentido si el volumen supera 1.000 fotos/día o si se integra en tiempo real.** Mientras tanto, mantener YOLO en formato PyTorch nativo es más sencillo y fácil de actualizar.

### 6.7. El pipeline actual ya usa fallbacks para detección

Cuando YOLO11n no detecta nada (común en insectos, fauna marina), el pipeline cae a Gemini/Groq/OpenRouter para obtener bounding boxes. Esto funciona pero:
- Depende de APIs externas (latencias de red, rate limits)
- Añade coste si se superan los límites gratuitos
- **Un detector local entrenado específicamente para la fauna local eliminaría esta dependencia**

### 6.8. Prioridad real: mejorar el detector, no crear un clasificador

Basado en el estado actual del código y los backlogs:

| Prioridad | Qué | Por qué |
|-----------|-----|---------|
| 🔴 Alta | Entrenar detector 1-clase (`animal`) con YOLO11s | Elimina los fallbacks a APIs externas. Mejora detección de insectos y fauna marina |
| 🟡 Media | Migrar inferencia a GPU (añadir runtime nvidia al compose) | Acelera 10x la inferencia. Permite usar modelos más grandes sin penalización |
| 🟢 Baja | Clasificador YOLO-cls propio | BioCLIP ya cubre esta necesidad. Solo si BioCLIP no cumple |

### 6.9. Estimación de esfuerzo realista

| Fase | Tiempo estimado | Dependencias | Riesgos |
|------|----------------|-------------|---------|
| Descarga datos (200 img/especie × 500 especies) | 2-3 días (rate limit iNat: 10k requests/día) | API key iNat, espacio en disco | Rate limits, names changes |
| Auto-etiquetado con MegaDetector | 1 día | MegaDetector v5 (pesa ~400 MB) | Falsos positivos en fondos complejos |
| Entrenamiento YOLO11s (10 épocas, 500 especies) | 4-6 horas GPU | GPU libre (karaoke-worker-1) | VRAM, overfitting en especies con pocas muestras |
| Exportación + limpieza | 2-4 horas | ONNX/TensorRT | Dependencias CUDA en el contenedor |
| Integración en FastAPI | 4-8 horas | Acceso a GPU desde fauna_api | Runtime nvidia, pruebas |
| **Total** | **~1 semana (dedicado)** | | |

### 6.10. Alternativa pragmática (no implementar el plan completo)

Dado que:
1. BioCLIP ya clasifica especies (zero-shot, sin entrenar)
2. YOLO11n ya detecta animales (limitado a COCO-80)
3. El servidor tiene GPU infrautilizada (RTX 3060, 12 GB)
4. Los contenedores de karaoke ya tienen runtime nvidia

**Estrategia recomendada (objetivo real: reemplazar iNaturalist CV):**

1. ✅ **Fase 1 — Detector generalista**: Entrenar YOLO11s (`animal`, 1 clase) con datos de Minka-sdg. Reemplaza al YOLO11n COCO-80 actual y elimina los fallbacks a APIs externas para detección.
2. ✅ **Fase 2 — Clasificador de especie**: Entrenar YOLO11-cls con las ~1.000 especies de Minka-sdg. Este es el **corazón del proyecto**: será el sustituto local de iNaturalist CV.
3. 🔧 **Infraestructura**: Añadir runtime nvidia al contenedor `fauna_api` para que tanto el detector como el clasificador usen GPU en inferencia.
4. 📦 **Exportación**: ONNX/TensorRT solo si el volumen de peticiones lo requiere (>1.000 fotos/día o latencia >1s).
5. 📈 **Escalado**: Las imágenes originales de entrenamiento se mueven a `/mnt/archive` tras entrenar cada especie, conservando solo el buffer de retención (100 imágenes/especie) en SSD.

**BioCLIP queda como contingencia** — si el clasificador YOLO-cls no reconoce una especie (confianza baja), se puede caer a BioCLIP para zero-shot, y de ahí a iNaturalist CV como último recurso.


# I. SIGUIENTES PASOS — FASE 2 (mejorar aciertos) — GUÍA PARA CONTINUAR (otra IA)
**Estado 2026-08-01: BBDD CONSOLIDADA** = 1257 patrones (toda la fauna med con ≥20 fotos).
Embed en reposo (cola embebible=0). 3 especies algas/hierba aparcadas en
`./docker/fotofauna-yolo/dataset/_parked_oom/` (reversibles: devolver los .jpg a
`dataset/images/<slug>/`). Wave auto-publish Minka **PAUSADO** (`autoid_schedules` id=1).
Acierto actual (campo, out-of-sample): ~~68-77% especie · ~81% género · ~83% familia~~
**DESFASADO — esos números tenían fuga de datos. Reales: 66.7% especie · 72.6% género ·
77.3% familia (ver sección L.0).**
La cola larga (402 especies con <20 fotos) NO se puede embeber sin más imágenes.

**Objetivo fase 2: subir acierto y CALIBRAR la confianza** (para auto-publicar sin meter
errores en Minka). Pasos por impacto:

1. **Cabeza discriminativa MLP** sobre embeddings 768-dim. Base: `scripts/test_head.py`
   (mide +3.4 pt → ~80% in-sample). Entrenar UNA vez con la BBDD estable; guardar
   `dataset/mlp_head.pt` + `labels.json`. Ayuda: separa congéneres crípticos mejor que kNN.
2. **Fine-tuning del encoder BioCLIP** (metric learning: triplet/ArcFace, enfocado en los
   pares crípticos que saca `scripts/analyze_acc.py`). GPU-días. Es el mayor techo de mejora
   para crípticas. Guardar encoder afinado y **regenerar embeddings**.
3. **Integrar en `identify_service.py`** (ensemble kNN+MLP; embeber con el encoder afinado).
   Mantener la abstención género/familia ya existente.
4. **CALIBRAR la confianza** (temperature/Platt scaling sobre held-out) → que "0.90" ≈ 90%
   de acierto real. CLAVE: la similitud coseno NO es probabilidad; sin calibrar, el
   auto-publish no es fiable (ver caso Peltodoris→Chondrosia 0.938 erróneo).
5. **Reactivar wave** (`UPDATE autoid_schedules SET enabled=true WHERE id=1`) SOLO tras
   calibrar, con el safeguard iNat ya puesto (`WAVE_YOLOFAUNA_REQUIRE_INAT=1`). Empezar con
   umbral alto y bajarlo con datos reales.
6. **Palancas extra** (menor esfuerzo): fusión de sinónimos (analyze_acc detecta duplicados
   tipo ambigolimax≡lehmannia); priors GPS+estación en la API; recuperar las 3 aparcadas y
   la cola larga cuando haya más fotos.

**Cómo probar sin reentrenar:** subir fotos de especies conocidas en FotoFauna → el panel
Identificar muestra el top-k 🔷 YF; o `scripts/duel_inat.py` (YOLOFauna vs iNat CV).


## J. FASE 2 EN CURSO (2026-08-01)
- **MLP head lanzado**: `scripts/train_head.py` (exec detached en `fotofauna-embed`). Mide
  kNN vs MLP en held-out 80/20 y **guarda `dataset/mlp_head.pt` + `dataset/mlp_labels.json`**.
  Resultado/estado en `logs/train_head.log` (el kNN baseline es lento en CPU; el MLP va en GPU).
- **Siguientes (para continuar):** 1) verificar `logs/train_head.log` (MLP% > kNN%?) y que
  existe `dataset/mlp_head.pt`; 2) integrar el MLP en `identify_service.py` como ensemble
  (kNN + MLP) sobre el embedding de la foto, manteniendo abstención género/familia; reiniciar
  el contenedor (con cuidado, resumible); 3) **calibrar** (temperature scaling) para que la
  confianza sea fiable; 4) reactivar wave Minka con safeguard. Ver sección I.


## L. CALIBRACIÓN DE LA CONFIANZA (2026-08-01) — ✅ HECHA Y DESPLEGADA

### L.0. ⚠️ CORRECCIÓN IMPORTANTE: `eval_field.py` medía con FUGA DE DATOS
`eval_field.py` asumía que "como descargamos las observaciones más antiguas primero
(`order=asc`), las recientes casi seguro NO están en la BBDD". **Es falso.** Al excluir por
`obs id` contra `_manifest.jsonl` se midió que **23.507 de 25.523 (92%) de las observaciones
"recientes" YA estaban en la BBDD** — porque la mayoría de especies tienen menos
observaciones que el tope de descarga, así que `asc` se las baja *todas*, recientes incluidas.

**Consecuencia: los números 68-77% de especie de las secciones E/I estaban inflados.**
Medido sobre un set limpio de verdad (2.016 fotos, 904 especies, exclusión por manifest):

| Nivel | Acierto real out-of-sample |
|-------|---------------------------|
| Especie top-1 | **66.7%** |
| Género | **72.6%** |
| Familia | **77.3%** |

Por tier: 0 (heterobranquios) 65% · 1 (marino no-hetero) 66% · 2 (terrestre) 72%. Ojo: el
~89% que la sección I atribuía al tier 1 era fuga, no habilidad.

### L.1. Qué se construyó
- **`scripts/dexec.py`** — ejecuta comandos dentro del contenedor vía el socket de Docker
  (en code-server NO hay CLI `docker`). `python3 scripts/dexec.py [-d] <cont> <cmd...>`.
  Reutilizable para cualquier trabajo futuro en `fotofauna-embed`.
- **`scripts/harvest_calib.py`** — cosecha el set de calibración honesto: baja fotos de Minka
  **excluyendo por `obs id`** las ya usadas, puntúa contra la BBDD completa replicando
  exactamente la decisión de `identify_service.py`, y guarda features en
  `dataset/calib_raw.jsonl` (resumible). Embebe en **CPU** a propósito: no compite por la GPU.
  ~86 min para 2.016 muestras.
- **`scripts/fit_calib.py`** — ajusta y elige calibrador por **NLL** (regla de puntuación
  propia; penaliza el exceso de confianza, que es justo nuestro fallo). Split **por especie**
  (sin solape). Calibra **tres niveles**: especie/género/familia. Guarda
  `dataset/calibration.json`. Log en `logs/fit_calib.log`.
- **Integración en `identify_service.py`**: `load_calib()` + `_calibrate()`; `/identify`
  devuelve `prediction.p_species` (probabilidad calibrada) y `prediction.calibrated=true`.
  `/reload` recarga también la calibración. **`confidence` se mantiene = similitud coseno**
  por compatibilidad con el frontend. Si `calibration.json` no existe, el servicio funciona
  como antes (degradación limpia).

### L.2. Resultado — la confianza estaba rota, ahora no
Held-out (590 muestras, especies no vistas en el ajuste). ECE = distancia media entre la
confianza declarada y el acierto real:

| Modelo | NLL | **ECE** | Brier | AUC |
|--------|-----|---------|-------|-----|
| `raw` (= lo que devolvía la API) | 0.765 | **0.221** | 0.254 | 0.708 |
| `platt_s1` | 0.592 | 0.081 | 0.202 | 0.708 |
| **`full` (elegido)** | **0.436** | **0.048** | **0.141** | **0.854** |
| `full+iso` | 0.439 | 0.034 | 0.141 | 0.851 |

La similitud coseno se desviaba **22 puntos** del acierto real (de ahí el 0.938 erróneo del
caso Peltodoris→Chondrosia). Calibrada: declarada 0.955 → real 0.960. Y el AUC sube de 0.71 a
**0.85**: las features del kNN (margen, votos, share, nº de referencias) discriminan mucho
mejor que la similitud sola.

### L.3. La tabla que importa — umbral → precisión REAL (nivel especie)
| Umbral `p_species` | Cobertura | Precisión real |
|--------------------|-----------|----------------|
| ≥ 0.70 | 56% | 89.5% |
| ≥ 0.75 | 52% | 92.1% |
| ≥ 0.80 | 46% | 93.8% |
| **≥ 0.86** | **40%** | **95%** |
| ≥ 0.90 | 29% | 96.0% |
| ≥ 0.98 | 2.5% | 93.3% (n=15, ruido) |

Umbrales por nivel para precisión objetivo (`calibration.json → levels[*].thresholds`):

| Nivel | 90% precisión | 95% precisión | 98% precisión |
|-------|---------------|---------------|---------------|
| especie | p ≥ 0.72 | p ≥ 0.86 | no alcanzable |
| género  | p ≥ 0.74 | p ≥ 0.91 | no alcanzable |
| familia | p ≥ 0.69 | p ≥ 0.84 | **p ≥ 0.93** |

**Lectura:** sí existe punto seguro de auto-publicación a nivel especie (95% a p≥0.86,
cubriendo el 40%). El 98% solo es alcanzable a **familia**. Nota: `calibration.json` guarda
en la raíz los coeficientes del nivel **especie** (que es lo que consume el servicio); los
tres niveles completos están en la clave `levels`.

### L.4. Estado desplegado
Contenedor `fotofauna-embed` **reiniciado el 2026-08-01** con la calibración activa
(`[identify] calibración logistic (n=2016...)`, 1257 prototipos, GPU). Verificado en vivo:

    cratena_peregrina  -> species Cratena peregrina  sim=1.0    p_species=0.884
    flabellina_affinis -> family  Flabellinidae      sim=1.0    p_species=0.883
    doto_millbayana    -> genus   Doto               sim=0.888  p_species=0.435

El caso `Doto` es la prueba de que funciona: similitud alta (0.888) pero probabilidad baja
(0.435) — el calibrador reconoce el género sumidero y el servicio abstiene a género.

### L.4bis. RECALIBRADO con la BBDD de 1369 (2026-08-01, tarde) — **ESTO ES LO VIGENTE**
Tras subir de 1257 a 1369 patrones (alguien bajó `MIN_IMGS` 20→10, +111 especies) hubo que
recalibrar. Nueva cosecha **2.078 muestras / 937 especies**. Modelo elegido: logistic (full features).

**Acierto real (sustituye a la tabla de L.0):** especie **65.9%** · género **71.6%** ·
familia **76.7%**. Por tier: 0 → 65%, 1 → 65%, 2 → 73%.

> ⚠️ **`MIN_IMGS` 20→10 NO mejoró: empeoró un poco** (66.7% → 65.9% en especie, mismo
> método). Coherente con lo esperado: una especie con 10 fotos hace un prototipo débil y
> además añade una clase confundible para las 1258 que ya estaban. La caída es pequeña y
> puede ser ruido, pero **la hipótesis "añadir especies sube el acierto" queda descartada**.
> Si se quiere cobertura, vale; si se quiere precisión, considerar volver a `MIN_IMGS=20`.

**Modelo elegido: `full+iso`** (logística + isotónica). ECE de la similitud cruda **0.241**
→ calibrada **0.028**. AUC 0.74 → 0.86.

**Umbrales VIGENTES** (sustituyen a los de L.3):

| Nivel | 90% precisión | 95% precisión | 98% |
|-------|---------------|---------------|-----|
| especie | p ≥ 0.63 | p ≥ 0.89 | no alcanzable |
| género  | p ≥ 0.70 | p ≥ 0.86 | no alcanzable |
| familia | p ≥ 0.68 | p ≥ 0.90 | no alcanzable |

Punto de operación recomendado para el wave: **`p_species ≥ 0.90` → 96.0% de precisión real,
27% de cobertura**. (A p≥0.95 la precisión BAJA a 92.4% con n=79: es ruido de muestra
pequeña, no una regresión — no subas por encima de 0.90 buscando más precisión.)

Desplegado y verificado: `[identify] calibración logistic+isotonic (n=2078)`, 1369 prototipos.

> **2026-08-02 (~22:30): RE-CALIBRADO** tras añadir geo priors. Mismas 2078 muestras.
> Modelo: logistic (full features, sin isotonica). p>=0.90 → 95.6% precision.
> Umbrales: p90=0.68, p95=0.90. La calibracion con isotonica (L.4bis) fue reemplazada
> por logistic pura (mejor NLL en este dataset). Ver seccion M.


### L.5. SIGUIENTE PASO CONCRETO — reactivar el wave (NO hecho aún)
Ya está el requisito que faltaba (confianza fiable). Para reactivar:
1. En el consumidor del wave, **filtrar por `prediction.p_species`, NO por `confidence`**
   (`confidence` sigue siendo similitud sin calibrar — no usarla para decidir).
2. Empezar **conservador: `p_species ≥ 0.90`** (96% precisión medida, 29% de cobertura).
   Bajar a 0.86 (95%) solo tras ver resultados reales en Minka.
3. Mantener el safeguard `WAVE_YOLOFAUNA_REQUIRE_INAT=1` al principio.
4. `UPDATE autoid_schedules SET enabled=true WHERE id=1;`
5. Considerar publicar **a familia con p≥0.93** (98% de precisión) en los casos en que la
   especie no llegue al umbral: es una ID válida en Minka y casi nunca se equivoca.
6. **Recalibrar** cuando la BBDD cambie de forma relevante (más especies, encoder afinado):
   `harvest_calib.py` + `fit_calib.py` + reinicio. La calibración envejece con el modelo.

### L.6. Bajar `YOLOFAUNA_MIN_CONFIDENCE` — ojo
La sección de integración proponía bajar `YOLOFAUNA_MIN_CONFIDENCE` de 0.90 a 0.85. Ese env
compara contra la **similitud**, no contra la probabilidad. Ahora que hay calibración, lo
correcto es migrar ese umbral a `p_species` (donde 0.90 sí significa 96% de acierto) en vez
de seguir moviendo un número que no tiene interpretación.

## K. RESULTADO MLP (2026-08-01) — conclusión honesta
`train_head.py` (GPU, CAP=150, held-out 20%): **MLP = 69.4%** con 1257 especies. Modelo en
`dataset/mlp_head.pt` (+`mlp_labels.json`). **A escala completa el MLP NO bate al kNN**
(campo ~68-77%): el +3.4pt anterior era con ~300-440 especies; con 1257 clases + datos
capados la cabeza lineal/MLP no aporta. **Decisión: NO integrar el MLP solo.**
La verdadera palanca es el **fine-tuning del encoder BioCLIP** (metric learning enfocado en
pares crípticos) — mueve el espacio de embeddings, no solo la frontera de decisión. Es un
trabajo GPU-días, para sesión con más presupuesto. Alternativas baratas mientras: fusión de
sinónimos, priors GPS+estación, abstención género/familia (ya activa). El `mlp_head.pt` queda
guardado por si se quiere probar un ensemble kNN+MLP calibrado.
