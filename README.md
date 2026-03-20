# ansible-node-hardening

Ansible playbook for automated Ubuntu server provisioning and CIS-inspired security hardening. Designed to pair with [terraform-infra-demo](https://github.com/amiine-snoussi/terraform-infra-demo) — Terraform provisions the instance, Ansible configures it.

## What It Does

One command takes a fresh Ubuntu 22.04 server from zero to production-ready:

```
                      ┌────────────────────────┐
                      │  Fresh Ubuntu 22.04    │
                      └───────────┬────────────┘
                                  │
                      ansible-playbook site.yml
                                  │
         ┌────────────────────────┼────────────────────────┐
         ▼                        ▼                        ▼
   ┌───────────┐          ┌─────────────┐          ┌─────────────┐
   │  common   │          │ssh-hardening│          │  firewall   │
   ├───────────┤          ├─────────────┤          ├─────────────┤
   │ Packages  │          │ Port config │          │ UFW setup   │
   │ Users     │          │ Key-only    │          │ Deny all in │
   │ Timezone  │          │ Root denied │          │ Allow list  │
   │ sysctl    │          │ Modern algo │          │ Rate limit  │
   │ fail2ban  │          │ Session mgmt│          │ Logging     │
   │ Auto-patch│          │ Banner      │          │             │
   └───────────┘          └─────────────┘          └─────────────┘
         │                                                │
         ▼                                                ▼
   ┌───────────┐                                   ┌─────────────┐
   │  docker   │                                   │ monitoring  │
   ├───────────┤                                   ├─────────────┤
   │ Docker CE │                                   │ Node Exporter│
   │ Compose   │                                   │ systemd svc │
   │ Log mgmt  │                                   │ Port 9100   │
   │ Users     │                                   │ Hardened    │
   └───────────┘                                   └─────────────┘
                                  │
                      ┌───────────▼────────────┐
                      │  Production-Ready      │
                      │  Hardened Server       │
                      └────────────────────────┘
```

## Security Hardening Checklist

| Control | Implementation |
|---------|---------------|
| SSH root login | Disabled |
| SSH password auth | Disabled (key-only) |
| SSH crypto policy | Modern ciphers, KEX, MACs only |
| SSH session limits | MaxAuthTries 3, idle timeout, MaxSessions 3 |
| Firewall | UFW deny-all-in, explicit allow list |
| Brute force protection | fail2ban enabled |
| Kernel hardening | 12 sysctl parameters (ICMP, redirects, SYN cookies, ASLR) |
| Filesystem hardening | Disabled cramfs, hfs, squashfs, udf modules |
| Shared memory | noexec,nosuid,nodev on /run/shm |
| Auto-patching | unattended-upgrades for security updates |
| Cron directories | 700 permissions (root only) |
| Docker logging | JSON file driver, 10MB max, 3 file rotation |
| Node Exporter | Runs as unprivileged user with systemd sandboxing |
| Login banner | Legal warning on SSH connect |

## Roles

| Role | Purpose | Tags |
|------|---------|------|
| `common` | System packages, deploy user, kernel hardening, fail2ban, auto-updates | `common`, `base` |
| `ssh-hardening` | sshd_config lockdown, key-only auth, modern crypto, login banner | `ssh`, `security` |
| `firewall` | UFW with deny-all default, explicit port allow list, logging | `firewall`, `security` |
| `docker` | Docker CE + Compose plugin from official repo, daemon config, log rotation | `docker` |
| `monitoring` | Prometheus Node Exporter as hardened systemd service | `monitoring` |

## Prerequisites

- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/) >= 2.15
- SSH access to target server(s)
- Python 3 on target hosts

## Quick Start

```bash
# Clone
git clone https://github.com/amiine-snoussi/ansible-node-hardening.git
cd ansible-node-hardening

# Install required collections
ansible-galaxy collection install -r requirements.yml

# Configure your target hosts
cp inventory/hosts.ini inventory/hosts.ini.bak
vim inventory/hosts.ini   # Add your server IPs

# Dry-run (check mode)
ansible-playbook site.yml --check --diff

# Deploy
ansible-playbook site.yml

# Run specific roles only
ansible-playbook site.yml --tags "ssh,firewall"
ansible-playbook site.yml --tags "docker"
```

## Integration with Other Projects

This repo is designed as part of a complete DevOps pipeline:

```
terraform-infra-demo          ansible-node-hardening         fleetlane-lite
─────────────────────  ──►  ─────────────────────────  ──►  ───────────────
Provision EC2 instance       Harden & configure server       Deploy application
(IaC)                        (Config management)             (Docker Compose)
```

## Project Structure

```
.
├── site.yml                          # Main playbook
├── ansible.cfg                       # Ansible configuration
├── requirements.yml                  # Galaxy dependencies
├── inventory/
│   └── hosts.ini                     # Target host inventory
├── group_vars/
│   └── all.yml                       # Global variables
├── roles/
│   ├── common/
│   │   ├── tasks/main.yml            # Base provisioning & OS hardening
│   │   ├── handlers/main.yml
│   │   └── defaults/main.yml
│   ├── ssh-hardening/
│   │   ├── tasks/main.yml            # SSH lockdown
│   │   ├── handlers/main.yml
│   │   ├── templates/sshd_config.j2  # Hardened sshd template
│   │   └── defaults/main.yml
│   ├── firewall/
│   │   ├── tasks/main.yml            # UFW configuration
│   │   └── defaults/main.yml
│   ├── docker/
│   │   ├── tasks/main.yml            # Docker CE installation
│   │   ├── handlers/main.yml
│   │   └── defaults/main.yml
│   └── monitoring/
│       ├── tasks/main.yml            # Node Exporter setup
│       ├── handlers/main.yml
│       ├── templates/node_exporter.service.j2
│       └── defaults/main.yml
├── .ansible-lint
├── .gitignore
└── README.md
```

## Customization

Override any default in `group_vars/all.yml` or create host-specific files under `host_vars/`:

```yaml
# Example: tighten SSH access to specific IP
ssh_allowed_cidrs: ["YOUR_IP/32"]
ssh_port: 2222

# Example: skip Docker installation
install_docker: false

# Example: add Grafana/Prometheus ports to firewall
ufw_rules:
  - { rule: allow, port: 22, proto: tcp, comment: "SSH" }
  - { rule: allow, port: 80, proto: tcp, comment: "HTTP" }
  - { rule: allow, port: 3000, proto: tcp, comment: "Grafana" }
  - { rule: allow, port: 9090, proto: tcp, comment: "Prometheus" }
```

## Related Projects

| Project | Description |
|---------|-------------|
| [fleetlane-lite](https://github.com/amiine-snoussi/fleetlane-lite) | FastAPI app with Docker, nginx, GitHub Actions CI, Prometheus + Grafana |
| [terraform-infra-demo](https://github.com/amiine-snoussi/terraform-infra-demo) | Terraform IaC to provision EC2 on AWS free tier |
| [bash-sysadmin-toolkit](https://github.com/amiine-snoussi/bash-sysadmin-toolkit) | Production Bash scripts for Linux sysadmin tasks |

## Author

**Amine Snoussi** — CS graduate, Université du Québec à Montréal (UQAM)

- GitHub: [@amiine-snoussi](https://github.com/amiine-snoussi)
