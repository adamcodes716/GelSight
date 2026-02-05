# Phase 0 Summary: What We've Learned About GelSight Data

**Date**: February 4, 2026  
**Status**: Initial discovery complete; ready for deeper Phase 0 work

---

## **WHAT WE HAVE (In the Samples)**

### **File Inventory**

| File Type | Count | Purpose | Complexity |
|-----------|-------|---------|-----------|
| `scan.yaml` | 5 | Scan metadata, device context, calibration ref | ⭐ Moderate |
| `analysis/scancontext.yaml` | 5 | Shapes + routine measurements | ⭐⭐⭐ High |
| `Calib-*.yaml` | 2 unique | Optical calibration coefficients | ⭐ Moderate |
| `.tmd` files | 4 | 3D height maps (binary) | ⭐⭐⭐ Unclear |
| `.x3p` file | 1 | 3D point cloud (XML) | ⭐⭐ Moderate |
| `.png` images | ~40 | Aligned surface images + normal maps | ⭐ Simple storage |
| `io_*/` folders | 3 | Unknown intermediate output | ⭐⭐⭐ Unclear |

### **Routines Analyzed**

1. **Offset**: Measures distance between shapes → `offset`, `width` (mm)
2. **DefectDetection**: Finds surface defects → `length`, `width`, `area`, `volume`, `depth`
3. **HoleDiameter**: Fits circles to holes → `diameter`, `circularity` (mm)
4. **PitDetection**: Detects surface pits (similar to DefectDetection)
5. **SurfaceRoughness**: Texture analysis (schema not yet analyzed)

### **Devices Observed**

- **Modulus**: 00020015 (Firmware 170) → `mmperpixel: 0.00274`
- **Series 2**: CIMAF2243008 (Firmware 415) → `mmperpixel: 0.00682`

---

## **THE BIG INSIGHTS**

### **Insight 1: Schema Heterogeneity is the Central Challenge**

Each routine outputs completely different measurements. There's no "one true measurement schema."

| Routine | Output 1 | Output 2 | Output 3 | Output 4 |
|---------|----------|----------|----------|----------|
| Offset | offset (mm) | width (mm) | regionoffset1 | regionoffset2 |
| DefectDetection | length (mm) | width (mm) | area (mm²) | volume (mm³) + depth (mm) |
| HoleDiameter | diameter (mm) | circularity (mm) | — | — |
| PitDetection | depth (mm) | area (mm²) | volume (mm³) | location |

**Implication**: Data modeler can't use a rigid fact table. Must choose between:
- Option A: One generic fact table with `measurement_name`, `measurement_value`, `unit`
- Option B: Separate fact tables per routine type
- Option C: Hybrid approach

---

### **Insight 2: Calibration is a Shared, Versioned Resource**

The same calibration file is used by multiple scans:
- `Calib-4E07-XNJU_20250917_1255.yaml` used by: DefectDetection, HoleDiameter, Offset, PitDetection (4 scans)
- `Calib-436U-Y58K_20250912_0928.yaml` used by: SurfaceRoughness (1 scan)

**Implication**: Calibration must be a **slowly-changing dimension** (SCD Type 2) in the Silver layer:
- Track calibration version history
- Know which scan used which calibration version
- Enable reprocessing with updated/different calibrations

---

### **Insight 3: Spatial Coordinates Need Careful Handling**

All shapes are in **pixel space**, but measurements are in **mm space**.

Conversion requires:
- `mmperpixel` from `scan.yaml` (simple scaling)
- `linfit` + `quadfit` matrices from `Calib-*.yaml` (optical distortion correction)

**Implication**: Must store both:
- Raw pixel coordinates (from YAML) in Bronze
- Converted mm coordinates (computed in Silver)
- `mmperpixel` value used for conversion (for reproducibility)

---

### **Insight 4: Version Tracking is Critical**

At least 4 independent version streams:
- SDK version: `4.2.240`
- App version: `4.2.240.0`
- Device firmware: `170` (Modulus) or `415` (Series2)
- Calibration version: `3.0` (file format) + internal `2014` (parameters)

**Implication**: Build a dimension table for `analysis_version_context`:
- Which scan used which combination of versions?
- Can query: "Show all scans processed with SDK 4.2.240 on Modulus"
- Enable reproducibility if algorithm changes

---

### **Insight 5: The `io_*/` Folders Are a Mystery**

Observed in 3 of 5 routine folders:
- DefectDetection: `io_881565834/`
- PitDetection: `io_1143073375/`
- SurfaceRoughness: `io_280719947/`

Not in: HoleDiameter, Offset

**Implication**: Must ask GelSight before designing Bronze layer ingestion.

---

## **THE BIG QUESTIONS (Must Ask GelSight)**

### **Blocking Phase 1 (Infrastructure)**
1. What are the `io_*/` folders? (Intermediate outputs? Debug data? Images?)
2. How will scans be delivered to the platform? (ZIP drops? API? Cloud sync?)
3. How frequently are new scans created? (Per day? Per week?)
4. Are calibrations versioned externally or only by timestamp?

### **Blocking Phase 2 (Data Modeling)**
5. Are there analysis routines beyond the 5 we've seen?
6. Can you provide one sample of each routine type?
7. What do cryptic fields mean? (`scratchtype36`, `levelregions`, `snapmargin`, etc.)
8. How is `nominaldiameter` provided to HoleDiameter analysis? (From user? Product spec? Hardcoded?)

### **Planning Phase 3 & 4**
9. What are the tolerance ranges for each measurement? (e.g., hole diameter ±0.1mm?)
10. How will pass/fail be determined? (Tolerance check? ML classification? Operator input?)
11. What ML models already exist? (Defect detection? Roughness prediction?)
12. What historical data can we ingest to train new models?

---

## **WHAT'S READY FOR DATA MODELER**

Your data modeler now has:

1. ✅ **Complete schema documentation** (`DATA_INVENTORY.md`)
   - Every field in every file type
   - Data types, examples, business meaning
   - Coordinate system explanations
   - Schema variability analysis

2. ✅ **Sample files** (in `/sample_schemas/`)
   - Real YAML from each routine type
   - Can examine actual data structure

3. ✅ **Modeling decisions** (listed in DATA_INVENTORY section 12)
   - Star schema vs. Snowflake
   - Shared vs. routine-specific fact tables
   - Measurement normalization strategy
   - Coordinate transformation timing

4. ✅ **Cheat sheet** (`QUICK_REFERENCE.md`)
   - One-page summary for new team members

5. ✅ **Tactical checklist** (`PHASE_0_CHECKLIST.md`)
   - Concrete next steps for completing Phase 0

---

## **TIMELINE TO DATA MODELER HANDOFF**

| Task | Timeline | Owner | Status |
|------|----------|-------|--------|
| GelSight engineering kickoff | Week 1-2 | Tech Lead | 🟡 Scheduled? |
| Answer blocking questions | Week 2-3 | GelSight | ⏳ Pending |
| Complete Phase 0 checklist | Week 3-4 | Data Engineer | ⏳ Starting |
| Data modeler onboarding | Week 4-5 | Tech Lead | ⏳ Waiting |
| **Phase 2 design kicks off** | **Week 5-6** | **Data Modeler** | ⏳ Ready |

---

## **IMMEDIATE ACTION ITEMS (This Week)**

1. **Schedule GelSight engineering kickoff**
   - Share this document
   - Walk through "Big Questions" section
   - Get answers to blocking questions

2. **Assemble the data modeler**
   - Share this summary
   - Share DATA_INVENTORY.md
   - Schedule Week 5 kickoff meeting

3. **Start Phase 0 Checklist tasks** (in parallel):
   - Task 1.3: Decode cryptic fields (document questions for GelSight)
   - Task 6.2: Understand `io_*/` folders (add to GelSight questions)

---

## **KEY DECISION POINTS FOR DATA MODELER**

When they join, they'll need to decide on:

### **Fact Table Design**
```
Option A: One unified fact table (highest flexibility)
─────────────────────────────────────────────
fact_measurements
  measurement_id, scan_id, routine_type, shape_id, 
  measurement_name, measurement_value, unit, 
  tolerance_min, tolerance_max, pass_fail

Option B: Separate fact tables per routine (strictest structure)
─────────────────────────────────────────────
fact_offset_measurements (scan_id, shape_id, offset_mm, width_mm, ...)
fact_defect_measurements (scan_id, shape_id, length_mm, width_mm, area_mm2, ...)
fact_hole_diameter_measurements (scan_id, shape_id, diameter_mm, circularity_mm, ...)
... etc for each routine

Option C: Hybrid (best of both?)
─────────────────────────────────────────────
shared_measurements + routine-specific tables
```

### **Coordinate Transformation**
```
Option A: Transform pixel → mm in Bronze layer (simple, but early commitment)
Option B: Transform in Silver layer (delays conversion, enables audit trail)
Option C: Store both (takes space, enables reproducibility)
```

### **Profile Data Storage**
```
Some routines (DefectDetection) have dense profile arrays with 360+ points.

Option A: Store raw arrays as JSON/text
Option B: Compute summary stats (min, max, mean, std) and discard raw
Option C: Store in separate timeseries table
```

---

## **SUMMARY: YOU NOW UNDERSTAND**

✅ The complete file structure  
✅ The heterogeneous schemas  
✅ The device ecosystem  
✅ The calibration strategy  
✅ The version tracking needs  
✅ The coordinate transformation requirements  
✅ The key questions to ask GelSight  
✅ The modeling decisions ahead  

**You're ready to hand off to a data modeler.** They'll take sections 12 of DATA_INVENTORY and turn them into a silver-layer schema. You'll implement it in Phase 1-2.

---

## **FILES IN THIS FOLDER**

- **IMPLEMENTATION_SCAFFOLD.md** (in repo root) - Overall project plan
- **DATA_INVENTORY.md** - Everything about the data (this is the bible)
- **QUICK_REFERENCE.md** - One-pager for new team members
- **PHASE_0_CHECKLIST.md** - Tactical action items
- **sample_schemas/** - Real YAML files (examples of each routine type)

---

**Next Document to Read**: Talk to GelSight engineering, get answers to the "Big Questions" section above, then complete PHASE_0_CHECKLIST tasks.

