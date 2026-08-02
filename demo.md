# Demo · Self-healing

Kill the OSD holding a log chunk; logs keep flowing (size=3, min_size=2); cluster recovers.

```bash
# 1. trace a chunk to its OSDs
OBJ=$(rados -p default.rgw.buckets.data ls | grep -i fake | head -1)
ceph osd map default.rgw.buckets.data "$OBJ"      # -> [5,0,4]  (primary osd.5)

# 2. kill the primary, watch it degrade (logs still flow)
ceph orch daemon stop osd.5
ceph -s                                            # HEALTH_WARN, pgs undersized+degraded
ceph osd map default.rgw.buckets.data "$OBJ"       # -> [0,4]  (still readable)

# 3. recover
ceph orch daemon start osd.5                        # peering -> recovering -> active+clean
```

On Grafana: `sum(ceph_osd_up)` 6→5, `sum(ceph_pg_degraded)` >0, health 0→1, log panel keeps streaming.

> 3-host lab: a PG stays `undersized` until osd.5 returns (N+1 rule — needs ≥4 hosts to self-heal automatically).

---

# Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `clock skew` / MON out of quorum | VM clock drift, no NTP | chrony with `makestep 1.0 -1` on every node |
| `localhost:8080 refused` | kubeconfig in wrong path | copy `k3s.yaml` to `~/.kube/config` + chown |
| `s3 ls` shows no chunks | Loki hasn't flushed | wait, or lower `chunk_idle_period` (lab) |
| `too many PGs per OSD` | many RGW pools, few OSDs | enable `pg_autoscaler` on the pools |
| `mon low on available space` | MON store grew | `podman system prune -a -f`; `ceph tell mon.<h> compact` |
