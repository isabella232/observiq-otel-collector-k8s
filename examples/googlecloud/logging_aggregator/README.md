# Google Cloud Logging

This example can be used to ship logs from any Kubernetes cluster to Google Cloud.

## Architecture

The aggregator model a common pattern for large clusters. It is useful if you wish to
reduce the number of collectors making outbound connections to Google Cloud as it may not
be desireable to have 100s of collectors making connections to Google Cloud.

**Telemetry flow**:

Daemonset log collectors --> Deployment aggregator collectors --> Google Cloud

**Pros**:
- Google credentials are exposed to the aggregator collectors only.
- Credentials can be rotated without restarting the collectors that read logs.
- Aggregator collectors can perform additional processing, such a adding or removing metadata.
- Only the aggregator collectors are making expensive network calls to Google Cloud, allowing the log collectors to focus on simply reading and forwarding logs to the aggregator.

**Cons**:
- While not overly complex, this archetecture is more complex. It is simpler to have the log collectors use the Google exporter.
- Additional collectors (more overhead). Not suitable for small clusters.

## Usage

Update the configuration to reflect your environment.

**Authentication**

If running **outside of GCP**, deploy Google credentials:
```bash
kubectl create secret generic gcp-credentials \
    --from-file=credentials.json
```

**Set Cluster Name**

Open `deployment.yaml` in your editor of choice.

Update the cluster name environment variable. This is a "friendly" name that
can be searched on in the Cloud Logging interface.

```yaml
- name: K8S_CLUSTER
  value: minikube
```

You can filter on the cluster name with this Cloud Logging query:

```
resource.labels.cluster_name="minikube"
```

**Update Cluster Location (Optional)**

Google requires that a region and availability zone resource be set. This is part of the
[Monitored Resource Types](https://cloud.google.com/logging/docs/api/v2/resource-list) mapping.

Feel free to modify the following values, but understand that they must line up with Google supported values. A bad configuration
will result in logs coming in as "generic_node" resource type, instead of "k8s_container".

```yaml
resource/mapping:
  attributes:
    - key: cloud.region
      value: us-east1
      action: insert
    - key: cloud.availability_zone
      value: us-east1-b
      action: insert
    - key: cloud.platform
      value: gcp_kubernetes_engine
      action: insert
```

**Deploy**

```bash
kubectl apply -f deployment.yaml
kubectl apply -f daemonset.yaml
```
