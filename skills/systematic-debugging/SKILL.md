---
name: systematic-debugging
description: >-
  Use when encountering any Ansible failure - playbook errors, Molecule test
  failures, lint violations, AAP job failures, or unexpected behavior.
  4-phase root cause investigation with Ansible-specific diagnostic tooling.
---

# Systematic Debugging for Ansible

**Iron Law: NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST.**

Do not guess. Do not apply "common fixes." Do not change code hoping the error goes away. Investigate systematically, form a hypothesis, test it, THEN implement the fix.

**Type: Rigid** -- follow the phases in order.

## Phase 1: Root Cause Investigation

Gather evidence before forming any hypothesis.

### Diagnostic Commands

| What Failed | Diagnostic Command | What To Look For |
|-------------|-------------------|------------------|
| **Playbook error** | `ansible-playbook -vvv playbook.yml --check` | Module args, connection details, Python errors |
| **Module failure** | `ansible-doc <module>` then compare args | Wrong parameters, missing required args, version issues |
| **Lint violation** | `ansible-lint --generate-config` then `ansible-lint -v` | Rule ID, file, line, violation explanation |
| **Molecule failure** | `molecule --debug test` | Which phase failed (create, converge, verify, idempotence) |
| **Variable undefined** | `ansible -m debug -a "var=my_var" -i inventory host` | Where variable should be defined, precedence |
| **Connection error** | `ansible -m ping host -vvv` | SSH config, credentials, host reachability |
| **Template error** | `ansible -m template -a "src=template.j2 dest=/tmp/test"` | Jinja2 syntax, undefined variables, filter errors |
| **AAP job failure** | Check job stdout in Controller UI or API | Host failures, credential issues, EE problems |
| **Handler not firing** | `ansible-playbook -vvv` and search for "notified" | Task didn't report changed, handler name mismatch |
| **Idempotency failure** | `molecule converge && molecule converge 2>&1 | grep changed` | Which tasks report changed on second run |

### Gather Facts

```bash
ansible -m ansible.builtin.setup -i inventory target_host
ansible -m ansible.builtin.setup -a "filter=ansible_os_family" target_host
```

### Check Module Documentation

```bash
ansible-doc ansible.builtin.dnf
ansible-doc -l | grep keyword
ansible-doc --type connection ssh
```

### Verbose Execution

```bash
ansible-playbook playbook.yml -vvv --diff 2>&1 | tee /tmp/debug.log
```

## Phase 2: Pattern Analysis

With evidence gathered, identify the pattern:

| Pattern | Likely Cause | Where To Look |
|---------|-------------|---------------|
| "No such file or directory" | Path wrong or file not templated yet | Task ordering, `creates:` conditions |
| "Undefined variable" | Wrong scope or missing default | `defaults/main.yml`, inventory, `host_vars/` |
| "Permission denied" | `become: true` missing or wrong user | Task-level or play-level become |
| "Module not found" | Missing collection or wrong FQCN | `requirements.yml`, collection paths |
| "changed" on second run | Non-idempotent task | Missing `changed_when`, shell commands |
| "could not resolve" | DNS or hostname issue | Inventory host definitions |
| "Timeout" | Slow operation or unreachable host | Connection settings, async tasks |
| Multiple tasks fail | Early task failure cascading | Check first failure, not last |

## Phase 3: Hypothesis and Testing

1. **Form exactly ONE hypothesis** -- not three. One.
2. **State it explicitly:** "I believe the failure is caused by X because evidence Y shows Z."
3. **Design a minimal test:** What single thing can I check to prove/disprove this?
4. **Run the test** -- not the full playbook, just the test.
5. **Evaluate:** Did the test confirm or refute the hypothesis?

If refuted, go back to Phase 2. Do NOT stack hypotheses.

### Minimal Testing Techniques

```bash
# Test a single task
ansible target -m ansible.builtin.dnf -a "name=nginx state=present" --check

# Test a single play/role with tags
ansible-playbook playbook.yml --tags "install" --check -vvv

# Test template rendering
ansible target -m ansible.builtin.template -a "src=template.j2 dest=/tmp/test" --check --diff

# Test variable resolution
ansible target -m ansible.builtin.debug -a "var=hostvars[inventory_hostname]"

# Test with molecule on a single scenario
molecule converge -s scenario_name
```

## Phase 4: Implementation

Only after root cause is confirmed:

1. **Fix the root cause** -- not a symptom, not a workaround
2. **Verify the fix:**
   - `ansible-lint --profile production` on changed files
   - `ansible-playbook --check --diff` for the specific play
   - `molecule converge && molecule verify` for the scenario
3. **Verify idempotency:**
   - `molecule converge` a second time -- expect 0 changes
4. **Check for side effects:**
   - Did fixing this break anything else?
   - Run full `molecule test` (destroy + create + converge + idempotence + verify)

## Common Ansible Debugging Mistakes

| Mistake | Correct Approach |
|---------|-----------------|
| Adding `ignore_errors: true` | Fix the error. `ignore_errors` hides bugs. |
| Changing module args randomly | Read `ansible-doc <module>` first. |
| Switching to `shell:` when module fails | The module is correct. Your args are wrong. |
| Restarting from scratch | Investigate what broke. Don't throw away work. |
| Adding `when: ansible_os_family == ...` to skip | If it should run, fix it. Don't condition it away. |
| Running full `molecule test` repeatedly | Run only the failing phase to iterate faster. |
