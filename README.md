# Vector Metrics Agent

A simple Helm chart for collecting Kubernetes metrics using Vector. Automatically sets up RBAC and collects node, kubelet, and container metrics.
This helm chart is designed to simply the setting to deploy the metrics agent using vector.dev in k8s environment, limit the use case to trade for
simplicity. 

## Quick Install

```bash
helm install metrics-agent oci://your-registry.com/charts/vector-metrics-agent \
  --set cluster.name="my-cluster" \
  --set client.id="my-client-001" \
  --set client.username="metrics-user" \
  --set client.password="your-password" \
  --set metrics.server.endpoint="https://your-metrics-server.com/api/v1/write"
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
  server:
    endpoint: "https://metrics.acme.com/api/v1/write"
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