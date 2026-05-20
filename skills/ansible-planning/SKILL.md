---
name: ansible-planning
description: >-
  Use when you have an approved design for Ansible automation and need to create
  an implementation plan before writing code. Enforces scaffold-then-fill
  methodology with lint gates and idempotency proof steps.
---

# Ansible Planning

Write comprehensive implementation plans assuming the engineer has zero context and follows the scaffold-then-fill methodology. Each task is one Ansible concern with lint validation after every step.

**Announce at start:** "I'm using the ansible-planning skill to create the implementation plan."

## Scope Check

If the design covers multiple independent roles or subsystems, break into separate plans -- one per role/component. Each plan should produce a working, testable artifact on its own.

## Scaffold-Then-Fill Enforcement

Plans MUST follow this order for every role:

1. **Scaffold first:** `ansible-creator init` or manual directory creation
2. **Variables first:** Fill `defaults/main.yml` and `meta/argument_specs.yml`
3. **One task at a time:** Each plan step adds exactly one task
4. **Lint after each task:** `ansible-lint --profile production`
5. **Check before apply:** `ansible-playbook --check` when possible
6. **Tests before more tasks:** Write Molecule verify for completed tasks

## Plan Document Header

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** Use subagent-driven-development to implement
> this plan task-by-task. Steps use checkbox syntax for tracking.

**Goal:** [One sentence describing converged state]

**Architecture:** [2-3 sentences about approach]

**Context:** [Role/Collection/Playbook being created or modified]

**Sub-persona:** [Ansible Developer / Module Author / AAP Admin / Platform Engineer]

---
```

## Task Structure

````markdown
### Task N: [Component Name]

**Files:**
- Create: `roles/role_name/tasks/main.yml`
- Create: `roles/role_name/defaults/main.yml`
- Test: `roles/role_name/molecule/default/verify.yml`

**State Reasoning:**
- Current: [What exists now]
- Desired: [What should exist after]
- Delta: [What changes]
- Module: [Which module handles this]

- [ ] **Step 1: Scaffold the role**

```bash
ansible-creator add resource role role_name --project .
```

- [ ] **Step 2: Define variables in defaults/main.yml**

```yaml
---
role_var_name: "default_value"
```

- [ ] **Step 3: Write the task**

```yaml
- name: Install required packages
  ansible.builtin.dnf:
    name: "{{ role_packages }}"
    state: present
```

- [ ] **Step 4: Lint**

```bash
ansible-lint --profile production roles/role_name/
```
Expected: 0 violations

- [ ] **Step 5: Write Molecule verify**

```yaml
- name: Verify package is installed
  ansible.builtin.command:
    cmd: rpm -q {{ item }}
  loop: "{{ role_packages }}"
  changed_when: false
  failed_when: result.rc != 0
  register: result
```

- [ ] **Step 6: Run Molecule converge + verify**

```bash
molecule converge && molecule verify
```

- [ ] **Step 7: Checkpoint with the user**

Summarize what changed, which validation steps passed, and whether the task is ready for a commit.

Only create a git commit if the user explicitly asks for one.
````

## Idempotency Proof Steps

Every plan MUST include an idempotency verification task:

```markdown
### Task N+1: Idempotency Verification

- [ ] **Step 1: Run converge a second time**

```bash
molecule converge
```
Expected: 0 changed tasks

- [ ] **Step 2: Run full molecule test**

```bash
molecule test
```
Expected: All phases pass including idempotence check
```

## No Placeholders

Every step must contain actual content. These are plan failures:
- "TBD", "TODO", "implement later"
- "Add appropriate error handling"
- "Write tests for the above" (without actual test code)
- "Similar to Task N" (repeat the code)
- Steps without code blocks for code steps

## Self-Review

After writing the complete plan:
1. **Design coverage:** Can you point to a task for each design requirement?
2. **Placeholder scan:** Any "TBD" or vague steps? Fix them.
3. **FQCN check:** All modules use fully qualified collection names?
4. **Lint step check:** Every code task followed by a lint step?
5. **Idempotency step:** Plan includes second-run verification?

## Execution Handoff

After saving the plan, offer execution:

**"Plan saved. Two execution options:**
1. **Subagent-Driven (recommended)** -- fresh subagent per task, Ansible-aware review between tasks
2. **Inline Execution** -- execute tasks in this session with checkpoints

**Which approach?"**

If Subagent-Driven: invoke `subagent-driven-development`
If Inline: execute tasks sequentially with lint validation after each.
