# lab-automation

![Lint](https://github.com/Rob2riches/lab-automation/actions/workflows/lint.yml/badge.svg)

Ansible automation for a small multi-host lab. Built to practise configuration
management, patching and operational safety patterns against real hosts rather
than slides.

## Environment

| Role | Host | OS |
|---|---|---|
| Control node | Ubuntu 26.04 VM (VMware) | ansible-core |
| Managed node | linux01 (LXD container) | Ubuntu 24.04 |
| Managed node | linux02 (LXD container) | Ubuntu 24.04 |

SSH with ed25519 key authentication. The Python interpreter is pinned in
`ansible.cfg` rather than left to discovery.

## Layout

    ansible.cfg                   Inventory path, roles path, interpreter pin
    inventory/hosts.yml           Static inventory, grouped
    playbooks/                    Playbooks
    roles/linux_baseline/         Configuration baseline role
    .github/workflows/lint.yml    CI pipeline

## Usage

    ansible-inventory --graph
    ansible linux -m ping
    ansible-playbook playbooks/site.yml --check --diff
    ansible-playbook playbooks/site.yml

## Contents

**roles/linux_baseline** - configuration baseline. Admin group, required
packages, and an SSH login banner rendered per host from a Jinja2 template.
The `sshd_config` edit runs `validate:` first, so a change that would break
sshd is never written. Variables live in `defaults/` and are prefixed with the
role name.

**playbooks/baseline-linux.yml** - applies the role.

**playbooks/patch-linux.yml** - rolling patch run. Updates the apt cache,
upgrades packages, removes orphans, reports whether a reboot is pending.

**playbooks/site.yml** - single entry point, imports both in order.

## Design notes

**Idempotent by default.** A second run with no drift reports `changed=0`.
That is what makes it safe to schedule.

**No volatile data in config files.** An earlier banner template embedded the
render date, so the first run on any new day reported a change and restarted
sshd for no operational reason. Removed.

**Rolling execution.** `patch-linux.yml` uses `serial: 1` so at most one host
is affected at a time. On a larger fleet this becomes a percentage.

**Dangerous actions are opt-in.** The reboot needs both a real
`/var/run/reboot-required` flag and an explicit `allow_reboot=true`.

**Validate before writing.** Config edits that could lock out access are
syntax-checked before the file is saved.

**Check mode first.** `--check --diff` shows what would change before anything
does.

**Role-prefixed variables.** Ansible has no namespacing, so two roles defining
`admin_group` would collide silently. Prefixing is the namespace.

**CI gates the merge.** `main` is protected and requires the lint workflow to
pass. The bypass list is empty, so the rule applies to the repository owner too.

**CI discovers what to check.** The syntax-check step iterates over
`playbooks/*.yml` rather than carrying a hardcoded list — an earlier version
broke twice when the repo changed shape.
