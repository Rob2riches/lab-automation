# lab-automation
![Lint](https://github.com/Rob2riches/lab-automation/actions/workflows/lint.yml/badge.svg)
Ansible automation for a small multi-host lab environment. Built to practise
configuration management, patching and operational safety patterns against real
hosts rather than slides.

## Environment

| Role | Host | OS |
|---|---|---|
| Control node | Ubuntu 26.04 VM (VMware) | ansible-core |
| Managed node | linux01 (LXD container) | Ubuntu 24.04 |
| Managed node | linux02 (LXD container) | Ubuntu 24.04 |

Connection is SSH with ed25519 key authentication. The Python interpreter is
pinned in `ansible.cfg` rather than left to discovery.

## Layout

    ansible.cfg              Inventory path, interpreter pin, host key policy
    inventory/hosts.yml      Static inventory, grouped
    playbooks/               Playbooks
    playbooks/templates/     Jinja2 templates

## Usage

    ansible-inventory --graph                             # verify parsing
    ansible linux -m ping                                 # verify connectivity
    ansible-playbook playbooks/baseline-linux.yml --check --diff
    ansible-playbook playbooks/baseline-linux.yml

## Playbooks

**baseline-linux.yml** - configuration baseline. Ensures an admin group, a set
of required packages, and an SSH login banner rendered per host from a Jinja2
template. The banner change is enforced in `sshd_config` with `validate:`, so a
config that would break sshd is never written.

**patch-linux.yml** - rolling patch run. Updates the apt cache, upgrades all
packages, removes orphans, and reports whether a reboot is pending.

## Design notes

**Idempotent by default.** Every playbook can run repeatedly. A second run with
no drift reports `changed=0`. That is what makes it safe to schedule.

**Rolling execution.** `patch-linux.yml` uses `serial: 1` so at most one host is
affected at a time. On a larger fleet this becomes a percentage.

**Dangerous actions are opt-in.** The reboot task requires both a genuine
`/var/run/reboot-required` flag and an explicit `allow_reboot=true`. The default
run reports a pending reboot without taking anything down.

**Validate before writing.** Config edits that could lock out access run a
syntax check before the file is saved.

**Check mode first.** `--check --diff` shows what would change before anything
does.
