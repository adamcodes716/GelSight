# GelSight Data Inventory & Schema Discovery

**Document Purpose**: Comprehensive catalog of all data artifacts, file formats, and YAML schemas in the GelSight analysis ecosystem.

**Created**: February 4, 2026  
**Status**: Phase 0 - Initial Discovery (Complete for provided samples)

---

## **EXECUTIVE SUMMARY**

GelSight's analysis workflow produces **three primary file categories**:

1. **Scan Metadata** (`scan.yaml`): Universal scan context, calibration refs, camera/device info
2. **Shape & Routine Data** (`analysis/scancontext.yaml`): Routine-specific shapes, measurements, profiles
3. **3D Surface Models** (`.tmd`, `.x3p`, `.stl`): 3D geometry files (not yet parsed)
4. **Calibration Files** (`Calib-*.yaml`): Optical calibration data (shared across multiple scans)
5. **Raw Images** (`.png`): Aligned surface images and alignment reference images

**Key Finding**: The same calibration file is referenced by multiple scans, making it a "slowly-changing dimension" in the data model.

---

## **1. SCAN METADATA FILE (`scan.yaml`)**

### **Purpose**
Captures device configuration, scan parameters, calibration reference, and image inventory for a single scan execution.

### **File Location**
Each analysis routine folder contains one:
- `GelSightAnalysis/DefectDetection/scan.yaml`
- `GelSightAnalysis/HoleDiameter/scan.yaml`
- `GelSightAnalysis/SurfaceRoughness/scan.yaml`
- etc.

### **Schema Overview**

| Field | Data Type | Example | Notes |
|-------|-----------|---------|-------|
| `version` | string | `"3.0"` | Scan file format version (may evolve) |
| `guid` | string (UUID) | `"bd691d23-5e89-4e38-96a1-ccf8fb41f6b9"` | Unique scan identifier |
| `createdon` | datetime | `"2025-09-17 12:58:41"` | When scan was executed |
| `mmperpixel` | float | `0.00274095190847` | **CRITICAL**: Optical calibration metric - pixel-to-mm conversion |
| `sdkversion` | string | `"4.2.240"` | Software version that created scan |
| `crop` | tuple (4 ints) | `(103, 103, 2266, 1858)` | Image region bounds (left, top, right, bottom) |
| `scanwidth` | int | `2472` | Image width in pixels |
| `scanheight` | int | `2064` | Image height in pixels |
| `images` | array of strings | `["aligned01.png", ..., "aligned06.png"]` | List of captured surface images |
| `activeheightmap` | string | `"G500-08b-repeat-001.tmd"` | 3D surface file (height map) |
| `activenormalmap` | string | `"G500-08b-repeat-001_nrm.png"` | Surface normal map (PNG) |
| `calib` | string (path) | `"C:\Users\Public\...\Calib-4E07-XNJU_20250917_1255.yaml"` | Path to calibration file |
| `aligned` | boolean | `true` | Whether images were aligned/registered |
| `alignmentimages` | array | `["image07.png"]` | Reference images used for alignment |

### **Nested Structures**

#### **metadata** (Operator & Device Identity)
```yaml
metadata:
  username: Kimo Johnson              # Operator name
  gelid: 4E07-XNJU                   # Gel/sensor ID (unique identifier)
  gelusecount: 17                    # How many times this sensor has been used
  geluseby: 2027-02-01 00:00:00      # Sensor expiration date
  gelproduct: MB101                  # Gel type/model
  appname: GelSight.Mobile           # Application name
  appversion: 4.2.240.0              # Full app version
  sdkversion: 4.2.240.0              # SDK version
```

#### **camera** (Optical Sensor)
```yaml
camera:
  cameraid: 00020015                 # Unique camera serial
  cameratype: Ximea                  # Manufacturer (Ximea is typical)
  shutter: 2.689                     # Exposure time (milliseconds)
  gelid: 4E07-XNJU                   # Links to gel above
  sensor_bit_depth: 8                # 8-bit (256 levels) or higher
  mmperpixel: 0.00274                # Calibration constant (redundant, also at root)
  lensfocuspos: 1735.00              # Lens focus position (hardware setting)
```

#### **device** (Physical GelSight Hardware)
```yaml
device:
  configuration: Modulus16mm90Deg    # Device model/configuration
  serialnumber: 00020015             # Hardware serial
  devicefirmware: 170                # Firmware version
  devicemodel: Modulus               # Model family
  devicetemp: 50.6                   # Temperature during scan (°C)
  lensserial: 4581A013               # Lens component serial
```

#### **calibration** (Who/When Calibration Was Set)
```yaml
calibration:
  username: Kimo Johnson             # Who performed calibration
  date: 2025-09-17 12:55:50          # When calibration was created
```

### **Critical Observations**

1. **`mmperpixel` appears twice**:
   - At root level: `0.00274095190847` (high precision)
   - In camera section: `0.00274` (rounded)
   - → Store full precision in Silver layer for accuracy

2. **Calibration file is external**:
   - Each scan references an external `Calib-*.yaml` file
   - Multiple scans can share the same calibration
   - Calibration files rarely change but may be updated
   - → Requires SCD Type 2 (slowly-changing dimension) tracking

3. **Version proliferation risk**:
   - `appversion`, `sdkversion`, `devicefirmware`, `calibration` versions all tracked separately
   - These can drift independently
   - → Track all versions; enable reprocessing with specific version combinations

4. **Two device types observed**:
   - `Modulus16mm90Deg` (most common in sample)
   - `Series2HalfX05` (seen in SurfaceRoughness scan)
   - → Schema must accommodate different device configurations

---

## **2. SHAPE & ROUTINE DATA FILE (`analysis/scancontext.yaml`)**

### **Purpose**
Contains the detected geometric shapes and routine-specific measurements for the scan.

### **File Location**
- `GelSightAnalysis/{Routine}/analysis/scancontext.yaml`

### **Two-Part Structure**

#### **PART A: SHAPES (Geometric Primitives)**

Detected geometry on the scanned surface. Types observed:

| Shape Type | Fields | Example |
|------------|--------|---------|
| **circle** | `id`, `type`, `x`, `y`, `r`, `snapmargin` | `x: 1619.94, y: 1419.31, r: 752.55` (center X,Y and radius) |
| **line** | `id`, `type`, `x1`, `y1`, `x2`, `y2`, `regions`, `snapmargin` | Two endpoints defining a line segment |
| **rectangle** | `id`, `type`, `x`, `y`, `w`, `h`, `rotation` | Position, width, height, rotation angle |
| **ruler** | `id`, `type`, `x1`, `y1`, `x2`, `y2` | Calibration ruler (like a line, but semantic meaning) |

**Key Fields**:
- `id`: Unique identifier within this scan (used as foreign key)
- `type`: Shape classification (circle, line, rectangle, ruler, region?, etc.)
- `snapmargin`: Tolerance for automatic shape adjustment (in mm)
- Coordinates are in **pixel space** (not mm), require `mmperpixel` from `scan.yaml` to convert

**Observed Shapes in Sample Data**:
- 10 shapes detected in Offset routine (3 circles, 3 rectangles, 4 lines + 1 ruler)
- 1 rectangle in DefectDetection routine
- 1 circle in HoleDiameter routine
- → Schema must be flexible for variable shape counts and types

#### **PART B: ROUTINES (Routine-Specific Measurements)**

Each routine processes shapes to produce measurements. **Highly heterogeneous schemas**.

### **ROUTINE SCHEMAS**

#### **1. OFFSET ROUTINE**

```yaml
routines:
- type: Offset
  id: 1262040199
  name: Offset
  annotationids: '[806847983]'              # Shape IDs used
  levelorder: 0
  levelregions: '[]'
  numprofiles: 3                            # Number of measurement profiles
  profilewidth: 0.2                         # Width of each profile (mm)
  reintegrate: false
  primaryshapeid: 1633818830                # Primary shape being measured (line)
  
  # RESULTS (mm):
  offsetregion1: (0, 0.245369880771, Min)  # Region-based measurement
  offsetregion2: (9.56942535007, 9.81479523084, Max)
  offset: 0.21364525798                    # Main measurement: offset distance
  width: 9.81479504604                     # Width measurement
  
  debug: false
  meta_PassedAnalysis: True
  meta_sdkversion: 4.2.240.0
```

**Measurement Fields**:
- `offset`: Primary result (distance, mm)
- `width`: Secondary measurement (mm)
- `offsetregion1/2`: Region-based measurements with mode (Min/Max)
- `numprofiles`: Metadata about how measurement was computed

---

#### **2. DEFECT DETECTION ROUTINE**

**Most complex schema in sample data**:

```yaml
routines:
- type: DefectDetection
  id: 881565834
  name: Defect Detection
  annotationids: '[]'
  scratchtype36: 2                          # Classification code
  autothreshold: true
  depththreshold: 0.005                     # 5 micrometers
  levelwidth: 0.35                          # Level detection width (mm)
  leveldiscardwidth: 0.2
  
  # Regions (elevation bands):
  levelregions: '[(0, 0.350841848806, None), (1.54041499241, 1.88851588928, None)]'
  
  primaryshapeid: 1285316935                # Rectangle being analyzed
  debug: false
  
  # RESULTS (measured defect properties):
  length: 1.10483882013 mm                  # Defect length
  width: 1.19671271864 mm                   # Defect width
  area: 0.77854575067 mm²                   # Defect area
  volume: 0.0218922324125 mm³               # Defect volume
  depth: 0.0937405147929 mm                 # Maximum depth
  point: (1327.9153795, 1087.67204659)     # Defect location (pixels)
  image: image.png                          # Associated image
  
  # DETAILED PROFILE DATA (list of ~360 points):
  profile: '[(0, 0.00306085375458), (0.0027...), ...]'  # X, Z cross-section
  
  # PLOT DATA (surface elevation map):
  plot: '[...]'                             # Detailed elevation measurements
  
  # GEOMETRY COMPUTED:
  profilelines: '[...]'                     # Bounding box coordinates
  minpt: (1.12379029696, -0.0937405147929) # Minimum point in plot
  
  meta_PassedAnalysis: True
  meta_sdkversion: 4.2.240.0
```

**Key Observations**:
- **Volumetric data**: `length`, `width`, `area`, `volume`, `depth`
- **Spatial location**: `point` (in pixels, must convert to mm)
- **Raw profiles**: Dense arrays of elevation measurements (360+ points)
- **Classification**: `scratchtype36` suggests a taxonomy of defect types

---

#### **3. HOLE DIAMETER ROUTINE**

```yaml
routines:
- type: HoleByEdge
  id: 1538058288
  name: Hole Diameter
  annotationids: '[]'
  edgesearch: 0.6                           # Edge detection radius (mm)
  regionmode: top                           # Search mode (top, bottom, etc.)
  nominaldiameter: 1.778                    # Expected diameter (mm)
  primaryshapeid: 2048951235                # Circle being fitted
  
  # RESULTS:
  diameter: 1.91052823765 mm                # Measured hole diameter
  circularity: 0.0304728653581 mm           # Circularity error (tolerance)
  circle: (1116.56578235, 927.845944351, 348.515461315)  # (X, Y, radius in pixels)
  minpt: (1387, 716)                        # Bounding box
  maxpt: (972, 604)
  
  meta_PassedAnalysis: True
  meta_sdkversion: 4.2.240.0
```

**Key Observations**:
- **Circle fitting**: Returns center (X,Y) and radius
- **Tolerance**: `circularity` measures how round the hole is
- **Nominal comparison**: Compares to `nominaldiameter`
- → Should add pass/fail logic: `diameter within tolerance of nominaldiameter`

---

#### **4. PIT DETECTION ROUTINE**

(Schema similar to DefectDetection—detects pits instead of scratches)

---

#### **5. SURFACE ROUGHNESS ROUTINE**

(Different measurements: Ra, Rz, etc.—not yet visible in sample scan.yaml, but likely in analysis/scancontext.yaml)

---

## **3. CALIBRATION FILES (`Calib-*.yaml`)**

### **Purpose**
Optical calibration data that converts raw pixel coordinates to real-world mm measurements. **Shared across multiple scans and rarely updated**.

### **File Example**

```yaml
version: 3.0
createdon: 2025-09-17 12:55:50
nL: 6                                       # Number of LED rings
width: 2472                                 # Image width (pixels)
height: 2064                                # Image height (pixels)
mmperpixel: 0.00274095190847               # CRITICAL: pixel-to-mm conversion
magnification: 0.999652708804              # Optical magnification factor

# Lighting correction (for each LED ring):
darkthresh: 0
lightthresh: 1
black: [0.234, 0.232, 0.232, 0.239, 0.250, 0.243]  # Dark level per LED
white: [0.852, 0.873, 0.864, 0.868, 0.875, 0.858]  # White level per LED

# Geometric calibration:
parameters:
    datafactor: 5000                        # Scaling for height data
    fov: 6.09765625                         # Field of view (degrees)
    modelsize: 2                            # ?
    radiusbias: 0.0337789321757            # Radius correction offset
    version: 2014

# Linear correction matrix (6 LED rings × 18 coefficients):
linfit:
    rows: 6
    cols: 18
    values: [-0.0140636658768, 0.133317116258, ...]

# Quadratic correction matrix (6 LED rings × 60 coefficients):
quadfit:
    rows: 6
    cols: 60
    values: [...]

# Device identification:
camera:
    cameraid: UCGAC2413002
    cameratype: Ximea
    gelid: 4E07-XNJU
    shutter: 2.689
    lensfocuspos: 1735.00
    sensor_bit_depth: 8

device:
    serialnumber: 00020015
    devicemodel: Modulus
    configuration: Modulus16mm90Deg
    devicefirmware: 170
    lensserial: 4581A013

# Reference materials/standards:
flatfield:
    modelfile: Calib-4E07-XNJU_20250917_1255.png
    modelsize: (618, 516)
    nL: 6

metadata:
    appname: GelSight.Mobile
    appversion: 4.2.240.0
    sdkversion: 4.2.240.0
    date: 2025-09-17 12:55:52

result:
    name: "GelSight CAL-FBG-250"             # Calibration standard name
    calibrationset: Calib_DB-0002-0015_...   # Unique calibration identifier
    circledistance_um: 4827                  # Reference circle distance (micrometers)
    groovedepth: 0.24518254506484377         # Reference groove depth
    groovewidth: 1.46407                     # Reference groove width
    isverified: True                         # QA flag
    nominaldepth: 0.246                      # Expected reference depth
    serialnumber: C1-001                     # Calibration standard serial

targetscans:                                 # UUIDs of scans using this calibration
  - 538dbdb9-eceb-4738-9d0d-1ebe2db42c2d
  - fb7b784b-4189-4161-a6d7-fcd50699c3f8
  - d6da8559-e46b-491f-aaa4-f2aa288b6cb9
```

**Key Observations**:

1. **Linear + Quadratic Corrections**: 
   - LED ring-specific calibration (6 rings per device)
   - Corrects for optical distortion (barrel/pincushion)
   - → Store both sets of coefficients

2. **Calibration Tracking**:
   - `version`, `date`, `appversion`, `sdkversion` all specified
   - `targetscans` lists which scans use it (important for provenance)
   - `isverified` indicates QA status
   - → Create a `dim_calibration` table with SCD Type 2 (track history)

3. **Two Calibration Files in Sample Data**:
   - `Calib-4E07-XNJU_20250917_1255.yaml` (used by Modulus device, most scans)
   - `Calib-436U-Y58K_20250912_0928.yaml` (used by Series2 device, SurfaceRoughness)
   - → Different devices, different calibrations

---

## **4. 3D SURFACE FILES (`.tmd`, `.x3p`, `.stl`)**

### **Observed in Sample Data**

| File | Routine | Device Type | Format |
|------|---------|-------------|--------|
| `G500-08b-repeat-001.tmd` | DefectDetection | Modulus | GelSight proprietary |
| `G500-08b-repeat-002.tmd` | PitDetection | Modulus | GelSight proprietary |
| `G500-08b-repeat-003.tmd` | HoleDiameter | Modulus | GelSight proprietary |
| `Scan-001.tmd` | Offset | Modulus | GelSight proprietary |
| `Scan-006.tmd` | SurfaceRoughness | Series2 | GelSight proprietary |
| `PitDetection.x3p` | PitDetection | Modulus | GelSight X3P format |

### **Format Information**

- **`.tmd`**: GelSight binary height map format (proprietary)
  - Contains 3D surface data (X, Y, Z coordinates)
  - Associated with `.png` normal maps for visualization
  - Requires proprietary parser to extract
  
- **`.x3p`**: ASTM ISO eXtensible 3D Point Cloud format
  - XML-based (human-readable structure)
  - Includes metadata: surface type, field of view, scan dimensions
  - Can contain multiple data layers
  - More standardized; easier to parse

- **`.stl`**: Stereolithography format (not observed in samples yet)
  - Standard CAD format
  - Contains triangulated surface mesh

### **Next Steps**
- [ ] Parse one `.tmd` file to understand binary structure
- [ ] Parse the `.x3p` file to understand layer structure
- [ ] Determine if `.tmd` and `.x3p` contain the same geometric data
- [ ] Establish conversion strategy (keep native? Convert to single format?)

---

## **5. RAW IMAGES (`.png`)**

### **Image Types Observed**

1. **Aligned Surface Images** (scan.yaml → `images` array):
   - `aligned01.png`, `aligned02.png`, ..., `aligned06.png`
   - 6 images per scan (typical for GelSight multi-view imaging)
   - Used for 3D reconstruction and texture capture

2. **Alignment Reference Image** (scan.yaml → `alignmentimages`):
   - `image07.png`
   - Reference used to register/align the 6 surface images

3. **Normal Map** (scan.yaml → `activenormalmap`):
   - `G500-08b-repeat-001_nrm.png`
   - Derived surface normal information (for visualization/analysis)

4. **Flatfield Model** (Calib-*.yaml → `flatfield.modelfile`):
   - `Calib-4E07-XNJU_20250917_1255.png`
   - Lighting/optical correction model (low resolution: 618×516)

5. **Defect Image Reference** (DefectDetection routine → `image`):
   - `image.png`
   - Specific image showing the detected defect

### **Storage Considerations**

- All images are **binary files** → store as blobs or Cloud Storage objects
- Images are **large** (2472×2064 pixels × 6 images = ~30 MB per scan)
- **Do not** embed in YAML; use file references
- → Store images in separate blob storage layer with metadata links

---

## **6. DISCOVERED SCHEMA VARIABILITY (Bronze → Silver Challenge)**

### **Routine-Specific Measurement Fields**

| Routine | Measurement 1 | Measurement 2 | Measurement 3 | Measurement 4 |
|---------|--------------|--------------|--------------|--------------|
| **Offset** | `offset` (mm) | `width` (mm) | `offsetregion1` | `offsetregion2` |
| **DefectDetection** | `length` (mm) | `width` (mm) | `area` (mm²) | `volume` (mm³), `depth` |
| **HoleDiameter** | `diameter` (mm) | `circularity` (mm) | `nominaldiameter` | — |
| **PitDetection** | `depth` (mm) | `area` (mm²) | `volume` (mm³) | `point` (location) |
| **SurfaceRoughness** | `Ra` (µm)? | `Rz` (µm)? | `Rq` (µm)? | (unknown—not in sample) |

**Challenge**: No two routines have identical output schemas.

**Modeling Solution** (for Silver layer):
```sql
-- Fact table (generic):
fact_measurements (
  measurement_id, scan_id, routine_type, shape_id, 
  measurement_name, measurement_value, measurement_unit,
  tolerance_min, tolerance_max, pass_fail_flag
)

-- OR: Routine-specific fact tables + shared dimension:
fact_offset_measurements (scan_id, shape_id, offset_mm, width_mm, ...)
fact_defect_measurements (scan_id, shape_id, length_mm, width_mm, area_mm2, volume_mm3, ...)
fact_hole_diameter_measurements (scan_id, shape_id, diameter_mm, circularity_mm, ...)

-- Data modeler will decide which approach fits better
```

---

## **7. SPATIAL COORDINATE SYSTEMS**

### **Three Coordinate Spaces**

| Space | Origin | Units | Used By | Example |
|-------|--------|-------|---------|---------|
| **Pixel** | Top-left (0, 0) | Pixels | Shapes in YAML | `x: 1619.94, y: 1419.31` |
| **Cropped Image** | Top-left of crop region | Pixels | Region selection | `crop: (103, 103, 2266, 1858)` |
| **Real-World (mm)** | Optical center of gel | Millimeters | Final measurements | `offset: 0.21364525798 mm` |

### **Conversion Chain**

```
Pixel Coordinates 
  ↓ (divide by mmperpixel)
Real-World Coordinates in mm
  ↓ (apply calibration linfit/quadfit corrections)
Calibrated Real-World Coordinates
```

**Implication**: Must store `mmperpixel` with every measurement to enable reprocessing with updated calibrations.

---

## **8. DATA QUALITY FLAGS**

### **Observed Flags**

- `meta_PassedAnalysis: True/False` 
  - Indicates whether the routine successfully completed
  - May indicate scan quality or algorithm convergence
  
- `isverified: True/False` (in calibrations)
  - QA flag for calibration quality

- `debug: True/False`
  - Whether scan was run in debug mode (may have reduced quality)

### **Implications**

- [ ] Define what "passed" means for each routine
- [ ] Create a `quality_checks` table to track:
  - Which scans failed analysis
  - Why they failed (error message? Timeout?)
  - Can they be reprocessed?

---

## **9. TEMPORAL VERSIONING**

### **Version Fields Across All Files**

| Component | Version Field | Current Value | Implication |
|-----------|---------------|---------------|-------------|
| Scan format | `version` | 3.0 | May evolve; handle backwards-compatibility |
| Calibration | `version` | 3.0 (calib), 2014 (parameters) | Multiple version streams |
| SDK | `sdkversion` | 4.2.240 | Algorithm behavior varies by version |
| Device firmware | `devicefirmware` | 170 (Modulus), 415 (Series2) | Hardware-specific behavior |
| App | `appversion` | 4.2.240.0 | App-level features |

**Modeling Implication**:
- Track version combinations as context dimensions
- Enable querying: "Show all scans run with SDK 4.2.240 on Modulus devices"
- Support reprocessing with specific version combos

---

## **10. DEVICE ECOSYSTEM**

### **Device Types Observed**

| Device Type | Model | Serial | Configuration | Firmware | Camera |
|-------------|-------|--------|---|---|---|
| Modulus | Modulus | 00020015 | Modulus16mm90Deg | 170 | UCGAC2413002 (Ximea) |
| Series 2 | Series 2 | CIMAF2243008 | Series2HalfX05 | 415 | CIMAF2243008 (Ximea) |

**Calibration Impact**:
- Each device type has its own calibration coefficients
- Different `mmperpixel` values (0.00274 vs 0.00682)
- → Device type is a key dimension in the calibration table

---

## **11. FOLDER STRUCTURE & ANALYSIS ROUTINE ORGANIZATION**

```
GelSightAnalysis/
│
├── DefectDetection/
│   ├── Calib-4E07-XNJU_20250917_1255.yaml        # Shared calibration
│   ├── G500-08b-repeat-001.tmd                   # 3D surface file
│   ├── scan.yaml                                 # Scan metadata
│   ├── scan.yaml.bak                             # Backup of scan.yaml
│   └── analysis/
│       ├── scancontext.yaml                      # Shapes + routine results
│       └── io_881565834/                         # Intermediate output (unknown)
│
├── HoleDiameter/
│   ├── Calib-4E07-XNJU_20250917_1255.yaml
│   ├── G500-08b-repeat-003.tmd
│   ├── scan.yaml
│   └── analysis/
│       └── scancontext.yaml
│
├── Offset/
│   ├── Calib-4E07-XNJU_20250917_1255.yaml
│   ├── Scan-001.tmd
│   ├── scan.yaml
│   └── analysis/
│       └── scancontext.yaml
│
├── PitDetection/
│   ├── Calib-4E07-XNJU_20250917_1255.yaml
│   ├── G500-08b-repeat-002.tmd
│   ├── PitDetection.x3p                         # Alternative 3D format
│   ├── scan.yaml
│   └── analysis/
│       ├── scancontext.yaml
│       └── io_1143073375/
│
├── SurfaceRoughness/
│   ├── Calib-436U-Y58K_20250912_0928.yaml       # Different calibration!
│   ├── Scan-006.tmd
│   ├── scan.yaml
│   └── analysis/
│       ├── scancontext.yaml
│       └── io_280719947/
│
└── Reports/                                      # (Empty in sample)

routines/
└── scancontext.yaml                              # Master/template schema?
```

### **Key Observations**

1. **Routine folders are isolated**: Each routine is independently runnable
2. **Calibration is shared**: Same calib file across multiple routines (except SurfaceRoughness)
3. **`io_*` folders are mysterious**: Appear inconsistently (3 of 5 routines have them)
4. **`scan.yaml.bak` exists**: Suggests overwrites during reprocessing

**Next Steps**:
- [ ] Understand what `io_*` folders contain
- [ ] Clarify the relationship between `routines/scancontext.yaml` and per-routine versions

---

## **12. QUESTIONS FOR GELSIGHT ENGINEERING**

### **Critical (Blocking)**
- [ ] What does the `io_*` folder contain? (Intermediate outputs? Debug data? Images?)
- [ ] What is the structure of `.tmd` binary files?
- [ ] How often are calibrations updated? What triggers recalibration?
- [ ] What other analysis routines exist beyond the 5 in our sample?
- [ ] Are there analysis routines we haven't seen in the samples?

### **Important (Schedule-Affecting)**
- [ ] How does Surface Roughness define its measurements (Ra, Rz, etc.)?
- [ ] What do `levelregions` and `scratchtype36` mean in DefectDetection?
- [ ] What is in the `routines/scancontext.yaml` file in the repo root?
- [ ] What happens if analysis fails? What error handling exists?
- [ ] How are nominal/tolerance values (e.g., `nominaldiameter`) provided?

### **Nice-to-Have (Optimization)**
- [ ] Can we get scans with different device types for testing?
- [ ] How large is a typical scan dataset (GB? TB?)
- [ ] How frequently are new scans added?
- [ ] Are there archived scans we should ingest?
- [ ] What ML models already exist for these routines?

---

## **13. BRONZE LAYER ARCHITECTURE**

### **Storage Strategy: Relational DB + Blob Storage**

**Relational Database** (SQL - scan DATA):
- Scan metadata (from scan.yaml)
- Calibration metadata (from Calib-*.yaml)
- Shapes detected (from scancontext.yaml)
- Measurements/results (from scancontext.yaml)
- Lineage (which scan used which calibration)

**Blob Storage** (Azure Data Lake Gen2 - raw files):
- Raw images (aligned01.png, etc.) - not indexed in DB
- 3D height maps (.tmd files) - not indexed in DB
- Original YAML files - preserved for audit trail
- io_*/ folders - preserved as-is

**Rationale**: Images and 3D files are large (30 MB per scan) and don't need relational indexing—they stay in blobs for scanning operations. The **scan DATA** (results, metadata, lineage) goes into relational DB for fast querying.

### **Bronze Layer Relational Tables**

```sql
-- Core Scan Metadata
bronze_scans (
  scan_id UUID PRIMARY KEY,
  guid UUID UNIQUE,
  created_date TIMESTAMP,
  routine_type VARCHAR(50),           -- DefectDetection, HoleDiameter, etc.
  mmperpixel DECIMAL(18, 11),         -- Optical conversion constant
  sdkversion VARCHAR(20),
  devicemodel VARCHAR(50),
  device_serialnumber VARCHAR(50),
  device_firmware VARCHAR(20),
  device_temperature DECIMAL(5, 1),
  operator_username VARCHAR(100),
  gel_id VARCHAR(50),
  gel_usecount INT,
  ingestion_timestamp TIMESTAMP,
  parse_success BOOLEAN,
  file_hash VARCHAR(64)               -- MD5/SHA256 of scancontext.yaml
);

-- Calibration Metadata (SCD Type 2 ready)
bronze_calibrations (
  calib_id UUID PRIMARY KEY,
  filename VARCHAR(200) UNIQUE,
  device_id VARCHAR(50),
  device_model VARCHAR(50),
  created_date TIMESTAMP,
  version VARCHAR(20),
  mmperpixel DECIMAL(18, 11),
  magnification DECIMAL(10, 9),
  is_verified BOOLEAN,
  effective_date TIMESTAMP,
  end_date TIMESTAMP,
  file_hash VARCHAR(64)
);

-- Link scans to calibrations (lineage)
bronze_scan_calib_lineage (
  scan_id UUID,
  calib_id UUID,
  PRIMARY KEY (scan_id, calib_id),
  FOREIGN KEY (scan_id) REFERENCES bronze_scans(scan_id),
  FOREIGN KEY (calib_id) REFERENCES bronze_calibrations(calib_id)
);

-- Geometric Shapes Detected
bronze_shapes (
  shape_id BIGINT,
  scan_id UUID,
  shape_type VARCHAR(50),             -- circle, line, rectangle, ruler
  x_pixels DECIMAL(12, 4),
  y_pixels DECIMAL(12, 4),
  radius_pixels DECIMAL(12, 4),       -- For circles
  x2_pixels DECIMAL(12, 4),           -- For lines
  y2_pixels DECIMAL(12, 4),
  width_pixels DECIMAL(12, 4),        -- For rectangles
  height_pixels DECIMAL(12, 4),
  rotation DECIMAL(5, 2),
  snapmargin_mm DECIMAL(5, 2),
  PRIMARY KEY (scan_id, shape_id),
  FOREIGN KEY (scan_id) REFERENCES bronze_scans(scan_id)
);

-- Measurements/Results (Normalized Structure)
bronze_measurements (
  measurement_id BIGINT,
  scan_id UUID,
  shape_id BIGINT,
  routine_type VARCHAR(50),
  measurement_name VARCHAR(100),      -- 'offset', 'diameter', 'length', etc.
  measurement_value DECIMAL(18, 10),
  unit VARCHAR(20),                   -- 'mm', 'mm2', 'mm3', etc.
  passed_analysis BOOLEAN,
  sdkversion_used VARCHAR(20),
  raw_json JSONB,                     -- Raw measurement object (for future fields)
  PRIMARY KEY (scan_id, measurement_id),
  FOREIGN KEY (scan_id) REFERENCES bronze_scans(scan_id),
  FOREIGN KEY (scan_id, shape_id) REFERENCES bronze_shapes(scan_id, shape_id)
);

-- Version Context (for reproducibility)
bronze_version_context (
  version_context_id UUID PRIMARY KEY,
  scan_id UUID UNIQUE,
  sdk_version VARCHAR(20),
  app_version VARCHAR(20),
  device_firmware VARCHAR(20),
  calib_version VARCHAR(20),
  created_date TIMESTAMP,
  FOREIGN KEY (scan_id) REFERENCES bronze_scans(scan_id)
);
```

### **Blob Storage Layout**

```
/gelsight-bronze/
├── images/
│   ├── {scan_guid}/
│   │   ├── aligned01.png
│   │   ├── aligned02.png
│   │   ├── ... (6 images total)
│   │   └── alignment_reference.png
│
├── heightmaps/
│   ├── {scan_guid}/
│   │   └── {activeheightmap}.tmd
│
├── raw_artifacts/
│   ├── {scan_guid}/
│   │   ├── scan.yaml
│   │   └── scancontext.yaml
│   ├── calibrations/
│   │   ├── Calib-4E07-XNJU_20250917_1255.yaml
│   │   └── Calib-436U-Y58K_20250912_0928.yaml
│
└── io_folders/
    └── {scan_guid}/
        └── io_*/ (as-is until clarified)
```

### **What to Ingest**

For each scan, ingest:

1. ✅ **scan.yaml** → Parse into `bronze_scans` table
2. ✅ **analysis/scancontext.yaml** → Parse into `bronze_shapes` + `bronze_measurements` tables
3. ✅ **Calib-*.yaml** → Parse into `bronze_calibrations` table (once per unique file)
4. ✅ **Raw images** → Store in blob storage, reference by scan_guid (not in DB)
5. ✅ **3D files** → Store in blob storage, reference by scan_guid (not in DB)
6. ✅ **raw YAML** → Preserve in blob storage for audit trail
7. ⏳ **io_*/ folder** → Store in blob storage pending clarification

### **Metadata Tracking (for Lineage)**

For each ingested item, record in the database:
- Ingestion timestamp
- Source file path (for lineage)
- File hash (MD5/SHA256 for reproducibility)
- Parse success/failure
- SDK/app/firmware versions used
- Which calibration version was active

---

## **14. BRONZE LAYER INGESTION SPECIFICATION**

### **Ingestion Process**

```
Source ZIP Drop (from GelSight)
├── GelSightAnalysis/
│   ├── DefectDetection/scan.yaml
│   ├── DefectDetection/analysis/scancontext.yaml
│   ├── DefectDetection/Calib-*.yaml
│   ├── DefectDetection/images/ (6 PNG files)
│   ├── DefectDetection/G500-08b-repeat-001.tmd
│   └── ... (other routines)

         ↓ INGESTION PIPELINE

1. Parse scan.yaml
   → Extract to bronze_scans table
   → Generate scan_id (if not provided)

2. Parse analysis/scancontext.yaml
   → Extract shapes to bronze_shapes table
   → Extract measurements to bronze_measurements table

3. Parse Calib-*.yaml
   → Check if calib_id exists in bronze_calibrations
   → If new, INSERT; if exists, SKIP (or SCD Type 2 UPDATE)
   → INSERT into bronze_scan_calib_lineage

4. Copy images/3D files to blob storage
   → Path: /gelsight-bronze/images/{scan_id}/
   → No DB entries (just file references)

5. Copy raw YAML to blob storage
   → Path: /gelsight-bronze/raw_artifacts/{scan_id}/

6. Compute file hashes
   → Store in bronze_scans.file_hash
   → Use for deduplication on re-ingest

7. Record version context
   → INSERT into bronze_version_context
   → For reproducibility tracking
```

### **Data Quality Checks**

- [ ] All required fields present in scan.yaml
- [ ] All referenced shapes exist
- [ ] All measurements have matching shapes
- [ ] File hashes don't duplicate (detect re-ingestion)
- [ ] Calibration file exists and is valid
- [ ] Parse errors logged and tracked

---

## **SUMMARY: WHAT TO TELL THE DATA MODELER**

**"Here's what you're working with:"**

1. **Multiple routine types** with heterogeneous schemas
   - Each routine has its own measurement vocabulary
   - No two routines measure the same things
   - But all routines share: shapes, device context, calibration

2. **Highly structured geometric data**
   - Shapes (circles, lines, rectangles, rulers) as dimensional entities
   - Routines as facts that attach to shapes
   - Measurements in pixel space (must convert to mm using `mmperpixel`)

3. **Calibration as SCD Type 2**
   - Same calibration shared by multiple scans
   - Calibrations are versioned and validated
   - Must track calibration lineage for reproducibility

4. **Variability in structure**
   - Device types vary → different calibrations
   - SDK versions vary → may need algorithm versioning
   - New routines will likely be added → design for extensibility

5. **Rich measurement data**
   - Some routines produce simple scalars (diameter)
   - Some produce volumetric data (area, volume)
   - Some produce dense profiles (360+ point arrays)
   - Some produce classifications (scratch type)

**"You need to decide:"**

- Star schema vs. snowflake vs. hybrid?
- Shared measurement table vs. routine-specific fact tables?
- How to normalize variable shape counts?
- How to store dense profile arrays (full storage vs. summary statistics)?
- When to normalize pixel coordinates to mm (Bronze vs. Silver)?

---

**Status**: Ready for data modeler review. Waiting on feedback from GelSight engineering (questions in section 12).
