---
name: finishing-a-development-branch
description: >-
  Use when all implementation and verification is complete and you need to
  finalize the git branch for merge. Handles commit hygiene, squashing,
  changelog, and PR preparation following trunk-based development.
---

# Finishing a Development Branch

Finalize git history and prepare for merge following trunk-based development practices.

## Pre-Flight Check

Before finishing:

1. **All verification gates passed** -- invoke `verification-before-completion` if not done
2. **Code review complete** -- invoke `ansible-code-review` if not done
3. **No uncommitted changes** -- `git status` is clean
4. **Branch is up to date** -- `git pull --rebase origin main`

## Branch Naming Convention

Branches follow the pattern:
- `feature/<description>` -- new functionality
- `fix/<description>` -- bug fixes
- `hotfix/<description>` -- urgent production fixes

## Commit Hygiene

### Conventional Commits

All commits follow the format:

```
type(scope): description

[optional body]

[optional footer]
```

Types:
- `feat` -- new feature or role
- `fix` -- bug fix
- `refactor` -- restructuring without behavior change
- `test` -- adding or modifying tests
- `docs` -- documentation only
- `chore` -- build process, CI, dependencies
- `style` -- formatting, lint fixes

Examples:
```
feat(nginx): add TLS configuration support
fix(molecule): correct idempotency in package task
test(nginx): add negative test for missing cert
docs(nginx): document TLS variables and examples
```

### Interactive Rebase (Clean History)

If the branch has messy WIP commits, clean them up:

```bash
git log --oneline main..HEAD  # review commits
git rebase -i main            # interactive rebase
```

Guidelines:
- Squash WIP and fixup commits
- Keep logical commits separate (don't squash unrelated changes)
- Every remaining commit should pass lint and tests on its own

## Changelog

If the project uses a CHANGELOG, update it:

```markdown
## [Unreleased]

### Added
- TLS configuration support for nginx role

### Fixed
- Idempotency issue in package installation task

### Changed
- Default nginx worker_processes from 1 to auto
```

## PR Preparation

### PR Title
Follow conventional commit format: `type(scope): description`

### PR Body Template

```markdown
## Summary
[1-3 sentences describing what this PR does]

## Changes
- [List of concrete changes]

## Testing
- [ ] `ansible-lint --profile production` passes
- [ ] `molecule test` passes
- [ ] Idempotency verified (second run = 0 changes)
- [ ] Functional verification passes

## Verification Report
[Paste from verification-before-completion output]

## Documentation
- [ ] README updated
- [ ] Variables documented
- [ ] Examples provided
```

## CalVer Tagging (if applicable)

For projects using CalVer (YY.M.D-PATCH):

```bash
# After merge to main
git tag $(date +%y.%-m.%-d)-0
git push origin --tags
```

## Final Checklist

- [ ] Branch rebased on latest main
- [ ] All commits follow conventional format
- [ ] No WIP or fixup commits
- [ ] Changelog updated
- [ ] PR created with full template
- [ ] CI pipeline passes
- [ ] Ready for review
