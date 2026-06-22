---
name: ansible-molecule
description: >-
  Ansible Molecule testing best practices. Use when writing or reviewing
  Molecule test scenarios. Covers functional verification over state checking,
  ansible-native testing architecture, inventory optimization, negative
  testing, multi-test sequences, shared state for collections, and CI
  integration patterns. Referenced by the ansible-architect orchestrator.
---
# Ansible Molecule Testing

## Core Rule: Functional Verification Over State Checking

The highest priority is **functional verification** — proving that business logic or security policy is actually operational, not merely that a file exists or a service is "started."

### When state checking is acceptable
- Testing custom Ansible modules (verifying a file was created or package installed is fine)

### When functional testing is REQUIRED
- Roles, playbooks, and collections: assert the configuration WORKS in practice

### Examples

| Configuration | State Checking (WRONG) | Functional Testing (CORRECT) |
|--------------|----------------------|---------------------------|
| Web servers | Check config file exists with correct permissions | `ansible.builtin.wait_for` on TCP socket + HTTP GET request |
| Security/Identity | Verify `/etc/pam.d/su` file exists | Execute commands to establish a real session and verify OS response |
| API endpoints | Check systemd service is "started" | `ansible.builtin.uri` to validate response payload structure |
| Firewall rules | Check iptables rule file exists | Attempt blocked connection and verify rejection |

## Ansible-Native Architecture

Use standard Ansible playbooks, inventory, and collections for the entire test lifecycle. Do NOT use:
- External Python frameworks (Testinfra)
- Proprietary DSLs
- Non-Ansible verification tools

Requirements:
- Standard declarative YAML syntax with familiar modules
- Output format: YAML for readability
- `ANSIBLE_HOST_KEY_CHECKING: false` to prevent CI hangs
- `pipelining: true` for execution speed

## Inventory and Execution

* Point arguments to `${MOLECULE_SCENARIO_DIRECTORY}/inventory/` absolute paths.
* Enforce alphabetical file loading for dynamic groups.
* Static hosts in `01-inventory.yml` (loaded first).
* Constructed inventory plugins in `02-constructed.yml` (processes static hosts, evaluates conditionals).
* **Collection testing:** Use `shared_state: true` in base config so multiple scenarios share a single provisioned inventory (avoids massive CI overhead).

## Negative Testing

Validate that systems correctly reject invalid inputs or terminate unstable processes:

* Use `failed_when` to override standard return logic.
* Register output with `register`.
* Assert strict pass conditions in `ansible.builtin.assert` with clear `fail_msg`.
* Use `block`/`rescue` for intentionally destructive tests — catch expected failures in `rescue` to validate security mechanisms.

```yaml
- name: Verify unauthorized access is denied
  block:
    - name: Attempt privileged action as unprivileged user
      ansible.builtin.command:
        cmd: systemctl restart critical-service
      become: false
      register: unauth_result
      failed_when: unauth_result.rc == 0

  rescue:
    - name: Confirm denial was enforced
      ansible.builtin.assert:
        that:
          - unauth_result.rc != 0
        fail_msg: "Security violation: unprivileged user could restart critical service"
```

## Documentation Standards

### Scenario README

Each molecule scenario should have documentation (in `molecule/<scenario>/README.md` or as a header comment in `molecule.yml`):

```markdown
# Scenario: default

## What this tests
- Service installs and starts correctly
- Configuration is applied and functional
- Idempotency (second converge produces no changes)

## Platform requirements
- Docker driver
- Requires network access for package installation

## Running
molecule test
molecule converge   # Apply without destroy
molecule verify     # Run verification only
```

### Verify Playbook Comments

In `verify.yml`, comment each assertion block explaining WHAT business logic is being proven, not what the Ansible task does:

```yaml
# Prove the web server accepts connections on the configured port
- name: Verify HTTP response
  ansible.builtin.uri:
    url: "http://{{ ansible_host }}:{{ nginx_port }}"
    status_code: 200

# Prove TLS certificate is valid and matches expected domain
- name: Verify TLS handshake
  ansible.builtin.command:
    cmd: "openssl s_client -connect {{ ansible_host }}:443 -servername {{ domain }}"
  register: tls_result
  changed_when: false
  failed_when: "'Verify return code: 0' not in tls_result.stdout"
```

## Multi-Test Sequences

For stateful software validation, intersperse verify and side_effect actions:

```yaml
scenario:
  test_sequence:
    - converge
    - idempotence               # Proves convergent state before disruption
    - verify                    # Validates initial deployment
    - side_effect               # Simulates reboot or process crash
    - converge                  # Re-converge to restore state
    - verify                    # Proves auto-recovery
```

This catches bugs that single-verify pipelines miss (e.g., services not surviving reboot). Note: `idempotence` runs before `side_effect` to test convergence on a clean state, and `converge` re-runs after `side_effect` to restore state before the recovery `verify`.

## Collection Testing with Shared State

For collections with multiple roles, use `shared_state: true` to avoid provisioning separate infrastructure per scenario:

```yaml
# extensions/molecule/config.yml
shared_state: true
```

The `default` scenario owns `create` and `destroy`. Component scenarios inherit provisioned infrastructure and run only `prepare`, `converge`, and `verify`.

## CI Integration

In CI pipelines, run all Molecule scenarios in parallel:

```yaml
# GitHub Actions matrix strategy
strategy:
  matrix:
    scenario: [default, centos, ubuntu]
  fail-fast: false
```

Always use `molecule test` (full lifecycle including idempotence) in CI, not just `molecule converge`.

## Common Verification Patterns

### Package installed
```yaml
- name: Verify package
  ansible.builtin.package_facts:
    manager: auto
- name: Assert package present
  ansible.builtin.assert:
    that: "'nginx' in ansible_facts.packages"
```

### Service running
```yaml
- name: Get service facts
  ansible.builtin.service_facts:
- name: Assert service active
  ansible.builtin.assert:
    that: "ansible_facts.services['nginx.service'].state == 'running'"
```

### Port listening
```yaml
- name: Check port is listening
  ansible.builtin.wait_for:
    port: 80
    timeout: 5
```

### File content
```yaml
- name: Read config file
  ansible.builtin.slurp:
    src: /etc/nginx/nginx.conf
  register: config_content
- name: Assert config contains expected setting
  ansible.builtin.assert:
    that: "'worker_processes auto;' in config_content.content | b64decode"
```
