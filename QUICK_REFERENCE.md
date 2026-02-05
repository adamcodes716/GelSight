# GelSight Data Modeling: Quick Reference for Data Modeler

**Document Purpose:** 15-minute orientation for data modeler joining Phase 2 (Silver layer design)  
**Date:** February 5, 2026  
**Status:** Phase 0 Complete; Ready for Silver Design

---

## 1. The Business Context

**GelSight Hardware:** Optical surface scanning sensors that produce 3D images of surfaces  
- **Modulus device** (firmware 170) - used in DefectDetection, HoleDiameter, PitDetection, Offset routines
- **Series2 device** (firmware 415) - used in SurfaceRoughness routine
- Each device has unique calibration coefficients (optical distortion corrections)

**Analysis Routines:** Five known routines run on scans, each producing different measurements

---

## 2. Data Flow (Bronze → Silver → Gold)

```
Physical Scan
    ↓
[BRONZE] → Raw YAML files stored as JSONB
    ├─ scan.yaml (device metadata, image list)
    ├─ analysis/scancontext.yaml (routine results, measurements, profiles)
    └─ Calib-*.yaml (optical calibration coefficients)
    ↓
[SILVER] → Normalized relational schema (YOUR DESIGN)
    ├─ Scans (one per physical scan event)
    ├─ Devices (Modulus vs Series2 reference data)
    ├─ Calibrations (versioned optical corrections - SCD Type 2)
    ├─ Measurements (5 routines, heterogeneous measurement types)
    └─ Shapes (detected geometry: circles, lines, rectangles)
    ↓
[GOLD] → Analytics layer
    ├─ Aggregations
    ├─ Trends
    └─ ML features
```

---

## 3. The Five Analysis Routines & Their Measurements

| Routine | Measurements | Example Values | Notes |
|---------|--------------|-----------------|-------|
| **Offset** | offset, width | 1.5 mm, 2.3 mm | Two scalar values |
| **DefectDetection** | length, width, area, volume, depth | 1.10 mm, 1.19 mm, 0.77 mm², 0.021 mm³, 0.093 mm | Five scalars + 360-point profile array |
| **HoleDiameter** | diameter, circularity | 2.3 mm, 0.95 ratio | Two scalars + nominal value |
| **PitDetection** | Similar to DefectDetection | Same structure | Five scalars + 360-point profile |
| **SurfaceRoughness** | Ra, Rz, peak_count | 0.8 µm, 2.1 µm, 47 points | Different device (Series2), different units |

**Key insight:** Schema heterogeneity. No two routines have identical measurement sets.

---

## 4. File Structure Examples

### scan.yaml (Metadata - Same for all routines)
```yaml
image:
  camera_type: gelsight_modulus
  firmware: 170
  timestamp: 2025-09-17T12:55:00Z
calibration_reference: Calib-4E07-XNJU_20250917_1255.yaml
image_files:
  - image.png
  - image2.png
```

### analysis/scancontext.yaml (Routine Output - Varies by routine)
```yaml
version: 3.7
routines:
  - type: DefectDetection
    id: 881565834
    length: 1.10483882013 mm
    width: 1.19671271864 mm
    area: 0.77854575067 mm2
    volume: 0.0218922324125 mm3
    depth: 0.0937405147929 mm
    profile: [(0, 0.003), (0.003, 0.002), ... 360 points ...]
    point: (1327.91, 1087.67)  # Pixel coordinates
    meta_PassedAnalysis: true
```

### Calibration YAML (Device-specific, versioned)
```yaml
device_id: 4E07-XNJU
device_type: Modulus
firmware: 170
timestamp: 2025-09-17T12:55:00Z
calibration_coefficients:
  x_pixel_to_mm: 0.00432
  y_pixel_to_mm: 0.00432
  optical_distortion: [matrix_data...]
  version: 3
```

---

## 5. Critical Design Decisions (You'll Make These in Phase 2)

### Decision 1: Measurement Storage
**Question:** How do you store measurements given routine heterogeneity?

**Option A - Rigid Schema** (one table per routine)
- `defect_detection_measurements (scan_id, length, width, area, volume, depth)`
- `hole_diameter_measurements (scan_id, diameter, circularity, nominal_diameter)`
- *Problem:* New routine = new table, requires schema migration

**Option B - Flexible Schema** (recommended for future-proofing)
- `measurements (scan_id, routine_type, measurement_name, measurement_value, unit)`
- DefectDetection produces 5 rows, HoleDiameter produces 2 rows, future routine produces N rows
- *Benefit:* New routine = new rows, zero schema changes

### Decision 2: Calibration Versioning
**Question:** Calibrations change over time. How do you track which calibration was used for which scan?

**Approach:** Slowly-Changing Dimension (SCD Type 2)
- `calibrations (calib_id, device_id, firmware, version, created_at, valid_from, valid_to)`
- `scan_calibration_lineage (scan_id, calib_id)` - tracks which calibration was active
- *Benefit:* Reproducible analysis; can re-calculate using historical calibrations

### Decision 3: Coordinate Systems
**Question:** Raw measurements are in pixel space (from camera image). How do you handle?

**Approach:** Store raw in Silver, transform to real-world mm during analysis
- Bronze layer: Preserve pixel-space `point: (1327.91, 1087.67)` as-is
- Silver layer: Create `measurements_mm` alongside pixel measurements
- Transformation uses calibration coefficients: `mm = pixels × calibration_coeff`
- *Benefit:* Audit trail; can recompute if calibration updated

### Decision 4: Shapes & Geometry
**Question:** scancontext.yaml contains `shapes` (detected rectangles/circles). How do you model?

**Approach:** Separate dimension table
- `shapes (shape_id, scan_id, shape_type, x, y, width, height, ...)`
- Link measurements to shapes if needed
- *Benefit:* Flexible for future geometry types

### Decision 5: Profiles & Complex Arrays
**Question:** 360-point profile array. Store normalized or keep as JSON?

**Options:**
- A: Normalize to 360 rows in a `profile_points` table
- B: Keep as array in JSON column in measurements table
- C: Reference external file (future if scale demands)
- *Recommendation:* Start with B (simplicity); migrate to A if query patterns demand it

### Decision 6: Measurement Validation
**Question:** meta_PassedAnalysis field indicates if routine passed quality checks. How do you model?

**Approach:** Add to measurements table
- `measurements (scan_id, routine_type, measurement_name, measurement_value, unit, passed_analysis, quality_flags)`
- *Benefit:* Can filter Gold layer to only "trusted" measurements

---

## 6. Schema Variability by Device

**Modulus devices (firmware 170):**
- DefectDetection, HoleDiameter, PitDetection, Offset
- All use same calibration file format
- Measurements in mm

**Series2 device (firmware 415):**
- SurfaceRoughness routine
- Different calibration file format (observed in sample data)
- Measurements in µm (micrometers)

**Implication:** Normalize units in Silver layer (all mm? all base units?). Decide at design time.

---

## 7. Known Data Characteristics

| Characteristic | Details |
|---|---|
| **Current Scale** | ~10 scans (sample data) |
| **Projected Scale** | 100-1000s of scans (Phase 0 will clarify) |
| **Raw File Size** | 50-500 KB per scan YAML |
| **Bronze Storage** | JSONB in relational DB (no blob storage needed currently) |
| **Routine Consistency** | Variations exist (e.g., SurfaceRoughness different from others) |
| **Future Unknowns** | New routines/devices will arrive; schema must accommodate |

---

## 8. Phase 0 → Phase 2 Transition

**What's Done (Phase 0):**
- ✅ Data inventory complete
- ✅ All routines analyzed
- ✅ Bronze layer architecture finalized (JSONB storage)
- ✅ Configuration-driven ETL approach defined (routine_schemas.yaml)

**What You Design (Phase 2 - Silver Layer):**
- Schema for normalized measurements
- Calibration dimension tables
- Coordinate transformation rules
- Data validation/quality gates
- Aggregate fact tables for Gold

**What's Pending:**
- GelSight engineering answers (routine catalog, data delivery mechanism, device ecosystem confirmation)
- Data engineer to build Phase 0 checklist tasks (calibration inventory, version tracking setup)

---

## 9. Configuration-Driven ETL Pattern

Your Silver layer will use **routine_schemas.yaml** to guide transformation (no hardcoded parsing):

```yaml
DefectDetection:
  measurements:
    - source_path: "length"
      measurement_name: "length"
      unit: "mm"
    - source_path: "width"
      measurement_name: "width"
      unit: "mm"
    - source_path: "area"
      measurement_name: "area"
      unit: "mm2"

FutureRoutine:
  measurements:
    - source_path: "analysis.results.computed.value"  # Any nesting depth
      measurement_name: "new_metric"
      unit: "unit_here"
```

**Benefit:** New routine with weird nested structure → Just add schema definition. No Python code changes.

---

## 10. Quick Decision Checklist

Before your first design session, clarify:

1. ☐ Will you use flexible schema (key-value measurements) or rigid schema (one table per routine)?
2. ☐ How many years of historical calibration data must you track?
3. ☐ Should pixel-space measurements be in Silver, or only real-world mm?
4. ☐ Will users query individual profile points, or just aggregates?
5. ☐ Do you normalize all units to one standard (e.g., all mm), or preserve routine-specific units?
6. ☐ What's the allowable latency from raw scan to Silver availability?

---

## 11. Where to Find More Detail

- **DATA_INVENTORY.md** - Full technical spec with real YAML examples (30-45 min read)
- **VISUAL_OVERVIEW.md** - Architecture diagrams and entity relationships
- **IMPLEMENTATION_SCAFFOLD.md** - 4-phase roadmap and timeline
- **Sample YAML files** - In `/GelSightAnalysis/*/analysis/scancontext.yaml`

---

## 12. Key Contact Points

- **Questions about data?** → See DATA_INVENTORY.md sections 2-4
- **Questions about routines?** → See routine analysis in DATA_INVENTORY.md section 3
- **Questions about Bronze ingestion?** → See DATA_INVENTORY.md section 13
- **Questions about future expansion?** → See design decisions section above (5 & 6)
- **Questions about schedule?** → See IMPLEMENTATION_SCAFFOLD.md
