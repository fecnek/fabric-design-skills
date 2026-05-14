# Fabric Design Skills

A collection of Claude Cowork skills for designing and building **Microsoft Fabric analytical platforms** using a structured 4-phase SDLC framework.

## Skills

| Skill | Phase | Description |
|---|---|---|
| `fabric-design` | Orchestrator | Overview & phase router — start here if unsure where you are |
| `fabric-design-concept` | Phase 1 | Domain & organisation discovery — map business domains and org structure |
| `fabric-design-technical` | Phase 2 | Technical architecture — workspace design, capacity, pipelines, security |
| `fabric-design-plan` | Phase 3 | Task breakdown — work backlog, sequencing, sprint planning |
| `fabric-design-implementation` | Phase 4 | Implementation scaffolding — repo structure, CI/CD, pipeline templates |

## Structure

Each skill folder contains a `SKILL.md` — the skill prompt loaded by Claude when the skill is invoked.

```
fabric-design-skills/
├── fabric-design/
├── fabric-design-concept/
├── fabric-design-technical/
├── fabric-design-plan/
└── fabric-design-implementation/
```

## Usage

These skills are designed for use with **Claude Cowork**. To install them, package this folder as a Cowork plugin or copy individual skill folders into your skills directory.

Invoke a skill by name in a Cowork session, e.g.:

> *"Run the fabric design framework"* → triggers `fabric-design`  
> *"Phase 2 technical design"* → triggers `fabric-design-technical`  
> *"Scaffold the implementation"* → triggers `fabric-design-implementation`
