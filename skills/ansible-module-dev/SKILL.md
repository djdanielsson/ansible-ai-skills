---
name: ansible-module-dev
description: >-
  Custom Ansible module development in Python. Use when writing, reviewing, or
  optimizing custom Ansible modules, filter plugins, lookup plugins, or module
  utilities. Covers declarative design, argument_spec validation, idempotency
  with check mode and diff support, security posture, testing patterns, and
  Python code style. Referenced by the ansible-architect orchestrator.
---
# Custom Ansible Module Development

## Design Philosophy

* Implement operations declaratively using a `state` parameter (e.g., `started`, `stopped`, `present`, `absent`) to dictate the final system condition. Never create imperative CRUD wrappers.
* Scope modules to a single responsibility. Build dedicated `_info` or `_facts` modules for data retrieval instead of adding `get`/`list` options to mutation modules.
* Map non-standard API naming to Ansible conventions using option aliases (always accept `name` as primary identifier).
* Implement a `provider` utility dictionary for authentication parameters.

## Required Module Structure

Python modules MUST follow this exact sequential order:

1. **Shebang & Encoding:** `#!/usr/bin/python` + UTF-8 encoding declaration
2. **Copyright & License:** Standard licensing (e.g., GPL-3.0+)
3. **DOCUMENTATION block:** YAML string defining identity, author, version, and all options
4. **EXAMPLES block:** Functional YAML snippets using FQCN
5. **RETURN block:** YAML dictionary of output variables, types, and return conditions
6. **Python Imports:** Standard library first, then `ansible.module_utils`. No wildcard imports.
7. **Execution Logic:** All code inside a `main()` function

## Argument Specification

Construct a comprehensive `argument_spec` using these validation constraints:

| Constraint | Purpose |
|-----------|---------|
| `type` | Expected format (`str`, `int`, `bool`, `list`, `dict`) |
| `required_together` | Atomic groupings (all or none) |
| `mutually_exclusive` | Halt if incompatible params provided together |
| `required_one_of` | At least one from a subset must be present |
| `required_if` | Dynamic dependencies based on parameter values |

## Idempotency and State Management

This is the most critical aspect. You MUST:

1. **Compare current vs. desired state BEFORE any action.** Query the system first.
2. Return `changed=False` if system already matches desired state.
3. Return `changed=True` only if a mutation was performed.
4. Support check mode: pass `supports_check_mode=True` to `AnsibleModule()`.
5. In check mode: perform reads, calculate `changed` boolean, call `module.exit_json()` BEFORE any mutations.
6. Provide `diff` dict with `before` and `after` for string/config manipulations.

```python
def main():
    module = AnsibleModule(argument_spec=spec, supports_check_mode=True)

    current_state = get_current_state(module)
    desired_state = build_desired_state(module.params)

    changed = current_state != desired_state

    if module.check_mode:
        module.exit_json(changed=changed, diff={"before": current_state, "after": desired_state})

    if changed:
        apply_state(module, desired_state)

    module.exit_json(changed=changed, state=desired_state)
```

## Security Posture

* Use `module.fail_json()` exclusively for error termination. NEVER `sys.exit()`.
* No catch-all exception blocks. Extract specific errors into `fail_json()`.
* Never read/write files in-place. Use `atomic_move` to prevent corruption.
* Strip heavy payloads before `exit_json()`.
* NEVER use `print()` (corrupts JSON output).
* Use `module.run_command()` for external binaries. Never `subprocess` directly.
* Use `sanitize_keys()` to scrub secrets from output.
* Use `fetch_url`/`open_url` for network requests (ensures TLS compliance).

## Testing

* **Unit tests:** pytest with mock objects for system functions.
* **Integration tests:** Verify state with a SEPARATE module (don't trust the module's own return value).
* **Six-step idempotency proof:**
  1. Apply (changed=True)
  2. Apply again (changed=False)
  3. Verify state independently
  4. Check mode before change (changed=True, no mutation)
  5. Check mode after convergence (changed=False)
  6. Remove/revert and verify

* **Network modules:** Execute on control node with persistent sessions. Return `before`, `after`, and `commands` keys.

## Python Code Style

Follow the `ansible-code-style` skill for Python formatting:

* `black --line-length 100` for formatting
* `isort` for import ordering
* `flake8` and `pylint` for linting
* `bandit` for security analysis
* Type hints on all functions
* Docstrings on all public functions

### Import Order

```python
from __future__ import absolute_import, division, print_function
__metaclass__ = type

import os
import sys
from typing import Dict, List, Optional

from ansible.module_utils.basic import AnsibleModule

from ansible_collections.myorg.collection.plugins.module_utils import helper
```

## CI Integration

Modules must pass these CI checks before merge:

* `ansible-test sanity` -- validates module structure and documentation
* `pytest` with coverage -- unit tests for all code paths
* `molecule test` -- integration tests verify real behavior
* `ansible-lint --profile production` -- style and best practice checks
* `bandit` -- security scanning for Python code

## Documentation Standards

### DOCUMENTATION Block

Every module MUST have a complete DOCUMENTATION string:

```python
DOCUMENTATION = r"""
---
module: namespace.collection.module_name
short_description: One line describing what this module does
version_added: "1.0.0"
description:
  - Full description of module purpose and behavior.
  - Additional context about when to use this module.
author:
  - Your Name (@github_handle)
options:
  name:
    description:
      - The name of the resource to manage.
    type: str
    required: true
  state:
    description:
      - Desired state of the resource.
    type: str
    choices: ['present', 'absent']
    default: present
requirements:
  - python >= 3.9
  - any_required_library
seealso:
  - module: related.module.name
notes:
  - Idempotency behavior notes.
  - Platform-specific caveats.
"""
```

### EXAMPLES Block

Provide at least 3 examples covering common usage, all options, and removal:

```python
EXAMPLES = r"""
- name: Create resource with minimum options
  namespace.collection.module_name:
    name: my_resource
    state: present

- name: Create resource with all options
  namespace.collection.module_name:
    name: my_resource
    state: present
    option_a: value
    option_b: value

- name: Remove resource
  namespace.collection.module_name:
    name: my_resource
    state: absent
"""
```

### RETURN Block

Document every key the module returns:

```python
RETURN = r"""
resource:
  description: The resource object after module execution.
  returned: success
  type: dict
  sample:
    name: my_resource
    status: active
changed_fields:
  description: List of fields that were modified.
  returned: changed
  type: list
  sample: ["option_a", "option_b"]
"""
```
