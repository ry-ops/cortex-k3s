# Cortex K8s Documentation

**Official documentation repository for Cortex infrastructure management system**

## Overview

This repository contains comprehensive documentation for the Cortex system, discovered and documented through extensive testing of the deployed Kubernetes infrastructure.

All documentation is stored as Kubernetes ConfigMaps in the `cortex` namespace and backed up here for version control and reference.

## Quick Links

- **Primary Access**: https://chat.ry-ops.dev
- **System Status**: Production-ready and operational
- **Last Updated**: 2026-01-05

## Documentation Index

All documents stored in `configmaps/` directory:

### 1. 8-Hour Exploration Summary (25KB) - START HERE
**File**: `8-hour-exploration-summary-backup.yaml`  
**Purpose**: Executive summary of entire exploration

**Contents**:
- Mission and approach
- 8 key discoveries
- Complete testing results
- Performance metrics
- Deployment architecture
- Key insights and recommendations

**Use this to**: Get high-level understanding of Cortex system

### 2. Cortex Integration Guide (32KB) - MASTER REFERENCE
**File**: `cortex-integration-guide-backup.yaml`  
**Purpose**: Complete reference connecting all components

**Contents**:
- Quick start (3 access methods)
- Complete architecture layers (1-5)
- Request flow examples with timing
- Rate limiting and token budgets
- Deployment structure
- Monitoring and observability
- Troubleshooting guide
- Security considerations

**Use this to**: Understand how everything connects and how to use Cortex

### 3. Cortex Workflows (18KB)
**File**: `cortex-workflows-backup.yaml`  
**Purpose**: Real workflow documentation

**Contents**:
- Architecture overview
- Component connections
- Detailed workflow examples
- Redis queue management
- Token throttle system
- Current system status

**Use this to**: Understand how requests flow through the system

### 4. Cortex Tools Catalog (14KB)
**File**: `cortex-tools-catalog-backup.yaml`  
**Purpose**: Complete catalog of all 17 tools

**Contents**:
- Tool categories
- Infrastructure Query (3 tools)
- UniFi Network (3 tools)
- Sandfly Security (6 tools)
- Task Management (4 tools)
- Agent Management (1 tool)
- Usage examples

**Use this to**: Reference available tools and their capabilities

### 5. LLM-D Architecture (9.6KB)
**File**: `llm-d-architecture-backup.yaml`  
**Purpose**: LLM Daemon architecture (discovered = Orchestrator)

**Contents**:
- What is LLM-D (Orchestrator pod)
- Request flow
- Token management
- Caching strategy
- Tool execution
- Observable behavior

**Use this to**: Understand the core orchestrator component

### 6. Task Processing (22KB)
**File**: `cortex-task-processing-backup.yaml`  
**Purpose**: Task queue and worker processing

**Contents**:
- Complete task lifecycle
- Dual persistence (Redis + filesystem)
- Priority queue implementation
- Rate limiting (40k tokens/min)
- Worker auto-shutdown
- Testing examples

**Use this to**: Understand how tasks are created and processed

### 7. MoE Routing (28KB)
**File**: `cortex-moe-routing-backup.yaml`  
**Purpose**: Mixture of Experts routing system

**Contents**:
- 6-tier routing system
- Keyword-based confidence scoring
- Master agent definitions
- Routing algorithm
- Token budget allocations
- Implementation state

**Use this to**: Understand the routing and master agent architecture

## Quick Start

### Option 1: Chat Interface (Recommended)
```
1. Open: https://chat.ry-ops.dev
2. Ask: "Show me pods in cortex namespace"
3. Try: "What's the UniFi network status?"
4. Create task: "Create a task to scan for vulnerabilities"
```

### Option 2: View Documentation in K8s
```bash
# List all ConfigMaps
kubectl get configmaps -n cortex -l component=documentation

# View a specific doc
kubectl get configmap -n cortex cortex-integration-guide -o jsonpath='{.data.integration-guide\.md}'

# Or use kubectl describe
kubectl describe configmap -n cortex 8-hour-exploration-summary
```

### Option 3: Read from GitHub
```bash
# Clone this repo
git clone https://github.com/ry-ops/cortex-k3s.git
cd cortex-k3s/configmaps

# View any doc with your favorite editor
cat 8-hour-exploration-summary-backup.yaml
```

## System Architecture

```
User (Browser)
     ↓
https://chat.ry-ops.dev (Nginx)
     ↓
Backend (cortex-chat-backend-simple:8080)
     ↓
Orchestrator (cortex-orchestrator:8000)
     ↓
Claude API (api.anthropic.com)
     ↓
17 Tools → MCP Servers / Direct APIs
     ↓
Infrastructure (UniFi, Proxmox, Sandfly, K8s)
```

## Key Components

- **Orchestrator**: Central coordinator (cortex-orchestrator pod)
- **Queue Workers**: Task processors (2 replicas)
- **Redis**: Priority queues and caching
- **MCP Servers**: UniFi, Proxmox, Sandfly integration
- **Documentation Master**: Active master agent for Sandfly docs

## System Status

**Production Ready**: ✅  
**Chat Interface**: ✅ Working  
**17 Tools**: ✅ Functional  
**Task Processing**: ✅ Verified (test task: 61 tokens!)  
**MCP Servers**: ✅ Healthy  
**Rate Limiting**: ✅ Active  

## Documentation Stats

- **Total ConfigMaps**: 7
- **Total Size**: 148KB
- **Created**: 2026-01-05
- **Method**: Deployed system testing (not local file reading)
- **Tokens Used**: ~85,000 tokens
- **Testing**: End-to-end workflows verified

## How to Use This Documentation

**If you're new to Cortex**:
1. Start with: `8-hour-exploration-summary-backup.yaml`
2. Then read: `cortex-integration-guide-backup.yaml`
3. Explore specific topics as needed

**If you're an operator**:
1. Reference: `cortex-integration-guide-backup.yaml` (troubleshooting, scaling)
2. Monitor: `cortex-workflows-backup.yaml` (request flows)
3. Debug: `cortex-task-processing-backup.yaml` (task issues)

**If you're a developer**:
1. Architecture: `llm-d-architecture-backup.yaml`
2. Tools: `cortex-tools-catalog-backup.yaml`
3. MoE: `cortex-moe-routing-backup.yaml`

## Deploying ConfigMaps

To deploy these ConfigMaps to your cluster:

```bash
# Deploy all ConfigMaps
kubectl apply -f configmaps/

# Deploy a specific ConfigMap
kubectl apply -f configmaps/cortex-integration-guide-backup.yaml

# Verify
kubectl get configmaps -n cortex -l component=documentation
```

## Contributing

This documentation was generated through:
1. Testing deployed k8s infrastructure
2. Examining running pods and services
3. Testing actual workflows
4. Verifying all tools and endpoints
5. Creating end-to-end test tasks

To update:
1. Test changes in deployed environment
2. Update corresponding ConfigMap YAML
3. Apply to cluster: `kubectl apply -f configmaps/NAME.yaml`
4. Commit and push to this repo

## License

Internal documentation for Cortex infrastructure management system.

## Contact

For access to Cortex system: https://chat.ry-ops.dev

---

**Generated with Claude Code**  
**Last Updated**: 2026-01-05  
**Status**: ✅ Complete and operational
