# k3s-documentation

Living documentation for the **Imagoro / scout lab k3s** setup — how the k3s
cluster hangs together on the WireGuard mesh, node to node.

The first and reference piece is `k3s-node-to-node.md`, which explains the
mesh-only two-node k3s topology (server + agent on `scout0`, 10.44.0.0/24),
how a node joins, how pod/NodePort traffic crosses nodes, and where the
related implementation lives in the
[scout-trixie](https://github.com/scout-hazard-system/scout-trixie) package
tree.

All deployments, placement presets, firewall template and env wiring are
owned by the `scout-k3s` package; this repo only explains and links them.

## Index

| doc | what it covers |
|-----|----------------|
| [k3s-node-to-node.md](./k3s-node-to-node.md) | k3s node-to-node mechanics on the WireGuard mesh + pointers into the code (Imagoro k3s context) |

## Related

- `scout-hazard-system/scout-trixie` — the package source that implements it
  (usr/sbin/scout-k3s, etc/scout/k3s, shard Dockerfiles)
- `scout-hazard-system/scout_crew` / `secure-mesh-navigation` — upstream
  payloads (blackboard/map) and the mesh that carries the cluster