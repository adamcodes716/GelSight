# GelSight Data Platform – Implementation Scaffold

**Document Purpose**: Strategic planning framework and decision matrix for the GelSight data modernization project. This is a *thinking tool* to organize the journey from current state to the Bronze → Silver → Gold architecture.

**Last Updated**: February 4, 2026  
**Audience**: Project stakeholders, technical architect, data modelers, analytics engineers  
**Status**: Pre-execution planning phase

---

## **EXECUTIVE SUMMARY**

GelSight is building a **three-layer data platform** that will:
1. Ingest all scan artifacts (images, 3D files, analysis YAML)
2. Normalize "tens" of heterogeneous analysis formats into a flexible schema
3. Enable automated QA reporting, ML training, and eventually device-edge deployment

This document maps the journey and identifies critical decision points.

---

## **PHASE 0: DISCOVERY & INVENTORY (Current Phase)**

### **Objectives**
- [ ] Fully catalog the data universe
- [ ] Understand each analysis routine's semantics
- [ ] Identify data quality issues and gaps
- [ ] Establish baseline metrics

### **Key Questions to Answer**

| Question | Owner | Input Source | Status |
|----------|-------|--------------|--------|
| What analysis routines exist in GelSight's software? | Product team | GelSight software docs | Incomplete |
| What is the complete schema for each routine's YAML output? | Technical lead | Sample YAML files + GelSight engineering | Incomplete |
| How is calibration versioned and tracked? | GelSight engineering | Device firmware / calibration logs | Unknown |
| What does each 3D file format (STL, TMD, X3P) contain? Do they encode the same geometry? | Data engineer | File specs + sample files | Incomplete |
| What is in the `io_*` folders (e.g., `io_881565834/`)? Intermediate outputs? Images? | GelSight engineering | File exploration + docs | Unknown |
| How many unique Calibration files exist? How often are they updated? | Data engineer | Scan metadata inventory | Incomplete |
| Are there scan/analysis failures or missing outputs? | QA team | Historical scan logs | Unknown |
| What is the current naming/versioning scheme for scans? | Data engineer | File system inspection | Incomplete |

### **Deliverables**
- [ ] **Data Inventory Document**: Catalog all file types, folder structures, naming patterns
- [ ] **Routine Specification Document**: For each routine (DefectDetection, HoleDiameter, Offset, PitDetection, SurfaceRoughness), document:
  - What it measures
  - Expected input (calibration, scan image, 3D file)
  - YAML schema (example + all fields)
  - Success criteria (how to know it worked)
  - Known failure modes
- [ ] **Sample Files Archive**: Extract representative samples of each file type for modeling team
- [ ] **Data Quality Baseline**: Row counts, nulls, anomalies, schema variance

---

## **PHASE 1: BRONZE LAYER ARCHITECTURE (Data Ingestion)**

### **Objectives**
- Define how to ingest raw artifacts into cloud storage
- Establish reproducibility (lineage, versioning, audit trails)
- Create the "single source of truth" for raw data

### **Key Decisions**

| Decision | Options | Recommendation | Data Modeler Input | Owner |
|----------|---------|-----------------|-------------------|-------|
| **Cloud Storage Layout** | Flat blob structure vs. nested by routine vs. hierarchical by scan/date | Hierarchical by scan ID + routine (mirrors source) | Minimal | Platform architect |
| **Metadata Tracking** | Embed in folder structure vs. separate metadata DB vs. file tagging | Separate metadata DB linked to cloud objects | Strongly recommend | Platform architect |
| **File Format Preservation** | Keep YAML, TMD, STL, X3P as-is vs. convert to standard formats | Keep original + derive standard formats in Silver | Required | Platform architect |
| **Partitioning Strategy** | By date / by routine / by device / hybrid | By scan date + routine (enables fast queries) | **Data modeler decides** | Data modeler |
| **Versioning** | Single-version-per-file vs. audit trail of all versions | Audit trail (immutable, append-only) | Strongly recommend | Platform architect |

### **Key Questions**

- [ ] Does GelSight have an automated way to export scan data to a standardized format? Or do we need to manually choreograph drops?
- [ ] What is the expected ingestion frequency? (Daily? Weekly? Ad-hoc?)
- [ ] What is the expected data volume? (TB/month? PB/year?)
- [ ] Are there data retention requirements? (HIPAA? Industry-specific? Customer contracts?)
- [ ] Who approves ingestion? (Automated or human gate?)

### **Deliverables**
- [ ] **Bronze Layer Cloud Architecture Document**: Storage account design, naming conventions, folder structure, partitioning
- [ ] **Metadata Schema**: Database schema or file catalog for tracking lineage
- [ ] **Ingestion Pipeline Spec**: What triggers ingestion? What validation occurs? What happens on failure?
- [ ] **Data Dictionary (Bronze)**: For each file type, document all columns/fields and their semantics

---

## **PHASE 2: SILVER LAYER DESIGN (Transformation & Normalization)**

### **Objectives**
- Design a flexible schema that accommodates all current + future analysis routines
- Normalize routine-specific YAML into fact/dimension tables
- Create "shapes" and "measurements" as normalized entities

### **Key Decisions**

| Decision | Options | Recommendation | Data Modeler Input | Owner |
|----------|---------|-----------------|-------------------|-------|
| **Fact/Dimension Model** | Star schema vs. Snowflake vs. Event streaming vs. Document DB | Star schema (proven for this use case) | **Data modeler leads** | Data modeler |
| **Measurement Storage** | Normalize all measurements into one table vs. routine-specific tables | Hybrid: common attributes in shared table, routine-specific in separate tables | **Data modeler decides** | Data modeler |
| **Shape Representation** | Store shapes as geometry objects vs. flattened attributes | Flattened attributes + geometry reference | **Data modeler decides** | Data modeler |
| **Calibration Handling** | Inline in measurement vs. separate slowly-changing dimension | Separate SCD Type 2 (track history) | Strongly recommend | Data modeler |
| **Data Types & Precision** | Store all numeric as double vs. preserve original precision | Preserve original (traceability) | Required | Data modeler |

### **Key Questions**

- [ ] What is the *minimum* information needed to reproduce a measurement? (Calibration + image + 3D file + algorithm version?)
- [ ] Do shapes overlap? Can a single point belong to multiple shapes?
- [ ] Are there hierarchies in shapes? (e.g., "container defect" → "sub-defects")
- [ ] What is the expected cardinality? (Scans with 1 shape? 100 shapes? 1000?)
- [ ] Are there temporal dynamics? (Shapes detected at different stages of analysis?)

### **Deliverables**
- [ ] **Dimensional Model Specification**: ERD showing fact tables, dimension tables, and relationships
- [ ] **Attribute Catalog**: For each dimension/fact, list all attributes, data types, business rules
- [ ] **Shape Taxonomy**: Classify all possible shapes (circle, line, rectangle, ruler, region, etc.)
- [ ] **Routine Mapping Document**: For each routine, show how its YAML schema maps to the fact/dimension model
- [ ] **Data Quality Rules**: For each table, document what constitutes valid/invalid data
- [ ] **Transformation Logic Spec**: Pseudo-code or decision trees for parsing YAML → normalized tables

### **Expected Complexity**

This phase is where **domain expertise is critical**. The data modeler needs to:
- Understand shape detection algorithms (What is a "shape" really?)
- Understand measurement semantics (Is a "hole diameter" different from a "circle radius"? How?)
- Predict future analysis types (What schema will accommodate routines not yet built?)

---

## **PHASE 3: GOLD LAYER & ANALYTICS (Ready for BI/ML)**

### **Objectives**
- Create clean, business-consumable views
- Enable QA dashboards and trend reporting
- Prepare datasets for ML training

### **Key Decisions**

| Decision | Options | Recommendation | Owner |
|----------|---------|-----------------|--------|
| **BI Tool** | Power BI vs. Tableau vs. Looker vs. custom dashboards | (Customer preference?) | Project manager |
| **ML Platforms** | Azure ML vs. Databricks vs. custom Python vs. MLflow | Depends on customer infrastructure | Data scientist |
| **Real-time vs. Batch** | Streaming analytics vs. daily snapshot vs. on-demand | Daily snapshots initially, streaming later | Platform architect |

### **Key Questions**

- [ ] What are the top 10 reports GelSight operators need?
- [ ] What SLAs are required? (How fast must a dashboard update after a scan?)
- [ ] What are the ML success metrics? (Precision/recall/F1 targets for defect detection?)
- [ ] How will ground truth be established for ML training? (Manual labeling? Statistical bounds?)

### **Deliverables**
- [ ] **Gold Layer Views**: SQL definitions for each BI/analytics query
- [ ] **Reporting Requirements Document**: Wireframes or mockups of key dashboards
- [ ] **ML Dataset Specs**: Which tables, which time windows, class balance, feature engineering rules
- [ ] **Monitoring & Alerting**: What metrics indicate data/pipeline quality issues?

---

## **PHASE 4: EDGE DEPLOYMENT (Future State)**

### **Objectives**
- Package trained ML models for device hardware
- Enable autonomous inspection at the edge

### **Key Decisions**

- [ ] What hardware does GelSight's device run? (Windows? Linux? Embedded RTOS?)
- [ ] What are the latency/memory constraints?
- [ ] What model frameworks are compatible? (ONNX? TensorFlow Lite? PyTorch?)

### **Deliverables**
- [ ] **Model Packaging Spec**
- [ ] **Device Integration Plan**
- [ ] **Feedback Loop Design** (How do edge predictions flow back to cloud for model retraining?)

---

## **WORKSTREAMS & SEQUENCING**

```
Phase 0 (Discovery)           Phase 1 (Bronze)           Phase 2 (Silver)         Phase 3 (Gold)
├─ Data Inventory             ├─ Cloud Architecture      ├─ Dimensional Model     ├─ BI Dashboards
├─ Routine Specs              ├─ Ingestion Pipeline      ├─ Transformation Code   ├─ ML Datasets
├─ Sample Files               ├─ Metadata Tracking       ├─ Data Quality Rules    └─ Monitoring
└─ Quality Baseline           └─ Validation Rules        └─ Documentation

Timeline (estimate):
Phase 0:  4–6 weeks (parallel data exploration)
Phase 1:  2–3 weeks (infrastructure, should be fast)
Phase 2:  6–10 weeks (data modeler + engineers, highest complexity)
Phase 3:  3–4 weeks (BI + analytics)
Phase 4:  Ongoing (model development, deployment, feedback loops)
```

---

## **CRITICAL ASSUMPTIONS & RISKS**

### **Assumptions**
1. GelSight can provide regular data drops (or API access)
2. The same calibration file applies to all routines in a scan
3. YAML schemas are stable (not frequently changing)
4. Ground truth labels for ML training are available or can be created
5. Azure is the target cloud platform

### **Risks & Mitigations**

| Risk | Impact | Mitigation |
|------|--------|-----------|
| **YAML schema variability** | Silver layer redesign | Conduct Phase 0 discovery thoroughly; build schema flexibility into model |
| **Missing/corrupted files** | Incomplete scans in warehouse | Implement validation in Phase 1; track data quality metrics |
| **Calibration errors** | Measurement inaccuracy | Track calibration version; enable reprocessing with new calibration |
| **ML model accuracy too low** | Edge deployment delays | Start with realistic targets; plan for human-in-the-loop initially |
| **Data volume explosion** | Cost/performance issues | Plan storage strategy early; implement tiering/archival in Phase 1 |
| **Scope creep (more routines)** | Schedule slips | Build Phase 2 schema with future flexibility; gate new routines |

---

## **STAKEHOLDER ROLES**

| Role | Responsibilities | Key Decisions |
|------|------------------|---------------|
| **Data Modeler** | Design fact/dimension schema, cardinality analysis, attribute catalog | Silver layer architecture |
| **Data Engineer** | Ingestion pipeline, data quality, transformation logic | Bronze & Silver implementation |
| **Cloud Architect** | Azure infrastructure, scalability, security | Platform, storage, networking |
| **BI/Analytics Engineer** | Dashboards, reporting, Gold layer views | BI tool selection, dashboard design |
| **Data Scientist / ML Engineer** | Model training, validation, edge deployment | ML architecture, success metrics |
| **GelSight Product/Engineering** | Requirements, validation, edge device constraints | Routine specs, hardware constraints |
| **Project Manager** | Timeline, budget, stakeholder alignment | Phase gates, resource allocation |

---

## **SUCCESS CRITERIA**

### **Phase 0**
- [ ] Data inventory complete (100% of file types cataloged)
- [ ] All routine YAML schemas documented
- [ ] Sample files acquired for modeling team
- [ ] Team alignment on target architecture

### **Phase 1**
- [ ] First data successfully ingested to cloud storage
- [ ] Metadata tracking operational
- [ ] Validation rules enforced on ingest
- [ ] No data loss or corruption

### **Phase 2**
- [ ] All existing routines successfully mapped to normalized schema
- [ ] Data quality metrics at >95% completeness
- [ ] Schema flexible enough to accommodate new routine (proof-of-concept)
- [ ] Data lineage traceable from Gold → Silver → Bronze

### **Phase 3**
- [ ] Top 3 dashboards operational
- [ ] ML datasets prepared and balanced
- [ ] Operators can make QA decisions from Gold layer data
- [ ] Trend analysis operational

### **Phase 4**
- [ ] ML model accuracy meets targets (TBD with GelSight)
- [ ] Model packaged for device hardware
- [ ] Edge deployment plan validated with GelSight engineering

---

## **NEXT STEPS (Immediate Actions)**

1. **Schedule kickoff with GelSight engineering**
   - Confirm Phase 0 questions (see table above)
   - Request access to software documentation
   - Arrange access to device for hands-on exploration

2. **Assemble data modeler**
   - Share this scaffold document
   - Have them review sample files (once acquired)
   - Schedule design review for Phase 2

3. **Set up Cloud environment**
   - Create Azure subscription / resource group
   - Establish naming conventions, RBAC policies
   - Prepare for Phase 1 infrastructure

4. **Create project tracking** (e.g., Azure DevOps, Jira)
   - Map workstreams to user stories
   - Assign Phase 0 discovery tasks
   - Set 2-week sprints

5. **Document assumptions**
   - Circulate this scaffold to stakeholders
   - Collect feedback on Phase 0 priorities
   - Lock in timeline and budget

---

## **APPENDIX: REFERENCE MATERIALS**

- [Project Summary](./Project%20Summary.md) – High-level vision and context
- [Deep Dive Analysis](./ANALYSIS_DEEP_DIVE.md) – Detailed technical breakdown (optional, if you want to capture my earlier analysis)
- Sample files: `/GelSightAnalysis/*/analysis/scancontext.yaml` – Real schema examples
- Workspace structure: See `Project%20Summary.md` → Resources for you to analyze

---

**Document Version**: 1.0  
**Next Review**: After Phase 0 discovery complete (approx. 6 weeks)
