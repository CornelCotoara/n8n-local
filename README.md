# n8n AI Failover — k3s deployment

Self-hosted n8n running the "GPT/Gemini/Dahl automatic failover" workflow,
deployed on the existing k3s homelab (3x Rocky Linux 9 VMs, MetalLB, Traefik).

## Project summary

A working proof of concept, built and troubleshot end to end on real
(if small) infrastructure — not a tutorial followed to completion, but a
deployment that broke in several realistic ways and got fixed. Technologies
with direct, hands-on exposure in this project:

- **Kubernetes (k3s)** — Deployments, StatefulSets, Namespaces, ConfigMaps,
  Secrets, PVCs, security contexts (`fsGroup`), declarative management via
  **Kustomize**. Homelab-scale (3 nodes), not production-scale, debugged for running issues.
- **Distributed storage (Longhorn)** — replicated block storage as a CSI
  provisioner; diagnosed and fixed real preflight failures (missing kernel
  module, missing package) and a real node-level disk-pressure incident
  (undersized VM disk causing cluster-wide pod eviction).
- **Networking** — Traefik `IngressRoute`, MetalLB load balancing,
  `firewalld` rule management across a multi-node cluster, internal DNS via
  Pi-hole.
- **Databases** — PostgreSQL running as a stateful Kubernetes workload
  (StatefulSet + PVC), separate from the app tier.
- **Linux systems administration** — kernel module persistence, disk/
  partition resizing on Rocky Linux VMs, systemd/etcd troubleshooting
  (diagnosing and removing a stale etcd member left over from initial
  cluster bring-up).
- **AI/LLM integration** — multi-provider failover chain across
  OpenAI-compatible APIs (OpenAI, Gemini, Dahl), configured as interchangeable
  credentials in an automation platform (n8n) rather than hardcoded to one
  vendor.
- **Workflow automation (n8n)** — self-hosted, backed by Postgres, exposed
  through the cluster's own ingress rather than n8n Cloud.

Full troubleshooting narrative — what broke, root cause, fix — in
`DEPLOYMENT-LOG.md`.

## Storage decision

Postgres (n8n's state) needs storage that survives a node dying or being
reinstalled. Two options were rejected:

- **NFS** — SQLite-style `flock()` issues already hit this cluster once
  (Pi-hole). Postgres is less fragile about locking than SQLite, but NFS
  still isn't the recommended backend for a database's data directory.
- **local-path** — ties the PV to whichever single node it was created on.
  If that node is rebuilt or lost, the data is gone. Fine for stateless
  scratch data, not for the workflow database.

**Chosen: Longhorn**, replicated block storage across the 3 nodes.
See `storage/longhorn/` — a volume backed by Longhorn survives any single
node going down, because the data exists as full replicas on other nodes,
not just on the node the pod happens to run on.

## Install order

1. **Node prerequisites** (run on all 3 Rocky Linux 9 nodes):
   ```bash
   sudo dnf install -y iscsi-initiator-utils nfs-utils cryptsetup
   sudo systemctl enable --now iscsid

   # iscsid doesn't force-load the iscsi_tcp module by itself — Longhorn's
   # preflight check expects it already loaded, so load it explicitly and
   # persist it across reboots:
   echo iscsi_tcp | sudo tee /etc/modules-load.d/iscsi_tcp.conf
   sudo modprobe iscsi_tcp
   ```
   Longhorn's engine talks to its frontend over iSCSI — without `iscsid`
   running and `iscsi_tcp` loaded, volumes will fail to attach.
   `cryptsetup` is required by the CSI driver regardless of whether
   encrypted volumes are actually used. Run Longhorn's own preflight
   checker to confirm before installing:
   ```bash
   curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.7.2/scripts/environment_check.sh | bash
   ```
   If `firewalld` is active on the nodes, Longhorn also needs open
   node-to-node traffic on `9500-9503/tcp` (longhorn-manager) and
   `10000-30000/tcp` (replica sync between instance-manager pods).

2. **Longhorn**:
   ```bash
   helm repo add longhorn https://charts.longhorn.io
   helm repo update
   kubectl apply -f storage/longhorn/namespace.yaml
   helm install longhorn longhorn/longhorn \
     --namespace longhorn-system \
     -f storage/longhorn/helm-values.yaml
   kubectl apply -f storage/longhorn/storageclass.yaml
   ```
   Wait for all pods in `longhorn-system` to be Running before continuing.
   UI (optional, for visibility): port-forward `longhorn-frontend` svc.

3. **n8n + Postgres**:
   ```bash
   cp base/shared-secret.example.yaml base/shared-secret.yaml
   # generate both values the same way: openssl rand -hex 32
   # fill them into shared-secret.yaml (gitignored)
   kubectl apply -k base/
   ```
   One secret, two consumers: Postgres reads `postgres-password` as
   `POSTGRES_PASSWORD` (sets the role's password on first boot), n8n reads
   the same key as `DB_POSTGRESDB_PASSWORD` (connects with it). Keeping it
   in one Secret means there's nothing to keep in sync by hand.

4. **Import the workflow**: export the n8n workflow JSON from the UI
   (Settings → Download) into `workflows/gpt-gemini-failover.json` and
   commit it — that's your version control for the workflow itself,
   separate from the infra manifests.

## Longhorn StorageClass name

The class is called `longhorn-replicated` (not `longhorn-postgres`) — it's
used by both the Postgres PVC and n8n's own data PVC (`base/n8n/pvc.yaml`),
since n8n's encryption key and binary-data cache also need to survive a
node loss, not just the database.

## Notes

- Postgres runs as a single replica StatefulSet — Longhorn is what gives
  it resilience, not Postgres-level replication. That's a deliberate
  simplification.
- `numberOfReplicas: 2` on the Longhorn StorageClass, not 3 — with only
  3 nodes total, 2 replicas already survives one node loss while leaving
  headroom, and costs less disk than 3.
- Both PVCs (Postgres data, n8n data) use the same `longhorn-replicated`
  StorageClass — see note above.
- See `DEPLOYMENT-LOG.md` for the issues hit during the first rollout
  (Longhorn preflight, firewalld, disk sizing, a ghost node, n8n
  permissions, the secure-cookie login issue) and their fixes.


# TODO
## Provider credentials in n8n and setup workflows

Configure three credentials of type "OpenAI" inside n8n (all three are
OpenAI-protocol compatible, only the Base URL and key differ), connected
in this order to the `Fallback Models` node:

| Order | Provider | Base URL | Notes |
|---|---|---|---|
| 1 | OpenAI | `https://api.openai.com/v1` | primary |
| 2 | Google Gemini | (native Gemini node, not OpenAI-compatible) | secondary |
| 3 | Dahl | `https://inference.dahl.global/v1` | third |

