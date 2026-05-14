---
name: fabric-design-technical
description: >
  Fabric SDLC — Phase 2: Technical Design. Use this skill whenever the user wants to
  translate domain discovery outputs into a full technical architecture for a Microsoft
  Fabric platform. Covers: tenant design, workspace design, capacity planning, data
  pipeline design, source system integration, infrastructure, naming conventions, data
  quality rules, security model, operations/monitoring, and semantic layer. Triggers:
  "/fabric-design-technical", "P2", "phase 2", "technical design", "workspace design", "capacity design", "pipeline
  design", "naming conventions", "fabric architecture", "data governance design",
  "security model", "semantic layer design", "design the platform". Always run after
  Phase 1. Produces a dynamic single-page HTML app with one tab per design area, plus
  one MD file per design area and one MD file per workspace.
---

# Phase 2 — Technical Design

## What this phase produces

Phase 2 takes the domain and organisation files from Phase 1 and produces a complete
technical blueprint for the Microsoft Fabric platform. Every design decision here becomes
a direct input to Phase 3 (Task Breakdown) and Phase 4 (Implementation).

**Outputs:**
- `p2-outputs/design/` — one MD file per design area (10 files)
- `p2-outputs/workspaces/` — one MD file per Fabric workspace
- `p2-outputs/architecture.html` — interactive single-page app with one tab per design area
- `p2-outputs/_design_summary.md` — top-level index

---

## Step 0 — Load Phase 1 outputs

Before generating any design, load and read all Phase 1 files:
- `p1-outputs/_discovery_summary.md`
- All `p1-outputs/domains/domain_*.md`
- All `p1-outputs/organisations/org_*.md`

If Phase 1 files are not available, ask the user to either run Phase 1 first or provide
the domain and org information directly before continuing.

---

## Architecture foundation

The platform is always structured in **3 layers**:

```
Layer 3 — Consumption       Semantic models, Power BI reports, dashboards
               ▲                    reads from Layer 2 via DirectLake
Layer 2 — Aggregation       Cross-domain Gold data products, aggregated views
               ▲                    reads from Layer 1 via OneLake shortcuts
Layer 1 — Core & Supporting  Per-domain workspaces (Bronze → Silver medallion)
                              + Supporting workspaces (Ops, Governance, DQ)
```

Every domain from Phase 1 becomes one or more Layer 1 workspaces.
Layer 2 workspaces aggregate data across related domains.
Layer 3 workspaces serve end consumers (Power BI, notebooks, APIs).

Supporting workspaces (Operations, Data Governance) sit alongside Layer 1 as
platform-level services. They are built first.

---

## The 10 design areas

Work through each of the following in order. For each area, generate the MD file and
accumulate the data needed for the HTML visualisation.

---

### Design Area 1 — Tenant Design

File: `design/01_tenant_design.md`

Determine the Fabric tenant topology:

1. **Tenant count**: Single tenant or multiple? Multi-tenant is required when there are
   separate legal entities with strict data isolation needs, different Azure AD directories,
   or regulatory jurisdictions that prohibit data commingling.

2. **Tenant hierarchy** (for multi-tenant):
   - Map each company / org unit from Phase 1 to a tenant
   - Identify shared services that span tenants (cross-tenant Purview, shared capacity admin)
   - Document cross-tenant integration patterns (data copy vs. shortcut vs. API)

3. **Azure AD / Entra ID alignment**: One AAD tenant per Fabric tenant. Map group naming.

4. **Admin boundaries**: Who are the Fabric Capacity Admins and Workspace Admins per tenant?

MD template:
```markdown
# Tenant Design

## Topology
- Tenant count: [1 / N]
- Rationale: [Why single or multi]

## Tenant Map
| Tenant ID | Name | AAD Directory | Companies / Org Units | Regulatory Jurisdiction |
|-----------|------|--------------|----------------------|------------------------|

## Cross-Tenant Integration
| Integration | Method | From Tenant | To Tenant | Notes |
|-------------|--------|------------|----------|-------|

## Admin Roles
| Role | Responsible Party | Tenant |
|------|------------------|--------|
| Capacity Admin | | |
| Workspace Admin | | |
| Purview Admin | | |
```

---

### Design Area 2 — Workspace & Capacity Design

File: `design/02_workspace_capacity_design.md`

**Workspace design rules:**
- One Fabric workspace per domain (Layer 1) — strict single responsibility
- One workspace per cross-domain aggregation group (Layer 2)
- One workspace per consumption audience (Layer 3)
- One workspace for Operations/Monitoring
- One workspace for Data Governance (Purview integration + DQ rulesets)

For each workspace define:
- Workspace name (follows naming convention from Design Area 6)
- Layer (L1 / L2 / L3 / Supporting)
- Domain ownership
- Medallion tiers present (Bronze, Silver, Gold, SemanticModel)
- SLA tier (P1 = 4h / P2 = 8h / P3 = 24h)
- Data classification (Public / Internal / Confidential / Restricted)

**Capacity design:**
- Map workspaces to Fabric capacities (F SKUs)
- Core domain P1 workspaces → dedicated F4+ capacity recommended
- Supporting domain P2 workspaces → shared F2 capacity acceptable
- Generic / P3 workspaces → shared F2 or trial capacity
- For multi-region deployments: one capacity per target Azure region minimum

MD template:
```markdown
# Workspace & Capacity Design

## Workspace Catalog
| WS ID | Workspace Name | Layer | Domain | SLA | Classification | Capacity | Medallion Tiers | Region |
|-------|---------------|-------|--------|-----|----------------|----------|----------------|--------|

## Capacity Map
| Capacity Name | SKU | Region | Assigned Workspaces | Rationale |
|--------------|-----|--------|---------------------|-----------|

## Supporting Workspaces
| Workspace | Purpose | SLA | Classification |
|-----------|---------|-----|----------------|
| ws_ops_monitoring | Pipeline health, SLA tracking, incident management | P1 | Internal |
| ws_gov_data_governance | Purview sync, DQ rulesets, data contracts, naming audit | P1 | Confidential |
```

Also generate a **separate MD file per workspace** (see Workspace MD files section below).

---

### Design Area 3 — Data Pipeline Design

File: `design/03_pipeline_design.md`

For each Layer 1 workspace, define the full pipeline chain:

```
Source System → [Ingest Pipeline] → Bronze Lakehouse
Bronze Lakehouse → [Transform Pipeline] → Silver Lakehouse  (DQ gate applied here)
Silver Lakehouse → [OneLake Shortcut] → Layer 2 Gold Lakehouse
Gold Lakehouse → [Semantic Model] → Layer 3 Power BI
```

For each pipeline specify:
- Pipeline name (follows naming convention)
- Tool: Azure Data Factory (ADF) or Fabric Dataflow Gen2
  - Use ADF for: complex transformations, cross-cloud sources, SLA-critical workloads,
    existing ADF investment
  - Use Dataflow Gen2 for: simpler transformations, self-service, low-latency small datasets
- Trigger type: Scheduled / Event-driven / Manual
- Source → Sink
- Estimated row volume
- Key transformation logic

Metadata-driven pattern: every pipeline reads its configuration from a metadata table
in the workspace's own Lakehouse (source connection, table list, watermark columns,
DQ rule set reference). The metadata is owned by the workspace; aggregates are pushed
to the Operations workspace for monitoring.

MD template:
```markdown
# Pipeline Design

## Pipeline Catalog
| Pipeline ID | Workspace | Name | Tool | Trigger | Source | Sink | Volume | Frequency |
|-------------|-----------|------|------|---------|--------|------|--------|-----------|

## Metadata-Driven Architecture
Each workspace contains a `pipeline_metadata` table:
| Column | Description |
|--------|-------------|
| source_system | Connection string / linked service name |
| source_table | Table or API endpoint |
| sink_lakehouse | Target Lakehouse name |
| sink_table | Target table name |
| watermark_column | Delta/incremental column (e.g. modified_date) |
| watermark_value | Last successfully loaded value |
| dq_ruleset_id | Reference to DQ rule set in Governance workspace |
| is_active | Boolean flag to enable/disable without deleting |

The Operations workspace `ws_ops_monitoring` receives pipeline run status events from
all workspaces (via Fabric Event Streams or scheduled log push) and aggregates them.
```

---

### Design Area 4 — Source Systems & Integration

File: `design/04_source_systems_integration.md`

For every upstream system identified in Phase 1, define the integration point:

- **Connection method**: ADF Linked Service / Fabric native connector / API / SFTP / File drop
- **Authentication**: Service Principal / Managed Identity / Key Vault secret / Basic
- **Network path**: Public internet / Private Endpoint / VPN / ExpressRoute
- **Extract pattern**: Full load / Incremental (watermark) / CDC / Event stream
- **Landing format**: Parquet / Delta / CSV / JSON
- **Schema evolution handling**: Fail / Warn and continue / Auto-evolve

MD template:
```markdown
# Source Systems & Integration

## Integration Catalog
| System | Domain | Connector / Method | Auth | Network Path | Extract Pattern | Format | Schema Evolution |
|--------|--------|-------------------|------|-------------|-----------------|--------|-----------------|

## Connection Secrets
All credentials stored in Azure Key Vault. Workspace Managed Identities used where possible.
Key Vault naming: `kv-[company-code]-fabric-[env]`

## Integration Risks & Mitigations
| System | Risk | Mitigation |
|--------|------|-----------|
```

---

### Design Area 5 — Infrastructure Design

File: `design/05_infrastructure_design.md`

Define the Azure infrastructure surrounding Fabric:

1. **Networking**: Private Endpoints for Fabric (where required by classification level),
   VNet integration, firewall rules, NSG configuration.
2. **Key Vault**: One per environment (dev/test/prod) per tenant. Stores all pipeline secrets.
3. **Azure Monitor / Log Analytics**: Centralised logging for Fabric capacity metrics,
   pipeline logs, and security events.
4. **Azure DevOps / GitHub**: Source control and CI/CD platform (referenced in Phase 4).
5. **Purview account**: One per tenant. Connected to all Fabric workspaces for data cataloguing.
6. **Environments**: dev / test / prod — describe workspace naming and capacity assignment per env.

MD template:
```markdown
# Infrastructure Design

## Environments
| Environment | Purpose | Capacity SKU | Workspace Suffix | Deployment Gate |
|-------------|---------|-------------|-----------------|----------------|
| dev | Development and unit testing | F2 | _dev | Auto-deploy on PR merge |
| test | Integration and UAT | F4 | _test | Manual approval |
| prod | Production | F8+ | (no suffix) | Manual approval + change ticket |

## Azure Resources
| Resource | Name | Region | Purpose |
|----------|------|--------|---------|

## Networking
| Workspace Classification | Network Access | Private Endpoint? | Firewall Rule |
|--------------------------|---------------|-------------------|---------------|
| Public / Internal | Public network | No | Fabric tenant allow-list |
| Confidential | Managed VNet | Yes | No public internet |
| Restricted | Managed VNet + Private Link | Yes | Full network isolation |

## Key Vault
| Vault Name | Environment | Stores | Access Policy |
|-----------|-------------|--------|--------------|
```

---

### Design Area 6 — Naming Conventions

File: `design/06_naming_conventions.md`

Define naming rules for every artefact type. These are enforced by the Governance
workspace's naming audit pipeline in Phase 4.

MD template:
```markdown
# Naming Conventions

## Workspace Names
Pattern: `ws_[layer]_[domain-code]_[short-name]_[env]`
- Layer prefixes: `d` (domain L1), `agg` (aggregation L2), `pbi` (consumption L3),
  `ops` (operations), `gov` (governance)
- Environment suffix: omitted for prod, `_dev` / `_test` for lower envs
- Examples: `ws_d_d1_risk_assessment`, `ws_agg_client_risk`, `ws_pbi_executive`

## Lakehouse Names
Pattern: `lh_[tier]_[domain-code]_[short-name]`
- Tier: `bronze`, `silver`, `gold`
- Examples: `lh_bronze_d1_risk`, `lh_silver_d1_risk`, `lh_gold_client_risk`

## Pipeline Names
Pattern: `pl_[type]_[source-or-domain]_[target]`
- Type: `ingest`, `transform`, `aggregate`, `dq`, `sync`
- Examples: `pl_ingest_erp_risk`, `pl_transform_d1_risk`, `pl_dq_silver_risk`

## Table Names
Pattern: `[domain_code]_[entity]` (snake_case, no spaces)
- Bronze: raw prefix optional — `d1_policy_raw`
- Silver: `d1_policy`
- Gold: `agg_client_policy_summary`

## Column Names
- snake_case throughout
- Surrogate keys: `[entity]_sk` (e.g. `policy_sk`)
- Business keys: `[entity]_id` (e.g. `policy_id`)
- Timestamps: `[event]_at` (e.g. `created_at`, `modified_at`)
- Audit columns mandatory on all silver/gold tables:
  `_ingested_at`, `_source_system`, `_pipeline_run_id`

## Environment Tags
All Azure resources tagged: `environment`, `domain`, `owner`, `cost-centre`
```

---

### Design Area 7 — Data Governance & Quality Rules

File: `design/07_data_governance_dq.md`

Define the DQ framework and governance model:

1. **DQ dimensions**: Completeness, Validity, Uniqueness, Timeliness, Consistency, Referential Integrity
2. **DQ gate placement**: DQ gates run at Bronze→Silver (mandatory) and Silver→Gold (mandatory)
3. **Rule severity levels**:
   - `hard_block` — pipeline fails, no data promoted, alert fired
   - `quarantine` — failing rows removed to quarantine table, rest promoted, warning fired
   - `warn` — rule logged, data promoted, dashboard flagged
   - `monitor` — logged only, no action

4. **Rule definition format** (YAML, stored in Governance workspace):
```yaml
rule_id: DQ-[DOMAIN-CODE]-[NNN]
domain: [domain code]
workspace: [workspace name]
target_table: [silver or gold table name]
column: [column name or "*" for table-level]
check_type: not_null | unique | regex | range | referential | custom_sql
severity: hard_block | quarantine | warn | monitor
action_on_fail: block | quarantine | warn | log
version: "1.0"
effective_from: "[YYYY-MM-DD]"
```

5. **Data contracts**: Every gold layer table has a versioned YAML data contract
   enforced by the Governance workspace. Schema drift triggers an alert.

6. **Purview integration**: All Fabric workspaces registered as sources in Microsoft Purview.
   Automated lineage captured via Fabric-Purview native connector. Sensitivity labels
   applied to all Restricted and Confidential tables.

MD template:
```markdown
# Data Governance & Quality Rules

## DQ Framework
| Layer | Gate | Severity Levels Applied | Action on Failure |
|-------|------|------------------------|-------------------|

## DQ Rule Catalogue (initial set)
| Rule ID | Domain | Table | Column | Check Type | Severity | Action |
|---------|--------|-------|--------|-----------|----------|--------|

## Data Contract Standard
Contract file: `data_contract_[table].yaml` stored in each workspace repo.
Validated nightly by Governance workspace pipeline.

## Purview Setup
| Workspace | Purview Source Registered | Sensitivity Label | Lineage Auto-Captured |
|-----------|--------------------------|-------------------|----------------------|
```

---

### Design Area 8 — Security Design

File: `design/08_security_design.md`

1. **AAD group structure**: One group per workspace role (Admin, Member, Contributor, Viewer)
   named `grp-fabric-[workspace-short-name]-[role]`
2. **Classification label enforcement**: All tables/lakehouses at Confidential or Restricted
   must have Purview sensitivity labels applied. Labels enforced via DLP policy.
3. **Row-Level Security (RLS)**: Define RLS on any gold-layer semantic model that serves
   data to external parties or different org units.
4. **Service principals**: All pipeline runs use a dedicated Service Principal per workspace.
   No interactive credentials in pipelines.
5. **Private endpoints**: Mandatory for Confidential/Restricted workspaces (see Infra).
6. **Audit logging**: All data access events logged to Operations workspace. Reviewed weekly.

MD template:
```markdown
# Security Design

## AAD Group Structure
| Group Name Pattern | Role | Workspace Type | Assigned To |
|-------------------|------|---------------|-------------|

## Classification Label Matrix
| Classification | Workspace Examples | Label Applied | DLP Policy | Private Endpoint Required |
|---------------|-------------------|--------------|-----------|--------------------------|

## Row-Level Security
| Workspace | Semantic Model | RLS Dimension | Enforced By | Notes |
|-----------|---------------|--------------|-------------|-------|

## Service Principals
| SP Name | Used By | Permissions | Secret Rotation |
|---------|---------|------------|----------------|
```

---

### Design Area 9 — Operations & Monitoring

File: `design/09_operations_monitoring.md`

The `ws_ops_monitoring` workspace is the single pane of glass for platform health.

1. **SLA tiers**:
   - P1 (Core domain workspaces): data available within 4 hours of source refresh
   - P2 (Supporting domain workspaces): within 8 hours
   - P3 (Generic / consumption workspaces): within 24 hours

2. **Log schema**: Every pipeline in every workspace emits a structured log event:
```json
{
  "run_id": "uuid",
  "workspace_name": "ws_d_d1_risk_assessment",
  "pipeline_name": "pl_ingest_erp_risk",
  "layer": "L1",
  "status": "success | failed | warning",
  "started_at": "ISO8601",
  "duration_seconds": 0,
  "rows_ingested": 0,
  "rows_rejected": 0,
  "dq_gate_passed": true,
  "error_code": null,
  "sla_tier": "P1"
}
```

3. **Alerting**: P1 SLA breaches trigger immediate Teams/email alert. P2 daily digest.
4. **Operations dashboard**: Power BI report in `ws_ops_monitoring` showing pipeline health,
   SLA compliance %, DQ score trends, active incidents.

MD template:
```markdown
# Operations & Monitoring Design

## SLA Definition
| Tier | Domains | Max Latency | Alert on Breach | Escalation |
|------|---------|------------|----------------|-----------|

## Monitoring Workspace: ws_ops_monitoring
| Fabric Item | Purpose |
|-------------|---------|
| lh_bronze_ops_logs | Raw pipeline log events |
| lh_silver_ops_logs | Parsed, enriched log data |
| lh_silver_ops_sla | SLA compliance tracking |
| pl_ingest_pipeline_logs | Ingests log events from all workspaces |
| pl_evaluate_sla | Hourly SLA evaluation per workspace |
| ds_ops_semantic | DirectLake semantic model |
| rpt_ops_dashboard | Pipeline health and SLA Power BI report |

## Alerting Rules
| Alert | Trigger | Channel | Audience |
|-------|---------|---------|---------|
```

---

### Design Area 10 — Semantic Layer

File: `design/10_semantic_layer.md`

Define semantic models (Power BI datasets) at Layer 3:

1. **One semantic model per consumption audience** (e.g. Executive, Finance, Operations)
2. **DirectLake mode** — all models connect directly to Layer 2 Gold lakehouses, no import
3. **Certified datasets**: only models published via CI/CD pipeline are marked Certified
4. **Measures & KPIs**: list the top-level business metrics each model must expose
5. **Composite model rules**: a single report may combine multiple semantic models only
   if the models are in the same Layer 3 workspace

MD template:
```markdown
# Semantic Layer Design

## Semantic Model Catalogue
| Model Name | Workspace | Source (L2 workspace) | Audience | KPIs / Key Measures | RLS? | Certified? |
|-----------|-----------|----------------------|---------|---------------------|------|-----------|

## KPI Definitions
| KPI Name | Calculation | Source Tables | Owner | Refresh Frequency |
|----------|------------|--------------|-------|------------------|
```

---

## Workspace MD files

For **every workspace** identified in Design Area 2, generate a file:
`workspaces/ws_[name].md`

```markdown
# Workspace: [ws_name]

## Overview
| Field | Value |
|-------|-------|
| Workspace Name | ws_[layer]_[domain]_[name] |
| Layer | L1 / L2 / L3 / Supporting |
| Domain | [Domain name and code] |
| SLA Tier | P1 / P2 / P3 |
| Data Classification | Public / Internal / Confidential / Restricted |
| Capacity | [F SKU and capacity name] |
| Region | [Azure region] |
| Owning Team | [Team name] |
| Service Principal | sp-fabric-[ws-short-name] |

## Medallion Architecture
| Tier | Lakehouse Name | Purpose |
|------|---------------|---------|
| Bronze | lh_bronze_[name] | Raw ingest, immutable, schema-on-read |
| Silver | lh_silver_[name] | Cleansed, typed, DQ-gated, business keys resolved |
| Gold | lh_gold_[name] | (L2 workspaces only) Cross-domain aggregated data product |

## Data Contract (Gold layer — L2 workspaces)
Contract file: `data_contract_[table].yaml`
Enforced by: Governance workspace nightly validation pipeline
Schema drift action: Alert + pipeline pause

## Pipelines
| Pipeline Name | Tool | Trigger | Source → Sink | Volume |
|--------------|------|---------|--------------|--------|

## Upstream Links (OneLake Shortcuts)
| Shortcut Name | Source Workspace | Source Lakehouse | Source Table Path |
|--------------|-----------------|-----------------|------------------|

## Downstream Consumers
| Consumer Workspace | Layer | Access Method |
|-------------------|-------|--------------|

## DQ Rules Applied
| Rule ID | Table | Check | Severity |
|---------|-------|-------|---------|

## Security
| AAD Group | Role | Members |
|-----------|------|---------|
```

---

## HTML Single-Page Application

After generating all MD files, create `architecture.html` — a fully self-contained
single-page app with **10 tabs**, one per design area. Requirements:

- No external dependencies (fully self-contained HTML + embedded CSS + JS)
- Tab navigation at the top (tabs clickable, active tab highlighted)
- Each tab renders the content of its design area in a structured, readable layout
- Use cards, tables, and diagrams (SVG-based, not images) for visualisations
- Colour coding: Core domains = deep blue, Supporting = amber, Generic = grey,
  Operations = teal, Governance = purple
- Architecture Overview tab (Tab 1) shows the 3-layer diagram as an animated SVG:
  - L1 workspace cards on the left column, L2 in centre, L3 on right
  - Arrows showing data flow (L1 → L2 → L3)
  - Supporting workspaces shown in a separate column below
  - Click any workspace card: highlights its upstream and downstream connections
- Workspace tab (Tab 2): filterable/sortable table of all workspaces; click row expands
  to show medallion layout and pipeline list
- All other tabs render their MD content as structured HTML tables and text
- Dark header, white content area, professional enterprise styling

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

## Step 11a — Cost Estimation

Update `p2-outputs/cost_estimates.md`. Read `p1-outputs/cost_estimates.md` to carry
forward the prior estimates and show the delta — what changed and why.

### What to count from Phase 2 outputs

Read `design/02_workspace_capacity_design.md` for the authoritative workspace catalogue:
- **W** = actual total workspace count
- **W_L1** = L1 domain workspaces
- **W_L2** = L2 aggregation workspaces
- **W_L3** = L3 consumption workspaces
- **Capacities** = list of (capacity_name, SKU, region, assigned_workspaces)
- **Envs** = environments defined in `design/05_infrastructure_design.md`

Read `design/03_pipeline_design.md`:
- **P** = total pipeline count across all workspaces

Read `design/07_data_governance_dq.md`:
- **Q** = total DQ rule count

### AI cost — Phase 3 (Task Breakdown) forecast

Phase 3 generates 11 discipline task files + build_order.md + task_summary.md.
Tasks are generated from workspace designs, pipeline catalogue, and DQ rules.

| Component | Input tokens | Output tokens |
|-----------|-------------|--------------|
| Base context (P2 outputs loaded) | 60,000 | — |
| Per workspace processed | W × 3,000 | — |
| Per pipeline processed | P × 1,500 | — |
| Per DQ rule processed | Q × 500 | — |
| Task file generation (11 disciplines) | 10,000 | 11 × 4,000 |
| Build order + summary | 5,000 | 6,000 |

**Formula:**
- Input = 60,000 + (W × 3,000) + (P × 1,500) + (Q × 500) + 10,000 + 5,000
- Output = (11 × 4,000) + 6,000

**Estimated cost** = (Input / 1,000,000 × $3.00) + (Output / 1,000,000 × $15.00)

### MS Fabric cost — refined from workspace catalogue

Now we have actual capacity assignments. Calculate precisely:

For each capacity in the workspace catalogue:
- **Prod cost** = CU_count × 730 hours × $0.18/CU-hour
- **Dev cost** = CU_count_dev × 176 hours × $0.18/CU-hour  (dev typically 1 SKU tier lower)
- **Test cost** = CU_count_test × 40 hours × $0.18/CU-hour (test typically same as prod)

Storage estimate (now refined from source system integration in Design Area 4):
- Sum estimated data volumes per source system × 0.25 (Delta compression) × $0.023/GB/month
- Add 20% headroom for intermediate/quarantine tables

Purview: now we know workspace count → refine:
- Data Map: ceil(total_data_GB / 100) CU × $0.496/CU/hour × 730h
- Scanning: W × 1 scan/day × 0.1 vCore/h × $0.30/vCore-h × 730h

Flag any significant variances from Phase 1 estimate with a brief explanation.

### Output file format

```markdown
# Cost Estimates — [Project Name]
*Updated: Phase 2 (Technical Design)*

## Phase 3 (Task Breakdown) — AI Cost Forecast
| Metric | Value |
|--------|-------|
| Workspaces (W) | [N] (L1:[N] L2:[N] L3:[N] Supporting:[N]) |
| Pipelines (P) | [N] |
| DQ rules (Q) | [N] |
| Estimated input tokens | [N] |
| Estimated output tokens | [N] |
| **Estimated Phase 3 AI cost** | **$[X.XX]** |

## MS Fabric — Refined Monthly Estimate
| Capacity | SKU | Env | Hours/month | $/month |
|----------|-----|-----|------------|---------|
| [cap_name] | F[N] | prod | 730 | $[X] |
| [cap_name]_dev | F[N] | dev | 176 | $[X] |
| [cap_name]_test | F[N] | test | 40 | $[X] |
| OneLake storage | — | all | — | $[X] |
| Purview | — | all | — | $[X] |
| Azure support resources | — | all | — | $[X] |
| **Total revised estimate** | | | | **$[X]** |

### Delta from Phase 1 estimate
| Item | Phase 1 | Phase 2 | Change | Reason |
|------|---------|---------|--------|--------|

> Variance from Phase 1: [+/-X%]. [Brief explanation]

## Cumulative AI Cost To Date
| Phase | Input tokens | Output tokens | Cost |
|-------|-------------|--------------|------|
| Phase 1 (actual) | [from p1 file] | [from p1 file] | [from p1 file] |
| Phase 2 (actual) | [N] | [N] | $[X.XX] |
| **Cumulative** | | | **$[X.XX]** |
```

## Step 11 — Review gate

Present all design outputs. Confirm:

- [ ] Every Phase 1 domain has at least one workspace
- [ ] All 3 layers are represented (L1 / L2 / L3)
- [ ] Operations workspace and Governance workspace are defined
- [ ] Every workspace has a capacity, SLA tier, and classification
- [ ] Naming convention covers all artefact types
- [ ] DQ gates are defined for Bronze→Silver and Silver→Gold
- [ ] Security groups and service principals are named
- [ ] Semantic models are defined for all consumption audiences
- [ ] Cost estimates updated at `p2-outputs/cost_estimates.md` with refined capacity costs

When confirmed, state:

> **Phase 2 complete.** Technical design locked.
> [N] workspace MD files, 10 design area MD files, and the architecture SPA saved to `p2-outputs/`.
> `p2-outputs/cost_estimates.md` refined Fabric monthly cost and Phase 3 AI cost forecast.
> Ready to proceed to **Phase 3 — Task Breakdown**.

---

## Output structure

```
p2-outputs/
├── _design_summary.md
├── cost_estimates.md
├── architecture.html                    ← interactive SPA
├── design/
│   ├── 01_tenant_design.md
│   ├── 02_workspace_capacity_design.md
│   ├── 03_pipeline_design.md
│   ├── 04_source_systems_integration.md
│   ├── 05_infrastructure_design.md
│   ├── 06_naming_conventions.md
│   ├── 07_data_governance_dq.md
│   ├── 08_security_design.md
│   ├── 09_operations_monitoring.md
│   └── 10_semantic_layer.md
└── workspaces/
    ├── ws_d_d1_[name].md
    ├── ws_d_d2_[name].md
    ├── ws_agg_[name].md
    ├── ws_pbi_[name].md
    ├── ws_ops_monitoring.md
    └── ws_gov_data_governance.md
```
