# OpenClaw Lab Service Setup Guide

## Prerequisites

### Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| GPU Nodes | 1x L40 GPU | 2x L40 GPUs |
| Compute Nodes | 1x (4 cores, 16GB RAM) | 2x (8 cores, 32GB RAM) |
| Storage | 100GB SSD | 500GB SSD |
| Network | 1Gbps | 10Gbps |

### Software Requirements

- **Proxmox VE 8.x** or later
- **Ansible 2.15+** with community modules:
  ```bash
  ansible-galaxy collection install community.general
  ansible-galaxy collection install community.docker
  ```
- **Docker** (for local testing)
- **Proxmox API Token** with VM/LXC permissions

## Quick Start

### 1. Clone and Configure

```bash
cd /opt
git clone https://github.com/your-org/lab-openclaw-service.git
cd lab-openclaw-service/ansible
```

### 2. Configure Inventory

Edit `inventory/hosts.yml`:

```yaml
proxmox_hosts:
  hosts:
    pve-gpu-01:
      ansible_host: YOUR_GPU_NODE_IP
      gpu_node: true
      gpus:
        - nvidia_l40_0
        - nvidia_l40_1
    pve-compute-01:
      ansible_host: YOUR_COMPUTE_NODE_IP
      gpu_node: false
  vars:
    ansible_user: root
```

### 3. Set Up Authentication

Create `inventory/group_vars/proxmox_hosts.yml`:

```yaml
---
proxmox_token_id: "your-token-id"
proxmox_token_secret: "your-token-secret"
```

Or use SSH keys:

```bash
ssh-copy-id root@YOUR_PROXMOX_IP
```

### 4. Deploy Ollama Backends

```bash
cd ansible
ansible-playbook -i inventory/ playbooks/deploy-ollama-lxc.yml
```

This creates LXC containers with Ollama on each GPU.

### 5. Provision First Tenant

```bash
ansible-playbook -i inventory/ playbooks/provision-tenant.yml
```

You'll be prompted for:
- Tenant name (e.g., `team-alpha`)
- Tier (basic/standard/premium)
- Number of OpenClaw instances

## Detailed Configuration

### Ollama LXC Configuration

Edit `group_vars/ollama_backends.yml`:

```yaml
---
# Ollama LXC settings
ollama_memory: 32768
ollama_cores: 8
ollama_disk: 32

# Models to pre-load
ollama_default_models:
  - llama3.2
  - mistral
  - codellama
```

### Tenant Tier Configuration

Edit `group_vars/tenant_vms.yml`:

```yaml
---
tenant_tiers:
  basic:
    cores: 2
    memory: 4096
    disk: 20
    openclaw_instances: 2
    monthly_cost: 50
  
  standard:
    cores: 4
    memory: 8192
    disk: 40
    openclaw_instances: 5
    monthly_cost: 100
  
  premium:
    cores: 8
    memory: 16384
    disk: 80
    openclaw_instances: 10
    monthly_cost: 200
```

### OpenClaw Configuration Template

Edit `templates/openclaw-config.yml.j2` for default tenant settings.

## Tenant Workflow

### Creating a New Tenant

```bash
# Interactive provisioning
ansible-playbook -i inventory/ playbooks/provision-tenant.yml

# Or with variables
ansible-playbook -i inventory/ playbooks/provision-tenant.yml \
  -e "tenant_name=team-beta" \
  -e "tenant_tier=standard" \
  -e "openclaw_instances=3"
```

### Managing Tenants

```bash
# Check status
ansible-playbook -i inventory/ playbooks/tenant-ops.yml \
  -e "operation=status tenant=team-alpha"

# Restart services
ansible-playbook -i inventory/ playbooks/tenant-ops.yml \
  -e "operation=restart tenant=team-alpha"

# View logs
ansible-playbook -i inventory/ playbooks/tenant-ops.yml \
  -e "operation=logs tenant=team-alpha"

# Update OpenClaw
ansible-playbook -i inventory/ playbooks/tenant-ops.yml \
  -e "operation=update tenant=team-alpha"

# Add instance
ansible-playbook -i inventory/ playbooks/tenant-ops.yml \
  -e "operation=add-instance tenant=team-alpha"

# Delete tenant (⚠️ Destructive)
ansible-playbook -i inventory/ playbooks/tenant-ops.yml \
  -e "operation=remove tenant=team-alpha"
```

## Access Patterns

### Direct VM Access

Each tenant gets SSH access to their VM:

```bash
ssh openclaw@<tenant-vm-ip>
```

### OpenClaw Web UI

Access individual OpenClaw instances:

```
http://<tenant-vm-ip>:3000  # Instance 1
http://<tenant-vm-ip>:3001  # Instance 2
...
```

### Reverse Proxy (Optional)

Set up nginx/traefik for cleaner URLs:

```nginx
# Example nginx config
server {
    listen 80;
    server_name team-alpha.lab.local;
    
    location / {
        proxy_pass http://<tenant-vm-ip>:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

## Monitoring

### Check GPU Utilization

```bash
# On GPU nodes
nvidia-smi
watch -n 1 nvidia-smi
```

### Check LXC Status

```bash
pct list
pct status <lxc-id>
```

### Check All Tenants

```bash
ansible-playbook -i inventory/ playbooks/tenant-ops.yml \
  -e "operation=status tenant=all"
```

## Troubleshooting

### Ollama LXC Won't Start

1. Check GPU passthrough configuration
2. Verify nvidia drivers on host: `nvidia-smi`
3. Check LXC logs: `pct logs <lxc-id>`

### OpenClaw Can't Connect to Models

1. Verify Ollama is running: `curl http://<ollama-ip>:11434/api/tags`
2. Check firewall rules between tenant VMs and GPU nodes
3. Verify config.yaml has correct endpoints

### VM Provisioning Fails

1. Verify template exists: `qm list`
2. Check storage availability: `pvesm status`
3. Review API token permissions

## Security Considerations

1. **Network Isolation**: Use VLANs per tenant
2. **Firewall**: Restrict inter-tenant communication
3. **Access Control**: Rotate SSH keys regularly
4. **Updates**: Automate security updates for LXCs and VMs
5. **Logging**: Centralize logs for audit trails

## Backup and Recovery

### Backup Tenant VMs

```bash
# Create VM backup
vzdump <vmid> --storage <backup-storage> --mode snapshot

# Restore from backup
qmrestore <backup-file> <new-vmid>
```

### Backup Ollama Models

Models are stored in LXC at `/root/.ollama/models/`. Back up the LXC:

```bash
vzdump <lxc-id> --storage <backup-storage>
```

## Maintenance

### Update Ollama

```bash
# Update all Ollama LXCs
ansible-playbook -i inventory/ playbooks/update-ollama.yml
```

### Clean Up Old Containers

```bash
# On tenant VMs
docker system prune -a
```

### Scale Up

Add more GPU nodes to the inventory and re-run deployment.