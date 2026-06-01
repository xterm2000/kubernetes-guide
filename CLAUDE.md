# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A Kubernetes lab cluster on AWS — infrastructure-as-code, K8s manifests, and operational runbooks for a self-managed K8s environment running on EC2. Learning/reference implementation, not production.

## Scope Discipline

- When asked to "show", "look at", or "explain" a file, do NOT begin editing. Wait for an explicit edit/change/fix instruction before using Edit/Write tools.
- Confirm scope before proceeding when a request could be interpreted multiple ways (e.g., which template, which namespace, which target group).

## Key Scripts

```bash
# Cluster lifecycle (invokes Lambda functions)
./AWS/cloudshell/cluster.sh start | stop | status

# AWS resource inventory (EC2, EBS, VPCs, LBs, etc. in us-east-1)
./AWS/cloudshell/services.sh
```

`cluster.sh` defaults to Lambda function names `k8s-lab-startup` / `k8s-lab-shutdown`; override with env vars `STARTUP_LAMBDA` / `SHUTDOWN_LAMBDA`. Region defaults to `us-east-1` via `AWS_DEFAULT_REGION`. `services.sh` has `us-east-1` hardcoded.

## Directory Layout

```
AWS/
  cloudshell/              cluster.sh, services.sh
  cluster-infrastracture/  CloudFormation templates + base K8s manifests

k8s/
  test-cluster/
    test-1/                Initial lab manifests (VirtualServer, TLS, Deployments)
    test-2/                Advanced: mock API, Calico Whisker, Headlamp, wildcard TLS
  ingress/
    ingress-routing.md     Path rewriting, redirects, and routing patterns
    nginxf5/               F5 NGINX IC files (ingress-setup.md, nginx-install.sh)
    traefik/               Traefik files (traefik-values.yaml, route.yaml, guides)
  TLS-nginxF5-ingress/     TLS setup guide + CloudFormation HTTPS snippets
  tracing/                 Jaeger all-in-one distributed tracing manifests
  helper-scripts/          Debugging, scaling, connectivity check scripts
  *.md                     K8s operational runbooks

linux/                     Linux/sysadmin runbooks and reference (nodes.env lives here)
network/                   iptables, DNS, SSH config references
```

## Infrastructure Architecture

**AWS Layer — `AWS/cluster-infrastracture/`**

| Template | Lambda Automation | HTTPS TG (30091) | Use For |
|----------|------------------|------------------|---------|
| `automated-template.yaml` | Yes | No | Standard lab + cost control |
| `automated-template-for-tls.yaml` | Yes | Yes | Lab with TLS at the IC |
| `basic-template.yml` | No | No | Minimal, no Lambda scheduling |

- Topology: 1 Bastion (public subnet, t3.small) + 1 Control Plane (private, t3.medium) + 1 Worker (private, t3.small)
- VPC: `10.0.0.0/16`, public `10.0.1.0/24`, private `10.0.2.0/24`
- NLB: `:80/:443` → Worker NodePort 30080/30091, `:6443` → Control Plane (kubectl)
- Rocky Linux 9 on all nodes; bootstrapped via Kubespray from the bastion host using Podman

**Kubernetes Layer**
- CNI: Calico with VXLAN — AWS Security Groups **must allow UDP 4789** between nodes or cross-node pod traffic silently drops
- Ingress: F5 NGINX Ingress Controller using `VirtualServer` CRD (`k8s.nginx.org/v1`)
- Traefik is demonstrated as an alternative — see `k8s/ingress/traefik/`
- Node IPs: defined in `linux/nodes.env` (source it before running cluster scripts)

## Networking / Ingress Notes

- Calico VXLAN requires UDP 4789 open in the node Security Group — missing this causes silent cross-node packet loss
- `k8s/helper-scripts/cluster-connectvity.sh` deploys a curl-probe DaemonSet and runs a full inter-node connectivity matrix
- F5 NGINX IC **requires** a `host:` field in every `VirtualServer` — omitting it causes rejection
- F5 NGINX IC rejects a standard `Ingress` if a `VirtualServer` already owns that hostname — this is why `k8s/test-cluster/test-1/ingress.yaml` and `k8s/ingress/nginxf5/ingress.yaml` are kept for reference only, not applied
- F5 NGINX IC (OSS) does **not** support `ExternalName` Service upstreams — that requires NGINX Plus (relevant to `k8s/tracing/jaeger-proxy.yaml`)
- CloudFormation target group `Port` is immutable — rename the resource logical ID when changing a NodePort to avoid `AlreadyExists` errors
- Path rewriting and redirect patterns: `k8s/ingress/ingress-routing.md`
- Troubleshooting checklist (9-step + symptom→cause table): `k8s/ingress/traefik/troubleshooting-ingress.md`

## `k8s/test-cluster/` Layout

### test-1 — Initial cluster setup

| File | Purpose |
|------|---------|
| `vs.yaml` | **Active** — VirtualServer for `api.mydomain.com` (F5 NGINX IC) |
| `vs-root.yaml` | **Active** — VirtualServer for bare `mydomain.com` (301 → api subdomain) |
| `cluster-issuer.yaml` | cert-manager ClusterIssuer `letsencrypt-prod` (Route53 DNS01) |
| `certificate.yaml` | Certificate for `api.mydomain.com` + `mydomain.com` → Secret `myapp-tls` |
| `deploy.yaml` / `hashi.yaml` | Application Deployments |
| `svc.yaml` / `hashi-svc.yaml` | Services |
| `ingress.yaml` | **Not used** — standard Ingress rejected by F5 NGINX IC when a VirtualServer owns the hostname |

### test-2 — Advanced setup (mock API, Calico Whisker, Headlamp, wildcard TLS)

| File | Purpose |
|------|---------|
| `vs-mock-api.yaml` | VirtualServer for `api.mydomain.com` with multi-service routing |
| `vs-mock-dev.yaml` | VirtualServer for `dev.mydomain.com` |
| `vs-whisker.yaml` | VirtualServer for `whisker.mydomain.com` → Calico Whisker UI (calico-system ns) |
| `vs-mock-calico.yaml` / `vs-mock-calico2.yaml` | VirtualServerRoute for `/whisker` and `/whisker-backend` sub-paths |
| `vs-mock-monitor.yaml` | VirtualServerRoute for `/headlamp` (monitor ns, Headlamp K8s dashboard) |
| `calico-policy.yaml` | Calico NetworkPolicy examples |
| `tls/cluster-issuer.yaml` | ClusterIssuer `letsencrypt-aws` for wildcard cert |
| `tls/letsencrypt-cert.yaml` | Wildcard `*.mydomain.com` cert → Secret `route53` |

**After a fresh stack:** update `hostedZoneID`, `accessKeyID`, email in `cluster-issuer.yaml`, then recreate the Route53 credentials secret before applying TLS manifests.

## TLS

- cert-manager + Let's Encrypt DNS-01 via Route53 (HTTP-01 doesn't work with F5 NGINX IC)
- test-1: single-domain cert (`myapp-tls` secret); test-2: wildcard `*.mydomain.com` (`route53` secret)
- TLS terminates at the NGINX IC; pods receive plain HTTP internally
- Full walkthrough: `k8s/TLS-nginxF5-ingress/openssl3 - k8s TLS setup.md`
- CloudFormation + K8s snippets for the HTTPS NLB target group: `k8s/TLS-nginxF5-ingress/infra-change.yaml`

## Distributed Tracing — Jaeger (`k8s/tracing/`)

Jaeger all-in-one in the `monitor` namespace, exposed via F5 NGINX IC VirtualServer.

```bash
# Apply in order:
kubectl apply -f k8s/tracing/00-namespace.yaml
kubectl apply -f k8s/tracing/jaeger-collector.yaml   # UI :16686, OTLP gRPC :4317, HTTP :4318
kubectl apply -f k8s/tracing/jaeger-proxy.yaml       # ExternalName bridge (NGINX Plus only)
kubectl apply -f k8s/tracing/jaeger-vs.yaml          # jaeger.mydomain.com → Jaeger UI
kubectl apply -f k8s/tracing/main-vs-patched.yaml    # adds /jaeger/ route to main VirtualServer
kubectl apply -f k8s/tracing/jaeger-tracegen.yaml    # smoke-test trace generator (optional)
```

> `jaeger-proxy.yaml` (ExternalName upstream) is rejected by NGINX IC OSS. Workaround: deploy Jaeger in `default` ns, or deploy the VirtualServer inside the `monitor` namespace directly.

## Documentation Index

```
k8s/kubespray-bastion-aws-ec2.md              Kubespray bootstrap (5-phase: inventory → SSH → Ansible)
k8s/helper-scripts/k8s-debugging.md          Pod failures, taints, OOM, fix kubelet.conf
k8s/helper-scripts/scaling.sh                Add / remove worker nodes
k8s/helper-scripts/whisker-debug.md          Debugging Calico Whisker via NGINX IC
k8s/memory-pressure.md                       Eviction, OOM, resource limits
k8s/k8s-API.md                               Direct API server access via curl (extract kubeconfig certs)
k8s/k8s-cluster.md                           Bash script: create/stop/start/terminate EC2 nodes directly
k8s/kubectl-aliases.md                       Shell aliases for common kubectl commands
k8s/kubectl-autocomplete.md                  kubectl tab completion (bash/zsh)
k8s/kubectl-dry-run.md                       --dry-run usage patterns
k8s/kubernetes-dump-cluster.md               Export all cluster resources to YAML

k8s/ingress/nginxf5/ingress-setup.md         F5 NGINX IC installation checklist
k8s/ingress/traefik/traefik-via-helm.md      Traefik v3 guide: NodePort, self-signed TLS, IngressRoute, DaemonSet
k8s/ingress/traefik/troubleshooting-ingress.md  9-step ingress debugging checklist

linux/nodes.env                              Node IP map — edit before running cluster scripts
linux/passwordless-login.md                  SSH key distribution across nodes
linux/reboot-machines.md                     Mass SSH reboot script (sources nodes.env)
linux/openssl-pki.md                         Full PKI walkthrough + OpenSSL reference + TLS analysis
linux/ssl-server-key-checks.md               Inspect, parse, and validate X.509 certs

network/node connect.md                      SSH to private nodes via bastion ProxyJump
network/kubelet DNS error.md                 kubelet DNS resolution failures
```
