# Full Disaster Recovery Runbook

## Context

This documents every step to rebuild the entire homelab cluster from scratch — assuming total loss of both VMs but intact Proxmox host, the primary Forgejo repo plus GitHub mirror, and Backblaze B2 backups.

## Prerequisites (stored in Bitwarden)

You need these 3 files that are NOT in Git:
1. **`age.key`** — SOPS master decryption key (without this, all secrets are unreadable)
2. **`clusterconfig/talosconfig`** — Talos API access (regenerated during bootstrap)
3. **`kubeconfig`** — Kubernetes access (regenerated during bootstrap)

Plus these credentials:
- **Talos disk encryption passphrase** (`TALOS_DISK_PASSPHRASE`)
- **Backblaze B2 key** (in encrypted secrets, but useful to have separately)
- **Tailscale auth key** (generate new one at login.tailscale.com)

---

## Phase 1: Recreate Talos Cluster (~15 min)

### 1.1 Create VMs on Proxmox
- talos-cp: 4 vCPU, 6GB RAM, 60GB SSD (VM ID 100)
- talos-worker: 4 vCPU, 8GB RAM, 80GB SSD (VM ID 101)
- Boot from Talos ISO

### 1.2 Generate configs and bootstrap
```bash
# Clone the repo from GitHub mirror for bootstrap (Forgejo may still be down)
git clone git@github.com:pmd-coutinho/homelab.git
cd homelab

# Place age.key in repo root (from Bitwarden)
# Set env vars
export SOPS_AGE_KEY_FILE=$(pwd)/age.key
export TALOS_DISK_PASSPHRASE="..."  # from Bitwarden

# Decrypt Talos secrets and generate configs
sops --decrypt talsecret.enc.yaml > talsecret.sops.yaml
talhelper genconfig
rm talsecret.sops.yaml

export TALOSCONFIG=$(pwd)/clusterconfig/talosconfig

# Apply to VMs (use DHCP IPs from Proxmox console first)
talosctl apply-config --nodes <CP_DHCP_IP> --file clusterconfig/homelab-talos-cp.yaml --insecure
talosctl apply-config --nodes <WORKER_DHCP_IP> --file clusterconfig/homelab-talos-worker.yaml --insecure

# Bootstrap etcd (ONCE, on CP only)
talosctl bootstrap --nodes <CP_DHCP_IP>

# Get kubeconfig
talosctl kubeconfig kubeconfig --nodes 192.168.50.100 --force
export KUBECONFIG=$(pwd)/kubeconfig
```

### 1.3 Install Cilium (must be first — nodes won't be Ready without CNI)
```bash
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --namespace kube-system \
  --set ipam.mode=kubernetes \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=192.168.50.100 \
  --set k8sServicePort=6443 \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set l2announcements.enabled=true \
  --set externalIPs.enabled=true \
  --set securityContext.privileged=true \
  --set cgroup.autoMount.enabled=false \
  --set cgroup.hostRoot=/sys/fs/cgroup
```

Wait for nodes to become Ready: `kubectl get nodes -w`

---

## Phase 2: Bootstrap Flux (~5 min)

### 2.1 Bootstrap from GitHub mirror
```bash
flux bootstrap github \
  --owner=pmd-coutinho \
  --repository=homelab \
  --branch=main \
  --path=clusters/homelab \
  --personal
```

### 2.2 Create SOPS age secret for Flux decryption
```bash
cat age.key | kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin
```

### 2.3 Trigger reconciliation
```bash
flux reconcile kustomization flux-system --with-source
```

Flux will now deploy everything in dependency order:
1. Infrastructure (Longhorn, cert-manager, Traefik, PostgreSQL, Volsync, Loki, Alloy, NFS provisioner)
2. Apps (monitoring, Forgejo, Authelia, Vaultwarden, Hoarder, OtterWiki, Homepage, Gatus)

**Wait for infrastructure to be healthy before proceeding:** `flux get kustomizations -w`

### 2.4 Repoint Flux back to Forgejo after Forgejo is healthy

GitHub is only the bootstrap path here. The steady-state Flux source is internal Forgejo.

After Forgejo is up and serving the current repo state, verify or restore the live source:

```bash
kubectl -n flux-system get gitrepository flux-system -o jsonpath='{.spec.url}{"\n"}'
```

Expected value:

```text
http://forgejo-http.forgejo.svc.cluster.local:3000/pmd-coutinho/homelab.git
```

If it is not pointing there, re-apply the repo-owned Flux manifests instead of creating an ad hoc source:

```bash
kubectl apply -k clusters/homelab/flux-system
flux reconcile source git flux-system
flux reconcile kustomization flux-system --with-source
```

---

## Phase 3: Restore PostgreSQL (~10 min)

PostgreSQL will start with an empty database. To restore from Barman backup:

### 3.1 Rotate the Barman serverName before applying

`infrastructure/postgres/cluster.yaml` is already wired for recovery from Barman, but **the `serverName` values must be rotated before each recovery** to avoid WAL split-brain with the pre-disaster stream:

- `spec.externalClusters[0].barmanObjectStore.serverName` → set to the **current** stream you're recovering from (e.g., `homelab-postgres-v8`)
- `spec.backup.barmanObjectStore.serverName` → set to a **new, unused** stream name (e.g., `homelab-postgres-v9`)

If you leave both pointing at the same name, the recovered cluster will try to write WAL back into the stream it's reading from, which corrupts backups.

Optionally remove `spec.bootstrap.recovery.recoveryTarget.backupID` to recover to the **latest** point-in-time (default) instead of a specific backup.

### 3.2 Apply and wait
```bash
kubectl apply -f infrastructure/postgres/cluster.yaml
# Watch recovery progress
kubectl logs -n postgres -l cnpg.io/cluster=homelab-postgres -f
```

The bootstrap creates a one-shot `...-full-recovery` pod that downloads the base backup, replays WAL, then exits. The normal `homelab-postgres-1` primary comes up afterward.

### 3.3 After recovery completes
No manual edit needed — the rotated `serverName` values from step 3.1 are the new steady state. Commit them to git so Flux matches the live resource.

### 3.4 Recreate additional databases
The Barman backup includes all databases (forgejo, authelia, vaultwarden, hoarder). If any are missing:
```bash
kubectl exec -n postgres <primary-pod> -- psql -U postgres -c "CREATE DATABASE vaultwarden OWNER forgejo;"
kubectl exec -n postgres <primary-pod> -- psql -U postgres -c "CREATE DATABASE hoarder OWNER forgejo;"
```

---

## Phase 4: Restore PVC Data via Volsync (~15 min)

Each app's PVC needs to be restored from the Restic backup on Backblaze B2.

### 4.1 Create ReplicationDestination for each app

For each app (forgejo, grafana, vaultwarden, hoarder), create a ReplicationDestination:

```yaml
apiVersion: volsync.backube/v1alpha1
kind: ReplicationDestination
metadata:
  name: <app>-restore
  namespace: <namespace>
spec:
  trigger:
    manual: restore-once
  restic:
    repository: <app>-backup-secret  # same secret as ReplicationSource
    destinationPVC: <pvc-name>       # must match the PVC the app expects
    copyMethod: Direct
    capacity: <size>
    storageClassName: longhorn        # or nfs for hoarder
    accessModes:
      - ReadWriteOnce
```

### 4.2 Restore each app's data

| App | Namespace | PVC Name | Size | StorageClass |
|---|---|---|---|---|
| Forgejo | forgejo | gitea-shared-storage | 20Gi | longhorn |
| Grafana | monitoring | kube-prometheus-stack-grafana | 2Gi | longhorn |
| Vaultwarden | vaultwarden | vaultwarden-data | 2Gi | longhorn |
| Hoarder | hoarder | hoarder-data-nfs | 50Gi | nfs |

### 4.3 Restart apps after PVC restore
Scale down each app, wait for ReplicationDestination to complete, then scale up:
```bash
kubectl scale deploy/<app> -n <ns> --replicas=0
# Apply ReplicationDestination
# Wait for completion: kubectl get replicationdestination -n <ns> -w
kubectl scale deploy/<app> -n <ns> --replicas=1
```

---

## Phase 5: Post-Restore Manual Steps (~10 min)

### 5.1 Keep Flux source on Forgejo
After recovery, ensure Flux has moved back to the repo-owned internal Forgejo source:

```bash
kubectl -n flux-system get gitrepository flux-system -o jsonpath='{.spec.url}{"\n"}'
```

Expected value:

```text
http://forgejo-http.forgejo.svc.cluster.local:3000/pmd-coutinho/homelab.git
```

If it drifted, re-apply the repo manifests:

```bash
kubectl apply -k clusters/homelab/flux-system
flux reconcile source git flux-system
flux reconcile kustomization flux-system --with-source
```

### 5.2 Tailscale
Tailscale runs on the **Pi-hole VM** (`192.168.50.2`), not in the cluster. See [tailscale.md](tailscale.md). If the Pi-hole VM itself was lost, reinstall per that doc. A cluster rebuild does not require any Tailscale changes.

### 5.3 Pi-hole DNS
All DNS entries in `/etc/dnsmasq.d/homelab.conf` on 192.168.50.2 should still be intact (Pi-hole is on Proxmox, not the cluster). Verify:
```bash
ssh root@192.168.50.2 "cat /etc/dnsmasq.d/homelab.conf"
```

### 5.4 NFS storage
NFS server on Proxmox (192.168.50.17) should still be intact. Verify:
```bash
showmount -e 192.168.50.17
```

### 5.5 OtterWiki
OtterWiki has **no backup**. Content can be restored from the Forgejo wiki repo (bidirectional sync was configured). Pull from Forgejo after it's restored.

### 5.6 Cert-manager
Let's Encrypt certificates will be automatically re-issued via DNS-01 challenge. First issuance takes ~2 minutes.

### 5.7 Verify all services
```bash
# Check all pods are running
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded

# Check all HelmReleases are ready
flux get helmreleases -A

# Check backups resume
kubectl get replicationsource -A
```

---

## What's backed up vs. what's not

| Data | Backed Up? | Recovery Method |
|---|---|---|
| PostgreSQL (all DBs) | Yes | Barman restore from B2 |
| Forgejo repos + config | Yes | Volsync restic from B2 |
| Grafana dashboards | Yes | Volsync restic from B2 |
| Vaultwarden vault | Yes | Volsync restic from B2 |
| Hoarder bookmarks + archives | Yes | Volsync restic from B2 |
| OtterWiki content | Partial | Restore from Forgejo wiki repo |
| Prometheus metrics | No | Re-scraped (15d history lost) |
| Loki logs | No | Re-collected (7d history lost) |
| AlertManager state | No | Auto-recovers |
| Tailscale identity | N/A | Not in cluster — lives on Pi-hole VM, see docs/tailscale.md |
| Meilisearch index | No | Auto-rebuilds from Hoarder data |
| All K8s manifests + secrets | Yes | In Git (SOPS encrypted) |

## Gap: OtterWiki backup

OtterWiki currently has no Volsync backup. The wiki content is synced to Forgejo via git push, so it's recoverable from the Forgejo repo after Forgejo is restored. But adding a Volsync ReplicationSource would be safer.

---

## Estimated total recovery time: ~45-60 minutes

| Phase | Time |
|---|---|
| Create VMs + bootstrap Talos | 15 min |
| Bootstrap Flux | 5 min |
| Wait for infrastructure | 10 min |
| Restore PostgreSQL | 10 min |
| Restore PVC data | 15 min |
| Post-restore manual steps | 10 min |

---

## Lessons learned from past outages

- **VM boot order in Proxmox matters.** If the VMs boot from the Talos ISO instead of disk, they come up in maintenance mode with no hostname and no machine config, and the cluster looks completely dead. Hardware → remove or downgrade the CD-ROM device so disk is always first.
- **Don't run Tailscale as a `hostNetwork` subnet router inside K8s.** Advertising the node's own subnet with `NET_ADMIN` corrupts the node's routing table. Tailscale lives on the Pi-hole VM — see [tailscale.md](tailscale.md).
- **Memory limits matter more than requests on a constrained host.** With 16GB total RAM across both Talos VMs, the cluster is OOM-prone during mass pod restarts (e.g., after a reboot) because all pods initialise simultaneously and approach their limits at once. Keep controller-style workloads (Flux controllers, Volsync manager, etc.) at modest limits (~400Mi), not the 1Gi defaults.
- **Cilium "maintenance" backend state tracks Kubernetes EndpointSlice `ready/serving`.** If a pod isn't passing its readiness probe, Cilium marks the backend as maintenance and connections fail with `operation not permitted`. This is a symptom, not the cause — always investigate the target pod's readiness before suspecting Cilium.
