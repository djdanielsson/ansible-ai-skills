---
name: ansible-brainstorming
description: >-
  Use before any creative Ansible work - creating roles, playbooks, collections,
  CaC configurations, or modifying automation behavior. Explores infrastructure
  context, requirements, and design before implementation.
---

# Ansible Brainstorming

Help turn automation ideas into fully formed designs through collaborative dialogue focused on infrastructure context.

<HARD-GATE>
Do NOT write any Ansible code, scaffold any project, or take any implementation action until you have presented a design and the user has approved it. This applies to EVERY project regardless of perceived simplicity.
</HARD-GATE>

## Anti-Pattern: "This Is Too Simple To Need A Design"

Every project goes through this process. A single role, a one-task playbook, a variable change -- all of them. "Simple" automation is where unexamined assumptions about state, idempotency, and dependencies cause the most wasted work.

## Checklist

Complete in order:

1. **Explore project context** -- check files, existing roles, collections, inventory structure, recent commits
2. **Detect infrastructure context** -- what targets? what OS families? what existing automation touches this?
3. **Ask clarifying questions** -- one at a time: purpose, target state, constraints, success criteria
4. **State Reasoning walkthrough** -- for the core change: what is current state, desired state, delta?
5. **Propose 2-3 approaches** -- with idempotency trade-offs, module choices, and your recommendation
6. **Present design** -- in Ansible terms: roles, plays, variables, handlers, templates, tests
7. **Write design doc** -- save and commit
8. **User reviews spec** -- ask user to review before proceeding
9. **Transition** -- invoke `ansible-planning` skill to create implementation plan

## Infrastructure-Aware Questions

Unlike general brainstorming, you must explore:

- **Target hosts:** What OS families? What package managers? Cloud vs on-prem?
- **Existing automation:** What roles/playbooks already manage these hosts? What could conflict?
- **Inventory structure:** Static or dynamic? What groups exist? What variables are set?
- **AAP integration:** Will this run in AAP/AWX? Does it need an EE? What credentials?
- **Idempotency requirements:** What happens on re-run? Can we prove zero changes on converged state?
- **Testing approach:** Molecule? What driver (docker, podman, vagrant)? What functional verification?
- **Security:** Any secrets needed? Vault or external? What `no_log` tasks?

## Design Presentation Format

Present designs in Ansible terms:

```markdown
## Role: <role_name>

### Purpose
One sentence describing what converged state looks like.

### Variables (defaults/main.yml)
| Variable | Default | Description |
|----------|---------|-------------|
| `role_var` | `value` | What it controls |

### Tasks (high-level)
1. Install packages (ansible.builtin.apt/dnf)
2. Deploy config template (ansible.builtin.template)
3. Enable service (ansible.builtin.systemd)

### Handlers
- Restart <service> on config change

### Molecule Verification
- Functional test: HTTP GET to port X returns 200
- Idempotency: second converge produces zero changes

### Dependencies
- Requires: <other_role> for base config
- Notifies: <handler> in parent playbook
```

## Key Principles

- **One question at a time** -- don't overwhelm
- **State-first thinking** -- always reason about current vs desired state
- **YAGNI ruthlessly** -- remove unnecessary features
- **Explore alternatives** -- always propose 2-3 approaches
- **Idempotency by design** -- not as an afterthought
- **Test plan included** -- every design includes how to verify it works

## After the Design

- Save the design doc and commit to git
- Invoke the `ansible-planning` skill to create the implementation plan
- Do NOT invoke any other skill. `ansible-planning` is the next step.
