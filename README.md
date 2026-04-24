# k8s-5g-nf-deploy

> Cloud-native 5G Core Network Function deployment on Kubernetes — NRF + SCP with Helm, Jenkins CI/CD, HPA, and Prometheus observability.

---

## Overview

This repository demonstrates a production-style Kubernetes deployment of two 5G Core Network Functions:

- **OC-NRF** (Network Repository Function) — service registration and discovery for 5G NFs
- **OC-SCP** (Service Communication Proxy) — indirect communication and delegated discovery (Model D)

The deployment follows **3GPP TS 23.501** architectural principles for Service-Based Architecture (SBA), including:
- Indirect communication via SCP (Model D pattern)
- NRF-based NF profile registration and discovery
- Namespace isolation per 5G network function domain
- Readiness and liveness probes aligned with NF health-check patterns
- Horizontal Pod Autoscaler (HPA) for traffic-driven scaling
- Prometheus metrics scraping and Grafana-ready annotations

> **Note:** NRF and SCP containers use an nginx stub to simulate NF HTTP/2 SBI endpoints. The focus is the Kubernetes architecture, Helm packaging, and CI/CD pipeline — not the NF software itself. This mirrors real-world deployment patterns used in Oracle OC-NRF/OC-SCP production rollouts.

---

## Architecture

```
                        ┌─────────────────────────────────┐
                        │        Namespace: 5g-core        │
                        │                                  │
   AMF / SMF / PCF ────▶│  SCP (Model D proxy)             │
   (NF consumers)       │    └──▶ NRF (NF discovery)       │
                        │                                  │
                        │  Prometheus scrapes /metrics     │
                        └─────────────────────────────────┘

   Jenkins Pipeline:
   Git push ──▶ Lint/Validate ──▶ Helm dry-run ──▶ Deploy to staging ──▶ Deploy to prod
```

**Model D Indirect Communication Flow:**
1. NF Consumer (e.g. AMF) sends request to SCP
2. SCP performs NRF discovery on behalf of the consumer
3. SCP routes the request to the selected NF Producer
4. This offloads discovery logic from each NF — core design in Oracle OC-SCP deployments

---

## Repository Structure

```
k8s-5g-nf-deploy/
├── README.md
├── manifests/
│   ├── namespace.yaml          # Namespace + ResourceQuota + LimitRange
│   ├── configmap.yaml          # NRF and SCP runtime configuration
│   └── hpa.yaml                # HorizontalPodAutoscaler for NRF and SCP
├── helm/
│   ├── nrf/                    # Helm chart for OC-NRF
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       └── servicemonitor.yaml
│   └── scp/                    # Helm chart for OC-SCP
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           └── servicemonitor.yaml
└── jenkins/
    └── Jenkinsfile             # CI/CD pipeline: validate → staging → prod
```

---

## Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| kubectl | 1.27+ | Kubernetes CLI |
| Helm | 3.12+ | Chart packaging and deployment |
| Kubernetes cluster | 1.27+ | Minikube / kind / EKS / OKD |
| Jenkins | 2.400+ | CI/CD pipeline (optional) |

---

## Quick Start

### 1. Create the namespace and base resources

```bash
kubectl apply -f manifests/namespace.yaml
kubectl apply -f manifests/configmap.yaml
```

### 2. Deploy NRF

```bash
helm install oc-nrf helm/nrf \
  --namespace 5g-core \
  --values helm/nrf/values.yaml
```

### 3. Deploy SCP

```bash
helm install oc-scp helm/scp \
  --namespace 5g-core \
  --values helm/scp/values.yaml
```

### 4. Verify deployments

```bash
kubectl get pods -n 5g-core
kubectl get svc -n 5g-core
kubectl get hpa -n 5g-core
```

### 5. Test SCP → NRF connectivity (Model D simulation)

```bash
# Port-forward the SCP service
kubectl port-forward svc/oc-scp 8080:80 -n 5g-core

# Simulate an NF discovery request via SCP
curl http://localhost:8080/healthz
```

---

## Helm Configuration

### NRF values (helm/nrf/values.yaml)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `replicaCount` | `2` | Number of NRF pods |
| `image.tag` | `latest` | Container image tag |
| `resources.requests.cpu` | `250m` | CPU request per pod |
| `resources.requests.memory` | `256Mi` | Memory request per pod |
| `autoscaling.enabled` | `true` | Enable HPA |
| `autoscaling.maxReplicas` | `5` | Max pods under load |

### SCP values (helm/scp/values.yaml)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `replicaCount` | `2` | Number of SCP pods |
| `nrf.endpoint` | `http://oc-nrf:8080` | NRF service endpoint for discovery |
| `autoscaling.enabled` | `true` | Enable HPA |

---

## Jenkins CI/CD Pipeline

The `jenkins/Jenkinsfile` implements a multi-stage pipeline:

```
Checkout → Helm Lint → Helm Dry-Run → Deploy Staging → Smoke Test → Deploy Production
```

- Staging deploys to namespace `5g-core-staging`
- Production deploys to namespace `5g-core`
- Rollback is triggered automatically if smoke tests fail
- Pipeline uses `helm upgrade --install` for idempotent deployments

---

## Observability

Both NRF and SCP deployments include `ServiceMonitor` resources (Prometheus Operator CRD) to enable automatic metrics scraping. Annotations on pods follow the standard Prometheus scrape pattern:

```yaml
prometheus.io/scrape: "true"
prometheus.io/port: "9090"
prometheus.io/path: "/metrics"
```

Import the included Grafana dashboard JSON (`grafana/5g-nf-dashboard.json`) to visualise:
- Pod CPU and memory utilisation
- HPA replica counts vs target
- HTTP request rates (simulated)

---

## Background: Why This Architecture Matters

In Oracle OC-SCP / OC-NRF production deployments (e.g. Rakuten Mobile, Korea Telecom):

- **Namespace isolation** separates 5G NF domains, limiting blast radius of failures
- **Model D** (indirect SCP routing) is preferred over Model C because it centralises NRF interaction, reducing per-NF complexity
- **HPA** is essential — SCP and NRF must scale horizontally under peak signalling load (e.g. mass attach storms during network outages)
- **Helm charts** enable reproducible, version-controlled rollouts with environment-specific value overrides
- **Jenkins pipelines** with `helm upgrade --install` support zero-downtime rolling upgrades, which is critical in 24/7 carrier environments

---

## Author

**Ravinder Kumar** — Senior Principal Consultant, Cloud & Telecom Delivery  
[LinkedIn](https://linkedin.com/in/ravinder-kumar-76943620) · [GitHub](https://github.com/ravionline86)

16+ years in Tier-1 carrier environments. Hands-on with Oracle OC-NRF and OC-SCP deployments for Rakuten Mobile (Japan) and Korea Telecom.
