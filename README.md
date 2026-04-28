# Infernal Management System — Hell Database

A relational database project built in **T-SQL (MS SQL Server)** that models the administrative infrastructure of Hell as a fully operational bureaucratic system. Designed as a university database course project, it demonstrates end-to-end database engineering from conceptual modelling to stored procedures with complex business logic.

---

## Project Overview

The system manages the day-to-day operations of Hell, treating it as an organisation with employees (devils), clients (sinners), governance (the Infernal Council), and operational workflows (punishments, temptations, promotions). Every design decision is grounded in a defined ruleset, making this a realistic exercise in translating business requirements into a working database.

## Database Schema

The schema was designed using Vertabelo (conceptual + implementation diagrams) and consists of 14 tables, built with careful attention to dependency order to ensure referential integrity.

### Core Entities

| Table                          | Description                                                                    |
| ------------------------------ | ------------------------------------------------------------------------------ |
| `Grzesznik` (Sinner)           | Deceased individuals serving sentences in Hell                                 |
| `Diabel` (Devil)               | Hell's workforce - may originate from promoted sinners or be primordial demons |
| `Grzech` (Sin)                 | Sin catalogue with three severity categories: Light, Medium, Heavy             |
| `Kara` (Punishment)            | Dictionary of available punishment types                                       |
| `Osoba_zyjaca` (Living Person) | Tracked for temptation targeting and prayer monitoring                         |

### Operational Tables

| Table               | Description                                                                                    |
| ------------------- | ---------------------------------------------------------------------------------------------- |
| `Wymierzona_kara`   | Assigned punishment per sinner, with a severity score (0–1000) reducible by prayers            |
| `Nadzor_kary`       | Devil-to-punishment supervision assignments (cap: 1,000 per devil)                             |
| `Wpis_grzech`       | Sin log - records each sin committed with a score and date                                     |
| `Kuszenie`          | Temptation attempts by devils against living persons                                           |
| `Modlitwa`          | Prayer log — each prayer reduces punishment severity by 0.5 and shortens its duration by 1 day |
| `Przydzial_funkcji` | Devil role assignments (Tempter or Tormentor) with performance ratings (1–10)                  |

### Governance Tables

| Table              | Description                                                 |
| ------------------ | ----------------------------------------------------------- |
| `Rada_piekielna`   | Infernal Council - 6,666-year terms                         |
| `Czlonek_rady`     | Council membership records                                  |
| `Doradca_lucyfera` | Lucifer's advisors - appointed from retired council members |
| `Raport`           | Population reports generated every 666 years                |

### Performance Indexes

Non-clustered indexes were added on all high-frequency JOIN and WHERE columns, including sin logs, supervision assignments, temptation lookups, and punishment duration tracking, to minimise full-table scans.

## Stored Procedures

Six stored procedures handle the core business logic of the system, each with input validation and error handling via `RAISERROR`.

### `DodajNadzorKary` — Assign Punishment Supervision

> **Challenge:** Enforce capacity limits and role eligibility without application-layer logic.
> **Solution:** Gate-checked insert that validates role, availability, and a hard cap of 1,000 active supervisions per devil before any write occurs.

Assigns a devil to supervise a sinner's punishment. Validates that:

- the devil exists and holds an active Tormentor role
- the punishment isn't already actively supervised
- the devil hasn't exceeded the 1,000 simultaneous supervision cap

### `DodajModlitwe` — Register a Prayer

> **Challenge:** External events (prayers) must propagate side-effects across related records atomically.
> **Solution:** Single procedure call logs the prayer and immediately updates punishment severity and duration, keeping all dependent state in sync.

Logs a prayer from a living person for a sinner and automatically updates the sinner's punishment reducing severity by 0.5 and shortening duration by one day. Validates liveness of the praying person and active punishment status.

### `ZmienFunkcjeDiabla` — Change Devil Role

> **Challenge:** Prevent role-change abuse while still enabling performance-based reassignment.
> **Solution:** Dual-condition guard (66-year cooldown + performance rating ≤ 3/10) enforced at the DB layer, with full assignment history preserved on every transition.

Switches a devil between Tempter and Tormentor roles. Enforces a 66-year cooldown and requires a performance rating ≤ 3/10, ensuring only underperforming devils are rotated.

### `NadwyzkaDiablow` — Devil Surplus Rebalancing

> **Challenge:** Population overflow management - too many Tormentors relative to available sinners.
> **Solution:** Automated resource rebalancing selects the least-loaded Tormentors (by active supervision count) and reassigns them as Tempters, maintaining operational efficiency without manual intervention.

Triggered when the devil population exceeds 100,000,000. Identifies the surplus count, selects the least-loaded active Tormentors (by active supervision count), closes their current role assignments, and reassigns them as Tempters - preserving full assignment history.

### `UtworzRade` / `ZakonczRade` — Council Lifecycle Management

> **Challenge:** Multi-step governance transitions must update entity statuses, close active assignments, and handle two distinct end-of-term outcomes (natural expiry vs. early dissolution) without leaving orphaned records.
> **Solution:** Branching procedure logic detects term length, conditionally promotes or demotes members, and cascades all related record closures in a single call.

- **`UtworzRade`**: Elects a new Infernal Council by selecting top-performing devils across four merit categories (most successful temptations, temptations of devout people, temptations leading to mortal sins, most supervised punishments) - 333 per category, deduplicated. Updates statuses, closes role assignments, and ends active supervisions for elected members.
- **`ZakonczRade`**: Ends a council's term. If the full 6,666-year term is served, members are promoted to **Lucifer's Advisors** with a randomly assigned sphere of influence; if dissolved early by Lucifer, members revert to standard devil status.

### `UtworzRaport` — Population Report

> **Challenge:** Periodic executive reporting with period-over-period comparisons and duplicate prevention.
> **Solution:** Cooldown-gated snapshot procedure that calculates deltas against the previous report and stores them denormalised for fast retrieval, a deliberate trade-off documented in the design.

Generates a governance report every 666 years capturing total devils (broken down by status), active sinners, and period-over-period changes in both populations. Enforced cooldown prevents duplicate reports.

## Business Queries

The project includes analytical SQL queries addressing operational intelligence needs:

1. **Prayer Risk Report** — ranks living persons by prayer frequency to identify high-priority temptation targets (frequent prayers destabilise punishments)
2. **Sinner Prayer Coverage** — shows which sinners receive the most prayers and when the last one was logged
3. **Devil Workload Distribution** — displays current role assignments and supervision counts relative to total devil population, using CTEs
4. **Council Candidate Ranking** — four sub-queries identifying the top 333 candidates per election category, mirroring the `UtworzRade` procedure logic

## Technical Highlights

- **T-SQL / MS SQL Server** — full use of `IDENTITY`, `CHECK` constraints, `DEFAULT` values, `UNIQUE` constraints, and nullable foreign keys for optional relationships
- **CTEs and Window Functions** — used in council formation logic, workload distribution queries, and data generation scripts
- **`CROSS APPLY` with `NEWID()`** — randomised sphere assignment during council graduation without cursor overhead
- **`SCOPE_IDENTITY()`** — used to reliably reference newly inserted council IDs across subsequent inserts in the same transaction
- **Realistic test data** — generated using `CROSS JOIN`, `CHECKSUM(NEWID())`, and external tooling (BeekeeperStudio), with deterministic seed data for static reference tables
- **Penalty scoring formula** — punishment severity is calculated as `SUM(sin_scores) / 6.0`, capped at 1,000; sin frequency is normalised per year of life for fair cross-sinner comparison

## Project Structure

```
📁 Pieklo_BD_NOTEBOOK_NOWOSAD_JUSTYNA.ipynb
├── Section 1 — Schema creation (tables + indexes)
├── Section 2 — Test data insertion
└── Section 3 — Business queries + stored procedures
```

Schema was first modelled conceptually and implementationally in Vertabelo before being translated into SQL.

---

## Why This Project

This project was built as a university database course assignment, but the domain was chosen deliberately to make the data model non-trivial. Hell-as-a-bureaucracy introduces real-world database design challenges:

- **multi-role entities** (a sinner can become a devil; a devil can become a council member or advisor)
- **time-tracked relationships** (role assignments, council memberships, supervision periods all have start/end dates)
- **business rule enforcement** at the database layer rather than the application layer
- **cascading state changes** (electing a council member closes their role assignment and their active supervisions simultaneously)

These are the same patterns that appear in HR systems, CRM platforms, and operational databases in enterprise environments.
