# Specification - raspi-mk8s-bootstrap

## Objective

Bootstrap a MicroK8s Kubernetes cluster across one or more Raspberry Pi nodes via Ansible, starting from a fresh Raspberry Pi OS with SSH access.

## Target environment

| Component | Version / Spec |
|-----------|---------------|
| Hardware | Raspberry Pi 4 or 5 |
| OS | Raspberry Pi OS (Debian Bookworm, 64-bit) |
| Kubernetes | MicroK8s 1.30/stable |
| Architecture | arm64 (aarch64) |

## Cluster topology

```
┌─────────────────────┐
│   Control machine    │
│   (laptop / CI)      │
│   runs Ansible       │
└──────────┬──────────┘
           │ SSH
     ┌─────┴──────┐
     ▼            ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│  Master  │  │ Worker 1│  │ Worker N│
│ (1 node) │  │         │  │         │
│ MicroK8s │◄─│ MicroK8s│  │ MicroK8s│
│ API srv  │  │  joined │  │  joined │
└─────────┘  └─────────┘  └─────────┘
```

- **1 master** node runs the control plane (API server, etcd via dqlite, scheduler, controller-manager)
- **N worker** nodes join the cluster automatically via token generated on the master
- All nodes are in the same local network

## Roles specification

### Role: base

**Purpose:** Prepare the OS for running MicroK8s.

| Task | Details |
|------|---------|
| System upgrade | `apt update` + `dist-upgrade` |
| Packages | git, curl, wget, vim, htop, tmux, jq, unzip, ca-certificates, gnupg, apt-transport-https, python3-pip, ufw, snapd, linux-modules-extra-raspi |
| Hostname | Set per node from inventory hostname |
| Timezone | Configurable, default `Europe/Paris` |
| Locale | Configurable, default `en_US.UTF-8` |
| cgroups | Enable `cgroup_memory=1` in `/boot/firmware/cmdline.txt` (required for K8s) |
| Firewall (UFW) | Deny incoming, allow outgoing, allow SSH (22), allow MicroK8s ports: 16443, 10250, 10255, 25000, 12379, 10257, 10259, 19001 |

### Role: microk8s

**Purpose:** Install MicroK8s and form the cluster.

| Task | Details |
|------|---------|
| Install | `snap install microk8s --classic --channel=1.30/stable` |
| User group | Add `ansible_user` to `microk8s` group |
| Wait ready | `microk8s status --wait-ready` |
| Addons | Enable `dns`, `rbac` |
| Cluster join | Master generates token (`microk8s add-node`), workers join with token |
| Aliases | `/etc/profile.d/microk8s.sh` with `kubectl` and `helm` aliases |
| Kubeconfig | Export to `~/.kube/config` on master |

## Inventory structure

```yaml
all:
  children:
    microk8s_cluster:
      children:
        microk8s_master:
          hosts:
            raspi-master: { ansible_host: <IP> }
        microk8s_workers:
          hosts:
            raspi-worker-1: { ansible_host: <IP> }
            raspi-worker-N: { ansible_host: <IP> }
```

## Testing strategy

| Layer | Tool | Scope |
|-------|------|-------|
| YAML syntax | yamllint | All YAML files |
| Ansible best practices | ansible-lint | Playbooks and roles |
| Playbook syntax | `--syntax-check` | Playbook structure |
| Integration | Molecule + Docker | Full role execution on Debian Bookworm container |
| Idempotence | Molecule | Second run produces zero changes |
| Verification | Molecule verify | Assert packages installed, alias script present |

Hardware-specific tasks (snap install, UFW, hostname, cgroups) are skipped in tests via `molecule_test` flag.

## CI/CD pipeline

```
push / PR to main
       │
       ▼
   ┌──────┐
   │ Lint  │  yamllint + ansible-lint + syntax-check
   └──┬───┘
      │
      ▼
┌─────────────┐
│ Integration  │  molecule test (Docker)
└─────────────┘
```

## Security considerations

- SSH key-based authentication only (no passwords)
- UFW firewall enabled with minimal open ports
- Secrets managed via Ansible Vault (`inventory/group_vars/all/vault.yml`)
- `.gitignore` excludes vault passwords and local inventory overrides

## Future enhancements

- Storage addon (hostpath-storage or NFS)
- MetalLB for LoadBalancer services
- Ingress controller (nginx or traefik)
- Monitoring stack (Prometheus + Grafana)
- Automated OS image provisioning (cloud-init / Packer)
- Multi-master HA topology
