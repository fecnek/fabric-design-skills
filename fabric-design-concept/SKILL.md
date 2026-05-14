---
name: fabric-design-concept
description: >
  Fabric SDLC — Phase 1: Domain & Organisation Discovery. Use this skill whenever the user
  wants to start designing a Microsoft Fabric analytical platform, map business domains,
  identify core vs supporting vs generic domains, capture multi-company or geographic
  organisational structure, or define domain entities, processes, and upstream systems.
  Triggers: "/fabric-design-concept", "P1", "phase 1", "domain discovery", "start fabric design", "map our domains",
  "org mapping", "identify domains", "begin the framework", "start analytics platform design",
  "map business capabilities". This is always the first phase — always run it before any
  technical decisions. Outputs one MD file per domain and one MD file per org unit, which
  become the authoritative inputs to Phase 2.
---

# Phase 1 — Domain & Organisation Discovery

## What this phase produces

By the end of Phase 1 you will have a set of structured Markdown files that capture
*what the organisation does*, *how it is structured*, and *where the data lives today*.
These files are the single source of truth for all subsequent phases:

- `_discovery_summary.md` — top-level index of everything discovered
- `organisations/org_[CODE]_[name].md` — one file per company / legal entity / major org unit
- `domains/domain_[CODE]_[name].md` — one file per business domain

Save all outputs to a `p1-outputs/` folder in the workspace.

---

## Step 0 — Choose input mode

Check whether the user has attached a file (Excel, CSV, Word, PDF).

- **File attached** → Import Mode: parse the file, extract org and domain data, fill the
  templates below, ask the user to confirm or fill gaps.
- **No file** → Interview Mode: ask the structured question rounds below, wait for answers
  after each round before continuing.

---

## Interview Mode — question rounds

### Round 1: Organisational structure
1. How many legal entities / companies are in scope for this platform?
2. For each company: name, primary business activity, HQ location.
3. Is the organisation geographically distributed across countries or regions?
4. Do any regions have specific data residency or regulatory jurisdiction requirements?
5. Are there entities that share business processes or data, vs. fully independent ones?

### Round 2: Domain landscape
6. What are the main business areas or departments across the whole organisation?
7. Which of these is where the company's competitive edge really lives — something unique
   to how you do business that no off-the-shelf software captures well? *(Core domain candidates)*
8. Which areas are important but largely the same as any other company in your industry —
   Finance, HR, basic CRM? *(Supporting domain candidates)*
9. Which areas could be replaced tomorrow by an off-the-shelf SaaS product with minimal
   impact? *(Generic domain candidates)*
10. Are there concerns that cut across many domains — Master Data, Regulatory Reporting,
    shared Reference Data?

### Round 3: Domain deep-dive (repeat for each domain from Round 2)
11. What are the key **entities** in this domain? (e.g. Policy, Customer, Order, Claim)
12. What are the main **processes or workflows** that happen here?
13. Are there **regulatory or compliance obligations** attached to this domain's data?
14. What are the biggest **pain points** in accessing or using data from this domain today?
15. Which **upstream systems** produce the data for this domain? (ERP, CRM, APIs, files, etc.)
16. Who **consumes** analytics from this domain, and for what purpose?

### Round 4: Upstream systems inventory
17. List all known source systems with: system name, type (ERP / CRM / custom app / file /
    API / external feed), which domains it serves, approximate data volume and refresh
    frequency, and whether it is on-premises or cloud.
18. Is there an existing Fabric, Synapse, or data warehouse environment? Describe its structure.

### Round 5: Strategic context
19. What is the primary goal of this analytics platform?
    (operational reporting / regulatory compliance / strategic insight / self-service / ML/AI)
20. Is there a delivery timeline or phased rollout expectation?
21. Who is the executive sponsor and who will own data governance?

---

## Step 1 — Classify every domain

After collecting data, classify each domain. Show the classification to the user and ask
for confirmation before generating files.

| Type | What it means | Signals |
|------|--------------|---------|
| **Core** | Primary competitive differentiation. Unique to this org. | Complex domain rules, strong business owner investment, "only we do it this way", concentrated pain points |
| **Supporting** | Necessary enabler, not differentiating. Shared-service pattern. | Finance, HR, basic CRM — important but generic across industries |
| **Generic** | Commodity. Could be an off-the-shelf product. | Email, file storage, standard billing, basic IT ticketing |

Also tag any **Cross-Cutting** concerns: capabilities that span multiple domains and need
their own platform services (MDM, Regulatory Reporting, Shared Reference Data, Security).

For **multi-company / multi-geography organisations** add:
- **Ownership scope**: which company / region owns or co-owns this domain
- **Shared vs local**: is this domain shared across entities, or purely local to one?
- **Data residency**: does the domain's data need to stay in a specific cloud region?

Domain codes: Core → `D1`, `D2`… Supporting → `S1`, `S2`… Generic → `G1`, `G2`…

---

## Step 2 — Generate Domain MD files

For every domain, generate one file: `domains/domain_[CODE]_[slug].md`

```markdown
# Domain: [Full Domain Name]

## Classification
| Field | Value |
|-------|-------|
| Domain Code | D1 / S1 / G1 |
| Type | Core / Supporting / Generic |
| Cross-Cutting? | Yes / No |
| Owner (Company) | [Legal entity or org code] |
| Owner (Org Unit) | [Team or department] |
| Geography / Region | [Region or "Global"] |
| Data Residency | [Cloud region or "No restriction"] |
| Shared Across Entities | Yes / No |

## Purpose
[One paragraph. What this domain is responsible for. For Core domains, explicitly state
the competitive differentiation angle. For Supporting/Generic, state why the org needs it.]

## Key Entities
| Entity | What it represents | Key identifiers / attributes |
|--------|--------------------|------------------------------|
| [Entity] | [Description] | [ID fields, key attributes] |

## Key Processes
| Process | Description | Frequency | Regulatory obligation? |
|---------|-------------|-----------|------------------------|
| [Process name] | [What happens, inputs/outputs] | [Daily / event-driven / etc.] | Yes / No |

## Upstream Systems
| System Name | Type | Data Provided | Volume / Frequency | On-prem or Cloud |
|-------------|------|---------------|--------------------|------------------|
| [Name] | ERP / CRM / API / File | [Tables, feeds, files] | [e.g. 100k rows/day] | [On-prem / Azure / AWS] |

## Requirements & Constraints
- **Regulatory:** [GDPR, Solvency II, FCA, HIPAA, SOX, etc. — or "None identified"]
- **Data Classification:** [Public / Internal / Confidential / Restricted]
- **Refresh SLA:** [e.g. "Available by 07:00 CET daily"]
- **Data Quality Pain Points:** [Known quality or completeness issues]
- **Dependencies on Other Domains:** [Which domains share data or entities with this one]

## Analytics Consumers
| Consumer (team/role) | Analytical need | Frequency of use |
|----------------------|-----------------|-----------------|
| [Name / team] | [What insight they need] | [Daily / ad-hoc / etc.] |

## Open Questions
- [Any unresolved items to clarify in Phase 2]
```

---

## Step 3 — Generate Organisation MD files

For every company or major org unit, generate one file: `organisations/org_[CODE]_[slug].md`

```markdown
# Organisation: [Company / Org Unit Name]

## Profile
| Field | Value |
|-------|-------|
| Org Code | CO1 / CO2 etc. |
| Legal Name | [Full legal entity name] |
| Primary Business | [What it does] |
| HQ Location | [City, Country] |
| Regulatory Jurisdiction | [e.g. UK FCA, EU GDPR, US SEC — list all] |
| Relationship to Group | [Subsidiary of / Division of / Independent] |

## Domains Owned or Co-Owned
| Domain Code | Domain Name | Type | Local-only or Shared? | Shared with |
|-------------|-------------|------|----------------------|-------------|
| [D1] | [Name] | Core / Supporting / Generic | Local / Shared | [CO2, CO3 or n/a] |

## Upstream Systems
| System | Purpose | Shared with other entities? |
|--------|---------|----------------------------|
| [System] | [Use] | Yes — [which] / No |

## Data Residency & Compliance Notes
[Describe any data sovereignty rules, cross-border transfer restrictions, cloud region
requirements, or regulatory reporting obligations specific to this entity.]

## Key Stakeholders
| Role | Name | Responsibility |
|------|------|----------------|
| Business Sponsor | [Name / TBD] | Platform funding and priority |
| Data Owner | [Name / TBD] | Data quality and governance accountability |
| IT / Delivery Lead | [Name / TBD] | Technical delivery ownership |
```

---

## Step 4 — Generate the Discovery Summary

Create `_discovery_summary.md` at the root of `p1-outputs/`:

```markdown
# Discovery Summary — [Organisation Name]
*Generated: [Date]*

## Organisation Map
| Org Code | Company Name | Region | Regulatory Jurisdiction | Domain Count |
|----------|-------------|--------|------------------------|--------------|

## Domain Inventory
| Code | Domain Name | Type | Owner Company | Upstream System Count | Cross-Cutting? |
|------|-------------|------|--------------|----------------------|----------------|

## Cross-Cutting Concerns
| Concern | Domains Spanned | Notes |
|---------|----------------|-------|

## Upstream Systems Registry
| System | Type | Serving Domains | Volume | On-prem / Cloud |
|--------|------|-----------------|--------|-----------------|

## Phase 2 Readiness Checklist
- [ ] All domains classified (Core / Supporting / Generic)
- [ ] All upstream systems identified with owner domain
- [ ] Data residency requirements captured per company
- [ ] Analytics consumers identified for all Core domains
- [ ] Cross-cutting concerns flagged
- [ ] Open questions resolved

## Open Questions for Phase 2
[List any items that need resolution before technical design begins]
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

## Step 5a — Cost Estimation

Generate `p1-outputs/cost_estimates.md`. This is the **first cut** of costs — rough
order-of-magnitude estimates based on domain and org counts discovered in this phase.
Every subsequent phase refines these numbers; Phase 4 produces the final TCO.

### What to count from Phase 1 outputs
- **D** = number of core + supporting domains (exclude generic)
- **O** = number of companies / org units
- **S** = number of upstream source systems (sum across all domain files)
- **W_est** = estimated workspace count = (D × 2) + (O × 1 L2 aggregation) + 3 (L3 + Ops + Gov)

### AI cost — Phase 2 (Technical Design) forecast

Phase 2 produces 10 design area MDs, one workspace MD per workspace, one
architecture SPA, and a design summary. Estimate token consumption as:

| Component | Input tokens | Output tokens |
|-----------|-------------|--------------|
| Base context (P1 outputs loaded) | 30,000 | — |
| Per domain file processed | D × 4,000 | — |
| Per org file processed | O × 3,000 | — |
| Design area generation (10 areas) | 20,000 | 10 × 3,000 |
| Per workspace MD | W_est × 2,000 | W_est × 3,000 |
| Architecture SPA | 5,000 | 8,000 |

**Formula:**
- Input = 30,000 + (D × 4,000) + (O × 3,000) + 20,000 + (W_est × 2,000) + 5,000
- Output = (10 × 3,000) + (W_est × 3,000) + 8,000

**Estimated cost** = (Input / 1,000,000 × $3.00) + (Output / 1,000,000 × $15.00)

### MS Fabric cost — initial rough estimate

At this stage we don't know capacity SKUs. Use this rule of thumb:
- One F8 capacity per company with core domains (prod)
- One F4 shared capacity for all supporting/generic domains (prod)
- Dev: one F2 per company (active 176h/month)
- Test: one F4 shared (active 40h/month)
- Add OneLake storage: S × 50 GB baseline × 0.25 compression factor × $0.023/GB/month
- Add Purview: ~$200/month flat estimate at this stage
- Add Azure support resources: ~$100/month

Note clearly: **"Will be refined in Phase 2 once capacity assignments are decided."**

### Output file format

```markdown
# Cost Estimates — [Project Name]

## Phase 2 (Technical Design) — AI Cost Forecast
| Metric | Value |
|--------|-------|
| Domains (D) | [N] |
| Org units (O) | [N] |
| Source systems (S) | [N] |
| Estimated workspaces (W_est) | [N] |
| Estimated input tokens | [N] |
| Estimated output tokens | [N] |
| **Estimated Phase 2 AI cost** | **$[X.XX]** |

## MS Fabric — Initial Monthly Estimate (prod only)
| Component | Basis | Est. $/month |
|-----------|-------|-------------|
| Prod capacity ([O]× F8) | [N] CU/h × 730h × $0.18/CU | $[X] |
| Dev capacity ([O]× F2) | [N] CU/h × 176h × $0.18/CU | $[X] |
| Test capacity (shared F4) | 4 CU/h × 40h × $0.18/CU | $[X] |
| OneLake storage | [S]×50GB×0.25×$0.023 | $[X] |
| Purview | flat estimate | $200 |
| Azure support resources | flat estimate | $100 |
| **Total initial estimate** | | **$[X]** |

> ⚠️ This is a Phase 1 rough estimate (±50%). Phase 2 will refine capacity SKUs
> and workspace counts. Phase 3 will add implementation effort. Phase 4 delivers
> the final TCO.

## Cumulative AI Cost To Date
| Phase | Input tokens | Output tokens | Cost |
|-------|-------------|--------------|------|
| Phase 1 (actual) | [N] | [N] | $[X.XX] |
```

Add the review gate check: **"Cost estimates file generated and saved to `p1-outputs/`"**

## Step 5 — Review gate

Present the full output to the user. Confirm:

- Every business area has a domain file
- Every company / org unit has an org file
- Every domain has a type (Core / Supporting / Generic)
- Every domain has at least one upstream system
- Cost estimates file generated at `p1-outputs/cost_estimates.md`
- Data residency constraints are captured where relevant

When confirmed, state:

> **Phase 1 complete.** Domain and organisation model is locked.
> [N] domain files and [N] organisation files saved to `p1-outputs/`.
> `p1-outputs/cost_estimates.md` contains Phase 2 AI cost forecast and initial Fabric monthly estimate.
> Ready to proceed to **Phase 2 — Technical Design**.

---

## Output structure

```
p1-outputs/
├── _discovery_summary.md
├── cost_estimates.md
├── domains/
│   ├── domain_D1_[name].md
│   ├── domain_D2_[name].md
│   ├── domain_S1_[name].md
│   └── domain_G1_[name].md
└── organisations/
    ├── org_CO1_[name].md
    └── org_CO2_[name].md
```
