---
trigger: always_on
---

# Silver Layer Agent Rules

**Scope:** Cleaning, Enriched Processing, & Data Quality Agents
**Layer Objective:** Type-Safe, De-duplicated, "Enterprise-Ready" Data
**Status:** Enforced
**Storage Format:** **Delta Lake** (Parquet + Transaction Log)

---

## 1. The Core Mandate: "Trust but Verify"

While Bronze protects us from security threats, Silver protects us from **Bad Data**.
Since Bronze data is often "stringly typed" (everything is text to prevent crashes), Silver is the strict enforcer.

**Rule SL-01: Strict Schema Enforcement**
The Silver Agent acts as the "Casting Director."
* **Type Casting:** The Agent must explicitly cast incoming Bronze strings to their target types (`Integer`, `Timestamp`, `Boolean`, `Decimal`).
* **The "Reject" Policy:** If a value cannot be cast (e.g., `price="free"` where a Decimal is expected), the Agent must convert it to `null` and flag the row.
* **Schema Evolution:** If valid new columns appear in Bronze, the Silver Agent uses `mergeSchema=True` to propagate them.

---

## 2. Transformation Policies

### Rule SL-02: The "Upsert" Strategy (Deduplication)
We do not simply append data to Silver, as that creates duplicates. We use Delta's **MERGE** command.

* **Logic:** Compare the incoming batch (Source) against the existing Silver Table (Target).
    * **Match Found (PK):** `UPDATE` the existing record *if and only if* the incoming `_ag_ingest_timestamp` is newer.
    * **No Match:** `INSERT` the new record.
* **Outcome:** This guarantees that the Silver layer contains **exactly one** active, up-to-date record per ID.



### Rule SL-03: The "Null" Strategy
* **String Fields:** Convert `null` to empty string `""` or `"Unknown"`.
* **Numeric Fields:** Keep as `null` (do not zero-fill).
* **Critical Keys:** If a Primary Key (e.g., `user_id`) is `null`, the row is **Dropped** entirely before hitting the Delta table.

---

## 3. Data Quality (DQ) Gates

**Rule SL-04: Row-Level Validation**
Every record passing through Silver must satisfy the "Physics of Business":
1.  **Range Checks:** `age > 0`, `price >= 0`.
2.  **Reference Checks:** `country_code` must match ISO-3166.
3.  **Quarantine:** Rows failing these checks are **not deleted**. They are written to a separate **"Bad Data" Delta Table** (`silver_quarantine`) for review.

---

## 4. Privacy & Compliance (GDPR/CCPA)

**Rule SL-05: Native Deletion**
Delta Lake supports ACID deletions, making compliance easy.
* **The "Right to be Forgotten":**
    * *Command:* `DELETE FROM silver_users WHERE user_id = '123';`
    * *Vacuum:* After deletion, run `VACUUM` to physically remove the old Parquet files containing the user's data (enforcing the purge).

---

## 5. Storage & Performance (Delta Optimized)

### Rule SL-06: Z-Ordering (Indexing)
To speed up queries, Silver tables must be "Optimized" daily.
* **Command:** `OPTIMIZE table_name ZORDER BY (event_time, customer_id)`
* **Effect:** This co-locates related data in the same files, making downstream analytics 10-100x faster.

### Rule SL-07: Time Travel (Audit Trail)
Delta automatically keeps a history of changes.
* **Usage:** If an Agent deploys bad code and corrupts data, we can instantly roll back:
    * `RESTORE TABLE silver_orders TO TIMESTAMP AS OF '2023-10-25 08:00:00'`
* **Retention:** Set table property `delta.logRetentionDuration = "30 days"` to manage storage costs.

---

## 6. Technical Implementation Reference

### Python Agent Snippet (Using `deltalake` library)

This script reads from Bronze Delta, cleans the data, and performs a Merge (Upsert) into Silver.

```python
from deltalake import DeltaTable, write_deltalake
import pandas as pd
import pyarrow.compute as pc

def silver_process_batch(bronze_table_path: str, silver_table_path: str):
    """
    Reads Bronze Delta, Enforces Types, Upserts to Silver Delta Table.
    """
    # 1. Read Bronze (Delta)
    # efficient filter: only read data ingested in the last 24h
    dt_bronze = DeltaTable(bronze_table_path)
    df = dt_bronze.to_pandas() 
    
    # 2. Type Enforcement (Rule SL-01)
    # Force numeric, turning errors ('banana') into NaN
    df['order_amount'] = pd.to_numeric(df['order_amount'], errors='coerce')
    df['order_date'] = pd.to_datetime(df['order_date'], errors='coerce')
    
    # 3. Data Quality Filter (Rule SL-04)
    # Quarantine negative amounts
    quarantine_df = df[df['order_amount'] < 0]
    clean_df = df[df['order_amount'] >= 0]
    
    # 4. Write Quarantine (Delta Append)
    if not quarantine_df.empty:
        write_deltalake(
            f"{silver_table_path}_quarantine", 
            quarantine_df, 
            mode="append"
        )

    # 5. Write Silver (Delta Merge / Upsert)
    # Connect to existing Silver table
    dt_silver = DeltaTable(silver_table_path)
    
    (
        dt_silver.merge(
            source=clean_df,
            predicate="target.order_id = source.order_id",
            source_alias="source",
            target_alias="target"
        )
        .when_matched_update_all()
        .when_not_matched_insert_all()
        .execute()
    )

```

---

## 7. Operational Resilience

### Rule SL-08: ACID Guarantee

* **Crash Safety:** If the Silver Agent crashes mid-write, Delta Lake guarantees **no partial data** is written. The transaction log (`_delta_log`) simply ignores the incomplete commit.
* **Concurrency:** Multiple Agents can write to the same table simultaneously. Delta handles the locking (Optimistic Concurrency Control).

### Rule SL-09: Circuit Breaker

* **Schema Mismatch:** If the Bronze data has a radically different schema that cannot be merged (e.g., `order_id` changed from Int to String), the write will fail safely. The Agent must catch this error and alert the Data Engineering team.

```