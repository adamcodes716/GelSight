# GelSight Data Architecture Overview (Visual)

---

## **THE DATA FLOW**

```
PHYSICAL GELSIGHT DEVICE
    ↓
    ├─→ Captures surface images (6 photos per angle)
    ├─→ Measures 3D height map (.tmd binary file)
    ├─→ Detects geometric shapes (circles, lines, rectangles)
    └─→ Runs analysis routines (Offset, DefectDetection, HoleDiameter, PitDetection, SurfaceRoughness)

OUTPUTS STORED IN NESTED FOLDERS:
    ↓
    GelSightAnalysis/
    ├── Offset/
    │   ├── scan.yaml                    ← Device context, image list
    │   ├── Scan-001.tmd                 ← 3D height map
    │   ├── Calib-4E07-XNJU_...yaml      ← Optical calibration
    │   └── analysis/scancontext.yaml    ← Shapes + measurements
    │
    ├── DefectDetection/
    │   ├── scan.yaml
    │   ├── G500-08b-repeat-001.tmd
    │   ├── Calib-4E07-XNJU_...yaml
    │   └── analysis/scancontext.yaml
    │
    ├── HoleDiameter/
    ├── PitDetection/
    └── SurfaceRoughness/

GOAL: INGEST ALL OF THIS INTO CLOUD PLATFORM
    ↓
```

---

## **THREE-LAYER ARCHITECTURE (Medallion Pattern)**

```
┌─────────────────────────────────────────────────────────────────┐
│                      GOLD LAYER (Analytics)                     │
│                                                                 │
│  • BI Dashboards (Quality reporting, trends)                   │
│  • ML Training Datasets (Defect detection models)              │
│  • Pass/Fail Analysis (QA decisions)                           │
│  • Time-series trends (Measurement variations)                 │
│  • Operator performance metrics                                │
│                                                                 │
│  (This is where Operators see the results)                      │
└─────────────────────────────────────────────────────────────────┘
                              ↑
                    Transform & Normalize
                              ↑
┌─────────────────────────────────────────────────────────────────┐
│                     SILVER LAYER (Modeling)                     │
│                                                                 │
│  dim_calibration  (Optical calibration, versioned)            │
│  dim_device       (GelSight hardware, configurations)          │
│  dim_shapes       (Circles, lines, rectangles detected)        │
│  dim_routines     (Offset, HoleDiameter, DefectDetection, ...) │
│  fact_measurements (Unified or routine-specific)               │
│  fact_calibration_lineage (Which calib for which scan)        │
│  dim_analysis_versions (SDK/firmware/app version combos)       │
│                                                                 │
│  • Coordinates converted to mm (from pixels)                   │
│  • Shapes normalized (pixel → real-world)                      │
│  • Calibration versioning tracked (SCD Type 2)                 │
│  • Measurements validated against business rules               │
│                                                                 │
│  (Data modeler designs this layer)                             │
└─────────────────────────────────────────────────────────────────┘
                              ↑
                    Parse & Standardize
                              ↑
┌─────────────────────────────────────────────────────────────────┐
│                     BRONZE LAYER (Raw Data)                     │
│                                                                 │
│  /gelsight-bronze/scans/
│    └── {scan_guid}/
│        ├── scan_metadata.json      (parsed scan.yaml)         │
│        ├── analysis_results.json   (parsed scancontext.yaml)  │
│        ├── images/                 (6 PNG images)             │
│        ├── heightmaps/             (.tmd binary file)         │
│        └── raw_yaml/               (preserve originals)       │
│                                                                 │
│  /gelsight-bronze/calibrations/
│    └── {calibration_filename}/
│        ├── calib_metadata.json     (parsed Calib-*.yaml)      │
│        └── raw_yaml/               (preserve original)        │
│                                                                 │
│  • One-to-one with source files                               │
│  • Lossless ingestion (keep originals)                        │
│  • Metadata tracking (lineage, versioning)                    │
│  • No business logic applied                                  │
│                                                                 │
│  (Data engineer designs this layer)                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## **KEY DATA ENTITIES & RELATIONSHIPS**

```
┌──────────────────┐
│   dim_device     │
├──────────────────┤
│ device_id        │
│ device_model     │
│ firmware_version │
│ camera_id        │
│ lens_serial      │
└────────┬─────────┘
         │
         │ 1:N
         │
┌────────▼──────────────┐
│   dim_calibration     │
├───────────────────────┤
│ calibration_id        │
│ device_id (FK)        │
│ version               │
│ created_date          │
│ mmperpixel            │
│ linfit_coeffs         │
│ quadfit_coeffs        │
│ is_verified           │
└────────┬──────────────┘
         │
         │ 1:N
         │
┌────────▼─────────────────┐
│  fact_measurements       │      ┌──────────────┐
├──────────────────────────┤      │  dim_shapes  │
│ measurement_id           │──┐   ├──────────────┤
│ scan_id (FK)             │  │   │ shape_id     │
│ calibration_id (FK)      │  ├──→│ scan_id (FK) │
│ shape_id (FK)            │──┤   │ type         │
│ routine_type             │  │   │ x, y         │
│ measurement_name         │  │   │ r / w / h    │
│ measurement_value        │  │   │ rotation     │
│ unit                     │  │   └──────────────┘
│ tolerance_min / max      │  │
│ pass_fail_flag           │  │   ┌──────────────────┐
│ confidence_score (?)     │  └──→│  dim_scan_meta   │
└──────────────────────────┘      ├──────────────────┤
                                   │ scan_id          │
                                   │ guid             │
                                   │ created_date     │
                                   │ username         │
                                   │ mmperpixel       │
                                   │ device_id (FK)   │
                                   │ sdk_version      │
                                   │ app_version      │
                                   └──────────────────┘
```

---

## **COORDINATE SYSTEM TRANSFORMATION**

```
SOURCE: scan.yaml + analysis/scancontext.yaml
  ↓
  shapes: 
    - id: 1619949791
      type: circle
      x: 1619.94 pixels          ← PIXEL SPACE
      y: 1419.31 pixels          ← Optical center at (0,0)
      r: 752.55 pixels

  scan.yaml:
    mmperpixel: 0.00274095190847

TRANSFORMATION CHAIN:
  ↓
  pixel_x × mmperpixel = real_world_x_mm (simple scaling)
  1619.94 × 0.00274095 = 4.438 mm
  
  ↓ (apply optical distortion correction)
  
  Calib-*.yaml:
    linfit: [−0.0140636658768, 0.133317116258, ...]  ← 6 LED rings × 18 coeff
    quadfit: [0.0349013603931, 0.00768228929447, ...]  ← 6 LED rings × 60 coeff
  
  ↓
  corrected_x = x + linear_term + quadratic_term
  
RESULT: BRONZE LAYER
  ✓ Store raw pixel coordinates (for audit)
  ✓ Store mmperpixel value used (for reproducibility)
  
RESULT: SILVER LAYER
  ✓ Compute mm coordinates (for analysis)
  ✓ Apply calibration correction (for accuracy)
  
RESULT: GOLD LAYER
  ✓ Use calibrated mm values (for dashboards)
```

---

## **SCHEMA HETEROGENEITY CHALLENGE**

```
One routine → One measurement type
─────────────────────────────────

OFFSET ROUTINE:
  ┌─────────────────────┐
  │ Offset Measurement  │
  ├─────────────────────┤
  │ offset: 0.214 mm    │
  │ width: 9.815 mm     │
  └─────────────────────┘

DEFECT DETECTION ROUTINE:
  ┌──────────────────────────────┐
  │ DefectDetection Measurement  │
  ├──────────────────────────────┤
  │ length: 1.105 mm             │
  │ width: 1.197 mm              │
  │ area: 0.779 mm²              │
  │ volume: 0.0219 mm³           │
  │ depth: 0.0937 mm             │
  │ point: (1327.9, 1087.7) px   │
  │ profile: [360 elevation pts]  │
  └──────────────────────────────┘

HOLE DIAMETER ROUTINE:
  ┌──────────────────────────┐
  │ HoleDiameter Measurement │
  ├──────────────────────────┤
  │ diameter: 1.911 mm       │
  │ circularity: 0.0305 mm   │
  │ nominal_diameter: 1.778  │
  └──────────────────────────┘

PROBLEM: No two routines have identical output schema!

SOLUTION (Designed by Data Modeler):

  Option A: GENERIC FACT TABLE
  ─────────────────────────────
  fact_measurements (
    measurement_id, scan_id, routine_type, shape_id,
    measurement_name, measurement_value, unit, tolerance_min, tolerance_max
  )
  
  Example rows:
    (101, scan-xyz, Offset, shape-123, 'offset', 0.214, 'mm', 0.200, 0.300)
    (102, scan-xyz, Offset, shape-123, 'width', 9.815, 'mm', 9.000, 10.000)
    (103, scan-xyz, DefectDetection, shape-124, 'length', 1.105, 'mm', 0, NULL)
    (104, scan-xyz, DefectDetection, shape-124, 'depth', 0.0937, 'mm', NULL, 0.100)

  Option B: ROUTINE-SPECIFIC FACT TABLES
  ──────────────────────────────────────
  fact_offset_measurements (scan_id, shape_id, offset_mm, width_mm, ...)
  fact_defect_measurements (scan_id, shape_id, length_mm, width_mm, area_mm2, volume_mm3, depth_mm, ...)
  fact_hole_diameter_measurements (scan_id, shape_id, diameter_mm, circularity_mm, ...)

  Option C: HYBRID
  ───────────────
  shared_measurements + routine_specific_columns

DATA MODELER CHOOSES THE APPROACH.
```

---

## **CALIBRATION AS SLOWLY-CHANGING DIMENSION**

```
PROBLEM: Same calibration used by multiple scans

Calib-4E07-XNJU_20250917_1255.yaml
  ↓
  Used by scan: DefectDetection (scan-001) → created Sept 17, 2025
  Used by scan: HoleDiameter (scan-002) → created Sept 17, 2025
  Used by scan: Offset (scan-003) → created Sept 17, 2025
  Used by scan: PitDetection (scan-004) → created Sept 17, 2025

THEN (Oct 1, 2025):

Calib-4E07-XNJU_20251001_1005.yaml (NEW VERSION)
  ↓
  Used by scan: DefectDetection (scan-005) → created Oct 1, 2025
  Used by scan: HoleDiameter (scan-006) → created Oct 1, 2025

SOLUTION: SCD TYPE 2 (Track History)

dim_calibration (SCD Type 2)
├─ calibration_sk (surrogate key)
├─ calibration_id (natural key, e.g., device_id + date)
├─ version
├─ mmperpixel
├─ linfit_coeffs
├─ quadfit_coeffs
├─ effective_date (when this calibration became active)
├─ end_date (when it was superseded)
└─ is_current

Example:
  (calib_sk=1, calibration_id='4E07-XNJU', version=1, mmperpixel=0.00274, 
   effective_date=2025-09-17, end_date=2025-09-30, is_current=false)
  (calib_sk=2, calibration_id='4E07-XNJU', version=2, mmperpixel=0.00274, 
   effective_date=2025-10-01, end_date=NULL, is_current=true)

BENEFIT: Can query "Which scan used which calibration version?"
```

---

## **VERSION TRACKING FOR REPRODUCIBILITY**

```
At least FOUR independent version streams:

A. SDK VERSION
   ┌─────────────────────┐
   │ SDK 4.2.240         │ ← Algorithm definitions
   │ (Algorithm logic)   │
   └─────────────────────┘
        ↓ may change in next version

B. DEVICE FIRMWARE
   ┌─────────────────────────┐
   │ Modulus Firmware: 170    │ ← Hardware behavior
   │ Series2 Firmware: 415    │
   └─────────────────────────┘
        ↓ may differ per device

C. APP VERSION
   ┌──────────────────────┐
   │ GelSight.Mobile 4.2.240.0 │ ← UI/UX features
   └──────────────────────┘
        ↓ may change

D. CALIBRATION VERSION
   ┌──────────────────────────┐
   │ Calib-4E07-XNJU_... (v1)  │ ← Optical constants
   │ Calib-4E07-XNJU_... (v2)  │
   └──────────────────────────┘
        ↓ may change

CHALLENGE: Which combination of versions was used for each scan?

SOLUTION: Track version context
─────────────────────────────────
dim_analysis_version_context
├─ version_context_id
├─ sdk_version
├─ app_version
├─ device_firmware
├─ calibration_version
├─ scan_date_range
└─ notes

BENEFIT:
  - Query: "All scans with SDK 4.2.240 on Modulus devices"
  - Enable reprocessing: "Run algorithm X with SDK 4.2.250"
  - Reproducibility: "Exactly which versions made this measurement?"
```

---

## **GELSIGHT'S LONG-TERM ML VISION**

```
TODAY (Manual Inspection):
   ┌──────────────────────────────────────────┐
   │ 1. Operator triggers scan                 │
   │ 2. Device captures images + 3D surface    │
   │ 3. Operator manually reviews results      │ ← Manual bottleneck
   │ 4. Operator adjusts parameters (if need)  │ ← Manual bottleneck
   │ 5. Operator documents pass/fail           │ ← Manual bottleneck
   │ 6. Results go to Excel/QA system          │
   └──────────────────────────────────────────┘

TOMORROW (Automated with ML on Device):
   ┌──────────────────────────────────────────┐
   │ 1. Operator triggers scan                 │
   │ 2. Device captures images + 3D surface    │
   │ 3. ML model runs on device (edge)         │ ← Fully automated!
   │ 4. Device returns pass/fail result        │ ← Instant decision
   │ 5. Measurement data sent to cloud         │ ← For record-keeping
   │ 6. Results auto-populated in QA system    │ ← No human intervention
   └──────────────────────────────────────────┘

THIS PLATFORM ENABLES THAT FUTURE:

  Step 1: Build data warehouse (this project)
            ↓
  Step 2: Train ML models on cloud data
            ↓
  Step 3: Export trained models to device
            ↓
  Step 4: Deploy models to edge (on device hardware)
            ↓
  RESULT: Autonomous inspection, instant decisions, no manual steps

WHY THIS PLATFORM MATTERS:
  - Clean, consistent training data
  - Version tracking (know which model version made which prediction)
  - Feedback loops (edge predictions feed back to cloud for retraining)
  - Compliance (audit trail of measurements and decisions)
```

---

## **FILE EXAMPLE: WHAT EACH FILE LOOKS LIKE**

```
GelSightAnalysis/DefectDetection/

1. scan.yaml (260 lines)
   ─────────────────────
   version: 3.0
   guid: bd691d23-5e89-4e38-96a1-ccf8fb41f6b9
   createdon: 2025-09-17 12:58:41
   mmperpixel: 0.00274095190847
   images: [aligned01.png, ..., aligned06.png]
   activeheightmap: G500-08b-repeat-001.tmd
   calib: C:\Users\Public\...\Calib-4E07-XNJU_20250917_1255.yaml
   metadata:
     username: Kimo Johnson
     gelid: 4E07-XNJU
     gelusecount: 17
   camera: { cameraid: 00020015, cameratype: Ximea, ... }
   device: { serialnumber: 00020015, devicemodel: Modulus, ... }

2. analysis/scancontext.yaml (3000+ lines)
   ──────────────────────────────
   version: 3.7
   shapes:
     - id: 1285316935
       type: rectangle
       x: 662.14, y: 545.77, w: 1360.40, h: 1149.72, rotation: 0
   routines:
     - type: DefectDetection
       id: 881565834
       length: 1.10483882013 mm
       width: 1.19671271864 mm
       area: 0.77854575067 mm²
       volume: 0.0218922324125 mm³
       depth: 0.0937405147929 mm
       point: (1327.9, 1087.7)
       profile: [(0, 0.00306), (0.00274, 0.00276), ...]  ← 360+ points
       plot: [...]

3. Calib-4E07-XNJU_20250917_1255.yaml (400 lines)
   ─────────────────────────────────────────
   version: 3.0
   createdon: 2025-09-17 12:55:50
   mmperpixel: 0.00274095190847
   nL: 6
   width: 2472, height: 2064
   black: [0.234, 0.232, 0.232, 0.239, 0.250, 0.243]  ← Per LED ring
   white: [0.852, 0.873, 0.864, 0.868, 0.875, 0.858]
   parameters: { datafactor: 5000, fov: 6.09765625, ... }
   linfit:
     rows: 6, cols: 18
     values: [-0.014, 0.133, 0.017, -0.216, ...]  ← 108 coefficients
   quadfit:
     rows: 6, cols: 60
     values: [0.034, 0.007, 0.008, -0.034, ...]   ← 360 coefficients

4. G500-08b-repeat-001.tmd
   ──────────────────────
   (Binary file, not human-readable)
   - Contains height map (2D array of Z values)
   - Format: GelSight proprietary
   - Size: ~1-2 MB per scan

5. Images: aligned01.png, ..., aligned06.png
   ────────────────────────────────────
   - 6 images per scan (multi-view)
   - Resolution: 2472×2064 pixels
   - Size: ~4-5 MB per image (~30 MB per scan)
```

---

**For detailed explanations, see: DATA_INVENTORY.md**

