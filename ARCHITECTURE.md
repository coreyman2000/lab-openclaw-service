# OpenClaw Lab Service Architecture

## Overview
Multi-tenant OpenClaw service where each tenant gets isolated Proxmox VMs with Docker-based OpenClaw instances, connecting to shared GPU-powered Ollama backends.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           PROXMOX CLUSTER                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────┐     ┌──────────────────┐     ┌─────────────────┐ │
│  │   GPU Node 1     │     │   GPU Node 2     │     │   Compute Node  │ │
│  │   (L40 GPUs)     │     │   (L40 GPUs)     │     │   (VMs)           │ │
│  │                  │     │                  │     │                  │ │
│  │  ┌────────────┐  │     │  ┌────────────┐  │     │  ┌───────────┐   │ │
│  │  │ Ollama LXC │  │     │  │ Ollama LXC │  │     │  │ Tenant VM │   │ │
│  │  │ Container  │  │     │  │ Container  │  │     │  │ (Team A)  │   │ │
│  │  │ http://...  │  │     │  │ http://...  │  │     │  │           │   │ │
│  │  │ :11434     │  │◄────┼──┤ :11434     │  │     │  │ ┌───────┐ │   │ │
│  │  └────────────┘  │     │  └────────────┘  │     │  │ │OpenClaw│ │   │ │
│  │        ▲         │     │        ▲         │     │  │ │Container│ │   │ │
│  └────────┼─────────┘     └────────┼─────────┘     │  │ └───────┘ │   │ │
│           │                      │               │  │ ┌───────┐ │   │ │
│           │     Shared GPU       │               │  │ │OpenClaw│ │   │ │
│           │     Model Pool       │               │  │ │Container│ │   │ │
│           │                      │               │  │ │(Proj 2) │ │   │ │
│           └──────────────────────┘               │  │ └───────┘ │   │ │
│                                                 │  └───────────┘   │ │
│                                                 │                  │ │
│                                                 │  ┌───────────┐   │ │
│                                                 │  │ Tenant VM │   │ │
│                                                 │  │ (Team B)  │   │ │
│                                                 │  │           │   │ │
│                                                 │  │ ┌───────┐ │   │ │
│                                                 │  │ │OpenClaw│ │   │ │
│                                                 │  │ │Container│ │   │ │
│                                                 │  │ └───────┘ │   │ │
│                                                 │  └───────────┘   │ │
│                                                 └──────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

## Key Components

### 1. GPU Model Layer (LXC on GPU Nodes)
- **Purpose**: Shared Ollama instances providing model inference
- **Access**: HTTP API endpoints (no direct tenant access)
- **Scaling**: Multiple Ollama LXCs for load balancing
- **Configuration**: Each Ollama LXC registers with service discovery

### 2. Tenant Layer (VMs on Compute Nodes)
- **Isolation**: Each tenant gets dedicated Proxmox VM
- **OS**: Ubuntu 22.04/24.04 LTS
- **Docker**: Docker CE with compose plugin
- **OpenClaw**: Multiple OpenClaw containers per tenant (1 per project/team)

### 3. Service Discovery
- Simple registry tracking available Ollama endpoints
- Tenants configure OpenClaw to connect to model pool

## Resource Allocation

| Tenant Tier | VM Specs | OpenClaw Instances | Concurrent Users |
|-------------|----------|-------------------|------------------|
| Basic | 2 vCPU, 4GB RAM | 1-2 | 5-10 |
| Standard | 4 vCPU, 8GB RAM | 3-5 | 15-25 |
| Premium | 8 vCPU, 16GB RAM | 10+ | 50+ |

## Network Design

```
Tenant VM Network:
┌─────────────────────────────────────┐
│         Tenant VM                  │
│  ┌─────────────────────────────┐   │
│  │      OpenClaw Container 1   │   │
│  │      Port: 3000              │   │
│  └─────────────────────────────┘   │
│  ┌─────────────────────────────┐   │
│  │      OpenClaw Container 2   │   │
│  │      Port: 3001              │   │
│  └─────────────────────────────┘   │
│  ┌─────────────────────────────┐   │
│  │      OpenClaw Container N   │   │
│  │      Port: 3000+N            │   │
│  └─────────────────────────────┘   │
│                                    │
│  Outbound: Ollama Pool (GPU LXCs)  │
│          Port 11434               │
└─────────────────────────────────────┘
```

## Security Model

1. **VM Isolation**: Each tenant in separate VM (kernel-level isolation)
2. **Network Segmentation**: VLANs per tenant or private network ranges
3. **Model Access**: Read-only API access to Ollama (no model management)
4. **Data Isolation**: Each OpenClaw container has isolated volumes
5. **No GPU Passthrough**: Tenants never touch GPU hardware directly

## Deployment Workflow

1. **Provision**: Ansible creates Proxmox VM from template
2. **Bootstrap**: VM boots, Docker installed via cloud-init
3. **Configure**: OpenClaw containers deployed via Docker Compose
4. **Register**: Tenant instance registered in service catalog
5. **Handoff**: Credentials delivered to tenant team
