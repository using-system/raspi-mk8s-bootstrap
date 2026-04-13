# raspi-mk8s-bootstrap

Ansible project to bootstrap a MicroK8s cluster on Raspberry Pi nodes.

## Commands

```bash
# Lint
yamllint -s .
ansible-lint playbooks/bootstrap.yml
ansible-playbook playbooks/bootstrap.yml --syntax-check

# Run playbook
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml

# Run single role
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml --tags base
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml --tags microk8s

# Test
molecule test
```

## Architecture

- `playbooks/bootstrap.yml` - Main entrypoint, orchestrates roles
- `roles/base/` - OS config: packages, hostname, timezone, cgroups, firewall (UFW)
- `roles/microk8s/` - MicroK8s install via snap, addons, cluster join
- `inventory/hosts.yml` - Cluster inventory with master/worker groups
- `molecule/` - Integration tests with Docker (Debian Bookworm)

## Conventions

- All roles use `molecule_test | default(false)` guard for hardware-specific tasks
- Roles are idempotent and tagged for selective execution
- Inventory supports multiple worker nodes under `microk8s_workers` group
- Vault for secrets: `ansible-vault create inventory/group_vars/all/vault.yml`

## Git workflow

### Semantic versioning

Use [Semantic Versioning](https://semver.org/) for tags and releases: `vMAJOR.MINOR.PATCH`.

Branch naming convention:
- `feature/<description>` - new functionality
- `fix/<description>` - bug fixes
- `chore/<description>` - maintenance, CI, docs

Commit messages follow [Conventional Commits](https://www.conventionalcommits.org/):
- `feat: add storage addon support`
- `fix: correct UFW port for dqlite`
- `chore: update CI Python version`
- `docs: update README with HA section`

### Branch safety

Before starting any work, verify you are NOT on `main`. Always work on a `feature/` or `fix/` branch. If on `main`, create or switch to the appropriate branch first.

### Pull request workflow

After each pushed commit:
1. If no PR exists for the current branch, create one (title and body in English)
2. If a PR already exists, update its title and body to reflect the current state of the work
3. PR title should be concise and describe the overall goal
4. PR body should summarize completed work and remaining tasks
5. After push, monitor the CI run and report the result
