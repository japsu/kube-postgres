# CLAUDE.md

## What this repo is

Tooling to spin up a multi-node k3s cluster on OrbStack VMs and run CloudNativePG on it. Single shared PostgreSQL cluster serving multiple projects, each with its own database and role. k3s is provisioned via cloud-init at VM boot — no Ansible or external provisioner needed.

## Sandbox setup

Claude runs under the **clank** nono profile (`nono run --profile clank --allow-cwd -- task <name>`). Key allowed paths:
- `~/.kube` — for kubectl
- `~/.orbstack` — for the `orb` CLI
- `/private/tmp/claude-501` — Helm state and cloud-init rendered templates

Helm env vars redirect all state away from `~/Library/Preferences/helm` (blocked):
- `HELM_CACHE_HOME=/private/tmp/claude-501/helm-cache`
- `HELM_CONFIG_HOME=/private/tmp/claude-501/helm-config`
- `HELM_DATA_HOME=/private/tmp/claude-501/helm-data`

All three are in Taskfile's top-level `env:` so they apply to every task. Run `task claude:install` once per sandbox session to create those dirs before running any Helm task.

## File layout

```
Taskfile.yaml                   entry point for all operations
cloud-init/
  control-plane.yaml            k3s server setup; ${K3S_TOKEN} substituted at VM creation
  worker.yaml                   k3s agent setup; ${K3S_TOKEN} and ${CP_IP} substituted at VM creation
.cluster-token                  generated once by vms:create, persisted, gitignored
manifests/
  cluster.yaml                  CNPG Cluster (3 instances, required pod anti-affinity); managed roles appended by cnpg:app:deploy
kubeconfig-orbstack.yaml        written by k3s:kubeconfig, gitignored
```

## Conventions

**Namespaces**
- CNPG operator lives in `cnpg-system`; the cluster and all Database CRDs live in `postgres`
- One namespace per application, named after the app (e.g., `myapp`)

**Secrets**
- Credential secrets are named `<app>-credentials` and carry `username` and `password` keys
- Created in the `postgres` namespace by `task credentials:create`
- `task credentials:copy` propagates them to the app namespace

**Adding a new project**
1. `task credentials:create -- <app>` — creates the k8s secret (must run first; cnpg:app:deploy checks for it)
2. `task cnpg:app:deploy -- <app>` — patches `manifests/cluster.yaml` to add the managed role, applies the cluster, then creates the Database CRD
3. `task credentials:copy -- <app>` — copies secret to the app namespace

**Connection strings from app pods**
- Read-write: `postgres-rw.postgres.svc.cluster.local:5432`
- Read-only: `postgres-ro.postgres.svc.cluster.local:5432`

## k3s cluster details

- Control plane: `k3s-cp` → `k3s-cp.orb.local` (reachable from macOS host only)
- Workers: `k3s-w1`, `k3s-w2`
- kubeconfig server: `https://k3s-cp.orb.local:6443`
- k3s kubeconfig on node: `/etc/rancher/k3s/k3s.yaml` (readable as root)
- Workers join via the CP's actual IP (from `orb list`), not the `.orb.local` hostname — `.orb.local` DNS does not resolve between VMs

## CNPG notes

- Operator installed via Helm chart `cnpg/cloudnative-pg` (latest stable)
- The `Database` CRD reports readiness via `status.applied: true` — not a `Ready` condition; check `.status.applied`, not `kubectl wait --for=condition=Ready`
- Superuser secret is not created by default in v1.25+; use managed role credentials
- `postgres-rw` service always points to the current primary; CNPG updates it automatically on failover
