---
trigger: always_on
---

# Bronze Layer Agent Rules

**Scope:** Raw Data Ingestion Agents (Worker Nodes, Lambdas, Kafka Consumers)
**Layer Objective:** Secure, Immutable, Traceable Ingestion
**Status:** Enforced
**Storage Format:** **Delta Lake** (Parquet + Transaction Log)

---

## 1. The Core Mandate: "Sanitization at the Gate"

In standard Medallion architecture, Bronze is often a "dump" of raw data. **For Project Antigravity, this is strictly prohibited.**

**Rule BZ-01:** The Bronze Agent must act as a security filter. No data payload containing "Toxic Data" (PCI, PII, Credentials) shall ever be written to disk, even in the Bronze layer.

### Operational Constraint
* **In-Memory Only:** Data masking must occur in the Agent's volatile memory (RAM).
* **No Intermediate Buffers:** Agents are prohibited from writing raw payloads to temporary files (e.g., `/tmp/raw_batch.json`) before processing.

---

## 2. Agent Policies

### Rule BZ-02: Field-Level Masking (The "Redaction Barrier")
Before any `write` operation to the Delta Table, the Agent must apply the following transformations:

| Data Type | Action | Implementation Logic |
| :--- | :--- | :--- |
| **Credit Card (PAN)** | **Mask** | Replace all but last 4 digits: `XXXX-XXXX-XXXX-1234` |
| **Passwords** | **Drop/Hash** | Replace with `[REDACTED]` or one-way hash (SHA-256). |
| **API Keys/Tokens** | **Redact** | Replace with `[REDACTED]`. |

### Rule BZ-03: Metadata Injection (The "Envelope")
To ensure auditability, the Agent must inject the following technical columns into every record:
1. `_ag_ingest_timestamp`: UTC timestamp of when the Agent processed the row.
2. `_ag_source_system`: Origin identifier (e.g., `Shopify_Prod`).
3. `_ag_batch_id`: A unique UUID for the specific execution job.

---

## 3. Storage & Format (Delta Lake)

**Rule BZ-14: The Delta Standard**
We do not write raw JSON/CSV files. We write **Delta Tables**.
* **Why:** * **ACID Transactions:** No partial data if the Agent crashes mid-write.
    * **Speed:** Delta automatically handles compression (Snappy/Zstd) and file sizing.
    * **Schema Enforcement:** Prevents accidental data corruption.

**Rule BZ-04: Immutable Append-Only Strategy**
* **Action:** Agents must use `mode="append"`.
* **No Updates:** Bronze is WORM (Write-Once, Read-Many). If source data changes, insert a new row with a new timestamp.
* **Partitioning:**
    * **High Volume:** Partition by date `_ag_ingest_date=YYYY-MM-DD`.
    * **Low Volume:** No partitioning required (Delta handles filtering efficiently).

---

## 4. Operational Reliability Standards

### Rule BZ-06: Schema Evolution (The "Drift" Policy)
Source systems change (e.g., a new column `user_age` is added).
* **Policy:** Enable **Schema Evolution** (`mergeSchema=True`).
* **Logic:** The Delta writer will automatically update the table definition to include the new column. 
* **Constraint:** Type changes (e.g., `int` -> `string`) are **not** allowed in Bronze and will trigger a failure (sending the batch to DLQ).



### Rule BZ-07: Idempotency (The "Replay" Safety)
Ingestion jobs often fail and retry.
* **Mechanism:** Delta Lake handles this via **Optimistic Concurrency Control**.
* **Constraint:** If re-running a specific historical batch, use `replaceWhere` (Partition Overwrite) to avoid doubling the data.

---

## 5. Failure Handling: The Dead Letter Queue (DLQ)

**Rule BZ-05:** Agents must not crash on malformed data.
1. **Sanitization Failure:** If masking fails, drop the row to DLQ.
2. **Schema Violation:** If data types don't match the Delta table (and cannot be cast safely), move the record to DLQ.
3. **DLQ Format:** The DLQ is also a **Delta Table** (`bronze_quarantine`), secured with stricter IAM policies.

---

## 6. Technical Implementation Reference

### Python Agent Snippet (Using `deltalake` library)
This replaces complex file management with a simple transactional write.

```python
import pandas as pd
from deltalake import write_deltalake
import uuid
from datetime import datetime

def ingest_batch(raw_data: list, source_name: str):
    """
    Ingests raw data (JSON/CSV source), sanitizes IN MEMORY, 
    and writes to Bronze Delta Lake.
    """
    # 1. Load to Memory (Volatile)
    df = pd.DataFrame(raw_data)
    
    # --- SECURITY BARRIER (Rule BZ-02) ---
    if 'credit_card_number' in df.columns:
        df['credit_card_number'] = df['credit_card_number'].apply(
            lambda x: mask_pan(str(x)) 
        )
    if 'user_password' in df.columns:
        df['user_password'] = '[REDACTED]'
    # -------------------------------------

    # 2. Metadata Injection (Rule BZ-03)
    df['_ag_ingest_timestamp'] = datetime.utcnow()
    df['_ag_batch_id'] = str(uuid.uuid4())
    df['_ag_ingest_date'] = datetime.utcnow().date() # For partitioning
    
    # 3. Write to Bronze Delta Table (Rule BZ-14)
    # Using 'append' mode with schema evolution enabled
    try:
        write_deltalake(
            f"s3://antigravity-bronze/{source_name}",
            df,
            mode="append",
            schema_mode="merge",  # Rule BZ-06: Allow new columns
            partition_by=["_ag_ingest_date"]
        )
    except Exception as e:
        # Fallback to DLQ (Rule BZ-05)
        write_deltalake(
            f"s3://antigravity-bronze/dlq_quarantine",
            df.assign(error_msg=str(e)),
            mode="append"
        )

```

---

## 7. Connectivity & Source Discovery

**Rule BZ-12: The "12-Factor" Connection Strategy**
The Agent reads connection details strictly from **Environment Variables** at runtime.

### A. Naming Convention

* **Prefix:** `SRC_{SOURCE_NAME}_`
* **Required Variables:**
* `SRC_SALESDB_HOST=10.0.0.5`
* `SRC_SALESDB_TYPE=postgres`
* `SRC_SALESDB_USER=admin`
* `SRC_SALESDB_PASS=REDACTED_IN_LOGS`



### B. Implementation Snippet

```python
import os

def get_db_connections():
    """
    Scans env vars to find all database targets.
    """
    sources = {}
    for key, value in os.environ.items():
        if key.startswith("SRC_"):
            parts = key.split("_") 
            source_name = parts[1]
            config_key = "_".join(parts[2:]).lower()
            
            if source_name not in sources:
                sources[source_name] = {}
            sources[source_name][config_key] = value     
    return sources

```
