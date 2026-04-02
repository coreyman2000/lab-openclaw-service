# OpenClaw Lab Service

Multi-tenant OpenClaw deployment for labs and organizations using Proxmox, Docker, and shared GPU resources.

## 🎯 Overview

Provide isolated OpenClaw instances to multiple teams/tenants while sharing expensive GPU resources efficiently.

### Key Features

- 🏢 **Multi-Tenancy**: Each team gets isolated VMs with dedicated OpenClaw instances
- 🐳 **Docker-Based**: OpenClaw runs in containers for easy management and updates
- 🚀 **GPU Sharing**: Shared Ollama backends on LXC containers with GPU passthrough
- 🤖 **Ansible Automation**: Full provisioning and lifecycle management
- 📊 **Flexible Tiers**: Basic, Standard, Premium tiers with different resource allocations

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                      PROXMOX CLUSTER                        │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │  GPU Node 1  │    │  GPU Node 2  │    │ Compute Node │  │
│  │  L40 GPUs    │    │  L40 GPUs    │    │ (VM Host)    │  │
│  │  ┌────────┐  │    │  ┌────────┐  │    │  ┌────────┐  │  │
│  │  │ Ollama │  │    │  │ Ollama │  │    │  │Tenant A│  │  │
│  │  │  LXC   │  │    │  │  LXC   │  │    │  │  VMs   │  │  │
│  │  └────────┘  │    │  └────────┘  │    │  └────────┘  │  │
│  └──────────────┘    └──────────────┘    │  ┌────────┐  │  │
│       ▲                     ▲            │  │Tenant B│  │  │
│       └─────────────────────┘            │  │  VMs   │  │  │
│            Shared Model Pool             │  └────────┘  │  │
│                                          └──────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

## 🚀 Quick Start

### 1. Prerequisites

- Proxmox VE 8.x cluster
- Ansible 2.15+ with community modules
- GPU nodes with NVIDIA drivers installed
- SSH access to Proxmox hosts

### 2. Install Ansible Collections

```bash
ansible-galaxy collection install community.general community.docker
```

### 3. Configure Inventory

Edit `ansible/inventory/hosts.yml` with your Proxmox hosts and GPU configuration.

### 4. Deploy Ollama Backends

```bash
cd ansible
ansible-playbook -i inventory/ playbooks/deploy-ollama-lxc.yml
```

### 5. Create First Tenant

```bash
ansible-playbook -i inventory/ playbooks/provision-tenant.yml
# Follow interactive prompts
```

## 📁 Repository Structure

```
lab-openclaw-service/
├── ARCHITECTURE.md           # Detailed architecture documentation
├── docs/
│   └── SETUP.md              # Complete setup guide
├── ansible/
│   ├── inventory/
│   │   ├── hosts.yml         # Proxmox hosts and tenant VMs
│   │   └── group_vars/       # Configuration per group
│   ├── playbooks/
│   │   ├── deploy-ollama-lxc.yml    # Deploy GPU backends
│   │   ├── provision-tenant.yml     # Create tenant VMs
│   │   └── tenant-ops.yml           # Manage existing tenants
│   └── templates/
│       ├── docker-compose.yml.j2    # OpenClaw compose file
│       └── openclaw-config.yml.j2   # OpenClaw configuration
└── scripts/
    └── tenant-create.sh      # Quick tenant creation script
```

## 🔧 Tenant Operations

### Check Status

```bash
ansible-playbook -i inventory/ playbooks/tenant-ops.yml \
  -e "operation=status tenant=all"
```

### Restart Services

```bash
ansible-playbook -i inventory/ playbooks/tenant-ops.yml \
  -e "operation=restart tenant=team-alpha"
```

### Update OpenClaw

```bash
ansible-playbook -i inventory/ playbooks/tenant-ops.yml \
  -e "operation=update tenant=team-alpha"
```

### Delete Tenant

```bash
ansible-playbook -i inventory/ playbooks/tenant-ops.yml \
  -e "operation=remove tenant=team-alpha"
```

## 📊 Resource Tiers

| Tier | vCPU | RAM | Disk | OpenClaw Instances | Est. Users |
|------|------|-----|------|-------------------|------------|
| Basic | 2 | 4GB | 20GB | 2 | 5-10 |
| Standard | 4 | 8GB | 40GB | 5 | 15-25 |
| Premium | 8 | 16GB | 80GB | 10+ | 50+ |

## 🔒 Security

- **VM Isolation**: Each tenant in separate Proxmox VM (kernel-level isolation)
- **Network Segmentation**: Supports VLANs per tenant
- **No GPU Access**: Tenants never access GPU hardware directly
- **Read-Only Models**: Tenants can use models but not modify them

## 📖 Documentation

- [Architecture Details](ARCHITECTURE.md)
- [Setup Guide](docs/SETUP.md)
- [Tenant Operations](docs/OPERATIONS.md) (coming soon)

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Submit a pull request

## 📄 License

MIT License - See LICENSE file for details.

## 💬 Support

- Open an issue for bugs or feature requests
- Join the discussion in our Discord community
- Check [OpenClaw documentation](https://docs.openclaw.ai)