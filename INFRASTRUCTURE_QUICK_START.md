# GelSight Data Platform – Infrastructure Quick Start Guide

**Purpose**: High-level overview of infrastructure components and how they fit together.

**Date**: February 11, 2026  
**Audience**: All team members, new hires, stakeholders  
**Read Time**: 5-10 minutes

---

## **INFRASTRUCTURE AT A GLANCE**

### **What We're Building**

```
┌─────────────────────────────────────────────────┐
│   GelSight Data Platform (Azure-based)         │
├─────────────────────────────────────────────────┤
│                                                 │
│  Input: Raw scan files (YAML, images, 3D)      │
│    ↓                                            │
│  [Bronze Layer: Ingest & Parse]                │
│    ↓                                            │
│  [Silver Layer: Transform & Model]             │
│    ↓                                            │
│  [Gold Layer: Analyze & Report]                │
│    ↓                                            │
│  Output: Dashboards, ML datasets, trends       │
│                                                 │
└─────────────────────────────────────────────────┘
```

### **Key Components**

| Component | Purpose | Technology |
|-----------|---------|-----------|
| **Storage** | Raw data + transformed data | Azure Data Lake Storage Gen2 (ADLS2) |
| **Compute** | Data processing + transformations | Azure Databricks |
| **Database** | Metadata + dimensional model | Azure SQL Database |
| **Orchestration** | Schedule pipelines | Azure Data Factory (ADF) |
| **Secrets** | Secure credentials | Azure Key Vault |
| **Code Repo** | Source code + CI/CD | GitHub |
| **Monitoring** | Track health + performance | Azure Monitor + Application Insights |

---

## **INFRASTRUCTURE OVERVIEW**

### **Three Environments**

```
DEV (Development)           UAT (Staging)             PROD (Production)
├─ Shared compute          ├─ Dedicated cluster       ├─ Auto-scaling cluster  
├─ Dev SQL Database        ├─ UAT SQL Database       ├─ Prod SQL Database
├─ Dev storage (GRS)       ├─ UAT storage (GRS)      ├─ Prod storage (GRS)
├─ Manual pipelines        ├─ Scheduled pipelines    ├─ Scheduled pipelines
└─ Run tests here          └─ Full regression tests  └─ Live customer data
```

### **Storage Layout**

**Bronze Layer** (Raw Ingestion):
```
gelsightbronzedev/
├── scans/{scan_guid}/
│   ├── metadata/ (scan.yaml parsed to JSON)
│   ├── analysis/ (scancontext.yaml parsed to JSON)
│   ├── images/ (aligned*.png files)
│   └── processing_log.json
├── calibrations/{calib_id}/
│   ├── calib_metadata.json
│   └── coefficients.json
└── [Immutable, 7-year retention]
```

**Silver Layer** (Normalized Data):
```
gelsightsilbergolddev/
├── dim-calibration/ (Parquet, SCD Type 2)
├── dim-device/ (Parquet)
├── dim-shapes/ (Parquet)
├── dim-routines/ (Parquet)
├── fact-measurements/ (Parquet, partitioned by date)
└── fact-scan-metadata/ (Parquet)
```

**Gold Layer** (Analytics):
```
gelsightsilbergolddev/
├── qc-reporting/ (Quality control reports)
├── measurement-trends/ (Time-series aggregates)
├── operator-performance/ (KPIs)
└── ml-training-datasets/ (ML feature sets)
```

### **Database Schema** (SQL)

Stores:
- `dim_calibration`: Optical calibrations (versioned, SCD Type 2)
- `dim_device`: GelSight hardware info
- `dim_shapes`: Detected geometric shapes
- `dim_routines`: Analysis routine definitions
- `fact_measurements`: Measurements (one row per measurement)
- `fact_scan_metadata`: Scan context
- `ingestion_log`: Processing metadata (what was ingested when)
- `data_quality_metrics`: Quality scores

---

## **DATA FLOW**

```
Day 1: New Scans Collected
└─ Operator runs analysis on GelSight device
└─ Files saved: scan.yaml, analysis/scancontext.yaml, images, calibration, 3D files

Day 2: ADF Bronze Ingest Pipeline (Nightly)
└─ Trigger: Timer (2 AM) or HTTP POST
└─ Activity 1: Copy files from input location to ADLS Bronze
└─ Activity 2: Databricks notebook 01_ingest_scans.py
   └─ Parse YAML files → JSON/structured
   └─ Store in /mnt/bronze/scans/{scan_guid}/
└─ Activity 3: Databricks notebook 02_ingest_calibrations.py
   └─ Parse Calib-*.yaml files
   └─ Deduplicate, store in /mnt/bronze/calibrations/
└─ Activity 4: Validate (schema checks, completeness)
└─ Activity 5: Update ingestion_log table in SQL DB
└─ Result: Raw data in Bronze, metadata in SQL

Day 3: ADF Silver Transform Pipeline (Nightly)
└─ Trigger: ADF dependency on Bronze completion
└─ Activity 1: Databricks notebook 01_transform_calibrations.py
   └─ Build dim_calibration table (SCD Type 2)
   └─ Parse coefficients, track lineage
└─ Activity 2: Build dim_device, dim_shapes, dim_routines
└─ Activity 3: Databricks notebook 04_transform_measurements.py
   └─ Convert pixel → mm coordinates
   └─ Build fact_measurements table
   └─ Link to calibration, device, shape dimensions
└─ Activity 4: Quality checks (validation rules)
└─ Result: Normalized tables in Silver layer, ready for analysis

Day 4: ADF Gold Analytics Pipeline (Nightly)
└─ Trigger: ADF dependency on Silver completion
└─ Activity 1: Databricks notebook 01_qc_reporting.py
   └─ Build pass/fail tables
   └─ Create quality reporting views
└─ Activity 2: Build measurement trends
└─ Activity 3: Build operator performance KPIs
└─ Activity 4: Prepare ML training datasets
└─ Result: Analytics-ready data in Gold layer

Day 5: BI Tools & Reports
└─ Power BI connects to Gold layer
└─ Operators see QA dashboards
└─ Trends visible, pass/fail visible
└─ ML engineers use training datasets
```

---

## **CODE STRUCTURE**

```
Repository: gelsight-data-platform (GitHub)

bronze/                          # Phase 1: Data Ingestion
├── notebooks/
│   ├── 01_ingest_scans.py      # Parse scan YAML
│   ├── 02_ingest_calibrations.py
│   ├── 03_ingest_images.py
│   └── 04_validate_bronze.py   # Quality checks
├── src/                         # Python modules
│   ├── ingestion.py
│   ├── validation.py
│   └── metadata_tracking.py
└── tests/
    └── test_*.py               # Unit tests

silver/                         # Phase 2: Transformation
├── notebooks/
│   ├── 01_transform_calibrations.py  # Build dimensions
│   ├── 02_transform_devices.py
│   ├── 03_transform_shapes.py
│   ├── 04_transform_measurements.py  # Build facts
│   └── 05_quality_checks.py
├── src/
│   ├── transformations.py
│   ├── coordinate_transform.py
│   └── quality_rules.py
└── tests/
    └── test_*.py

gold/                           # Phase 3: Analytics
├── notebooks/
│   ├── 01_qc_reporting.py
│   ├── 02_measurement_trends.py
│   ├── 03_operator_performance.py
│   └── 04_ml_dataset_prep.py
└── tests/

infrastructure/                 # Infrastructure as Code
├── terraform/
│   ├── main.tf
│   ├── storage.tf
│   ├── database.tf
│   ├── variables.tf
│   └── env/
│       ├── dev.tfvars
│       ├── uat.tfvars
│       └── prod.tfvars
└── scripts/
    ├── deploy.sh
    └── init-databricks.sh

pipelines/                      # Azure Data Factory
├── templates/
│   ├── bronze_ingest_pipeline.json
│   ├── silver_transform_pipeline.json
│   └── gold_analytics_pipeline.json
└── linked_services/

.github/workflows/              # CI/CD (GitHub Actions)
├── ci-build.yml               # Run tests on PR
├── deploy-dev.yml             # Deploy to Dev after merge
├── deploy-uat.yml             # Deploy to UAT (manual)
└── deploy-prod.yml            # Deploy to Prod (manual + approval)
```

---

## **GIT WORKFLOW**

```
Developer creates feature branch
    ↓
    feature/GELS-123-my-feature
    ↓
Makes commits (git commit)
    ↓
Push to GitHub (git push)
    ↓
Create Pull Request
    ↓
GitHub Actions: ci-build.yml
├─ Run tests
├─ Check coverage (must be ≥80%)
├─ Lint code
└─ ✓ PASS or ✗ FAIL
    ↓
Code Review (1+ approvals)
    ↓
Merge to develop
    ↓
GitHub Actions: deploy-dev.yml
├─ Upload notebooks to Dev Databricks
├─ Run integration tests
└─ Notify team
    ↓
Dev Ready ✓
(test here for 1-2 weeks)
    ↓
Tech Lead: Create release/v0.1.0
    ↓
GitHub Actions: deploy-uat.yml
├─ Deploy to UAT Databricks
├─ Full regression testing
└─ Email UAT team
    ↓
UAT Testing (1-2 weeks)
    ↓
⏳ Approval Gate: Requires 2 approvals + environment approval
    ↓
GitHub Actions: deploy-prod.yml
├─ Backup databases & storage
├─ Deploy to Production
├─ Run smoke tests
└─ Notify stakeholders
    ↓
Production Ready ✓
(merge to main, tag with version)
```

---

## **COST BREAKDOWN**

**Monthly costs** (estimated) by environment:

| Component | Dev | UAT | Prod | Notes |
|-----------|-----|-----|------|-------|
| Databricks | $500 | $1,500 | $3,000 | Autoscaling clusters |
| Storage (ADLS2) | $200 | $500 | $1,500 | 100GB-1TB growth/month |
| SQL Database | $200 | $500 | $1,500 | Standard tier |
| Data Factory | $100 | $200 | $500 | Pay per pipeline run |
| Key Vault | $20 | $20 | $20 | Fixed cost |
| Monitoring | $50 | $100 | $200 | Logs + metrics |
| Networking | $10 | $30 | $100 | Bandwidth & endpoints |
| **TOTAL** | **~$1,080** | **~$2,850** | **~$7,320** | Per month |

**Optimization opportunities**:
- Use reserved instances for Databricks (save 25-30%)
- Archive Bronze data after 2 years (save on storage)
- Use spot VMs for dev/UAT (save 50-70%)

---

## **NEXT STEPS (This Week)**

1. **Review** this architecture with stakeholders
2. **Gather** Azure subscription details
3. **Create** GitHub repository and folder structure
4. **Start** INFRASTRUCTURE_SETUP_CHECKLIST.md (Phase 1)
   - Create storage accounts
   - Create databases
   - Configure Key Vault
   - Set up Databricks mounts
5. **Run** ci-build.yml and deployment workflows (test)

---

## **KEY DOCUMENTS**

| Document | Purpose | Audience |
|----------|---------|----------|
| **INFRASTRUCTURE_ARCHITECTURE.md** | Complete technical design | Architects, engineers |
| **INFRASTRUCTURE_SETUP_CHECKLIST.md** | Step-by-step setup tasks | DevOps, engineers |
| **CODE_PROMOTION_STRATEGY.md** | Git workflow, code review, deployment | All developers |
| **[This Guide]** | Quick overview | Everyone |

---

## **FREQUENTLY ASKED QUESTIONS**

**Q: Who can deploy to production?**  
A: Only designated Release Manager (requires 2-factor auth + 2 approvals)

**Q: What if something breaks in production?**  
A: Automated backup restores. Manual procedures in runbooks. Post-mortem after incident.

**Q: How long does deployment take?**  
A: Dev: 30 min | UAT: 2 hours | Prod: 2 hours (+ manual approval)

**Q: Can I rollback a deployment?**  
A: Yes, restore from backup (database & storage). Automated workflow available.

**Q: What if I need to hotfix something critical?**  
A: Create hotfix/* branch from main, expedited review (1 approval), fast-track to prod.

**Q: How do I know if my code is ready to deploy?**  
A: Merge to develop → runs in Dev env → soak for 1-2 weeks → release candidate → UAT → prod

**Q: Where do secrets go?**  
A: Azure Key Vault only. Never hardcode passwords/keys in notebooks or code.

---

## **CONTACT & SUPPORT**

| Question | Contact | Channel |
|----------|---------|---------|
| Infrastructure issues | Cloud Architect | #cloud-infrastructure Slack |
| Code review questions | Tech Lead | code-review@company.com |
| Deployment issues | DevOps Engineer | #data-platform Slack |
| Data issues | Data Engineer | #data-modeling Slack |
| Emergency (production down) | On-call engineer | PagerDuty |

---

**Document Version**: 1.0  
**Last Updated**: February 11, 2026  
**Next Review**: After first infrastructure deployment (Week 4)
