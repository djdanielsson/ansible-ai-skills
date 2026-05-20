---
name: ansible-code-review
description: >-
  Use after implementation to review Ansible code for production readiness.
  Comprehensive checklist covering lint, FQCN, idempotency, security, style,
  APME policy compliance, and constitutional principles.
---

# Ansible Code Review

Systematic review of Ansible automation for production readiness. Run through every section -- do not skip any.

**Type: Rigid** -- complete every section of the checklist.

## Review Process

1. Read all changed files
2. Run automated checks
3. Walk through manual checklist
4. Document findings as PASS / FAIL / N/A
5. Request fixes for any FAIL item before approving

## Automated Checks (Run First)

```bash
# Lint with production profile
ansible-lint --profile production .

# Syntax check all playbooks
find . -name "*.yml" -path "*/playbooks/*" -exec ansible-playbook --syntax-check {} \;

# FQCN check (prefer ansible-lint fqcn rule, or use this heuristic)
ansible-lint -R -r fqcn . 2>/dev/null || \
  grep -rn "^\s\+\w\+:$" --include="*.yml" roles/ tasks/ | grep -v "\." | grep -v "name:\|when:\|register:\|tags:\|become:\|notify:\|listen:\|block:\|rescue:\|always:\|loop:\|vars:" && echo "WARN: Possible short module names found"

# Verify no secrets in plain text
grep -rn "password\|secret\|token\|api_key" --include="*.yml" | grep -v "vault\|no_log\|lookup\|{{ " && echo "WARN: Possible plaintext secrets"
```

## Manual Checklist

### Module Usage
- [ ] All modules use FQCN (`ansible.builtin.copy`, not `copy`)
- [ ] No `shell:` or `command:` when a native module exists
- [ ] `changed_when:` on every `command:` / `shell:` / `raw:` task
- [ ] `creates:` / `removes:` used where applicable
- [ ] Module args match current documentation (`ansible-doc <module>`)

### Idempotency
- [ ] Second run produces zero changes
- [ ] No `creates:` used to mask non-idempotent commands
- [ ] `state:` parameter explicitly set (not relying on module defaults)
- [ ] File modes explicitly set where relevant
- [ ] Template changes trigger handlers (not unconditional restarts)

### Variables
- [ ] User-facing defaults in `defaults/main.yml`
- [ ] Internal constants in `vars/main.yml`
- [ ] Variable names prefixed with role name to avoid collision
- [ ] Sensitive variables documented as requiring vault
- [ ] No hardcoded values that should be variables

### Security
- [ ] `no_log: true` on tasks handling sensitive data
- [ ] Secrets managed through vault or external lookup
- [ ] No credentials in version control
- [ ] File permissions explicitly set (especially for config files)
- [ ] `become:` used only where required (least privilege)

### Error Handling
- [ ] `failed_when:` for tasks with ambiguous return codes
- [ ] `block: / rescue: / always:` for critical multi-step operations
- [ ] Error messages are actionable (user knows what to fix)
- [ ] `ansible.builtin.fail` with clear `msg:` for precondition failures

### Style (per ansible-code-style skill)
- [ ] Task names are descriptive (sentence case, explains desired state)
- [ ] YAML uses consistent indentation (2 spaces)
- [ ] `true` / `false` not `yes` / `no`
- [ ] Jinja2 has spaces inside braces: `{{ var }}` not `{{var}}`
- [ ] No trailing whitespace
- [ ] One blank line between tasks

### Tags and Documentation
- [ ] Tags applied consistently for selective execution
- [ ] Role README documents all variables, dependencies, platforms
- [ ] Complex logic has comments explaining WHY (not what)
- [ ] Example playbook in README or `tests/`

### Testing
- [ ] Molecule scenarios exist for main functionality
- [ ] Verify tests check functional behavior (not just file existence)
- [ ] Negative tests for error handling paths
- [ ] CI pipeline runs `molecule test` (not just converge)

### APME Policy Compliance
- [ ] No `ansible.builtin.shell` or `ansible.builtin.command` without `changed_when`
- [ ] No `ansible.builtin.raw` without documented justification
- [ ] No deprecated modules (check `ansible-doc -l --type module`)
- [ ] Collection version pinned in `requirements.yml`
- [ ] No `ignore_errors: true` without documented justification

### Constitutional Compliance
- [ ] **GitOps First:** All config is in Git, no manual changes assumed
- [ ] **Separation of Duties:** Platform vs application concerns separated
- [ ] **Atomic Promotion:** Components version-locked where applicable
- [ ] **Production-Grade Quality:** Idempotent, tested, documented
- [ ] **Zero-Trust Security:** Least privilege, no secrets in Git

## Review Output Format

```markdown
## Code Review: [component name]

### Automated Results
- ansible-lint: PASS/FAIL (N violations)
- syntax-check: PASS/FAIL
- FQCN: PASS/FAIL
- secret scan: PASS/FAIL

### Manual Review
| Category | Status | Notes |
|----------|--------|-------|
| Module Usage | PASS/FAIL | details |
| Idempotency | PASS/FAIL | details |
| Variables | PASS/FAIL | details |
| Security | PASS/FAIL | details |
| Error Handling | PASS/FAIL | details |
| Style | PASS/FAIL | details |
| Testing | PASS/FAIL | details |
| APME | PASS/FAIL | details |
| Constitutional | PASS/FAIL | details |

### Required Changes
1. [specific change needed, file, line]

### Recommendations (non-blocking)
1. [improvement suggestion]
```
