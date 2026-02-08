---
trigger: always_on
---

# Operations - Governance

**Scope:** Pipeline Orchestration, Maintenance, & Documentation
**Environment:** Localhost Execution
**Objective:** Reliability, Reproducibility, and "Sanity Management"

---

## 1. Orchestration (The "Runbook")

You cannot run Gold before Silver. You need a strict dependency graph.

**Rule OPS-01: The DAG Mandate (Directed Acyclic Graph)**
Pipelines must be executed in strict layer order.
* **Sequence:** `Ingest (Bronze)` -> `Clean (Silver)` -> `Aggregate (Gold)`.
* **Blocking:** If Bronze fails, Silver **must not** start (to prevent propagating bad data).
* **Local Implementation:** Use a `main.py` orchestrator or a `Makefile` to enforce this. Do not rely on manually running separate scripts in random order.

**Rule OPS-02: Idempotent Re-Runs**
You will break things. You will need to re-run.
* **Constraint:** Running the pipeline twice on the same day must produce the **exact same result** (not double data).
* **Mechanism:**
    * **Bronze:** Append-Only (deduplication happens downstream).
    * **Silver/Gold:** Use `MERGE` (Upsert) or `OVERWRITE` on specific partitions. Never use simple `APPEND` in Silver/Gold.

---

## 2. Maintenance & Storage Hygiene

Delta Lake tables grow indefinitely if left unchecked. On a local machine, this will eat your SSD.

**Rule OPS-03: The "Vacuum" Policy**
Delta keeps old file versions for "Time Travel." You don't need infinite history locally.
* **Action:** Run `VACUUM` on all Delta Tables once a week.
* **Retention:** Keep `7 days` of history (allows you to debug last week's issues).
    ```python
    delta_table.vacuum(retention_hours=168) # 7 Days
    ```
* **Safety:** Never run `VACUUM` with `retention=0` unless you are actively fixing a corruption or purging toxic data.

**Rule OPS-04: Log Rotation**
Local logs file up quickly.
* **Constraint:** Application logs (`app.log`) must rotate daily and delete after 14 days.
* **Tool:** Use Python's `RotatingFileHandler`.

---

## 3. Documentation (The Data Dictionary)

"Self-documenting code" is a myth in data projects.

**Rule OPS-05: The "Readme per Mart"**
Every Gold Data Mart (e.g., `gold/sales/`) must contain a `README.md`.
* **Content:**
    1.  **Business Definition:** What is "Net Revenue"? (e.g., `(Price * Qty) - Tax`).
    2.  **Owner:** Who asked for this? (e.g., "CFO Office").
    3.  **Refresh Rate:** How often is it updated? (e.g., "Daily at 8 AM").

**Rule OPS-06: Schema Versioning**
If you change a Silver/Gold schema (add/remove columns), you must bump the **Semantic Version** in the table properties.
* **Property:** `delta.userMetadata = {"version": "1.2.0"}`.
* **Why:** Helps debug when a dashboard suddenly breaks ("Oh, we are on v1.2, but the dashboard expects v1.1").

---

## 4. Disaster Recovery (Local Edition)

If your laptop dies, the project shouldn't die with it.

**Rule OPS-07: The "Code vs. Data" Split**
* **Code:** Commited to Git (GitHub/GitLab). **Zero Tolerance** for uncommitted work at end of day.
* **Data:** Local Delta tables are **NOT** committed to Git.
* **Backup:** Sync your `data/` folder to an external drive or S3 bucket ("Cold Storage") once a week using `rclone` or `aws s3 sync`.

**Rule OPS-08: The "Clean Slate" Protocol**
New developers (or you on a new machine) must be able to bootstrap in <10 minutes.
* **Requirement:** A `setup.sh` script that:
    1.  Creates the virtual environment.
    2.  Installs `requirements.txt`.
    3.  Creates the empty folder structures (`data/bronze`, `data/silver`, etc.).
    4.  Generates dummy `.env` file (from template).

---

## 5. Monitoring & Alerts

**Rule OPS-09: The "Heartbeat"**
Even locally, you need to know if it worked.
* **Success:** Print a green "✓ Pipeline Finished: Processed X records" to stdout.
* **Failure:** Print a red "✗ Pipeline Failed: [Error Message]" to stderr.
* **Metric:** Track **"End-to-End Latency"**. (Start time of Bronze - End time of Gold). If this creeps up over time, you need to optimize (Z-Order).

---

## 6. Implementation Reference: The Orchestrator (`main.py`)



```python
import sys
from src.bronze import ingest
from src.silver import clean
from src.gold import aggregate

def run_pipeline():
    try:
        print("--- Starting Bronze ---")
        ingest.run() # Writes to data/bronze
        
        print("--- Starting Silver ---")
        clean.run()  # Reads bronze, writes data/silver
        
        print("--- Starting Gold ---")
        aggregate.run() # Reads silver, writes data/gold
        
        print("✅ Pipeline Success")
        
    except Exception as e:
        print(f"❌ Pipeline Failed: {e}", file=sys.stderr)
        sys.exit(1)

if __name__ == "__main__":
    run_pipeline()

```
