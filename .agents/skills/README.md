# Ansible Agent Skills

Skills and instructions for AI agents producing enterprise-grade Ansible automation. Designed to counteract documented LLM weaknesses with declarative infrastructure code.

## Quick Start

1. Read `AGENTS.md` (or `claude.md`) first — it defines personas, mandatory rules, and workflow
2. Match your persona to the task (Ansible Developer, Python Developer, AAP Administrator)
3. Read the relevant SKILL.md files listed below before writing code

## Available Skills

| Skill | Path | Persona | Description |
|-------|------|---------|-------------|
| **ansible-architect** | [ansible/](ansible/SKILL.md) | Ansible Developer | Core architecture: idempotency, module selection, role engineering, linting, security |
| **ansible-dev-tools** | [tools/](tools/SKILL.md) | All | ADT toolchain: ansible-creator, navigator, builder, lint, molecule, tox-ansible |
| **ansible-module-dev** | [module/](module/SKILL.md) | Python Developer | Custom module development: argument_spec, check mode, diff, DOCUMENTATION blocks |
| **aap-config-as-code** | [config-as-code/](config-as-code/SKILL.md) | AAP Administrator | Declarative CaC: infra.aap_configuration, dependency sequencing, vault, dispatch |
| **ansible-molecule** | [molecule/](molecule/SKILL.md) | Ansible Developer | Testing: functional verification, ansible-native architecture, negative testing |

## Persona → Skill Mapping

### Ansible Developer
Primary: `ansible-architect` + `ansible-molecule`
Supporting: `ansible-dev-tools`

### Python Developer (Module Author)
Primary: `ansible-module-dev`
Supporting: `ansible-dev-tools`

### AAP Administrator
Primary: `aap-config-as-code`
Supporting: `ansible-architect` + `ansible-dev-tools`

## Why These Exist

LLMs struggle with Ansible specifically due to structural factors:

| Problem | Impact | Mitigation in These Skills |
|---------|--------|---------------------------|
| State reconciliation failures | 45% of errors | State Reasoning Protocol (AGENTS.md) |
| Module knowledge gaps | 24% of errors | Module Selection Hierarchy (ansible skill) |
| Shortest Path Optimization | Bypasses Ansible for shell | Action Space Constraints (AGENTS.md) |
| Autonomy collapse on IaC | Passive writing, missed dependencies | Cross-Resource Dependency Awareness (AGENTS.md) |
| Correctness-Congruence Gap | Syntactically valid but architecturally wrong | Quality Gates + scaffold-then-fill workflow |
| Training data bias | Imperative patterns dominate | Explicit anti-patterns with correct alternatives |
