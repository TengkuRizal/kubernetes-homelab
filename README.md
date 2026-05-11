# kubernetes-homelab

3-node Kubernetes cluster built with kubeadm — spanning multiple security zones, with Kyverno policy enforcement, full observability stack, and GitLab CI/CD integration.

![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.29.15-326CE5?logo=kubernetes)
![Kyverno](https://img.shields.io/badge/Policy-Kyverno-00B2B2)
![Prometheus](https://img.shields.io/badge/Monitoring-Prometheus%20%2B%20Grafana-E6522C?logo=prometheus)
![GitLab](https://img.shields.io/badge/CI%2FCD-GitLab-FC6D26?logo=gitlab)
![Status](https://img.shields.io/badge/Cluster-Ready-brightgreen)

---

## Cluster Architecture

<pre>
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                           │
│                                                                 │
│  k8master       10.10.1.70   VLAN 10 (SOC)    control-plane     │
│  k8s-worker-1   10.10.2.71   VLAN 20 (Target) worker            │
│  k8s-worker-2   10.10.3.72   VLAN 30 (Attacker) worker          │
│                                                                 │
│  Version    : Kubernetes v1.29.15                               │
│  Runtime    : containerd                                        │
│  CNI        : Calico                                            │
│  OS         : Ubuntu 20.04 / 22.04 / 24.04                      │
└─────────────────────────────────────────────────────────────────┘
</pre>

Workers intentionally span across security VLANs — reflecting real-world multi-zone cluster design where workloads run in different trust boundaries.

---

## Deployed Stack

| Namespace | Component | Purpose |
|---|---|---|
| `kube-system` | Calico CNI | Pod networking + NetworkPolicy enforcement |
| `kyverno` | Kyverno | Admission control — Policy as Code |
| `monitoring` | kube-prometheus-stack | Prometheus + Grafana + AlertManager |
| `monitoring` | node-exporter | Node-level metrics (all 3 nodes) |
| `ingress-nginx` | NGINX Ingress Controller | HTTP/S ingress routing |
| `local-path-storage` | local-path-provisioner | Dynamic PV provisioning |
| `demo` | demo-nginx | Demo workload — 2 replicas |

---

## Kyverno — Policy as Code

3 ClusterPolicies enforced cluster-wide with `validationFailureAction: Enforce`.

### 1. disallow-latest-image-tag

Blocks any Pod using `:latest` tag or untagged images.

```yaml
# Enforces explicit versioned tags on all containers
deny:
  conditions:
    any:
    - key: "{{ element.image }}"
      operator: Equals
      value: "*:latest"
```

**Why:** `:latest` images are non-deterministic — same tag can pull different code on every deploy. Explicit tags ensure reproducible deployments.

---

### 2. disallow-privileged-containers

Blocks privileged containers across all container types.

```yaml
# Applies to containers, initContainers, ephemeralContainers
=(securityContext):
  =(privileged): "false"
```

**Why:** Privileged containers have full host access — equivalent to running as root on the node. This is a critical container escape vector.

---

### 3. require-approved-registry

**Supply chain security gate** — only images from the internal GitLab registry are allowed.

```yaml
# Only GitLab self-hosted registry permitted
image: "10.10.1.101:5050/*"
```

**Why:** This policy directly enforces that all workloads must pass through the 9-stage GitLab CI/CD security pipeline before deployment. Images from Docker Hub, GitHub Container Registry, or any external source are blocked at the admission controller level.

This closes the loop between the CI/CD pipeline and runtime — a container cannot run in this cluster unless it was built, scanned, and pushed through the internal pipeline.

---

## Observability Stack

kube-prometheus-stack deployed via Helm in `monitoring` namespace.

<pre>
┌─────────────────────────────────────────────────────────────┐
│  Prometheus (retention: 7d)                                 │
│    ← scrapes: node-exporter, kube-state-metrics,            │
│               kubelet, cAdvisor, AlertManager               │
│                                                             │
│  Grafana (NodePort: 30080)                                  │
│    ← dashboards: cluster health, node metrics,              │
│                  pod/container resource usage               │
│                                                             │
│  AlertManager                                               │
│    ← routes alerts from Prometheus rules                    │
└─────────────────────────────────────────────────────────────┘
</pre>

---

## GitLab CI/CD Integration

This cluster is the deployment target for the [gitlab-devsecops-pipeline](https://github.com/TengkuRizal/gitlab-devsecops-pipeline).

<pre>
GitLab Pipeline (9 stages)
         │
         │ Stage 8: deploy
         ▼
kubectl apply -f k8s/
         │
         ▼
Kyverno Admission Controller
  ├── disallow-latest-image-tag    ✅ pass
  ├── disallow-privileged-containers ✅ pass
  └── require-approved-registry    ✅ pass (image from 10.10.1.101:5050)
         │
         ▼
Pod scheduled → Running
         │
         ▼
Stage 9: verify
kubectl rollout status
</pre>

---

## Repository Structure

<pre>
kubernetes-homelab/
├── policies/
│   ├── disallow-latest-image-tag.yaml      # Block :latest images
│   ├── disallow-privileged-containers.yaml  # Block privileged containers
│   └── require-approved-registry.yaml       # Supply chain enforcement
├── monitoring/
│   └── kps-values.yaml                     # kube-prometheus-stack Helm values
├── manifests/
│   └── demo-nginx-ingress.yaml             # NGINX Ingress for demo workload
└── README.md
</pre>

---

## Quick Commands

```bash
# Cluster status
kubectl get nodes -o wide
kubectl get pods -A

# Kyverno policies
kubectl get clusterpolicies

# Test policy enforcement (should be blocked)
kubectl run test --image=nginx:latest -n demo
# Error: Images must use an explicit non-latest tag.

# Monitoring
kubectl get pods -n monitoring
# Access Grafana: http://10.10.1.70:30080
```

---

## Related Projects

| Project | Description |
|---|---|
| [gitlab-devsecops-pipeline](https://github.com/TengkuRizal/gitlab-devsecops-pipeline) | 9-stage CI/CD pipeline that deploys to this cluster |
| [wazuh-siem-lab](https://github.com/TengkuRizal/wazuh-siem-lab) | SIEM monitoring agents running on cluster nodes |
| [devsecops-homelab](https://github.com/TengkuRizal/devsecops-homelab) | Full homelab architecture |
| [terraform-aws-devsecops](https://github.com/TengkuRizal/terraform-aws-devsecops) | AWS IaC with Checkov scanning |

---

## Author

**Tengku Rizal** — DevSecOps Engineer
Building: GitLab CI/CD · Kubernetes · Wazuh SIEM · Terraform · Security Automation
Location: Kuala Lumpur, Malaysia
