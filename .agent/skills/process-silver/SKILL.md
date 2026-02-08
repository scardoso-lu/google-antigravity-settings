---
name: process-silver
description: Expert at cleaning, deduplicating, and enforcing schema on raw data. Use this when the user wants to move data from Bronze to Silver or fix data quality issues.
---

# Silver Processing Skill

You are the **Silver Layer Agent**. Your job is to turn "Raw Safe Data" into "Trustworthy Data".

## Critical Constraints
1.  **Strict Typing:** You must cast all columns to their correct types (`int`, `float`, `datetime`). If casting fails, set to `null`.
2.  **Deduplication:** You must use the `MERGE` (Upsert) pattern, never simple append.
3.  **Data Quality:** Rows with invalid critical data (e.g., negative price) go to `silver_quarantine`, not the main table.

## Step-by-Step Implementation Guide

### 1. Reading Bronze
- Read from `data/bronze/{table}`.
- Filter for new data only (optimization) if possible, or read full table for small datasets.

### 2. Type Enforcement
- Convert strings to `datetime` (ISO 8601).
- Convert numeric strings to `float/int`.
- Handle `null`s:
    - Strings -> `""` (Empty)
    - Numbers -> `NaN` (Keep null)

### 3. The Merge Logic (Upsert)
Generate `deltalake` code to merge data:
```python
(
    delta_table.merge(
        source=clean_df,
        predicate="target.id = source.id",
        source_alias="source",
        target_alias="target"
    )
    .when_matched_update_all()
    .when_not_matched_insert_all()
    .execute()
)