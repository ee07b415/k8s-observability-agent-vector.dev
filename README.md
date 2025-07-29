# Vector Metrics Agent

A simple Helm chart for collecting Kubernetes metrics using Vector. Automatically sets up RBAC and collects node, kubelet, and container metrics.
This helm chart is designed to simply the setting to deploy the metrics agent using vector.dev in k8s environment, limit the use case to trade for
simplicity. 

## Prerequisites

In order to meet the SOC 2 complaince, the TLS is a required field please install the cert manager before install this chart
```bash
# Install cert-manager in your cluster (one-time setup)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Create a ClusterIssuer (example with Let's Encrypt)
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@yourcompany.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

## Quick Install

```bash
helm install metrics-agent oci://your-registry.com/charts/vector-metrics-agent \
  --set cluster.name="my-cluster" \
  --set client.id="my-client-001" \
  --set client.username="metrics-user" \
  --set client.password="your-password" \
  --set metrics.server.endpoint="https://your-metrics-server.com/api/v1/write"
```

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Node                          │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   Kubelet    │  │ Node Exporter│  │   cAdvisor   │       │
│  │   :10250     │  │    :9100     │  │   :10250     │       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
│         │                 │                 │               │
│         └─────────────────┼─────────────────┘               │
│                           │                                 │
│  ┌────────────────────────▼────────────────────────────┐    │
│  │              Vector Agent (DaemonSet)               │    │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐ │    │
│  │  │  Collector  │  │ 20GB Buffer  │  │  Exporter   │ │    │
│  │  │             │  │   (Disk)     │  │   :9598     │ │    │
│  │  └─────────────┘  └──────────────┘  └─────────────┘ │    │
│  └────────────────────────┬────────────────────────────┘    │
└───────────────────────────┼─────────────────────────────────┘
                            │
              ┌─────────────▼─────────────┐
              │    Remote Metrics Server  │
              │  (Prometheus Remote Write)│
              │  (Central vector agg)     │
              └───────────────────────────┘
```

## Required Configuration

| Parameter | Description | Example |
|-----------|-------------|---------|
| `cluster.name` | Your cluster name | `"production"` |
| `client.id` | Unique client ID | `"company-prod-001"` |
| `client.username` | Metrics server username | `"metrics-user"` |
| `client.password` | Metrics server password | `"your-password"` |
| `metrics.server.endpoint` | Where to send metrics | `"https://metrics.example.com/api/v1/write"` |

## Example with Values File

```yaml
# values.yaml
cluster:
  name: "production-k8s"
client:
  id: "acme-corp-prod"
  username: "vector-user"
  password: "secure-password"
metrics:
  prom_server:
    endpoint: "https://prometheus.metrics.example.com/write"
  server:
    endpoint: "https://metrics.acme.com/api/v1/write"
```

## Authentication by secret file
```bash
kubectl create secret generic metrics-credentials \
  --from-literal=password="my-secret-password"

helm install metrics-agent . \
  --set client.passwordSecret.name="metrics-credentials" \
  --set client.passwordSecret.key="password" \
```

```yaml
# values.yaml
client:
  id: "prod-001"
  username: "metrics-user"
  passwordSecret:
    name: "metrics-credentials"
    key: "password"
```

```bash
helm install metrics-agent oci://your-registry.com/charts/vector-metrics-agent -f values.yaml
```

## What It Collects

- Node metrics (CPU, memory, disk, network)
- Kubelet metrics (pod and node resource usage)
- Container metrics (individual container stats)
- Automatically adds cluster/client labels to all metrics

## Local Development

```bash
# Clone for development
git clone <repo-url>
cd vector-metrics-agent

# Test locally
# First, update dependencies
helm dependency update

# Install
helm install metrics-agent . \
  --set cluster.name="my-cluster" \
  --set client.id="client-001" \
  --set client.username="user" \
  --set client.password="pass" \
  --set metrics.prom_server.endpoint="https://prometheus.metrics.example.com/write" \
  --set metrics.server.endpoint="https://metrics.example.com/write"

# Access metrics endpoint
kubectl port-forward daemonset/test-metrics 9598:9598

# Publish
helm package .
helm push vector-metrics-agent-0.1.0.tgz oci://your-registry.com/charts
```

## Troubleshooting

**No metrics appearing?** Check Vector logs:
```bash
kubectl logs -n default daemonset/metrics-agent
```

**Permission errors?** The chart auto-creates RBAC, ensure you have cluster admin access during install.

**Node exporter errors?** Disable if not installed:
```bash
helm upgrade metrics-agent . --set vector.scraping.nodeExporter.enabled=false
```