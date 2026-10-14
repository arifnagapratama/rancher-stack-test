# Loki integration design

## Goal

Deploy Loki alongside the existing Rancher Monitoring and Rancher Logging stacks, route cluster logs to Loki, and provision Loki as a Grafana datasource so logs are available in Grafana Explore.

## Current context

- Rancher Monitoring is deployed from `monitoring/10-stack`, in `cattle-monitoring-system`.
- Grafana is part of Rancher Monitoring and persists data on a 5Gi PVC.
- Rancher Logging and its CRD chart are deployed in `cattle-logging-system` at chart version `106.0.8+up4.10.0-rancher.24`.
- Rancher Logging uses the Logging Operator 4.10 line, which supports Loki outputs and cluster-wide flows.
- The installed Grafana version is expected to be 11.5.5. Grafana's Loki datasource is built in, so no plugin installation is needed.

## Design

### Loki deployment

Add a Fleet bundle for Loki under `monitoring/40-loki`. Use the Grafana Community Loki chart, pinned to chart version `18.7.6` (Loki application version `3.7.6`), in monolithic/single-binary mode. Run it in `cattle-logging-system` to keep the log backend near Rancher Logging. This chart repository is the maintained OSS path after the Loki chart migration.

Use filesystem storage on a 20Gi `ReadWriteOnce` PVC and retain logs for 15 days. This follows the existing monitoring retention window and storage class behavior. Loki will remain internal to the cluster; no external ingress or authentication layer is part of this change.

### Log delivery

Add a Fleet bundle under `monitoring/50-logging-pipeline` containing:

- A `ClusterOutput` in `cattle-logging-system` that sends records to Loki's in-cluster service.
- A `ClusterFlow` matching all namespaces and forwarding matching records to that output.

The bundle must depend on both the Rancher Logging bundle and the Loki bundle, so the Logging CRDs and Loki service exist before the pipeline is applied. Retain Kubernetes metadata and provide a bounded Fluentd buffer with retries so temporary Loki unavailability does not immediately discard queued records.

### Grafana datasource

Add a provisioned datasource named `Loki` to `monitoring/10-stack/values.yaml`, targeting Loki's in-cluster query service. Keep it non-default so the existing Prometheus datasource remains Grafana's default. This makes Loki available from Explore and dashboards without manual Grafana setup.

## Alternatives considered

1. **Monolithic Loki with a PVC (selected):** simplest fit for the repository's existing single-cluster Helm deployments and explicitly approved local-storage/15-day assumption.
2. **Loki with S3/MinIO object storage:** better suited to larger or more durable installations, but this repository has no object-storage endpoint or credentials to configure.
3. **Grafana Alloy instead of Rancher Logging for shipping:** Grafana recommends Alloy for new Grafana-native collection pipelines, but this cluster already has Rancher Logging installed and its operator supports the required Loki output. Adding a second collector would duplicate log collection.

## Failure behavior and boundaries

- Loki runs as a single binary with one PVC; this is not a highly available logging backend.
- If Loki is unavailable, Fluentd retries using its configured buffer; buffer exhaustion can still cause loss, so capacity and retry settings must be bounded.
- PVC capacity is fixed at 20Gi in this initial configuration. Growth beyond that requires increasing the PVC and may require a storage migration depending on the cluster's storage class.
- This change routes cluster container logs collected by Rancher Logging. It does not add custom parsing, retention dashboards, alert rules, or external access.

## Validation criteria

- Fleet renders and accepts the Loki, logging pipeline, and datasource configuration.
- Loki becomes ready with its PVC bound.
- Rancher Logging reports the `ClusterOutput` and `ClusterFlow` active and forwards records to Loki.
- Grafana shows the Loki datasource as healthy, and Grafana Explore can query recent Kubernetes log records.

## References

- [Grafana Loki Helm chart migration](https://grafana.com/docs/loki/latest/setup/upgrade/)
- [Install the monolithic Loki Helm chart](https://grafana.com/docs/loki/latest/setup/install/helm/install-monolithic/)
- [Grafana Loki datasource](https://grafana.com/docs/grafana/latest/datasources/loki/)
- [Logging Operator 4.10 output types](https://kube-logging.dev/4.10/docs/configuration/crds/v1beta1/output_types/)
