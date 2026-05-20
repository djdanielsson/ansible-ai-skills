---
name: ansible-security
description: >-
  Use when handling secrets, credentials, RBAC, supply chain security, or
  security scanning in Ansible automation. Covers zero-trust model, secret
  management, APME policy enforcement, and compliance reporting.
---

# Ansible Security

Zero-trust security model for Ansible automation platforms.

**Constitutional Article V: No secrets in Git. Least-privilege access. Always.**

## Secret Management

### The Rule

**No plaintext secrets in Git.** Not in comments, not in variable names that hint at values, not "just for dev." NEVER.

Vault-encrypted secrets (`ansible-vault`) are acceptable in Git for local development and non-production workflows. For CI/CD and production, use external secret managers (OCP Secrets, HashiCorp Vault, AAP Credentials).

### Where Secrets Live

| Environment | Secret Storage | How Accessed |
|------------|---------------|--------------|
| **Local dev** | `ansible-vault` encrypted files (committed to Git) | `--ask-vault-pass` or vault password file |
| **CI/CD** | GitHub Secrets / OCP Secrets (external) | Injected as environment variables or mounted files |
| **AAP** | Controller Credentials (external) | Referenced by name in job templates |
| **Kubernetes** | OCP Secret objects (encrypted at rest) | Mounted as files in pods |

### Vault Usage

```bash
# Encrypt a variable file
ansible-vault encrypt group_vars/all/vault.yml

# Encrypt a single string
ansible-vault encrypt_string '<YOUR_SECRET>' --name 'vault_db_password'
```

Reference vault variables in playbooks:

```yaml
db_password: "{{ vault_db_password }}"
```

### Credential Patterns

```yaml
# CORRECT: Reference credential by name in AAP
controller_templates:
  - name: "Deploy App"
    credentials:
      - "Production SSH Key"
      - "Database Credentials"

# WRONG: Hardcoded values
controller_templates:
  - name: "Deploy App"
    extra_vars:
      db_password: "PLAINTEXT_SECRET_NEVER_DO_THIS"
```

### no_log Usage

```yaml
# Use no_log on tasks that handle sensitive data
- name: Set database password
  ansible.builtin.user:
    name: dbadmin
    password: "{{ vault_db_password | password_hash('sha512') }}"
  no_log: true

# Also on tasks that DISPLAY sensitive data
- name: Create API token
  ansible.builtin.uri:
    url: "https://api.example.com/token"
    method: POST
    body:
      username: "{{ vault_api_user }}"
      password: "{{ vault_api_pass }}"
    body_format: json
  register: api_result
  no_log: true
```

## RBAC Enforcement

### Least Privilege Principle

| Role | Can Do | Cannot Do |
|------|--------|-----------|
| **Developer** | Push branches, create PRs | Merge to main, create tags |
| **Reviewer** | Approve PRs, merge | Create tags, deploy to prod |
| **Release Manager** | Create tags, promote to QA | Deploy to prod without CAB |
| **Platform Admin** | Full access | N/A |

### Service Account Permissions

```yaml
# Tekton service accounts with minimal permissions
tekton-pr-sa:      # PR validation -- no secret access
tekton-cac-sa:     # CaC deployment -- read secrets, run pods
tekton-promotion-sa: # Promotion -- read secrets, push images
```

### Verification

```bash
# Check what a service account can do
oc auth can-i --as=system:serviceaccount:dev-tools:tekton-pr-sa \
  get secrets -n dev-tools
# Expected: no (PR pipeline doesn't need secrets)
```

## APME (Ansible Policy and Migration Engine)

Multi-validator static analysis for architectural compliance:

### Validators

| Validator | What It Checks |
|-----------|---------------|
| **Python Validator** | Module structure, import patterns, deprecated APIs |
| **Ansible-Runtime** | FQCN usage, module compatibility, collection requirements |
| **OPA Rego** | Policy rules: no `ignore_errors` without justification, no `shell` without `changed_when` |
| **Gitleaks** | Secret patterns in all files |

### Key APME Rules

```rego
# No shell without changed_when
deny[msg] {
  input.tasks[i].module == "ansible.builtin.shell"
  not input.tasks[i].changed_when
  msg := sprintf("Task '%s' uses shell without changed_when", [input.tasks[i].name])
}

# No ignore_errors without justification
deny[msg] {
  input.tasks[i].ignore_errors == true
  not input.tasks[i].tags[_] == "ignore_errors_justified"
  msg := sprintf("Task '%s' uses ignore_errors without justification", [input.tasks[i].name])
}
```

## Supply Chain Security

### EE Image Security

```bash
# Scan images for vulnerabilities
trivy image quay.io/myorg/automation-ee:26.1.5-0

# Generate SBOM
syft quay.io/myorg/automation-ee:26.1.5-0 -o json > sbom.json

# Scan SBOM for known vulnerabilities
grype sbom:sbom.json
```

### Base Image Validation

- Use official AAP base images only
- Pin base image version (never use `latest`)
- Verify image digest in release manifest

### Dependency Pinning

```yaml
# requirements.yml -- always pin versions
collections:
  - name: ansible.posix
    version: "1.5.4"

# requirements.txt -- always pin versions
jmespath==1.0.1
netaddr==0.9.0
```

## Security Scanning in CI

### Pre-commit

```yaml
# .pre-commit-config.yaml entries for security
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
```

### PR Validation

Every PR checks:
- No plaintext secrets in changed files
- No credentials in variable files
- `no_log: true` on tasks with sensitive parameters
- Vault-encrypted files have `.vault` extension or are in `vault/` directory

## Network Policies

```yaml
# Isolate namespaces
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: aap-prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}
        - namespaceSelector:
            matchLabels:
              name: dev-tools
```

## Audit Trail

```bash
# Git audit trail
git log --all --pretty=format:'%h|%an|%ae|%ad|%s' --date=iso

# Who created a specific release?
git log --all --grep="26.1.6-0"

# What changed in prod AAP config?
git log -- group_vars/aap_prod.yml
```

## Security Checklist

Before any deployment:

- [ ] No secrets in Git (run gitleaks)
- [ ] `no_log: true` on all sensitive tasks
- [ ] Credentials reference external stores (not inline values)
- [ ] Service accounts use least-privilege permissions
- [ ] EE images scanned for vulnerabilities
- [ ] Dependencies version-pinned
- [ ] Base images from trusted registries
- [ ] Network policies isolate namespaces
- [ ] Audit logging enabled
