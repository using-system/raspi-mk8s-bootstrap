# raspi-mk8s-bootstrap

Ansible playbooks to provision and bootstrap a lightweight MicroK8s Kubernetes cluster on Raspberry Pi.

## Prerequisites

- One or more Raspberry Pi (4/5) running Raspberry Pi OS (Debian Bookworm)
- SSH access configured on each node (key-based authentication)
- Ansible >= 2.15 installed on the control machine
- Python 3.10+

## Quick start

```bash
# Install Ansible collections
ansible-galaxy collection install -r requirements.yml

# Edit the inventory with your Pi IP addresses
vim inventory/hosts.yml

# Run the full bootstrap
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml
```

## Inventory

The inventory defines two groups:

| Group | Role | Description |
|-------|------|-------------|
| `microk8s_master` | Control plane | Single master node running the MicroK8s API server |
| `microk8s_workers` | Worker nodes | One or more workers that join the cluster |

Edit `inventory/hosts.yml` to match your network:

```yaml
microk8s_master:
  hosts:
    raspi-master:
      ansible_host: 192.168.1.100
microk8s_workers:
  hosts:
    raspi-worker-1:
      ansible_host: 192.168.1.101
```

## Roles

### base

Prepares the Raspberry Pi OS:

- System upgrade and essential packages (git, curl, jq, snapd, etc.)
- Hostname, timezone, and locale configuration
- cgroups memory enabled (required for Kubernetes on Pi)
- UFW firewall: SSH + MicroK8s ports (16443, 25000, 10250, etc.)

### microk8s

Installs and configures the MicroK8s cluster:

- MicroK8s installed via snap (channel `1.30/stable` by default)
- User added to `microk8s` group
- Addons enabled: `dns`, `rbac`
- Automatic cluster join: token generated on master, workers join automatically
- kubectl/helm aliases and kubeconfig export

## Running specific roles

```bash
# Base setup only
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml --tags base

# MicroK8s only
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml --tags microk8s
```

## Configuration

Key variables (override in `inventory/hosts.yml` or `group_vars`):

| Variable | Default | Description |
|----------|---------|-------------|
| `base_hostname` | `{{ inventory_hostname }}` | Node hostname |
| `base_timezone` | `Europe/Paris` | System timezone |
| `base_locale` | `en_US.UTF-8` | System locale |
| `microk8s_channel` | `1.30/stable` | MicroK8s snap channel |
| `microk8s_addons` | `[dns, rbac]` | Addons to enable |

## Testing

Integration tests run with Molecule and Docker:

```bash
pip install ansible molecule molecule-plugins[docker] docker
molecule test
```

## CI/CD

GitHub Actions pipeline on push/PR to `main`:

1. **Lint** - yamllint, ansible-lint, syntax check
2. **Integration** - Molecule test with Debian Bookworm container

## License

See [LICENSE](LICENSE).
