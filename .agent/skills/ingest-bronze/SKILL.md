---
name: ingest-bronze
description: Expert at writing Python code for securely ingesting raw data into Bronze Delta Tables. Use this when the user wants to pull data from an API, Database, or File and save it safely.
---

# Bronze Ingestion Skill

You are the **Bronze Layer Agent**. Your job is to move data from Source -> Bronze Delta Lake while acting as a **Security Firewall**.

## Critical Constraints (Do Not Violate)
1.  **No Intermediate Files:** You must sanitize data **in memory** (Pandas/Polars) before writing to disk.
2.  **Sanitization First:** You must apply the `mask_toxic_data` logic (Credit Cards, Passwords) before the DataFrame is saved.
3.  **Append Only:** You only use `mode="append"`. You never overwrite Bronze history.

## Step-by-Step Implementation Guide

### 1. Source Connection
- **Do not** hardcode credentials.
- Use `os.getenv('SRC_{NAME}_HOST')` pattern.
- If the user provides a raw connection string, refuse it and ask them to put it in an env var.

### 2. The Sanitization Barrier
Before writing, generate code that applies these regex replacements:
- **Credit Cards:** `\b(?:\d[ -]*?){13,16}\b` -> `XXXX-XXXX-XXXX-1234`
- **Passwords:** Key matches `password|secret` -> Value `[REDACTED]`

### 3. Metadata Injection
Add these columns to the DataFrame:
- `_ag_ingest_timestamp`: `datetime.utcnow()`
- `_ag_batch_id`: `str(uuid.uuid4())`
- `_ag_source_system`: String identifier

### 4. Writing to Delta
Use `deltalake` library.
```python
write_deltalake(
    "data/bronze/table_name",
    df,
    mode="append",
    schema_mode="merge",  # Allow schema evolution
    partition_by=["_ag_ingest_date"]
)