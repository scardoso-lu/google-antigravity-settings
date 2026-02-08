---
trigger: always_on
---

# Code Principles (Local Edition)

**Scope:** Python, SQL, & Configuration Management
**Environment:** Localhost (Docker / VirtualEnv)
**Objective:** Write code that is secure, portable, and ready for future scale.

---

## 1. The "Local-First" Manifesto

**Rule CP-01: Environment Parity**
Your local setup must mimic production behavior, even if the "cloud" is just a folder on your SSD.
* **Constraint:** Never use absolute paths like `C:\Users\Name\Project`.
* **Solution:** Use relative paths or environment variables.
    * *Bad:* `df = pd.read_csv("D:/data/raw.csv")`
    * *Good:* `DATA_DIR = os.getenv('DATA_DIR', './data'); df = pd.read_csv(f"{DATA_DIR}/raw.csv")`

**Rule CP-02: Containerization Ready**
Even if running via `python main.py` today, write code as if it's inside Docker.
* **No GUI Dependencies:** Do not use `matplotlib.pyplot.show()` or `input()`. The code must run headless.
* **Logs to Stdout:** Do not write logs to local files only; stream them to the console so Docker logs can capture them.

---

## 2. Security in Code (The Antigravity Standard)

**Rule CP-03: The "No-Hardcode" Zone**

We treat local code as if it were public.
* **Secrets:** Never paste a password, even for a "quick test." Use `.env` files (added to `.gitignore`).
* **Logic:** `if os.getenv('ENV') == 'local': load_dotenv()`

**Rule CP-04: The "Safe-Fail" Block**
Ingestion code must be defensive.
* **Try/Except:** Wrap all IO operations (Network/Disk) in `try/except` blocks.
* **Silent Failure is Forbidden:** If an error occurs, log the *specific* error (without leaking secrets) and re-raise or send to Dead Letter Queue (DLQ).

---

## 3. Python Style Guide (PEP-8 Plus)

**Rule CP-05: Type Hinting (Strict)**
Data engineering is messy; types bring order.
* **Mandatory:** All function signatures must be typed.
    ```python
    # Good
    def mask_pan(pan: str) -> str: ...
    
    # Bad
    def mask_pan(pan): ...
    ```

**Rule CP-06: Function Atomicity**
* **One Job:** A function should do one thing (e.g., "fetch_data" OR "clean_data", never both).
* **Length:** If a function exceeds 50 lines, refactor it.

**Rule CP-07: Imports Management**
* **Grouping:** Standard Libs first, Third Party second, Local Modules third.
* **No Wildcards:** `from module import *` is **strictly prohibited** (pollutes namespace and hides dependencies).

---

## 4. Local Data Management

**Rule CP-08: The ".gitignore" Treaty**
You are responsible for ensuring "Toxic Data" never enters Git.
* **Mandatory Entries:**
    ```gitignore
    .env
    __pycache__/
    *.csv
    *.json
    *.parquet
    data/
    logs/
    .delta_log/
    ```

**Rule CP-09: Mock Data Generation**
Do not use real customer data for local testing.
* **Faker Library:** Use the `Faker` library to generate synthetic PII (Names, Emails, IPs) for development.
* **Deterministic Seeds:** Set `Faker.seed(42)` so your tests are reproducible.

---

## 5. Testing Strategy

**Rule CP-10: Unit vs. Integration**
* **Unit Tests:** Test logic (e.g., "Does my regex mask this credit card?"). Run fast, no disk IO.
* **Integration Tests:** Test the pipeline (e.g., "Can I write to the local Delta table?"). Slower, uses local disk.

**Rule CP-11: The "Clean Slate" Fixture**
Tests must not depend on leftover data from previous runs.
* **Setup/Teardown:** Each test run must create a temporary directory for Delta tables and delete it afterwards.

---

## 6. Project Structure (Standard Layout)

Keep your local project organized:


```

antigravity/
├── src/
│   ├── bronze/          # Bronze Agent Code
│   ├── silver/          # Silver Agent Code
│   ├── gold/            # Gold Agent Code
│   └── shared/          # Common utils (Masking, Logging, Config)
├── tests/               # Pytest folder
├── config/              # YAML configs (Non-sensitive only)
├── data/                # LOCAL ONLY (Ignored by Git)
│   ├── landing/         # Raw JSON dumps
│   ├── bronze/          # Delta Tables
│   └── silver/          # Delta Tables
├── .env.example         # Template for secrets
├── .gitignore
├── README.md
├── requirements.txt
└── main.py              # Entry point

```