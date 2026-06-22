---
name: ansible-release-management
description: >-
  Use when creating releases, managing release manifests, promoting between
  environments, or coordinating atomic promotion of Ansible automation
  components. Covers CalVer synchronization and rollback procedures.
---

# Ansible Release Management

Manage atomic promotion of synchronized Ansible automation components across environments.

## Atomic Promotion Principle

All components are promoted together as a version-locked unit. A release is NOT just code -- it's code + EE + CaC configuration, all at the same version.

```
Release: 26.1.5-0
  ├── AAP Config (aap-config-as-code):  tag 26.1.5-0
  ├── Collection (automation-collection): tag 26.1.5-0
  └── EE Image (automation-ee):          tag 26.1.5-0
```

## Release Manifest

Every release produces a manifest file:

```yaml
# releases/release-26.1.5-0.yaml
version: "26.1.5-0"
created: "2026-01-05T10:00:00Z"
created_by: "tekton-pipeline"

components:
  aap_configuration:
    repository: "github.com/org/aap-config-as-code"
    ref_type: "tag"
    ref: "26.1.5-0"
    commit: "abc1234567890abcdef1234567890abcdef12345"

  automation_collection:
    repository: "github.com/org/automation-collection"
    ref_type: "tag"
    ref: "26.1.5-0"
    commit: "def4567890abcdef1234567890abcdef45678901"

  execution_environment:
    name: "automation-ee"
    registry: "quay.io"
    image: "quay.io/org/automation-ee:26.1.5-0"
    digest: "sha256:1234567890abcdef..."
    built_at: "2026-01-05T10:25:00Z"
    base_image: "quay.io/ansible/creator-ee:v0.20.1"

testing:
  unit_tests: "passed"
  integration_tests: "passed"
  molecule_tests: "passed"
  security_scan: "passed"

environments:
  dev:
    deployed_at: "2026-01-05T09:00:00Z"
  qa:
    deployed_at: "2026-01-05T11:00:00Z"
    validated: true
    approvals:
      - approver: "qa-lead@example.com"
        approved_at: "2026-01-05T14:30:00Z"
  prod:
    deployed_at: "2026-01-05T15:00:00Z"
    approved_by: "CAB"
    ticket: "CHG0001234"

artifacts:
  sbom: "https://artifactory.example.com/sbom/26.1.5-0.json"
  vulnerability_scan: "https://artifactory.example.com/scans/26.1.5-0.json"
```

## Manifest Validation Rules

| Field | Validation |
|-------|-----------|
| version | Matches CalVer regex |
| commit SHAs | Full 40-character hex strings |
| image digests | Format `sha256:<64 hex chars>` |
| image tags | No `:latest`, `:main`, `:master` in QA/Prod |
| component versions | All match the release version |

## Release Workflow

### Creating a Release

```bash
# 1. Ensure all tests pass on main
gh run list --workflow=ansible-test.yml --branch=main

# 2. Create release tag
git checkout main && git pull
git tag -a 26.1.5-0 -m "Release January 5, 2026

Features:
- Add webserver role
- Update monitoring integration

Testing: All molecule tests passed
Rollback: Revert to 26.1.4-0 if issues"

git push origin 26.1.5-0
```

### Promoting to QA

```bash
tkn pipeline start promote \
  -p VERSION=26.1.5-0 \
  -p FROM_ENVIRONMENT=dev \
  -p TO_ENVIRONMENT=qa
```

**Gates:**
- Dev tests passed
- Code review completed
- QA Lead approval

### Promoting to Production

```bash
tkn pipeline start promote \
  -p VERSION=26.1.5-0 \
  -p FROM_ENVIRONMENT=qa \
  -p TO_ENVIRONMENT=prod
```

**Gates:**
- QA testing completed and signed off
- Security scan passed
- CAB approval received
- Change window scheduled
- Rollback plan documented

## Rollback Procedure

### Via Pipeline (Recommended)

```bash
tkn pipeline start rollback \
  -p TARGET_VERSION=26.1.4-0 \
  -p ENVIRONMENT=prod
```

### Rollback Checklist

- [ ] Identify previous working version
- [ ] Verify previous EE image exists in registry
- [ ] Update AAP job template `scm_branch` to previous tag
- [ ] Update AAP `execution_environment` to previous EE
- [ ] Apply CaC changes via pipeline
- [ ] Run smoke tests
- [ ] Monitor for 1 hour
- [ ] Document incident and rollback in post-mortem
- [ ] Create new CalVer tag documenting the rollback

### Creating Audit Trail for Rollback

```bash
git tag -a 26.1.6-0 -m "Rollback to 26.1.4-0 state

Rolled back from: 26.1.5-0
Reason: Critical issue in webserver role
Rollback approved: CHG0001236
Incident: INC0005678"
```

## AAP Configuration Updates

When promoting, AAP projects and job templates are updated via CaC:

```yaml
# Per-environment configuration
controller_projects:
  - name: "Automation Collection - Prod"
    scm_url: "https://github.com/org/automation-collection"
    scm_branch: "26.1.5-0"
    scm_update_on_launch: false

controller_execution_environments:
  - name: "Automation EE - Prod"
    image: "quay.io/org/automation-ee:26.1.5-0"
    pull: "missing"
```

## Breaking Changes

Document breaking changes prominently:

```bash
git tag -a 26.2.1-0 -m "⚠️ BREAKING CHANGES
- Removed deprecated inventory format
- Changed role variable names: app_port -> webserver_port
See CHANGELOG.md for migration guide"
```

## Anti-Patterns

| Anti-Pattern | Why | Do Instead |
|-------------|-----|------------|
| Promoting components individually | Runtime mismatches | Atomic promotion |
| Missing release manifest | No audit trail | Generate manifest for every release |
| Skipping QA | Unknown production behavior | Always promote through all environments |
| Incomplete rollback | Partial state recovery | Roll back all components together |
| No rollback plan | Stuck in failed state | Document rollback before deploying |
