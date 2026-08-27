# FotoFauna — Feature Documentation

## Table of Contents
1. [Photo Upload & Organism Detection](#1-photo-upload--organism-detection)
2. [Multi-Engine Identification Pipeline](#2-multi-engine-identification-pipeline)
3. [Confidence Calibration & Auto-Publication](#3-confidence-calibration--auto-publication)
4. [Wave System (Batch AutoID)](#4-wave-system-batch-autoid)
5. [Photo Gallery & Search](#5-photo-gallery--search)
6. [Species Academy](#6-species-academy)
7. [Geographic & Seasonal Context](#7-geographic--seasonal-context)
8. [Admin Panel](#8-admin-panel)

## 1. Photo Upload & Organism Detection

### 1.1 Upload Flow
Users upload photos via drag-and-drop or file picker. Accepted formats: JPEG, PNG, WebP. Max size: 20 MB.

### 1.2 YOLO-Based Organism Detection
Before identification, the system optionally detects and crops the organism from the photo using YOLOv8-nano segmentation:
- Detects the bounding box of the organism
- Crops to isolate the subject from background
- Multiple organisms per photo are detected and identified separately
- Crops stored as temporary files for the identification pipeline

### 1.3 Manual Crop Tool
Users can manually adjust the crop region with drag handles, click-and-drag repositioning, and real-time preview. Server-side vision filters apply to the active crop rectangle when one exists, not the full frame.

### 1.4 Image Preprocessing
Resize to 224x224 for BioCLIP, EXIF rotation, thumbnail generation (400px).

## 2. Multi-Engine Identification Pipeline

### 2.1 YOLOFauna (Primary — Local GPU)
- BioCLIP ViT-L/14 fine-tuned with QLoRA, <1s latency
- 1,369 Mediterranean marine species
- k-NN (k=25) with cosine similarity
- Multi-level logistic calibration (ECE=0.045)
- Taxonomic abstention to genus/family when uncertain

### 2.2 iNaturalist Computer Vision (Fallback)
- api.inaturalist.org/v1/computervision/score_image
- 2-5s latency, 80,000+ global taxa
- JWT token renewed hourly via cron

### 2.3 Minka Computer Vision (Tertiary)
- minka-sdg.org API, 3-6s latency
- Mediterranean-focused taxa

### 2.4 Gemini Vision & Groq Vision (Optional AI)
- Google Gemini 2.0 Flash and Llama 3.2 Vision for edge cases

### 2.5 Identification Fusion
Best result selected by priority: YOLOFauna (p>=0.90) → iNat CV → Minka CV

## 3. Confidence Calibration & Auto-Publication

10 k-NN features mapped to calibrated probabilities via logistic regression:

| p_species >= | Precision | Coverage |
|-------------|-----------|----------|
| 0.90 | 92.2% | 30% |
| 0.85 | 92.1% | 38% |
| 0.80 | 90.5% | 43% |

Auto-publish to Minka when p>=0.90. Curator review by xasalva, bertinhaco, mpontes.

## 4. Wave System (Batch AutoID)

Batch processes un-identified observations on configurable schedule:
- WAVE_YOLOFAUNA_MIN_P_SPECIES (default 0.90)
- WAVE_BATCH_SIZE (default 200)
- WAVE_COOLDOWN_MIN (default 50)
- All publications recorded in autoid_history

## 5. Photo Gallery & Search

Infinite-scroll gallery with WebP thumbnails. Filter by species, date, location, user. Autocomplete search across scientific + common names (Minka + iNat APIs). Observation detail with full photo, identification history, GPS map, taxonomic tree.

## 6. Species Academy

1,369 species cards with: representative photos, scientific/common names, WoRMS-validated taxonomy, IUCN conservation status, GROC/OPK morphological descriptions, similar species, depth/habitat/seasonality. Nightly thumbnail pre-caching from iNaturalist.

## 7. Geographic & Seasonal Context

GPS EXIF auto-geotagging. Geo priors: 77,244 occurrence points for 1,348 species (Haversine distance, Gaussian boost σ=200km). Seasonal awareness for migration/breeding patterns.

## 8. Admin Panel

Dashboard (observations, species, AutoID stats, GPU status). Species management (add/edit/import, WoRMS validation). AutoID configuration (wave scheduler, thresholds, engine selection). System monitoring (Docker, PostgreSQL, GPU, API limits, error logs).
