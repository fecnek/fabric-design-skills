---
name: fabric-design
description: >
  Fabric SDLC — Master orchestrator. Use this skill whenever the user wants an overview of
  the full Fabric SDLC framework, wants to know which phase to run next, wants to start the
  end-to-end analytical platform design process, or asks "what is the Fabric SDLC" or "how
  does the framework work". Also use this skill when the user's intent is unclear and they
  seem to be somewhere in the middle of the framework. Triggers: "/fabric-design", "fabric sdlc", "what phase
  am I on", "where do I start", "run the framework", "which skill do I use", "fabric platform
  design process", "end to end fabric design". This skill explains the 4-phase framework and
  routes the user to the correct phase skill.
---

# Fabric SDLC — Framework Overview

## The 4-phase framework

The Fabric SDLC is a structured, phase-gated approach for designing and building a
Microsoft Fabric analytical platform. Each phase has a defined set of outputs that
become the inputs to the next phase. Phases should be run in order.

```
Phase 1 — Domain & Org Discovery
    ↓  outputs: domain MD files, org MD files
Phase 2 — Technical Design
    ↓  outputs: workspace MD files, design area MD files, architecture SPA
Phase 3 — Task Breakdown
    ↓  outputs: discipline task lists, build order
Phase 4 — Implementation Scaffolding
       outputs: GitHub repo structure, CI/CD pipelines, ETL templates, infra-as-code
```

---

## Phase summaries

### Phase 1 — Domain & Organisation Discovery
**Skill:** `fabric-sdlc-p1-domain-discovery`

Captures what the organisation does, how it is structured, and where the data lives today.
Works with multi-company and geographically distributed organisations. Classifies all
business areas into Core, Supporting, or Generic domains. Identifies every upstream system.

**Output:** `p1-outputs/` — one MD file per domain, one MD file per company / org unit,
and a discovery summary.

**Run this when:** Starting from scratch, or when the domain model needs to be defined
before any technical decisions are made.

---

### Phase 2 — Technical Design
**Skill:** `fabric-sdlc-p2-technical-design`

Translates the domain model from Phase 1 into a complete Microsoft Fabric architecture.
Covers all 10 design areas: tenant design, workspace and capacity design, pipeline design,
source system integration, infrastructure, naming conventions, data quality rules,
security model, operations and monitoring, and the semantic layer.

The 3-layer architecture is always applied:
- **Layer 1** — per-domain workspaces (Bronze → Silver medallion, DQ-gated)
- **Layer 2** — aggregation workspaces (Silver → Gold, cross-domain data products)
- **Layer 3** — consumption workspaces (semantic models, Power BI reports)

Supporting workspaces (Operations, Data Governance) are designed here and deployed first.

**Output:** `p2-outputs/` — 10 design area MD files, one workspace MD file per workspace,
and a self-contained interactive HTML architecture viewer (single-page app with one tab
per design area).

**Run this when:** Phase 1 is complete and locked, or when the user has domain knowledge
and wants to jump directly to the technical design.

---

### Phase 3 — Task Breakdown
**Skill:** `fabric-sdlc-p3-task-breakdown`

Converts the Phase 2 design into an actionable, sequenced work breakdown structure.
Tasks are organised across 11 disciplines (Infrastructure, Governance, Security,
Operations, Workspace Setup, Data Ingestion, Data Quality, Aggregation, Semantic Layer,
CI/CD, Testing). Dependencies between tasks are explicit. A build order file shows the
critical path and parallel workstreams.

**Output:** `p3-outputs/` — one task file per discipline, a build order file, and a
task summary with effort estimates.

**Run this when:** Phase 2 is complete and the team needs a delivery backlog to work from.

---

### Phase 4 — Implementation Scaffolding
**Skill:** `fabric-sdlc-p4-implementation`

Generates the complete GitHub / Azure DevOps repository folder structure, ready for the
engineering team to start building. Every workspace gets its own folder with:
- Metadata-driven pipeline JSON templates (ADF or Dataflow Gen2)
- DQ gate PySpark notebooks
- Data contract YAML files (Gold layer, versioned)
- CI/CD pipeline definitions (GitHub Actions or Azure DevOps)

Also generates shared infrastructure-as-code (Bicep), Governance workspace files
(DQ rules catalogue, Purview integration scripts), Operations workspace files
(log ingestion, SLA evaluation, alerting), and root-level validation scripts.

**Output:** `p4-outputs/` — full repo scaffold, ready to commit.

**Run this when:** Phase 3 is complete and the engineering team is ready to build.

---

## Where are you now?

Use this checklist to find the right phase:

| Check | Phase |
|-------|-------|
| No domain or org model yet | Start with **Phase 1** |
| Have domain model, no technical design | Start with **Phase 2** |
| Have technical design, no task backlog | Start with **Phase 3** |
| Have task backlog, no repo scaffold | Start with **Phase 4** |
| Have all 4 phases complete | Build from the scaffold |

---

## Key design principles applied throughout

**3-layer architecture:** All platforms use L1 (domain workspaces) → L2 (aggregation) →
L3 (consumption). No shortcuts between layers (e.g. L3 never reads from L1 directly).

**Medallion pattern:** Every L1 workspace has Bronze (raw, immutable) and Silver (cleansed,
DQ-gated). L2 workspaces have Gold (aggregated data product). Data contracts are enforced
on the Gold layer.

**Metadata-driven pipelines:** Every pipeline reads its configuration from a metadata
table in the workspace's own Lakehouse. The metadata is workspace-owned; run logs are
pushed to the shared Operations workspace for monitoring.

**Supporting workspaces first:** Operations and Governance workspaces are always deployed
before domain workspaces. Domain workspaces depend on them (log emission, DQ rule loading,
contract validation) but never block on them.

**Single responsibility:** One domain = one L1 workspace. No L1 workspace reads from
another L1 workspace. Each workspace has its own repo folder, CI/CD pipeline, and
Service Principal.

**Purview integration:** All workspaces register as Purview sources. Lineage is captured
automatically. Sensitivity labels are applied to all Confidential and Restricted tables.

---

## Output folder conventions

Each phase saves its outputs to a numbered folder in the working directory:
- `p1-outputs/` — Phase 1 domain and org MD files
- `p2-outputs/` — Phase 2 design MD files, workspace MD files, architecture SPA
- `p3-outputs/` — Phase 3 task files and build order
- `p4-outputs/` — Phase 4 implementation scaffold

These folders are designed to be committed to source control as the living documentation
of the platform design.
