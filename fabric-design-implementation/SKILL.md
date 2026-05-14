---
name: fabric-design-implementation
description: >
  Fabric SDLC — Phase 4: Implementation Scaffolding. Use this skill whenever the user wants
  to generate the GitHub/Azure DevOps repository structure, CI/CD pipeline definitions,
  ETL pipeline templates, metadata-driven pipeline configurations, data governance workspace
  setup, or Purview integration scaffolding for a Microsoft Fabric platform. Triggers:
  "/fabric-design-implementation", "P4", "phase 4", "generate repo structure", "scaffold the implementation", "github folders",
  "CI/CD pipelines", "ADF pipelines", "dataflow pipelines", "metadata pipeline", "implementation
  scaffolding", "generate the folders", "create the repos", "purview integration", "speckit".
  Always run after Phase 3. Produces ready-to-use folder structures, pipeline JSON templates,
  CI/CD YAML files, and infrastructure-as-code stubs for every workspace.
---

# Phase 4 — Implementation Scaffolding

## What this phase produces

Phase 4 generates the complete repository and file scaffolding that the engineering team
uses to build the platform. Every workspace gets its own repository folder. Every pipeline
gets a template. Every workspace gets a CI/CD pipeline definition.

This phase does not deploy anything to Azure or Fabric — it creates the source code and
configuration files that the CI/CD pipelines will deploy.

**Outputs:**
- `p4-outputs/` — the full repository folder tree, ready for GitHub/Azure DevOps
- One folder per workspace: `workspaces/[ws-name]/`
- `infra/` — shared infrastructure-as-code
- `governance/` — Governance workspace files + Purview integration
- `operations/` — Operations workspace files
- `p4-outputs/_scaffold_summary.md` — index of everything generated

---

## Step 0 — Load prior phase outputs

Load and read:
- `p2-outputs/design/02_workspace_capacity_design.md` (workspace list)
- `p2-outputs/design/03_pipeline_design.md` (pipeline catalogue)
- `p2-outputs/design/04_source_systems_integration.md` (source systems)
- `p2-outputs/design/05_infrastructure_design.md` (infra design)
- `p2-outputs/design/06_naming_conventions.md` (naming rules)
- `p2-outputs/design/07_data_governance_dq.md` (DQ rules)
- `p2-outputs/workspaces/ws_*.md` (all workspace files)
- `p3-outputs/build_order.md` (task sequence, for CI/CD stage ordering)

If prior phase files are not available, ask the user to describe the workspace list and
pipeline catalogue directly before continuing.

---

## Repository structure

The full output folder structure is:

```
p4-outputs/
├── README.md                          ← platform overview, how to deploy
├── infra/                             ← shared Azure infrastructure
│   ├── bicep/                         ← Bicep templates for Azure resources
│   ├── scripts/                       ← PowerShell / bash helper scripts
│   └── environments/                  ← per-environment variable files
│       ├── dev.tfvars (or dev.json)
│       ├── test.tfvars
│       └── prod.tfvars
├── governance/                        ← Governance workspace + Purview
│   ├── workspace/                     ← Fabric item definitions
│   ├── dq_rules/                      ← DQ rule YAML catalogue
│   ├── data_contracts/                ← Gold table contract YAML files
│   ├── purview/                       ← Purview integration scripts
│   └── cicd/                          ← CI/CD pipeline for governance workspace
├── operations/                        ← Operations workspace
│   ├── workspace/
│   ├── pipelines/
│   └── cicd/
└── workspaces/
    ├── ws_[name_1]/                   ← one folder per domain/agg/consumption workspace
    ├── ws_[name_2]/
    └── ...
```

Generate every folder and every file described below.

---

## Section 1 — Infrastructure (`infra/`)

### `infra/README.md`
Explains what Azure resources are provisioned, the environments, and how to run the
Bicep deployment.

### `infra/bicep/main.bicep`
Top-level Bicep file that orchestrates:
- Resource Group creation per environment
- Key Vault deployment (one per env)
- Log Analytics Workspace
- Fabric Capacity (via REST API call — Fabric capacities are provisioned via ARM/REST)
- Purview Account

Use modules: `modules/keyvault.bicep`, `modules/loganalytics.bicep`,
`modules/fabric_capacity.bicep`, `modules/purview.bicep`

### `infra/bicep/modules/`
One module file per resource type. Each module accepts `env`, `location`, `name_prefix`,
and `tags` parameters.

### `infra/environments/dev.json`, `test.json`, `prod.json`
Parameter files:
```json
{
  "environment": "dev",
  "location": "[primary Azure region from Design Area 5]",
  "fabric_sku": "F2",
  "name_prefix": "[org-code]-fabric",
  "tags": {
    "environment": "dev",
    "managed_by": "iac",
    "project": "fabric-analytics-platform"
  }
}
```

### `infra/scripts/deploy_infra.sh`
Bash script that:
1. Logs into Azure (`az login --service-principal`)
2. Deploys Bicep with the correct parameter file
3. Stores output values (Key Vault URI, Log Analytics workspace ID) in a local `.env` file
4. Prints a deployment summary

### `infra/scripts/create_fabric_workspaces.ps1`
PowerShell script that calls the Fabric REST API to:
1. Create all workspaces from the workspace catalogue (Design Area 2)
2. Assign each workspace to the correct capacity
3. Add AAD groups with the correct roles (Admin / Member / Contributor / Viewer)
4. Print a creation report

Input: reads workspace definitions from `infra/workspace_catalogue.json` (generated
from Design Area 2 workspace table).

### `infra/workspace_catalogue.json`
Machine-readable version of the workspace catalogue:
```json
[
  {
    "workspace_name": "ws_d_d1_[name]",
    "layer": "L1",
    "domain_code": "D1",
    "sla_tier": "P1",
    "classification": "Confidential",
    "capacity_name": "[capacity-name]",
    "region": "[Azure region]",
    "aad_groups": {
      "admin": "grp-fabric-[ws-short-name]-admin",
      "member": "grp-fabric-[ws-short-name]-member",
      "contributor": "grp-fabric-[ws-short-name]-contributor",
      "viewer": "grp-fabric-[ws-short-name]-viewer"
    }
  }
]
```

Generate one entry per workspace from the workspace catalogue.

---

## Section 2 — Per-Workspace folder (`workspaces/ws_[name]/`)

Generate one folder per workspace. Each folder has this structure:

```
workspaces/ws_[name]/
├── README.md
├── metadata/
│   └── pipeline_metadata.json
├── entities/
│   └── entity_[domain]_[name].yaml     (one per logical entity — all workspaces)
├── schemas/
│   └── schema_[table].json             (Silver/Gold Delta schema definitions)
├── pipelines/
│   ├── pl_ingest_[source].json         (L1 workspaces only — ADF or Dataflow template)
│   ├── pl_transform_[name].json        (L1 workspaces only)
│   └── pl_aggregate_[name].json        (L2 workspaces only)
├── notebooks/
│   └── nb_dq_[name].py                 (DQ gate notebook — L1 workspaces)
├── data_contracts/
│   └── contract_[table].yaml           (L2 workspaces — one per Gold table)
├── speckit/                             ← GitHub Speckit-compatible handover bundle
│   ├── spec.md                         ← Speckit spec: user stories, acceptance criteria, constraints
│   ├── plan.md                         ← Speckit plan: tech decisions, architecture, phase gates
│   ├── data-model.md                   ← entity/schema definitions, PII fields, business keys
│   ├── research.md                     ← library comparisons, pattern choices, open questions
│   ├── contracts/                      ← data contracts, DQ rule contracts, pipeline event contracts
│   │   ├── contract_[table].yaml
│   │   └── dq_contract_[ws].yaml
│   ├── quickstart.md                   ← key validation scenarios and smoke-test guide
│   └── tasks.md                        ← executable task list with parallelisation markers [P]
└── cicd/
    ├── deploy.yml                       ← CI/CD pipeline (GitHub Actions or ADO)
    └── validate.yml                     ← PR validation pipeline
```

### `workspaces/ws_[name]/README.md`
```markdown
# Workspace: [ws_name]

## Overview
[Layer, domain, SLA tier, classification — from workspace MD file]

## Getting Started
1. Deploy infrastructure first: `infra/scripts/deploy_infra.sh`
2. Create workspace: `infra/scripts/create_fabric_workspaces.ps1 --workspace [ws_name]`
3. Deploy pipelines: trigger the CI/CD pipeline `cicd/deploy.yml`

## Pipelines
[List of pipelines in this workspace with brief descriptions]

## Data Flow
[Source systems → Bronze → Silver (with DQ gate) → OneLake shortcut to L2]

## Contacts
| Role | Name |
|------|------|
| Domain Owner | [from workspace MD] |
| Delivery Lead | |
```

### `workspaces/ws_[name]/metadata/pipeline_metadata.json`
The metadata table seed file — loaded into the workspace Lakehouse on first deploy:
```json
[
  {
    "metadata_id": "uuid",
    "workspace_name": "[ws_name]",
    "source_system": "[source system name]",
    "source_table": "[table or endpoint]",
    "sink_lakehouse": "lh_bronze_[name]",
    "sink_table": "[target table name]",
    "watermark_column": "[e.g. modified_date]",
    "watermark_value": "1900-01-01",
    "dq_ruleset_id": "[from DQ rule catalogue]",
    "extract_pattern": "incremental",
    "is_active": true,
    "created_at": "[date]",
    "notes": ""
  }
]
```
Generate one entry per source table / pipeline defined in Design Area 3 for this workspace.

### `workspaces/ws_[name]/pipelines/pl_ingest_[source].json`
ADF pipeline JSON template for metadata-driven ingestion. The pipeline:
1. Reads the `pipeline_metadata` table (Filter: `is_active = true`)
2. ForEach loop over active metadata rows
3. For each row: runs a Copy Activity (source = source_system / source_table,
   sink = Bronze Lakehouse Delta table)
4. After copy: updates `watermark_value` in metadata table
5. After copy: emits pipeline log event to Operations workspace (HTTP activity or
   Fabric Event Stream)

Parameterised with: `workspace_name`, `environment`, `keyvault_uri`

### `workspaces/ws_[name]/pipelines/pl_transform_[name].json`
ADF or Dataflow Gen2 template for Bronze→Silver transformation:
1. Reads from Bronze Lakehouse
2. Applies type casting, null handling, business key resolution
3. Calls DQ gate notebook (`nb_dq_[name].py`)
4. Writes clean rows to Silver Lakehouse Delta table
5. Writes rejected rows to Silver quarantine table
6. Pushes DQ score results to Governance workspace via HTTP

For **Dataflow Gen2** instead of ADF: generate a Power Query M script stub with
documented transformation steps, not ADF JSON.

### `workspaces/ws_[name]/notebooks/nb_dq_[name].py`
PySpark notebook for DQ gate execution:
```python
# DQ Gate Notebook — [workspace name]
# Reads DQ rules from Governance workspace via OneLake shortcut
# Evaluates rules against Silver staging table
# Returns: clean_df, quarantine_df, dq_score_dict

from pyspark.sql import SparkSession
from pyspark.sql.functions import col, lit, current_timestamp
import json

spark = SparkSession.builder.getOrCreate()

# Load DQ rules for this workspace from Governance lakehouse shortcut
dq_rules = spark.read.format("delta").load(
    "abfss://ws_gov_data_governance@onelake.dfs.fabric.microsoft.com/"
    "lh_silver_gov_rules/Tables/dq_rules_silver"
).filter(col("workspace_name") == "[ws_name]")

# Load staging table
staging_df = spark.read.format("delta").load(
    "Tables/[bronze_table_name]"
)

# Apply rules (expand this per rule type)
clean_rows = []
quarantine_rows = []
dq_scores = {}

# TODO: implement rule evaluation loop per rule_id
# For each rule: evaluate check_type, apply action_on_fail

# Push DQ scores to Governance workspace
# (via REST call to Governance pipeline endpoint)
```

### `workspaces/ws_[name]/entities/entity_[domain]_[name].yaml`
One YAML file per logical domain entity (all workspace types). Captures the full
domain definition that engineers need before writing a single line of code:

```yaml
entity_id: entity-[domain]-[name]          # e.g. entity-d1-worker
domain: [domain_code]                       # e.g. D1
workspace: [ws_name]
source_system: [system_name]
business_key: [field_name]                  # primary natural key
fields:
  - name: [field_name]
    type: [string | integer | decimal | date | timestamp | boolean]
    nullable: true | false
    pii: true | false
    classification: [Public | Internal | Confidential | Restricted]
    description: "[What this field contains]"
    dq_rules:
      - [DQ-RULE-ID]
relationships:
  - entity: [related_entity_name]
    type: [many_to_one | one_to_many | one_to_one]
    join_key: [field_name]
    workspace: [source_workspace]           # if cross-workspace via OneLake shortcut
gdpr:
  consent_field: [field_name]              # GDPR Art.6 consent flag field
  erasure_supported: true | false          # GDPR Art.17
  data_residency_country: [ISO-2]          # e.g. DE, FR
  erasure_note: "[Retention exception if any]"
aueg:                                       # DE entities only — omit for non-DE
  aueg_relevant: true | false
  aueg_field: [field_name]                 # field tracking cumulative months
  regulatory_reference: "AÜG §1(1b)"
retention_years: [N]
```

Generate one entity file per logical domain entity derived from the workspace MD
file's "Key Silver Tables" section. For CO2/CO3/CO4 entities that mirror CO1,
adapt the regulatory body references (BfDI → CNIL for FR, AÜG → Code du Travail
for FR) and data residency country accordingly.

### `workspaces/ws_[name]/schemas/schema_[table].json`
One JSON file per Silver or Gold Delta table. Provides the exact schema engineers
apply when creating Lakehouse tables — no guessing at field types or PII scope:

```json
{
  "table": "[table_name]",
  "lakehouse": "lh_[tier]_[ws_short_name]",
  "format": "delta",
  "layer": "silver | gold",
  "schema": [
    {
      "name": "[field_name]",
      "type": "string | integer | decimal(p,s) | date | timestamp | boolean | long",
      "nullable": true,
      "pii": false,
      "classification": "Internal | Confidential | Restricted | Public",
      "comment": "[What this field contains]"
    }
  ],
  "partition_by": ["[partition_column]"],
  "z_order_by": ["[zorder_column_1]", "[zorder_column_2]"],
  "table_properties": {
    "delta.autoOptimize.optimizeWrite": "true",
    "delta.autoOptimize.autoCompact": "true"
  }
}
```

Rules for schema generation:
- Every Silver/Gold table must include the three audit columns:
  `_ingested_at` (timestamp), `_source_system` (string), `_pipeline_run_id` (string)
- Surrogate keys (`[entity]_sk`) are always `string`, `nullable: false`, `pii: false`
- PII fields: use `"delta.columnMapping.mode": "name"` in table_properties to
  enable column-level encryption/masking downstream
- Partition by entity_code for L1 workspaces; by date + entity for L2 Gold tables
- For Restricted workspaces, add `"delta.columnMapping.mode": "name"` to table_properties

Generate one schema file per Silver table (L1 workspaces) and per Gold table
(L2 workspaces). Supporting workspaces (governance, operations) get schemas for
their Silver operational tables (dq_rules, pipeline_logs, sla_compliance).

### `workspaces/ws_[name]/data_contracts/contract_[table].yaml`
(L2 workspaces only — one per Gold table):
```yaml
contract_id: contract-[ws-short-name]-[table]
version: "1.0"
workspace: [ws_name]
table: [gold_table_name]
layer: gold
effective_from: "[YYYY-MM-DD]"
owner: [domain owner team]
columns:
  - name: [column_name]
    type: [string | integer | decimal | timestamp | boolean]
    nullable: true | false
    description: "[What this column contains]"
    pii: true | false
    classification: [Public | Internal | Confidential | Restricted]
schema_drift_action: alert_and_pause  # alert_and_pause | alert_only | ignore
consumers:
  - workspace: [L3 workspace name]
    model: [semantic model name]
```
Generate the column list from the workspace MD file's Gold table definitions.

### `workspaces/ws_[name]/speckit/` — GitHub Speckit-compatible handover bundle

The speckit folder is generated for **every workspace** (L1, L2, L3, and supporting).
Its structure mirrors [GitHub Speckit](https://github.com/github/spec-kit) — the
Spec-Driven Development toolkit — so that the AI agent or engineer working on each
workspace can open the folder and build directly from the spec without hunting through
Phase 2 and Phase 3 outputs. The seven artefacts are:

---

**`spec.md`** — the Speckit feature specification for this workspace. Written in
Spec-Driven Development (SDD) format: WHAT the workspace delivers and WHY, not HOW
to implement it. Mark unresolved design questions with `[NEEDS CLARIFICATION: question]`.

Structure:
```markdown
# Spec: [ws_name]

## Overview
[One-paragraph description of what this workspace does and its role in the platform]

## User Stories
- As a [role], I want [capability] so that [business value]

## Acceptance Criteria
- [ ] [Measurable, testable criterion — e.g. "Silver table refreshes within P2 SLA (24h)"]

## Non-Functional Requirements
- Data classification: [Public | Internal | Confidential | Restricted]
- SLA tier: [P1 | P2 | P3]
- Regulatory: [GDPR / AÜG / none]

## Constraints & Dependencies
- Upstream source systems: [list]
- Downstream consumers: [list]
- [NEEDS CLARIFICATION: any open questions]

## Out of Scope
- [Explicit exclusions]
```

---

**`plan.md`** — the Speckit implementation plan. Translates the spec into concrete
technical decisions with documented rationale. Each tech choice must trace back to
a specific acceptance criterion or constraint.

Structure:
```markdown
# Plan: [ws_name]

## Phase -1: Pre-Implementation Gates

### Simplicity Gate
- [ ] Workspace uses standard Medallion layers (Bronze/Silver/Gold) — no custom layers?
- [ ] No future-proofing: only tables/pipelines in scope are built now?

### Integration-First Gate
- [ ] Data contracts defined (see contracts/) before pipeline code written?
- [ ] DQ rules defined before Notebook code written?

## Architecture Decisions
| Decision | Choice | Rationale | Traces to |
|----------|--------|-----------|-----------|

## Fabric Items to Create
| Item type | Name | Purpose |
|-----------|------|---------|

## Pipeline Design
[Ingest → Bronze → DQ gate → Silver → (OneLake shortcut for L2) flow]

## Security Model
[AAD groups, roles, RLS predicates if applicable]

## Complexity Tracking
[Document any justified deviations from the Simplicity Gate]
```

---

**`data-model.md`** — the full entity and schema model for this workspace. Every
Delta table is listed with its fields, types, keys, and PII scope.

Structure:
```markdown
# Data Model: [ws_name]

## Entities

### [entity_name]
- **Lakehouse**: lh_[tier]_[ws_short_name]
- **Layer**: bronze | silver | gold
- **Partition by**: [column]
- **Z-order by**: [columns]
- **Surrogate key**: [field]_sk
- **Business keys**: [field(s)]
- **PII fields** (GDPR Art.17 erasure scope): [fields]

| Field | Type | Nullable | Role | Description |
|-------|------|----------|------|-------------|
| [name] | string\|integer\|decimal\|date\|timestamp\|boolean | true\|false | PK\|BK\|FK\|ATTR\|MEASURE\|AUDIT | [description] |

## Key Relationships
- [entity_A] → [entity_B] ([cardinality], join key: [field])

## OneLake Shortcut Dependencies (L2 workspaces)
- [shortcut_name] → onelake://[l1_workspace]/[lakehouse]/Tables/[table]
```

---

**`research.md`** — technical research notes captured during Phase 2 and Phase 4.
Includes library options considered, patterns evaluated, and the rationale for choices
made. Written to answer the question an engineer will ask when they pick up the spec:
"Why did we do it this way?"

Structure:
```markdown
# Research: [ws_name]

## Pipeline Pattern
[Why metadata-driven ingestion was chosen; alternatives considered]

## DQ Gate Approach
[Why PySpark notebook gate vs inline Dataflow validation]

## Scheduling Strategy
[SLA tier → trigger frequency mapping and why]

## Open Questions
- [Any unresolved technical decisions — flag for architect review]
```

---

**`contracts/`** — versioned data and DQ contracts for this workspace.

**`contracts/contract_[table].yaml`** — one file per Gold table (L2 workspaces) or
per Silver table that is consumed cross-workspace (L1 workspaces):

```yaml
contract_id: contract-[ws-short-name]-[table]
version: "1.0"
workspace: [ws_name]
table: [table_name]
layer: silver | gold
effective_from: "[YYYY-MM-DD]"
owner: [domain owner team]
grain: "[one row per ...]"
columns:
  - name: [column_name]
    type: string | integer | decimal | timestamp | boolean
    nullable: true | false
    role: PK | BK | FK | ATTR | MEASURE | AUDIT
    description: "[What this column contains]"
    pii: true | false
    classification: Public | Internal | Confidential | Restricted
schema_drift_action: alert_and_pause | alert_only | ignore
consumers:
  - workspace: [consumer_workspace_name]
    model: [semantic model or pipeline name]
```

**`contracts/dq_contract_[ws].yaml`** — the DQ rule contract for this workspace.
Pre-filtered from `governance/dq_rules/dq_rules_catalogue.json`. Defines every
rule the DQ gate notebook will enforce at Bronze→Silver promotion time:

```yaml
# DQ Contract: [ws_name]
# Source: governance/dq_rules/dq_rules_catalogue.json
workspace: [ws_name]
effective_from: "[YYYY-MM-DD]"
rules:
  - rule_id: DQ-[DOMAIN]-[NNN]
    rule_name: [entity]_[field]_[check_type]
    target_table: [table_name]
    column: [column_name]          # omit for table-level rules
    check_type: not_null | unique | regex | range | referential | enum | cross_field | row_count_drift | duplicate_bk
    severity: mandatory_gate | monitoring | recommended | optional
    action_on_fail: quarantine | warn | log
    threshold: [value]             # for range / completeness / drift rules
    pattern: "[regex]"             # for regex rules
    allowed_values: []             # for enum rules
    cross_field_expr: "[expr]"     # for cross-field rules
    version: "1.0"
    effective_from: "[YYYY-MM-DD]"
```

Severity semantics:
- `mandatory_gate` — failure blocks Bronze→Silver promotion; record quarantined
- `monitoring` — failure logged to Governance workspace, alert raised, processing continues
- `recommended` — failure logged only
- `optional` — informational

---

**`quickstart.md`** — the smoke-test and key validation guide for the engineer
accepting delivery of the workspace. Lists the minimum set of scenarios to run to
confirm the workspace is working correctly end-to-end.

Structure:
```markdown
# Quickstart: [ws_name]

## Pre-requisites
- [ ] Infrastructure deployed (see infra/README.md)
- [ ] Workspace created and capacity assigned
- [ ] Governance workspace healthy (DQ rules loaded)

## Validation Scenarios

### Scenario 1: Happy path ingest
1. Seed source system with [N] test records
2. Trigger ingest pipeline manually
3. Verify records appear in Bronze Lakehouse
4. Trigger transform pipeline
5. Verify clean records in Silver, zero in quarantine

### Scenario 2: DQ gate blocks bad data
1. Seed source with 1 record missing mandatory [field]
2. Run full pipeline
3. Verify record appears in quarantine table, not Silver
4. Verify DQ score logged to Governance workspace

### Scenario 3: SLA compliance
1. Check last successful pipeline run timestamp
2. Confirm delta < [P1=4h | P2=24h | P3=72h]
3. Verify SLA status in Operations dashboard
```

---

**`tasks.md`** — the executable task list for the engineer building this workspace.
Tasks are derived from the plan and contracts. Independent tasks are marked `[P]`
(parallelisable). The engineer works top-to-bottom and ticks off tasks before
marking the workspace "Ready for QA".

Structure:
```markdown
# Tasks: [ws_name]

## Parallel Group 1 — Infrastructure Setup [P]
- [ ] [P] Deploy Azure infrastructure (`infra/scripts/deploy_infra.sh --env dev`)
- [ ] [P] Create Fabric workspace via `create_fabric_workspaces.ps1`
- [ ] [P] Create AAD groups and assign roles (see plan.md → Security Model)

## Group 2 — Lakehouse Setup (after Group 1)
- [ ] Create Bronze Lakehouse: lh_bronze_[ws_short_name]
- [ ] Create Silver Lakehouse: lh_silver_[ws_short_name]
- [ ] Apply Delta schemas from `schemas/*.json`
- [ ] Seed pipeline metadata: `metadata/pipeline_metadata.json`

## Parallel Group 3 — Contract + DQ Setup [P]
- [ ] [P] Register data contract with Governance workspace (see contracts/)
- [ ] [P] Load DQ rules from `contracts/dq_contract_[ws].yaml` to Governance

## Group 4 — Pipeline Deployment (after Groups 2 & 3)
- [ ] Deploy ingest pipeline and run end-to-end test
- [ ] Deploy transform pipeline + DQ gate notebook
- [ ] Verify quarantine table is created and DQ scores logged

## Group 5 — Regulatory Checks (L1 workspaces handling personal data)
- [ ] GDPR consent check (DQ-GDPR-001) verified
- [ ] Data residency check (DQ-GDPR-003) verified
- [ ] Erasure field (`erasure_requested_at`) present in PII tables
- [ ] PII fields pseudonymised at Gold layer boundary
- [ ] [DE workspaces only] AÜG hard-block thresholds tested

## Group 6 — CI/CD
- [ ] Configure GitHub Secrets (DEV_KEYVAULT_URI, PROD_KEYVAULT_URI)
- [ ] `cicd/deploy.yml` passes on `develop` branch
- [ ] `cicd/validate.yml` PR validation passing

## Handover Gate
- [ ] All Quickstart scenarios pass (see quickstart.md)
- [ ] README.md reviewed and approved by domain owner
- [ ] Purview lineage verified for all pipelines
- [ ] SLA baseline confirmed in Operations dashboard
- [ ] Workspace marked `ready` in `infra/workspace_catalogue.json`
```

### `workspaces/ws_[name]/cicd/deploy.yml`
GitHub Actions (or Azure DevOps YAML — generate the appropriate format based on the
platform identified in Design Area 5) deployment pipeline:

```yaml
# Deployment pipeline: [ws_name]
# Triggers on merge to main (prod) or merge to develop (dev/test)

name: Deploy [ws_name]

on:
  push:
    branches: [main, develop]
    paths:
      - 'workspaces/[ws_name]/**'
      - 'infra/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Validate data contracts
        run: python scripts/validate_contracts.py workspaces/[ws_name]/data_contracts/
      - name: Validate naming conventions
        run: python scripts/validate_naming.py workspaces/[ws_name]/
      - name: Validate pipeline metadata schema
        run: python scripts/validate_metadata.py workspaces/[ws_name]/metadata/

  deploy_dev:
    needs: validate
    if: github.ref == 'refs/heads/develop'
    environment: dev
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Fabric (dev)
        run: |
          python scripts/deploy_workspace.py \
            --workspace [ws_name] \
            --environment dev \
            --keyvault ${{ secrets.DEV_KEYVAULT_URI }}

  deploy_prod:
    needs: validate
    if: github.ref == 'refs/heads/main'
    environment: prod
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Fabric (prod)
        run: |
          python scripts/deploy_workspace.py \
            --workspace [ws_name] \
            --environment prod \
            --keyvault ${{ secrets.PROD_KEYVAULT_URI }}
```

### `workspaces/ws_[name]/cicd/validate.yml`
PR validation pipeline: runs contract validation, naming check, and metadata schema
validation on every pull request. Does not deploy.

---

## Section 3 — Governance workspace (`governance/`)

### `governance/workspace/`
Fabric item definitions for `ws_gov_data_governance`:
- `lh_bronze_gov_registry/` — Lakehouse item stub
- `lh_silver_gov_registry/` — Lakehouse item stub
- `pl_naming_audit.json` — Pipeline JSON: scans all registered workspaces via Fabric
  Admin REST API, compares workspace names against naming convention rules from
  `06_naming_conventions.md`, writes violations to `lh_silver_gov_registry`

### `governance/dq_rules/`
One YAML file per DQ rule, derived from Design Area 7:
`dq_rule_[RULE-ID].yaml` — using the rule definition format from Design Area 7.

Also include `dq_rules_catalogue.json` — machine-readable aggregate of all rules:
```json
[
  {
    "rule_id": "DQ-D1-001",
    "domain": "D1",
    "workspace": "ws_d_d1_[name]",
    "target_table": "d1_[entity]",
    "column": "[column_name]",
    "check_type": "not_null",
    "severity": "quarantine",
    "action_on_fail": "quarantine",
    "version": "1.0",
    "effective_from": "[date]"
  }
]
```

### `governance/data_contracts/`
Data contract YAML files for all Gold tables (same format as workspace data contracts).
These are the authoritative source; workspace copies are generated from here.

### `governance/purview/`
Scripts for Purview integration:

`governance/purview/register_sources.py` — Python script that:
1. Authenticates to Azure Purview (using Service Principal from Key Vault)
2. Reads `infra/workspace_catalogue.json`
3. Calls Purview REST API to register each Fabric workspace as a data source
4. Prints registration report

`governance/purview/push_lineage.py` — Python script / Fabric notebook that:
1. Reads pipeline run logs from Operations workspace
2. Constructs Purview Atlas lineage entities (process + input/output datasets)
3. Pushes lineage to Purview via REST API

`governance/purview/apply_sensitivity_labels.py` — Python script that:
1. Reads classification levels from `infra/workspace_catalogue.json`
2. Calls Microsoft Information Protection API to apply sensitivity labels to
   all tables in Confidential / Restricted workspaces

### `governance/cicd/deploy.yml`
CI/CD pipeline for the Governance workspace. Deploys DQ rules catalogue, data contracts,
and Purview integration scripts. Must deploy before any domain workspace.

---

## Section 4 — Operations workspace (`operations/`)

### `operations/workspace/`
Fabric item definitions for `ws_ops_monitoring`:
- `lh_bronze_ops_logs/` — Lakehouse stub
- `lh_silver_ops_logs/` — Lakehouse stub (parsed pipeline log events)
- `lh_silver_ops_sla/` — Lakehouse stub (SLA compliance tracking)

### `operations/pipelines/`
`pl_ingest_pipeline_logs.json` — ingests log events from all workspaces:
```
Trigger: every 15 minutes
For each active workspace in workspace_catalogue.json:
  → call workspace log endpoint or read from shared Event Stream
  → append to lh_bronze_ops_logs
  → run parse/enrich transform → lh_silver_ops_logs
```

`pl_evaluate_sla.json` — hourly SLA evaluation:
```
For each workspace in workspace_catalogue.json:
  → look up last successful pipeline run time from lh_silver_ops_logs
  → compare to SLA threshold (P1=4h, P2=8h, P3=24h)
  → write result to lh_silver_ops_sla
  → if breach: trigger alert webhook (Teams / email)
```

`pl_incident_rollup.json` — daily incident aggregation.

`alert_rules.json` — alert definitions:
```json
[
  {
    "alert_id": "SLA-P1-BREACH",
    "condition": "pipeline_last_success_hours > 4 AND sla_tier = 'P1'",
    "channel": "teams_webhook",
    "severity": "critical",
    "message_template": "⚠️ P1 SLA breach: {workspace_name} has not refreshed in {hours}h"
  }
]
```

### `operations/cicd/deploy.yml`
CI/CD deployment pipeline for the Operations workspace. Must deploy before domain workspaces.

---

## Section 5 — Root-level scripts (`scripts/`)

Generate shared Python utility scripts used by all CI/CD pipelines:

`scripts/validate_contracts.py` — validates all data contract YAML files in a given
directory against the contract JSON schema. Exit code 1 if any contract is invalid.

`scripts/validate_naming.py` — validates workspace names, pipeline names, table names,
and column names in metadata files against the naming convention rules from Design Area 6.

`scripts/validate_metadata.py` — validates `pipeline_metadata.json` files against the
required schema (required fields, valid data types, active flag type).

`scripts/deploy_workspace.py` — calls the Fabric REST API to deploy pipeline definitions,
upload metadata to Lakehouse, and verify the workspace is healthy post-deploy.

`scripts/load_metadata.py` — loads `pipeline_metadata.json` into the workspace's
`pipeline_metadata` Lakehouse table (Delta format).

---

## Step 6 — Generate the scaffold summary

Create `p4-outputs/_scaffold_summary.md`:

```markdown
# Implementation Scaffold Summary

## What was generated
| Section | Files Generated | Notes |
|---------|----------------|-------|

## Workspace Folders
| Workspace | Folder | Pipelines | Notebooks | Data Contracts | CI/CD |
|-----------|--------|-----------|-----------|----------------|-------|

## Next steps for the engineering team
1. Clone / initialise the repo with this folder structure
2. Run `infra/scripts/deploy_infra.sh --env dev` to provision Azure resources
3. Run `infra/scripts/create_fabric_workspaces.ps1` to create Fabric workspaces
4. Commit and push — CI/CD pipelines will deploy governance and operations workspaces first
5. Domain workspace pipelines deploy automatically after governance and operations are healthy
6. Validate with `scripts/validate_contracts.py` and `scripts/validate_naming.py`

## Open items before build-start
[List any stubs or TODOs embedded in the generated files that need completion]
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

## Step 7 — Final Cost Estimation (TCO)

Generate `p4-outputs/cost_estimates.md`. Read `p3-outputs/cost_estimates.md` to
carry all prior estimates forward. This is the **final, authoritative TCO** — the
document that goes to the project sponsor.

### What to count from Phase 4 outputs

From `infra/workspace_catalogue.json`:
- **W** = actual workspace count generated
- **capacities** = all (capacity_name, SKU) pairs

From the workspace folders generated:
- **P** = total pipeline JSON files across all workspaces/pipelines/
- **E** = total entity YAML files across all workspaces/entities/
- **Sc** = total schema JSON files across all workspaces/schemas/
- **SK** = total speckit bundles (= W)
- **CI** = total CI/CD pipeline files

### AI cost — final actual + forecast recap

Phase 4 actual: count the files generated and estimate tokens consumed:
- Input: sum of all P2+P3 context loaded + cross-references during generation
- Output: total characters written across all generated files ÷ 4 (chars→tokens)

Provide a **4-phase AI cost summary table** showing estimated vs actual where known.
For phases already complete, mark as "actual (estimated)". Note that token counts
are estimates — actual costs appear in Anthropic console billing.

### MS Fabric cost — final TCO

Now that the full workspace catalogue is known, produce the definitive cost model:

**Monthly Operational Cost (prod, steady state):**
For each capacity in `infra/workspace_catalogue.json`:
- Extract SKU and map to $/month from the pricing table
- Sum all prod capacities
- Dev: each prod capacity at one SKU tier lower × (176/730) utilisation factor
- Test: each prod capacity at same SKU × (40/730) utilisation factor

**Storage (now refined from entity schemas):**
- Count total tables across all schema files
- Estimate rows/day from pipeline_metadata.json entries
- rows/day × avg_row_size_kb × 365 days / 1024 / 1024 (GB) × $0.023
- Use 250 bytes as default average row size if not specified

**Purview (final):**
- ceil(total_tables / 50) Purview Data Map CUs × $0.496/CU/h × 730h
- W × 1 scan/day × 0.05 vCore/h × $0.30 × 730h

**Annual cost projection:**
- Monthly opex × 12 + one-time build costs (from Phase 3)

### Output file format

```markdown
# Cost Estimates — [Project Name]
*Final: Phase 4 (Implementation Scaffolding)*
*Generated: [date]*

---

## Executive Summary
| Item | Value |
|------|-------|
| Total Fabric workspaces | [W] |
| Production capacities | [list SKUs] |
| **Estimated monthly Fabric cost (prod)** | **$[X]/month** |
| **Estimated monthly Fabric cost (all envs)** | **$[X]/month** |
| **Estimated annual Fabric cost** | **$[X]/year** |
| One-time build cost (initial load + CI/CD) | $[X] |
| Total AI cost across all 4 phases | $[X.XX] |

---

## MS Fabric — Final Monthly Cost Breakdown

### Production Capacities
| Capacity | SKU | CU | Hours/month | $/month |
|----------|-----|----|------------|---------|
| [cap_name] | F[N] | [N] | 730 | $[X] |
| **Subtotal prod capacity** | | | | **$[X]** |

### Dev & Test Capacities (pause/resume model)
| Capacity | SKU | Active hours/month | $/month |
|----------|-----|--------------------|---------|
| [cap_dev] | F[N] | 176 | $[X] |
| [cap_test]| F[N] | 40  | $[X] |
| **Subtotal dev/test** | | | **$[X]** |

### Data & Supporting Services
| Service | Basis | $/month |
|---------|-------|---------|
| OneLake storage | [N] GB est. × $0.023 | $[X] |
| Microsoft Purview | [N] Data Map CUs + scanning | $[X] |
| Azure Key Vault | secrets + ops | ~$20 |
| Log Analytics | ~10GB/month ingest | ~$25 |
| **Subtotal services** | | **$[X]** |

### Total Monthly Estimate
| Scope | $/month |
|-------|---------|
| Prod only | $[X] |
| All environments (prod + dev + test) | $[X] |

---

## One-Time Build Costs
| Item | Basis | Est. cost |
|------|-------|----------|
| Initial full data load | [from Phase 3] | $[X] |
| CI/CD pipeline runs | [W] × 10 deploys | $[X] |
| **Total one-time** | | **$[X]** |

---

## Annual Cost Projection
| Item | $/year |
|------|--------|
| Fabric operational (prod only) | $[X] |
| Fabric operational (all envs) | $[X] |
| Supporting services | $[X] |
| One-time build costs | $[X] |
| **Total Year 1** | **$[X]** |
| **Total Year 2+ (steady state)** | **$[X]** |

---

## AI Cost Summary — All 4 Phases
| Phase | Activity | Est. input tokens | Est. output tokens | Est. cost |
|-------|----------|------------------|-------------------|----------|
| P1 — Discovery | Domain + org mapping | [N] | [N] | $[X] |
| P2 — Technical Design | 10 design areas + workspace MDs | [N] | [N] | $[X] |
| P3 — Task Breakdown | 11 discipline task files | [N] | [N] | $[X] |
| P4 — Implementation | [W] workspace folders + infra | [N] | [N] | $[X] |
| **Total** | | **[N]** | **[N]** | **$[X.XX]** |

> Note: Actual token costs appear in Anthropic Console → Billing.
> Estimates assume Claude Sonnet at $3.00/MTok input, $15.00/MTok output.

---

## Cost Optimisation Recommendations

1. **Pause dev/test capacities** outside working hours — saves ~[X]% of dev/test cost
2. **Scale prod to F4** for initial launch, scale up to F8 once load is confirmed
3. **OneLake lifecycle policies** — archive Bronze tables >90 days old to cold tier ($0.001/GB/month)
4. **Purview selective scanning** — scan Confidential/Restricted workspaces only (reduce scan costs)
5. **Capacity Reservation** — 1-year reserved capacity saves ~17% vs pay-as-you-go

> ⚠️ All estimates should be validated against the Azure Pricing Calculator
> (azure.microsoft.com/pricing/calculator) before presenting to finance/sponsors.
> Region, reservation terms, and EA discounts all affect final pricing.
```

## Review gate

Present all outputs. Confirm:

- [ ] Every workspace from Phase 2 has a folder in `workspaces/`
- [ ] Every workspace has an `entities/` folder with one YAML per logical entity
- [ ] Every workspace has a `schemas/` folder with Delta schema JSON for every Silver/Gold table
- [ ] Every workspace has a complete GitHub Speckit-compatible `speckit/` bundle (spec.md · plan.md · data-model.md · research.md · contracts/ · quickstart.md · tasks.md)
- [ ] Every L1 workspace has ingest pipeline JSON, transform pipeline JSON, and DQ notebook
- [ ] Every L2 workspace has data contracts for all Gold tables
- [ ] Every workspace has a CI/CD deploy and validate pipeline
- [ ] `governance/` folder covers Purview registration, DQ rule catalogue, and data contracts
- [ ] `operations/` folder covers log ingestion, SLA evaluation, and alerting
- [ ] `infra/` folder covers Bicep templates, workspace catalogue, and deploy scripts
- [ ] Final TCO generated at `p4-outputs/cost_estimates.md` with 4-phase AI cost summary, monthly Fabric cost, and annual projection

When confirmed, state:

> **Phase 4 complete.** Implementation scaffold generated.
> [N] workspace folders, [N] entity YAML files, [N] Delta schema files,
> [N] GitHub Speckit-compatible bundles (spec · plan · data-model · research · contracts · quickstart · tasks),
> [N] pipeline templates, [N] CI/CD pipelines saved to `p4-outputs/`.
> The engineering team can begin building immediately from this scaffold.
> `p4-outputs/cost_estimates.md` contains the final TCO: monthly Fabric operational cost,
> annual projection, one-time build costs, and 4-phase AI cost summary.
> **All 4 phases of the Fabric SDLC are complete.**

Save all files to the workspace folder (not just the outputs directory) so the user
can access them directly.

---

## Output structure

```
p4-outputs/
├── README.md
├── _scaffold_summary.md
├── cost_estimates.md
├── infra/
│   ├── bicep/
│   │   ├── main.bicep
│   │   └── modules/
│   ├── environments/
│   │   ├── dev.json
│   │   ├── test.json
│   │   └── prod.json
│   ├── workspace_catalogue.json
│   └── scripts/
│       ├── deploy_infra.sh
│       └── create_fabric_workspaces.ps1
├── governance/
│   ├── workspace/
│   ├── dq_rules/
│   ├── data_contracts/
│   ├── purview/
│   └── cicd/
├── operations/
│   ├── workspace/
│   ├── pipelines/
│   └── cicd/
├── scripts/
│   ├── validate_contracts.py
│   ├── validate_naming.py
│   ├── validate_metadata.py
│   ├── deploy_workspace.py
│   └── load_metadata.py
└── workspaces/
    ├── ws_[name_1]/
    │   ├── README.md
    │   ├── metadata/pipeline_metadata.json
    │   ├── entities/                      ← entity YAML files
    │   ├── schemas/                       ← Delta schema JSON files
    │   ├── pipelines/
    │   ├── notebooks/
    │   ├── speckit/                       ← GitHub Speckit-compatible handover bundle
    │   │   ├── spec.md                    ← SDD feature specification
    │   │   ├── plan.md                    ← implementation plan with phase gates
    │   │   ├── data-model.md              ← entity/schema definitions
    │   │   ├── research.md                ← pattern choices and open questions
    │   │   ├── contracts/                 ← data + DQ contracts
    │   │   │   ├── contract_[table].yaml
    │   │   │   └── dq_contract_[ws].yaml
    │   │   ├── quickstart.md              ← smoke-test validation guide
    │   │   └── tasks.md                   ← executable task list with [P] markers
    │   └── cicd/
    └── ws_[name_N]/
        └── ...
```
