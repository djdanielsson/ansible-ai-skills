# Ansible Superpowers

A Superpowers-style skill system for AI agents producing enterprise-grade Ansible automation. Based on the [Superpowers](https://github.com/obra/superpowers) framework, adapted for the Ansible ecosystem.

## Why

LLMs achieve under 12% pass rate on Ansible tasks. The dominant failure modes:

| Failure Mode | Frequency | Countermeasure |
|-------------|-----------|----------------|
| State reconciliation errors | 45% | State Reasoning Protocol |
| Module knowledge gaps | 24% | Module Selection Hierarchy |
| Shortest Path Optimization | common | Action Space Constraints |
| Autonomy collapse on IaC | common | Cross-Resource Dependency Awareness |

This skill system counteracts each failure mode with specific architectural interventions, workflow discipline, and domain knowledge.

## Installation

### Cursor

1. Clone or copy this repository into your project (or reference it as a submodule):

```bash
git clone https://github.com/djdanielsson/skills.git .skills
```

2. Copy or symlink the hooks directory into your project root:

```bash
cp -r .skills/hooks ./hooks
```

Or, if you prefer symlinks:

```bash
ln -s .skills/hooks ./hooks
```

3. Add the skills directory to your Cursor rules or `.cursorrules`:

```
Read and follow .skills/skills/using-ansible-superpowers/SKILL.md before any Ansible work.
```

Alternatively, reference the `AGENTS.md` file which Cursor reads automatically if present at the project root.

### Claude Code

1. Clone or copy this repository:

```bash
git clone https://github.com/djdanielsson/skills.git .skills
```

2. Copy or symlink the hooks directory:

```bash
cp -r .skills/hooks ./hooks
```

3. The session-start hook automatically injects the bootstrap skill at the beginning of every conversation.

### Manual (Any AI Agent)

1. At the start of any Ansible conversation, paste or reference:

```
Read skills/using-ansible-superpowers/SKILL.md and follow its instructions.
```

2. The bootstrap skill will guide the agent to invoke the Ansible Architect, which handles everything else.

### Standalone (This Repo Is Your Project)

If this repository IS your Ansible project (not a submodule), everything works out of the box. `AGENTS.md` and `claude.md` at the root already reference the skill system.

## How It Works

```
Session Start
  └── using-ansible-superpowers (bootstrap)
        └── ansible-architect (orchestrator)
              ├── Context Detection
              │   ├── Collection? Role? CaC? EE? Module? Playbook? CI?
              │   └── Activates correct sub-persona
              ├── Sub-Personas
              │   ├── Ansible Developer (playbooks, roles, collections)
              │   ├── Module Author (custom Python modules/plugins)
              │   ├── AAP Administrator (Config as Code)
              │   └── Platform Engineer (CI/CD, EEs, releases)
              └── Development Lifecycle
                  ├── Brainstorm → Design before code
                  ├── Plan → Scaffold-then-fill with lint gates
                  ├── Execute → Subagent per task with review
                  ├── Review → Production readiness checklist
                  ├── Verify → 8-gate proof (lint, idempotency, functional)
                  └── Ship → Git hygiene, CalVer, PR
```

### Key Concepts

**Ansible Architect** -- The primary orchestrator. Detects your project type, activates the right specialist persona, enforces the State Reasoning Protocol, and manages the full lifecycle. Always invoked first.

**State Reasoning Protocol** -- Before writing ANY Ansible task, explicitly reason through: current state, desired state, delta, module selection, idempotency proof, and check mode support.

**Action Space Constraints** -- State-changing actions MUST be Ansible modules. No `apt install`, no `systemctl enable`, no `sed -i`. Diagnostic/read-only commands are fine.

**Scaffold-Then-Fill** -- Don't write complete roles from scratch. Scaffold first, variables second, one task at a time, lint after each step.

**Constitutional Principles** -- All work must comply with: GitOps First, Separation of Duties, Atomic Promotion, Production-Grade Quality, Zero-Trust Security.

## Skills Reference

### Orchestrator

| Skill | When to Use |
|-------|------------|
| `using-ansible-superpowers` | Session bootstrap (automatic) |
| `ansible-architect` | Start of any Ansible work |

### Workflow

| Skill | When to Use |
|-------|------------|
| `ansible-brainstorming` | Designing new automation |
| `ansible-planning` | Creating implementation plans |
| `subagent-driven-development` | Executing multi-task plans |
| `test-driven-development` | Writing Molecule tests (RED-GREEN-REFACTOR) |
| `systematic-debugging` | Fixing failures (root cause first) |
| `ansible-code-review` | Reviewing for production readiness |
| `verification-before-completion` | Proving it works (8 gates) |
| `finishing-a-development-branch` | Git hygiene and PR prep |

### Domain

| Skill | When to Use |
|-------|------------|
| `ansible-architecture` | Writing/reviewing playbooks, roles, collections |
| `ansible-molecule` | Writing Molecule test scenarios |
| `ansible-module-dev` | Developing custom Python modules |
| `aap-config-as-code` | Configuring AAP declaratively |
| `ansible-dev-tools` | Using the ADT toolchain |
| `ansible-git-workflow` | Branching, versioning, promotion |
| `ansible-cicd` | CI/CD pipeline design |
| `ansible-security` | Secrets, RBAC, supply chain |
| `ansible-ee-builder` | Building execution environments |
| `ansible-release-management` | Release manifests, atomic promotion |
| `ansible-code-style` | Code formatting standards |

## Directory Structure

```
.
├── AGENTS.md                  # Agent instructions (Cursor/Copilot)
├── claude.md                  # Agent instructions (Claude Code)
├── README.md                  # This file
├── hooks/
│   ├── session-start          # Bootstrap script
│   ├── hooks.json             # Claude Code hook config
│   └── hooks-cursor.json      # Cursor hook config
├── skills/
│   ├── using-ansible-superpowers/   # Bootstrap skill
│   ├── ansible-architect/           # Orchestrator
│   ├── ansible-brainstorming/       # Workflow: design
│   ├── ansible-planning/            # Workflow: plan
│   ├── subagent-driven-development/ # Workflow: execute
│   ├── test-driven-development/     # Workflow: TDD
│   ├── systematic-debugging/        # Workflow: debug
│   ├── ansible-code-review/         # Workflow: review
│   ├── verification-before-completion/ # Workflow: verify
│   ├── finishing-a-development-branch/ # Workflow: ship
│   ├── ansible-architecture/        # Domain: core patterns
│   ├── ansible-molecule/            # Domain: testing
│   ├── ansible-module-dev/          # Domain: Python modules
│   ├── aap-config-as-code/          # Domain: AAP CaC
│   ├── ansible-dev-tools/           # Domain: ADT toolchain
│   ├── ansible-git-workflow/        # Domain: git + CalVer
│   ├── ansible-cicd/                # Domain: CI/CD
│   ├── ansible-security/            # Domain: security
│   ├── ansible-ee-builder/          # Domain: EE lifecycle
│   ├── ansible-release-management/  # Domain: releases
│   └── ansible-code-style/          # Domain: formatting
└── sources/                         # Reference materials
```

## Contributing

### Adding a New Skill

1. Create a directory under `skills/<skill-name>/`
2. Write `SKILL.md` with frontmatter (`name`, `description`) and content
3. Add the skill to the catalog in `using-ansible-superpowers/SKILL.md`
4. Update `AGENTS.md` and `claude.md` quick reference tables
5. Test by asking an agent to invoke it

### Skill Types

**Rigid** skills (architect, TDD, debugging, code-review) must be followed exactly. They enforce discipline.

**Flexible** skills (architecture patterns, tooling) can be adapted to context. Their principles guide, not dictate.

The skill itself declares which type it is.

## Acknowledgments

- [Superpowers](https://github.com/obra/superpowers) by Jesse Vincent -- the framework this is based on
- [rh1-docs](https://github.com/djdanielsson/rh1-docs) -- Cloud-Native Ansible Lifecycle Platform documentation
- [Red Hat Communities of Practice](https://redhat-cop.github.io/automation-good-practices/) -- Ansible good practices
- [Ansible Development Tools](https://ansible.readthedocs.io/projects/dev-tools/) -- The ADT toolchain

## License

MIT
