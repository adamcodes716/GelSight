# GelSight Data Platform – Code Promotion & Branching Strategy

**Purpose**: Define how code moves from development → UAT → Production, including branching strategy, code review, testing, and deployment procedures.

**Date**: February 11, 2026  
**Status**: Ready for implementation  
**Audience**: All developers, tech leads, DevOps engineers

---

## **EXECUTIVE SUMMARY**

This document defines:
1. **Git branching model** (feature branches, release branches, hotfix branches)
2. **Code review requirements** (who approves, what criteria)
3. **Testing gates** (unit, integration, performance tests)
4. **Promotion workflow** (Dev → UAT → Prod)
5. **Rollback procedures** (what to do if production fails)
6. **Release versioning** (semantic versioning: major.minor.patch)

---

## **1. GIT BRANCHING STRATEGY**

### **Branch Hierarchy**

```
┌─ main (Production)
│  ├─ protected: require 2 approvals, status checks pass
│  ├─ latest release, tagged with version numbers
│  └─ only receives commits via release/* or hotfix/*
│
├─ release/v* (UAT Staging)
│  ├─ created from develop when ready for UAT
│  ├─ allow bugfixes only (no new features)
│  ├─ merged back to main when approved
│  └─ merged back to develop to keep in sync
│
├─ develop (Dev Integration)
│  ├─ protected: require 1 approval, status checks pass
│  ├─ all feature branches merge here
│  ├─ nightly builds deployed to Dev environment
│  └─ basis for release branches
│
├─ feature/* (Feature Development)
│  ├─ created from develop
│  ├─ one feature per branch (e.g., feature/calibration-scd2)
│  ├─ deleted after merge
│  └─ examples:
│     ├─ feature/bronze-ingestion-pipeline
│     ├─ feature/silver-coordinate-transform
│     └─ feature/gold-qc-dashboard
│
├─ bugfix/* (Bug Fixes)
│  ├─ created from develop
│  ├─ fixes non-critical bugs
│  ├─ merged to develop
│  └─ examples:
│     └─ bugfix/coordinate-precision-issue
│
└─ hotfix/* (Production Emergency Fixes)
   ├─ created from main
   ├─ fixes critical production issues
   ├─ merged to main AND develop
   ├─ tagged with version increment
   └─ examples:
      └─ hotfix/duplicate-measurements-fix
```

---

## **2. BRANCHING WORKFLOW FOR DEVELOPERS**

### **Creating a New Feature Branch**

```bash
# Step 1: Start from develop (latest)
git checkout develop
git pull origin develop

# Step 2: Create feature branch
# Naming: feature/{jira-ticket}/{short-description}
git checkout -b feature/GELS-123-calibration-scd2-dimension

# Step 3: Make commits (atomic, descriptive)
git add src/transformations.py
git commit -m "GELS-123: Implement SCD Type 2 for calibration dimension"

# Step 4: Push to remote
git push origin feature/GELS-123-calibration-scd2-dimension

# Step 5: Open Pull Request on GitHub
# [open GitHub → Create Pull Request]
```

### **Pull Request Template** (`.github/pull_request_template.md`)

```markdown
## Description
Brief description of what this PR does.

## Related Issue
Closes #(issue number or JIRA ticket)

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Data model change
- [ ] Infrastructure change
- [ ] Documentation

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests passed
- [ ] Test coverage >= 80%

## Checklist
- [ ] Code follows style guide
- [ ] Self-review completed
- [ ] Comments added for complex logic
- [ ] Documentation updated
- [ ] No breaking changes to API/schema

## Reviewers
@tech-lead @databricks-expert
```

### **Code Review Requirements**

| Branch | Required Approvals | Approvers | Stale Check |
|--------|-------------------|-----------|-------------|
| `develop` | 1 approval | Any senior engineer | Dismiss & re-request |
| `release/*` | 2 approvals | Tech lead + architect | Dismiss only when resolved |
| `main` | 2 approvals + environment approval | Tech lead + release manager | Never dismiss |

### **Code Review Checklist** (For Reviewers)

- [ ] Code is readable and follows naming conventions
- [ ] Changes align with architecture (Bronze → Silver → Gold)
- [ ] No hardcoded secrets, credentials, or API keys
- [ ] Tests are comprehensive (unit + integration)
- [ ] Performance impact assessed (no N+1 queries, etc.)
- [ ] Error handling present and appropriate
- [ ] Backward compatibility preserved (if applicable)
- [ ] Documentation updated
- [ ] Data quality checks in place (if data transformation)

---

## **3. CODE PIPELINE & TESTING GATES**

### **Continuous Integration (CI) Pipeline**

```
Developer Push to feature/* branch
        ↓
GitHub Actions: ci-build.yml
├─ Checkout code
├─ Set up Python 3.11
├─ Install dependencies (requirements-dev.txt)
├─ Run linting (pylint)
│  └─ Must pass with grade >= 8.0/10
├─ Run unit tests (pytest)
│  └─ Must pass all tests
│  └─ Coverage must be >= 80%
├─ Run static analysis (bandit for security)
│  └─ No critical/high severity issues
├─ Check secrets (gitleaks)
│  └─ No credentials detected
└─ Upload coverage to Codecov

Status: ✓ PASS or ✗ FAIL (blocks merge)
```

**`requirements-dev.txt`**:
```
# Core data libraries
pandas>=1.5.0
pyspark>=3.4.0
numpy>=1.24.0

# Testing
pytest>=7.4.0
pytest-cov>=4.1.0
pytest-mock>=3.11.1

# Code quality
pylint>=2.17.0
black>=23.7.0
flake8>=6.0.0
bandit>=1.7.5

# Development
ipython>=8.14.0
jupyter>=1.0.0

# YAML parsing
pyyaml>=6.0

# Data validation
pydantic>=2.0

# Database
sqlalchemy>=2.0
pyodbc>=4.0.39
```

### **Manual Testing Gates** (PR → Develop)

Before approving a PR, reviewer should:

```bash
# 1. Check out the feature branch locally
git checkout feature/GELS-123-calibration-scd2-dimension
git pull origin feature/GELS-123-calibration-scd2-dimension

# 2. Run tests locally
pytest bronze/tests/ -v
pytest silver/tests/ -v
pytest gold/tests/ -v

# 3. Test in notebook (if applicable)
# Upload test notebook to Databricks Dev cluster
# Run interactively to verify behavior

# 4. Review changes visually
# Ensure:
# - No large files committed
# - No data accidentally committed
# - Comments clear and helpful

# 5. Check integration test results from CI
# Click "Details" on GitHub status check to see ADF pipeline test run
```

### **Integration Testing** (After Merge to Develop)

```
Merge to develop (via GitHub)
        ↓
GitHub Actions: deploy-dev.yml
├─ Upload notebooks to Dev Databricks workspace
├─ Upload configuration YAML files
├─ Run integration tests
│  ├─ Test storage connectivity
│  ├─ Test database connectivity
│  ├─ Test Bronze ingest with sample data
│  ├─ Test Silver transform with ingested data
│  └─ Test Gold analytics with Silver data
├─ Check data quality metrics
├─ Email team on success/failure
└─ Create deployment notification in Teams

Duration: ~30 minutes
```

---

## **4. PROMOTION WORKFLOW (Dev → UAT → Prod)**

### **Phase 1: Development (Continuous)**

```
Developer creates feature/* branch
        ↓ (multiple commits/days)
        ↓
Open Pull Request to develop
        ↓
Code Review (1 approval required)
        ↓
Merge to develop
        ↓
GitHub Actions: deploy-dev.yml
├─ Deploy to Dev Databricks
├─ Run integration tests
└─ Notify team
        ↓
Dev-ready ✓
(stays here: 1-2 weeks, developer iterates as needed)
```

### **Phase 2: Release Candidate (Weekly)**

```
Tech Lead: Create release branch
        ↓
$ git checkout -b release/v0.1.0 develop

        ↓
Add/update CHANGELOG.md
        ↓
Bump version: 0.0.0 → 0.1.0 (in version.py or setup.py)
        ↓
Commit: "Release v0.1.0: Add calibration dimension"
        ↓
Push to remote
        ↓
Create Pull Request: release/v0.1.0 → main
        ↓
UAT Testing Gates (Run Manually):
│
└─ GitHub Actions: deploy-uat.yml
   ├─ Deploy to UAT Databricks workspace
   ├─ Ingest sample data (full regression suite)
   ├─ Run ALL integration tests
   ├─ Verify data quality (>95% completeness)
   ├─ Check pipeline performance metrics
   └─ Email team: "UAT Ready"
        ↓
Wait 1-2 weeks: UAT team tests thoroughly
        ↓
UAT Sign-off: ✓ Ready for Production
```

### **Phase 3: Production (Controlled)**

```
Tech Lead approves UAT PR
        ↓
Request environment approval
        ↓
Production Manager approves (requires 2-factor auth)
        ↓
GitHub Actions: deploy-prod.yml TRIGGERED MANUALLY
├─ Create backup of prod database
├─ Create backup of prod storage
├─ Deploy to Production Databricks
├─ Run smoke tests (verify core functionality)
├─ Monitor pipeline completion
├─ Verify data quality checks
├─ Create deployment report
└─ Notify stakeholders
        ↓
Merge to main
        ↓
Create Release Tag: v0.1.0
        ↓
Production Ready ✓
        ↓
Merge back to develop (keep in sync)
```

### **Promotion Timeline**

| Phase | Duration | Owner | Status |
|-------|----------|-------|--------|
| Feature Development | 3-5 days | Developer | Iterative |
| Code Review | 1-2 days | Tech Lead | Blocking |
| Dev Deployment | 30 min | CI/CD | Automatic |
| Dev Soak | 1-2 weeks | QA Team | Parallel |
| UAT Promotion | 2 hours | Tech Lead | Manual |
| UAT Testing | 1-2 weeks | QA + Business | Blocking |
| UAT Sign-off | 1 day | Manager | Required |
| Production Promotion | 2 hours | Ops Team | Manual |
| **Total**: | **3-4 weeks** | **Team** | **Controlled** |

---

## **5. HOTFIX PROCEDURE (Emergency Production Fixes)**

If critical issue discovered in production:

```
Incident detected (Data incorrect, pipeline crashing, etc.)
        ↓
Create hotfix branch from main
        ↓
$ git checkout -b hotfix/GELS-999-critical-fix main

        ↓
Fix code
        ↓
Test thoroughly with sample data
        ↓
Bump patch version: 0.1.0 → 0.1.1
        ↓
Create Pull Request: hotfix/GELS-999 → main
        ↓
Fast-track Code Review (1 approval, expedited)
        ↓
Merge to main
        ↓
GitHub Actions: deploy-prod.yml (MANUAL TRIGGER)
├─ Deploy to Production (high priority)
├─ Run smoke tests
└─ Notify stakeholders
        ↓
Tag: v0.1.1-hotfix
        ↓
ALSO merge hotfix/GELS-999 → develop
(Keep develop in sync with fix)
        ↓
Post-Incident Review: Ensure root cause understood
```

**Hotfix Rules**:
- No new features in hotfixes
- Only fix critical bugs
- Must be merged to both main AND develop
- Requires expedited approval
- Must document in incident report

---

## **6. VERSIONING STRATEGY (Semantic Versioning)**

Format: `MAJOR.MINOR.PATCH`

Examples:
- `v0.0.0`: Initial development
- `v0.1.0`: Beta release (Bronze layer feature complete)
- `v0.1.1`: Hotfix (coordinate precision bug)
- `v1.0.0`: Production-ready (all layers complete)
- `v1.1.0`: Phase 2: New routine added to Silver
- `v2.0.0`: Breaking change (schema redesign)

### **When to Increment**

| Version | When | Example |
|---------|------|---------|
| MAJOR | Breaking changes, schema redesign | Completely new fact table structure |
| MINOR | New features, new routines supported | Add PitDetection routine |
| PATCH | Bugfixes, performance improvements | Fix coordinate transformation precision |

### **Version File** (`version.py`)

```python
# bronze/__init__.py
__version__ = "0.1.0"
__release_date__ = "2026-02-21"

# Can be read in notebooks:
# from bronze import __version__
# print(f"Bronze layer v{__version__}")
```

---

## **7. RELEASENOTES & CHANGELOG**

### **CHANGELOG.md Format**

```markdown
# Changelog
All notable changes to this project are documented here.

## [0.1.0] - 2026-02-21

### Added
- Implemented Bronze layer ingestion pipeline
- Support for Offset, DefectDetection, HoleDiameter, PitDetection routines
- Calibration file parsing and lineage tracking

### Fixed
- Issue #45: Coordinate transformation precision
- Issue #52: Duplicate file uploads on retry

### Changed
- Updated YAML parsing to handle nested structures
- Improved error messages in validation

### Security
- No hardcoded credentials in notebooks
- All secrets moved to Key Vault

### Known Issues
- SurfaceRoughness routine not yet supported
- Large file uploads (>1GB) may timeout (workaround: split)

---

## [0.0.1] - 2026-02-04

### Added
- Initial infrastructure setup
- Storage accounts and database created
- Databricks cluster configured
```

### **Release Summary** (Posted to Teams/Slack)

```
🚀 Production Release: v0.1.0

🎯 Overview
Deployed Bronze Layer Ingestion and Calibration Dimension Tables

📊 What's New
✅ Full YAML parsing for 4 routines (Offset, DefectDetection, HoleDiameter, PitDetection)
✅ Calibration SCD Type 2 dimension implemented
✅ Ingestion metadata tracking
✅ Data quality metrics baseline established

🐛 Bugs Fixed
- Issue #45: Coordinate precision (was 4 decimals, now 10)
- Issue #52: Retry loop duplicate uploads

📈 Metrics
- Average ingestion time: 2 min per scan
- Data quality: 98.5% completeness
- Pipeline success rate: 99.2%

⚠️ Known Limitations
- SurfaceRoughness routine not yet supported (planned for v0.2)
- Large files (>1GB) require manual chunking

🔗 Links
📖 Full Release Notes: https://github.com/...
📋 Tickets Closed: v0.1.0 Milestone
🐞 Report Issues: https://github.com/.../issues
```

---

## **8. MONITORING & ALERTS**

### **Pipeline Monitoring Dashboard**

Monitor these metrics:

| Metric | Green | Yellow | Red |
|--------|-------|--------|-----|
| Build Success Rate | >95% | 85-95% | <85% |
| Test Coverage | >85% | 70-85% | <70% |
| Average Merge Time | <2 weeks | 2-4 weeks | >4 weeks |
| Code Review Time | <2 days | 2-5 days | >5 days |
| Hotfix Rate | <1/month | 1-3/month | >3/month |
| Production Incidents | 0/month | 1-2/month | >2/month |

### **Alerts**

Configure notifications in GitHub Actions for:

- ✅ **Build Successful**: Posted to #data-platform Slack channel
- ❌ **Build Failed**: Ping @dev-team in Slack
- ⚠️ **UAT Sign-off Pending**: Weekly reminder to @qa-lead
- 🚀 **Production Ready**: Notification to exec team

---

## **9. DEVELOPER SETUP INSTRUCTIONS**

### **Initial Setup**

```bash
# 1. Clone the repository
git clone https://github.com/yourorga/gelsight-data-platform.git
cd gelsight-data-platform

# 2. Create and activate virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements-dev.txt

# 4. Configure git hooks (pre-commit checks)
pip install pre-commit
pre-commit install

# 5. Set up git user (if not done)
git config user.name "Your Name"
git config user.email "your.email@company.com"
```

### **Pre-commit Hooks** (`.pre-commit-config.yaml`)

```yaml
repos:
  - repo: https://github.com/psf/black
    rev: 23.7.0
    hooks:
      - id: black
        language_version: python3.11
  
  - repo: https://github.com/PyCQA/pylint
    rev: pylint-2.17.0
    hooks:
      - id: pylint
        args: [--load-plugins=pylint_django, --disable=C0111,W0212]
  
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: [--baseline, .secrets.baseline]
```

### **Daily Workflow**

```bash
# Updating feature branch
git checkout feature/GELS-123-my-feature
git fetch origin
git rebase origin/develop

# Making a commit
git add bronze/src/my_changes.py
git commit -m "GELS-123: Description of what I changed"

# Pushing and creating PR
git push origin feature/GELS-123-my-feature
# Then open GitHub to create PR

# Pulling latest from develop
git checkout develop
git pull origin develop

# Running tests locally before pushing
pytest bronze/tests/ -v --cov=bronze/src
```

---

## **10. COMPLIANCE & GOVERNANCE**

### **Code Review SLA**

- **Target**: Review completed within 2 business days
- **Critical/Hotfix**: Within 4 hours
- **Default**: Dismiss stale reviews if changes made

### **Testing Requirements**

- All PRs must pass CI pipeline (tests, coverage, security)
- Coverage must be ≥80% for new code
- All tests must be deterministic (no flaky tests)
- Integration tests must pass in Dev environment

### **Documentation Requirements**

- Every new function/notebook must have docstring
- Schema changes must be documented in INFRASTRUCTURE_ARCHITECTURE.md
- Data transformations must include unit tests
- Config changes must be commented

### **Security Requirements**

- No hardcoded secrets (use Key Vault)
- No credentials in code (use service principals)
- All data access logged and auditable
- Private endpoints for all Azure services
- TLS 1.2+ for all communications

---

## **11. TROUBLESHOOTING COMMON ISSUES**

### **Issue: Merge Conflict**

```bash
# Pull latest from develop
git fetch origin
git rebase origin/develop

# Resolve conflicts in editor
# Mark as resolved
git add <resolved-file>
git rebase --continue

# Force push to feature branch
git push origin feature/my-feature --force
```

### **Issue: Accidentally Committed to Main**

```bash
# Undo last commit (keep changes)
git reset --soft HEAD~1

# Create feature branch
git checkout -b feature/accidental-commit
git push origin feature/accidental-commit

# Create PR from feature branch
```

### **Issue: Test Failure in CI**

```bash
# Pull latest code
git pull origin feature/my-branch

# Run tests locally
pytest -v

# Debug failing test
pytest bronze/tests/test_ingestion.py::test_parse_yaml -vvs

# Fix code
# Commit and push
git push origin feature/my-branch
```

---

## **12. QUICK REFERENCE**

### **Common Commands**

```bash
# Create feature branch
git checkout -b feature/GELS-{ticket}-{description} develop

# Make changes and commit
git add .
git commit -m "GELS-{ticket}: Description"

# Push to remote
git push origin feature/GELS-{ticket}-{description}

# Update from develop (rebase)
git fetch origin
git rebase origin/develop

# Finalize and push
git push origin feature/GELS-{ticket}-{description} --force-with-lease

# Squash commits before merge (if needed)
git rebase -i origin/develop
# Mark commits as 'squash', save
# Push --force-with-lease
```

### **GitHub CLI Commands**

```bash
# Create PR
gh pr create --base develop --title "GELS-123: Feature title"

# List PRs waiting for review
gh pr list --state open

# Request review
gh pr review <PR_number> --comment "Approved"

# Merge PR
gh pr merge <PR_number> --squash --delete-branch
```

---

## **Sign-Off**

**Document Version**: 1.0  
**Owner**: Tech Lead  
**Reviewed By**: Architecture Board  
**Effective Date**: February 11, 2026  
**Next Review**: After first production release
