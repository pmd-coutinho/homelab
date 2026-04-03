# Full Disaster Recovery Runbook

## Context

This documents every step to rebuild the entire homelab cluster from scratch — assuming total loss of both VMs but intact Proxmox host, Git repo (GitHub mirror), and Backblaze B2 backups.

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
# Clone the repo from GitHub (Forgejo is down)
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

### 2.1 Bootstrap from GitHub (Forgejo is down)
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
2. Apps (monitoring, Forgejo, Authelia, Vaultwarden, Hoarder, OtterWiki, Homepage, Gatus, Tailscale)

**Wait for infrastructure to be healthy before proceeding:** `flux get kustomizations -w`

---

## Phase 3: Restore PostgreSQL (~10 min)

PostgreSQL will start with an empty database. To restore from Barman backup:

### 3.1 Modify cluster.yaml temporarily for recovery

Replace the `bootstrap` section in `infrastructure/postgres/cluster.yaml`:
```yaml
spec:
  bootstrap:
    recovery:
      source: homelab-postgres-backup
  externalClusters:
    - name: homelab-postgres-backup
      barmanObjectStore:
        destinationPath: s3://pmd-homelab-backups/postgres
        endpointURL: https://s3.eu-central-003.backblazeb2.com
        s3Credentials:
          accessKeyId:
            name: pg-backup-secret
            key: ACCESS_KEY_ID
          secretAccessKey:
            name: pg-backup-secret
            key: ACCESS_SECRET_KEY
        wal:
          compression: gzip
```

### 3.2 Apply and wait
```bash
kubectl apply -f infrastructure/postgres/cluster.yaml
# Watch recovery progress
kubectl logs -n postgres -l cnpg.io/cluster=homelab-postgres -f
```

### 3.3 After recovery completes
Revert `cluster.yaml` back to the normal `initdb` bootstrap (so Flux doesn't keep trying to recover).

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

### 5.1 Switch Flux source back to Forgejo
Once Forgejo is healthy:
```bash
flux create source git homelab \
  --url=ssh://git@forgejo-ssh.pedrocoutinho.eu/pmd-coutinho/homelab.git \
  --branch=main \
  --secret-ref=forgejo-auth
```

### 5.2 Tailscale
Generate a new auth key at https://login.tailscale.com/admin/settings/keys.
Update the SOPS secret and restart Tailscale pod.
Approve subnet router in Tailscale admin console.

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
| Tailscale identity | No | Re-register with new auth key |
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
