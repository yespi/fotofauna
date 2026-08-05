# FotoFauna: A Citizen Science Platform for AI-Assisted Mediterranean Marine Species Identification

**Authors**: Gustavo Zafra (Yespi)
**Repository**: https://github.com/yespi/fotofauna
**Live**: https://fotofauna.yespi.es

## Abstract

FotoFauna is a web-based citizen science platform that integrates automated AI species identification with community validation for Mediterranean marine fauna. The platform combines a region-specific AI engine (YOLOFauna) trained on 525,253 images of 1,369 species with a multi-engine identification pipeline, organism detection via YOLOv8 segmentation, and automated publication to the Minka citizen science network. High-confidence identifications (calibrated probability >= 0.90) are auto-published with 92.2% precision, achieving 100% curator confirmation rate on 21 reviewed observations. The platform has processed 32,686 observations and serves as both a data collection tool and a testbed for AI-assisted identification workflows. This paper describes the platform architecture, identification pipeline, auto-publication system, and the feedback loop between automated and expert-curated identifications.

## 1. Introduction

### 1.1 The Mediterranean Identification Challenge

The Mediterranean Sea hosts over 17,000 marine species (Coll et al., 2010), yet the number of qualified taxonomists capable of identifying them continues to decline (Hopkins & Freckleton, 2002; Kim & Byrne, 2006). Citizen science platforms like iNaturalist and Minka have partially addressed this gap through community-based identification, but the process remains slow — observations can wait days or weeks for expert attention, and rare species may never be identified.

### 1.2 AI-Assisted Citizen Science

Automated image-based identification offers a complementary approach: providing instant species suggestions that accelerate the identification pipeline. Recent advances in vision-language models, particularly BioCLIP (Stevens et al., 2024), have enabled region-specific fine-tuning on consumer hardware (Zafra, 2026), making AI-assisted identification feasible for specialized platforms.

### 1.3 Platform Goals

FotoFauna was developed with three primary goals:

1. **Speed**: Provide instant species identification (<2 seconds) for Mediterranean marine photographs
2. **Accuracy**: Achieve >90% precision on auto-published identifications through calibrated confidence thresholds
3. **Feedback**: Create a virtuous cycle where AI identifications are validated by experts, and corrections feed back into model improvement

## 2. Platform Architecture

### 2.1 System Overview

FotoFauna runs on a self-hosted Ubuntu server with the following components:

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Frontend | Vanilla JS SPA + WebP | User interface, photo upload, gallery |
| Backend API | FastAPI (Python) | Business logic, authentication, routing |
| AI Engine | YOLOFauna (BioCLIP + QLoRA) | Species identification |
| Database | PostgreSQL + PostGIS | Observations, users, species catalog |
| Proxy | Nginx | SSL termination, caching, routing |
| GPU | NVIDIA RTX 3060 (12 GB) | AI inference (<1s/image) |
| Container | Docker Compose | Service orchestration |

### 2.2 Authentication

Users authenticate via Google OAuth 2.0 with JWT token sessions. Three access levels:
- **Public**: Browse gallery, view species academy
- **Authenticated**: Upload photos, suggest identifications
- **Admin**: Manage species, configure AutoID, access analytics

### 2.3 External Integrations

| Service | Integration | Purpose |
|---------|------------|---------|
| Minka API | REST | Publish observations, fetch taxonomy |
| iNaturalist API | REST + JWT | Computer vision fallback, taxonomy search |
| GROC/OPK | Data import | Morphological descriptions, species validation |
| WoRMS | API | Taxonomic name validation and synonym resolution |

## 3. Photo Upload and Processing

### 3.1 Upload Interface

The upload system supports:
- Drag-and-drop (desktop)
- File picker dialog
- Clipboard paste (Ctrl+V)
- Multiple simultaneous uploads
- Accepted formats: JPEG, PNG, WebP (max 20 MB)

### 3.2 EXIF Processing

Upon upload, the system extracts:
- GPS coordinates (for geographic priors and mapping)
- Capture date/time (for seasonal context)
- Camera orientation (auto-rotate)
- Camera model and settings (metadata only, not used for identification)

If GPS data is absent, the user can manually place a pin on an interactive Leaflet/OpenStreetMap.

### 3.3 Image Preprocessing

Images undergo a preprocessing pipeline:
1. Convert to RGB colorspace
2. Strip EXIF metadata for privacy
3. Generate WebP thumbnails (200px, 400px, 800px)
4. Compute perceptual hash (pHash) for duplicate detection
5. Store original resolution for archival

### 3.4 Organism Detection (YOLO)

Before identification, the system optionally detects and crops the organism from the photo using YOLOv8-nano segmentation (yolo26n-seg.pt):

**Detection Process:**
- Scans the full image for organism bounding boxes
- Applies confidence threshold (configurable)
- Returns crop regions with coordinates
- Handles multiple organisms per photo

**Benefits of Cropping:**
- Removes background noise (water, rocks, sand)
- Isolates the subject for more accurate embedding
- Enables multi-organism identification from group photos
- Standardizes input to BioCLIP (224x224 center crop on detected region)

### 3.5 Manual Crop Tool

Users can manually refine or create crops using an interactive editor:

**Crop Controls:**
- 8 drag handles (4 corners + 4 edge midpoints)
- Center-drag for repositioning
- Aspect ratio presets (free, 1:1, 4:3, 16:9)
- Rule of thirds overlay grid

**Image Adjustments:**
- Brightness (-100 to +100)
- Contrast (-100 to +100)
- Saturation (0-200%)
- Sharpness (0-100%)
- Reset to original

**Keyboard Shortcuts:**
| Key | Action |
|-----|--------|
| Enter | Confirm crop |
| Escape | Cancel/Reset |
| R | Reset to full image |
| 1-4 | Aspect ratio presets |
| Arrows | Nudge 1px |
| Shift+Arrows | Nudge 10px |

**Mobile/Touch Support:**
- Pinch zoom
- Two-finger pan
- Touch-drag handles
- Double-tap reset
- Haptic feedback on snap

**Multi-Crop Mode:**
When multiple organisms are detected in one photo:
- Sidebar list of all detected crops
- Manual add/delete crop regions
- Independent settings per crop
- Batch adjustments across all crops
- Reorder via drag-and-drop

## 4. Multi-Engine Identification Pipeline

FotoFauna queries multiple identification engines with a priority-based fusion strategy.

### 4.1 YOLOFauna (Primary)

- **Model**: BioCLIP ViT-L/14 fine-tuned with QLoRA
- **Coverage**: 1,369 Mediterranean marine species
- **Latency**: <1 second
- **Method**: k-NN (k=25) with cosine similarity on 768-dim embeddings
- **Calibration**: Multi-level logistic regression (ECE=0.045)
- **Geographic priors**: GPS-weighted scoring (77,244 points for 1,348 species)

### 4.2 iNaturalist Computer Vision (Fallback)

- **Endpoint**: api.inaturalist.org/v1/computervision/score_image
- **Coverage**: 80,000+ global taxa
- **Latency**: 2-5 seconds
- **Authentication**: JWT token, renewed hourly via cron
- **Rate limiting**: Semaphore with max 5 concurrent requests
- **Trigger**: When YOLOFauna p_species < 0.90

### 4.3 Minka Computer Vision (Tertiary)

- **Coverage**: Mediterranean-focused taxa
- **Latency**: 3-6 seconds
- **Used when**: Both YOLOFauna and iNaturalist CV are unavailable or low-confidence

### 4.4 AI Vision Models (Experimental)

- **Gemini 2.0 Flash** (Google): General vision model for edge cases
- **Groq Vision** (Llama 3.2): Alternative AI opinion
- **OpenRouter**: Free vision models for comparison

### 4.5 Identification Fusion

The best identification is selected by priority:
1. YOLOFauna with p_species >= 0.90 -> used directly
2. YOLOFauna with lower confidence -> compared with iNat CV
3. iNat CV as primary fallback
4. Minka CV as secondary fallback
5. AI vision models for consensus when engines disagree

## 5. Confidence and Auto-Publication

### 5.1 Calibrated Confidence

Raw cosine similarity scores are not probabilities. A logistic regression calibrator maps 10 k-NN features to well-calibrated probability estimates (ECE=0.045):

**Features used:**
- s1, s2, margin (top similarity scores)
- votes1, share1 (k-NN voting statistics)
- lognref1 (reference set size)
- meansim, kclasses (distribution characteristics)
- same_genus_12, same_family_12 (taxonomic coherence)

### 5.2 Auto-Publication Thresholds

| p_species >= | Precision | Coverage | Action |
|-------------|-----------|----------|--------|
| 0.90 | 92.2% | 30% | Auto-publish |
| 0.85 | 92.1% | 38% | Optional auto-publish |
| 0.80 | 90.5% | 43% | Flag for review |
| 0.75 | 88.7% | 49% | Manual review recommended |

### 5.3 Taxonomically-Aware Publication

When the model abstains to genus or family level, the observation is published with the higher taxonomic rank and a note indicating uncertainty at the species level. This allows curators to quickly identify observations that need species-level review.

### 5.4 Publication to Minka

Observations are published to Minka via its REST API:
- Scientific name (or genus/family when abstaining)
- Calibrated probability
- GPS coordinates and date
- Photograph URLs (hosted on FotoFauna)
- AI engine version and identification source
- Taxonomic notes

## 6. Wave System (Batch AutoID)

The Wave system processes batches of uploaded-but-unidentified observations on a configurable schedule.

### 6.1 Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| WAVE_YOLOFAUNA_MIN_P_SPECIES | 0.90 | Minimum calibrated probability |
| WAVE_BATCH_SIZE | 200 | Observations per wave |
| WAVE_COOLDOWN_MIN | 50 | Minutes between waves |
| WAVE_MAX_WAVES | -1 | Maximum waves (unlimited) |

### 6.2 Wave Processing Pipeline

For each wave:
1. Query database for unprocessed observations
2. Download photo (if external URL)
3. Run organism detection (YOLOv8)
4. Run identification pipeline
5. Check confidence against threshold
6. If >= threshold: publish to Minka
7. If < threshold: save for manual review
8. Record in autoid_history table

### 6.3 Autoid History

All auto-published observations are recorded with:
- Observation ID and URI
- Scientific name and confidence
- Identification source (YOLOFauna, iNat CV, Minka CV)
- Timestamp
- Publication status

## 7. Photo Gallery and Search

### 7.1 Gallery Features

- Infinite-scroll grid with lazy loading
- WebP thumbnails for fast loading
- Filter by species, date, location, user
- Sort by newest, species name, confidence
- Responsive grid (1-4 columns)

### 7.2 Species Search

Autocomplete search with debounced input (300ms):
- Searches across scientific names, common names, Catalan names
- Sources: Minka taxonomy API + iNaturalist taxonomy API
- Results cached in PostgreSQL
- Fuzzy matching for misspellings

### 7.3 Observation Detail View

Full observation page with:
- Full-resolution photo viewer (zoom/pan)
- Identification history (all engine results)
- Confidence breakdown with visual indicator
- Interactive GPS map (Leaflet/OpenStreetMap)
- Taxonomic tree (species -> genus -> family -> order)
- Similar observations (same species, nearby location)
- Curator review status

## 8. Species Academy

### 8.1 Species Cards

Each of 1,369 species has a detail card with:
- Representative photo gallery
- Scientific and common names (Catalan/Spanish/English)
- WoRMS-validated taxonomy
- IUCN conservation status
- Morphological description (from GROC/OPK)
- Similar species
- Depth, habitat, seasonality

### 8.2 Thumbnail Caching

Nightly cron job:
- Downloads research-grade photos from iNaturalist
- Generates WebP thumbnails at 200px, 400px, 800px
- Caches locally for instant loading
- Fallback to Minka photos

## 9. Geographic and Seasonal Context

### 9.1 GPS Integration

- Automatic geotagging from EXIF
- Manual location via map pin or text search
- Reverse geocoding for place names

### 9.2 Geo Priors

YOLOFauna geographic priors:
- 77,244 occurrence points for 1,348 species
- Haversine distance (great-circle)
- Gaussian boost: sigma=200km, max 3x multiplier
- No penalty for missing data (boost=1.0)

### 9.3 Seasonal Awareness

- Observation date affects probability
- Seasonal species patterns boost in-season
- Migration/breeding accounted for

## 10. Curator Feedback Loop

### 10.1 Publication Status Tracking

Each auto-published observation is tracked through its lifecycle:
1. Pending (published, awaiting review)
2. Confirmed (curator agreed with AI identification)
3. Corrected (curator changed the identification)
4. Rejected (curator determined unidentifiable)

### 10.2 Feedback Integration

Curator actions feed back into the system:
- **Confirmations**: Added to training data with high weight
- **Corrections**: Used as hard negative pairs for triplet loss
- **Rejections**: Flag species for additional training data collection

### 10.3 Validation Results

As of August 2026:
- 100 observations auto-published
- 21 curator-reviewed
- 100% confirmation rate
- Zero corrections

## 11. Deployment Results

### 11.1 Usage Statistics

| Metric | Value |
|--------|-------|
| Total observations | 32,686 |
| Auto-published to Minka | 100 |
| AI identifications (YOLOFauna) | 18% |
| AI identifications (iNaturalist CV) | 81% |
| Average identification time | <2 seconds |
| Platform uptime | >99% |

### 11.2 Curator Community

The platform has established relationships with professional taxonomists:
- Xavier Salvador (GROC/OPK): primary reviewer
- Miquel Pontes (GROC/OPK): species validation
- Manuel Ballesteros (UB): taxonomic authority

## 12. Discussion

### 12.1 AI-Human Collaboration

FotoFauna demonstrates a practical model for AI-human collaboration in citizen science. Rather than replacing human expertise, the AI accelerates the pipeline by handling routine identifications (30% at high confidence), freeing curators to focus on challenging cases. The 100% curator confirmation rate suggests the AI is appropriately conservative.

### 12.2 Platform Sustainability

The self-hosted architecture, consumer GPU, and open-source model release ensure that the platform can be replicated by other research groups without dependence on commercial AI APIs or cloud infrastructure.

### 12.3 Limitations

- Language: Interface primarily in Spanish
- Self-hosting complexity
- Species coverage gaps (394/1369 lack training images)
- Small curator review sample (21/100)
- Geographic scope limited to Mediterranean

## 13. Conclusion

FotoFauna demonstrates that AI-assisted citizen science is practical, accurate, and sustainable on consumer hardware. The combination of region-specific AI, calibrated confidence, and expert-curated feedback creates a platform that accelerates biodiversity data collection while maintaining high taxonomic standards. The 100% curator confirmation rate validates the approach, and the open-source release enables replication for other regions and taxonomic groups.

## References

[References shared with YOLOFauna paper]

---

*Paper in preparation. Version 2026-08-05.*
