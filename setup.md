# Setup

## Ceph (cephadm)

```bash
cephadm bootstrap --mon-ip <CEPH_NODE1_IP>
ceph orch host add ceph-node2 <IP>; ceph orch host add ceph-node3 <IP>
ceph orch apply mon 3
ceph orch apply osd --all-available-devices
ceph -s        # HEALTH_OK · mon quorum 3 · 6 osd up/in
```

Requirements per node: Ubuntu 22.04, Podman/Docker, **chrony** (time sync — mandatory for MON quorum), one spare disk per OSD.

## k3s

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -
mkdir -p ~/.kube && sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
kubectl get nodes
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Log Lake

**1. RGW + S3 user + bucket** (on a Ceph node)
```bash
ceph orch apply rgw loglake --placement="1 ceph-node1" --port=80
radosgw-admin user create --uid=loki --access-key=<ACCESS_KEY> --secret-key=<SECRET_KEY>
aws --endpoint-url http://<CEPH_NODE_IP>:80 s3 mb s3://loki-data
```

**2. Loki + Promtail** (on k3s)
```bash
helm repo add grafana https://grafana.github.io/helm-charts && helm repo update
kubectl create ns loki
helm install loki     grafana/loki     -n loki -f manifests/loki-values.yaml
helm install promtail grafana/promtail -n loki -f manifests/promtail-values.yaml
```

**3. Log source**
```bash
kubectl apply -f manifests/logger-pod.yaml
```

**4. Prometheus + Grafana**
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
kubectl create ns monitoring
helm install kps prometheus-community/kube-prometheus-stack -n monitoring -f manifests/kps-values.yaml
# Grafana: http://<K3S_IP>:30300
```
