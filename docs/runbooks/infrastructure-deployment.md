# Infrastructure Deployment Runbook

## Overview

This runbook documents the complete process for deploying new HTTP services on Cortex k3s with:
- Scale-to-zero / Scale-from-zero via KEDA HTTP Add-on
- TLS certificates via cert-manager with DNS-01 challenge
- Traefik ingress routing
- Proper NetworkPolicy configuration

## Prerequisites

### Cluster Components Required
- KEDA with HTTP Add-on installed in `keda` namespace
- cert-manager installed
- Traefik ingress controller
- Cloudflare DNS for `ry-ops.dev` domain

### Cloudflare API Token
A Cloudflare API token with the following permissions is required for DNS-01 certificate validation:
- Zone → DNS → Edit
- Zone → Zone → Read
- Scoped to: `ry-ops.dev` zone

Token stored in: `cert-manager/cloudflare-api-token` secret

---

## Step-by-Step Deployment Process

### Step 1: Create the Deployment and Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
  namespace: my-namespace
  labels:
    app: my-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-service
  template:
    metadata:
      labels:
        app: my-service
    spec:
      containers:
      - name: server
        image: my-image:tag
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: my-namespace
  labels:
    app: my-service
spec:
  selector:
    app: my-service
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  type: ClusterIP
```

### Step 2: Create HTTPScaledObject for Scale-from-Zero

```yaml
apiVersion: http.keda.sh/v1alpha1
kind: HTTPScaledObject
metadata:
  name: my-service-http
  namespace: my-namespace
spec:
  hosts:
    - my-service.my-namespace.svc.cluster.local
  pathPrefixes:
    - /
  scaleTargetRef:
    name: my-service
    kind: Deployment
    apiVersion: apps/v1
    service: my-service
    port: 8080
  replicas:
    min: 0
    max: 5
  scalingMetric:
    requestRate:
      targetValue: 10
  scaledownPeriod: 120
```

### Step 3: Create NetworkPolicy for KEDA Interceptor Access

**CRITICAL**: If your namespace has `default-deny-all` NetworkPolicy, you MUST create a policy to allow KEDA interceptor traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-keda-interceptor
  namespace: my-namespace
spec:
  podSelector:
    matchLabels:
      app: my-service
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: keda
    ports:
    - protocol: TCP
      port: 8080
```

### Step 4: Add DNS Record in Cloudflare

Add an A record pointing to Traefik's LoadBalancer IP:

| Type | Name | Content | Proxy Status |
|------|------|---------|--------------|
| A | `my-service` | `10.88.145.200` | DNS only (grey cloud) |

**IMPORTANT**: Use "DNS only" (grey cloud), NOT "Proxied" (orange cloud) for internal services.

### Step 5: Create TLS Certificate with DNS-01 Challenge

**ALWAYS use `letsencrypt-prod-dns` ClusterIssuer** - it uses Cloudflare DNS-01 validation which:
- Bypasses HTTP routing complexity
- Works even with DNSSEC enabled/disabled
- Doesn't require NetworkPolicy changes for ACME solvers
- Supports wildcard certificates

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-service-tls
  namespace: my-namespace
spec:
  secretName: my-service-tls
  issuerRef:
    name: letsencrypt-prod-dns  # USE THIS - NOT letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - my-service.ry-ops.dev
```

### Step 6: Create Traefik IngressRoute

**Option A: Direct to Service** (simpler, recommended for most cases)
```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: my-service
  namespace: my-namespace
spec:
  entryPoints:
  - websecure
  routes:
  - kind: Rule
    match: Host(`my-service.ry-ops.dev`)
    services:
    - name: my-service
      port: 8080
  tls:
    secretName: my-service-tls
```

**Option B: Through KEDA Interceptor** (required for scale-from-zero via external traffic)

Requires Traefik `--providers.kubernetescrd.allowCrossNamespace=true` flag.

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: my-service
  namespace: my-namespace
spec:
  entryPoints:
  - websecure
  routes:
  - kind: Rule
    match: Host(`my-service.ry-ops.dev`)
    middlewares:
    - name: my-service-host-rewrite
      namespace: my-namespace
    services:
    - name: keda-add-ons-http-interceptor-proxy
      namespace: keda
      port: 8080
  tls:
    secretName: my-service-tls
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: my-service-host-rewrite
  namespace: my-namespace
spec:
  headers:
    customRequestHeaders:
      Host: my-service.my-namespace.svc.cluster.local
```

---

## Troubleshooting Guide

### Issue: Certificate Challenge Fails with DNSSEC Error

**Symptom:**
```
DNSSEC: RRSIGs Missing: validation failure
```

**Cause:** DNSSEC signatures are invalid or missing for the DNS records.

**Solutions:**
1. Use `letsencrypt-prod-dns` ClusterIssuer (DNS-01 challenge) instead of HTTP-01
2. If using HTTP-01, temporarily disable DNSSEC in Cloudflare (takes 24-48h to propagate)
3. Verify DS records at registrar match Cloudflare DNSSEC configuration

### Issue: KEDA Interceptor Returns 502 Bad Gateway

**Symptom:**
```
dial tcp <service-ip>:port: connect: connection refused
```

**Cause:** NetworkPolicy blocking traffic from keda namespace.

**Solution:** Create NetworkPolicy allowing ingress from keda namespace (see Step 3).

### Issue: Certificate HTTP-01 Challenge Returns 404

**Symptom:** cert-manager logs show "wrong status code '404', expected '200'"

**Causes:**
1. ACME solver pods not created (check for LimitRange CPU minimum issues)
2. Traefik not picking up the solver Ingress
3. DNS not pointing to Traefik IP

**Solution:** Use DNS-01 challenge instead - it's more reliable.

### Issue: Traefik Cross-Namespace Service Error

**Symptom:**
```
service keda/keda-add-ons-http-interceptor-proxy not in the parent resource namespace
```

**Cause:** Traefik doesn't allow cross-namespace service references by default.

**Solution:** Add these flags to Traefik deployment:
```
--providers.kubernetescrd.allowCrossNamespace=true
--providers.kubernetescrd.allowExternalNameServices=true
```

### Issue: Pods Failing with "minimum cpu usage per Container is 100m"

**Cause:** Namespace has a LimitRange with minimum CPU requirements.

**Solution:** Either adjust the LimitRange or ensure all pods meet minimum requirements.

### Issue: Image Pull Failures (TLS Handshake Error)

**Symptom:**
```
failed to do request: Head "https://registry-1.docker.io/...": remote error: tls: handshake failure
```

**Cause:** Network/firewall issues reaching Docker Hub.

**Solutions:**
1. Use images already cached on nodes
2. Use internal registry
3. Schedule pods on nodes that have the image cached

---

## Verification Checklist

After deployment, verify each component:

```bash
# 1. Check HTTPScaledObject is active
kubectl get httpscaledobjects -n my-namespace

# 2. Check Certificate is ready
kubectl get certificates -n my-namespace

# 3. Check IngressRoute exists
kubectl get ingressroutes -n my-namespace

# 4. Check NetworkPolicy allows KEDA
kubectl get networkpolicies -n my-namespace

# 5. Test scale-from-zero internally
kubectl exec -n my-namespace some-pod -- wget -qO- \
  --header="Host: my-service.my-namespace.svc.cluster.local" \
  http://keda-add-ons-http-interceptor-proxy.keda.svc.cluster.local:8080/health

# 6. Check pod scaled up
kubectl get pods -n my-namespace -l app=my-service

# 7. Check KEDA events
kubectl get events -n my-namespace --field-selector reason=KEDAScaleTargetActivated
```

---

## Key Resources

| Resource | Namespace | Purpose |
|----------|-----------|---------|
| `keda-add-ons-http-interceptor-proxy` | keda | HTTP proxy for scale-from-zero |
| `letsencrypt-prod-dns` | (cluster-scoped) | ClusterIssuer with Cloudflare DNS-01 |
| `cloudflare-api-token` | cert-manager | Cloudflare API credentials |
| Traefik LoadBalancer IP | kube-system | `10.88.145.200` |

---

## Architecture Flow

```
External Request
       ↓
   DNS (Cloudflare)
       ↓
   Traefik (10.88.145.200)
       ↓
   [Option A: Direct]     [Option B: Via KEDA]
         ↓                        ↓
   Service → Pod          KEDA Interceptor
                                  ↓
                          (scales pod if at 0)
                                  ↓
                          Service → Pod
```

---

## Quick Reference Commands

```bash
# Delete and recreate certificate
kubectl delete certificate my-service-tls -n my-namespace
kubectl apply -f certificate.yaml

# Check challenge status
kubectl get challenges -n my-namespace
kubectl describe challenge <name> -n my-namespace

# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager --tail=100

# Check KEDA interceptor logs
kubectl logs -n keda -l app.kubernetes.io/component=interceptor --tail=50

# Restart KEDA interceptor (if needed)
kubectl rollout restart deploy/keda-add-ons-http-interceptor -n keda
```

---

*Last Updated: 2026-01-17*
*Author: Cortex Infrastructure Team*
