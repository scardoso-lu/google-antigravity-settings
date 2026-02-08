---
trigger: always_on
---

# Security Policy

**Scope:** End-to-End Data Pipeline (Source -> Bronze -> Silver)
**Architecture:** Medallion on Delta Lake
**Status:** Enforced

---

## 1. The "Antigravity" Security Model

Our security model is built on three unshakeable pillars. These are not guidelines; they are architectural constraints hard-coded into our Agents.

1.  **Zero-Trust Ingestion:** We assume all incoming data is "toxic" (contains PII/Secrets) until proven otherwise.
2.  **Immutable Audit:** We never delete history without a trace (Delta Time Travel).
3.  **Ephemeral Secrets:** Agents never "know" credentials; they are injected only at runtime.

---

## 2. Credential & Secret Management

**Rule SEC-01: The "12-Factor" Injection Mandate**
* **Constraint:** No Agent (Bronze or Silver) shall ever possess a configuration file containing credentials.
* **Mechanism:** Connections are strictly managed via the `SRC_{NAME}_` Environment Variable protocol defined in the Bronze Layer.
* **Detection:** All PRs must pass the **Semgrep** scanner to ensure no high-entropy strings (API Keys) are committed to `main`.

| Secret Type | Allowed Injection | Prohibited |
| :--- | :--- | :--- |
| **DB Passwords** | `os.environ['SRC_DB_PASS']` | Hardcoded strings, `.env` files |
| **AWS/Cloud Keys** | IAM Role / Instance Profile | Long-lived Access Keys in code |
| **API Tokens** | Secrets Manager Reference | Plain text in `config.yaml` |

---

## 3. The "Sanitization Barrier" (Bronze Layer)

**Rule SEC-02: The "No-Write" Policy**
To prevent toxic data leaks, the following strict order of operations is enforced in the **Bronze Agent**:

1.  **Fetch:** Pull data from Source to **RAM**.
2.  **Sanitize:** Apply Regex Masking (Credit Cards, Passwords) **in RAM**.
3.  **Write:** Only *then* write to **Delta Lake**.

**❌ Violation:** Writing raw, unmasked data to a temporary disk file (e.g., `/tmp/debug.json`) is a **Severity 1 Security Incident**.

**Rule SEC-03: Toxic Data Classification**
The following data types are defined as "Toxic" and must be neutralized before landing in Bronze:

| Category | Regex / Pattern | Action | Implementation |
| :--- | :--- | :--- | :--- |
| **Credit Card (PAN)** | `\b(?:\d[ -]*?){13,16}\b` | **Mask** (Last 4 visible) | `XXXX-XXXX-XXXX-1234` |
| **Passwords** | Keys matching `pass`, `pwd`, `secret` | **Drop or Hash** | `[REDACTED]` |
| **API Bearer Tokens** | `Bearer\s+[a-zA-Z0-9\-\._~\+\/]+=*` | **Redact** | `Bearer *****` |

---

## 4. Storage Security (Delta Lake)

**Rule SEC-04: Encryption & Access**
* **At Rest:** All Delta Tables (Bronze & Silver) must sit on buckets with **Server-Side Encryption (SSE-S3 or KMS)** enabled by default.
* **In Transit:** Agents must enforce `sslmode=require` (Postgres) or `useSSL=true` (MySQL) when connecting to sources.
* **Access Control:** Direct access to the raw Parquet files is restricted to the **System Admin**. Developers must query via the SQL Endpoint or Spark Cluster, which enforces Table ACLs.

**Rule SEC-05: The "Right to be Forgotten" (GDPR/CCPA)**
* **Mechanism:** Deletions are processed in the **Silver Layer** using Delta's `DELETE` command.
* **Purge:** A `VACUUM` command must run within 7 days of a deletion request to physically remove the Parquet files containing the user's data from storage history.

---

## 5. Audit & Traceability

**Rule SEC-06: Metadata Lineage**
Security logs are not enough. The data itself must prove its origin.
Every record in Antigravity must carry the **"Security Envelope"** columns:
* `_ag_ingest_timestamp`: *When did we let this in?*
* `_ag_source_system`: *Who gave us this data?*
* `_ag_batch_id`: *Which job run is responsible?*



---

## 6. Incident Response

**Trigger:** A secret is leaked in a commit, or unmasked PII is found in Delta/Logs.

1.  **Stop the Bleeding:** Immediately revoke the compromised credential (rotate the key).
2.  **Clean the Lake:**
    * If PII is in **Bronze**: Run a `DELETE` job immediately and `VACUUM` the table (0 retention) to wipe the physical file.
    * **Warning:** Do not just "overwrite" the data; Delta's Time Travel will keep the toxic version unless you `VACUUM`.
3.  **Patch the Agent:** Update the Bronze Agent's Regex filter to catch the leaked pattern.
4.  **Report:** Notify `#antigravity-security`.

---

## 7. Developer Checklist

Before merging any code to `antigravity-core`:
* [ ] **Scanner:** Did `pre-commit` run without errors?
* [ ] **Logs:** Did you verify you are NOT logging raw dicts (`print(payload)`)?
* [ ] **Masking:** If adding a new source, did you update the Regex list for that source's specific toxic fields?