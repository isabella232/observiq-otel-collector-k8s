# StatsD Metrics with Google Cloud

This example can be used to ship StatsD metrics from any Kubernetes cluster
to Google Cloud.

- [Setup](#setup)
  * [Authentication](#authentication)
  * [Cluster Name](#cluster-name)
- [Usage](#usage)
- [Collectors](#collectors)
  * [gateway.yaml](#gatewayyaml)
  * [collector-deployment.yaml](#collector-deploymentyaml)
  * [collector-daemonset.yaml](#collector-daemonsetyaml)
  * [app-remote.yaml](#app-remoteyaml)
  * [app-sidecar.yaml](#app-sidecaryaml)
  * [app-sidecar-standalone.yaml](#app-sidecar-standaloneyaml)
- [Architecture](#architecture)
  * [Side Car](#side-car)
  * [Deployment](#deployment)
  * [Daemonset](#daemonset)
- [FAQ](#faq)

## Setup

### Authentication

If running **outside of GCP**, deploy Google credentials. The credentials file
name must be `credentials.json`.

```bash
kubectl create secret generic gcp-credentials \
    --from-file=credentials.json
```

### Cluster Name

If running outside of GKE, a cluster name must be set in `gateway.yaml`. Update
the environment section.

```yaml
env:
  - name: OTEL_RESOURCE_ATTRIBUTES_CLUSTER_NAME
    value: minikube
```

If running within GKE, the following resource attributes are detected automatically.
- k8s.cluster.name
- cloud.region
- cloud.availability_zone
- cloud.platform

## Usage

Deploy to your cluster.

```bash
kubectl apply -f gateway.yaml
kubectl apply -f collector-deployment.yaml
kubectl apply -f app-remote.yaml
kubectl apply -f app-sidecar.yaml
kubectl apply -f app-sidecar-standalone.yaml
```

## Collectors

The following collectors are deployed.

### gateway.yaml

Collector with OTLP receiver and Google Cloud exporter.

This collector contains processors which add the following resources to
all metrics:
- `k8s.cluster.name`
- `cloud.region`
- `cloud.availability_zone`
- `cloud.platform`

These resources are required in order to map to `k8s_pod` [Monitored Resource Types](https://cloud.google.com/logging/docs/api/v2/resource-list).

### collector-deployment.yaml

Collector deployment with StatsD receiver and OTLP exporter.
StatsD clients access the collector(s) via a clusterIP service.

This collector contains processors which convert the following
metric labels (statsd tags) to resource attributes:
- `k8s.pod.name`
- `k8s.namespace.name`

These tags are required in order to map to `k8s_pod` [Monitored Resource Types](https://cloud.google.com/logging/docs/api/v2/resource-list).

### collector-daemonset.yaml

Collector daemonset with StatsD receiver and OTLP exporter.
The collector(s) are listening for StatsD metrics using a hostPort.
StatsD clients forward metrics to their's node's IP address on the StatsD
port. This allows all pods on a given node to forward metrics to the daemonset
collector running on that same node.

This collector contains processors which convert the following
metric labels (statsd tags) to resource attributes:
- `k8s.pod.name`
- `k8s.namespace.name`

These tags are required in order to map to `k8s_pod` [Monitored Resource Types](https://cloud.google.com/logging/docs/api/v2/resource-list).

### app-remote.yaml

StatsD example application, which forwards metrics to `collector-deployment.yaml`'s 
clusterIP service `observiq-statsd-collector` on port `8125/udp`.

StatsD metrics emitted by this deployment contain the following tags:
- `k8s.pod.name`
- `k8s.namespace.name`

These tags are required in order to map to `k8s_pod` [Monitored Resource Types](https://cloud.google.com/logging/docs/api/v2/resource-list).

### app-sidecar.yaml

StatsD example application which forwards metrics to a side car collector which
is running in the same pod as the application container. This allows the application
to send metrics to `localhost` instead of a remote ip address or hostname.

The collector contains processors which add the following resource attributes:
- `k8s.pod.name`
- `k8s.namespace.name`

### app-sidecar-standalone.yaml

Identical to `app-sidecar.yaml`, except the OTLP export has been replaced by the Google Cloud exporter.
This allows the application pod to send StatsD metrics straight to Google Cloud logging. This is the simplest
solution, however, it may not be ideal to have application pods mounting Google Cloud credentials. 

If you want more control over credentials and outbound connection to Google cloud, see the `app-remote.yaml` option.

## Architecture

Two collector architecture options are available.
- Side car collector
- Collector deployment

### Side Car

When running a collector with the [statsd receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/statsdreceiver)
as a sidecar, applications can forward StatsD metrics to `localhost`.

The side car method does not require changes to your codebase.

This example architecture implements sidecar collectors in the following way.

```
Application pod (with sidecar collector) --> Google gateway collector --> Google Cloud
```

Because metrics are exported via the OTLP exporter to the Google gateway collector,
the application pods do not need to mount Google credentials.

If you wish to send metrics from the application pods straight to google, replace
the OTLP exporter with the Google Cloud exporter, and mount credentials to the application
pods. See `gateway.yaml` for an example.

**Pros and Cons**

- Pros
  - Simple setup.
    - Send metrics to localhost
    - Metric tags do not require modification
- Cons
  - All statsd application pods contain a collector container.
  - Higher memory footprint.
    - High aggregation intervals will lead to higher memory consumption.

### Deployment

When running a collector with the [statsd receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/statsdreceiver)
as a deployment, applications can forward StatsD metrics to a Kubernetes clusterIP service hostname, which then forwards
metrics to a collector pod from the deployment.

This method is preferable, however, it requires that all statsd metrics being emitted by your applications
contain the following tags.
- `k8s.pod.name`
- `k8s.namespace.name`

Implementing these tags allows metrics to appear under "Kubernetes Pod" when searching metrics in the Google Cloud metrics explorer.

This example architecture is implemented in the following way.

```
Application pod --> StatsD collector --> Google gateway collector --> Google Cloud
```

**Pros and Cons**

- Pros
  - Less collector containers.
- Cons
  - Possibility of [uneaven load balancing](https://github.com/kubernetes/kubernetes/issues/76517) among collector pods.
    - This can become a problem if many statsd clients come online suddenly, before the pod autoscaler
      can scale up the collector deployment.
  - Must implement metric tags derived from downward api environment variables.
    - `k8s.pod.name`
    - `k8s.namespace.name`

### Daemonset

The daemonset collector functions similar to the deployment collector. It scales up
and down with the node pool instead of using a pod autoscaler.

Daemonset is simpler, as it does not require a clusterIP servce or autoscaler, however, it may have a larger footprint than the deployment option because it requires that a collector
run on every node.

## FAQ

**Q: Can collector-deployment.yaml and gateway.yaml collectors be merged?**

**A**: Yes. They are seperate collectors in order to maintain a seperation of duties. The statsd collector handles statsd metrics while the gateway collector handles mapping metrics to `k8s_pod` [Monitored Resource Type](https://cloud.google.com/logging/docs/api/v2/resource-list). It is important to retain the `batch`, `groupbyattrs`, `resource`, and `resourcedetection` processors. The `groupbyattrs` should come after `batch`, while batch should come after all other processors.

**Q: Are StatsD tags mapped to metrics labels?**

**A**: Yes. StatsD tags are converted to Open Telemetry metric labels.
