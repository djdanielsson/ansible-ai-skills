---
name: ansible-code-style
description: >-
  Use when writing or reviewing Ansible automation code for style compliance.
  Covers YAML, Ansible task, Python, Shell, and Jinja2 formatting standards
  based on Zen of Ansible and Red Hat CoP practices.
---

# Ansible Code Style Guide

Style standards for all code in Ansible automation projects.

## Principles (Zen of Ansible)

1. **Consistency** -- follow established patterns
2. **Readability** -- code is read more than written
3. **Simplicity** -- prefer simple over clever
4. **Declarative over imperative** -- most of the time
5. **Clear over cluttered** -- readability counts

## Enforcement

- `ansible-lint --profile production` (Ansible)
- `yamllint` (YAML)
- `black --line-length 100` (Python)
- `isort` (Python imports)
- `flake8` (Python style)
- `shellcheck` (Shell scripts)

## YAML

### Indentation: 2 spaces, always

```yaml
# Correct
---
- name: Example task
  ansible.builtin.debug:
    msg: "Hello"

# Wrong -- 4 spaces
---
-   name: Example task
    ansible.builtin.debug:
        msg: "Hello"
```

### Always start with `---`

### Booleans: `true`/`false`, never `yes`/`no`

```yaml
enabled: true       # correct
ssl_verify: false   # correct
enabled: yes        # wrong
```

### Strings: quote when needed

```yaml
name: "Server with spaces"
version: "1.0.0"
state: present       # no quotes needed for simple values
```

### Line length: 160 max (warning)

Break long lines by using list format for module parameters:

```yaml
- name: Install packages
  ansible.builtin.package:
    name:
      - httpd
      - mod_ssl
      - httpd-tools
    state: present
```

### Comments: above the block, never inline

```yaml
# Configure web server
- name: Install Apache
  ansible.builtin.package:
    name: httpd
    state: present
```

## Ansible Tasks

### FQCN always

```yaml
# Correct
ansible.builtin.copy:
ansible.builtin.template:
ansible.builtin.systemd:

# Wrong
copy:
template:
systemd:
```

### Descriptive task names (sentence case, describes desired state)

```yaml
# Good -- sentence case, descriptive
- name: Install Apache web server
- name: Enable and start nginx service
- name: Deploy application configuration

# Bad -- vague, non-descriptive
- name: Install package
- name: Run command
- name: Step 3
```

### Module parameters: one per line

```yaml
# Correct
- name: Create user account
  ansible.builtin.user:
    name: webapp
    group: webapp
    home: /opt/webapp
    shell: /bin/bash
    state: present

# Wrong -- key=value format
- name: Create user
  ansible.builtin.user: name=webapp group=webapp home=/opt/webapp
```

### Conditionals: no `{{ }}` in `when`

```yaml
# Correct
when: ansible_os_family == "RedHat"

# Correct -- multiple conditions as list
when:
  - ansible_os_family == "RedHat"
  - ansible_distribution_major_version | int >= 8

# Correct -- boolean with filter
when: webserver_ssl_enabled | bool

# Wrong
when: "{{ ansible_os_family == 'RedHat' }}"
```

### Loops: use `loop_control` with `loop_var` and `label`

```yaml
- name: Install packages
  ansible.builtin.package:
    name: "{{ package_name }}"
    state: present
  loop:
    - httpd
    - mod_ssl
  loop_control:
    loop_var: package_name
    label: "{{ package_name }}"
```

### Variables: prefix with role name

```yaml
# Correct
webserver_port: 80
webserver_ssl_enabled: true
database_name: webapp

# Wrong -- generic, collision-prone
port: 80
enabled: true
name: webapp
```

### Tags: hierarchical and consistent

```yaml
- name: Install Apache
  ansible.builtin.package:
    name: httpd
  tags:
    - install
    - webserver
    - webserver:install
```

### Handlers: use `listen` for grouping

```yaml
handlers:
  - name: restart apache
    ansible.builtin.service:
      name: httpd
      state: restarted
    listen: restart webserver

  - name: reload apache
    ansible.builtin.service:
      name: httpd
      state: reloaded
    listen: reload webserver
```

## Python (Modules and Plugins)

### Formatting

- `black --line-length 100`
- `isort` for import ordering
- Type hints on all functions
- Docstrings on all public functions

### Import Order

```python
from __future__ import absolute_import, division, print_function
__metaclass__ = type

# Standard library
import os
import sys
from typing import Dict, List, Optional

# Third party
from ansible.module_utils.basic import AnsibleModule

# Local
from ansible_collections.myorg.collection.plugins.module_utils import helper
```

### Module Structure

```python
DOCUMENTATION = r'''...'''
EXAMPLES = r'''...'''
RETURN = r'''...'''

from ansible.module_utils.basic import AnsibleModule

def run_module():
    module_args = dict(
        name=dict(type='str', required=True),
    )
    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True,
    )
    result = dict(changed=False)
    module.exit_json(**result)

def main():
    run_module()

if __name__ == '__main__':
    main()
```

## Shell Scripts

### Header

```bash
#!/bin/bash
set -euo pipefail
```

### Variables

- `UPPERCASE` for constants
- `lowercase` for local variables
- Always quote: `"${variable}"`

### Functions

```bash
function install_packages() {
    local package_list=("$@")
    for package in "${package_list[@]}"; do
        echo "Installing $package..."
    done
}
```

## Jinja2 Templates

### Spacing inside braces

```jinja
{{ variable }}              {# correct #}
{{ variable | default('') }} {# correct #}
{{variable}}               {# wrong #}
```

### Whitespace control

```jinja
{% for item in items -%}
    {{ item }}
{% endfor -%}
```

### Use filters

```jinja
{{ variable | default('default_value') }}
{{ list_var | join(', ') }}
{{ dict_var | to_nice_yaml }}
```

## yamllint Configuration

```yaml
# .yamllint
extends: default
rules:
  line-length:
    max: 160
    level: warning
  indentation:
    spaces: 2
    indent-sequences: true
  document-start:
    present: true
  truthy:
    allowed-values: ['true', 'false']
```
