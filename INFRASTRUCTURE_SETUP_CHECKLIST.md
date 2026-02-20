# GelSight Data Platform – Infrastructure Setup Checklist

**Purpose**: Tactical action items for setting up DEV infrastructure and proving out the data pipeline.

**Date**: February 12, 2026  
**Status**: BLOCK 1-5 Complete  
**Current Progress**: 
- ✅ BLOCK 1: Databricks workspace verified (USCU-PROD-DATA-PROCESSING-01, Active)
- ✅ BLOCK 2: Bronze storage created (uscubronzedev, Standard_GRS)
- ✅ BLOCK 2: Silver/Gold storage created (uscusilvergolddev, Premium_LRS)
- ✅ BLOCK 3.1: SQL Server + Database created (uscu-sql-dev, uscu_metadata, Local backup)
- ✅ BLOCK 3.2: DDL schemas/tables executed (7 tables created)
- ✅ BLOCK 4: Key Vault secrets stored (3 secrets in gelsightprod-kv01)
- ✅ BLOCK 5: Databricks clusters & Unity Catalog (external locations, managed identity) COMPLETE
- ⏳ BLOCK 6-7: Pending

**Next Immediate Actions**:
1. Create Databricks dev cluster (uscu-dev)
2. Configure Unity Catalog with external locations for Bronze/Silver/Gold using managed identity
3. BLOCK 6: Unity Catalog schema/table setup
4. BLOCK 7: Manual ingestion & validation

**Target**: Complete DEV environment (Weeks 1-2), Decide on UAT/PROD workspace strategy (Week 2)  
**Approach**: Manual Azure CLI setup for DEV, then convert to Terraform for reproducible uat/prod scaling

---

## **QUICK REFERENCE: WHAT WE'VE DECIDED**

| Component | Decision | Why |
|-----------|----------|-----|
| **Storage Layer** | Bronze = Standard_GRS, Silver/Gold = Premium_LRS consolidated | Resilience + cost optimization for each layer |
| **SQL Backup** | Local redundancy (not Geo) | Dev doesn't need multi-region backups; can upgrade to Geo for uat/prod |
| **Landing Folder** | Upload → `scans/incoming/`, validate in-place, organize later | Avoid double file movement, simpler ingestion |
| **Metadata DB** | Single `uscu_metadata` database (not bronze/silver/gold separate) | Metadata is environment-agnostic, schemas organize by type |
| **Fact Table: Scans** | `fact_scan_metadata` = Bronze inventory only (filename, blob path, upload time) | Simple tracking, not a pipeline journal |
| **Fact Table: Measurements** | `fact_measurements` = Extracted measurements from transformation | One row per measurement per scan (normalized) |
| **Audit Trail** | `meta.ingestion_log` per transformation job (not per file) | Cleaner querying, reduces row explosion |
| **Calibration History** | `dim_calibration` with SCD Type 2 (effective_date, end_date, is_current) | Track parameter evolution over time |

---

---

## **KEY ARCHITECTURAL DECISIONS**

**Storage Layer**:
- Bronze = pure ingestion (raw files as-is, no transformation logic)
- Single Silver/Gold storage account (consolidated for simplicity, can separate later if needed)
- Landing → incoming/ (upload directly to blob, validate then process, avoid double-movement)

**Metadata Layer**:
- Single SQL database (`uscu_metadata`) for all environments
- Three schemas: `dim` (dimensions), `fact` (measurements), `meta` (audit)
- `fact_scan_metadata`: Bronze intake tracking only (filename, path, upload time) — NOT a transformation pipeline journal
- Rationale: Simpler schema, no status churn, audit trail in `ingestion_log` only

**Landing Zone Alignment**:
- Working in existing USCU-PROD-DBW-RG (respecting landing zone resource group)
- Using uscu-* naming convention (aligned with enterprise standard)
- **Pre-UAT note**: Check with landing zone team about RBAC/VNet/tagging requirements before scaling

---

## **PHASE 1: DEV FOUNDATION (Weeks 1-2)**

### **BLOCK 1: Azure Subscriptions & Resource Groups**

**Owner**: Cloud Administrator  
**Timeline**: 30 minutes

**Note**: We'll work in existing USCU-PROD-DBW-RG. Differentiate environments by resource naming (dev/uat/prod suffixes).

#### Task 1.1: Verify Existing Resources
- [x] List available Azure subscriptions
  ```powershell
  az account list --output table
  ```
- [x] Verify existing resource group
  ```powershell
  az group show --name USCU-PROD-DBW-RG --query "{Name:name, Location:location}"
  ```
- [x] Verify Databricks workspace exists
  ```powershell
  az databricks workspace show --name "USCU-PROD-DATA-PROCESSING-01" --resource-group USCU-PROD-DBW-RG
  ```
  **Status**: ✅ Completed Feb 11. Workspace USCU-PROD-DATA-PROCESSING-01 confirmed as Active/Hybrid in centralus

#### Task 1.2: Note for Later
- [ ] **UAT/PROD workspace decision**: After dev is working, decide if you want:
  - Separate Databricks workspaces for uat/prod, OR
  - Single workspace with separate clusters per environment

---

### **BLOCK 2: Azure Storage Accounts (DEV)**

**Owner**: Cloud Infrastructure Engineer  
**Timeline**: 2-3 hours  
**Status**: ✅ COMPLETED Feb 11  

**Decisions Made**:
- Bronze uses Standard_GRS (geo-redundancy: centralus + eastus2) for resilience, lower cost than Premium
- Silver/Gold consolidated into single Premium_LRS account (simpler management, lower egress costs, can split later if isolation needed)
- Landing → incoming/ folder approach: Upload directly to `uscubronzedev/scans/incoming/`, validate in place, then organize by date after transform complete

#### Task 2.1: Create Bronze Layer Storage

**Status**: ✅ COMPLETED Feb 11

```powershell
$resourceGroup = "USCU-PROD-DBW-RG"
$storageAccount = "uscubronzedev"
$region = "centralus"

# Create storage account (Standard_GRS for cost + redundancy)
az storage account create \
  --name $storageAccount \
  --resource-group $resourceGroup \
  --location $region \
  --sku Standard_GRS \
  --min-tls-version TLS1_2 \
  --https-only true

# Create containers
az storage container create --account-name $storageAccount --name scans
az storage container create --account-name $storageAccount --name calibrations
az storage container create --account-name $storageAccount --name archive
```

**Details**: All containers created successfully. No soft-delete/versioning enabled at this time (can add post-validation).

- [ ] Enable soft delete (30 days) — Deferred to post-MVP validation
  ```powershell
  az storage account blob-service-properties update \
    --account-name $storageAccount \
    --resource-group $resourceGroup \
    --enable-delete-retention true \
    --delete-retention-days 30
  ```

- [ ] Enable versioning — Deferred to post-MVP validation
  ```powershell
  az storage account blob-service-properties update \
    --account-name $storageAccount \
    --resource-group $resourceGroup \
    --enable-versioning true
  ```

#### Task 2.2: Create Silver/Gold Layer Storage

**Status**: ✅ COMPLETED Feb 11

```powershell
$storageAccount = "uscusilvergolddev"

# Create storage account (Premium_LRS for transformed/analytical data)
az storage account create \
  --name $storageAccount \
  --resource-group $resourceGroup \
  --location $region \
  --sku Premium_LRS \
  --min-tls-version TLS1_2 \
  --https-only true

# Silver containers (normalized, structured data from transformation)
az storage container create --account-name $storageAccount --name dim-calibration
az storage container create --account-name $storageAccount --name dim-device
az storage container create --account-name $storageAccount --name dim-shapes
az storage container create --account-name $storageAccount --name dim-routines
az storage container create --account-name $storageAccount --name fact-measurements
az storage container create --account-name $storageAccount --name fact-scan-metadata

# Gold containers (reporting, aggregations, business-ready outputs)
az storage container create --account-name $storageAccount --name qc-reporting
az storage container create --account-name $storageAccount --name measurement-trends
az storage container create --account-name $storageAccount --name operator-performance
```

**Decision**: Single consolidated account for simplicity. Powers = 9 containers represent what goes OUT of Databricks (not IN). Rationale: Consolidated = cheaper egress, simpler mount management, can split into separate accounts post-validation if isolation/cost analysis requires it.

- [x] Verify both storage accounts created

---

### **BLOCK 3: Azure SQL Database (DEV)**

**Owner**: Database Administrator  
**Timeline**: 2-3 hours  
**Status**: 🟡 IN PROGRESS (provider registration required)

**Decisions Made**:
- Single database `uscu_metadata` for all environments (not separate bronze/silver/gold databases)
- Rationale: Metadata is environment-agnostic; schemas (dim/fact/meta) organize by data type, not layer
- `fact_scan_metadata`: Tracks Bronze intake only (filename, blob path, upload time)
  - NOT a transformation pipeline journal (too chatty, harder to query)
  - Transformation audit in `meta.ingestion_log` instead
- Three-schema structure for clarity: `dim` (dimensions), `fact` (measurements), `meta` (operations)

#### Task 3.1: Register SQL Provider & Create SQL Server

**Status**: ✅ COMPLETED Feb 11

**Provider Registration**: (Skipped manually, provider was already enabled)

```powershell
# Step 2: Create SQL Server ✅
$resourceGroup = "USCU-PROD-DBW-RG"
$region = "centralus"
$sqlServerName = "uscu-sql-dev"
$databaseName = "uscu_metadata"
$adminLogin = "sqladmin"

az sql server create \
  --name $sqlServerName \
  --resource-group $resourceGroup \
  --location $region \
  --admin-user $adminLogin \
  --admin-password "ComplexP@ssw0rd2024!" \
  --minimal-tls-version 1.2

# Step 3: Create database ✅
az sql db create \
  --server $sqlServerName \
  --name $databaseName \
  --resource-group $resourceGroup \
  --service-objective S0

# Step 4: Allow Azure services to connect ✅
az sql server firewall-rule create \
  --server $sqlServerName \
  --resource-group $resourceGroup \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Step 5: Downgrade backup redundancy (Local instead of Geo) ✅
# Decision: Dev doesn't need Geo-redundant backups; switching to Local saves ~50% backup cost
az sql db update \
  --name $databaseName \
  --server $sqlServerName \
  --resource-group $resourceGroup \
  --backup-storage-redundancy Local
```

**Results**:
- Server: uscu-sql-dev (Ready, TLS1.2, public network enabled for dev)
- Database: uscu_metadata (Online, Standard S0 tier, 250GB max)
- Backup: Local redundancy (cost-optimized for dev MVP)
- Firewall: AllowAzureServices enabled (Databricks + Key Vault)

**Notes**: 
- Removed invalid `--accessed-tier Hot` parameter
- Removed `--enable-public-network true` (implicitly enabled)
- Service Objective S0 = single database Standard tier (sufficient for metadata workloads)
- AllowAzureServices rule allows Databricks + Logic Apps + Key Vault to authenticate
- **Backup redundancy**: Started as Geo, downgraded to Local for cost savings in dev (can upgrade to Geo for uat/prod if needed)

#### Task 3.2: Create Database Schemas & Tables

**Status**: ⏳ PENDING (execute DDL scripts after SQL Server available)

**METADATA SCHEMA DESIGN** (Single Database, Three Schemas):
- `dim`: Dimension tables (calibration, device, shapes, routines)
- `fact`: Fact tables (measurements, scan_metadata)
- `meta`: Operational tables (ingestion_log)

**Why this structure?**
- `fact_scan_metadata`: Bronze file inventory (filename, path, upload time) — **one row per uploaded file**
- `fact_measurements`: Extracted measurements from transformation — **one row per measurement per scan**
- `dim_calibration`: SCD Type 2 (tracks calibration parameter history over time)
- `meta.ingestion_log`: Pipeline execution audit — **one row per transformation job** (not per file)

**SQL DDL** (execute via Azure Data Studio or sqlcmd after SQL Server/database created):

```sql
-- Execute in Azure Data Studio, connected to uscu_metadata database

-- PHASE 1: Create schemas
CREATE SCHEMA dim;
CREATE SCHEMA fact;
CREATE SCHEMA meta;
GO

-- PHASE 2: Create dimension tables

CREATE TABLE dim.dim_calibration (
    calibration_id INT PRIMARY KEY IDENTITY(1,1),
    calibration_key VARCHAR(255) NOT NULL UNIQUE,  -- e.g. "Calib-4E07-XNJU_20250917_1255"
    gelsight_id VARCHAR(50) NOT NULL,               -- e.g. "4E07-XNJU"
    gelsight_model VARCHAR(100),                    -- e.g. "MB101"
    magnification FLOAT,
    focal_length_mm FLOAT,
    mmperpixel FLOAT,
    effective_date DATE NOT NULL DEFAULT CAST(GETDATE() AS DATE),
    end_date DATE DEFAULT '9999-12-31',
    is_current BIT DEFAULT 1,
    created_at DATETIME DEFAULT GETDATE(),
    updated_at DATETIME DEFAULT GETDATE()
);
CREATE INDEX idx_dim_calib_key ON dim.dim_calibration(calibration_key);
CREATE INDEX idx_dim_calib_current ON dim.dim_calibration(gelsight_id, is_current);
GO

CREATE TABLE dim.dim_device (
    device_id INT PRIMARY KEY IDENTITY(1,1),
    device_key VARCHAR(100) NOT NULL UNIQUE,       -- e.g. "00020015"
    device_type VARCHAR(50),                        -- e.g. "Modulus"
    location VARCHAR(100),
    serial_number VARCHAR(100),
    created_at DATETIME DEFAULT GETDATE()
);
GO

CREATE TABLE dim.dim_shapes (
    shape_id INT PRIMARY KEY IDENTITY(1,1),
    shape_key VARCHAR(100) NOT NULL UNIQUE,        -- e.g. "hole_diameter"
    shape_name VARCHAR(100),
    shape_type VARCHAR(50),                         -- e.g. "measurement", "defect", "roughness"
    unit_of_measure VARCHAR(20),                    -- e.g. "mm", "µm", "count"
    created_at DATETIME DEFAULT GETDATE()
);
GO

CREATE TABLE dim.dim_routines (
    routine_id INT PRIMARY KEY IDENTITY(1,1),
    routine_key VARCHAR(100) NOT NULL UNIQUE,      -- e.g. "hole_detection_v2"
    routine_name VARCHAR(100),
    routine_description VARCHAR(500),
    version VARCHAR(20),
    created_at DATETIME DEFAULT GETDATE()
);
GO

-- PHASE 3: Create fact tables

CREATE TABLE fact.fact_scan_metadata (
    scan_id BIGINT PRIMARY KEY IDENTITY(1,1),
    filename VARCHAR(500) NOT NULL,                -- e.g. "G500-08b-repeat-001.tmd"
    blob_path VARCHAR(1000) NOT NULL,              -- Full blob URI in uscubronzedev
    file_size_bytes BIGINT,
    upload_timestamp DATETIME NOT NULL,
    uploaded_by VARCHAR(100),
    scan_guid VARCHAR(100),                        -- GUID from scan.yaml
    created_at DATETIME DEFAULT GETDATE()
);
CREATE INDEX idx_fact_scan_meta_filename ON fact.fact_scan_metadata(filename);
CREATE INDEX idx_fact_scan_meta_upload_ts ON fact.fact_scan_metadata(upload_timestamp DESC);
GO

CREATE TABLE fact.fact_measurements (
    measurement_id BIGINT PRIMARY KEY IDENTITY(1,1),
    scan_id BIGINT NOT NULL FOREIGN KEY REFERENCES fact.fact_scan_metadata(scan_id),
    device_id INT NOT NULL FOREIGN KEY REFERENCES dim.dim_device(device_id),
    calibration_id INT NOT NULL FOREIGN KEY REFERENCES dim.dim_calibration(calibration_id),
    shape_id INT NOT NULL FOREIGN KEY REFERENCES dim.dim_shapes(shape_id),
    routine_id INT FOREIGN KEY REFERENCES dim.dim_routines(routine_id),
    measurement_value FLOAT NOT NULL,
    measurement_unit VARCHAR(20),                   -- e.g. "mm", "µm", "count"
    measurement_timestamp DATETIME NOT NULL,
    quality_flag VARCHAR(20),                       -- e.g. "good", "warning", "error"
    measurement_notes VARCHAR(500),
    created_at DATETIME DEFAULT GETDATE()
);
CREATE INDEX idx_fact_measurements_scan ON fact.fact_measurements(scan_id);
CREATE INDEX idx_fact_measurements_device ON fact.fact_measurements(device_id);
CREATE INDEX idx_fact_measurements_shape ON fact.fact_measurements(shape_id);
GO

-- PHASE 4: Create metadata/audit table

CREATE TABLE meta.ingestion_log (
    log_id BIGINT PRIMARY KEY IDENTITY(1,1),
    source_path VARCHAR(500),                       -- e.g. "uscubronzedev/scans/incoming"
    target_schema VARCHAR(100),                     -- e.g. "fact", "dim"
    job_name VARCHAR(100),                          -- e.g. "bronze_to_silver_transform"
    ingestion_start_time DATETIME NOT NULL,
    ingestion_end_time DATETIME,
    status VARCHAR(20) NOT NULL,                    -- e.g. "running", "success", "failed"
    rows_processed INT,
    rows_inserted INT,
    rows_failed INT,
    error_message VARCHAR(2000),
    executed_by VARCHAR(100),
    created_at DATETIME DEFAULT GETDATE()
);
CREATE INDEX idx_ingestion_log_status ON meta.ingestion_log(status, ingestion_end_time DESC);
GO
```

- [x] Wait for SQL Server provisioning to complete
- [x] Connect with Azure Data Studio
- [x] Run DDL script (all 4 phases)
- [x] Verify tables created: `SELECT * FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA IN ('dim', 'fact', 'meta')`

**Service Principal Permissions** (DEFERRED to UAT/PROD):
The section below is for later when using Databricks/ADF managed identities. Skip for dev MVP:
```sql
-- NOT NEEDED FOR DEV (skip this block)
-- Uncomment only when wiring up Databricks service principal for uat/prod
-- CREATE USER [gelsight-databricks-principal] FROM EXTERNAL PROVIDER;
-- ALTER ROLE db_datawriter ADD MEMBER [gelsight-databricks-principal];
-- CREATE USER [gelsight-adf-principal] FROM EXTERNAL PROVIDER;
-- ALTER ROLE db_datawriter ADD MEMBER [gelsight-adf-principal];
```

**LANDING ZONE ALIGNMENT NOTE**:
Before UAT/PROD, confirm with landing zone team:
- [ ] RBAC policy: Does SQL Server need Service Principal auth instead of basic SQL auth?
- [ ] VNet requirement: Should SQL Server be in a specific VNet/subnet?
- [ ] Tagging: Are resource tags required (cost center, owner, etc.)?
- [ ] Encryption: Do we need encryption-at-rest beyond TLS-in-transit?

---

### **BLOCK 4: Azure Key Vault**

**Owner**: Security / Cloud Administrator  
**Timeline**: 1-2 days

#### Task 4.1: Create Key Vault
```powershell
$resourceGroup = "USCU-PROD-DBW-RG"
$region = "centralus"
$keyVaultName = "uscu-kv-dev"

az keyvault create \
  --name $keyVaultName \
  --resource-group $resourceGroup \
  --location $region \
  --sku standard
```

- [ ] Verify Key Vault created

#### Task 4.2: Store Secrets
```powershell
# Storage account Bronze connection string
az keyvault secret set \
  --vault-name $keyVaultName \
  --name "storage-bronze-connstr" \
  --value $(az storage account show-connection-string --name uscubronzedev --resource-group $resourceGroup -o tsv)

# Storage account Silver/Gold connection string
az keyvault secret set \
  --vault-name $keyVaultName \
  --name "storage-silvergold-connstr" \
  --value $(az storage account show-connection-string --name uscusilvergolddev --resource-group $resourceGroup -o tsv)

# SQL Database connection string
az keyvault secret set \
  --vault-name $keyVaultName \
  --name "sql-db-connstr" \
  --value "Server=tcp:uscu-sql-dev.database.windows.net,1433;Initial Catalog=uscu_metadata;Encrypt=true;TrustServerCertificate=false;Connection Timeout=30;"
```

- [ ] Verify secrets created
  ```powershell
  az keyvault secret list --vault-name $keyVaultName
  ```

#### Task 4.3: Access Policies
```powershell
# Grant self (current user) access for administration
az keyvault set-policy \
  --name $keyVaultName \
  --object-id $(az ad signed-in-user show --query objectId -o tsv) \
  --secret-permissions get list set delete
```

- [ ] Verify access policies set

---

### **BLOCK 5: Databricks Clusters & Unity Catalog Setup**

-**UPDATE (Feb 13, 2026):**
-**Commands Used to Create External Locations:**

```sql
-- Create external locations for complete medallion architecture
-- All use the same storage credential with RBAC via managed identity

-- Bronze layer (already exists, but using IF NOT EXISTS for safety)
CREATE EXTERNAL LOCATION IF NOT EXISTS uscu_bronze
  URL 'abfss://customer01@gelsightprodstnd01.dfs.core.windows.net/bronze'
  WITH (STORAGE CREDENTIAL `uscu-storage-cred`)
  COMMENT 'Customer01 bronze layer - raw data';

-- Silver layer
CREATE EXTERNAL LOCATION IF NOT EXISTS uscu_silver
  URL 'abfss://customer01@gelsightprodstnd01.dfs.core.windows.net/silver'
  WITH (STORAGE CREDENTIAL `uscu-storage-cred`)
  COMMENT 'Customer01 silver layer - cleaned and validated data';

-- Gold layer
CREATE EXTERNAL LOCATION IF NOT EXISTS uscu_gold
  URL 'abfss://customer01@gelsightprodstnd01.dfs.core.windows.net/gold'
  WITH (STORAGE CREDENTIAL `uscu-storage-cred`)
  COMMENT 'Customer01 gold layer - business-level aggregates';

-- Grant permissions to yourself
GRANT READ FILES, WRITE FILES ON EXTERNAL LOCATION uscu_bronze TO `amorgan@netrixllc.com`;
GRANT READ FILES, WRITE FILES ON EXTERNAL LOCATION uscu_silver TO `amorgan@netrixllc.com`;
GRANT READ FILES, WRITE FILES ON EXTERNAL LOCATION uscu_gold TO `amorgan@netrixllc.com`;

-- Show all external locations
SHOW EXTERNAL LOCATIONS;
```
- Unity Catalog is now fully configured for secure, multi-tenant storage access.
- All storage access uses Azure Managed Identity via the `uscu-storage-cred` credential (no access keys in Spark config).
- Three external locations created in Unity Catalog:
  - `uscu_bronze` → abfss://customer01/bronze@gelsightprodstnd01.dfs.core.windows.net/
  - `uscu_silver` → abfss://customer01/silver@gelsightprodstnd01.dfs.core.windows.net/
  - `uscu_gold` → abfss://customer01/gold@gelsightprodstnd01.dfs.core.windows.net/
- All locations use the same managed identity credential for RBAC.
- No DBFS mounts or storage keys are used—access is managed by Unity Catalog and Azure RBAC.

**Owner**: Databricks Administrator  
**Timeline**: 2-3 hours  
**Status**: 🟡 IN PROGRESS (UI-based setup)

**Architecture Decision**: Use Unity Catalog + External Locations (not DBFS mounts).
- Rationale: UC is modern best practice, provides audit trail, better security posture
- No more deprecated DBFS mounts
- References existing storage containers (uscubronzedev, uscusilvergolddev)

#### Task 5.1: Use Existing Cluster
✅ **COMPLETED**: Using existing cluster `CLU-SCAN-1` (already running, fully configured)
- Cluster: CLU-SCAN-1
- Status: Running
- Runtime: 16.4
- Cores: 8

#### Task 5.2: Create Storage Credential
✅ **COMPLETED**: Storage credential `uscu-storage-cred` created successfully

Configuration used:
- Name: `uscu-storage-cred`
- Credential Type: Azure Managed Identity
- Access Connector ID: `/subscriptions/995bc675-2cd3-4186-a731-195bdc2bc722/resourceGroups/USCU-PROD-DBWMANAGED-RG/providers/Microsoft.Databricks/accessConnectors/unity-catalog-access-connector`
- Status: Ready for use with both Bronze and Silver/Gold storage accounts

#### Task 5.3: Create External Locations
1. Left sidebar → **Catalog** → **External Locations**
2. Click **Create External Location**
3. Create three external locations using **gelsightprodstnd01/customer01** with subfolder structure:

   **Location 1 - Bronze**:
   - Name: `uscu_bronze`
   - URL: `abfss://customer01/bronze@gelsightprodstnd01.dfs.core.windows.net/`
   - Storage Credential: `uscu-storage-cred`
   - Click **Create**

   **Location 2 - Silver**:
   - Name: `uscu_silver`
   - URL: `abfss://customer01/silver@gelsightprodstnd01.dfs.core.windows.net/`
   - Storage Credential: `uscu-storage-cred`
   - Click **Create**

   **Location 3 - Gold**:
   - Name: `uscu_gold`
   - URL: `abfss://customer01/gold@gelsightprodstnd01.dfs.core.windows.net/`
   - Storage Credential: `uscu-storage-cred`
   - Click **Create**

- ✅ External location `uscu_bronze` created
- ✅ External location `uscu_silver` created
- ✅ External location `uscu_gold` created

#### Task 5.4: Install Libraries
In a notebook cell on `CLU-SCAN-1` cluster, run:

```python
# Install libraries for data processing
%pip install pyyaml pydantic sqlalchemy pyodbc
```

- [ ] Libraries installed successfully

**Note**: The three external locations reference `gelsightprodstnd01/customer01` with subfolder structure:
- Bronze data goes to: `customer01/bronze/`
- Silver data goes to: `customer01/silver/`
- Gold data goes to: `customer01/gold/`

---

### **BLOCK 6: Unity Catalog Catalogs & Schemas**

**Owner**: Databricks Data Engineer  
**Timeline**: 1-2 hours  
**Status**: ⏳ PENDING (execute after BLOCK 5 complete)

**Architecture**: Create a Unity Catalog catalog (customer01_data) with a managed storage location, and create schemas for bronze, silver, and gold layers. Use distinct names for catalogs and external locations to avoid conflicts.

#### Task 6.1: Create Bronze Catalog & Schemas


**Commands Used to Create Catalog and Schemas:**

```sql
-- Create a catalog for customer01 data with managed storage location
CREATE CATALOG IF NOT EXISTS customer01_data
  MANAGED LOCATION 'abfss://unity-catalog-storage@dbstoragegp4ic7cjhmi4x4.dfs.core.windows.net/7405617301776963/customer01_data'
  COMMENT 'Customer01 data catalog';

-- Create schemas for medallion architecture
CREATE SCHEMA IF NOT EXISTS customer01_data.bronze
  COMMENT 'Bronze layer - raw data';

CREATE SCHEMA IF NOT EXISTS customer01_data.silver
  COMMENT 'Silver layer - cleaned and validated data';

CREATE SCHEMA IF NOT EXISTS customer01_data.gold
  COMMENT 'Gold layer - business-level aggregates';

-- Verify
SHOW CATALOGS;
SHOW SCHEMAS IN customer01_data;
```

- ✅ Catalog `customer01_data` created with managed storage location
- ✅ Schemas: bronze, silver, gold created in catalog


#### Task 6.2: (Removed)
**Note:** With the new architecture, all medallion layers (bronze, silver, gold) are now schemas within the single catalog `customer01_data`. There is no need to create separate catalogs for silver and gold. All data organization and access is managed through schemas in `customer01_data`.

### **BLOCK 7: Manual Bronze Data Ingestion**

**Owner**: Data Engineer  
**Timeline**: 2-4 hours (depending on data volume)  
**Status**: ⏳ PENDING (execute after BLOCK 6 complete)

#### Task 7.1: Upload Sample Data to Bronze Storage

From your local machine, upload sample scan files from GelSightAnalysis folder to **gelsightprodstnd01/customer01/bronze**:

```powershell
# Define paths
$sourceFolder = "c:\Projects\Clients\GelSight\Gelsight Application Folder\GelSightAnalysis\DefectDetection"
$storageAccount = "gelsightprodstnd01"
$container = "customer01"
$resourceGroup = "USCU-PROD-INFRA-RG"

# Upload all files from DefectDetection to bronze subfolder
az storage blob upload-batch `
  --account-name $storageAccount `
  --destination $container/bronze/scans `
  --source $sourceFolder `
  --pattern "*"
```

Verify upload:

```powershell
az storage blob list --account-name gelsightprodstnd01 --container-name customer01 --output table
```

- [ ] Sample files uploaded to gelsightprodstnd01/customer01/bronze/scans
- [ ] Verify file count > 0

#### Task 7.2: Create Bronze Ingestion Notebook

In Databricks, create notebook `bronze_ingest_sample`:

```python
# Notebook: bronze_ingest_sample

from pyspark.sql.types import StructType, StructField, StringType, TimestampType, LongType
from datetime import datetime
import os

# Configuration
bronze_root = "abfss://scans@uscubronzedev.dfs.core.windows.net"
catalog = "uscu_bronze"
schema = "scans"

# List files in Bronze storage
files = dbutils.fs.ls(bronze_root)
print(f"Found {len(files)} files in Bronze storage")

# Filter for scan.yaml metadata files
scan_files = [f for f in files if f.name.endswith('scan.yaml')]
print(f"Scan metadata files: {len(scan_files)}")

# Create table schema for raw scan metadata
schema_def = """
    file_path STRING,
    file_name STRING,
    file_size_bytes LONG,
    scan_timestamp TIMESTAMP,
    gelsight_id STRING,
    ingested_at TIMESTAMP,
    raw_content STRING
"""

# Create Delta table for scan metadata
spark.sql(f"""
    CREATE TABLE IF NOT EXISTS {catalog}.{schema}.scan_files (
        file_path STRING,
        file_name STRING,
        file_size_bytes LONG,
        scan_timestamp TIMESTAMP,
        gelsight_id STRING,
        ingested_at TIMESTAMP,
        raw_content STRING
    )
    USING DELTA
    LOCATION 'abfss://customer01/bronze@gelsightprodstnd01.dfs.core.windows.net/scans'
""")

print(f"✅ Table {catalog}.{schema}.scan_files created")

# Log ingestion event
log_entry = {
    "job_name": "bronze_ingest_sample",
    "ingestion_start_time": datetime.now(),
    "file_count": len(files),
    "scan_file_count": len(scan_files),
    "status": "completed"
}

print(f"Ingestion log: {log_entry}")
```

- [ ] Create notebook `bronze_ingest_sample`
- [ ] Assign cluster `uscu-dev`
- [ ] Run notebook, verify table created

#### Task 7.3: Verify Data in UC Tables

In a Databricks SQL cell, run:

```sql
-- Check if table was created
SHOW TABLES IN uscu_bronze.scans;

-- Query the table
SELECT * FROM uscu_bronze.scans.scan_files LIMIT 10;

-- Check row count
SELECT COUNT(*) as total_rows FROM uscu_bronze.scans.scan_files;
```

- [ ] Table `uscu_bronze.scans.scan_files` contains data
- [ ] Row count should match uploaded file count
- [ ] Sample data is readable and properly structured

---

## **PHASE 3: DECISION POINT & FUTURE WORK**

### **Completed (DEV only)**
✅ Storage (Bronze, Silver/Gold)
✅ SQL Metadata Database
✅ Key Vault
✅ Databricks workspace & clusters
✅ Unity Catalog setup
✅ Manual bronze data ingestion

### **Next Decisions**
- [ ] **Workspace Strategy**: Single workspace for all envs? Or separate workspaces for dev/uat/prod?
- [ ] **Codification**: Ready to convert dev to Terraform for uat/prod replication?
- [ ] **CI/CD**: Set up Azure DevOps for automated deployments?

### **Deferred - For Later (After Dev Proven)**
⏳ BLOCK 8: Terraform Infrastructure as Code (convert dev to IaC, replicate to uat/prod)
⏳ BLOCK 9: Azure Data Factory pipelines (orchestrate bronze→silver→gold)
⏳ BLOCKS 10-12: Full CI/CD, testing, documentation

---

## **CONNECTIVITY & VALIDATION TESTS (DEV)**

**Owner**: Data Engineer / QA  
**Timeline**: 1-2 hours

### Connectivity Checklist

**Owner**: QA / Test Engineer  
**Timeline**: 2-3 days

#### Task 10.1: Storage Connectivity
```python
# Notebook: test_storage_connectivity.py

from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("StorageTest").getOrCreate()

# Test Bronze mount
try:
    df_bronze = spark.read.format("parquet").load("/mnt/bronze/scans")
    print("✓ Bronze mount accessible")
except Exception as e:
    print(f"✗ Bronze mount failed: {e}")

# Test Silver mount
try:
    df_silver = spark.read.format("parquet").load("/mnt/silver/dim-calibration")
    print("✓ Silver mount accessible")
except Exception as e:
    print(f"✗ Silver mount failed: {e}")

# Test write (small test file)
try:
    test_df = spark.createDataFrame([("test", 1)], ["col1", "col2"])
    test_df.write.format("parquet").mode("overwrite").save("/mnt/bronze/test_write")
    print("✓ Bronze write successful")
except Exception as e:
    print(f"✗ Bronze write failed: {e}")
```

#### Task 10.2: Database Connectivity
```python
# Notebook: test_db_connectivity.py

from pyspark.sql import SparkSession
import pyodbc

spark = SparkSession.builder.appName("DbTest").getOrCreate()

# Using JDBC (Databricks native)
jdbcHostname = "gelsight-sql-dev.database.windows.net"
jdbcPort = 1433
jdbcDatabase = "gelsight_metadata"

url = f"jdbc:sqlserver://{jdbcHostname}:{jdbcPort};database={jdbcDatabase};encrypt=true;trustServerCertificate=false;hostNameInCertificate=*.database.windows.net;loginTimeout=30;"

try:
    df = spark.read \
      .format("com.microsoft.sqlserver.jdbc.spark") \
      .option("url", url) \
      .option("dbtable", "dim.dim_device") \
      .option("user", "{user}@{server}") \
      .option("password", dbutils.secrets.get(scope="gelsight-keyvault", key="sql-password")) \
      .load()
    
    print(f"✓ Database readable, found {df.count()} rows")
except Exception as e:
    print(f"✗ Database connection failed: {e}")
```

#### Task 10.3: ADF Pipeline Validation
- [ ] Trigger Bronze Ingest pipeline manually
- [ ] Monitor pipeline run (should complete successfully)
- [ ] Verify data in Bronze storage
- [ ] Check ingestion_log table in SQL DB
- [ ] Trigger Silver Transform pipeline
- [ ] Verify Silver tables populated
- [ ] Trigger Gold Analytics pipeline
## **COMPLETION CHECKLIST - DEV ENVIRONMENT**

**Week 1-2 Deliverables (Manual Setup):**
- [ ] ✅ Storage accounts created (Bronze, Silver/Gold)
- [ ] ✅ SQL Metadata Database created with schema
- [ ] ✅ Key Vault set up with secrets  
- [ ] ✅ Databricks workspace verified & clusters configured
- [ ] ✅ Storage mounts tested from Databricks
- [ ] ✅ Database connectivity verified
- [ ] ✅ Unity Catalog enabled & schemas created
- [ ] ✅ Sample data uploaded to Bronze
- [ ] ✅ Bronze ingestion notebook created & tested

**Success Criteria:**
✓ Can read/write to all storage accounts from Databricks
✓ Can connect to SQL Database from Databricks
✓ Unity Catalog shows gelsight_bronze schemas
✓ Bronze tables populated with sample scan data
✓ No configuration errors or access issues

---

## **DEFERRED - PHASE 3 (After Dev Validated)**

Once dev is working and you've decided on workspace strategy:
- [ ] Terraform: Convert dev to IaC, create uat/prod environments
- [ ] Azure DevOps: Set up Git repository & automated pipelines
- [ ] ADF: Create orchestrated silver/gold transformation pipelines
- [ ] Documentation: Complete runbooks & architecture diagrams

---

## **SIGN-OFF**

**Infrastructure Setup Complete**: ☐  
**Date**: ___________  
**Signed by**: ___________  
**Technical Debt**: ☐ None ☐ Documented in GitHub Issues

**Next Phase**: Begin Bronze Layer Development (Phase 1)
