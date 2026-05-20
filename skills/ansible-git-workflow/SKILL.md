---
name: ansible-git-workflow
description: >-
  Use when managing git branches, versioning, tags, or promotions for Ansible
  automation projects. Covers trunk-based development with CalVer (YY.M.D-PATCH)
  tags, feature/fix/hotfix branches, atomic promotion, and rollback procedures.
---

# Ansible Git Workflow

Trunk-Based Development with CalVer tags for Ansible automation projects.

## Version Format

```
YY.M.D-PATCH
```

| Component | Description | Example |
|-----------|-------------|---------|
| **YY** | Two-digit year | `26` |
| **M** | Month (no leading zero) | `1` for January |
| **D** | Day (no leading zero) | `6` for the 6th |
| **PATCH** | Hotfix number | `0` (initial), `1` (hotfix) |

**Examples:** `26.1.6-0`, `26.1.6-1`, `26.2.15-0`

## Branch Structure

### Main Branch

- Single source of truth
- No direct commits (all changes via PR)
- Must pass CI/CD checks and code review
- Always in a releasable state

### Feature Branches

**Naming:** `feature/<description>` or `feat/<description>`

Short-lived (hours to days, not weeks). Merge frequently.

```bash
git checkout -b feature/add-webserver-role main
# develop, test locally
git push origin feature/add-webserver-role
# create PR -> review -> merge -> delete branch
```

### Fix Branches

**Naming:** `fix/<description>` or `bugfix/<description>`

### Hotfix Branches

**Naming:** `hotfix/<description>`

Branch from the release tag, not main:

```bash
git checkout -b hotfix/critical-security-fix 26.1.5-0
git commit -m "fix: patch critical vulnerability"
git checkout main && git merge hotfix/critical-security-fix
git tag -a 26.1.5-1 -m "Hotfix: security patch"
git push origin 26.1.5-1
```

## Promotion Workflow

One tag promotes through all environments:

| Environment | Tag | Action |
|-------------|-----|--------|
| Dev | `26.1.5-0` | Auto-deploy on tag creation |
| QA | `26.1.5-0` | Promote via pipeline after dev validation |
| Prod | `26.1.5-0` | Promote with CAB approval |

### Promotion Gates

**Dev -> QA:**
- Dev tests passed
- Code review completed
- QA Lead approval

**QA -> Prod:**
- QA testing completed
- Security scan passed
- CAB approval received
- Change window scheduled
- Rollback plan documented

## Component Synchronization

All components must use the same version tag:

```yaml
Release: 26.1.5-0
  ├── AAP Config:   26.1.5-0
  ├── Collection:   26.1.5-0
  └── EE Image:     26.1.5-0
```

Mismatched versions are a policy violation.

## Release Manifest

```yaml
version: "26.1.5-0"
created: "2026-01-05T10:00:00Z"
components:
  aap_configuration:
    commit: "abc123..."
    tag: "26.1.5-0"
  collections:
    version: "26.1.5-0"
  execution_environment:
    tag: "26.1.5-0"
    digest: "sha256:fedcba..."
environments:
  dev:
    deployed_at: "2026-01-05T09:00:00Z"
  qa:
    deployed_at: "2026-01-05T11:00:00Z"
    validated: true
  prod:
    deployed_at: "2026-01-05T15:00:00Z"
    approved_by: "CAB"
```

## Rollback

```bash
# Via Tekton (recommended)
tkn pipeline start rollback -p TARGET_VERSION=26.1.4-0 -p ENVIRONMENT=prod

# Manual: create new tag pointing to previous state
git tag -a 26.1.6-0 -m "Rollback to 26.1.5-0 state
Rolled back from: 26.1.5-1
Reason: Critical issue
Rollback approved: CHG0001236"
```

## Tag Best Practices

- Always use annotated tags with descriptive messages
- Tags are immutable -- never delete or move them
- Include features, testing status, and rollback info in tag messages
- Validate tag format: `^[0-9]{2}\.(1[0-2]|[1-9])\.(3[01]|[12][0-9]|[1-9])-[0-9]+$`

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Do Instead |
|-------------|-------------|------------|
| Long-lived environment branches | Branches diverge, merge conflicts | Use main + tags |
| Using `latest` in production | Non-reproducible, mutable | Use version tags |
| Reusing or moving tags | Breaks audit trail | Create new PATCH version |
| Feature branches > 2 days | Integration conflicts | Merge frequently |

## Git Configuration

```yaml
# Branch protection for main
required_pull_request_reviews:
  required_approving_review_count: 1
required_status_checks:
  contexts: ["pre-commit", "ansible-lint", "molecule-test"]
allow_force_pushes: false
allow_deletions: false
```
