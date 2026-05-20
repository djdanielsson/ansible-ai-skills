---
name: ansible-architect
description: >-
  Enterprise Ansible architecture and development standards. Use when writing
  or reviewing playbooks, roles, collections, or inventories. Covers
  idempotency, role engineering, orchestration, collections, variable
  management, linting, and security. Use eagerly for any Ansible task.
---
# Enterprise Ansible Architecture

## Critical Reasoning Requirements

Before writing any Ansible code, you MUST reason through these questions explicitly:

1. **What is the current state?** What exists on the target system right now?
2. **What is the desired state?** What should exist after convergence?
3. **What is the delta?** What specifically needs to change?
4. **Is this idempotent?** Will a second run produce zero changes?
5. **What module handles this?** Is there a native module, or must you use command/shell?

Do NOT skip this reasoning. 44% of LLM Ansible failures stem from state reconciliation errors.

## Core Architectural Rules

* **Declarative over Imperative:** YAML is data serialization declaring desired state, not an imperative script. Never treat a playbook as a sequence of commands to execute.
* **The 4-Level Hierarchy:** Landscape (workflows) > Type (host classification playbooks) > Function (roles) > Component (task files).
* **No Numbered Playbooks:** Never create `1_build.yml`, `2_config.yml`. This violates declarative state paradigms.
* **Idempotency is Absolute:** Code must report zero changes if the system is already in the desired state. Test by running twice.

## Module Selection (Critical)

24% of LLM Ansible failures come from wrong module or wrong parameters. Follow this decision tree:

1. **Does a native module exist?** Check with `ansible-doc -l | grep <keyword>`. ALWAYS prefer native modules over command/shell.
2. **Use FQCN:** Always `ansible.builtin.apt`, never bare `apt`.
3. **Understand state parameters:** `state: present` is already idempotent. Do NOT add `when` guards around idempotent modules.
4. **Know list parameters:** `ansible.builtin.apt` accepts `name: [pkg1, pkg2]`. Do NOT write loops for this.

### Module Selection Hierarchy

| Need | Correct Module | NEVER Do This |
|------|---------------|---------------|
| Install packages | `ansible.builtin.apt/dnf/yum` with `state: present` | `shell: apt install -y pkg` |
| Manage services | `ansible.builtin.systemd` with `state: started, enabled: true` | `shell: systemctl enable --now svc` |
| Edit config lines | `ansible.builtin.lineinfile` or `template` | `shell: sed -i ...` |
| Create users | `ansible.builtin.user` | `shell: useradd ...` |
| Manage files | `ansible.builtin.file` with `state: directory/file/link/absent` | `shell: mkdir -p ...` |
| Copy content | `ansible.builtin.copy` or `ansible.builtin.template` | `shell: cp ...` or `cat > file` |
| Download files | `ansible.builtin.get_url` | `shell: curl -O ...` or `wget` |
| Manage cron | `ansible.builtin.cron` | `shell: echo "..." >> /etc/crontab` |
| Firewall rules | `ansible.posix.firewalld` or `community.general.ufw` | `shell: iptables ...` |

### When command/shell IS acceptable

Default to `ansible.builtin.command`. ONLY use `ansible.builtin.shell` if you need pipes (`|`), redirection (`>`), or glob expansion (`*`).

When using either, you MUST:
- Add a comment justifying why no native module exists
- Set `changed_when` based on actual output/rc
- Set `creates` or `removes` if applicable for idempotency

```yaml
# No native module for this vendor CLI tool
- name: Configure vendor appliance
  ansible.builtin.command:
    cmd: vendor-cli set-option feature=enabled
  register: vendor_result
  changed_when: "'already enabled' not in vendor_result.stdout"
```

## Role Engineering

* **Functional Abstraction:** Use `tasks/set_vars.yml` with `lookup('first_found', [...], skip=true)` to load OS-specific variables (e.g., `RedHat-8.yml`, `default.yml`).
* **Input Validation:** Define `meta/argument_specs.yml` with types, required status, defaults, and choices for every role variable.
* **Dry-Run Support:** Pass `supports_check_mode=True`. Use `check_mode: true` on registration tasks. Prefer `ansible.builtin.slurp` for data gathering in simulations.
* **Conditional Logic:** Always use `| bool` filter for bare variables in `when:`. Never do complex Jinja2 manipulation inline; use filter plugins.
* **Handlers:** Use handlers for actions triggered by change (service restarts, daemon reloads). Never restart services unconditionally in tasks. Always `notify` the handler from the task that changes config.

```yaml
# CORRECT: handler only fires when config actually changes
- name: Deploy nginx config
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: Restart nginx

# In handlers/main.yml:
- name: Restart nginx
  ansible.builtin.systemd:
    name: nginx
    state: restarted
```

## Playbook Orchestration

* **Never mix `roles:` and `tasks:`** in a single play. Use `roles:` for static compilation or `tasks:` with `include_role` for dynamic inclusion.
* **Tagging:** Apply tags to complete roles or operational concerns (`deploy`, `configure`). Never tag destructive partial tasks.
* **Debugging:** Every `debug` task MUST have `verbosity: 1`.

## Collection Architecture

* **Sourcing Priority:** Certified > Validated > Community > Custom.
* **Packaging:** Must include `README`, `LICENSE`, `meta/main.yml`, `meta/runtime.yml`.

## Inventory & Variable Management

* **Single Source of Truth:** Inventories represent desired state ("To-Be"), never current state ("As-Is").
* **No Extra Vars for State:** `extra_vars` are for ephemeral operations and debugging only.
* **Structured Inventories:** Use directory structure, not monolithic files.
* **Dynamic Grouping:** Loop over inventory structures. Never hardcode hostnames in playbooks.
* **Variable Precedence:** Role Vars (constants) > Inventory Vars (desired state) > Scoped Vars > Runtime Vars (facts) > Extra Vars (overrides only).

## Linting & Quality

* **Always validate:** `ansible-lint --profile production` and `ansible-playbook --syntax-check` before declaring work complete.
* **FQCN everywhere:** Never use deprecated bare module names or `collections:` keyword.
* **Bracket notation:** `my_dict['key']` not `my_dict.key` in Jinja2.
* **Modern loops:** `loop:` not `with_items:`.
* **No tabs.** Maintain key order. No deep nesting. No `../../` paths.

## Security

* **NEVER hardcode secrets.** Use Ansible Vault or external secret managers.
* **`no_log: true`** on any task handling secrets.
* **`validate_certs: true`** always (never `false` unless explicitly instructed for isolated test).
* **Use current module names.** Avoid deprecated aliases.

## AAP/AWX Operational Rules

* **Dependencies:** Define in `roles/requirements.yml` or `collections/requirements.yml`.
* **No `vars_prompt`.** Automation runs as a background daemon.
* **No `pause` without `timeout`.**
* **Performance:** `gather_facts: false` unless needed. Use `gather_subset` or fact cache. Use `synchronize` (rsync) for large payloads instead of `copy`.

## Documentation Standards

### Role README.md

Every role MUST have a README.md containing:

```markdown
# Role Name

Brief description of what this role does.

## Requirements

- Minimum Ansible version
- Required collections (FQCN)
- Platform requirements

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `role_var_name` | `default_value` | What it controls |

## Dependencies

List of other roles this depends on.

## Example Playbook

\`\`\`yaml
- hosts: servers
  roles:
    - role: namespace.collection.role_name
      vars:
        role_var: value
\`\`\`

## License

## Author Information
```

### meta/argument_specs.yml

Define every role variable with full type information:

```yaml
argument_specs:
  main:
    short_description: What this role does
    options:
      role_variable_name:
        type: str
        required: false
        default: "value"
        choices: ["option1", "option2"]
        description: What this variable controls
```

### Collection README.md

Must include: namespace, installation instructions, included content inventory (roles, modules, plugins), dependencies, and links to per-role docs.

### Playbook Documentation

- Header comment block with purpose, target hosts, and required variables
- Comments only on non-obvious `when` conditions or complex Jinja2 filters
- Never comment obvious tasks (do not write `# Install nginx` above an apt task for nginx)
