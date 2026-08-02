# Log Lake on Ceph

Kubernetes (k3s) log pipeline backed by an external **Ceph** cluster.
Pod logs → **Promtail → Loki → RGW (S3) → RADOS → OSD** (3× replicated),
observed on one **Grafana** pane with Ceph + K8s metrics, and surviving an OSD failure.

```
 k3s-node (.21)                         Ceph cluster (3 nodes, 6 OSD)
 ┌──────────────────────────┐          ┌─────────────────────────────┐
 │ pod ─stdout→ Promtail     │          │ RGW :80 (S3)                │
 │  (DaemonSet) → Loki       │──S3 PUT─►│   ▼ librados                │
 │ Grafana ←Loki / Prometheus│          │ pool default.rgw.buckets.data│
 │           └─ scrape :9283 ─┼─────────►│   object→PG→CRUSH→OSD ×3    │
 └──────────────────────────┘          │ MON ×3 (quorum) · MGR        │
                                        └─────────────────────────────┘
```

## Stack

| | |
|---|---|
| Ceph | Tentacle 20.2.2 · 6 OSD / 3 host · replicated size=3, min_size=2 |
| Kubernetes | k3s v1.36.2+k3s1 |
| Logs | Loki 3.x (SingleBinary) · Promtail (DaemonSet) |
| Metrics | kube-prometheus-stack (Prometheus + Grafana) |

## Docs

- [Architecture](docs/architecture.md) — layout, data flow, design choices
- [Setup](docs/setup.md) — Ceph · k3s · Log Lake
- [Demo & troubleshooting](docs/demo.md) — self-healing drill

> IPs are private lab addresses. Replace `<ACCESS_KEY>` / `<SECRET_KEY>` /
> `<GRAFANA_ADMIN_PASSWORD>` in `manifests/` with your own before deploying.

## Trace a log chunk to the OSDs

```bash
OBJ=$(rados -p default.rgw.buckets.data ls | grep -i fake | head -1)
ceph osd map default.rgw.buckets.data "$OBJ"
#  -> pg 9.80 -> up/acting [5,0,4]   (3 OSDs on 3 hosts)
```
