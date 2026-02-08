Reference implementation for building secure, autonomous Data Engineering pipelines using AI Agents and the Medallion Architecture (Bronze → Silver → Gold).

This repository demonstrates how to architect an "Agentic Workflow" where human engineers define the Governance (Rules) and Capabilities (Skills), while AI agents handle the implementation and execution. It is designed to run locally using Delta Lake for storage, offering an enterprise-grade ETL pipeline without the need for cloud infrastructure.

Core Architecture
🛡️ Security First: Implements a "Sanitization Barrier" at the Bronze layer. PII and secrets are masked in-memory before data ever touches the disk.

🏗️ Medallion Pattern:

Bronze: Raw, immutable, append-only ingestion with schema evolution.

Silver: Deduplicated, type-enforced, and cleaned data using Delta MERGE.

Gold: Aggregated Star Schema facts and dimensions for business reporting.

🤖 Agentic Design:

Rules: Markdown-based constraints (e.g., SECURITY.md, BRONZE_RULES.md) that agents must obey.

Skills: Reusable capabilities (e.g., ingest-bronze, audit-security) that extend the agent's toolkit.

Key Features
Local Delta Lake: ACID transactions, Time Travel, and Schema Enforcement running on localhost.

Zero-Trust Ingestion: Strict separation of configuration and credentials using environment variables.

Chaos Engineering Ready: Includes a chaos_generator.py to test system resilience against toxic data spills and schema drift.

Self-Healing Pipelines: Automated handling of duplicates and schema changes without crashing.

This project serves as a blueprint for the future of Hybrid Data Engineering—where engineers architect the guardrails and AI agents accelerate the build.
