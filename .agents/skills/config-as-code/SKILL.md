---
name: aap-config-as-code
description: >-
  AAP Configuration as Code using the infra.aap_configuration collection. Use
  when generating or reviewing declarative CaC for Ansible Automation Platform
  2.5+. Covers repository topology, dependency sequencing, credential
  management, and coding standards.
---
# AAP Configuration as Code

Exclusively use the declarative data modeling standards of the `infra.aap_configuration` collection. This is the unified architectural standard for AAP 2.5+.

## Repository Topology

Structure repositories based on Red Hat Communities of Practice templates:

* **Directory-Driven Isolation (`aap_configuration_template`):** Segregate platform objects by environment lifecycle.
* **Global Baselines:** `config/all/` for configurations applying symmetrically across all environments.
* **Environment Overrides:** `config/dev/`, `config/prod/` for environment-specific YAML.
* **Wildcard Variable Aggregation (`rh1-aap-config-as-code`):** Define dictionary objects appended to YAML lists (e.g., `controller_job_templates_*`) within `group_vars/`.
* **Strict SCM Mapping:** Pin project parameters to target branches (e.g., `scm_branch: develop`). Never allow unverified logic execution.

## Dependency Sequencing

Never write imperative tasks to create objects. Use the `dispatch` role to aggregate variables and sequence API object creation in this strict order:

1. **Organizations & RBAC** - Foundational tenancy before user access
2. **Execution Environments** - Before any automation references a runtime
3. **Credential Types** - Custom schemas before credentials use them
4. **Credentials** - Auth tokens before SCM pulls
5. **Projects** - Require Organizations + Credentials
6. **Inventories** - Require Organizations
7. **Job Templates** - Fail if any dependency is unresolved
8. **Workflow Templates** - Last; all job templates must exist

## Credential Management

* **Inline Encryption:** Use `secrets.yml` per environment with inline vault strings.
* **Encrypt via stdin:** `ansible-vault encrypt_string --stdin-name 'variable_name'` to avoid plaintext in shell history.
* **`!unsafe` tag:** Required for Custom Credential Type Jinja2 variables (e.g., `!unsafe "{{ vault_url }}"`). Prevents local evaluation; passes string verbatim to PostgreSQL for API injection.

## Coding Standards

* **File extension:** `.yml` only. Never `.yaml`.
* **Jinja whitespace:** `{{ var }}` not `{{var}}`.
* **Paths:** No trailing slashes (`my_path: /foo` not `/foo/`).
* **Naming:** Underscore separators (`_`) in role/playbook names. Dashes are misinterpreted by Python.
* **Task names:** Start with capital letter for log readability.
* **Python interpreter:** Explicit `ansible_python_interpreter: /usr/bin/python3.9` or higher.

## Variable Placement

* **`defaults/main.yml`:** Highly modifiable baselines initialized to empty arrays (`[]`).
* **`vars/main.yml`:** Rigid, non-negotiable logic in lowercase.

## Error Handling

* Never use `sys.exit()`. Use `fail_json()` or `fail_aws()` for structured failure payloads.
* Ensure idempotency by avoiding volatile dynamic variables (random hashes, timestamps) in configuration payloads.

## Documentation Standards

### Repository README.md

```markdown
# AAP Configuration as Code - [Environment/Org Name]

## Overview
What this repository manages (which Controller/Hub/EDA instances).

## Structure
config/
  all/          # Global baselines (all environments)
  dev/          # Development overrides
  staging/      # Staging overrides
  prod/         # Production overrides

## Applying Changes

1. Encrypt secrets: `ansible-vault encrypt_string --stdin-name 'var_name'`
2. Validate: `ansible-lint --profile production`
3. Dry run: `ansible-navigator run site.yml --check -m stdout`
4. Apply: `ansible-navigator run site.yml -m stdout`

## Object Dependency Order
Organizations → EEs → Credential Types → Credentials → Projects → Inventories → Job Templates → Workflows

## Credentials
See docs/credentials.md for credential inventory (never store actual secrets here).
```

### Per-Environment Documentation

Each environment directory should have a brief README listing:
- What objects differ from the global baseline
- Environment-specific credential references
- Any manual prerequisites (network access, DNS, etc.)

### Inline Comments

Use YAML comments only for:
- Explaining `!unsafe` usage and why it's needed
- Cross-environment reference notes
- Dependency explanations that aren't obvious from naming
