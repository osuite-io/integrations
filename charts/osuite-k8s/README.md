# osuite-k8s

Kubernetes cluster monitoring for [Osuite](https://osuite.io). The chart deploys OpenTelemetry collectors that ship cluster metrics, container logs, and Kubernetes events to your Osuite ingest endpoint over OTLP.

## What it installs

- **Cluster-wide collector** (`otel-clusterwide`, a single-replica Deployment) — always installed. Scrapes node, pod, and container metrics from the kubelet and cAdvisor, scrapes object-state metrics from kube-state-metrics, and watches cluster events through the `k8sobjects` receiver.
- **Node collector** (`otel-daemon`, a DaemonSet) — installed only when host metrics or container logs are enabled. Collects host CPU, memory, disk, filesystem, network, and load metrics, and container stdout/stderr logs with per-namespace filtering.
- **kube-state-metrics** — installed as a subchart and scraped by the cluster-wide collector.

Both collectors are plain Deployment/DaemonSet resources configured through ConfigMaps. Metrics, events, and container logs all carry the cluster name from `cluster`, so data from several clusters can be told apart.

## Prerequisites

- Kubernetes cluster with `kubectl` access and permission to create cluster-scoped RBAC.
- Helm 3.
- An Osuite ingest token and your region's ingest endpoint.

## Quickstart

```bash
helm repo add osuite https://osuite-io.github.io/integrations
helm repo update
```

Create a `values.yaml`:

```yaml
common:
  osuite_ingest_key: "<your-ingest-token>"
  osuite_ingest_endpoint: "ingest.us.osuite.io:443"

cluster: "prod-us-east"
environment: "production"

otel_daemon:
  enableHostMetrics: true
  containerlogs:
    enabled: true
```

Install into its own namespace:

```bash
helm install osuite-k8s osuite/osuite-k8s \
  --namespace osuite \
  --create-namespace \
  --values values.yaml
```

Verify the collectors are running:

```bash
kubectl get pods -n osuite
```

## Values

| Key | Description | Default |
|---|---|---|
| `common.osuite_ingest_key` | Osuite ingest token, sent as the `x-osuite-ingest-token` header. | `"change_me"` |
| `common.osuite_ingest_endpoint` | OTLP ingest endpoint for your region. | `"ingest.us.osuite.io:443"` |
| `common.insecure` | Disable TLS verification on the exporter. Keep `false` in production. | `false` |
| `cluster` | Cluster name, attached as `cluster` to metrics, events, and container logs. | `""` |
| `environment` | Environment name, attached as `service.environment` on container logs. | `""` |
| `otel_daemon.enableHostMetrics` | Deploy the DaemonSet and collect host metrics (CPU, memory, disk, filesystem, network, load). | `false` |
| `otel_daemon.containerlogs.enabled` | Deploy the DaemonSet and collect container stdout/stderr logs. | `false` |
| `otel_daemon.containerlogs.namespaces` | Per-namespace log filters. Empty collects all namespaces, pods, and levels; non-empty is a whitelist. See below. | `[]` |
| `otel_daemon.resources` | Resource requests and limits for the DaemonSet collector. | `256m`/`256Mi` limits, `50m`/`50Mi` requests |
| `kube-state-metrics.enabled` | Install the kube-state-metrics subchart. | `true` |

## Container log filtering

When `otel_daemon.containerlogs.enabled` is `true` and `namespaces` is empty, logs are collected from every namespace, pod, and severity level. Providing entries turns the list into a whitelist — only the listed namespaces are collected.

Each entry names a namespace and optionally narrows it:

```yaml
otel_daemon:
  containerlogs:
    enabled: true
    namespaces:
      - name: "production"
        include: ["^api-.*", "^web-.*"]
        exclude: [".*-canary-.*"]
        levels: ["ERROR", "WARN"]
```

- `include` — regexes matched against the pod name; a pod is kept only if it matches at least one. Omit to keep all pods.
- `exclude` — regexes matched against the pod name; a matching pod is dropped. Exclude wins over include. Omit to exclude nothing.
- `levels` — severity values to keep, from `ERROR`, `WARN`, and `INFO`. Omit to keep all levels.

## Uninstall

```bash
helm uninstall osuite-k8s --namespace osuite
```

## Documentation

- [Kubernetes monitoring](https://osuite.io/docs/infrastructure/kubernetes/)
- [Kubernetes events](https://osuite.io/docs/infrastructure/kubernetes/events/)
- [Container logs](https://osuite.io/docs/infrastructure/kubernetes/container-logs/)
- [Manual collector setup](https://osuite.io/docs/infrastructure/kubernetes/manual-collector/)
