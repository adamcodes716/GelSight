# GelSight Data Quick Reference

**Purpose**: One-page cheat sheet for understanding the data structure.

---

## **The Three File Types You'll See**

### **1. `scan.yaml` (Device Context)**
- **What it is**: Metadata about the scan equipment and parameters
- **Where**: Each routine folder has one
- **Key fields**:
  - `guid` - Unique scan ID
  - `mmperpixel` - How to convert pixels to millimeters
  - `calib` - Points to a calibration file
  - `metadata.username`, `metadata.gelid` - Who scanned, what sensor used
  - `images`, `activeheightmap` - File references

---

### **2. `analysis/scancontext.yaml` (Shapes + Measurements)**
- **What it is**: The routine-specific results (shapes detected, measurements computed)
- **Where**: Inside `analysis/` subfolder of each routine
- **Two parts**:
  - **Shapes**: Geometric primitives (circles, lines, rectangles, rulers) with pixel coordinates
  - **Routines**: Measurement results (offset, diameter, defects, roughness, etc.)
- **Key insight**: Each routine type has completely different measurement fields
  - Offset: `offset`, `width`
  - DefectDetection: `length`, `width`, `area`, `volume`, `depth`
  - HoleDiameter: `diameter`, `circularity`
  - etc.

---

### **3. `Calib-*.yaml` (Optical Calibration)**
- **What it is**: Optical correction coefficients that convert raw pixels → real-world mm
- **Where**: Referenced by scan.yaml; stored alongside scan files
- **Key insight**: **Same calibration file is shared by multiple scans** (SCD Type 2 dimension)
- **Key fields**:
  - `mmperpixel` - Pixel-to-mm conversion constant
  - `linfit`, `quadfit` - Optical distortion correction matrices
  - `version`, `date` - Calibration version/timestamp

---

## **Data Organization Pattern**

```
Routine Folder (e.g., DefectDetection/)
├── scan.yaml                      ← Device context, image list, calib reference
├── G500-08b-repeat-001.tmd        ← 3D surface (binary, not human-readable)
└── analysis/
    └── scancontext.yaml           ← Shapes + measurements
```

---

## **Coordinate Systems (Critical!)**

| Format | Space | Origin | Units | Conversion |
|--------|-------|--------|-------|-----------|
| Shapes in YAML | Pixel space | Top-left (0,0) | Pixels | `pixel_value × mmperpixel = mm_value` |
| Final measurements | Real-world | Optical center | mm | Already converted |

**Rule**: Multiply pixel coordinates by `mmperpixel` from `scan.yaml` to get millimeters.

---

## **The Five Analysis Routines (So Far)**

| Routine | Type | Primary Output | Example |
|---------|------|---|---|
| **Offset** | Distance | `offset` (mm) | 0.21364525798 |
| **DefectDetection** | Defect classification | `length`, `width`, `area`, `volume`, `depth` | Scratch detection |
| **HoleDiameter** | Circle fitting | `diameter` (mm), `circularity` (mm) | 1.91052823765 mm |
| **PitDetection** | Surface pit detection | `depth`, `area`, `volume`, `point` | Similar to DefectDetection |
| **SurfaceRoughness** | Texture analysis | `Ra`?, `Rz`?, ... | (Not yet analyzed) |

---

## **Devices in the Sample Data**

| Device | Model | Firmware | Calibration | mmperpixel |
|--------|-------|----------|---|---|
| Modulus (00020015) | Modulus | 170 | Calib-4E07-XNJU_... | 0.00274 |
| Series 2 (CIMAF224) | Series 2 | 415 | Calib-436U-Y58K_... | 0.00682 |

→ **Different devices use different calibrations.**

---

## **Key Realization: Schema Variability**

**Problem**: Each routine produces completely different measurements.

**Implication for Data Modeling**:
- Cannot force all measurements into one rigid schema
- Need flexible fact table design OR routine-specific fact tables
- Data modeler must decide best approach

---

## **Files to Ingest to Bronze Layer**

✅ Do ingest:
- `scan.yaml` → Parse to JSON
- `analysis/scancontext.yaml` → Parse to JSON
- `Calib-*.yaml` → Parse to JSON (once per unique file)
- Image files (`.png`) → Store as blobs
- 3D files (`.tmd`, `.x3p`) → Store as blobs

❓ Ask GelSight about:
- `io_*/` folders → What are they?
- `scan.yaml.bak` → Why backups?

---

## **Things to Remember**

1. **Calibration is shared** → Multiple scans use same `Calib-*.yaml` file
2. **Coordinates need conversion** → Pixel coords × mmperpixel = real-world mm
3. **Versions matter** → Track which SDK/firmware/app/calibration combo produced each measurement
4. **Routines are independent** → Same scan can be analyzed by multiple routines (different folders)
5. **No standard measurement schema** → Each routine has unique outputs → need flexible design

---

**For More Detail**: See `DATA_INVENTORY.md` in this folder.
