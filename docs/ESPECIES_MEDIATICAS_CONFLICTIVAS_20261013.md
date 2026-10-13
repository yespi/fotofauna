# Especies invasoras mediáticas / conflictivas (13-oct-2026)

Orden Gustavo 13-oct 14:40: tratar *Callinectes sapidus* (cangrejo azul) y *Rugulopteryx okamurae* igual que *Pterois miles* (pez león): especies de las que se habla mucho y en las que un error de identificación se nota más.

**Sin promote FAISS. AutoID global intacto (p≥0,83, 1 worker).**

## Qué tenía hoy el pez león

| Capa | Trato de *Pterois miles* |
|------|--------------------------|
| `yf_species.json` | Presente, `prio=4` |
| Academy `TARGET_SPECIES` | `group=invasora`, `invasive=True` |
| `KNOWN_INVASIVE_NAMES` | Sí |
| Alertas Academy (`bq_invasive_update._CURATED`) | Origen / vector / «alerta mediterránea» / nota de salud |
| Descarga BQ | En lista de obs a cosechar |
| Umbral jerárquico | Género *Pterois* threshold **0,8467** (más estricto que 0,83) |
| Listas BF | Weitzmann marinas; no en lista UE |
| Paper | Destacado en origen/vías y láminas de expansión |

## Cambios aplicados a las tres

1. Catálogo Academy + `KNOWN_INVASIVE` + flag `media_conflictive` (Callinectes y Rugulopteryx nuevos; Pterois ya estaba).
2. Alertas curadas para Callinectes y Rugulopteryx (mismo bloque que Pterois).
3. `yf_species`: Callinectes `prio` 0→4; common_es; flag mediática en las tres. (Pterois `common_es` corregido de «Rana» → «Pez león».)
4. Umbrales `calibration_hierarchical.by_species` de las tres = **0,8467** (como género Pterois). AutoID global no tocado.
5. Awards / Academy HTML / admin invasoras / descarga BQ actualizados.
6. Artifact: `artifacts/invasoras/especies_mediaticas_conflictivas_20261013.json` (+ espejo en docs).
7. Cola de cosecha: solo *Pterois miles* (galería corta + acierto bajo). Callinectes y Rugulopteryx ya ~1000 fotos.

**No** se añadió Callinectes a la lista UE (`target_species.json`): no es especie de preocupación de la Unión. Sí está en Weitzmann marinas. Rugulopteryx ya estaba en la lista UE.

## Galería y eval (calib_raw_t05 + stats.json, 13-oct)

| Especie | Fotos | Vectores FAISS | Eval n | Acierto | Confusiones principales |
|---------|------:|---------------:|-------:|--------:|-------------------------|
| *Pterois miles* | 546 | 500 | 40 | **60 %** | *P. volitans* 14, *Arbacia lixula* 2 |
| *Callinectes sapidus* | 978 | 978 | 30 | 90 % | *Caretta*, *Mugil*, *Cronius* (1 c/u) |
| *Rugulopteryx okamurae* | 1004 | 1027 | 30 | 86,7 % | *Valonia*, *Posidonia*, *Octopus*, *Dictyota* (1 c/u) |

Cola cosecha: `artifacts/invasoras/cola_cosecha_mediaticas_20261013.json` y `dataset/cola_cosecha_mediaticas_20261013.txt` → `pterois_miles`.
