# otel-collector

## Overview

This package deploys an [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) to your Kubernetes cluster. The Collector is managed via a Kubernetes ConfigMap (`configmap.yaml`), which defines how logs are received, processed, and exported.

## OpenTelemetry Collector Configuration

The Collector is configured using the `otel-collector-nephio-config-<clusterType>` ConfigMap in the `monitoring` namespace. The package-context.yaml provides another ConfigMap to kpt to allow customisation of the yaml files with kpt-setters. The configuration (`config.yaml`) includes:

### Receivers

- **otlp:** Accepts telemetry data over gRPC and HTTP.

  *Note: The OTLP receiver can be used for multi-cluster communication, enabling telemetry data to be sent from workload clusters to the management cluster.*
- **filelog:** Tails log files from specific Kubernetes pods:
  - `/var/log/pods/porch-system_porch-server-*/porch-server/*.log`
  - `/var/log/pods/porch-system_porch-controllers-*/porch-controllers/*.log`
  - The wildcard * can be used to observe other or all logs.
  - Starts reading from the beginning of each file.
  - Uses a regex parser to extract fields such as timestamp, log level, thread, and message from each log line.
  - Maps log levels (`I`, `W`, `E`, `F`) to standard severity names (`info`, `warn`, `error`, `fatal`).
  - **TODO: k8s_cluster:** Can be enabled to collect Kubernetes cluster-level metrics and metadata.


### Processors

- **batch:** Batches log records before exporting for improved performance.
  - Configured with a batch size of 1000, a timeout of 10s, and a max batch size of 10,000.

### Exporters

- **elasticsearch:** Sends logs to an Elasticsearch cluster.
  - Endpoint, user, and password are configurable via environment variables.
  - TLS is set to skip verification (for testing only; secure this in production).
  - Dynamic index creation for logs is enabled.
- **debug:** (For troubleshooting) Outputs detailed logs to the Collector’s own logs.
- **TODO prometheus:** Can be enabled to export metrics in Prometheus format for scraping by Prometheus servers. For this reason a Prometheus CR, ServiceMonitor, has been added to scrape a prometheus exporter endpoint in config.yaml (not yet implemented).


### Service & Pipelines

- **logs pipeline:** 
  - Receives logs from `filelog`.
  - Processes them with the `batch` processor.
  - Exports them to `elasticsearch`.
- **Telemetry:** Collector’s own logs are set to `debug` level with console encoding.

## Customization

- **Elasticsearch credentials:** Set `ELASTIC_HOST`, `ELASTIC_PORT`, and `ELASTIC_PASSWORD` as environment variables or Kubernetes secrets.
- **Log sources:** Modify the `include` patterns in `filelog` to collect logs from other pods or containers.
- **Regex parsing:** Adjust the `regex` and field mappings to match your log format.

## Usage

### Fetch the package

```sh
kpt pkg get REPO_URI[.git]/PKG_PATH[@VERSION] otel-collector
```
[Reference](https://kpt.dev/reference/cli/pkg/get/)

### View package content

```sh
kpt pkg tree otel-collector
```
[Reference](https://kpt.dev/reference/cli/pkg/tree/)

### Apply the package

```sh
kpt live init otel-collector
kpt live apply otel-collector --reconcile-timeout=2m --output=table
```
[Reference](https://kpt.dev/reference/cli/live/)

---

For more details on customizing the OpenTelemetry Collector, see the [official documentation](https://opentelemetry.io/docs/collector/configuration/).