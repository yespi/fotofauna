# FotoFauna — AI-Assisted Mediterranean Marine Species Identification

[![Status](https://img.shields.io/badge/status-active-brightgreen)]()
[![Species](https://img.shields.io/badge/species-4,643-blue)]()
[![Platform](https://img.shields.io/badge/platform-Minka-orange)]()

**Live**: https://fotofauna.yespi.es | **AI Engine**: [BioFauna](https://github.com/yespi/biofauna) (formerly YOLOFauna)

## Overview

FotoFauna is a citizen science platform for the Mediterranean Sea. Users upload photographs of marine fauna and receive instant AI-powered species identifications via **BioFauna** (BioCLIP-2.5 ViT-H + k-NN). High-confidence predictions are auto-published to the Minka citizen science network, where professional taxonomists validate the identifications.

> **Oct 2026 (10 Oct):** BioFauna panel OOS **83.22%** (78,145 rows) after invasoras promote; genus 87.96%, family 91.04%; live index **1,216,896** vectors / **4,643** species. AutoID p≥0.83 ≈96% precision / ≈66.5% coverage. See BioFauna paper §5.4.
>
> **Oct 2026 (9 Oct):** OOS **83.14%** after all08it2; 1,198,265 vectors / 4,543 species.
>
> **Oct 2026 (6 Oct):** BioFauna index 1,132,767 vectors / 4,543 species, panel 82.85%; FotoFauna PRE→PRO deployed.
>
> **Aug 2026:** Active remediation — consolidating SSD + HDD photo archive into embeddings to recover accuracy on rich species.

## How It Works

1. 📸 **Upload** a photograph of any Mediterranean marine organism
2. 🤖 **AI identifies** it using BioFauna (BioCLIP-2.5 ViT-H, ~3,000 species gallery)
3. ✅ **Auto-publish** to Minka when confidence ≥ 90% (92% precision)
4. 🔬 **Curator review** by professional taxonomists (Xavier Salvador, Miquel Pontes)

## Key Metrics

| Metric | Value |
|--------|-------|
| Species in model | 4,643 (2026-10-10; 1,216,896 gallery embeddings after invasoras promote) |
| Training / gallery images | ~1.22M on disk (SSD + archive); ~1.22M embedded |
| Field panel species accuracy (out-of-sample) | 83.22% (78,145 rows; series …→83.14 all08it2→83.22 invasoras; genus 87.96%, family 91.04%) |
| Published baseline (2026 paper cohort) | 71.7% |
| High-conf precision (p≥0.90) | 92.2% |
| Auto-published to Minka | 100+ |
| Curator confirmation rate | 100% (21/21) |

## Architecture

```
User → FotoFauna Web → BioFauna AI → Identification
                              ↓
                         Minka API → Auto-publish
                              ↓
                         Curator validation → Feedback
```

## Paper

See [paper/yolofauna.md](https://github.com/yespi/yolofauna/blob/master/paper/02_fotofauna.md) for the full research paper.

## BioFauna Fotos (admin export)

Administrators can download the BioFauna photo catalog from FotoFauna as independent ZIP files (≤ 2 GB), with a CSV inside each species folder. See [docs/BIOFAUNA_FOTOS.md](docs/BIOFAUNA_FOTOS.md).

## License

MIT — https://github.com/yespi/fotofauna
