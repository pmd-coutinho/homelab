# Tailscale

## Where it lives

Tailscale runs on the **Pi-hole VM** (`192.168.50.2`, Alpine Linux), **not** in the Kubernetes cluster. It advertises `192.168.50.0/24` as a subnet route, so a Tailscale client anywhere in the world can reach the whole homelab LAN.

## Why not in the cluster

A previous iteration ran Tailscale as a `hostNetwork: true` subnet router inside K8s. It worked until it didn't — the `NET_ADMIN` capability plus `TS_ROUTES` advertising the node's own subnet corrupted kubelet's routing on whichever node the pod landed on, which cascaded into a cluster-wide outage (etcd unreachable, Longhorn volumes went faulted).

Userspace mode inside the cluster is safe (no node-level routing changes) but **can't actually advertise subnet routes** — the pod just becomes a regular Tailscale node. "It's connected" in `tailscale status` is not the same as "subnet routing works."

Running it on the Pi-hole VM gives it real kernel networking (so subnet routes work), keeps it completely out of the cluster's blast radius, and means Tailscale survives every cluster rebuild.

## Install layout on Pi-hole

- Binaries: `/usr/bin/tailscale` and `/usr/sbin/tailscaled` — **manual static binaries from `pkgs.tailscale.com`**, not the Alpine `apk` package (the apk version lags significantly behind upstream)
- OpenRC init script: `/etc/init.d/tailscale` (manually created — the apk package's init script was removed along with the old binaries)
- State: `/var/lib/tailscale/tailscaled.state` (persists across binary swaps; upgrades don't require re-authenticating)
- IP forwarding: `/etc/sysctl.d/99-tailscale.conf` sets `net.ipv4.ip_forward=1` and the IPv6 equivalent

## Fresh install (e.g., rebuilding the Pi-hole VM)

```bash
ssh root@192.168.50.2
```

```bash
# Enable IP forwarding
cat > /etc/sysctl.d/99-tailscale.conf <<EOF
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF
sysctl -p /etc/sysctl.d/99-tailscale.conf

# Download and install static binaries
VERSION=1.96.4  # bump to current stable
cd /tmp
wget https://pkgs.tailscale.com/stable/tailscale_${VERSION}_amd64.tgz
wget https://pkgs.tailscale.com/stable/tailscale_${VERSION}_amd64.tgz.sha256
sha256sum -c tailscale_${VERSION}_amd64.tgz.sha256
tar -xzf tailscale_${VERSION}_amd64.tgz
cp tailscale_${VERSION}_amd64/tailscale /usr/bin/tailscale
cp tailscale_${VERSION}_amd64/tailscaled /usr/sbin/tailscaled
chmod +x /usr/bin/tailscale /usr/sbin/tailscaled

# Create OpenRC init script
cat > /etc/init.d/tailscale <<'EOF'
#!/sbin/openrc-run

command="/usr/sbin/tailscaled"
command_args="--state=/var/lib/tailscale/tailscaled.state --socket=/var/run/tailscale/tailscaled.sock --port=${PORT:-0}"
command_background="yes"
pidfile="/run/${RC_SVCNAME}.pid"
output_log="/var/log/tailscaled.log"
error_log="/var/log/tailscaled.log"

depend() {
    need net
    use dns
    after firewall
}

start_pre() {
    checkpath --directory --mode 0755 /var/lib/tailscale
    checkpath --directory --mode 0755 /var/run/tailscale
    checkpath --file --mode 0644 /var/log/tailscaled.log
}
EOF
chmod +x /etc/init.d/tailscale
rc-update add tailscale default
rc-service tailscale start

# Authenticate and advertise the homelab subnet
tailscale up --advertise-routes=192.168.50.0/24 --hostname=homelab-subnet-router --accept-dns=false
```

`tailscale up` will print a `https://login.tailscale.com/a/...` URL. Click it to link the machine to your tailnet.

After authenticating, **approve the advertised route** in the Tailscale admin console: <https://login.tailscale.com/admin/machines> → `homelab-subnet-router` → Edit route settings → enable `192.168.50.0/24`. Without this approval, the route is advertised but clients won't use it.

## Upgrading

New Tailscale release? Swap the binaries:

```bash
ssh root@192.168.50.2
VERSION=<new-version>
cd /tmp
wget https://pkgs.tailscale.com/stable/tailscale_${VERSION}_amd64.tgz
wget https://pkgs.tailscale.com/stable/tailscale_${VERSION}_amd64.tgz.sha256
sha256sum -c tailscale_${VERSION}_amd64.tgz.sha256
tar -xzf tailscale_${VERSION}_amd64.tgz
rc-service tailscale stop
cp tailscale_${VERSION}_amd64/tailscale /usr/bin/tailscale
cp tailscale_${VERSION}_amd64/tailscaled /usr/sbin/tailscaled
chmod +x /usr/bin/tailscale /usr/sbin/tailscaled
rc-service tailscale start
tailscale version
```

State persists, so no re-authentication.

Do not skip the `sha256sum -c` step. These binaries run as root on the Pi-hole VM.

**Do not `apk add tailscale`** — it will install the old 1.76.x series and either overwrite the manual binaries or create a confused mixed state. The apk package was deliberately removed.

## Health checks

```bash
ssh root@192.168.50.2 'tailscale status'        # should show the node as online
ssh root@192.168.50.2 'tailscale version'       # confirm the running binary version

# From a Tailscale client off-network:
ping 192.168.50.100   # Talos CP — should succeed if subnet route is approved
```
