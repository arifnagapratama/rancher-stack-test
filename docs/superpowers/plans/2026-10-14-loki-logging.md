# Loki and Rancher Logging Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy Loki, send cluster logs from Rancher Logging to Loki, and make Loki available in Grafana Explore through a provisioned datasource.

**Architecture:** Fleet installs the Grafana Community Loki chart in `cattle-logging-system` with monolithic mode, filesystem storage, a 20Gi PVC, and 15-day retention. A Git-managed Fleet bundle applies Rancher Logging `ClusterOutput` and `ClusterFlow` resources. Rancher Monitoring provisions an internal Loki datasource through its Grafana chart values.

**Tech Stack:** Rancher Fleet, Helm, Grafana Community Loki chart `18.7.6` (Loki `3.7.6`), Rancher Logging / Logging Operator `4.10`, Rancher Monitoring.

**Spec:** `docs/superpowers/specs/2026-10-14-loki-logging-design.md`

## Global Constraints

- Keep every added configuration in YAML files in this repository and apply it through Fleet.
- Use Loki chart `18.7.6`, Loki application version `3.7.6`, and namespace `cattle-logging-system`.
- Run Loki in monolithic mode with one replica, filesystem storage, a `ReadWriteOnce` 20Gi PVC, and 15-day retention.
- Keep Loki internal to the cluster; do not add external ingress or authentication.
- Forward logs from all namespaces through Rancher Logging in `cattle-logging-system`.
- Provision a non-default Grafana datasource named `Loki`; preserve Prometheus as the default datasource.
- Do not add dashboards, custom parsing, alerts, or another log collector.

## Review Focus

- **Chart values for chart 18.7.6:** ensure the exact pinned chart renders with monolithic mode, filesystem schema, 15-day retention, one Loki server replica, and a single PVC. Keep the Loki Canary enabled because the chart's enabled Helm test requires it. Validate the rendered manifests with `helm template` using the pinned chart and the committed values.
- **Gateway addresses:** Grafana queries the Loki gateway root at `http://loki-gateway.cattle-logging-system.svc.cluster.local:80`; Rancher Logging pushes to the gateway's `/loki/api/v1/push` endpoint. Keep the push path off the Grafana datasource URL.
- **Fleet ordering:** the pipeline bundle waits for both the Logging bundle and Loki bundle. Check dependency names against the existing naming pattern in `monitoring/10-stack/fleet.yaml`.
- **Cluster-scoped logging resources:** put `ClusterOutput` and `ClusterFlow` in the Logging control namespace, `cattle-logging-system`, and ensure the flow selects all namespaces.
- **Storage and retry behavior:** confirm the chart renders a 20Gi PVC and the file buffer path is available to Fluentd; bound the buffer at 2Gi and enable retries.

---

### Task 1: Deploy Loki through Fleet

**Files:**
- Create: `monitoring/40-loki/fleet.yaml`
- Create: `monitoring/40-loki/values.yaml`

**Interfaces:**
- Consumes: the Grafana Community Helm repository and the existing Fleet bundle pattern in `monitoring/10-stack/fleet.yaml`.
- Produces: the in-cluster service `loki-gateway` in `cattle-logging-system`, which the datasource and logging pipeline use.

- [ ] **Step 1: Add the Loki Fleet bundle**

Set `defaultNamespace: cattle-logging-system`, release name `loki`, chart repo `https://grafana-community.github.io/helm-charts`, chart `loki`, version `18.7.6`, and load `values.yaml`.

- [ ] **Step 2: Add Loki values**

Set `deploymentMode: Monolithic`, `loki.auth_enabled: false`, and `loki.commonConfig.replication_factor: 1`. Configure the TSDB schema with `object_store: filesystem` and filesystem chunk/rule directories. Set `loki.limits_config.retention_period: 360h`, `loki.compactor.retention_enabled: true`, and `loki.compactor.delete_request_store: filesystem`. Configure one monolithic replica and persistence enabled with size `20Gi` and access mode `ReadWriteOnce`. Disable the `backend`, `read`, and `write` scalable targets and set their replicas to zero. Keep the gateway enabled as an internal ClusterIP service and disable gateway ingress. Confirm all keys against chart version `18.7.6` while rendering.

- [ ] **Step 3: Render the pinned chart**

Run `helm template loki grafana-community/loki --version 18.7.6 --namespace cattle-logging-system --values monitoring/40-loki/values.yaml`.

Expected: rendering succeeds and includes one monolithic Loki server replica, its Loki Canary, the internal `loki-gateway` service, and a 20Gi PVC; no ingress or MinIO resources are rendered.

### Task 2: Provision Loki in Grafana

**Files:**
- Modify: `monitoring/10-stack/values.yaml`

**Interfaces:**
- Consumes: `loki-gateway` from Task 1.
- Produces: a Grafana datasource with stable UID `loki`, name `Loki`, and type `loki`.

- [ ] **Step 1: Add the datasource to `grafana.additionalDataSources`**

Set `access: proxy`, `url: http://loki-gateway.cattle-logging-system.svc.cluster.local:80`, `isDefault: false`, and `editable: false`. Do not append `/loki/api/v1/push` to this query URL.

- [ ] **Step 2: Check the Rancher Monitoring values structure**

Render or inspect the pinned Rancher Monitoring chart values to confirm `grafana.additionalDataSources` is accepted and appears in the Grafana datasource ConfigMap.

Expected: the Loki datasource is provisioned with the internal gateway URL, while Prometheus remains the default datasource.

### Task 3: Send Rancher Logging records to Loki

**Files:**
- Create: `monitoring/50-logging-pipeline/fleet.yaml`
- Create: `monitoring/50-logging-pipeline/cluster-output.yaml`
- Create: `monitoring/50-logging-pipeline/cluster-flow.yaml`

**Interfaces:**
- Consumes: Rancher Logging CRDs from bundle `monitoring-monitoring-30-logging` and Loki gateway from bundle `monitoring-monitoring-40-loki`.
- Produces: active `ClusterOutput` named `loki` and `ClusterFlow` named `all-to-loki` in `cattle-logging-system`.

- [ ] **Step 1: Add pipeline bundle dependencies**

Configure `dependsOn` for `monitoring-monitoring-30-logging` and `monitoring-monitoring-40-loki`. Follow the existing bundle naming convention shown by `monitoring/10-stack/fleet.yaml`.

- [ ] **Step 2: Define the Loki ClusterOutput**

Create a `logging.banzaicloud.io/v1beta1` `ClusterOutput` in `cattle-logging-system` with Loki URL `http://loki-gateway.cattle-logging-system.svc.cluster.local:80`, `configure_kubernetes_labels: true`, and a Fluentd file buffer at `/buffers/loki`. Set `chunk_limit_size: 8MB`, `total_limit_size: 2GB`, `flush_interval: 5s`, and `retry_forever: true`. The Logging Operator Loki plugin appends the push API path, so the configured URL is the gateway base URL.

- [ ] **Step 3: Define the all-namespace ClusterFlow**

Create a `logging.banzaicloud.io/v1beta1` `ClusterFlow` in `cattle-logging-system` with `match: - select: {}` and `globalOutputRefs: [loki]`.

- [ ] **Step 4: Review the manifest interfaces**

Check the output and flow against the Logging Operator 4.10 schemas and ensure the chart's Fluentd buffer volume supports `/buffers/loki`.

Expected: both YAML manifests are valid for Rancher Logging 4.10 and the flow sends records from all namespaces to the named ClusterOutput.

### Task 4: Verify the integrated deployment

**Files:**
- No additional files.

**Interfaces:**
- Consumes: Loki, datasource, ClusterOutput, and ClusterFlow from Tasks 1–3.
- Produces: confirmation that the configuration is ready for use in Grafana Explore.

- [ ] **Step 1: Check Fleet and Kubernetes resource readiness**

After Fleet applies the bundles, inspect bundle status, Loki pod and PVC status, and the Logging Operator status for the `ClusterOutput` and `ClusterFlow`.

Expected: all bundles are ready, Loki's PVC is bound, and both logging resources report active without problems.

- [ ] **Step 2: Check datasource and log queries in Grafana**

Confirm the provisioned Loki datasource is healthy, open Grafana Explore, and query recent logs using `{namespace=~".+"}`.

Expected: Grafana can query recent records from multiple namespaces, and Prometheus remains the default datasource.

- [ ] **Step 3: Check formatting and repository state**

Run `rtk git diff --check` and review `rtk git status --short`.

Expected: no whitespace errors; only the intended Fleet YAML and monitoring values files are modified or added.
