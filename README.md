# k8s-5g-nf-deploy

> Cloud-native 5G Core Network Function deployment on Kubernetes — NRF + SCP with Helm, Jenkins CI/CD, HPA, and Prometheus observability.

---

## Overview

This repository demonstrates a production-style Kubernetes deployment of two 5G Core Network Functions based on [3GPP TS 23.501](https://www.3gpp.org/ftp/Specs/archive/23_series/23.501/) Service-Based Architecture (SBA):

- **NRF** (Network Repository Function) — NF profile registration, deregistration, and discovery per 3GPP TS 29.510
- **SCP** (Service Communication Proxy) — indirect communication and delegated NRF discovery (Model D) per 3GPP TS 29.500

Deployment highlights:
- Indirect communication via SCP (Model D pattern)
- NRF-based NF profile registration and discovery
- Namespace isolation per 5G network function domain
- Readiness and liveness probes aligned with NF health-check patterns
- Horizontal Pod Autoscaler (HPA) for traffic-driven scaling
- Prometheus metrics scraping and Grafana-ready ServiceMonitor CRDs

> **Note:** NRF and SCP containers use an [nginx](https://nginx.org/) stub to simulate NF HTTP/2 SBI endpoints. The Kubernetes architecture, Helm packaging, and CI/CD pipeline are the focus — not the NF software itself. For a fully functional open-source 5G Core, see [free5GC](https://github.com/free5gc/free5gc) or [open5GS](https://github.com/open5gs/open5gs).

---

## Architecture

```
                        ┌─────────────────────────────────┐
                        │        Namespace: 5g-core        │
                        │                                  │
   AMF / SMF / PCF ────▶│  SCP  (Model D proxy)            │
   (NF consumers)       │    └──▶ NRF (NF discovery)       │
                        │                                  │
                        │  Prometheus scrapes /metrics     │
                        └─────────────────────────────────┘

   Jenkins Pipeline:
   Git push ──▶ Lint/Validate ──▶ Helm dry-run ──▶ Deploy Staging ──▶ Deploy Prod
```

**Model D Indirect Communication Flow (3GPP TS 29.500):**
1. NF Consumer (e.g. AMF) sends service request to SCP
2. SCP performs NRF discovery on behalf of the consumer (`Nnrf_NFDiscovery`)
3. SCP routes the request to the selected NF Producer
4. This offloads discovery logic from each NF — reducing per-NF complexity and centralising routing control

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
│   ├── nrf/                    # Helm chart for NRF
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       └── servicemonitor.yaml
│   └── scp/                    # Helm chart for SCP
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
| Jenkins | 2.400+ | CI/CD pipeline (optional for local testing) |

---

## Quick Start

### 1. Create namespace and base resources

```bash
kubectl apply -f manifests/namespace.yaml
kubectl apply -f manifests/configmap.yaml
```

### 2. Deploy NRF

```bash
helm install nrf helm/nrf \
  --namespace 5g-core \
  --values helm/nrf/values.yaml
```

### 3. Deploy SCP

```bash
helm install scp helm/scp \
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
kubectl port-forward svc/scp 8080:80 -n 5g-core

# Simulate an NF discovery request via SCP
curl http://localhost:8080/healthz
```

---

## Helm Configuration

### NRF values (`helm/nrf/values.yaml`)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `replicaCount` | `2` | Number of NRF pods |
| `image.tag` | `1.25-alpine` | Container image tag |
| `resources.requests.cpu` | `250m` | CPU request per pod |
| `resources.requests.memory` | `256Mi` | Memory request per pod |
| `autoscaling.enabled` | `true` | Enable HPA |
| `autoscaling.maxReplicas` | `5` | Max pods under load |

### SCP values (`helm/scp/values.yaml`)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `replicaCount` | `2` | Number of SCP pods |
| `nrf.endpoint` | `http://nrf:8080` | NRF service endpoint for discovery |
| `autoscaling.enabled` | `true` | Enable HPA |
| `autoscaling.maxReplicas` | `6` | Max pods under load |

---

## Jenkins CI/CD Pipeline

The `jenkins/Jenkinsfile` implements a multi-stage pipeline:

```
Checkout → Helm Lint → Manifest Validate → Helm Dry-Run → Deploy Staging → Smoke Test → Approval → Deploy Production
```

- Staging deploys to namespace `5g-core-staging`
- Production deploys to namespace `5g-core`
- `--atomic` flag ensures automatic rollback if production deploy fails
- Manual approval gate between staging and production (change window control)

---

## Observability

Both NRF and SCP deployments include `ServiceMonitor` resources (Prometheus Operator CRD) for automatic metrics scraping:

```yaml
prometheus.io/scrape: "true"
prometheus.io/port: "9090"
prometheus.io/path: "/metrics"
```

Deploy [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) to enable full observability:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
```

---

## Design Rationale

Key architecture decisions and why they matter in production 5G carrier environments:

- **Namespace isolation** limits blast radius of NF failures and simplifies RBAC and quota management
- **Model D SCP routing** centralises NRF interaction — NF consumers only talk to SCP, reducing per-NF complexity and improving observability
- **HPA with conservative scale-down** (`stabilizationWindowSeconds: 300`) prevents flapping during bursty signalling traffic (e.g. mass UE re-attach after outage)
- **`maxUnavailable: 0` rolling update** ensures zero dropped requests during NF upgrades — critical in 24/7 carrier environments
- **PodDisruptionBudget** ensures at least one NRF/SCP pod stays available during node maintenance
- **Init container on SCP** prevents SCP from starting before NRF is ready, avoiding early discovery failures

---

## References

- [3GPP TS 23.501](https://www.3gpp.org/ftp/Specs/archive/23_series/23.501/) — System Architecture for 5G
- [3GPP TS 29.500](https://www.3gpp.org/ftp/Specs/archive/29_series/29.500/) — 5G SBA Technical Realization
- [3GPP TS 29.510](https://www.3gpp.org/ftp/Specs/archive/29_series/29.510/) — NRF Services
- [free5GC](https://github.com/free5gc/free5gc) — Open-source 5G Core implementation
- [open5GS](https://github.com/open5gs/open5gs) — Open-source 5G/4G Core
- [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts) — Prometheus + Grafana on Kubernetes

---

## Author

**Ravinder Kumar** — Senior Cloud & Telecom Engineer  
[LinkedIn](https://linkedin.com/in/ravinder-kumar-76943620) · [GitHub](https://github.com/ravionline86)

16+ years in Tier-1 carrier environments. Hands-on experience deploying and operating cloud-native 5G Core Network Functions (NRF, SCP, PCF) on Kubernetes for international telecom operators.
