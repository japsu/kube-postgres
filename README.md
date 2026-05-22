# kube-postgres

Local OrbStack virtual machine environment for trying out multi-node Kubernetes clusters running [CloudNativePG](https://cloudnative-pg.io/) for fault-tolerant PostgreSQL.

## Not using OrbStack?

If you are not using OrbStack, ask your pet robot to adapt it to whatever means you are using for running local VMs. It should be fairly straightforward to replace the `orb` command and `.orb.local` DNS with environment-appropriate ones. The magic is in the `cloud-init` which should be fairly universal across VM managers.

## Architecture

- **3-node k3s cluster** — 1 control plane + 2 workers (OrbStack VMs), provisioned via cloud-init at boot
- **Single shared PostgreSQL cluster** — 3 instances with required pod anti-affinity (one pod per node)
- **Per-project databases and roles** — each project gets its own PostgreSQL database and user
- **Cross-namespace connectivity** — apps connect via `postgres-rw.postgres.svc.cluster.local:5432`

## Prerequisites

```
OrbStack     https://orbstack.dev
go-task      brew install go-task
helm         brew install helm
kubectl      bundled with OrbStack / brew install kubectl
yq           brew install yq
```

## Sandbox

This setup was developed to run under a [Nono](https://github.com/always-further/nono) sandbox with the [claude-code](https://github.com/always-further/nono-packs/tree/main/claude) profile. To use the sandbox, you can prefix any `task` etc. commands as follows:

```
nono run --profile claude-code --allow-cwd --allow ~/.orbstack -- task up
```

You need not use Nono for this setup to work. However, you may need to run

```
task claude:install
```

to create some temp directories under `/private/tmp/claude-501` (or redirect those paths in `Taskfile.yaml`).

## Quick start

```bash
task up
```

`task up` runs in order: create VMs → wait for k3s → fetch kubeconfig → install CNPG operator → deploy cluster.

k3s is provisioned automatically via cloud-init when the VMs boot — no separate provisioning step needed. The kubeconfig is written to `kubeconfig-orbstack.yaml` (gitignored) and all subsequent tasks use it automatically.

## Adding a project

```bash
# 1. Generate credentials secret in the postgres namespace (must come first)
task credentials:create -- myapp

# 2. Add managed role to cluster, apply it, and create the Database CRD
task cnpg:app:deploy -- myapp

# 3. Copy credentials to the app's own namespace
task credentials:copy -- myapp
```

`cnpg:app:deploy` patches `manifests/cluster.yaml` to add the PostgreSQL role (referencing the credentials secret), re-applies the cluster, then creates the `Database` CRD.

Apps connect to:
- writes: `postgres-rw.postgres.svc.cluster.local:5432`
- reads:  `postgres-ro.postgres.svc.cluster.local:5432`

Credential secret keys: `username`, `password`.

## Connecting locally

```bash
# Terminal 1 — keep running
task psql:port-forward

# Terminal 2
task psql:connect -- myapp
```

## Task reference

| Task | Description |
|------|-------------|
| `task up` | Full cluster setup from scratch |
| `task down` | Tear down all VMs (data will be lost) |
| `task vms:create` | Create the 3 OrbStack VMs (skips existing) |
| `task vms:delete` | Delete the OrbStack VMs |
| `task k3s:kubeconfig` | Fetch kubeconfig from control plane |
| `task cnpg:install` | Install CNPG operator via Helm |
| `task cnpg:deploy` | Apply the CNPG Cluster manifest |
| `task cnpg:app:deploy -- APP` | Create/update Database CRD for an app |
| `task credentials:create -- APP` | Generate credentials secret in `postgres` namespace |
| `task credentials:copy -- APP` | Copy credentials secret to app namespace |
| `task psql:port-forward` | Forward postgres-rw to localhost:5433 |
| `task psql:connect -- APP` | Connect as app user (requires port-forward) |
| `task claude:install` | Create sandbox tmp dirs (first-time setup) |
| `task claude:start` | Start Claude Code under the claude-code nono profile |
