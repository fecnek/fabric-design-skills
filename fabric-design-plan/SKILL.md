---
name: fabric-design-plan
description: >
  Fabric SDLC — Phase 3: Task Breakdown. Use this skill whenever the user wants to turn
  a technical design into an actionable work breakdown structure, create a task backlog for
  a Microsoft Fabric platform build, sequence the delivery order across disciplines, or
  plan the implementation sprints. Triggers: "/fabric-design-plan", "P3", "phase 3", "task breakdown", "build order",
  "delivery plan", "work breakdown", "sprint planning", "who does what", "sequence the build",
  "task backlog", "implementation tasks". Always run after Phase 2. Produces discipline-specific
  task lists with dependencies, ordered by build sequence, saved as MD files.
---

# Phase 3 — Task Breakdown

## What this phase produces

Phase 3 converts the Phase 2 technical design into a structured, sequenced backlog of
implementation tasks. Tasks are organised by discipline, each discipline has a defined
build order, and cross-discipline dependencies are made explicit.

**Outputs:**
- `p3-outputs/tasks/tasks_[discipline].md` — one file per discipline
- `p3-outputs/build_order.md` — the sequenced delivery plan across all disciplines
- `p3-outputs/_task_summary.md` — index of all tasks with IDs, owners, dependencies

---

## Step 0 — Load Phase 2 outputs

Load and read all Phase 2 files before generating tasks:
- `p2-outputs/_design_summary.md`
- `p2-outputs/design/*.md` (all 10 design area files)
- `p2-outputs/workspaces/ws_*.md` (all workspace files)

If Phase 2 files are not available, ask the user to either run Phase 2 first or describe
the design directly so tasks can be derived.

---

## Task ID format

Every task gets a unique ID: `T[DISCIPLINE-CODE]-[NNN]`

| Discipline Code | Discipline |
|----------------|-----------|
| `INF` | Infrastructure & Platform |
| `GOV` | Governance & Compliance |
| `SEC` | Security |
| `OPS` | Operations & Monitoring |
| `WS` | Workspace Setup |
| `ING` | Data Ingestion (per source system) |
| `DQ` | Data Quality |
| `AGG` | Aggregation Layer |
| `SEM` | Semantic Layer & Reporting |
| `CICD` | CI/CD & DevOps |
| `TEST` | Testing & Validation |

---

## The 11 disciplines

For each discipline below, generate the tasks derived from Phase 2. Every task entry
must follow this format:

```markdown
### T[CODE]-[NNN]: [Task name]
- **Description:** [What needs to be done and why]
- **Inputs:** [What this task depends on — files, completed tasks, deployed resources]
- **Output / Deliverable:** [The concrete artefact or state when done]
- **Blocked by:** [Task IDs that must complete before this one — or "None"]
- **Blocks:** [Task IDs that cannot start until this one is done]
- **Effort estimate:** [S = ≤1 day, M = 2–3 days, L = 1 week, XL = 2+ weeks]
- **Owner discipline:** [Who does this work]
- **Notes:** [Edge cases, decisions pending, open questions]
```

---

### Discipline 1 — Infrastructure & Platform (INF)

These tasks establish the Azure and Fabric foundations. Nothing else can start until INF
tasks are done. They must be sequenced strictly.

Generate tasks for:
- Create Azure Resource Groups per environment (dev/test/prod)
- Provision Azure Key Vaults (one per environment)
- Provision Fabric Capacities (F SKUs per environment, per region — derived from
  Design Area 2 capacity map)
- Configure Fabric tenant settings (Admin portal: capacities, workspaces, sensitivity
  labels enabled, Purview integration on)
- Set up Azure Monitor / Log Analytics workspace
- Configure private endpoints for Confidential/Restricted workspaces (from Design Area 5)
- Set up Azure DevOps or GitHub organisation and repositories structure (foundation
  for Phase 4)
- Provision Microsoft Purview account and connect to Fabric tenant
- Apply environment resource tagging policy

---

### Discipline 2 — Governance & Compliance (GOV)

These tasks set up the Governance workspace (`ws_gov_data_governance`) and the rules
that all other workspaces depend on. Deploy after INF.

Generate tasks for:
- Create the `ws_gov_data_governance` Fabric workspace
- Set up `lh_bronze_gov_registry` and `lh_silver_gov_registry` Lakehouses
- Define and load the initial workspace registry table (all workspace names, codes, owners,
  SLA tiers, classification — derived from Design Area 2)
- Implement naming convention audit pipeline (scans all workspaces nightly, reports violations)
- Define initial DQ rule catalogue in YAML (from Design Area 7 — one task per domain)
- Implement rule validation and publishing pipeline
- Set up data contract templates (one per Gold table type)
- Configure Purview source registration (automate workspace registration via API)
- Implement Purview lineage push notebook
- Create Governance health Power BI report

---

### Discipline 3 — Security (SEC)

Set up AAD groups, service principals, and classification labels. Deploy after INF,
coordinate with GOV (Purview labels need Purview account from GOV).

Generate tasks for:
- Create AAD security groups (one per workspace × role — from Design Area 8)
- Create Service Principals (one per workspace — from Design Area 8)
- Store Service Principal secrets in Key Vault
- Configure Purview sensitivity labels (Public / Internal / Confidential / Restricted)
- Apply DLP policies for Confidential and Restricted labels
- Configure RBAC assignments on all Fabric workspaces (using AAD groups)
- Set up Row-Level Security (RLS) definitions for semantic models that require it
  (from Design Area 8 RLS table)
- Implement access audit log pipeline (pushes access events to Ops workspace)
- Schedule weekly access review report for Confidential/Restricted workspaces

---

### Discipline 4 — Operations & Monitoring (OPS)

Set up the `ws_ops_monitoring` workspace. Deploy after INF and SEC.

Generate tasks for:
- Create `ws_ops_monitoring` Fabric workspace with Bronze and Silver lakehouses
- Define and implement the standard pipeline log schema (JSON, from Design Area 9)
- Implement the log ingestion pipeline (collects events from all other workspaces)
- Implement SLA evaluation pipeline (hourly, per workspace, per tier)
- Configure alerting rules (Teams/email) for P1 SLA breaches
- Build the Operations Power BI report (pipeline health, SLA compliance %, DQ trends)
- Create the incident tracking table and runbook template

---

### Discipline 5 — Workspace Setup (WS)

Create all domain, aggregation, and consumption workspaces. Deploy after INF, SEC, OPS,
and GOV. One task group per workspace — derive from the workspace MD files.

Generate tasks for each workspace:
- Create Fabric workspace (with correct capacity, region, AAD groups assigned)
- Create Bronze Lakehouse (L1 workspaces only)
- Create Silver Lakehouse (L1 workspaces only)
- Create Gold Lakehouse (L2 workspaces only)
- Create OneLake shortcuts (L2 workspaces — shortcuts to L1 Silver lakehouses)
- Configure metadata table in workspace Lakehouse (`pipeline_metadata`)
- Register workspace in Governance registry

Group workspace tasks by layer — create all L1 workspaces before L2, all L2 before L3.

---

### Discipline 6 — Data Ingestion (ING)

Build source-to-Bronze pipelines. Deploy after WS. One task group per source system /
workspace combination — derive from Design Area 3 and Design Area 4.

Generate tasks for each source system per workspace:
- Create ADF Linked Service or Fabric connector for the source system
- Store connection credentials in Key Vault; reference from Linked Service
- Implement metadata-driven ingest pipeline (reads `pipeline_metadata` table for
  source tables, watermarks, and targets)
- Implement Bronze table schema (Delta format, schema-on-read, `_ingested_at` audit column)
- Test ingest pipeline with sample data (validate row counts, no data loss)
- Register pipeline in Governance workspace registry
- Configure pipeline log emission to Operations workspace

---

### Discipline 7 — Data Quality (DQ)

Build Bronze→Silver transformation pipelines with DQ gates. Deploy after ING.

Generate tasks for each workspace:
- Implement Bronze→Silver transform pipeline (cleanse, type, resolve business keys)
- Load DQ rules from Governance workspace (via OneLake shortcut to rule catalogue)
- Implement DQ gate logic (hard_block / quarantine / warn per rule severity)
- Create quarantine table in Silver Lakehouse (stores rejected rows with rule reference)
- Configure DQ score push to Governance workspace after each pipeline run
- Test DQ gates with seeded bad data (verify quarantine and promotion behaviour)
- Document DQ rule coverage per table (% of columns covered by at least one rule)

---

### Discipline 8 — Aggregation Layer (AGG)

Build Silver→Gold aggregation pipelines in L2 workspaces. Deploy after DQ.

Generate tasks for each L2 workspace:
- Verify all required OneLake shortcuts are active and tables are readable
- Implement Gold aggregation pipeline (cross-domain join / aggregate logic)
- Enforce data contracts on Gold tables (validate schema against contract YAML on each run)
- Publish Gold semantic model (DirectLake dataset)
- Test Gold data against business acceptance criteria (validate KPI calculations)
- Register Gold data product in Governance workspace

---

### Discipline 9 — Semantic Layer & Reporting (SEM)

Build L3 semantic models and Power BI reports. Deploy after AGG.

Generate tasks for each semantic model / report:
- Create L3 Power BI workspace
- Create DirectLake semantic model connecting to L2 Gold lakehouse
- Define measures and KPIs (from Design Area 10)
- Apply RLS to semantic model (where required)
- Build Power BI reports and dashboards
- Certify dataset (via CI/CD pipeline — see CICD discipline)
- Conduct UAT with business consumers
- Publish to end users

---

### Discipline 10 — CI/CD & DevOps (CICD)

Set up source control and deployment pipelines. Start in parallel with INF; complete
before any workspace goes to test/prod.

Generate tasks for:
- Set up repo structure for each workspace (derived from Phase 4 folder structure)
- Implement Fabric deployment pipeline (dev → test → prod) per workspace
- Implement infrastructure-as-code templates (Bicep/ARM for Azure resources)
- Configure branch protection rules (PRs required, at least 1 reviewer for test/prod)
- Implement automated schema validation in CI pipeline (validates data contracts)
- Implement naming convention check in CI pipeline
- Set up environment-specific variable groups / secrets in CI/CD platform
- Implement workspace certification gate (semantic model certified only after CI passes)
- Set up Dependabot or equivalent for dependency updates

---

### Discipline 11 — Testing & Validation (TEST)

Cross-cutting testing tasks. Run after each discipline completes its core build.

Generate tasks for:
- Unit tests for DQ rule logic (per domain)
- Integration test: full pipeline run from source to Silver (per L1 workspace)
- Integration test: full pipeline run from Silver to Gold (per L2 workspace)
- Data contract validation: schema match between Gold tables and published contract
- SLA validation: confirm P1/P2/P3 pipelines complete within SLA windows under load
- Security validation: confirm RBAC prevents cross-workspace data access
- RLS validation: confirm RLS filters data correctly per user group
- Performance test: semantic model query response under expected user concurrency
- UAT sign-off by business domain owner (per Core domain)
- Regression test suite in CI/CD (runs on every PR)

---

## Build order file

After generating all discipline task files, create `p3-outputs/build_order.md` that
defines the sequence of delivery in phases/sprints:

```markdown
# Build Order

## Phase / Sprint Sequence

### Foundation (deploy first — everything depends on this)
| Order | Task IDs | Discipline | Dependency |
|-------|---------|-----------|-----------|
| 1 | INF-001 to INF-0NN | Infrastructure | None |
| 2 | SEC-001 to SEC-0NN | Security | INF complete |
| 3 | GOV-001 to GOV-0NN | Governance | INF, SEC |
| 4 | OPS-001 to OPS-0NN | Operations | INF, SEC |

### Domain Workspaces — Layer 1 (build supporting BCs before domain BCs)
| Order | Workspace | Task IDs | Blocked by |
|-------|-----------|---------|-----------|

### Aggregation Layer — Layer 2
| Order | Workspace | Task IDs | Blocked by |
|-------|-----------|---------|-----------|

### Consumption Layer — Layer 3
| Order | Workspace | Task IDs | Blocked by |
|-------|-----------|---------|-----------|

### CI/CD & Testing (runs in parallel with domain workspace builds)
| Order | Task IDs | Notes |
|-------|---------|-------|

## Critical Path
[List the tasks that form the longest dependency chain — these drive the delivery timeline]

## Parallel Workstreams
[List groups of tasks that can be done in parallel once foundation is complete]
```

---

## Task summary file

Create `p3-outputs/_task_summary.md`:

```markdown
# Task Summary

## Counts by Discipline
| Discipline | Task Count | S | M | L | XL | Total Effort (days est.) |
|-----------|-----------|---|---|---|----|--------------------------|

## Total Tasks: [N]
## Estimated Duration: [N] weeks (assuming [N] parallel workstreams)

## All Tasks Index
| Task ID | Name | Discipline | Effort | Blocked by | Blocks |
|---------|------|-----------|--------|-----------|--------|
```

---


## Cost Estimation Reference (use these in all phases)

### Claude API pricing (Sonnet tier — verify at console.anthropic.com/settings/billing)
| Metric | Rate |
|--------|------|
| Input tokens | $3.00 per million |
| Output tokens | $15.00 per million |
| Cached input | $0.30 per million |

### Microsoft Fabric Capacity pricing (West Europe / West Central — verify at azure.microsoft.com/pricing)
| SKU | CU | Est. $/hour | Est. $/month (730h) |
|-----|----|-----------|--------------------|
| F2  | 2  | $0.36     | ~$263              |
| F4  | 4  | $0.72     | ~$526              |
| F8  | 8  | $1.44     | ~$1,052            |
| F16 | 16 | $2.88     | ~$2,105            |
| F32 | 32 | $5.77     | ~$4,210            |
| F64 | 64 | $11.54    | ~$8,420            |

### OneLake storage
- $0.023 per GB/month (includes redundancy)
- Delta format compresses raw data ~3–5×; plan for 25–30% of raw source volume

### Microsoft Purview
- Data Map: $0.496 per CU/hour (1 CU per 100GB catalogued, minimum 1 CU)
- Automated scan: $0.30 per vCore/hour
- Typical small platform (~10 workspaces, <1TB): ~$150–$300/month

### Azure supporting resources (Key Vault, Log Analytics, Monitor)
- Typically $50–$150/month for a platform of this scale

### Fabric Pause/Resume
- Capacities can be paused when not in use (dev/test off-hours)
- Assume dev = 8h/day × 22 working days = 176h/month active
- Assume test = 4h/day × 10 test days/month = 40h/month active

---

## Cost Estimation

Update `p3-outputs/cost_estimates.md`. Read `p2-outputs/cost_estimates.md` to carry
forward refined estimates from Phase 2.

### What to count from Phase 3 outputs

Read `_task_summary.md` and the 11 task files:
- **T** = total task count
- **T_crit** = critical path task count (from `build_order.md`)
- **W** = workspace count (from Phase 2)
- **P** = pipeline count (from Phase 2)

### AI cost — Phase 4 (Implementation Scaffolding) forecast

Phase 4 generates one full folder tree per workspace including pipelines, notebooks,
schemas, entities, speckit bundles, CI/CD YAML, and shared infra/governance files.
It is the most token-intensive phase.

| Component | Input tokens | Output tokens |
|-----------|-------------|--------------|
| Base context (P2+P3 outputs loaded) | 80,000 | — |
| Per workspace folder generated | W × 5,000 | W × 9,000 |
| Per pipeline template | P × 2,000 | P × 3,000 |
| Per entity YAML | W × 2 entities avg × 1,500 | W × 2 × 2,000 |
| Per schema JSON | W × 2 tables avg × 1,000 | W × 2 × 1,500 |
| Per speckit bundle (4 files) | W × 2,000 | W × 4,000 |
| infra/ + governance/ + operations/ | 15,000 | 25,000 |
| Shared scripts (5 files) | 5,000 | 10,000 |

**Formula:**
- Input = 80,000 + (W × 5,000) + (P × 2,000) + (W × 2 × 1,500) + (W × 2 × 1,000) + (W × 2,000) + 15,000 + 5,000
- Output = (W × 9,000) + (P × 3,000) + (W × 2 × 2,000) + (W × 2 × 1,500) + (W × 4,000) + 25,000 + 10,000

**Estimated cost** = (Input / 1,000,000 × $3.00) + (Output / 1,000,000 × $15.00)

### MS Fabric cost — refined with build context

Carry the Phase 2 capacity estimates forward unchanged (capacity design is fixed).
Add new line items now visible from the task breakdown:

1. **Initial data load compute**: The first full load of all source systems will
   spike capacity consumption. Estimate: prod capacity running at 100% utilisation
   for `ceil(T_crit / 5)` business days of the critical path = additional
   `ceil(T_crit / 5) × 8h × prod_CU/h × $0.18` one-time cost.

2. **CI/CD pipeline runs (build phase)**: Each workspace deploy runs validate + deploy.
   Estimate W × 10 pipeline runs × 0.5h × F2 equivalent = one-time build cost.

3. **Revised ongoing monthly total**: Carry Phase 2 monthly total forward; note
   that Phase 4 may identify additional workspaces not in Phase 2 — flag any delta.

### Output file format

```markdown
# Cost Estimates — [Project Name]
*Updated: Phase 3 (Task Breakdown)*

## Phase 4 (Implementation Scaffolding) — AI Cost Forecast
| Metric | Value |
|--------|-------|
| Workspaces (W) | [N] |
| Pipelines (P) | [N] |
| Total tasks (T) | [N] |
| Critical path tasks (T_crit) | [N] |
| Estimated input tokens | [N] |
| Estimated output tokens | [N] |
| **Estimated Phase 4 AI cost** | **$[X.XX]** |

## MS Fabric — Refined Monthly Estimate (unchanged from Phase 2)
[Carry Phase 2 table forward verbatim]

## One-Time Implementation Costs
| Item | Basis | Est. cost |
|------|-------|----------|
| Initial full data load | [N] critical-path days × 8h × prod capacity | $[X] |
| CI/CD build runs | [W] workspaces × 10 deploys × 0.5h × F2 | $[X] |
| **Total one-time** | | **$[X]** |

### Delta from Phase 2 estimate
[Any changes and reasons]

## Cumulative AI Cost To Date
| Phase | Input tokens | Output tokens | Cost |
|-------|-------------|--------------|------|
| Phase 1 (actual) | | | |
| Phase 2 (actual) | | | |
| Phase 3 (actual) | [N] | [N] | $[X.XX] |
| **Cumulative** | | | **$[X.XX]** |
```

## Review gate

Present all outputs. Confirm:

- [ ] All Phase 2 workspaces have corresponding WS tasks
- [ ] All Phase 2 source systems have corresponding ING tasks
- [ ] All Phase 2 DQ rules have corresponding DQ tasks
- [ ] Foundation tasks (INF, SEC, GOV, OPS) are sequenced first
- [ ] Critical path is identified
- [ ] No task is missing a "Blocked by" or "Blocks" relationship
- [ ] Cost estimates updated at `p3-outputs/cost_estimates.md` with Phase 4 AI forecast and one-time build costs

When confirmed, state:

> **Phase 3 complete.** Task breakdown locked.
> [N] tasks across [N] disciplines saved to `p3-outputs/`.
> `p3-outputs/cost_estimates.md` adds Phase 4 AI cost forecast and one-time build cost estimates.
> Ready to proceed to **Phase 4 — Implementation Scaffolding**.

---

## Output structure

```
p3-outputs/
├── _task_summary.md
├── cost_estimates.md
├── build_order.md
└── tasks/
    ├── tasks_INF_infrastructure.md
    ├── tasks_GOV_governance.md
    ├── tasks_SEC_security.md
    ├── tasks_OPS_operations.md
    ├── tasks_WS_workspace_setup.md
    ├── tasks_ING_data_ingestion.md
    ├── tasks_DQ_data_quality.md
    ├── tasks_AGG_aggregation.md
    ├── tasks_SEM_semantic_reporting.md
    ├── tasks_CICD_devops.md
    └── tasks_TEST_testing.md
```
