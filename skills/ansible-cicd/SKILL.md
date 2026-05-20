---
name: ansible-cicd
description: >-
  Use when working on CI/CD pipelines for Ansible automation. Covers GitHub
  Actions for testing/linting and Tekton Pipelines for building/releasing,
  per-repository workflow design, quality gates, and troubleshooting.
---

# Ansible CI/CD

CI/CD workflows for Ansible automation projects using GitHub Actions (quality gates) and Tekton Pipelines (build/release).

## Architecture Split

**GitHub Actions** (Testing -- runs on every push/PR):
- Pre-commit validation
- Linting (ansible-lint, yamllint, Python linters)
- Testing (sanity, unit, integration, Molecule)
- Security scanning (secret detection, vulnerability scanning)
- PR validation and auto-labeling

**Tekton Pipelines** (Building & Releasing -- runs on cluster):
- Build collections
- Build execution environments
- Create release manifests
- Publish to registries (Galaxy, Quay.io)
- Promote between environments (dev -> QA -> prod)

## Workflow Matrix by Repository Type

### AAP Config as Code

| Workflow | Triggers | What It Does |
|----------|----------|-------------|
| pre-commit | push, PR | Pre-commit hooks, YAML lint, secret detection |
| ansible-lint | push, PR | Ansible-lint production profile, syntax checks |
| pr-validation | PR | Environment change detection, hardcoded secret check, idempotency |
| deploy-dev | manual | Deploy to AAP Dev with confirmation |

### Ansible Collection

| Workflow | Triggers | What It Does |
|----------|----------|-------------|
| pre-commit | push, PR | Pre-commit hooks |
| ansible-test | push, PR | Lint, sanity, unit tests, collection build |
| molecule-test | push, PR, manual | All Molecule scenarios in parallel |
| release | tags | Build, test install, publish to Galaxy |

### Execution Environment

| Workflow | Triggers | What It Does |
|----------|----------|-------------|
| validate-ee | push, PR | EE definition validation, version pinning checks |
| build-ee | push, PR, manual | Build with ansible-builder, security scan (Trivy) |
| release-ee | tags | Build, scan, push to Quay.io with digest |

### Release Manifest

| Workflow | Triggers | What It Does |
|----------|----------|-------------|
| validate-manifest | push, PR | Structure, version, SHA, digest validation |
| create-release | tags, manual | Create manifest, validate, commit, GitHub release |

## Quality Gates

Every PR must pass before merge:

```yaml
required_status_checks:
  contexts:
    - pre-commit
    - ansible-lint
    - molecule-test    # if applicable
    - ansible-test     # if collection
    - validate-ee      # if EE
```

## Secrets Management in CI

| Repository | Required Secrets |
|-----------|-----------------|
| cluster-config | None (validation only) |
| aap-config-as-code | `AAP_DEV_HOST`, `AAP_DEV_USERNAME`, `AAP_DEV_PASSWORD` (optional) |
| automation-collection | `GALAXY_API_KEY` (optional) |
| automation-ee | `QUAY_USERNAME`, `QUAY_PASSWORD` |
| release-manifest | None (uses `GITHUB_TOKEN`) |

```bash
# Set secrets via GitHub CLI
gh secret set QUAY_USERNAME --body "your-username" --repo owner/repo
gh secret set QUAY_PASSWORD --body "your-token" --repo owner/repo
```

## Workflow Design Patterns

### Caching for Speed

```yaml
- name: Cache pre-commit
  uses: actions/cache@v3
  with:
    path: ~/.cache/pre-commit
    key: pre-commit-${{ hashFiles('.pre-commit-config.yaml') }}
```

### Parallel Molecule Scenarios

```yaml
strategy:
  matrix:
    scenario: [default, centos, ubuntu]
  fail-fast: false
```

### Conditional Path-Based Execution

```yaml
on:
  push:
    paths:
      - 'roles/**'
      - 'plugins/**'
```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Pre-commit fails in CI but passes locally | `pre-commit autoupdate` then `pre-commit run --all-files` |
| Secret not found | `gh secret list --repo owner/repo`, then set missing secret |
| Permission denied | Add `permissions: {contents: write, pull-requests: write}` to workflow |
| Workflow not triggering | Check `paths` filter matches, branch name matches trigger |
| Matrix job failure | Check specific job logs, test locally with same params |

### Debug Logging

```bash
# Enable debug mode
gh secret set ACTIONS_RUNNER_DEBUG --body "true" --repo owner/repo
gh secret set ACTIONS_STEP_DEBUG --body "true" --repo owner/repo

# Re-run with debug
gh run rerun <run-id> --debug
```

## Constitutional Compliance

Every CI/CD workflow enforces:
- **Article I (GitOps):** All config in Git
- **Article III (Atomic Promotion):** Version locking
- **Article IV (Production Quality):** Comprehensive testing
- **Article V (Zero Trust):** No secrets in code, security scanning
