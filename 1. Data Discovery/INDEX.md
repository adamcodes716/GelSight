# 1. Data Discovery – Complete Index

**What is this folder?**  
Complete understanding of GelSight's data structure, file formats, and analysis routines. Everything you need to know before building the cloud platform.

**Created**: February 4, 2026  
**Status**: Phase 0 Discovery – Initial analysis complete

---

## **START HERE (Pick Your Role)**

### 👔 **Project Manager / Stakeholder**
→ Read: [README.md](README.md) (5 min)  
→ Then: [VISUAL_OVERVIEW.md](VISUAL_OVERVIEW.md) (10 min)  
→ Key takeaway: Understanding of 3-layer architecture and main challenges

### 👨‍💼 **Technical Lead / Architect**
→ Read: [README.md](README.md)  
→ Then: [IMPLEMENTATION_SCAFFOLD.md](../IMPLEMENTATION_SCAFFOLD.md) (in repo root)  
→ Then: [QUICK_REFERENCE.md](QUICK_REFERENCE.md)  
→ Key takeaway: Phase 0-4 planning, decision points, stakeholder roles

### 👨‍💻 **Data Modeler** (The person who designs the schema)
→ Read: [QUICK_REFERENCE.md](QUICK_REFERENCE.md) (3 min)  
→ Then: [DATA_INVENTORY.md](DATA_INVENTORY.md) Section 1-12 (30 min)  
→ Then: [VISUAL_OVERVIEW.md](VISUAL_OVERVIEW.md) (15 min)  
→ Then: Sample files in `/sample_schemas/`  
→ Key deliverable: Silver layer dimensional model (fact/dimension tables)

### 👨‍🔧 **Data Engineer** (The person who builds the pipelines)
→ Read: [QUICK_REFERENCE.md](QUICK_REFERENCE.md)  
→ Then: [DATA_INVENTORY.md](DATA_INVENTORY.md) sections 1, 3, 4, 5, 13  
→ Then: [PHASE_0_CHECKLIST.md](PHASE_0_CHECKLIST.md)  
→ Key deliverable: Bronze layer ingestion pipeline

### 📊 **BI/Analytics Engineer** (Dashboards, reporting)
→ Read: [QUICK_REFERENCE.md](QUICK_REFERENCE.md)  
→ Then: [DATA_INVENTORY.md](DATA_INVENTORY.md) sections 2, 6  
→ Then: [VISUAL_OVERVIEW.md](VISUAL_OVERVIEW.md)  
→ Key deliverable: Gold layer views, BI dashboards

---

## **DOCUMENT GUIDE**

### **1. README.md** (Start here – 10 min read)
- Summary of Phase 0 findings
- Big insights (5 critical realizations)
- Big questions (what to ask GelSight)
- Timeline to data modeler handoff
- Immediate action items

**When to read**: First thing. Sets context for all other docs.

---

### **2. QUICK_REFERENCE.md** (One-pager – 5 min read)
- File types summary (scan.yaml, scancontext.yaml, calibration)
- Three coordinate systems
- Five analysis routines overview
- Devices in the data
- Key reminders

**When to read**: Before diving into technical details. Great for new team members.

---

### **3. DATA_INVENTORY.md** (The Bible – 45 min read)
**Most important document.** Complete technical specification of all data.

| Section | Contents | For Whom |
|---------|----------|----------|
| 1. Executive Summary | File categories, key findings | Everyone |
| 2. Scan Metadata (`scan.yaml`) | Complete schema, all fields | Data engineer, data modeler |
| 3. Shape & Routine Data (`analysis/scancontext.yaml`) | Schemas for all 5 routines | Data modeler (critical!) |
| 4. Calibration Files | Complete calibration schema | Data modeler, data engineer |
| 5. 3D Surface Files | `.tmd`, `.x3p`, `.stl` formats | Data engineer |
| 6. Raw Images | Image types, storage strategy | Data engineer, BI engineer |
| 7. Schema Variability | The heterogeneity problem | Data modeler (critical!) |
| 8. Spatial Coordinates | Pixel-to-mm transformation | Data engineer, data modeler |
| 9. Data Quality Flags | `meta_PassedAnalysis` etc. | Data engineer, data scientist |
| 10. Temporal Versioning | Version tracking needs | Data engineer, data modeler |
| 11. Device Ecosystem | Device types, configurations | Data engineer |
| 12. Folder Structure | Organization patterns | Data engineer |
| 13. Bronze Layer Spec | What to ingest, how | Data engineer |

**When to read**: Deep dive. Use as reference manual.

---

### **4. VISUAL_OVERVIEW.md** (Diagrams & Examples – 15 min read)
- Data flow diagram
- Three-layer architecture (visual)
- Entity relationships
- Coordinate transformation explained
- Schema heterogeneity challenge (visual)
- Calibration as SCD Type 2
- Version tracking strategy
- ML vision (where this platform leads)
- File examples (what real YAML looks like)

**When to read**: After README, before deep dive. Helps visualize abstract concepts.

---

### **5. PHASE_0_CHECKLIST.md** (Action Items – 10 min read + tasks)
**Tactical planning for completing Phase 0.**

8 blocks of work:
1. Data file parsing & analysis
2. Calibration mapping & lineage
3. Routine completeness & taxonomy
4. Data quality & failure scenarios
5. Device ecosystem mapping
6. Folder structure & naming
7. Data volume & ingestion
8. Stakeholder alignment

Each block has:
- Specific tasks (with checkboxes)
- Deliverables
- Owner / timeline

**When to use**: Now. Start assigning Phase 0 tasks from this document.

---

### **6. IMPLEMENTATION_SCAFFOLD.md** (Project Plan)
*Note: This file is in the repo root, not this folder.*

4 phases of implementation:
- Phase 0: Discovery (You are here)
- Phase 1: Bronze layer (Data ingestion)
- Phase 2: Silver layer (Data modeling)
- Phase 3: Gold layer (Analytics)
- Phase 4: Edge deployment (ML models on device)

Each phase has:
- Objectives
- Key decisions (with decision matrix)
- Key questions
- Deliverables
- Critical assumptions & risks

**When to use**: Project planning, stakeholder communication.

---

## **SAMPLE FILES**

### **location**: `/sample_schemas/`

Contains real YAML files from each routine type:
- `Offset/scan.yaml` + `scancontext.yaml`
- `DefectDetection/scan.yaml` + `scancontext.yaml`
- `HoleDiameter/scan.yaml` + `scancontext.yaml`
- `PitDetection/scan.yaml` + `scancontext.yaml`
- `SurfaceRoughness/scan.yaml` + `scancontext.yaml`
- `Calib-4E07-XNJU_20250917_1255.yaml`
- `Calib-436U-Y58K_20250912_0928.yaml`

**Use case**: Data modelers examine these to understand real-world data structure.

---

## **DECISION MATRIX (For Data Modeler)**

Key decisions that need to be made for Silver layer design:

### **Fact Table Design**
| Approach | Pros | Cons | Use Case |
|----------|------|------|----------|
| **Option A: Generic fact table** | Maximum flexibility, easy to add new routines | Complex queries, redundant metadata | Future-proof, lots of new routines expected |
| **Option B: Routine-specific tables** | Simple queries, strict schema | Hard to add new routines, table explosion | Stable routine set, strong typing needed |
| **Option C: Hybrid** | Best of both | Complex to maintain | Recommended (common attributes shared + routine specifics separate) |

**Decision**: Data modeler chooses based on Section 12 of DATA_INVENTORY.md

---

### **Measurement Normalization**
| Decision | Impact | Timeline |
|----------|--------|----------|
| **Normalize pixel → mm in Bronze** | Simple, early commitment, can't change later | Fast BI, depends on correct mmperpixel |
| **Normalize in Silver** | Flexible, enable audit trail, enables reprocessing | Delayed conversion, adds complexity |
| **Store both** | Maximum reproducibility | Uses storage space |

**Decision**: Data modeler chooses based on tradeoffs.

---

### **Coordinate Systems**
| Approach | Pixels Stored | MM Stored | Calibration Applied | Best For |
|----------|---------------|-----------|---------------------|----------|
| **Raw only** | ✓ | ✗ | No | Audit trail, flexibility |
| **Converted only** | ✗ | ✓ | Yes | Performance, simplicity |
| **Both** | ✓ | ✓ | Yes | Reproducibility, audit trail |

**Decision**: Data modeler chooses based on requirements.

---

## **QUESTIONS TO ASK GELSIGHT (From README)**

### Blocking Phase 1:
1. What are `io_*/` folders?
2. How will scans be delivered?
3. How frequently are scans created?
4. How are calibrations versioned?

### Blocking Phase 2:
5. Are there routines beyond the 5 we've seen?
6. One sample of each routine type?
7. What do cryptic fields mean?
8. How is nominal diameter provided?

### Planning Phase 3-4:
9. Tolerance ranges per measurement?
10. How will pass/fail be determined?
11. What ML models exist?
12. What historical data for ML training?

---

## **TIMELINE**

| Week | Task | Owner | Status |
|------|------|-------|--------|
| 1 | Schedule GelSight kickoff | Tech Lead | 🟡 Todo |
| 2-3 | Complete Phase 0 checklist items | Data Engineer | 🟡 Todo |
| 3-4 | GelSight answers all questions | GelSight | ⏳ Waiting |
| 4-5 | Data modeler onboarding | Tech Lead | 🟡 Todo |
| 5-6 | **Phase 2 design kickoff** | Data Modeler | 🟡 Ready |
| 6-12 | Phase 2 implementation | Data Engineer + Modeler | ⏳ Waiting |

---

## **KEY TAKEAWAYS**

✅ **You now understand**:
- Complete file structure (scan.yaml, scancontext.yaml, calibration)
- Heterogeneous schemas (each routine outputs different measurements)
- Coordinate systems (pixel → mm transformation)
- Calibration versioning (SCD Type 2 dimension)
- Device ecosystem (multiple devices, different calibrations)
- Spatial data complexity (shapes as dimensions, measurements as facts)

✅ **You know what to ask GelSight**:
- All 12 critical questions documented

✅ **You know the next phase**:
- Phase 0 checklist has 8 blocks of work
- Data modeler joins after questions answered
- Phase 2 begins with dimensional modeling

✅ **You have a roadmap**:
- IMPLEMENTATION_SCAFFOLD.md covers phases 0-4
- PHASE_0_CHECKLIST.md covers next 4-6 weeks

---

## **QUICK LINKS**

- **Project plan**: `../IMPLEMENTATION_SCAFFOLD.md`
- **Data deep dive**: `DATA_INVENTORY.md` (this folder)
- **Quick summary**: `QUICK_REFERENCE.md` (this folder)
- **Visual diagrams**: `VISUAL_OVERVIEW.md` (this folder)
- **Phase 0 tasks**: `PHASE_0_CHECKLIST.md` (this folder)
- **Sample schemas**: `/sample_schemas/` (this folder)

---

## **QUESTIONS?**

| Question | Answer Location |
|----------|-----------------|
| What is this data? | README.md or QUICK_REFERENCE.md |
| How is it structured? | DATA_INVENTORY.md or VISUAL_OVERVIEW.md |
| What do I do next? | PHASE_0_CHECKLIST.md |
| How does this fit into the project? | IMPLEMENTATION_SCAFFOLD.md |
| How should we model it? | DATA_INVENTORY.md Section 12 + data modeler input |
| How do we ingest it? | DATA_INVENTORY.md Section 13 + data engineer design |

---

**Document Version**: 1.0  
**Last Updated**: February 4, 2026  
**Next Review**: After GelSight engineering kickoff (Week 2-3)

