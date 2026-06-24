# Intro
In this README you will find the full draft for the DB development. You can limit yourself to this.

```
README.md
slides_presentation.pptx
docs/
  assumption_requirements.pdf
  logical_tables.md
```

- **`README`** This document contains the draft for the DB implementation.
- **`slides_presentation.pptx`** Essentially the README, but in presentation form.
- **`docs/assumption_requirements.pdf`** Probably the most important document once the software development phase begins; it will need to be constantly updated by both parties.
- **`docs/logical_tables.md`** It contains an example of logical tables derived from some .P00 files found in the Allocare GDF documentation.


---


# Consolidation Database

A consolidation database is needed to ingest custody and portfolio data
from more than one bank and turns it into a single, reconciled, view. At the moment the two sources are
**Corner** (delivered as Allocare GDF files) and **Julius Baer** (delivered via BJB Digital Data
Services). The design's central goal is that adding a third bank later is a configuration and
adapter exercise, not a redesign.

However different banks have different data feed , with different formats, on
different schedules, with different identifiers and different codes for the same things. Allocare
sends wide tab-delimited files; Julius Baer sends CSV positions, ISO 20022 XML cash statements, and
zipped PDF documents. Nothing shares a key: the same security is a Valor at one bank and an ISIN at
another, and the same client is a portfolio name at one and an account number at the other. Without a
consolidation layer, producing a single cross-bank view of a client's holdings is manual,
error-prone, and slow. This system is that consolidation layer.


## Architecture

![Architecture](docs/4_layers.png)

The principle running top to bottom is the following: each bank's data stays in its native shape as long as possible,
is conformed to a common model exactly once, and is never merged blindly, meaning that overlapping figures are
**reconciled**.


## Layer 1 — Ingestion

**What it does.** Collects files from each bank into a landing area without altering them, and logs every arrival (name, source, version, timestamp, row count, outcome). Expected gaps are understood, Julius Baer delivers Tuesday to Saturday, so a quiet weekend is normal while a missing weekday file is flagged.

**Technology.** SFTP over SSH with key-based authentication and fixed-IP whitelisting (to note the deadline set by Julius Baer for a specific comunnication algorithm from 28 June 2026). Scheduling via cron, or an orchestrator such as Prefect or Airflow later; the landing area is the filesystem or object storage, and the audit trail is a database table.

**How it connects.** Once a file is landed and logged, the pipeline reads from this landing area, not from the banks, so any day can be reprocessed without re-fetching.

## Layer 2 — Pipeline

**What it does.** Turns raw files into validated, conformed records. Each format has its own **parser** (Allocare GDF, BJB CSV, camt.053 XML, eDocs index), all emitting the same canonical shapes, which then pass through a shared sequence: **transform** (resolve identifiers to internal keys, normalise dates and numbers, treat placeholders like Unknown as missing), **validate** (quarantine malformed rows with a reason, load the rest), and **load** (write to the database with inserts/updates/deletes and full lineage back to the source file).

**Technology.** Python — `pandas`, the standard XML and CSV libraries, schema validation, and SQLAlchemy. The parsers follow the **adapter pattern** behind a common interface (`fetch → parse → emit canonical records`), so transform/validate/load are written once and shared. Adding a bank means one new adapter, not touching the core pipeline.

**How it connects.** The pipeline is the only component that writes to the core, so the database never needs to know anything about file formats.

## Layer 3 — Core database

**What it does.** Holds all custody data in one unified schema, organised in three bands: a **staging band** keeping each feed raw, as delivered; a **conformed core** (instruments and their identifiers, names and features; portfolios and their cash and custody accounts; transactions and movements; dated position snapshots; camt.053 cash; market rates; the document register); and a **cross-cutting band** (identity crosswalks, a code dictionary, reconciliation, and lineage) that makes two banks behave as one.

**Technology.** A single relational database, such as PostgreSQL or MariaDB (essentialy an improved fork of MySQL), with volumes that sit comfortably on one server. Positions are stored as the banks deliver them rather than recalculated, so the custodian's figures remain the reference; reconciliation compares them against transaction history as a control.

**How it connects.** Applications read from the core through query-friendly views and a thin read API.

## Layer 4 — Applications

**What it does.** Everything the consolidated data makes possible, built on top without changing anything beneath. The first use is **consolidated reporting** (one cross-bank view of a client's holdings, cash and transactions). Beyond that, **simulation**, **analytics**, and **compliance**, since every figure traces back to a source file and reconciliation exceptions are tracked.

**Technology.** Off-the-shelf BI tools (Power BI, Metabase) on the reporting views, or bespoke apps on the read API. The layer is deliberately thin and replaceable — the value lives in the conformed data below it.

---

## Implementation roadmap

Five phases, each delivering something usable on its own. Indicative durations assume one developer and are ranges, not commitments — the second-bank work in particular depends on the open questions below.

- **Phase 1 — Foundation** *(3–4 weeks)*. Stand up the schema and get Corner/Allocare flowing end to end: SFTP ingestion, the GDF parser, the shared pipeline, and the in-scope files loaded. *Done when* an Allocare delivery loads automatically and a portfolio's positions and transactions are queryable — proving the schema on real data before a second source.
- **Phase 2 — Second bank** *(2–3 weeks)*. Add Julius Baer: the BJB adapters, the identity crosswalk linking both banks' instruments and accounts, and reconciliation so overlapping figures aren't double-counted. *Done when* a client held at both banks shows one correct view and mismatches surface as tracked exceptions.
- **Phase 3 — Hardening** *(2–3 weeks)*. Make it dependable: malformed-file handling, schedule monitoring, alerting on failures and open reconciliation breaks, and day-level reprocessing. *Done when* the system runs unattended and flags problems.
- **Phase 4 — Applications** *(to be seen)*. Reporting views plus a BI tool, the consolidated cross-bank client report, and a first analytical view. *Done when* the team can read a consolidated client picture without writing a query.
- **Phase 5 — Extensibility** *(to be seen)*. Onboard a third custodian as one new parser, extending the crosswalk and code dictionary with no change to the core or the shared stages. *Done when* its data joins the same consolidated view (adapter pattern).

---

## Assumptions affecting the plan

Two answers need answers before beginnig:

1. **Ingestion route.** Does Julius Baer data arrive directly, or does it flow into Allocare first
   and reach us as GDF? The design assumes the harder, direct case; if BJB flows through Allocare,
   Phase 2 shrinks substantially (the CSV/XML/ZIP adapters are not needed).
2. **BJB feed layout.** The exact column layout of the Julius Baer positions/transactions CSV is not
   yet in hand; the BJB-specific tables and crosswalk mappings are finalised when it arrives.

*NOTE*: The assumption and requiremetns pdf should be checked thourghly to be sure both our views of the system (what it should and shouldn't do) are aligned.