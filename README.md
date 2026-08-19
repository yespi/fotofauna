# FotoFauna — AI-Assisted Mediterranean Marine Species Identification

[![Status](https://img.shields.io/badge/status-active-brightgreen)]()
[![Species](https://img.shields.io/badge/species-1,369-blue)]()
[![Platform](https://img.shields.io/badge/platform-Minka-orange)]()

**Live**: https://fotofauna.yespi.es | **AI Engine**: [BioFauna](https://github.com/yespi/biofauna) (formerly YOLOFauna)

## Overview

FotoFauna is a citizen science platform for the Mediterranean Sea. Users upload photographs of marine fauna and receive instant AI-powered species identifications via **BioFauna** (BioCLIP-2.5 ViT-H + k-NN). High-confidence predictions are auto-published to the Minka citizen science network, where professional taxonomists validate the identifications.

> **Aug 2026:** Active remediation — consolidating SSD + HDD photo archive into embeddings to recover accuracy on rich species. See [BioFauna status](https://github.com/yespi/biofauna/blob/master/docs/STATUS.md).

## How It Works

1. 📸 **Upload** a photograph of any Mediterranean marine organism
2. 🤖 **AI identifies** it using BioFauna (BioCLIP-2.5 ViT-H, ~3,000 species gallery)
3. ✅ **Auto-publish** to Minka when confidence ≥ 90% (92% precision)
4. 🔬 **Curator review** by professional taxonomists (Xavier Salvador, Miquel Pontes)

## Key Metrics

| Metric | Value |
|--------|-------|
| Species in model | ~3,000 (Aug 2026 remediation) |
| Training / gallery images | ~900K total (SSD + archive) |
| Tier-1 species accuracy (remediation OOS) | ~64% |
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

## License

MIT — https://github.com/yespi/fotofauna
