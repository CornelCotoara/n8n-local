# Deployment Log

Short record of what broke during the first rollout and how it was fixed —
useful if you rebuild this cluster or hit the same symptoms elsewhere.

## 1. Longhorn preflight failures

`iscsi_tcp` kernel module not loaded and `cryptsetup` missing on worker
nodes. Fixed with:
```bash
sudo dnf install -y cryptsetup
echo iscsi_tcp | sudo tee /etc/modules-load.d/iscsi_tcp.conf
sudo modprobe iscsi_tcp
```
(now baked into the README's prerequisites section)

## 2. firewalld only on the master node

Only `k3s-server-01` had `firewalld` active; workers didn't. Since only
one node in the cluster enforces a firewall, the rule only needed adding
there — Longhorn's node-to-node ports:
```bash
sudo firewall-cmd --permanent --add-port=9500-9503/tcp
sudo firewall-cmd --permanent --add-port=10000-30000/tcp
sudo firewall-cmd --reload
```
Flagged as a homelab hardening gap for later: an inconsistent firewall
posture across nodes (open on 2/3) isn't a Longhorn problem, but worth
revisiting.

## 3. Mass pod evictions — undersized disk on `k3s-worker-02`

Root cause: `/` on `k3s-worker-02` was only 8.9G, already at 90% used —
Longhorn's `defaultDataPath: /var/lib/longhorn` writes replica data to
that same root disk, so it filled fast and triggered `DiskPressure`,
which evicted every pod on the node in a loop.

Fix: resized the VM's disk in Proxmox, then grew the partition and
filesystem in-guest:
```bash
qm resize <VMID> scsi0 +20G       # on the Proxmox host
sudo growpart /dev/sda 4          # in the VM
sudo resize2fs /dev/sda4          # or xfs_growfs, depending on fs
```

## 4. Ghost `localhost` node in the cluster

A leftover node named `localhost` — registered before the first
server's hostname was set during initial k3s install — was still
listed in `kubectl get nodes`, `NotAMember` of etcd. Removed with:
```bash
kubectl delete node localhost
```
The `error syncing 'localhost': handler managed-etcd-controller...`
log spam during removal is expected and self-resolves once the etcd
member cleanup finishes.

## 5. n8n CrashLoopBackOff — `EACCES` on `/home/node/.n8n/config`

The official `n8nio/n8n` image runs as non-root user `node` (uid/gid
1000). The Longhorn-backed PVC mounted owned by root, so n8n couldn't
write its config. Fixed by adding a pod-level `securityContext.fsGroup:
1000` to the Deployment — this is what actually resolved it, confirmed
by the pod finally staying `Running` and port-forward connecting
cleanly instead of "connection refused".

## 6. "Secure cookie" warning blocking login over HTTP

Browsers only treat `localhost` as a secure context over plain HTTP —
a real hostname like `n8n.home.arpa` over HTTP gets its `Secure`
cookie rejected, so login silently fails even though the server
responds fine (which is why `port-forward` to `localhost:5678` had
worked all along and the ingress hostname hadn't). Fixed with:
```yaml
N8N_SECURE_COOKIE: "false"
```
in `base/n8n/configmap.yaml`. Revisit once TLS is added on the
`websecure` entrypoint — this should go back to `true` (or be removed)
at that point, alongside switching `N8N_PROTOCOL`/`WEBHOOK_URL` to
`https`.

## Current status

n8n reachable at `http://n8n.home.arpa`, backed by Postgres, both PVCs
on Longhorn (`longhorn-replicated`, 2 replicas). Working end to end.
