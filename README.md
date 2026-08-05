# FotoFauna — AI-Assisted Mediterranean Marine Species Identification

[![Status](https://img.shields.io/badge/status-active-brightgreen)]()
[![Species](https://img.shields.io/badge/species-1,369-blue)]()
[![Platform](https://img.shields.io/badge/platform-Minka-orange)]()

**Live**: https://fotofauna.yespi.es | **AI Engine**: https://github.com/yespi/yolofauna

## Overview

FotoFauna is a citizen science platform for the Mediterranean Sea. Users upload photographs of marine fauna and receive instant AI-powered species identifications. High-confidence predictions are auto-published to the Minka citizen science network, where professional taxonomists validate the identifications.

## How It Works

1. 📸 **Upload** a photograph of any Mediterranean marine organism
2. 🤖 **AI identifies** it using YOLOFauna (BioCLIP fine-tuned on 525K images of 1,369 species)
3. ✅ **Auto-publish** to Minka when confidence ≥ 90% (92% precision)
4. 🔬 **Curator review** by professional taxonomists (Xavier Salvador, Miquel Pontes)

## Key Metrics

| Metric | Value |
|--------|-------|
| Species in model | 1,369 |
| Training images | 525,253 |
| Species accuracy | 63.9% |
| Weighted accuracy | 71.8% |
| High-conf precision (p≥0.90) | 92.2% |
| Auto-published to Minka | 100+ |
| Curator confirmation rate | 100% (21/21) |

## Architecture

```
User → FotoFauna Web → YOLOFauna AI → Identification
                              ↓
                         Minka API → Auto-publish
                              ↓
                         Curator validation → Feedback
```

## Paper

See [paper/yolofauna.md](https://github.com/yespi/yolofauna/blob/master/paper/02_fotofauna.md) for the full research paper.

## License

MIT — https://github.com/yespi/fotofauna
