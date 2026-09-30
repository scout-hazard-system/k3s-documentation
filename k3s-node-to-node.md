# Imagoro k3s context — how k3s node-to-node works (and where the code is)

The scout lab runs a **two-node k3s** cluster whose nodes are, first and
foremost, WireGuard peers. Every k3s control/data-path socket rides the mesh
(`scout0`, **10.44.0.0/24**) instead of a LAN or the internet, and the only
services exposing themselves to the outside of k3s — the NodePorts for the
blackboard/map shards and the lab Vaultwarden — are firewalled back to that
same CIDR by nftables.

This note walks node-to-node mechanics: join, API, pod network, workload
placement, NodePort reachability, and where each piece is implemented.

Implementation lives in the **`scout-k3s`** package in
[scout-trixie](https://github.com/scout-hazard-system/scout-trixie). File
paths below are relative to
`packages/scout-k3s/` in that repository.

---

## 1. Topology

```
                      10.44.0.0/24  (WireGuard mesh, scout0)
   ┌─────────────────────────┼─────────────────────────┐
   │                         │                         │
   HUB  scout-mesh server    │                    scout-mesh join
   ======================    │                    ===================
   k3s server node           │                    k3s agent node
   10.44.0.1                 │                    10.44.0.2
   ├─ kube-apiserver :6443 ──┼────────────────────────► agent
   ├─ k3s (containerd)       ┼─ flannel VXLAN ────────► k3s agent
   ├─ NodePort 30080/30180   ┼─ NodePort (kube-proxy) ─► workload pod
   └─ scout-shards pods      │                    scout-shards pods (opt)
```

* server = the k3s control plane node (single control-plane, no HA yet).
* agent = the second node, joins via the server's node-token over the mesh.
* Both carry the same `scout-role` label vocabulary (`server` / `agent`);
  placement presets pin the shard workloads to whichever role you choose.

## 2. How a node joins (server + agent)

k3s installs as a single static binary, started through systemd on the
target machines (`scout-k3s install` just wraps the official installer).

**Server** — binds *everything* to the node's WireGuard IP:

```sh
curl -sfL https://get.k3s.io | \
    INSTALL_K3S_EXEC="server \
        --node-ip 10.44.0.1 \
        --bind-address 10.44.0.1 \
        --advertise-address 10.44.0.1 \
        --tls-san 10.44.0.1 \
        --flannel-iface scout0 \
        --disable traefik,servicelb \
        --node-label scout-role=server" sh -
```

* `--flannel-iface scout0` makes the **pod network ride the WireGuard NIC**,
  not the default route — this is what keeps pod-to-pod traffic inside the
  mesh.
* `--node-ip` tells kubelet which IP to register, so `kubectl get nodes`
  shows 10.44.0.x, reachable from anywhere on the mesh.
* The join credential lives at
  `/var/lib/rancher/k3s/server/node-token` and is printed by the CLI.

Code: `usr/sbin/scout-k3s` → `install_k3s_server()`.

**Agent** — installs k3s-agent pointed at the server over the mesh:

```sh
curl -sfL https://get.k3s.io | \
    K3S_URL="https://10.44.0.1:6443" K3S_TOKEN="<node-token>" \
    INSTALL_K3S_EXEC="agent \
        --node-ip 10.44.0.2 \
        --flannel-iface scout0 \
        --node-label scout-role=agent" sh -
```

The agent keeps a long-lived connection to the server (kubelet → apiserver
on 6443, plus the flannel/VXLAN tunnel agents use to reach each other).
No inbound port is required on the agent — it dials out to the hub.

Code: `usr/sbin/scout-k3s` → `install_k3s_agent()`;
default addresses/ports in `etc/scout/k3s/k3s.conf`
(`SCOUT_K3S_SERVER_WG_IP`, `SCOUT_K3S_SERVER_PORT`, `SCOUT_K3S_NAMESPACE`).

## 3. Node-to-node data path: flannel over WireGuard

k3s ships flannel with a **VXLAN** backend by default. With
`--flannel-iface scout0` the VXLAN inner traffic is encapsulated **inside**
the WireGuard tunnel, so:

```
pod [10.42.x.y]  →  flannel VXLAN (VTEP on scout0)  →  WireGuard (10.44.0.x)
                  →  peer node's wg tunnel  →  flannel VXLAN  →  pod [10.42.x.z]
```

* Pod CIDRs (`10.42.0.0/16`) overlap mesh usage zero — they are nested
  inside the encrypted wg tunnel.
* Agent nodes carry the same flannel agent and the same wg endpoint, so pod
  traffic is symmetric node-to-node.
* No flannel/lan untrust: the physical underlay never sees pod packets
  unencapsulated; wg keeps them in the mesh.

## 4. Workload placement (which node runs what)

The sandbox (sharded blackboard + map) is declared in
`etc/scout/k3s/presets/sandbox/`:

| file | purpose |
|------|---------|
| `10-namespace.yaml` | `scout-shards` namespace |
| `20-blackboard-shard.yaml`, `21-map-shard.yaml` | Deployments (blackboard :8765 shard, map :18080 shard) |
| `30-nodeport-services.yaml` | NodePorts 30080 / 30180 |
| `40-placement-server.yaml`, `41-placement-agent.yaml` | placement presets: pin shards to the `scout-role` node label |
| `50-shard-data-pvc.yaml` | shared 2 Gi PVC for db/logs |

`scout-k3s sandbox deploy server|agent` applies the namespace + PVC + shards
+ services, then **patches a `nodeSelector: scout-role=<role>`** onto the two
Deployments so the pods land on the chosen role node (default: the data /
server node). The local `build-shards.sh` turns the installed `scout-server`
payload into `scout-shard/blackboard` / `scout-shard/map` images and imports
them into the node's `k3s ctr` store (`imagePullPolicy: Never`, no
registry), so node-to-node image distribution is a non-issue at this scale.

Code: `usr/sbin/scout-k3s` → `sandbox_deploy()`; images in
`usr/lib/scout-k3s/images/`.

## 5. How clients reach a shard (NodePort over the mesh)

Headless clients (agents) do **not** go DNS/Service-discovery-first; they
call a NodePort on the server's wg IP. kube-proxy on the node holding the
pod DNATs the request into the pod:

```
scout agent  →  http://10.44.0.1:30080 (NodePort)  →  pod:8765 blackboard shard
scout agent  →  http://10.44.0.1:30180 (NodePort)  →  pod:18080 map shard
```

That endpoint surface is exported to agents by
`etc/scout/k3s/agent/env.sh`:

```sh
SCOUT_SANDBOX_BLACKBOARD_URL=http://10.44.0.1:30080
SCOUT_SANDBOX_MAP_URL=http://10.44.0.1:30180
```

`scout-k3s env` prints the same block for `source`-ing.

## 6. Who is allowed to talk to the cluster (nftables)

Node-to-node is allowed **only if both ends are in the mesh**. The shipped
template `etc/scout/k3s/nft-wg-only.template` renders to
`/etc/nftables.d/scout-k3s.nft` and, for the ports the cluster exposes
(kube-apiserver 6443, NodePorts 30080/30180, Vaultwarden 8080), drops any
source not in `10.44.0.0/24`:

```nft
iifname != "scout0" tcp dport { 6443, 30080, 30180, 8080 } drop
```

`scout-k3s firewall apply` renders the template with the values from
`k3s.conf` and loads it with nft. UDP 51820 (the WireGuard listen port) stays
open by design — it must, so remote peers can still initiate the mesh.

## 7. Vaultwarden: off the cluster by design

The lab Vaultwarden is **not** a k3s pod. It runs as a standalone systemd
service (`presets/vaultwarden/vaultwarden.service`) bound to a wg IP via
`ROCKET_ADDRESS`, so the Vault bit is a mesh service, not a cluster service —
two different trust domains that share only the overlay their traffic
traverses. admin token handling + rotation:
`scout-k3s vaultwarden setup|reset-token`.

## References — code pointers (scout-trixie, `packages/scout-k3s/`)

| concern | file |
|---------|------|
| CLI / orchestration (install, deploy, firewall, env, status) | `usr/sbin/scout-k3s` |
| central config (IPs, ports, image refs, namespace) | `etc/scout/k3s/k3s.conf` |
| node-to-node firewall template | `etc/scout/k3s/nft-wg-only.template` |
| agent endpoint env | `etc/scout/k3s/agent/env.sh` |
| shard manifests + placement | `etc/scout/k3s/presets/sandbox/*` |
| vaultwarden unit + env (off-stack) | `etc/scout/k3s/presets/vaultwarden/*` |
| local image build/import | `usr/lib/scout-k3s/images/{blackboard,map}.Dockerfile`, `build-shards.sh` |
| package README (full config file map) | `usr/share/doc/scout-k3s/README.md` |

## Status / caveats

* Verified in the Debian-Trixie WSL build env: the deb installs, the CLI
  smoke-tests, and `scout-k3s env` prints the correct mesh endpoints.
* Real two-node bring-up (k3s + flannel over wg under systemd) is validated
  on the **target hardware**, not WSL — the WSL distro is a systemd-less
  debootstrap, so it is not a faithful systemd/k3s host.
* Order matters on a fresh lab: `scout-mesh` hub/join **first** (creates
  `scout0`), then `scout-k3s install --role server|agent`, then
  `sandbox deploy`, then `vaultwarden setup`.
