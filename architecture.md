# Architecture

## Physical layout

```
 Windows 11 host (Hyper-V, 32 GB RAM) — internal switch + NAT, gateway .1
 ┌───────────────────────────────────────────────────────────────────────────┐
 │  ceph-node1 (.11)   ceph-node2 (.12)   ceph-node3 (.13)   k3s-node (.21)  │
 │  MON+MGR+RGW        MON                MON                k3s (single)    │
 │  OSD ×2 (sdb,sdc)   OSD ×2             OSD ×2             Loki/Promtail/  │
 │                                                           Grafana/Prom    │
 └───────────────────────────────────────────────────────────────────────────┘
   Ceph = external cluster (cephadm)      K8s = consumer (disaggregated)
```

| Node | IP | Role |
|---|---|---|
| ceph-node1 | 192.168.100.11 | MON - MGR - **RGW :80** - 2 OSD |
| ceph-node2 | 192.168.100.12 | MON - 2 OSD |
| ceph-node3 | 192.168.100.13 | MON - 2 OSD |
| k3s-node | 192.168.100.21 | k3s - Loki - Promtail - Grafana - Prometheus |

Ceph: **6 OSD / 3 host**, BlueStore, replicated **size=3 / min_size=2**, CRUSH rule `chooseleaf firstn 0 type host`.

## Data flow (a log line → disk)

```
 [0] pod: echo "log line N"                → stdout
 [1] containerd → /var/log/pods/.../0.log  (temporary on the node)
 [2] Promtail (DaemonSet): tail + label {pod,ns} → HTTP push
 [3] Loki: gather stream → compress into CHUNK + index
 [4] Loki S3 client → PUT → RGW :80        (SigV4, path-style)
 [5] RGW → RADOS object in pool default.rgw.buckets.data
        object ─hash→ PG ─CRUSH(type host)→ [osd.a, osd.b, osd.c]
 [6] OSD (BlueStore) → raw disk, 3 replicas on 3 hosts
```

Read/metrics path:
```
 Grafana ─LogQL→ Loki → chunks from S3 (RGW→RADOS→OSD)
 Grafana ─PromQL→ Prometheus → scrape Ceph :9283/:9926/:9100 + K8s
```

## Architecture decision making

| Decision | Reason | Rejected alternative |
|---|---|---|
| External Ceph (not Rook) | isolate storage/compute; mirrors OpenStack | Rook: hides Ceph behind CRDs, shares resources |
| RGW / S3 backend (not PVC) | Loki is built for object stores; uses the object layer | PVC/RBD: different pattern, skips RGW |
| replicated size=3 (not EC) | only 3 hosts (N+1 rule) | EC 4+2: needs ≥6 hosts to place, ≥7 to self-heal |
| Loki `replication_factor=1` | durability delegated to RADOS (pool size=3) | Loki-level replication: duplicates what RADOS does |
| k3s (not kubeadm) | lightweight; K8s is only the consumer here | kubeadm: heavier, HA not the focus |

## Nested mechanisms 

```
 Encapsulation:  log line → chunk → S3 object → RADOS object → PG → OSD → disk extent
 Identity:       {pod,ns} → stream fingerprint → chunk key → RADOS name → PG → OSD
 Durability:     Loki rf=1 ─delegates→ pool size=3 ─via→ CRUSH type host ─guarded by→ min_size + peering
 Time sync:      chrony spans Ceph (Paxos), Loki (chunk timestamps), Prometheus (scrape), k3s (certs)
 Control/data:   MON/MGR + K8s control-plane are OFF the data path; only OSD/RGW/Loki carry data
```

## Self-healing drill

```
 kill osd.x  → MON commits new map (Paxos/quorum) → CRUSH recomputes
             → PG active+undersized+degraded  (still ≥ min_size=2 → I/O continues)
             → Loki unaware; logs keep flowing
 start osd.x → peering → recovering (delta via PG log) → active+clean
```

3-host limit (N+1): the replacement copy can't be placed until osd.x returns — on ≥4 hosts Ceph would self-heal without it.
