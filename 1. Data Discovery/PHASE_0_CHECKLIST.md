# Phase 0 Discovery Checklist

**Purpose**: Concrete action items for completing Phase 0 (data inventory & understanding).

**Owner**: Technical Lead / Data Engineer  
**Timeline**: 4-6 weeks  
**Target Completion**: Mid-March 2026

---

## **BLOCK 1: DATA FILE PARSING & ANALYSIS**

### **Task 1.1: Extract All Sample YAML Schemas**

**Objective**: Collect representative YAML files for each routine type.

- [ ] DefectDetection: Already analyzed (`analysis/scancontext.yaml`, `scan.yaml`)
- [ ] HoleDiameter: Already analyzed
- [ ] Offset: Already analyzed
- [ ] PitDetection: Extract and analyze `analysis/scancontext.yaml` and `scan.yaml`
- [ ] SurfaceRoughness: Extract and analyze (different measurement types?)
- [ ] **Any other routines**: Check GelSight software to identify all routine types

**Deliverable**: Folder `/1. Data Discovery/sample_schemas/` with:
- One example of each routine's `analysis/scancontext.yaml`
- One example of each routine's `scan.yaml`
- One example of each unique calibration file

---

### **Task 1.2: Understand 3D File Formats**

**Objective**: Know what geometric data is in each file type.

- [ ] **Parse `.tmd` file**:
  - Is it binary? What's the format?
  - Does it contain the same 3D surface as the shapes in YAML?
  - Extract sample data points (height map)
  
- [ ] **Parse `.x3p` file**:
  - XML structure? Understand layer hierarchy
  - Does it contain the same data as `.tmd`?
  - How to extract point cloud coordinates
  
- [ ] **Research `.stl` format** (if present):
  - Standard triangulated mesh format
  - How many files use this?

**Deliverable**: Document (`3D_FILE_FORMATS.md`):
- Binary layout of `.tmd` (or link to public spec if exists)
- XML schema of `.x3p`
- Decision: Keep native formats or convert to standard (ASCII STL)?
- Recommendation on which format(s) to store in Bronze layer

---

### **Task 1.3: Decode Mysterious Fields**

**Objective**: Understand business meaning of cryptic fields.

- [ ] **DefectDetection**:
  - What is `scratchtype36`? (Is there a scratch classification taxonomy?)
  - What do `levelregions` represent? (Elevation bands?)
  - What is `depththreshold: 0.005` measuring in? (millimeters?)
  
- [ ] **Offset**:
  - Explain `offsetregion1`, `offsetregion2`
  - What does "offset" measure? (Distance between two shapes?)
  - What are `profilewidth`, `numprofiles`?
  
- [ ] **HoleDiameter**:
  - What does `edgesearch: 0.6` do?
  - What is `regionmode: top`?
  - How is `nominaldiameter` provided? (From product spec? User input?)
  
- [ ] **General**:
  - What does `snapmargin` do? (Tolerance for shape adjustment?)
  - What is `levelorder`, `leveldiscardwidth`?

**Deliverable**: Document (`FIELD_SEMANTICS.md`):
- Table of cryptic fields with business definitions
- Data type and units for each field
- How field values are used in analysis

---

## **BLOCK 2: CALIBRATION MAPPING & LINEAGE**

### **Task 2.1: Map All Calibration Files**

**Objective**: Understand the universe of calibrations.

- [ ] List all unique calibration files in the samples:
  - Filename
  - Device ID
  - Device model
  - Creation date
  - Version
  - Number of scans using it
  
- [ ] Check GelSight for more samples:
  - Are there other calibrations beyond the 2 in samples?
  - How often are calibrations created/updated?
  - What triggers recalibration?

**Deliverable**: Spreadsheet (`calibration_inventory.xlsx`):
- One row per unique calibration
- Columns: filename, device_id, device_model, created_date, version, scan_count
- Identify which scans use which calibration

---

### **Task 2.2: Understand Calibration Change Process**

**Objective**: Know when/why calibrations are replaced.

- [ ] Interview GelSight on:
  - How long does one calibration stay valid?
  - What events trigger recalibration? (Device moved? Hardware drift? Scheduled?)
  - How are old calibrations archived?
  - Can scans be reprocessed with old calibration?
  
- [ ] Document the versioning scheme:
  - How are calibration versions named? (e.g., `Calib-4E07-XNJU_20250917_1255.yaml`)
  - Does timestamp in filename indicate creation time?
  - Is there a version number field inside the file?

**Deliverable**: Document (`calibration_lifecycle.md`):
- Calibration creation frequency
- Typical calibration lifespan
- Versioning scheme explanation
- Reprocessing workflow (if scans can be reprocessed)

---

## **BLOCK 3: ROUTINE COMPLETENESS & TAXONOMY**

### **Task 3.1: Enumerate All Analysis Routines**

**Objective**: Build complete catalog of routine types.

- [ ] In GelSight software, find:
  - How many analysis routines exist?
  - What are their names?
  - Which are commonly used? Which are rare?
  - Are there new routines in development?
  
- [ ] For each routine:
  - What does it measure?
  - What shapes can it analyze? (circle, line, rectangle, region?)
  - What are its output fields? (Collect schema)

**Deliverable**: Spreadsheet (`routine_catalog.xlsx`):
- One row per routine type
- Columns: name, description, measured_shapes, output_fields, commonality (common/uncommon/rare)

---

### **Task 3.2: Collect Schema Examples**

**Objective**: One example of each routine output.

For each routine in catalog:
- [ ] Request sample `analysis/scancontext.yaml` from GelSight
- [ ] Request sample `scan.yaml` from same scan
- [ ] Store in `/1. Data Discovery/sample_schemas/{routine_name}/`

**Deliverable**: Complete schema library in `/sample_schemas/`

---

## **BLOCK 4: DATA QUALITY & FAILURE SCENARIOS**

### **Task 4.1: Understand Analysis Failures**

**Objective**: Know when/why scans fail.

- [ ] Ask GelSight:
  - What percentage of scans produce `meta_PassedAnalysis: False`?
  - What causes failures? (Low contrast? Geometry out of spec? Timeout?)
  - Can failed scans be reprocessed?
  - Should failed scans be ingested to Bronze layer?
  
- [ ] Search sample data for failures:
  - Are any scans marked `meta_PassedAnalysis: False`?
  - What patterns indicate low-quality scans?

**Deliverable**: Document (`failure_handling.md`):
- Common failure modes
- Failure frequency/severity
- Recommendation: Ingest failed scans? Mark separately?

---

### **Task 4.2: Data Quality Baseline**

**Objective**: Establish metrics for data quality.

For each file type and routine:
- [ ] Count: Total files, total records
- [ ] Nulls: Any missing fields? Optional fields?
- [ ] Anomalies: 
  - Outlier values (measurements far from expected ranges)?
  - Date anomalies (future dates? Very old dates)?
  - Version mismatches (inconsistent version combinations)?
  
- [ ] Schema variance:
  - Do all routines of same type have identical field sets?
  - Are there optional fields?
  - Do field names vary spelling/casing?

**Deliverable**: Report (`data_quality_baseline.md`):
- Completeness: % of non-null values per field
- Freshness: Date range of data
- Integrity: Constraint violations
- Consistency: Schema variance identified
- Recommendations for validation rules in Bronze layer

---

## **BLOCK 5: DEVICE ECOSYSTEM MAPPING**

### **Task 5.1: Device Inventory**

**Objective**: Understand all device types.

- [ ] From samples, document:
  - Device models present (Modulus, Series 2, others?)
  - Serial numbers
  - Firmware versions
  - Associated calibrations
  
- [ ] From GelSight, ask:
  - How many devices are in the field?
  - What is the current device inventory?
  - Are new device models planned?
  - Which devices are active vs. retired?

**Deliverable**: Spreadsheet (`device_inventory.xlsx`):
- One row per device
- Columns: model, serial, firmware, calibration_id, last_scan_date, status

---

### **Task 5.2: Version Matrix**

**Objective**: Map version combinations.

- [ ] Build matrix:
  - Rows: Device models
  - Columns: Firmware versions
  - Cells: Calibration ID, SDK version used, date range
  
- [ ] Identify:
  - Which version combos are currently active?
  - Are there deprecated combos?
  - How often do versions change?

**Deliverable**: Spreadsheet (`version_matrix.xlsx`) + Document:
- Version compatibility table
- Versioning strategy explanation
- Impact on data modeling (do we need to track all combinations?)

---

## **BLOCK 6: FOLDER STRUCTURE & FILE NAMING**

### **Task 6.1: Folder Organization Rules**

**Objective**: Understand naming/structure conventions.

- [ ] Document observed patterns:
  - Folder naming: Why `DefectDetection` vs `HoleDiameter`? (routine name, title case?)
  - File naming: `Calib-4E07-XNJU_20250917_1255.yaml` → How is this constructed?
  - Image naming: `aligned01.png` → Why 01-06? Significance?
  - 3D file naming: `G500-08b-repeat-001.tmd` → What do parts mean?
  
- [ ] Ask GelSight:
  - Are there naming conventions I should follow?
  - Will naming change in future versions?
  - Is there a scan ID I should use instead of folder names?

**Deliverable**: Document (`naming_conventions.md`):
- Folder naming scheme
- File naming scheme
- Transformation rules (if renaming needed for Bronze layer)

---

### **Task 6.2: Understand `io_*/` Folders**

**Objective**: Mysterious folders in some routines.

- [ ] Observed pattern:
  - DefectDetection: `io_881565834/` (routine ID?)
  - PitDetection: `io_1143073375/`
  - SurfaceRoughness: `io_280719947/`
  - HoleDiameter: No `io_*` folder
  - Offset: No `io_*` folder
  
- [ ] Action:
  - List contents of each `io_*/` folder
  - Ask GelSight: What are these? (Intermediate files? Debug output? Images?)
  - Should we ingest them to Bronze layer?

**Deliverable**: Document (`mysterious_io_folders.md`):
- Content analysis of each `io_*/` folder
- Purpose/meaning explanation from GelSight
- Recommendation: ingest or skip?

---

## **BLOCK 7: DATA VOLUME & INGESTION PLANNING**

### **Task 7.1: Size Estimation**

**Objective**: Forecast storage and processing needs.

- [ ] Measure sample data:
  - Total size of all files in samples (with images)
  - Breakdown: YAML (~1 KB), 3D files (~1 MB), images (×6, ~30 MB per scan)
  - Approximate records: # of scans, # of shapes per scan, # of measurements
  
- [ ] From GelSight, ask:
  - How many new scans per day? Per week? Per month?
  - Expected data volume over next year?
  - Archive requirements (keep all data forever?)

**Deliverable**: Spreadsheet (`size_estimation.xlsx`):
- Per-scan size breakdown
- Annual volume projection
- Storage cost estimate (Azure)
- Processing time estimates

---

### **Task 7.2: Ingestion Frequency & Mechanism**

**Objective**: How will data arrive at the platform?

- [ ] Ask GelSight:
  - How are scans exported from the device?
  - Manual ZIP drops or automated API?
  - Frequency: Real-time streaming? Daily batches? On-demand?
  - File format for delivery (ZIP? Cloud sync?)
  
- [ ] Document:
  - Data ingestion SLA (how fast must we ingest?)
  - Error handling (what if files are corrupted?)
  - Deduplication (how do we know if a scan was already ingested?)

**Deliverable**: Document (`ingestion_mechanism.md`):
- Data delivery method
- Frequency & SLA
- Deduplication strategy
- Error handling approach

---

## **BLOCK 8: STAKEHOLDER ALIGNMENT**

### **Task 8.1: Kickoff with GelSight Engineering**

**Objective**: Answer all remaining questions.

**Meeting Agenda**:
- [ ] Present `DATA_INVENTORY.md` findings
- [ ] Walk through all "Questions for GelSight Engineering" (section 12 of DATA_INVENTORY)
- [ ] Discuss routine taxonomy (are there more than 5?)
- [ ] Discuss device ecosystem
- [ ] Discuss version strategy
- [ ] Discuss data delivery mechanism
- [ ] Confirm data quality expectations

**Deliverable**: Meeting notes with answers to all questions

---

### **Task 8.2: Assemble Data Modeling Team**

**Objective**: Prepare to hand off to data modeler.

- [ ] Identify data modeler
- [ ] Share:
  - `IMPLEMENTATION_SCAFFOLD.md` (project plan)
  - `DATA_INVENTORY.md` (detailed schema analysis)
  - `QUICK_REFERENCE.md` (cheat sheet)
  - `/sample_schemas/` (example YAML files)
  
- [ ] Schedule kickoff meeting:
  - Review data structure
  - Identify modeling decisions (section 12 of DATA_INVENTORY)
  - Set Phase 2 timeline

**Deliverable**: Data modeler onboarded and ready for Phase 2

---

## **COMPLETION CRITERIA**

Phase 0 is **complete** when:

- [x] All YAML file structures documented
- [x] All file types (3D, images, calibration) cataloged
- [x] All routine types enumerated
- [ ] All mysterious fields explained (Task 1.3)
- [ ] Calibration lineage understood (Task 2.2)
- [ ] Data quality baseline established (Task 4.2)
- [ ] Device ecosystem documented (Task 5.1)
- [ ] File naming/structure conventions documented (Task 6.1)
- [ ] Volume projections completed (Task 7.1)
- [ ] GelSight engineering alignment achieved (Task 8.1)
- [ ] Data modeler onboarded (Task 8.2)

**Status**: 1/11 complete (DATA_INVENTORY written; 10 blocks of tasks remaining)

---

## **NEXT STEP**

Schedule a kickoff meeting with GelSight engineering to answer the questions in **Task 8.1**. Most blocks can proceed in parallel after the meeting.

