# FotoFauna — Crop Section & Upload Flow

## Upload Flow (Step by Step)

### Step 1: File Selection
- **Drag & drop**: Drop photos anywhere on the page
- **File picker**: Click to open system file dialog
- **Clipboard paste**: Ctrl+V to paste from clipboard
- **Accepted formats**: JPEG, PNG, WebP
- **Max file size**: 20 MB per photo
- **Multiple upload**: Select multiple photos at once

### Step 2: EXIF Extraction
- GPS coordinates extracted from EXIF metadata
- Date/time of capture preserved
- Camera orientation applied (auto-rotate)
- If no GPS: user can manually set location on map

### Step 3: Pre-processing
- Convert to RGB colorspace
- Generate thumbnail (400px wide, WebP)
- Strip EXIF for privacy before storage
- Hash computation for duplicate detection

### Step 4: Organism Detection (YOLO)
- **Model**: YOLOv8-nano segmentation (`yolo26n-seg.pt`)
- Detects bounding box of the organism
- Separates subject from background
- Multiple organisms per photo → multiple detections
- Confidence threshold: configurable
- **Fallback**: If no organism detected, the full image is used

### Step 5: Crop Tool (Manual Adjustment)
Users can manually refine the crop:

#### Crop Handles
- **8 drag handles**: 4 corners + 4 edge midpoints
- **Drag corner**: Resize freely (maintains aspect ratio optional)
- **Drag edge**: Resize in one dimension
- **Drag center**: Reposition the crop region

#### Aspect Ratio Presets
- **Free**: No constraint
- **Square (1:1)**: For profile/avatar photos
- **4:3**: Standard photo ratio
- **16:9**: Widescreen
- **Auto-detect**: Matches the detected organism's natural proportions

#### Visual Aids
- **Rule of thirds grid**: Overlay for composition
- **Zoom indicator**: Shows crop region size relative to original
- **Edge detection**: Highlight the organism boundaries
- **Before/after toggle**: Compare crop with original

#### Image Adjustments
- **Brightness**: -100 to +100
- **Contrast**: -100 to +100
- **Saturation**: 0 to 200% (default 100%)
- **Sharpness**: 0 to 100%
- **Reset button**: Return to original

### Step 6: Multi-Crop Management
When multiple organisms are detected in one photo:
- **Crop list**: Sidebar showing all detected crops
- **Add crop**: Manually add additional crop region
- **Delete crop**: Remove unwanted detections
- **Reorder**: Drag to reorder crops
- **Individual adjustments**: Each crop has independent settings
- **Batch apply**: Apply same adjustments to all crops

### Step 7: Species Pre-identification (Optional)
Before publishing, the user can see AI suggestions:
- **YOLOFauna quick ID**: Instant species suggestion
- **Confidence indicator**: Visual bar showing p_species
- **Alternative suggestions**: Top-5 species with similarity scores
- **Taxonomic note**: If abstaining to genus/family

### Step 8: Metadata Entry
- **Scientific name**: Pre-filled from AI, editable
- **Common name**: Auto-populated
- **Location**: From GPS or manual map pin
- **Date observed**: From EXIF or manual
- **Depth**: If diving, manually entered
- **Habitat notes**: Free text
- **Visibility**: Public / Unlisted / Private

### Step 9: Publish / Save
- **Publish to Minka**: If confidence ≥ 0.90 and species verified
- **Save to gallery**: Always saved locally
- **Queue for review**: If confidence < 0.90
- **Discard**: Delete photo and data

## Crop Section Features (Detail)

### Keyboard Shortcuts
| Key | Action |
|-----|--------|
| Enter | Confirm crop |
| Escape | Cancel / Reset |
| R | Reset crop to full image |
| 1-4 | Select aspect ratio preset |
| Arrow keys | Nudge crop region (1px) |
| Shift+Arrows | Nudge crop region (10px) |

### Touch/Mobile Support
- Pinch to zoom
- Two-finger pan
- Touch drag for crop handles
- Double-tap to reset
- Haptic feedback on snap

### Crop Validation
- Minimum crop size: 100×100 pixels
- Warning if crop is too small for reliable AI identification
- Suggestion to use full image if crop quality is poor
- Resolution check: alerts if output < 224×224 (BioCLIP minimum)

### Performance
- Canvas-based rendering (hardware accelerated)
- Debounced redraw during drag operations
- Lazy loading of full-resolution image
- WebWorker for heavy image processing
- Progressive JPEG support for fast preview
