# genie-migration

> **Prerequisite:** The export and deployment scripts are intended to be run from **Databricks notebooks**. No local Python environment or `requirements.txt` is required.

GitHub repository for managing and reviewing **Databricks Genie Space** changes before promoting them from DEV to PROD.

> **Note:** PROD deployment is performed manually. This repository does not automatically deploy changes to PROD.


## Workflow

```text
DEV
 │
 │ Develop & test Genie Space
 ▼
Feature Branch
 │
 │ Push configuration
 ▼
Pull Request
 │
 │ Reviewer approval
 ▼
main
 │
 │ Manual migration
 ▼
PROD
```

## Process

1. **Develop & test** the Genie Space in the DEV Databricks workspace.

2. **Export/update** the Genie Space configuration using the **Genie Space JSON downloader** notebook.

3. **Create a feature branch** and push the changes.

4. **Open a Pull Request (PR)** from the feature branch to `main`.

5. **Reviewer reviews and approves** the changes.

6. **Merge the PR into `main`** once approved.

7. **Deploy to PROD** using the **Create Genie Space from JSON** notebook.

8. *(Optional)* Run the supporting scripts to create views, load data, or apply column descriptions.


## Branching

- `main` — Approved production configuration
- `feature/*` — Changes under development/review

Direct pushes to `main` should be restricted through **GitHub branch protection**.


## Environments

| Environment | Catalog       |
| ----------- | ------------- |
| DEV         | `com_edp_dev` |
| PROD        | `com_edp_prd` |


## Repository Structure

```text
genie-migration/
├── README.md
├── Scripts/
│   ├── Genie Space JSON downloader          # Export DEV space → JSON (with catalog/schema/table conversion)
│   ├── Create Genie Space from JSON         # Deploy JSON → PROD Genie Space
│   ├── Create temp view for current deployment  # Create PROD views for remapped tables
│   ├── HCP_HCO_Affiliation_Data_Load        # Truncate & reload hcp_hco_affiliation table
│   └── Insert Column descriptions to Claims Space Tables  # Apply column comments to PROD tables
└── Spaces/
    └── PatientJourney&ClaimsAnalytics.json   # Exported Genie Space configuration
```


---

# Configuration Variables

When migrating a different Genie Space, the following variables must be reviewed.

---

## Genie Space JSON downloader

**Notebook:** `Scripts/Genie Space JSON downloader`

This notebook exports a DEV Genie Space, converts all catalog/schema/table references to their PROD equivalents, merges Unity Catalog column comments into the Genie `column_configs`, and saves the result as a JSON file.

### 1. DEV Genie Space ID

Find the Genie Space ID in the **DEV Databricks Genie UI**.

Update:

```python
space = w.genie.get_space(
    space_id="<GENIE_SPACE_ID>"
)
```

This is the Genie Space that will be exported from DEV.


### 2. Conversion Configuration

The script uses three configuration variables to control the DEV → PROD conversion:

```python
SOURCE_CATALOG = "com_edp_dev"
TARGET_CATALOG = "com_edp_prd"
TARGET_SCHEMA  = "cmpa_insights_internal_schema"
```

| Variable         | Purpose                                                |
| ---------------- | ------------------------------------------------------ |
| `SOURCE_CATALOG` | The DEV catalog name to replace                        |
| `TARGET_CATALOG` | The PROD catalog name                                  |
| `TARGET_SCHEMA`  | The PROD schema that remapped tables are redirected to |


### 3. Table Remaps (`TABLE_REMAPS`)

Tables that have a different schema or name between DEV and PROD are listed in `TABLE_REMAPS`. Each entry maps a `(source_schema, source_table)` pair to the target table name. All remapped tables land in `TARGET_SCHEMA`.

Current configuration:

```python
TABLE_REMAPS = {
    ("com_intgr", "claims_medical_events"):  "vw_claims_medical_events",
    ("com_intgr", "claims_pharmacy_events"): "vw_claims_pharmacy_events",
    ("com_consm", "hcp_hco_affiliation"):    "vw_hcp_hco_affiliation",
}
```

The conversion is performed in three steps:

1. **Catalog conversion** — all occurrences of `SOURCE_CATALOG` are replaced with `TARGET_CATALOG`.
2. **Table name conversion** — source table names are replaced with their target names.
3. **Schema conversion** — source schemas referenced in `TABLE_REMAPS` (e.g. `com_intgr`, `com_consm`) are remapped to `TARGET_SCHEMA`, scoped to `TARGET_CATALOG`.

To add a new remapped table, add an entry to `TABLE_REMAPS`.


### 4. Column Descriptions

Column descriptions are handled in two stages:

1. **Genie space column_configs** — descriptions already present in the Genie Space configuration are extracted and preserved.
2. **Unity Catalog column comments merge** — the script reads `DESCRIBE TABLE` from the DEV tables and merges any UC column comments into the `column_configs`. Existing Genie-authored descriptions are never overwritten.

The DEV identifier for each table is derived automatically by reversing the `TABLE_REMAPS` mapping. If a table has a non-standard DEV path, update the reverse map logic.


### 5. Export Output Path

Update the output file path in the final cell:

```python
output_path = "/Workspace/Users/<USER>/<PATH>/<SPACE_NAME>.json"
```

The file name should correspond to the Genie Space being migrated.


---

## Create Genie Space from JSON

**Notebook:** `Scripts/Create Genie Space from JSON`

This notebook reads the exported JSON file and creates or updates a PROD Genie Space via the REST API. It includes a pre-flight check for column descriptions and a post-deployment verification step.

### 1. PROD Warehouse ID

```python
PRD_WAREHOUSE_ID = "<PRD_WAREHOUSE_ID>"
```

The SQL Warehouse that the PROD Genie Space will use.

### 2. PROD Space ID

To **update** an existing PROD Genie Space:

```python
PRD_SPACE_ID = "<PRD_SPACE_ID>"
```

To **create** a new PROD Genie Space:

```python
PRD_SPACE_ID = None
```

### 3. PROD Parent Path

The workspace folder where a new Genie Space will be created:

```python
PARENT_PATH = "/Workspace/Users/<USER>"
```

### 4. Input JSON File

Path to the approved JSON file (the version merged into `main`):

```python
with open("/Workspace/Users/<USER>/<PATH>/<SPACE_NAME>.json", "r") as f:
    _exported = json.load(f)
```

### Deployment Behavior

The script:

1. Loads the JSON export (containing `metadata` and `serialized_space`).
2. Sorts tables by identifier (required by the API).
3. Runs a **pre-flight check** reporting how many column descriptions are in the payload.
4. **Creates** (POST) or **updates** (PATCH) the Genie Space via the REST API.
5. **Verifies** the deployment by fetching the space back and comparing column description counts.


---

## Supporting Scripts

### Create temp view for current deployment

Creates `CREATE OR REPLACE VIEW` statements in PROD for the three remapped tables (e.g. `vw_claims_medical_events`, `vw_claims_pharmacy_events`, `vw_hcp_hco_affiliation`). Run this before deployment if the views don't already exist.

### HCP_HCO_Affiliation_Data_Load

Truncates and reloads the `hcp_hco_affiliation` table from upstream source tables. Accepts `catalog_name`, `schema_name`, and `table_name` as widget parameters.

### Insert Column descriptions to Claims Space Tables

Applies `ALTER TABLE ... ALTER COLUMN ... COMMENT` statements to 11 PROD tables using descriptions from the Genie Space metadata. Uses configurable `CATALOG` and `SCHEMA` variables (default: `com_edp_prd.cmpa_insights_internal_schema`).
