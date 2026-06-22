---
name: ansible-dev-tools
description: >-
  Ansible Development Tools (ADT) reference. Use when working with execution
  environments, project scaffolding, running playbooks, linting, signing,
  testing, pre-commit hooks, or the MCP server. Covers ansible-creator,
  ansible-navigator, ansible-builder, ansible-lint, ansible-sign, molecule,
  tox-ansible, pytest-ansible, and pre-commit integration. Referenced by the
  ansible-architect orchestrator.
---
# Ansible Development Tools (ADT)

The modern Ansible toolchain uses **Execution Environments** (OCI container images packaging ansible-core, Python, system libraries, and collections). Do not use legacy host-based global environments. The full toolchain is distributed as `ansible-dev-tools`.

## Tool Reference

### ansible-creator — Project Scaffolding

Bootstraps standard hierarchies for roles, playbooks, and collections.

```bash
ansible-creator init collection <namespace.name> <dir>
ansible-creator init playbook <namespace.name> <dir>
ansible-creator add resource role <role_name> --project <dir>
```

### ansible-dev-environment (ade) — Local Dependencies

Isolated virtual environments for collection development.

```bash
ade install -e . --venv .venv
```

Resolves from `requirements.txt`, `test-requirements.txt`, and `requirements.yml`.

### ansible-builder — Execution Environment Images

Builds OCI-compliant EEs from declarative V3 YAML schema.

```bash
ansible-builder build --tag custom-ee
```

Supports custom base images, inline requirements, and build steps (`prepend_base`, `append_base`).

### ansible-navigator — Execution Runner

Container-native execution engine and TUI. Supersedes all legacy CLI tools.

```bash
ansible-navigator run site.yml -m stdout
ansible-navigator inventory --list
ansible-navigator doc ansible.builtin.copy
```

Passes unrecognized parameters to the underlying engine.

### ansible-lint — Code Quality

Static analysis enforcing best practices. Configure via `.ansible-lint`.

```bash
ansible-lint --profile production
ansible-lint --offline -f sarif    # CI pipeline output
ansible-lint --fix                  # Auto-remediation
```

### ansible-sign — Supply Chain Security

GPG signing and verification via `MANIFEST.in`.

```bash
ansible-sign project gpg-sign .
ansible-sign project gpg-verify .
```

### molecule — Integration Testing

Validates automation against live infrastructure via lifecycle drivers.

```bash
molecule init scenario --driver-name docker
molecule test
molecule converge
molecule verify
```

See the `ansible-molecule` skill for testing best practices.

### tox-ansible — Matrix Testing

Generates test matrices across Python and ansible-core versions.

```bash
tox -e unit-py3.12-milestone --ansible
tox list --ansible
```

### pytest-ansible — Unit Testing

Fixtures for executing module behavior in Python test cases.

```bash
pytest --ansible-args="..."
```

Provides `ansible_adhoc` and `ansible_module` fixtures.

### ansible-mcp-server — AI Agent Integration

Implements Model Context Protocol to expose ADT to AI assistants. Enables agents to:
- Query environment diagnostics
- Analyze workspace errors
- Scaffold with ansible-creator
- Apply ansible-lint fixes
- Construct ansible-navigator parameters

### pre-commit — Git Hooks

Automate quality checks on every commit:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/ansible/ansible-lint
    rev: v24.7.0
    hooks:
      - id: ansible-lint
        args: [--profile, production]
  - repo: https://github.com/adrienverge/yamllint
    rev: v1.35.1
    hooks:
      - id: yamllint
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
```

Install and run:

```bash
pip install pre-commit
pre-commit install
pre-commit run --all-files
pre-commit autoupdate
```

## Lint Profiles

ansible-lint uses progressive profiles for gradual adoption:

| Profile | Inherits | Focus |
|---------|----------|-------|
| `min` | -- | Fatal parsing/syntax errors |
| `basic` | min | YAML formatting, key order, no tabs |
| `moderate` | basic | Task naming, readability |
| `safety` | moderate | Risky permissions, shell pipes |
| `shared` | safety | Packaging, metadata, distribution readiness |
| `production` | shared | FQCN enforcement, deprecated syntax, full compliance |

**Always use `--profile production` for enterprise code.**

## Recommended Workflow

```bash
# 1. Scaffold
ansible-creator init collection myorg.myapp ./myapp

# 2. Develop
# Write roles, tasks, variables

# 3. Lint continuously
ansible-lint --profile production

# 4. Test with Molecule
molecule test

# 5. Build (if collection)
ansible-builder build --tag my-ee

# 6. Run via navigator
ansible-navigator run site.yml -m stdout

# 7. Sign for distribution
ansible-sign project gpg-sign .
```
