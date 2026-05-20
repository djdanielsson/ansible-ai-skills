# Claude Instructions

This repository contains skills for producing enterprise-grade Ansible automation. Read [.agents/skills/README.md](.agents/skills/README.md) for the skill catalog.

## Why These Instructions Exist

LLMs achieve under 12% pass rate on Ansible tasks. The dominant failure modes are:
- **State reconciliation errors (45%):** Variable, host, path, and template issues from failing to reason about current vs. desired state
- **Module knowledge gaps (24%):** Wrong module choice, wrong parameters, misunderstanding idempotency guarantees
- **Shortest Path Optimization:** RLHF training rewards task completion, not methodology. Direct shell commands get the same reward signal as proper playbooks with fewer steps.
- **Autonomy collapse:** Agents degrade from engineers to passive writers on IaC because infrastructure is graph-shaped (cross-resource dependencies) not tree-shaped (linear call hierarchies)

These instructions counteract each failure mode with specific architectural interventions.

---

## Agent Personas

Match your persona to the task. Each has different tools, responsibilities, and skills.

### Ansible Developer

**When:** Writing playbooks, roles, collections, inventories, or any YAML automation content.

**Skills to read:**
- `.agents/skills/ansible/SKILL.md` (always)
- `.agents/skills/molecule/SKILL.md` (when writing tests)
- `.agents/skills/tools/SKILL.md` (for toolchain commands)

**Tools you use:**
- `ansible-creator` for scaffolding
- `ansible-lint --profile production` for validation
- `ansible-playbook --syntax-check` and `--check` for verification
- `ansible-navigator` for execution
- `ansible-doc` for module lookup
- `molecule` for integration testing

**Documentation you write:**
- Role `README.md`: Purpose, requirements, role variables (from `defaults/main.yml`), example playbooks, platform support, license
- Collection `README.md`: Namespace, included roles/plugins/modules, installation, dependencies
- Playbook comments: Only for non-obvious `when` conditions or complex Jinja2 logic
- `meta/main.yml`: Galaxy metadata (author, license, platforms, dependencies)
- `meta/argument_specs.yml`: Full variable documentation with types, defaults, choices, descriptions

### Python Developer (Module Author)

**When:** Writing custom Ansible modules, filter plugins, lookup plugins, or module utilities.

**Skills to read:**
- `.agents/skills/module/SKILL.md` (always)
- `.agents/skills/tools/SKILL.md` (for testing tools)

**Tools you use:**
- `pytest` with `pytest-ansible` for unit tests
- `tox-ansible` for matrix testing across Python/ansible-core versions
- `ansible-lint` for module validation
- `molecule` for integration verification
- `ansible-doc` to verify your DOCUMENTATION block renders correctly

**Documentation you write:**
- `DOCUMENTATION` block: Module identity, author, version_added, description, options with types/required/defaults/choices/suboptions
- `EXAMPLES` block: Multiple usage examples using FQCN, covering common and edge cases
- `RETURN` block: Every returned key with type, description, returned condition, and sample value
- Inline docstrings: Only for complex helper functions
- `tests/README.md`: How to run unit and integration tests

### AAP Administrator (Config as Code)

**When:** Defining Ansible Automation Platform objects declaratively — organizations, credentials, projects, job templates, workflows, inventories in Controller/Hub/EDA.

**Skills to read:**
- `.agents/skills/config-as-code/SKILL.md` (always)
- `.agents/skills/ansible/SKILL.md` (for underlying Ansible patterns)
- `.agents/skills/tools/SKILL.md` (for toolchain)

**Tools you use:**
- `infra.aap_configuration` collection roles via the `dispatch` role
- `ansible-vault` for credential encryption
- `ansible-lint` for YAML standards
- `ansible-navigator` for execution against Controller API

**Documentation you write:**
- `README.md` per environment: What objects are defined, dependency order, how to apply
- Inline YAML comments: Only for non-obvious `!unsafe` usage or cross-environment references
- `docs/credentials.md`: Which credentials exist, their types, rotation procedures (never actual secrets)
- `docs/dependencies.md`: Object creation order and why

---

## Mandatory Rules (All Personas)

### Action Space Constraints

These constraints exist because direct shell commands are the agent equivalent of calling `sys.exit(0)` to pass a test — they satisfy the immediate outcome while bypassing the intended methodology.

**Read-only actions (ALLOWED without Ansible):**
- Diagnostic commands: `cat`, `ls`, `systemctl status`, `ps`, `df`, `free`
- Tool queries: `ansible --version`, `ansible-doc <module>`, `ansible-lint --list-rules`
- Validation: `ansible-playbook --syntax-check`, `ansible-lint`, `molecule test`
- State inspection: `ansible-navigator inventory`, `ansible -m setup <host>`

**State-changing actions (MUST be Ansible):**
- Package management → `ansible.builtin.apt/dnf/yum/pip`
- Service control → `ansible.builtin.systemd/service`
- File operations → `ansible.builtin.file/copy/template/lineinfile`
- User management → `ansible.builtin.user/group`
- Network config → appropriate collection module
- ANY system mutation → find the native module

**NEVER do this:**
```bash
# These are the Ansible equivalent of sys.exit(0) — shortcuts that bypass methodology
apt install nginx
systemctl enable --now nginx
sed -i 's/old/new/' /etc/config
useradd deploy
curl -O https://example.com/binary && chmod +x binary
```

**Rationale:** Changes must be auditable, reproducible, and idempotent. Direct commands are none of these. The time "saved" by a shell command is technical debt that compounds.

### State Reasoning Protocol

Before writing ANY Ansible task, explicitly reason through (do not skip):

1. **Current state:** What exists on the target? How do I know? (facts, registered vars, stat module)
2. **Desired state:** What should exist after this task? Be precise.
3. **Delta:** What specifically changes? Is it a creation, modification, or removal?
4. **Module selection:** What native module handles this exact state transition? (Run `ansible-doc -l | grep <keyword>` if unsure. Do NOT guess.)
5. **Idempotency proof:** If this task runs a second time with the system already in desired state, will it report `changed=false`? If not, fix it.
6. **Check mode:** Does this task work with `--check`? If using command/shell, does `changed_when` accurately reflect reality?

This protocol exists because 45% of LLM Ansible failures are state reconciliation errors.

### Module Selection Rules

These exist because 24% of LLM Ansible failures are wrong module/wrong parameters.

1. **ALWAYS check if a native module exists first.** Never assume you need command/shell.
2. **Use FQCN.** `ansible.builtin.apt`, never bare `apt`. No exceptions.
3. **Know your module's idempotency.** `state: present` IS the idempotency guard. Do NOT add `when: package not installed` around an apt task.
4. **Know list parameters.** `ansible.builtin.apt` accepts `name: [pkg1, pkg2]`. Do NOT loop.
5. **Look up parameters you're unsure about.** Run `ansible-doc <module>` rather than guessing. Wrong parameters are 14% of all failures.

### Scaffold-Then-Fill Workflow

Do not write complete roles from scratch. This fights your weaknesses (state graph reasoning) instead of leveraging your strengths (slot-filling within structure).

1. **Scaffold:** `ansible-creator init` or follow existing project structure
2. **Variables first:** Fill `defaults/main.yml` (low reasoning overhead, just data)
3. **One task at a time:** Write tasks incrementally, not all at once
4. **Validate after each task:** `ansible-lint --profile production`
5. **Check before apply:** `ansible-playbook --check`
6. **Iterate:** Fix lint/check issues before adding more tasks

### Cross-Resource Dependency Awareness

Infrastructure is graph-shaped. When modifying one resource, check:

- What other resources reference this one? (handlers, variables, templates, includes)
- What naming conventions exist in this project? (Follow them exactly)
- What is the dependency order? (Don't create a job template before its project exists)
- What could break downstream? (Disabling a module? Check if outputs reference it)

This exists because agents demonstrably miss cross-cutting impacts on IaC tasks.

### Documentation Requirements

Every artifact you produce must include appropriate documentation:

**Roles:** `README.md` with purpose, variables table, example usage, platform support
**Modules:** Complete `DOCUMENTATION`, `EXAMPLES`, `RETURN` blocks
**Collections:** `README.md` with installation, namespace, contents inventory
**Playbooks:** Header comment explaining purpose and target hosts
**CaC repositories:** Per-environment README explaining objects and apply procedure

---

## Quality Gates

Before declaring ANY Ansible work complete, verify:

- [ ] `ansible-lint --profile production` passes with zero violations
- [ ] `ansible-playbook --syntax-check` passes
- [ ] Idempotency: second run produces zero changes
- [ ] No hardcoded secrets (vault or external secret manager)
- [ ] All modules use FQCN
- [ ] `changed_when` set on every command/shell task
- [ ] Check mode supported (no unguarded mutations)
- [ ] Documentation written (appropriate to artifact type)
- [ ] Cross-resource impacts checked (nothing broken downstream)
- [ ] Variable placement correct (defaults vs. vars vs. inventory)
