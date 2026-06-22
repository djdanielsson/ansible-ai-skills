---
name: verification-before-completion
description: >-
  Use before declaring any Ansible work complete. Proves the automation works
  through lint, idempotency, functional verification, and check mode. Must pass
  all gates before marking a task or branch as done.
---

# Verification Before Completion

Before declaring any Ansible work "done," prove it works. Not "I think it works." Prove it with evidence.

**Type: Rigid** -- every gate must pass or be explicitly documented as N/A with justification.

## Gates

### Gate 1: Lint Clean

```bash
ansible-lint --profile production .
```

**Expected:** 0 violations. Not "only warnings." Zero.

If violations exist, fix them. Do not proceed to Gate 2.

### Gate 2: Syntax Valid

```bash
# For playbooks
ansible-playbook --syntax-check playbook.yml

# For roles via molecule
molecule syntax
```

**Expected:** No syntax errors.

### Gate 3: Check Mode Works

```bash
ansible-playbook playbook.yml --check --diff
```

**Expected:** Shows what WOULD change without making changes. Any tasks that don't support check mode should use `check_mode: false` explicitly with a comment explaining why.

### Gate 4: Converge Succeeds

```bash
molecule converge
```

**Expected:** All tasks complete without error. Record the number of changed/ok/skipped tasks.

### Gate 5: Idempotency Proof

```bash
molecule converge  # run again
```

**Expected:** 0 changed tasks. This is the most important gate.

If any task reports `changed` on second run:
1. Identify which task
2. Determine why (usually missing `changed_when` or non-idempotent command)
3. Fix the root cause
4. Re-run Gates 4 and 5

### Gate 6: Functional Verification

```bash
molecule verify
```

**Expected:** All verify tests pass, confirming:
- Services are running
- Ports are listening
- Config files contain expected content
- Packages are installed
- Users/groups exist

### Gate 7: Full Test Cycle

```bash
molecule test
```

**Expected:** Complete destroy-create-converge-idempotence-verify-destroy cycle passes.

### Gate 8: Documentation Complete

Verify appropriate documentation exists:

- [ ] Role has `README.md` with variables table and example
- [ ] Module has `DOCUMENTATION`, `EXAMPLES`, `RETURN` blocks
- [ ] Complex playbooks have header comments
- [ ] CHANGELOG updated if applicable

## Verification Report

After all gates pass, produce a report:

```markdown
## Verification Report: [component]

| Gate | Status | Evidence |
|------|--------|----------|
| Lint | PASS | 0 violations |
| Syntax | PASS | clean |
| Check Mode | PASS | diff shows expected changes |
| Converge | PASS | X changed, Y ok |
| Idempotency | PASS | 0 changed on second run |
| Functional | PASS | N/N verify tests passed |
| Full Test | PASS | all phases clean |
| Documentation | PASS | README complete |

Verified at: [timestamp]
```

## What If a Gate Fails?

1. Do NOT skip the gate
2. Do NOT mark it as N/A unless genuinely not applicable
3. Fix the issue
4. Re-run from Gate 1 (lint may have changed)
5. Produce a clean report

## When Gates Are N/A

Some gates don't apply to all artifact types:

| Artifact | Gates That May Be N/A |
|----------|----------------------|
| **Filter plugin** | Check mode, Converge (use pytest instead) |
| **Inventory plugin** | Converge (test with `ansible-inventory`) |
| **CaC vars only** | Check mode (no tasks), Converge (depends on controller) |
| **EE definition** | Most gates (use `ansible-builder build` instead) |

Document why a gate is N/A. "I didn't feel like testing" is not N/A.
