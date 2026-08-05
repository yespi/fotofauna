# HANDOFF YOLOFauna — continuación fase 2 (escrito 2026-08-01)

Brief autocontenido para otro modelo/IA. Léelo entero antes de tocar nada. La doc de
referencia es [`YOLOFAUNA.md`](YOLOFAUNA.md) — **secciones A–F (arquitectura), L
(calibración)**. La sección 7 es histórica y NO refleja el estado actual.

---

## 0. Entorno — reglas que ahorran horas

- **Todo es LOCAL en HanSolo.** `/mnt/` es el disco de esta máquina. Se edita, ejecuta y
  verifica aquí. **Nunca `ssh`, nunca copiar ficheros a otro sitio.** Editar en
  `./docker/...` es editar producción directamente.
- **No hay CLI `docker`** en code-server, pero sí `/var/run/docker.sock`. Usa el helper:
  ```
  python3 ./docker/fotofauna-yolo/scripts/dexec.py fotofauna-embed <cmd...>
  python3 .../dexec.py -d fotofauna-embed sh -c 'cd /work && ... > logs/x.log 2>&1'
  ```
  Dentro del contenedor el repo se ve como `/work`. Tampoco hay `ps` dentro.
- **GPU**: RTX 3060 (12 GB). NO caben qwen (~11 GB) + BioCLIP a la vez. Para usar la GPU hay que
  **echar antes a qwen** (`free_qwen()` en `harvest_calib.py`, o `_free_gpu()` en el
  servicio) y **repetirlo cada ~60 s**: el poller de hansolo-agents lo recarga cada 120 s.
  Compensa: la cosecha de calibración pasó de 86 min (CPU) a 30 min (GPU).
- **Al reiniciar `identify_service.py` conserva las dos líneas
  `threading.Thread(target=_bg_loop...)` y `_qwen_suppressor`.** Se perdieron una vez y
  costó horas: sin ellas el auto-embed 24/7 muere en silencio.
- Antes de reiniciar el contenedor, comprueba que la cola de embed está vacía (si está
  embebiendo, lo cortas). Reiniciar:
  `curl -s -X POST --unix-socket /var/run/docker.sock http://localhost/containers/fotofauna-embed/restart?t=15`
- Tras cambios: **push a GitHub**. Repos: `/mnt/docker` (código) y `/mnt/docs` (doc), rama `main`.

---

## 1. RECALIBRAR — ✅ YA HECHO (2026-08-01 tarde). No repetir.

**Estado: la calibración vigente es `n=2078`, `logistic+isotonic`, ajustada contra la BBDD
de 1369 patrones y desplegada.** Verificado en `logs/identify.log`:
`[identify] calibración logistic+isotonic (n=2078...)`.

Resultados (detalle en YOLOFAUNA.md **L.4bis**): acierto real **65.9% especie · 71.6%
género · 76.7% familia**. ECE de la similitud cruda 0.241 → calibrada **0.028**.
**Hallazgo: `MIN_IMGS` 20→10 no mejoró, empeoró un poco** (66.7%→65.9%).

**Umbrales VIGENTES** (usa estos, no los de YOLOFAUNA.md L.3, que son de la BBDD vieja):

| Nivel | 90% precisión | 95% precisión |
|-------|---------------|---------------|
| especie | p ≥ 0.63 | p ≥ 0.89 |
| género  | p ≥ 0.70 | p ≥ 0.86 |
| familia | p ≥ 0.68 | p ≥ 0.90 |

**Empieza por la sección 2 (activar el wave).** Solo hay que volver a recalibrar si vuelve
a cambiar la BBDD o el encoder — y entonces se hace así:

<details><summary>Cómo recalibrar (para el futuro)</summary>

### Por qué
La calibración desplegada se ajustó con la BBDD en **1257 patrones**. Hoy hay **1369**
(alguien bajó `MIN_IMGS` de 20 a 10 y entraron +111 especies). Las features del calibrador
(`kclasses`, `share1`, `margin`) dependen de contra qué BBDD se puntúa: con un 8.8% más de
clases compitiendo en el top-25, **`p_species` quedó optimista**. Ese es exactamente el
sesgo peligroso: publicaría en Minka más de lo que su precisión real justifica.

**No sirve re-ajustar sobre el `calib_raw.jsonl` existente**: esas 2016 muestras se
puntuaron contra la BBDD vieja. Hay que **re-cosechar**. Es barato en tokens y caro en
reloj (~90 min).

### Pasos exactos
```bash
cd ./docker/fotofauna-yolo
mv dataset/calib_raw.jsonl dataset/calib_raw_1257.jsonl      # conservar para comparar
python3 scripts/dexec.py -d fotofauna-embed sh -c \
  'cd /work && python3 scripts/harvest_calib.py 3 > logs/harvest_calib.log 2>&1'
```
Vigilar con `tail -2 logs/harvest_calib.log` y `wc -l < dataset/calib_raw.jsonl`.
**El log solo escribe cada 25 especies (~1-2 min): que no se mueva 2 minutos NO es que esté
colgado.** Termina con la línea `=== COSECHA FIN:`. Luego:
```bash
python3 scripts/dexec.py fotofauna-embed sh -c \
  'cd /work && python3 scripts/fit_calib.py 2>&1 | tee logs/fit_calib.log'
# reiniciar para cargar la calibración nueva (ver reglas del punto 0)
```
Verificar en `logs/identify.log` la línea `[identify] calibración logistic (n=...)`.

### Qué mirar en el resultado
1. **ECE de `full`** debe seguir ≲0.06. Si se dispara, algo va mal.
2. **La nueva tabla umbral→precisión** sustituye a la de YOLOFAUNA.md L.3. Los umbrales
   viejos (especie p≥0.90 → 96%) **ya no son válidos**; usa los nuevos.
3. **Compara el acierto de especie con el 66.7% anterior** (mismo método, set limpio). Esto
   responde gratis una pregunta abierta: si bajó, `MIN_IMGS` 20→10 **perjudicó** — una
   especie con 10 fotos hace un prototipo débil y además añade una clase confundible para
   las 1258 que ya estaban. Si es así, considera volver a `MIN_IMGS=20` o exigir ≥20 solo
   para entrar al kNN. **No asumas que es neutro: mídelo.**
4. Actualiza YOLOFAUNA.md sección L con los números nuevos.

</details>

---

## 2. ⬅️ EMPIEZA AQUÍ — ACTIVAR EL WAVE

Detalle en YOLOFAUNA.md **L.5**. Resumen:

1. El consumidor del wave debe filtrar por **`prediction.p_species`**, NUNCA por
   `prediction.confidence`. **`confidence` sigue siendo la similitud coseno sin calibrar**
   — se mantuvo así solo por compatibilidad con el frontend y no tiene interpretación
   probabilística. Confundirlas es el bug que motivó todo esto.
2. Umbral: **`p_species >= 0.90`** → 96.0% de precisión real, 27% de cobertura (medido con
   la calibración vigente n=2078). **No subas a 0.95 buscando más precisión**: ahí baja a
   92.4% con n=79, que es ruido de muestra pequeña, no una mejora.
3. Mantener el safeguard `WAVE_YOLOFAUNA_REQUIRE_INAT=1` al principio.
4. `UPDATE autoid_schedules SET enabled=true WHERE id=1;`
5. **Vigilar las primeras publicaciones reales en Minka** antes de bajar el umbral o quitar
   el safeguard. Publicar mal en Minka es difícil de revertir (ya hubo que retirar 56 IDs).
6. Los env `YOLOFAUNA_MIN_CONFIDENCE` y `WAVE_YOLOFAUNA_MIN` (0.85) comparan contra
   **similitud**. Migrarlos a `p_species` en vez de seguir moviendo un número sin
   interpretación.

---

## 3. SUBIR EL ACIERTO — orden recomendado

> ### 🔍 REVISIÓN EXTERNA (Fable 5, 2026-08-01) — dos hallazgos VERIFICADOS
> Se pidió a otro modelo que cuestionara la estrategia. Dos de sus objeciones se
> comprobaron contra el código y **son ciertas**:
>
> **1. NO se recorta el organismo: se embebe la FOTO ENTERA.** Comprobado: no hay `crop`,
> `bbox` ni detector en `download_species.py`, `embed_species.py` ni `identify_service.py`.
> Se embebe la imagen `medium` de Minka completa, con fondo. En heterobranquios los rasgos
> diagnósticos son diminutos (rinóforos, ápices de los ceratas, branquias), así que el
> organismo puede ocupar una fracción del encuadre. **Es la palanca barata más prometedora.**
> ⚠️ *Matiz que Fable 5 no menciona*: BioCLIP-2 se entrenó con fotos estilo iNat (organismo
> entero con contexto), así que un recorte muy ajustado puede salirse de su distribución de
> entrenamiento. **No des por hecha la ganancia: es una pregunta empírica.** Y re-embeber
> las 533k imágenes son horas de GPU. → **Haz un A/B sobre ~30 especies crípticas primero.**
>
> **2. La comparación MLP vs kNN fue INJUSTA.** `train_head.py` usa `CAP=150` imágenes por
> especie mientras el kNN usa **todas** (hasta 1000). Que un head con datos capados no gane
> a un kNN con datos completos **no demuestra** que la frontera de decisión no sea el
> problema. La conclusión de la sección K ("no integrar el MLP") está **sobre-extraída**:
> no está refutado, está mal medido. Si se retoma, repetir sin `CAP`.
>
> **Otras objeciones, aceptadas sin verificar (son de criterio, no de hecho):**
> - **La isotónica es frágil con n≈590 de held-out.** El síntoma ya está en nuestros datos:
>   a p≥0.95 la precisión *baja* a 92.4% (n=79). Eso es lo que hace la isotónica en la cola
>   con pocos datos — y la cola es justo donde auto-publicamos. El margen de NLL sobre la
>   logística sola era mínimo (0.4218 vs 0.4309). **Cambiar a logística sola o a beta
>   calibration** (paramétrica y monótona). Ver 3.0.e.
> - **Falta intervalo de confianza en el punto de operación.** "p≥0.90 → 96%" está estimado
>   sobre unos cientos de muestras. Con bootstrap podría ser "94-97%", y eso mueve el umbral
>   inicial del wave. Ver 3.0.e.
> - **La abstención mezcla dos superficies de producto distintas.** Ver 3.0.f. Es la
>   objeción más fuerte de la revisión.
> - **Escalón intermedio antes de los GPU-días**: ensemble con DINOv2 congelado, o LoRA /
>   último bloque con ArcFace, en vez de saltar de "nada" a "fine-tuning completo".
>   ⚠️ *Matiz*: el ensemble no es gratis en operación — segundo encoder (~1.1 GB) + duplicar
>   el almacén de embeddings + doblar la latencia, en una 3060 que ya comparte con qwen.

El techo del kNN + coseno sobre BioCLIP está en ~67% a nivel especie para fauna marina.
280 de 975 especies son congéneres visualmente indistinguibles. **Pero ese "techo" no está
demostrado** hasta agotar recorte/TTA y separar error de modelo de ruido de etiqueta.

### 3.0 CALIDAD DE ETIQUETA — curadores de confianza ← **hacer ANTES que los priors**

Idea del usuario (2026-08-01): hay curadores de Minka mucho más fiables que la media.
Nombres que da como referencia: **`xasalva`, `badosa`, `bertinhaco`** (lista ampliable —
**guárdala en `dataset/trusted_curators.json`, no la hardcodees**).

#### Contexto imprescindible antes de tocar nada
- **El filtro de calidad YA está puesto**: `download_species.py` descarga solo
  `quality_grade=research` (Minka e iNat). Toda la BBDD ya pasó por curación. **No hay
  nada que ganar simplemente "fiándose de research grade": ya nos fiamos al 100%.**
- **`research grade` NO garantiza 2 identificaciones humanas.** Comprobado contra la API:
  hay observaciones research con `identifications_count: 1`. Y existen los campos
  `ai_identified` (observación) y `vision` (por identificación): **parte de las etiquetas
  salieron de la visión artificial de iNat, no de un humano.**
- **Riesgo de bucle**: entrenar con etiquetas generadas por el CV de iNat es destilar el
  modelo de iNat, errores incluidos — y es en crípticas donde más falla.
- **Sesgo correlacionado**: en congéneres crípticos el consenso no converge a la verdad,
  converge a *la especie más común del género*. Eso explica el sumidero medido en `Doto`
  (todo cae en `doto_millbayana`). Más peso al consenso genérico AMPLIFICA el sesgo; peso a
  curadores expertos, no.

#### 3.0.0 — RECORTE (crop) + TTA ⬅️ **TAREA 1 — antes que nada**
Hoy se embebe la **foto entera**. Prueba barata y decisiva, **no re-embebas los 533k de golpe**:
1. Coge ~30 especies crípticas (las peores de `analyze_acc.py`, géneros *Doto*, *Elysia*,
   *Favorinus*, *Haminoea*).
2. Re-embebe SOLO esas con el organismo recortado. Detector: `yolo26n.pt` está en
   `./docker/ecosistema-fauna/backend/`; hay además un `dataset/yolo26n-seg.pt` sin usar
   que permite recorte por máscara. Si el detector no encuentra nada (pasa con fauna marina),
   **conserva la foto entera** — no descartes la imagen.
3. Mide con el mismo protocolo out-of-sample (`harvest_calib.py` restringido a esas especies)
   y compara contra su acierto actual.
4. Prueba también **TTA**: multi-crop (centro + 4 esquinas) y mayor resolución, promediando
   los embeddings. Es gratis, no requiere re-descargar nada.
- **Si gana ≥3 pt** → re-embeber toda la BBDD (horas de GPU) y **recalibrar**.
- **Si no gana o empeora** → confirmado que BioCLIP prefiere la foto con contexto; anótalo
  en la doc y cierra la vía. Es un resultado útil igualmente.

#### 3.0.e — Endurecer la calibración para la cola (rápido, sin cosechar)
Sobre los datos que ya están en `dataset/calib_raw.jsonl`, sin re-descargar nada:
1. En `fit_calib.py`, sustituir la isotónica por **beta calibration** (o quedarse con la
   logística sola) y comparar el comportamiento **en la cola p≥0.90**, no solo el ECE global.
2. Añadir **CIs por bootstrap** (1000 remuestreos) a `operating_points`: cada umbral debe
   reportar `precision_real` con su intervalo. Publicar en `calibration.json` y en el panel.
3. Reportar la fiabilidad **por tier**: un género sumidero y una especie fácil no comparten
   la misma relación features→P(acierto).
Es media hora de trabajo y **cambia el umbral con el que arranca el wave**.

#### 3.0.f — Separar la política de abstención por superficie ⬅️ **objeción más fuerte**
Hoy una sola regla (`margin < 0.06` sobre similitud cruda) sirve a dos superficies que
necesitan cosas opuestas:
- **Wave autónomo → Minka: abstenerse es CORRECTO.** Escribir una especie mal en una BBDD
  científica compartida es caro (ya se retiraron 56 IDs). Una ID a género es una
  contribución legítima que otros curadores refinan. Precisión primero.
- **Usuario interactivo en FotoFauna: abstenerse en silencio INFRA-SIRVE.** El usuario
  estuvo allí y vio el animal. Lo correcto es **top-k de especies con su p calibrada bajo el
  titular seguro del género**: *"Doto — 43%; probablemente D. millbayana / D. koenneckeri /
  D. pinnatifida"*. La información ya se calcula; hoy se tira.
- **Y en ambas: la abstención se dispara ANTES de aplicar los priors GPS/estación.** Muchos
  empates de congéneres se rompen porque solo uno vive en ese sitio o temporada. La
  abstención debe ser el fallback **después** de los priors.
- **Migrar el disparador** de `margin < 0.06` a la **p calibrada por nivel**: especie si
  `p_species ≥ umbral`, si no género si `p_genus ≥ umbral`, si no familia. Los tres niveles
  ya están en `calibration.json → levels`.

#### 3.0.a — Set de evaluación limpio con curadores ⬅️ **TAREA 2**
**Hazlo primero, antes de tocar el kNN.** Hoy medimos el acierto contra etiquetas research
que en crípticas pueden estar mal. Es decir: **no sabemos cuánto del 65.9% es "el modelo
falla" y cuánto es "la etiqueta de referencia está mal".** Esta medición decide en qué
merece la pena gastar los GPU-días después. Coste: ~30-40 min de cosecha en GPU.

##### Campos de la API — YA VERIFICADOS, no los redescubras
`GET https://api.minka-sdg.org/v1/observations?taxon_id=<X>&quality_grade=research&photos=true&per_page=30&order=desc&order_by=id`
devuelve, **en la misma respuesta** (no hace falta otra llamada), un array
`identifications[]` donde cada elemento tiene:

| Campo | Para qué sirve |
|-------|----------------|
| `user.login` | login del identificador → cruzar con `trusted_curators.json` |
| `own_observation` | `true` = es el propio observador auto-identificándose → **NO es corroboración** |
| `vision` | `true` = la ID salió de visión artificial, no de un humano |
| `current` | `true` = la identificación sigue vigente (las retiradas quedan en `false`) |
| `taxon_id` | a qué taxón identificó (puede NO ser el de la observación) |
| `category` | `improving` / `supporting` / `leading` / `maverick` (concepto iNat) |
| `disagreement` | marca desacuerdo explícito |
| `ai_model` | modelo de IA si aplica |

A nivel de observación: `identifications_count`, `num_identification_agreements`,
`num_identification_disagreements`, `identifications_most_disagree`, `ai_identified`.

##### Definición de "identificación de confianza" (usa esta, exacta)
Una observación cuenta como **etiqueta de confianza** para la especie `S` si existe al menos
una entrada en `identifications[]` que cumpla **las cuatro condiciones**:
1. `user.login` ∈ `dataset/trusted_curators.json → curators`
2. `own_observation == false`  ← corroboración de un tercero, no auto-ID
3. `vision == false`           ← humano, no CV
4. `current == true` **y** `taxon_id` == `minka_taxon` de `S`

##### Implementación
1. Añadir a `harvest_calib.py` un flag `--trusted-only` (y `--out <fichero>`).
   **No hace falta ninguna llamada extra a la API**: `identifications[]` ya viene en la
   respuesta que el script ya pide. Solo hay que filtrar dentro del bucle de observaciones.
2. Mantener **intacta** la exclusión out-of-sample por `obs id` contra `_manifest.jsonl`.
   Sin eso la medición no vale nada.
3. Escribir a `dataset/calib_trusted.jsonl` — **fichero aparte**, que no se mezcle con
   `calib_raw.jsonl` (son distribuciones distintas y mezclarlas contamina la calibración).
4. Subir `PER_SP` a 5: los curadores de confianza cubren menos especies, hace falta más
   muestra por especie para llegar a n útil.

##### ⚠️ Comparación JUSTA — esto es lo que más fácil se hace mal
Los curadores de confianza son **especialistas en heterobranquios**, así que el set limpio
estará sesgado a tier 0. Comparar su acierto contra el 65.9% global sería comparar mezclas
de especies distintas y **el resultado no significaría nada**.

**Obligatorio:** restringe el set general a **las mismas especies** que aparecen en el set
de confianza y compara sobre esa intersección. `calib_raw.jsonl` ya está en disco con el
campo `true` (slug), así que es un filtro trivial:
```python
sp_trusted = {r["true"] for r in trusted}
base = [r for r in raw if r["true"] in sp_trusted]   # <- contra esto se compara
```
Reporta las dos cifras (intersección) y el número de especies y muestras de cada lado.

##### Criterios de aceptación
- n ≥ 400 muestras en el set de confianza, ≥ 80 especies. Si no llegas, dilo y no concluyas.
- Reporta: acierto especie/género/familia en el set de confianza, en la intersección del
  set general, y la diferencia con intervalo aproximado (±1.96·√(p(1-p)/n)).

##### Cómo interpretar
- **Sube ≥5 puntos** → buena parte del "error" era ruido de etiqueta. El motor es mejor de
  lo que creemos. Entonces: prioriza 3.0.b y 3.0.c (limpiar y ponderar etiquetas) y
  **replantea si el fine-tuning merece los GPU-días**.
- **No se mueve (±2 puntos)** → el problema es genuinamente visual. Entonces las etiquetas
  no son el cuello de botella: ve directo al fine-tuning (3.3) y no gastes en 3.0.c.
- **Baja** → sospecha del filtro; probablemente estés colando auto-IDs (`own_observation`)
  o IDs de visión. Revisa las cuatro condiciones.

#### 3.0.b — Limpiar etiquetas de origen IA
Filtro extra en `download_species.py`: descartar (o marcar) observaciones cuya identificación
venga de `vision`/`ai_identified` **sin corroboración humana**. Rompe el bucle con iNat.

#### 3.0.c — Ponderar el kNN por confianza de etiqueta
Solo después de 3.0.a. Diseño:
1. Sidecar `dataset/label_trust.jsonl`: `obs_id -> peso`. Se construye consultando la API por
   lotes de ids (`?id=1,2,3...`, hasta 200 por página) usando los `obs` que ya están en cada
   `_manifest.jsonl`. **No hay que volver a descargar imágenes.**
2. Fórmula sugerida (parámetros en JSON, para poder barrerlos):
   `w = 1.0 × (1.6 si lo identificó un curador de confianza) × (1 + 0.1·min(num_identification_agreements,5)) × (0.5 si vino de visión/IA sin humano)`, recortado a `[0.3, 2.0]`.
3. Guardar `weights.npy` junto a `embeddings.npy` en cada `dataset/patterns/<slug>/`.
4. En `identify_service.py`, el kNN pasa de `scores[l] += max(sim,0)` a
   `scores[l] += w_i · max(sim,0)`. Cambio pequeño y localizado.

#### 3.0.d — Detector automático de pares crípticos (gratis)
`identifications_most_disagree`, `num_identification_disagreements` y sobre todo
`identifications[].category == "maverick"` (identificación que contradice al consenso) dicen
**qué pares confunden a los humanos**. Extrae los pares `(taxon_id de la ID, taxon_id de la
observación)` de cada desacuerdo y cuéntalos: los más frecuentes son los pares crípticos
reales. Es exactamente la lista que necesita el fine-tuning de 3.3, y sale sola sin tener
que derivarla a mano de `analyze_acc.py`.

#### ⚠️ Dos avisos
- **Confusión fotógrafo/especie**: `xasalva` aporta 408 de las fotos de *Cratena peregrina*.
  Si subes el peso de sus fotos, puedes estar premiando **un estilo fotográfico** (misma
  cámara, mismos sitios, misma iluminación) y no la calidad de la etiqueta. Mide por tier y
  vigila que no baje el acierto en especies donde ese curador no participa.
- **Todo esto cambia la BBDD o el scoring ⇒ hay que RECALIBRAR después** (sección 1).

### 3.1 Priors GPS + fecha ← después de 3.0
Mejor relación valor/esfuerzo y **no necesita GPU**. Ataca la causa estructural: recorta el
pool de candidatos por ubicación y temporada, que es justo lo que necesitan los congéneres
indistinguibles. FotoFauna ya tiene GPS y fecha de la foto real.

Diseño sugerido:
- `/identify` acepta `lat`, `lon`, `date` opcionales (si faltan, comportamiento actual).
- Construir por especie un prior de rango/estación desde las observaciones de Minka que ya
  están en `_manifest.jsonl` (tiene `place`) o consultando la taxonomía de Minka.
- Re-rankear el score kNN multiplicándolo por el prior, con **suelo** (nunca eliminar del
  todo un candidato: los rangos se expanden y hay divagantes).
- **Al terminar hay que RECALIBRAR** — cambia la distribución de scores.

Ojo: no separa congéneres simpátricos (los que conviven), solo confusiones inter-región.

### 3.2 Fusión de sinónimos
Gratis pero pequeño: es un puñado de casos, no una palanca estructural. Caso confirmado:
`ambigolimax_valentianus` ≡ `lehmannia_valentiana` (misma especie, dos slugs → confusión
mutua garantizada). Detectar más con `scripts/analyze_acc.py`; **Minka es el árbitro** de
qué nombre gana (regla canónica, YOLOFAUNA.md sec. C).

### 3.3 Fine-tuning del encoder BioCLIP ← el único que rompe el techo
Metric learning (triplet/ArcFace) enfocado en los pares crípticos que saca
`analyze_acc.py`. Mueve el espacio de embeddings, no solo la frontera de decisión. **Son
GPU-días**: sesión dedicada. Al terminar hay que **regenerar todos los embeddings** y
**recalibrar**.

### 3.4 Más datos para especies sub-representadas
Añade cobertura, no acierto. Baja prioridad para precisión.

### NO ayudan (ya descartado, no repetir)
- ~~**MLP/cabeza lineal**~~ **REABIERTO**: la medición fue injusta (`CAP=150` en el MLP vs
  todos los embeddings en el kNN). No está refutado, está mal medido. Si se retoma, sin CAP.
- COCO/YOLO como clasificador (YOLO solo sirve para recortar).

---

## 4. Trampas conocidas

- **`eval_field.py` MIENTE**: mide con fuga de datos. Asume que las observaciones recientes
  no están en la BBDD porque se descarga con `order=asc`; falso — el 92% ya estaban dentro.
  **Para medir de verdad usa `harvest_calib.py`**, que excluye por `obs id` contra
  `_manifest.jsonl`. Los ~68-77% que aparecen en secciones antiguas de la doc son inflados;
  los reales son 66.7% especie · 72.6% género · 77.3% familia.
- `_manifest.jsonl` **sobrevive al archivado a HDD** aunque los `.jpg` se muevan a
  `/mnt/archive` — por eso la exclusión funciona.
- El acierto **in-sample** (`_write_eval`, `dataset/patterns/_eval.json`) no sirve para
  decidir umbrales de publicación. Solo el out-of-sample.
- `deploy-to-pro.sh` falla en code-server (el paso SEO llama al CLI `docker`, inexistente).
  Los `/composables/` son no-cache: copiarlos a mano vale.
- 3 especies de algas/hierba aparcadas en `dataset/_parked_oom/` (reversible: devolver los
  `.jpg` a `dataset/images/<slug>/`).

---

## 5. Ficheros clave

| Qué | Dónde |
|-----|-------|
| Servicio de identificación | `./docker/fotofauna-yolo/scripts/identify_service.py` |
| Cosecha del set de calibración | `scripts/harvest_calib.py` |
| Ajuste de la calibración | `scripts/fit_calib.py` |
| Helper de exec por socket | `scripts/dexec.py` |
| Calibración desplegada | `dataset/calibration.json` (3 niveles en la clave `levels`) |
| Análisis de confusión | `scripts/analyze_acc.py`, `scripts/test_knn.py` |
| Duelo contra iNat | `scripts/duel_inat.py` |
| Doc de referencia | `/mnt/docs/fotofauna/YOLOFAUNA.md` (sec. A–F, L) |
| Estado de tareas | `/mnt/docs/TAREAS_PENDIENTES.md` |
| Prompt de arranque para DeepSeek | [`PROMPT_DEEPSEEK.md`](PROMPT_DEEPSEEK.md) |

---

## 6. ⏭️ ARRANQUE INMEDIATO — TAREAS PARA DEEPSEEK (añadido 2026-08-03)

> Continuación operativa de las secciones 1–5. El **foco pedido por el usuario** es doble:
> **(1) avanzar el entrenamiento todo lo posible** y **(2) liberar/compactar disco**. Las
> tareas están ordenadas por ese foco. **Lee antes las secciones 0–5**: nada de aquí las
> contradice, las ejecuta. El prompt de arranque copiable está en
> [`PROMPT_DEEPSEEK.md`](PROMPT_DEEPSEEK.md).

### 6.0 Estado real VERIFICADO hoy (2026-08-03) — no lo redescubras

Mediciones baratas ya hechas (`du`, cabecera `.npy`, `nvidia-smi`). Úsalas como base:

| Cosa | Valor verificado |
|------|------------------|
| `./docker/fotofauna-yolo` | **16 GB** (SSD) |
| └ `dataset/images` (jpg vivos) | **14 GB**, 1659 dirs de especie |
| └ `dataset/patterns` (índice kNN) | **797 MB**, 1369 `embeddings.npy`, **533.823 filas** |
| └ `hf_cache` (pesos BioCLIP‑2) | **1,6 GB** (un único `open_clip_model.safetensors`) |
| └ `dataset/_parked_oom` | 186 MB (3 algas, reversible) · `logs/` 2,2 MB |
| `/mnt/archive/fotofauna-yolo/images` | **73 GB** en HDD = **set COMPLETO** de imágenes |
| **dtype de los embeddings** | **`float16` YA** (`<f2`) en los 1369 ficheros — ver 6.E |
| GPU (RTX 3060, 12 GB) | qwen **9884 MiB** + BioCLIP **1842 MiB** → **119 MiB libres** |

**Dos hechos que cambian el plan respecto a lo que se asumía:**
1. **Los embeddings ya son float16** (no float32). La compactación "a la mitad" del índice
   **ya está hecha**; ver Tarea E antes de intentar "convertir a float16".
2. **En `dataset/images/<slug>/` solo queda una MUESTRA** (p.ej. 80 de 222 jpg): el set
   completo vive en `/mnt/archive`. Esto es crítico para el re-embed (Tarea B) y explica la
   limpieza segura (Tarea D). Verificado: 76.655 jpg locales (**11,73 GB**) son
   byte‑idénticos a su gemelo en `/mnt/archive`; 1,94 GB son de especies aún sin archivar.

---

### Tarea A — Liberar la GPU de forma controlada ⬅️ **prerequisito de B y C**
**Objetivo:** dejar VRAM para el encoder (BioCLIP ~1,8 GB) + YOLO + margen de batch. Hoy qwen
ocupa 9,9 GB con `keep_alive` infinito y no caben los dos.

**Pasos:**
1. Expulsar qwen y **repetir cada ~60 s** (el poller de hansolo-agents lo recarga cada 120 s).
   Lanza en background:
   ```bash
   while true; do curl -s http://172.17.0.1:11434/api/generate \
     -d '{"model":"qwen3.6-hansolo","keep_alive":0}' >/dev/null 2>&1; sleep 60; done &
   ```
   (Es exactamente lo que hacen `free_qwen()` en `harvest_calib.py` y el bucle de
   `test_crop.py`.) Alternativa más permanente: fijar `keep_alive` finito a qwen en la config
   de hansolo-agents.
2. Verificar: `nvidia-smi --query-gpu=memory.free --format=csv` debe subir a ~10 GB libres.

**Trade-off (dilo, no lo escondas):** con qwen expulsado, el LLM de hansolo-agents responde
lento (recarga) o no responde mientras dure el trabajo GPU. Restablece parando el bucle al
terminar (qwen se recargará solo en ≤120 s).

**Criterio de aceptación:** ≥9 GB de VRAM libres de forma sostenida durante el trabajo.

---

### Tarea B — Re-embed completo con RECORTE + recalibrar ⬅️ **palanca +7,1 pt ya validada**
**Objetivo:** aplicar a TODA la BBDD el recorte del organismo, que en el A/B de 3.0.0 dio
**+7,1 pt en crípticas**, y recalibrar. Es la mejora de acierto con mejor relación
valor/esfuerzo que ya está demostrada.

**⚠️ Trampa crítica (verificada leyendo `embed_species.py` + estado de disco):**
- `embed_species.py` lee de `dataset/images/<slug>/*.jpg` y es **incremental** (salta especies
  que ya tienen `prototype.npy`). Para re-embeber hay que **forzarlo** (renombrar/mover
  `dataset/patterns/` a un backup antes).
- **En local solo está la MUESTRA** de cada especie archivada. Si re-embebes "en sitio",
  construirías el índice con ~1/3 de las imágenes → **degradarías la BBDD**. La fuente
  correcta es **`/mnt/archive/fotofauna-yolo/images/<slug>/`** (set completo).
- **No caben los 73 GB de golpe** en el SSD (~68 GB libres). → **Trocea por lotes de especie**:
  restaurar lote del archive → recortar+embeber → volver a archivar. Esto es justo lo que
  permite "avanzar todo lo posible" en sesiones cortas.

**Pasos:**
1. Tarea A (GPU libre).
2. `cp -r dataset/patterns dataset/patterns_full_backup` (revertible si baja el acierto).
3. Escribir `scripts/embed_crop.py` reutilizando `embed_species.py` + `crop_organism()` de
   `test_crop.py`: por cada jpg `crop = crop_organism(path, yolo)`; si `None` → **foto
   entera** (no descartar); embeber; guardar `embeddings.npy`/`prototype.npy` en **float16**
   (como ya hace el pipeline). Detector: `dataset/yolo26n-seg.pt`, margen 10 % (ya en
   `test_crop.py`).
4. Alimentarlo con las imágenes **completas del archive**, por lotes de N especies para no
   llenar el SSD.
5. Al terminar TODAS: **recalibrar** (sección 1: `harvest_calib.py` → `fit_calib.py`),
   actualizar `YOLOFAUNA.md` L y los umbrales, y reiniciar el contenedor **con la cola de
   embed vacía** (sección 0).
   Ejecuta siempre vía `dexec.py`:
   ```bash
   python3 scripts/dexec.py -d fotofauna-embed sh -c \
     'cd /work && python3 scripts/embed_crop.py <lote...> > logs/embed_crop.log 2>&1'
   ```

**Criterios de aceptación:**
- 1369 `prototype.npy` regenerados; nº de filas total ≈ 533.823 (±descartes de detección).
- Acierto out‑of‑sample (`harvest_calib.py`, **no** `eval_field.py`) **≥ 65,9 % especie**
  actual; en crípticas debería subir. **Si baja globalmente → revertir** con el backup.
- ECE de `full` ≲ 0,06 tras recalibrar.
- **Recordatorio:** cambió la BBDD ⇒ RECALIBRAR obligatorio (si no, `p_species` miente).

---

### Tarea C — Preparar/empezar el fine-tuning del encoder (3.3) ⬅️ **el único que rompe el techo**
**Objetivo:** metric learning (ArcFace/triplet) sobre BioCLIP enfocado en pares crípticos.
Mueve el espacio de embeddings, no solo la frontera. **Son GPU‑días** → trocear en sesiones
con checkpoint.

**Pasos:**
1. **Dataset de pares crípticos** (gratis): sácalos del detector de mavericks de 3.0.d
   (`identifications[].category=="maverick"`, `identifications_most_disagree`) y/o de la matriz
   de confusión de `analyze_acc.py`. Guardar `dataset/crypto_pairs.json`.
2. **Esquema por etapas** (recomendado por la revisión externa, §3): empezar por
   **LoRA / último bloque + ArcFace** antes de saltar al fine-tune completo. Congelar el
   backbone salvo el último bloque en la etapa 1.
3. **Checkpoints reanudables** en `dataset/ft_ckpts/`: cada sesión deja checkpoint + log de
   métricas de validación. Así se "avanza todo lo posible" sin sesión maratón.
4. Prerequisitos: Tarea A (GPU) y, para validar, imágenes restauradas por lotes del archive
   (como en B).
5. **Al desplegar cualquier checkpoint:** regenerar **todos** los embeddings con el encoder
   nuevo (Tarea B) **y recalibrar**. No negociable.

**Criterios de aceptación:**
- Acierto en el held‑out de pares crípticos sube vs BioCLIP base.
- **No** degrada el acierto global (medir con `harvest_calib.py` sobre el set general).
- Checkpoint desplegado documentado + instrucciones de reanudación.

---

### Tarea D — Limpieza de disco (hasta ~11,7 GB, verificado) 
**Hallazgo (medido hoy):** `dataset/images` = 14 GB; de ellos **11,73 GB (76.655 jpg)** son
**byte‑idénticos** a copias ya presentes en `/mnt/archive/fotofauna-yolo/images` (la "muestra
local" que deja `archive_processed.py`). El set completo está a salvo en el HDD.

**Pre-checks OBLIGATORIOS (no borres a ciegas):**
1. Confirmar que **ninguna ruta de servicio lee `dataset/images/<slug>/*.jpg` en caliente**
   (identify usa `embeddings.npy` + `_manifest.jsonl`). Si `build_species_thumbs.py` /
   frontend usan esas jpg para miniaturas, **regenerar thumbs antes**.
2. Borrar **solo** jpg que tengan gemelo **idéntico en tamaño** en el archive (el script de
   medición de 6.0 ya hace esa comprobación; reúsalo por especie).
3. **NO tocar** `_manifest.jsonl`, `.done`, `.archived`, ni las jpg de especies **sin**
   `.archived` (1,94 GB local‑only, aún sin archivar).

**Interacción con B/C:** como B y C leen las imágenes **completas del archive** de todos modos,
borrar la muestra local **no les afecta** (nunca fue la fuente completa). Es reversible
(re‑copiar del archive).

**Menores trivialmente seguros (opcional, <5 MB, confirmar antes):** `logs/eval_field*.log`
(el script que miente), `logs/download_flabellina.log`, `dataset/calibration_1257.json.bak`,
`scripts/build_geo_priors.py.bak`, y `dataset/calib_raw_1257.jsonl` **solo si ya no hace falta
comparar** con la calibración de 1257.

**Criterios de aceptación:**
- `du -sh dataset/images` baja ~11 GB.
- Siguen los **1369** `_manifest.jsonl` (`harvest_calib.py` mantiene la exclusión
  out‑of‑sample: si rompes el manifest, la medición deja de valer).

---

### Tarea E — Compactación del modelo/índice (leer antes de "convertir a float16")
**REALIDAD verificada:** el índice **ya está en float16** (`descr:'<f2'` en los 1369
`embeddings.npy`; 797 MB / 533.823×768). En float32 ocuparía ~1,6 GB → **la mitad ya está
ahorrada**, en disco y en VRAM. **La palanca "embeddings a float16" ya está hecha: no la
repitas.**

**Opciones restantes — EXPERIMENTAL (exigen re‑medir acierto y poder revertir):**
1. **Cuantizar el índice a int8 / product quantization (PQ, faiss).** Ahorra otro ~50 %, pero
   introduce error en el coseno → **medir acierto antes/después con `harvest_calib.py`**. Solo
   si el disco/VRAM aprieta de verdad (hoy no aprieta: 797 MB).
2. **Cuantizar el encoder (int8)** o mantenerlo en fp16 en VRAM (BioCLIP ya carga fp16).
   Ganancia marginal, riesgo real en acierto.
3. **Pesos BioCLIP‑2:** 1,6 GB en un único safetensors; no comprimible sin cuantizar. No tocar
   salvo experimento controlado.

**Regla:** cualquier cuantización cambia el scoring ⇒ **recalibrar** y comparar acierto; si
baja, revertir. El mayor ahorro de disco real hoy **no es compactar el índice, es la Tarea D**.

---

### Tarea F — (rápida, bajo riesgo) T5: umbrales del flujo interactivo → `p_species`
**Objetivo:** el **wave** ya publica por `p_species`; el flujo **interactivo** de FotoFauna
sigue decidiendo por `similarity ≥ 0.85` (sin calibrar). Migrarlo.

**Fichero:** `./docker/ecosistema-fauna/backend/vision_identify.py` (~L125‑126 y L524).
Cuando `local.get("calibrated")` sea true, decidir por `p_species ≥ YOLOFAUNA_MIN_P_SPECIES`
(reutiliza el 0.90 del wave); caer a `similarity` solo si no hay calibración.

**Criterios de aceptación:** foto con `p_species < 0.90` pero `similarity ≥ 0.85` → **no**
auto‑confirma (queda sugerencia); `p_species ≥ 0.90` → confirma. Re‑verificar que el grid
sigue mostrando la especie (histórico `ac4b5c2c`). Deploy: editar PRE → `bump-version.sh` →
promover a PRO → verificar (`curl`) → push. Ya registrado en `FOTOFAUNA_ANALISIS.md` (T5) y
`TAREAS_PENDIENTES.md` — **no dupliques el registro, ejecútalo.**

---

### Orden recomendado
**A → B → (D en paralelo, es independiente) → C → F.** E casi todo está hecho (índice ya
float16); trátala como "documentar y, si sobra tiempo, experimentar con PQ". Cada vez que
cambie **BBDD, encoder o scoring: RECALIBRAR** (sección 1).
