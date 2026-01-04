# Cortex K3S - Kubernetes Deployment

This repository contains the **Kubernetes/K3S deployment version** of Cortex, the autonomous AI orchestration platform.

## Repository Structure

```
cortex-k3s/
├── cortex-mcp-server/     # MCP server for k3s deployment
├── k3s-deployments/       # K3S-specific deployment manifests
├── k8s/                   # Kubernetes manifests (ITIL, monitoring, etc.)
├── coordination/          # Task coordination and state management
├── lib/                   # Core libraries
├── scripts/               # Deployment and management scripts
├── deploy/                # Deployment configurations
└── docs/                  # Documentation
```

## Key Components

### Cortex MCP Server
- **Location**: `cortex-mcp-server/`
- **Purpose**: Model Context Protocol server running in k3s
- **Access**: `https://cortex-mcp.ry-ops.dev` (via Tailscale)

### K3S Deployments
- **Cortex Chat**: Web UI at `https://chat.ry-ops.dev`
- **ITIL Services**: Incident management, service desk, change management
- **Monitoring**: Grafana, Prometheus, Loki, Tempo
- **Storage**: Longhorn distributed storage

### K8S Manifests
- **ITIL Stream 1**: Incident & Problem Management
- **ITIL Stream 2**: Service Level Management
- **ITIL Stream 4**: Continual Improvement
- **ITIL Stream 6**: Governance & Risk Management
- **Capacity Management**: Anomaly detection, forecasting
- **Router Mesh**: Intelligent routing system

## Deployment

All services are deployed to a k3s cluster with:
- **3 master nodes** (high availability)
- **4 worker nodes** (workload distribution)
- **MetalLB** for LoadBalancer services (L2 mode)
- **Traefik** for ingress (with Let's Encrypt SSL)
- **Tailscale** for secure private access

## Access

Services are accessible via Tailscale VPN:
- **Tailscale IP**: 100.81.79.19
- **Domain**: *.ry-ops.dev
- **Protocol**: HTTPS (self-signed certificates)

## Related Repositories

- **Local Cortex**: https://github.com/ry-ops/cortex (desktop development version)
- **K3S Cortex**: https://github.com/ry-ops/cortex-k3s (this repository)

## Architecture

Cortex K3S implements a microservices architecture with:
- Event-driven coordination
- Autonomous task routing via MoE (Mixture of Experts)
- ITIL 4 service management practices
- Self-healing and auto-recovery capabilities
- Comprehensive monitoring and observability

## Getting Started

1. Ensure kubectl is configured for the k3s cluster
2. Deploy core services: `./scripts/deploy-cortex.sh`
3. Access Cortex Chat: `https://chat.ry-ops.dev`
4. Monitor via Grafana: `https://grafana.ry-ops.dev`

## Documentation

See `docs/` directory for:
- Architecture diagrams
- Deployment guides
- ITIL implementation details
- API documentation
- Troubleshooting guides

---

**Note**: This is the production k3s deployment version. For local development, see https://github.com/ry-ops/cortex
