# Kubernetes Lab Cluster on AWS

A self-managed Kubernetes cluster on AWS EC2 — infrastructure-as-code, Helm charts, K8s manifests, and operational runbooks. Learning/reference implementation, not production.

---

## What's Covered

| Area | Details |
|------|---------|
| AWS Infrastructure | CloudFormation (VPC, EC2, NLB, Lambda) or direct EC2 script |
| Cluster Bootstrap | Kubespray from a bastion host via Podman |
| Networking | Calico CNI (VXLAN) + Network Policy examples |
| Ingress | F5 NGINX IC (`VirtualServer` CRD) + Traefik as alternative |
| TLS | cert-manager + Let's Encrypt DNS-01 via Route53 |
| Applications | Helm chart (`hashicorp/http-echo`) |
| Observability | Jaeger all-in-one in `monitor` namespace |

---

## Topology

```
Internet → NLB (:80/:443/:6443)
               │
    ┌──────────┴──────────┐
    │    VPC 10.0.0.0/16  │
    │                     │
    │  Public 10.0.1.0/24 │
    │  ┌─────────┐        │
    │  │ Bastion │        │   SSH entry point
    │  └────┬────┘        │
    │       │             │
    │  Private 10.0.2.0/24│
    │  ┌──────────┐  ┌──────────┐
    │  │  Control │  │  Worker  │   NodePort :30080 / :30091
    │  │  Plane   │  │          │
    │  └──────────┘  └──────────┘
    └─────────────────────────────
```

- **OS:** Rocky Linux 9 on all nodes
- **CNI:** Calico VXLAN — **UDP 4789 must be open** between nodes or cross-node traffic silently drops
- **CloudFormation templates:** `AWS/cluster-infrastracture/` — three variants (basic / automated / TLS)

---

## Quick Start

```bash
# 1. Provision infrastructure
aws cloudformation create-stack \
  --stack-name k8s-lab \
  --template-body file://AWS/cluster-infrastracture/automated-template.yaml \
  --capabilities CAPABILITY_NAMED_IAM

# 2. Cluster lifecycle (after stack is up)
./AWS/cloudshell/cluster.sh start | stop | status

# 3. AWS resource inventory
./AWS/cloudshell/services.sh

# 4. Deploy the Helm app
helm upgrade --install app-deployment ./app-deployment --namespace <ns> --create-namespace
```

---

## Runbooks

| Topic | File |
|-------|------|
| Kubespray bootstrap (5-phase) | `k8s/kubespray-bastion-aws-ec2.md` |
| F5 NGINX IC installation | `ingress/nginxf5/ingress-setup.md` |
| Ingress path rewriting & redirects | `ingress/ingress-routing.md` |
| Traefik installation guide | `ingress/traefik/traefik-via-helm.md` |
| Ingress debugging (9-step checklist) | `ingress/traefik/troubleshooting-ingress.md` |
| TLS setup (cert-manager + DNS-01) | `TLS-nginxF5-ingress/openssl3 - k8s TLS setup.md` |
| Debugging pods, taints, OOM | `k8s/helper-scripts/k8s-debugging.md` |
| Adding / removing worker nodes | `k8s/helper-scripts/scaling.md` |
| Inter-node connectivity probe | `k8s/helper-scripts/cluster-connectvity.sh` |
| Calico Whisker debugging | `k8s/helper-scripts/whisker-debug.md` |
| Node memory pressure & eviction | `k8s/memory-pressure.md` |
| Direct API server access via curl | `k8s/k8s-API.md` |
| SSH to private nodes via bastion | `network/node connect.md` |
| kubelet DNS failures | `network/kubelet DNS error.md` |
| OpenSSL PKI & TLS analysis | `linux/openssl-pki.md` |
| Linux / sysadmin reference | `linux/linux-commands.md` |
