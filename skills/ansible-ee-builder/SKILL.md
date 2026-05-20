---
name: ansible-ee-builder
description: >-
  Use when building, versioning, or managing Ansible Execution Environments.
  Covers ansible-builder V3 schema, version pinning, CalVer tagging,
  image security, SBOM generation, and AAP integration.
---

# Ansible Execution Environment Builder

Build immutable, version-locked Execution Environments synchronized with code releases.

## Core Principle

> Every code release tag has a corresponding EE image tag using YY.M.D-PATCH format.

```
Code Tag: 26.1.5-0  →  EE Image: quay.io/myorg/automation-ee:26.1.5-0
```

## V3 Schema (execution-environment.yml)

```yaml
---
version: 3

build_arg_defaults:
  ANSIBLE_GALAXY_CLI_COLLECTION_OPTS: '--pre'
  EE_BASE_IMAGE: 'quay.io/ansible/creator-ee:v0.20.1'

dependencies:
  galaxy: requirements.yml
  python: requirements.txt
  system: bindep.txt

additional_build_files:
  - src: files/ansible.cfg
    dest: configs

additional_build_steps:
  prepend_galaxy:
    - COPY _build/configs/ansible.cfg /etc/ansible/ansible.cfg
  append_final:
    - RUN pip install --no-cache-dir custom-package==1.0.0
```

## Dependency Pinning (Mandatory)

### Collections (requirements.yml)

```yaml
---
collections:
  - name: ansible.posix
    version: "1.5.4"
  - name: ansible.utils
    version: "3.1.0"
  - name: community.general
    version: "8.1.0"
  - name: infra.aap_configuration
    version: "2.5.0"
```

**Never** leave a collection unpinned. Unpinned collections produce non-reproducible builds.

### Python (requirements.txt)

```
jmespath==1.0.1
netaddr==0.9.0
python-dateutil==2.8.2
requests==2.31.0
```

### System (bindep.txt)

```
git [platform:centos-8 platform:rhel-8]
openssh-clients [platform:centos-8 platform:rhel-8]
```

## Build Process

### Local Development Build

```bash
cd automation-ee/
ansible-builder build \
  --tag "quay.io/myorg/automation-ee:dev-local" \
  --container-runtime podman \
  --verbosity 3
```

### Release Build (CI/CD)

```bash
VERSION="26.1.5-0"
COMMIT_SHA=$(git rev-parse HEAD)

ansible-builder build \
  --tag "quay.io/myorg/automation-ee:${VERSION}" \
  --tag "quay.io/myorg/automation-ee:sha-${COMMIT_SHA}" \
  --container-runtime podman

podman push "quay.io/myorg/automation-ee:${VERSION}"
podman push "quay.io/myorg/automation-ee:sha-${COMMIT_SHA}"
```

## Tag Immutability Rules

| Tag Type | Mutable? | Use In |
|----------|----------|--------|
| `YY.M.D-PATCH` (e.g., `26.1.5-0`) | IMMUTABLE | Dev, QA, Prod |
| `sha-<commit>` | IMMUTABLE | Traceability |
| `dev-latest` | MUTABLE | Dev only, never QA/Prod |
| `latest` | NEVER USE | Nowhere |

## Validation

```bash
# Validate EE definition
ansible-builder create --verbosity 3

# Check version pinning
grep -E "^  - name: [a-z]" requirements.yml | while read line; do
  if ! echo "$line" | grep -q "version:"; then
    echo "FAIL: Unpinned collection: $line"
  fi
done

# Check base image is official
grep "EE_BASE_IMAGE" execution-environment.yml
```

## Testing the Built Image

```bash
# Interactive inspection
podman run -it quay.io/myorg/automation-ee:26.1.5-0 /bin/bash

# Verify collections
podman run quay.io/myorg/automation-ee:26.1.5-0 \
  ansible-galaxy collection list

# Verify Python packages
podman run quay.io/myorg/automation-ee:26.1.5-0 \
  pip list

# Run a test playbook
podman run -v ./tests:/tests:Z quay.io/myorg/automation-ee:26.1.5-0 \
  ansible-playbook /tests/smoke-test.yml
```

## Security

### SBOM Generation

```bash
syft quay.io/myorg/automation-ee:26.1.5-0 -o json > sbom-26.1.5-0.json
```

### Vulnerability Scanning

```bash
# Scan with Trivy
trivy image --exit-code 1 quay.io/myorg/automation-ee:26.1.5-0

# Scan with Grype
grype quay.io/myorg/automation-ee:26.1.5-0 -o json > scan-26.1.5-0.json
```

### Production: Use Image Digests

```yaml
# Best -- immutable digest
image: "quay.io/myorg/automation-ee@sha256:abc123..."

# Good -- version tag
image: "quay.io/myorg/automation-ee:26.1.5-0"

# NEVER
image: "quay.io/myorg/automation-ee:latest"
```

## AAP Integration

### EE Registration in Controller

```yaml
controller_execution_environments:
  - name: "Automation EE - 26.1.5-0"
    description: "Execution Environment (26.1.5-0)"
    organization: "Platform"
    image: "quay.io/myorg/automation-ee:26.1.5-0"
    credential: "Quay.io Registry"
    pull: "missing"
```

### Job Template Linking

```yaml
controller_templates:
  - name: "Deploy Webserver - QA"
    execution_environment: "Automation EE - 26.1.5-0"
    scm_branch: "26.1.5-0"
```

## Release Manifest Integration

```yaml
components:
  execution_environment:
    name: "automation-ee"
    registry: "quay.io"
    image: "quay.io/myorg/automation-ee:26.1.5-0"
    digest: "sha256:1234567890abcdef..."
    built_at: "2026-01-04T10:25:00Z"
    base_image: "quay.io/ansible/creator-ee:v0.20.1"
    python_version: "3.11"
    ansible_core_version: "2.16.0"
    collections:
      - name: "ansible.posix"
        version: "1.5.4"
    python_packages:
      - name: "jmespath"
        version: "1.0.1"
```

## Anti-Patterns

| Anti-Pattern | Why | Do Instead |
|-------------|-----|------------|
| `image: ee:latest` in QA/Prod | Mutable, non-reproducible | Use version tag or digest |
| Unpinned collections | Build breaks on upstream update | Pin all versions |
| Mismatched code + EE tags | Runtime behavior mismatch | Synchronize tags |
| Overwriting version tags | Breaks audit trail | Create new PATCH version |
| Skipping security scans | Unknown vulnerabilities in prod | Scan before every promotion |
| No SBOM | Can't audit dependencies | Generate and store with release |

## Rollback

```yaml
# Revert job template to previous EE
controller_execution_environments:
  - name: "Automation EE - 26.1.4-0"
    image: "quay.io/myorg/automation-ee:26.1.4-0"

# Verify the previous image still exists
podman pull quay.io/myorg/automation-ee:26.1.4-0
```

Always verify the previous EE image exists in the registry before rolling back.
