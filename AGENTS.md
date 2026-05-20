# Agent Instructions

This repository provides a Superpowers-style skill system for producing enterprise-grade Ansible automation.

## Getting Started

**Read `skills/using-ansible-superpowers/SKILL.md` FIRST.** This is the bootstrap skill that tells you how to use the entire system. Do not skip it.

The **Ansible Architect** (`skills/ansible-architect/SKILL.md`) is your primary orchestrator. It detects project context, activates the correct sub-persona, enforces the State Reasoning Protocol, and manages the full development lifecycle.

## Why This System Exists

LLMs achieve under 12% pass rate on Ansible tasks. The dominant failure modes are:
- **State reconciliation errors (45%):** Failing to reason about current vs. desired state
- **Module knowledge gaps (24%):** Wrong module, wrong parameters, misunderstanding idempotency
- **Shortest Path Optimization:** Direct shell commands bypass methodology
- **Autonomy collapse:** Agents miss cross-resource dependencies in graph-shaped infrastructure

The skill system counteracts each failure mode with specific architectural interventions.

## Architecture

```
Session Start
  └── using-ansible-superpowers (bootstrap)
        └── ansible-architect (orchestrator)
              ├── Context Detection (what project type?)
              ├── Sub-Persona Activation
              │   ├── Ansible Developer
              │   ├── Module Author
              │   ├── AAP Administrator
              │   └── Platform Engineer
              └── Lifecycle Management
                  ├── ansible-brainstorming → design
                  ├── ansible-planning → implementation plan
                  ├── subagent-driven-development → execute
                  ├── ansible-code-review → review
                  ├── verification-before-completion → prove
                  └── finishing-a-development-branch → ship
```

## Skill Categories

### Orchestrator
- **ansible-architect** -- Primary orchestrator, context detection, persona activation

### Workflow (follow in order)
- **ansible-brainstorming** -- Infrastructure-aware design before implementation
- **ansible-planning** -- Scaffold-then-fill plans with lint gates
- **subagent-driven-development** -- Fresh subagent per task, two-stage review
- **test-driven-development** -- Molecule RED-GREEN-REFACTOR cycle
- **systematic-debugging** -- 4-phase root cause with Ansible diagnostics
- **ansible-code-review** -- Production readiness checklist
- **verification-before-completion** -- 8-gate verification proof
- **finishing-a-development-branch** -- Git hygiene and PR preparation

### Domain Reference
- **ansible-architecture** -- Core architecture: idempotency, modules, roles, linting, APME
- **ansible-molecule** -- Testing: functional verification, negative testing, CI patterns
- **ansible-module-dev** -- Custom modules: argument_spec, check mode, diff, DOCUMENTATION
- **aap-config-as-code** -- Declarative CaC: infra.aap_configuration, dispatch, promotion
- **ansible-dev-tools** -- ADT toolchain: creator, navigator, builder, lint, molecule, MCP
- **ansible-git-workflow** -- Trunk-based development, CalVer, atomic promotion
- **ansible-cicd** -- GitHub Actions + Tekton pipelines, quality gates
- **ansible-security** -- Zero-trust, secrets management, RBAC, APME, supply chain
- **ansible-ee-builder** -- Execution environment lifecycle, V3 schema, versioning
- **ansible-release-management** -- Release manifests, atomic promotion, CalVer sync
- **ansible-code-style** -- YAML/Ansible/Python/Shell/Jinja2 formatting standards

## Mandatory Rules

All mandatory rules (Action Space Constraints, State Reasoning Protocol, Module Selection, Scaffold-Then-Fill, Cross-Resource Dependency Awareness, Quality Gates) are defined in the **ansible-architect** skill. Read it.

## Quick Reference

| I want to... | Read this skill |
|-------------|----------------|
| Start any Ansible work | `ansible-architect` |
| Design a new role/playbook | `ansible-brainstorming` |
| Create an implementation plan | `ansible-planning` |
| Write or review Ansible code | `ansible-architecture` |
| Test with Molecule | `ansible-molecule` |
| Write a custom module | `ansible-module-dev` |
| Configure AAP via CaC | `aap-config-as-code` |
| Use ADT tools | `ansible-dev-tools` |
| Fix a bug | `systematic-debugging` |
| Review code quality | `ansible-code-review` |
| Prove it works | `verification-before-completion` |
| Manage git workflow | `ansible-git-workflow` |
| Set up CI/CD pipelines | `ansible-cicd` |
| Handle secrets/security | `ansible-security` |
| Build execution environments | `ansible-ee-builder` |
| Manage releases | `ansible-release-management` |
| Check code style | `ansible-code-style` |
| Finish and ship | `finishing-a-development-branch` |
