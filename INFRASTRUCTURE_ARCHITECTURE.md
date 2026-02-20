# GelSight Data Platform – Infrastructure Architecture

**Document Purpose**: Define cloud infrastructure, databases, storage, code repositories, pipelines, and code promotion strategy for the data modernization platform.

**Date**: February 11, 2026  
**Status**: Pre-implementation  
**Target**: Ready for Phase 1 (Bronze Layer Ingestion)

---

## **EXECUTIVE SUMMARY**

This document covers the complete infrastructure stack needed to support the 3-layer data platform (Bronze → Silver → Gold):

- **Cloud Platform**: Azure (already has Landing Zone + Databricks)
- **Storage Layer**: Azure Data Lake Storage Gen2 (ADLS2) for Bronze/Silver/Gold
- **Compute**: Azure Databricks (already set up) + optional Azure Functions for ingestion
- **Database**: Synapse or SQL Database for metadata + dimensional model
- **Code Repos**: GitHub/Azure DevOps repositories with branching strategy
- **Pipelines**: Azure Data Factory for orchestration; Databricks notebooks/jobs for ETL
- **Code Promotion**: Dev → UAT → Prod environments with CI/CD automation

---

## **CURRENT STATE (What You Have)**

Based on your existing setup:

| Component | Status |
|-----------|--------|
| Azure Landing Zone | ✅ Created |
| Networking | ✅ Configured |
| Azure Databricks Workspace | ✅ Created |
| Compute Clusters | ✅ Configured |
| **Missing**: Storage accounts | ⏳ Need to create |
| **Missing**: Databases | ⏳ Need to create |
| **Missing**: Code repositories | ⏳ Need to create |
| **Missing**: Pipelines | ⏳ Need to create |
| **Missing**: Code promotion strategy | ⏳ Need to define |

---

## **1. STORAGE ARCHITECTURE**

### **Overview**

Using **Azure Data Lake Storage Gen2 (ADLS2)** organized by layer:

```
gelsight-data-bronze/                    (Raw ingestion, immutable)
  ├── scans/
  │   └── {scan_guid}/
  │       ├── metadata/
  │       │   ├── scan_metadata.json
  │       │   ├── analysis_results.json
  │       │   └── raw_yaml/
  │       ├── images/
  │       ├── heightmaps/
  │       └── processing_log.json
  │
  └── calibrations/
      └── {calib_id}/
          ├── calib_metadata.json
          └── raw_yaml/

gelsight-data-silver/                     (Normalized, modeled data)
  ├── dim_calibration/
  │   └── {version_date}/*.parquet
  ├── dim_device/
  │   └── {version_date}/*.parquet
  ├── dim_shapes/
  │   └── {version_date}/*.parquet
  ├── dim_routines/
  │   └── {version_date}/*.parquet
  ├── fact_measurements/
  │   └── {year}/{month}/*.parquet
  └── fact_scan_metadata/
      └── {year}/{month}/*.parquet

gelsight-data-gold/                       (Analytics-ready, aggregated)
  ├── qc_reporting/
  │   └── {date}/*.parquet
  ├── measurement_trends/
  │   └── {date}/*.parquet
  ├── operator_performance/
  │   └── {date}/*.parquet
  └── ml_training_datasets/
      └── {version}/*.parquet
```

### **Storage Account Details**

**Account 1: Bronze Layer**
```
Name: gelsightbronze{env}
Tier: Standard
Replication: GRS (Geo-Redundant Storage)
Access: Private endpoints only
Retention: 7 years (compliance)
Lifecycle: Archive after 2 years
```

**Account 2: Silver + Gold Layers**
```
Name: gelsightsilbergold{env}
Tier: Premium (performance)
Replication: GRS
Access: Private endpoints only
Retention: 5 years (analytics)
Lifecycle: N/A (frequently accessed)
```

### **Key Design Decisions**

| Decision | Rationale |
|----------|-----------|
| **Separate Bronze from Silver/Gold** | Different access patterns, retention, lifecycle |
| **Partition Bronze by scan_guid** | Data lineage tracking, enables replay/reprocessing |
| **Immutable Bronze** | Audit trail, compliance, reproducibility |
| **Parquet format for Silver/Gold** | Column-oriented, Databricks-native, query efficient |
| **Date-based partitioning for facts** | Enables time-window queries, archive/tiering |

---

## **2. DATABASE ARCHITECTURE**

### **Metadata Database** (SQL Database or Synapse SQL Pool)

Stores:
- Ingestion metadata (lineage, versioning, processing status)
- Dimensional model (calibrations, devices, shapes, routines)
- Data quality metrics
- Processing logs

**Recommended**: Azure Synapse Analytics (SQL Pool) if scale demands it; SQL Database for simpler start.

### **Database Schema (Silver Layer)**

```sql
-- DIMENSIONS --

dim_device
  device_id (PK)
  device_model (Modulus, Series2, ...)
  device_firmware
  camera_id
  lens_serial
  created_date

dim_calibration (SCD Type 2)
  calibration_sk (surrogate key)
  calibration_id (natural key)
  device_id (FK)
  version
  mmperpixel
  linfit_coefficients (JSON)
  quadfit_coefficients (JSON)
  is_verified
  effective_date
  end_date
  is_current
  created_date

dim_shapes
  shape_id (PK)
  scan_id (FK)
  shape_type (circle, line, rectangle, ruler)
  x_pixel
  y_pixel
  width / height / radius
  rotation
  created_date

dim_routines
  routine_id (PK)
  routine_name (Offset, DefectDetection, ...)
  routine_description
  measurement_count
  is_active
  created_date

dim_analysis_version_context
  version_context_id (PK)
  sdk_version
  app_version
  device_firmware
  calibration_version
  scan_date_range_start
  scan_date_range_end

-- FACTS --

fact_measurements
  measurement_id (PK)
  scan_id (FK)
  routine_id (FK)
  shape_id (FK)
  calibration_sk (FK to dim_calibration)
  version_context_id (FK)
  measurement_name (offset, length, diameter, etc.)
  measurement_value (numeric)
  measurement_unit (mm, mm², mm³, µm)
  tolerance_min
  tolerance_max
  pass_fail_flag
  confidence_score
  x_pixel
  y_pixel
  x_mm (converted from pixel)
  y_mm (converted from pixel)
  created_date
  processed_date

fact_scan_metadata
  scan_id (PK)
  guid (from scan.yaml)
  device_id (FK)
  operator_name
  scan_timestamp
  mmperpixel_used
  sdk_version
  app_version
  calibration_id (FK)
  image_count
  is_aligned
  processing_status (SUCCESS, FAILED, REPROCESSING)
  created_date

-- METADATA & OPERATIONAL TABLES --

ingestion_log
  log_id (PK)
  scan_id (FK)
  batch_id
  source_path
  target_path_bronze
  file_count
  total_bytes
  parse_status (SUCCESS, FAILED, PARTIAL)
  error_message
  processing_start_time
  processing_end_time
  duration_seconds

data_quality_metrics
  metric_id (PK)
  measurement_date
  table_name
  completeness_pct (non-null %)
  uniqueness_pct
  validity_pct
  consistency_score
  anomaly_count
  created_date

calibration_lineage
  lineage_id (PK)
  scan_id (FK)
  calibration_sk (FK)
  applied_date
  processing_context (routine, reprocessing reason, etc.)
```

---

## **3. CODE REPOSITORIES**

### **Repository Structure**

**Repo 1: `gelsight-data-platform` (Main)**
```
gelsight-data-platform/
├── .github/
│   └── workflows/
│       ├── ci-build.yml
│       ├── deploy-dev.yml
│       ├── deploy-uat.yml
│       └── deploy-prod.yml
├── README.md
├── INFRASTRUCTURE.md (this file)
├── requirements-dev.txt
├── requirements-prod.txt
│
├── bronze/                              # Phase 1: Data Ingestion
│   ├── README.md
│   ├── config/
│   │   ├── ingestion_config.yaml
│   │   └── routine_schemas.yaml (how to parse each routine)
│   ├── notebooks/
│   │   ├── 01_ingest_scans.py
│   │   ├── 02_ingest_calibrations.py
│   │   ├── 03_ingest_images.py
│   │   └── 04_validate_bronze.py
│   ├── src/
│   │   ├── __init__.py
│   │   ├── ingestion.py
│   │   ├── validation.py
│   │   ├── file_handlers.py
│   │   └── metadata_tracking.py
│   └── tests/
│       ├── test_ingestion.py
│       └── test_validation.py
│
├── silver/                              # Phase 2: Transformation
│   ├── README.md
│   ├── config/
│   │   ├── transformation_rules.yaml
│   │   └── dimension_mappings.yaml
│   ├── notebooks/
│   │   ├── 01_transform_calibrations.py
│   │   ├── 02_transform_devices.py
│   │   ├── 03_transform_shapes.py
│   │   ├── 04_transform_measurements.py
│   │   └── 05_quality_checks.py
│   ├── src/
│   │   ├── __init__.py
│   │   ├── transformations.py
│   │   ├── coordinate_transform.py
│   │   ├── quality_rules.py
│   │   └── dimension_builders.py
│   └── tests/
│       ├── test_transformations.py
│       └── test_quality_rules.py
│
├── gold/                                # Phase 3: Analytics
│   ├── README.md
│   ├── notebooks/
│   │   ├── 01_qc_reporting.py
│   │   ├── 02_measurement_trends.py
│   │   ├── 03_operator_performance.py
│   │   └── 04_ml_dataset_prep.py
│   ├── src/
│   │   └── analytics.py
│   └── tests/
│       └── test_analytics.py
│
├── infrastructure/                      # IaC (Infrastructure as Code)
│   ├── README.md
│   ├── terraform/
│   │   ├── main.tf
│   │   ├── storage.tf
│   │   ├── database.tf
│   │   ├── networking.tf
│   │   └── variables.tf
│   ├── bicep/                           # Alternative to Terraform
│   │   ├── main.bicep
│   │   ├── storage.bicep
│   │   └── parameters.json
│   └── scripts/
│       ├── deploy.sh
│       └── init-databricks.sh
│
├── pipelines/                           # ADF/Orchestration
│   ├── README.md
│   ├── templates/
│   │   ├── bronze_ingest_pipeline.json
│   │   ├── silver_transform_pipeline.json
│   │   └── gold_analytics_pipeline.json
│   └── linked_services/
│       ├── databricks_ls.json
│       ├── adls_ls.json
│       └── sql_db_ls.json
│
├── docs/
│   ├── ARCHITECTURE.md
│   ├── SET_UP_GUIDE.md
│   ├── BRANCHING_STRATEGY.md
│   ├── DATA_LINEAGE.md
│   └── DISASTER_RECOVERY.md
│
└── scripts/
    ├── local_dev_setup.sh
    ├── run_tests.sh
    └── deploy_to_env.sh
```

### **Repository Branching Strategy**

```
main (production)
  ↑
  └← release/* (UAT staging)
       ↑
       └← develop (dev integration)
            ↑
            ├← feature/bronze-ingestion
            ├← feature/silver-transform
            ├← feature/gold-analytics
            └← bugfix/issue-123
```

**Naming Conventions**:
- `feature/`: New functionality (e.g., `feature/calibration-scd2`)
- `bugfix/`: Bug fixes (e.g., `bugfix/coordinate-transform-precision`)
- `release/`: Release candidates (e.g., `release/v0.1.0`)
- `hotfix/`: Production fixes (e.g., `hotfix/ingestion-crash`)

---

## **4. PIPELINE ARCHITECTURE**

### **Orchestration Tool**: Azure Data Factory (ADF)

**Pipeline 1: Bronze Layer Ingestion**
```
Trigger: Timer (daily) or HTTP POST (on-demand)
  ↓
Activity 1: Wait for input files
  └─ Check ADLS for new scan ZIP files
  ↓
Activity 2: Execute Databricks notebook
  └─ 01_ingest_scans.py (parse YAML, store to Bronze)
  ↓
Activity 3: Execute Databricks notebook
  └─ 02_ingest_calibrations.py (deduplicate, track lineage)
  ↓
Activity 4: Execute Databricks notebook
  └─ 03_ingest_images.py (copy large files)
  ↓
Activity 5: Validate
  └─ 04_validate_bronze.py (schema, completeness checks)
  ↓
Activity 6: Update SQL metadata
  └─ Insert into ingestion_log table
  ↓
Activity 7: Notification
  └─ Email/Teams on success/failure
```

**Pipeline 2: Silver Layer Transformation**
```
Trigger: ADF dependency on Bronze completion (or scheduled nightly)
  ↓
Activity 1: Execute Databricks notebook
  └─ 01_transform_calibrations.py (build dim_calibration SCD2)
  ↓
Activity 2: Execute Databricks notebook
  └─ 02_transform_devices.py (build dim_device)
  ↓
Activity 3: Execute Databricks notebook
  └─ 03_transform_shapes.py (build dim_shapes)
  ↓
Activity 4: Execute Databricks notebook
  └─ 04_transform_measurements.py (fact_measurements, coordinate transform)
  ↓
Activity 5: Quality checks
  └─ 05_quality_checks.py (validation, anomaly detection)
  ↓
Activity 6: Update metadata
  └─ Insert QA metrics into metadata DB
  ↓
Activity 7: Notification
  └─ Email/Teams on issues
```

**Pipeline 3: Gold Layer Analytics**
```
Trigger: ADF dependency on Silver completion (or scheduled nightly)
  ↓
Activity 1: Execute Databricks notebook
  └─ 01_qc_reporting.py (build reporting tables)
  ↓
Activity 2: Execute Databricks notebook
  └─ 02_measurement_trends.py (time-series aggregates)
  ↓
Activity 3: Execute Databricks notebook
  └─ 03_operator_performance.py (operator KPIs)
  ↓
Activity 4: Execute Databricks notebook
  └─ 04_ml_dataset_prep.py (ML training data)
  ↓
Activity 5: Notification
  └─ Email/Teams: Gold layer ready for BI tools
```

---

## **5. CODE PROMOTION STRATEGY**

### **Environments**

| Environment | Purpose | Databricks Cluster | Database | Storage | Triggers |
|-------------|---------|-------------------|----------|---------|----------|
| **DEV** | Local development | Shared, small cluster | Dev SQL DB | Dev storage account | Manual runs |
| **UAT** | Testing before prod | Dedicated cluster | UAT SQL DB | UAT storage account | Nightly schedule |
| **PROD** | Live data platform | Prod cluster (autoscale) | Prod SQL DB | Prod storage account | Daily schedule + on-demand |

### **Deployment Pipeline (CI/CD)**

```
Developer pushes to feature/* branch
  ↓
GitHub Actions: ci-build.yml
  ├─ Run unit tests
  ├─ Lint / code quality checks
  ├─ Build Python wheels (if applicable)
  └─ Report results
  ↓
  [if tests pass]
  ↓
Create Pull Request → develop
  ├─ Code review (1+ approvals)
  ├─ Manual UAT testing
  └─ Merge to develop
  ↓
GitHub Actions: deploy-dev.yml
  ├─ Deploy notebooks to Dev Databricks
  ├─ Run integration tests
  └─ Notify team
  ↓
[After 1-2 week soak in Dev]
  ↓
Create Pull Request: develop → release/v*
  ├─ Changelog entry
  ├─ Version bump (semantic versioning)
  └─ Merge to release branch
  ↓
GitHub Actions: deploy-uat.yml
  ├─ Deploy to UAT Databricks
  ├─ Run full test suite
  ├─ Performance benchmarks
  └─ Approval gate
  ↓
[After 1-2 week soak in UAT]
  ↓
Manual Approval (Tech Lead / Architect)
  ↓
Merge release → main
  ↓
GitHub Actions: deploy-prod.yml
  ├─ Deploy to Production Databricks
  ├─ Run smoke tests
  ├─ Verify data pipelines
  └─ Announce release to team
```

### **Key Policies**

1. **Main branch is production**: Always deployable, no direct commits
2. **Code review required**: All PRs need 1+ approvals
3. **Automated tests mandatory**: Cannot merge without passing tests
4. **Semantic versioning**: v{major}.{minor}.{patch}
5. **Release notes required**: Document changes before merging to main

---

## **6. NETWORK & SECURITY**

### **Network Architecture**

```
Azure Landing Zone (already set up)
  ├─ Virtual Network (VNET)
  │   └─ Private Subnets
  │       ├─ Databricks Subnet
  │       ├─ Database Subnet
  │       └─ Integration Subnet (Functions, ADF)
  │
  └─ Private Endpoints (no public internet access)
      ├─ Storage Accounts (Bronze, Silver/Gold)
      ├─ SQL Database
      ├─ Key Vault (secrets)
      └─ Databricks (via private link)
```

### **Security**

| Component | Security | Notes |
|-----------|----------|-------|
| **Storage Accounts** | Service Endpoints + RBAC | No public access, only from private VNet |
| **Database** | Azure AD authentication + RBAC | Service principals for apps |
| **Databricks** | Token-based + AAD integration | Cluster access policy enforced |
| **Secrets** | Azure Key Vault | Connection strings, API keys stored here |
| **Encryption** | TLS in transit, AES-256 at rest | Managed by Azure |
| **Access Logging** | Storage lifecycle + audit logs | Track all data access |

---

## **7. MONITORING & ALERTING**

### **What to Monitor**

```
Azure Monitor / Application Insights
  ├─ Storage
  │   ├─ Ingestion volume (bytes/hour)
  │   ├─ File counts
  │   └─ Failed uploads
  │
  ├─ Databricks
  │   ├─ Job success/failure rate
  │   ├─ Duration (slow jobs alert)
  │   ├─ Cluster health (node failures)
  │   └─ Spark task failures
  │
  ├─ Database
  │   ├─ Query performance (slow queries)
  │   ├─ Disk usage (growth rate)
  │   ├─ Connection pool exhaustion
  │   └─ Backup success
  │
  ├─ Pipelines
  │   ├─ Pipeline run success rate
  │   ├─ Activity duration
  │   └─ Error logs
  │
  └─ Data Quality
      ├─ Completeness (% non-null fields)
      ├─ Anomalies (outlier measurements)
      ├─ Schema violations
      └─ Data freshness (latest scan age)
```

### **Alert Thresholds**

| Alert | Threshold | Action |
|-------|-----------|--------|
| Pipeline failure | Any | Notify team via Teams/Email |
| Ingestion latency | >2 hours | Check ADF logs |
| Data quality drop | <95% completeness | Investigate source data |
| Job failure rate | >5% | Escalate to platform team |
| Storage growth | >20% month-over-month | Review retention policies |

---

## **8. INFRASTRUCTURE SETUP CHECKLIST**

### **Phase 1: Foundation (Week 1-2)**

- [ ] **Azure Subscriptions & Resource Groups**
  - [ ] Create resource group `gelsight-data-rg-{env}`
  - [ ] Set up Azure Policy (enforce encryption, naming conventions)
  - [ ] Configure RBAC (service principals, roles)

- [ ] **Storage Accounts**
  - [ ] Create `gelsightbronze{env}` (Bronze layer)
  - [ ] Create `gelsightsilbergold{env}` (Silver/Gold layers)
  - [ ] Set up private endpoints (no public access)
  - [ ] Enable immutability policy on Bronze
  - [ ] Configure lifecycle policies (archive after 2 years)
  - [ ] Enable soft delete (30 days retention)
  - [ ] Configure Key Vault for access keys

- [ ] **Database Setup**
  - [ ] Create Azure SQL Database (metadata + dimensional model)
  - [ ] Configure private endpoint
  - [ ] Set up Azure AD authentication
  - [ ] Create logins for service principals
  - [ ] Run schema creation script (DDL)
  - [ ] Set up automated backups (daily, 7-day retention)

- [ ] **Databricks Configuration**
  - [ ] Verify cluster creation
  - [ ] Configure cluster auto-scaling policies
  - [ ] Set up DBFS mount points to storage accounts
  - [ ] Configure cluster access policies (who can run notebooks)
  - [ ] Install required libraries (pandas, pyspark, pytest, etc.)

- [ ] **Key Vault Setup**
  - [ ] Create Key Vault in same VNET
  - [ ] Configure access policies
  - [ ] Store secrets:
    - [ ] Storage account connection strings
    - [ ] SQL Database connection string
    - [ ] Databricks PAT tokens
    - [ ] API keys (if applicable)

### **Phase 2: Code & Pipelines (Week 2-3)**

- [ ] **GitHub/Azure DevOps Repository**
  - [ ] Create main repository `gelsight-data-platform`
  - [ ] Set up branch protection (main branch)
  - [ ] Configure branch policy (require reviews, status checks)
  - [ ] Create folder structure (bronze/, silver/, gold/, infrastructure/, pipelines/)

- [ ] **CI/CD Workflows**
  - [ ] Create GitHub Actions workflows:
    - [ ] `ci-build.yml` (unit tests, lint)
    - [ ] `deploy-dev.yml`
    - [ ] `deploy-uat.yml`
    - [ ] `deploy-prod.yml`
  - [ ] Configure secrets storage (org-level secrets for credentials)

- [ ] **Infrastructure as Code**
  - [ ] Write Terraform/Bicep for all infrastructure
  - [ ] Test IaC against test subscription
  - [ ] Document variable naming conventions

- [ ] **Azure Data Factory Setup**
  - [ ] Create ADF instance
  - [ ] Set up Linked Services:
    - [ ] Databricks
    - [ ] Storage accounts (Bronze, Silver, Gold)
    - [ ] SQL Database
  - [ ] Create pipelines (Ingest → Transform → Analyze)
  - [ ] Set up triggers (time-based or event-based)
  - [ ] Configure error handling and notifications

### **Phase 3: Testing & Validation (Week 3-4)**

- [ ] **Connectivity Tests**
  - [ ] Databricks → Storage (read/write tests)
  - [ ] Databricks → Database (query tests)
  - [ ] ADF → Databricks (job submission)
  - [ ] Key Vault → All services

- [ ] **Sample Data Ingestion**
  - [ ] Ingest 1-2 sample scans to Bronze
  - [ ] Verify metadata tracked in ingestion_log
  - [ ] Validate YAML parsing

- [ ] **Sample Transformation**
  - [ ] Transform sample data to Silver
  - [ ] Verify dimensional tables built
  - [ ] Verify fact tables populated

- [ ] **Pipeline End-to-End**
  - [ ] Run complete pipeline: Bronze → Silver → Gold
  - [ ] Verify all outputs created
  - [ ] Check ADF activity logs for errors

- [ ] **Documentation**
  - [ ] Document environment setup (README)
  - [ ] Create runbooks for common tasks (restart cluster, reprocess data, etc.)
  - [ ] Document emergency procedures (data loss recovery, etc.)

---

## **9. DEPLOYMENT & OPERATIONS**

### **Initial Deployment**

```
Step 1: Infrastructure Deployment
  $ cd infrastructure/
  $ terraform init -backend-config="dev.hcl"
  $ terraform plan -var-file="dev.tfvars"
  $ terraform apply -var-file="dev.tfvars"
  [Creates all Azure resources]

Step 2: Database Initialization
  $ sqlcmd -S {server} -d {database} -U {user} -P {password}
  > :r schema/00_init.sql
  > :r schema/01_dimensions.sql
  > :r schema/02_facts.sql
  > :r schema/03_metadata.sql

Step 3: Databricks Setup
  $ databricks workspace import-dir ./bronze {workspace_path}/bronze
  $ databricks workspace import-dir ./silver {workspace_path}/silver
  $ databricks workspace import-dir ./gold {workspace_path}/gold
  [Imports all notebooks]

Step 4: ADF Deployment
  $ az datafactory create --resource-group {rg} --name {adf}
  $ az deployment group create --resource-group {rg} \
      --template-file pipelines/ --parameters-file parameters.json
  [Creates pipelines, triggers, linked services]

Step 5: Validation
  $ python scripts/run_tests.sh
  [Runs unit + integration tests]
```

### **Operational Procedures**

**Daily Standup Check**:
- [ ] ADF pipelines completed successfully
- [ ] Data quality metrics within thresholds
- [ ] No failed Databricks jobs
- [ ] Storage growth within expected range

**Weekly Review**:
- [ ] Performance metrics (pipeline duration trends)
- [ ] Cost analysis (cluster usage, storage)
- [ ] Security audit (access logs)
- [ ] Backup verification

**Monthly Maintenance**:
- [ ] Database optimization (index rebuild)
- [ ] Archive old data (move Bronze to cold storage)
- [ ] Cost optimization (reserved instances, spot VMs)
- [ ] Disaster recovery test (restore from backup)

---

## **10. COST ESTIMATION**

| Component | Dev | UAT | Prod | Notes |
|-----------|-----|-----|------|-------|
| Databricks | $500/mo | $1,500/mo | $3,000/mo | Autoscaling clusters |
| Storage (ADLS2) | $200/mo | $500/mo | $1,500/mo | 100GB-1TB growth/month |
| SQL Database | $200/mo | $500/mo | $1,500/mo | Standard tier, auto-scaling |
| Data Factory | $100/mo | $200/mo | $500/mo | Per pipeline run |
| Key Vault | $20/mo | $20/mo | $20/mo | Fixed + per secret access |
| Monitoring | $50/mo | $100/mo | $200/mo | Ingestion, analysis |
| Bandwidth | $10/mo | $30/mo | $100/mo | Inter-region replication |
| **TOTAL** | **~$1,080** | **~$2,850** | **~$7,320** | /month |

**Optimization Opportunities**:
- Use reserved instances for Databricks
- Archive Bronze data to blob storage after 2 years
- Use spot instances for dev/UAT
- Implement data retention policies more aggressively

---

## **11. NEXT STEPS (Immediate)**

1. ✅ **Review this architecture** with stakeholders
2. ⏳ **Gather Azure subscription details** (subscription IDs, resource group names)
3. ⏳ **Create GitHub repository** with folder structure
4. ⏳ **Write Terraform code** for infrastructure (Week 1)
5. ⏳ **Set up databases & storage** (Week 1-2)
6. ⏳ **Configure CI/CD pipelines** (Week 2)
7. ⏳ **Build Bronze layer notebooks** (Phase 1 work)
8. ⏳ **Test end-to-end** (Week 3-4)

---

## **12. ARCHITECTURE DIAGRAM**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            AZURE LANDING ZONE                               │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                      VNET (Private Network)                          │  │
│  │                                                                      │  │
│  │  ┌──────────────┐    ┌─────────────────┐    ┌──────────────────┐   │  │
│  │  │  Databricks  │    │  SQL Database   │    │   Key Vault      │   │  │
│  │  │  Workspace   │◄──►│  (Metadata)     │←──►│  (Secrets)       │   │  │
│  │  │  (Compute)   │    │                 │    │                  │   │  │
│  │  └──────────────┘    └─────────────────┘    └──────────────────┘   │  │
│  │         ▲ │                                                         │  │
│  │         │ │         ┌────────────────────────────────────┐        │  │
│  │         │ └────────►│  Azure Data Factory (Orchestration)│        │  │
│  │         │           └────────────────────────────────────┘        │  │
│  └─────────┼──────────────────────────────────────────────────────────┘  │
│            │                                                             │
│  ┌─────────▼────────────────────────────────────────────────────────────┐ │
│  │                    STORAGE (Private Endpoints)                        │ │
│  │                                                                       │ │
│  │  ┌──────────────────────┐      ┌──────────────────────────────────┐ │ │
│  │  │ Bronze Layer Storage │      │ Silver/Gold Layer Storage        │ │ │
│  │  │ (Immutable)          │      │ (Parquet, Optimized)            │ │ │
│  │  │ gelsightbronze{env}  │      │ gelsightsilbergold{env}         │ │ │
│  │  │                      │      │                                  │ │ │
│  │  │ /scans/              │      │ /dim_calibration/               │ │ │
│  │  │ /calibrations/       │───────────► /dim_device/               │ │ │
│  │  │ /archive/            │      │ /fact_measurements/            │ │ │
│  │  └──────────────────────┘      │ /qc_reporting/                  │ │ │
│  │                                └──────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

EXTERNAL (Public GitHub)
  ↓
GitHub Repository (gelsight-data-platform)
  ├─ CI/CD workflows (GitHub Actions)
  ├─ Infrastructure as Code (Terraform/Bicep)
  ├─ Notebooks (Bronze, Silver, Gold)
  └─ Tests & Documentation
```

---

**Document Version**: 1.0  
**Next Review**: After infrastructure deployment (Week 4)  
**Owner**: Cloud Architect / DevOps Lead
