**Summary:** Ah, the reverse vector, sir—extracting metadata and metrics *from* your Power BI semantic model and synchronising it *into* Databricks Unity Catalog. This enables a unified data governance view where Fabric semantic models become observable alongside your Databricks assets. The optimal approaches leverage the Power BI REST APIs, Fabric REST APIs, and the Semantic Link library.

---

**Analysis:**

There are four principal extraction methods, each surfacing different facets of your semantic model:

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Microsoft Fabric                             │
│  ┌───────────────┐    ┌───────────────┐                            │
│  │  Semantic     │───►│  Power BI /   │                            │
│  │  Model        │    │  Fabric APIs  │                            │
│  └───────────────┘    └───────┬───────┘                            │
│                               │                                     │
└───────────────────────────────┼─────────────────────────────────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │  Extraction Pipeline  │
                    │  (Notebook / ADF)     │
                    └───────────┬───────────┘
                                │
                                ▼
┌───────────────────────────────────────────────────────────────────┐
│                         Databricks                                 │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    Unity Catalog                             │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │  │
│  │  │ pbi_models  │  │ pbi_measures│  │ pbi_refresh_history │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
```

---

## Option 1: Semantic Link (Native Fabric → Databricks)

The most elegant approach within Fabric notebooks, sir. Microsoft's `semantic-link` library provides direct introspection.

**Fabric Notebook:**
```python
# Install if needed
# %pip install semantic-link

import sempy.fabric as fabric
from pyspark.sql import SparkSession
import pandas as pd
from datetime import datetime

# ============================================
# EXTRACT SEMANTIC MODEL METADATA
# ============================================

WORKSPACE_NAME = "Your Workspace"
DATASET_NAME = "Your Semantic Model"

# List all semantic models in workspace
models_df = fabric.list_datasets(workspace=WORKSPACE_NAME)
print(models_df)

# Extract tables metadata
tables_df = fabric.list_tables(
    dataset=DATASET_NAME, 
    workspace=WORKSPACE_NAME
)

# Extract measures with expressions
measures_df = fabric.list_measures(
    dataset=DATASET_NAME,
    workspace=WORKSPACE_NAME
)

# Extract relationships
relationships_df = fabric.list_relationships(
    dataset=DATASET_NAME,
    workspace=WORKSPACE_NAME
)

# Extract columns with data types
columns_df = fabric.list_columns(
    dataset=DATASET_NAME,
    workspace=WORKSPACE_NAME
)

# ============================================
# ENRICH WITH EXTRACTION METADATA
# ============================================

extraction_timestamp = datetime.utcnow().isoformat()

for df in [tables_df, measures_df, relationships_df, columns_df]:
    df['extraction_timestamp'] = extraction_timestamp
    df['source_workspace'] = WORKSPACE_NAME
    df['source_model'] = DATASET_NAME

# ============================================
# WRITE TO DATABRICKS UNITY CATALOG
# ============================================

# Configure Databricks connection
databricks_catalog = "governance"
databricks_schema = "powerbi_metadata"

# Convert to Spark DataFrames and write
spark.createDataFrame(tables_df).write \
    .mode("append") \
    .option("mergeSchema", "true") \
    .saveAsTable(f"{databricks_catalog}.{databricks_schema}.semantic_model_tables")

spark.createDataFrame(measures_df).write \
    .mode("append") \
    .option("mergeSchema", "true") \
    .saveAsTable(f"{databricks_catalog}.{databricks_schema}.semantic_model_measures")

spark.createDataFrame(relationships_df).write \
    .mode("append") \
    .option("mergeSchema", "true") \
    .saveAsTable(f"{databricks_catalog}.{databricks_schema}.semantic_model_relationships")

spark.createDataFrame(columns_df).write \
    .mode("append") \
    .option("mergeSchema", "true") \
    .saveAsTable(f"{databricks_catalog}.{databricks_schema}.semantic_model_columns")

print(f"Successfully synchronised {DATASET_NAME} to Unity Catalog")
```

---

## Option 2: Power BI REST API (Comprehensive Extraction)

For deeper metrics including refresh history, usage statistics, and lineage—requires Service Principal authentication.

**Python Pipeline (Databricks Notebook or ADF):**
```python
import requests
import pandas as pd
from datetime import datetime, timedelta
import json

# ============================================
# AUTHENTICATION
# ============================================

TENANT_ID = "<your-tenant-id>"
CLIENT_ID = "<your-service-principal-id>"
CLIENT_SECRET = dbutils.secrets.get(scope="keyvault", key="pbi-sp-secret")

def get_powerbi_token():
    """Acquire Azure AD token for Power BI API"""
    url = f"https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token"
    payload = {
        "grant_type": "client_credentials",
        "client_id": CLIENT_ID,
        "client_secret": CLIENT_SECRET,
        "scope": "https://analysis.windows.net/powerbi/api/.default"
    }
    response = requests.post(url, data=payload)
    return response.json()["access_token"]

token = get_powerbi_token()
headers = {"Authorization": f"Bearer {token}"}

# ============================================
# EXTRACT WORKSPACES & DATASETS
# ============================================

def get_workspaces():
    """Retrieve all accessible workspaces"""
    url = "https://api.powerbi.com/v1.0/myorg/groups"
    response = requests.get(url, headers=headers)
    return pd.DataFrame(response.json()["value"])

def get_datasets(workspace_id: str):
    """Retrieve datasets in a workspace"""
    url = f"https://api.powerbi.com/v1.0/myorg/groups/{workspace_id}/datasets"
    response = requests.get(url, headers=headers)
    return pd.DataFrame(response.json().get("value", []))

def get_dataset_refresh_history(workspace_id: str, dataset_id: str):
    """Retrieve refresh history for a dataset"""
    url = f"https://api.powerbi.com/v1.0/myorg/groups/{workspace_id}/datasets/{dataset_id}/refreshes"
    response = requests.get(url, headers=headers)
    return pd.DataFrame(response.json().get("value", []))

def get_dataset_datasources(workspace_id: str, dataset_id: str):
    """Retrieve datasource connections"""
    url = f"https://api.powerbi.com/v1.0/myorg/groups/{workspace_id}/datasets/{dataset_id}/datasources"
    response = requests.get(url, headers=headers)
    return pd.DataFrame(response.json().get("value", []))

# ============================================
# ENHANCED METADATA VIA SCANNER API
# ============================================

def trigger_workspace_scan(workspace_id: str):
    """Trigger enhanced metadata scan (Admin API)"""
    url = "https://api.powerbi.com/v1.0/myorg/admin/workspaces/getInfo"
    payload = {
        "workspaces": [workspace_id],
        "datasetExpressions": True,
        "datasetSchema": True,
        "datasourceDetails": True,
        "getArtifactUsers": True,
        "lineage": True
    }
    response = requests.post(url, headers=headers, json=payload)
    scan_id = response.json()["id"]
    return scan_id

def get_scan_result(scan_id: str):
    """Retrieve scan results"""
    # Poll for completion
    status_url = f"https://api.powerbi.com/v1.0/myorg/admin/workspaces/scanStatus/{scan_id}"
    while True:
        status = requests.get(status_url, headers=headers).json()
        if status["status"] == "Succeeded":
            break
        time.sleep(2)
    
    # Get results
    result_url = f"https://api.powerbi.com/v1.0/myorg/admin/workspaces/scanResult/{scan_id}"
    return requests.get(result_url, headers=headers).json()

# ============================================
# ORCHESTRATE FULL EXTRACTION
# ============================================

all_datasets = []
all_refreshes = []
all_measures = []
all_tables = []

workspaces = get_workspaces()

for _, ws in workspaces.iterrows():
    workspace_id = ws["id"]
    workspace_name = ws["name"]
    
    # Get datasets
    datasets = get_datasets(workspace_id)
    datasets["workspace_id"] = workspace_id
    datasets["workspace_name"] = workspace_name
    all_datasets.append(datasets)
    
    # Trigger enhanced scan for detailed schema
    try:
        scan_id = trigger_workspace_scan(workspace_id)
        scan_result = get_scan_result(scan_id)
        
        for ws_info in scan_result.get("workspaces", []):
            for dataset in ws_info.get("datasets", []):
                dataset_id = dataset["id"]
                
                # Extract tables
                for table in dataset.get("tables", []):
                    table_record = {
                        "workspace_id": workspace_id,
                        "workspace_name": workspace_name,
                        "dataset_id": dataset_id,
                        "dataset_name": dataset.get("name"),
                        "table_name": table.get("name"),
                        "is_hidden": table.get("isHidden", False),
                        "row_count": table.get("rows"),
                        "extraction_timestamp": datetime.utcnow().isoformat()
                    }
                    all_tables.append(table_record)
                
                # Extract measures
                for table in dataset.get("tables", []):
                    for measure in table.get("measures", []):
                        measure_record = {
                            "workspace_id": workspace_id,
                            "workspace_name": workspace_name,
                            "dataset_id": dataset_id,
                            "dataset_name": dataset.get("name"),
                            "table_name": table.get("name"),
                            "measure_name": measure.get("name"),
                            "expression": measure.get("expression"),
                            "data_type": measure.get("dataType"),
                            "is_hidden": measure.get("isHidden", False),
                            "extraction_timestamp": datetime.utcnow().isoformat()
                        }
                        all_measures.append(measure_record)
                
                # Get refresh history
                refresh_df = get_dataset_refresh_history(workspace_id, dataset_id)
                if not refresh_df.empty:
                    refresh_df["workspace_id"] = workspace_id
                    refresh_df["dataset_id"] = dataset_id
                    all_refreshes.append(refresh_df)
                    
    except Exception as e:
        print(f"Scan failed for {workspace_name}: {e}")

# ============================================
# CONSOLIDATE AND WRITE TO UNITY CATALOG
# ============================================

datasets_df = pd.concat(all_datasets, ignore_index=True)
refreshes_df = pd.concat(all_refreshes, ignore_index=True) if all_refreshes else pd.DataFrame()
measures_df = pd.DataFrame(all_measures)
tables_df = pd.DataFrame(all_tables)

# Write to Unity Catalog
catalog = "governance"
schema = "powerbi_metadata"

spark.createDataFrame(datasets_df).write \
    .mode("overwrite") \
    .saveAsTable(f"{catalog}.{schema}.datasets")

spark.createDataFrame(tables_df).write \
    .mode("overwrite") \
    .saveAsTable(f"{catalog}.{schema}.tables")

spark.createDataFrame(measures_df).write \
    .mode("overwrite") \
    .saveAsTable(f"{catalog}.{schema}.measures")

if not refreshes_df.empty:
    spark.createDataFrame(refreshes_df).write \
        .mode("overwrite") \
        .saveAsTable(f"{catalog}.{schema}.refresh_history")
```

---

## Option 3: XMLA Endpoint (Full Model Definition)

For complete semantic model extraction including calculated columns, hierarchies, and all DAX—requires Premium/Fabric capacity.

**Python with SSAS Tabular:**
```python
# Requires: pyadomd or TOM (Tabular Object Model)
# Best executed from Azure VM or local machine with .NET support

from pyadomd import Pyadomd
import pandas as pd
import json

XMLA_ENDPOINT = "powerbi://api.powerbi.com/v1.0/myorg/Your Workspace"
DATABASE_NAME = "Your Semantic Model"

# Connect via XMLA
conn_str = f"Provider=MSOLAP;Data Source={XMLA_ENDPOINT};Initial Catalog={DATABASE_NAME};"

# DMV Queries for metadata extraction
dmv_queries = {
    "tables": "SELECT * FROM $SYSTEM.TMSCHEMA_TABLES",
    "columns": "SELECT * FROM $SYSTEM.TMSCHEMA_COLUMNS", 
    "measures": "SELECT * FROM $SYSTEM.TMSCHEMA_MEASURES",
    "relationships": "SELECT * FROM $SYSTEM.TMSCHEMA_RELATIONSHIPS",
    "partitions": "SELECT * FROM $SYSTEM.TMSCHEMA_PARTITIONS",
    "roles": "SELECT * FROM $SYSTEM.TMSCHEMA_ROLES",
    "hierarchies": "SELECT * FROM $SYSTEM.TMSCHEMA_HIERARCHIES"
}

extracted_data = {}

with Pyadomd(conn_str) as conn:
    for name, query in dmv_queries.items():
        with conn.cursor().execute(query) as cursor:
            extracted_data[name] = pd.DataFrame(
                cursor.fetchall(),
                columns=[col[0] for col in cursor.description]
            )

# Write each to Unity Catalog
for table_name, df in extracted_data.items():
    df["extraction_timestamp"] = datetime.utcnow().isoformat()
    df["source_model"] = DATABASE_NAME
    
    spark.createDataFrame(df).write \
        .mode("overwrite") \
        .saveAsTable(f"governance.powerbi_metadata.{table_name}")
```

---

## Option 4: Fabric REST API for Semantic Model Definition

Extracts the complete TMDL (Tabular Model Definition Language) representation.

```python
import requests
import json

FABRIC_API = "https://api.fabric.microsoft.com/v1"
WORKSPACE_ID = "<workspace-guid>"
SEMANTIC_MODEL_ID = "<semantic-model-guid>"

def get_fabric_token():
    """Get token scoped for Fabric API"""
    url = f"https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token"
    payload = {
        "grant_type": "client_credentials",
        "client_id": CLIENT_ID,
        "client_secret": CLIENT_SECRET,
        "scope": "https://api.fabric.microsoft.com/.default"
    }
    return requests.post(url, data=payload).json()["access_token"]

token = get_fabric_token()
headers = {"Authorization": f"Bearer {token}"}

# Get semantic model definition (TMDL format)
url = f"{FABRIC_API}/workspaces/{WORKSPACE_ID}/semanticModels/{SEMANTIC_MODEL_ID}/getDefinition"

response = requests.post(url, headers=headers, json={"format": "tmdl"})
definition = response.json()

# Parse and store model definition
model_definition = {
    "semantic_model_id": SEMANTIC_MODEL_ID,
    "workspace_id": WORKSPACE_ID,
    "definition_json": json.dumps(definition),
    "extraction_timestamp": datetime.utcnow().isoformat()
}

spark.createDataFrame([model_definition]).write \
    .mode("append") \
    .saveAsTable("governance.powerbi_metadata.model_definitions")
```

---

## Unity Catalog Schema Design

I would recommend this schema structure in Databricks, sir:

```sql
-- Create catalog and schema
CREATE CATALOG IF NOT EXISTS governance;
CREATE SCHEMA IF NOT EXISTS governance.powerbi_metadata;

-- Core metadata tables
CREATE TABLE governance.powerbi_metadata.semantic_models (
    model_id STRING,
    model_name STRING,
    workspace_id STRING,
    workspace_name STRING,
    configured_by STRING,
    is_refreshable BOOLEAN,
    is_effective_identity_required BOOLEAN,
    created_date TIMESTAMP,
    extraction_timestamp TIMESTAMP
) USING DELTA;

CREATE TABLE governance.powerbi_metadata.measures (
    model_id STRING,
    model_name STRING,
    table_name STRING,
    measure_name STRING,
    expression STRING,
    data_type STRING,
    format_string STRING,
    is_hidden BOOLEAN,
    extraction_timestamp TIMESTAMP
) USING DELTA;

CREATE TABLE governance.powerbi_metadata.refresh_history (
    model_id STRING,
    refresh_id STRING,
    refresh_type STRING,
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    status STRING,
    request_id STRING,
    extraction_timestamp TIMESTAMP
) USING DELTA;

-- Enable Change Data Feed for tracking
ALTER TABLE governance.powerbi_metadata.measures 
SET TBLPROPERTIES (delta.enableChangeDataFeed = true);
```

---

## Orchestration Recommendation

**Azure Data Factory / Fabric Pipeline:**
```json
{
    "name": "PBI_to_UC_Sync_Pipeline",
    "properties": {
        "activities": [
            {
                "name": "Extract_PBI_Metadata",
                "type": "DatabricksNotebook",
                "typeProperties": {
                    "notebookPath": "/Shared/powerbi_metadata_extraction",
                    "baseParameters": {
                        "target_catalog": "governance",
                        "target_schema": "powerbi_metadata"
                    }
                }
            }
        ],
        "triggers": [
            {
                "type": "ScheduleTrigger",
                "recurrence": {
                    "frequency": "Day",
                    "interval": 1,
                    "startTime": "2024-01-01T02:00:00Z"
                }
            }
        ]
    }
}
```

---

**My Recommendation:**

For your environment, sir, I would suggest a **hybrid approach**:

1. **Semantic Link** (Option 1) for routine metadata synchronisation—elegant, native, minimal complexity
2. **REST API Scanner** (Option 2) for comprehensive governance metrics including refresh history and lineage
3. **Daily orchestration** via Fabric Pipeline or Databricks Workflow

This provides unified observability where both your Databricks assets *and* Power BI semantic models are governed within Unity Catalog—rather fitting for your Azure modernisation narrative, I should think.

Shall I elaborate on the Service Principal configuration, or perhaps construct a complete monitoring dashboard in Databricks SQL?