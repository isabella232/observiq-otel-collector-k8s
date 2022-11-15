# Google Cloud Logging

This example can be used to ship logs from GKE Autopilot cluster to Google Cloud.

## Prerequisites

GKE Autopilot uses [gke workload identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity), which means we need to
map the collector's Kubernetes Service Account (KSA) to a Google Cloud IAM account (GSA).

**Environment**

Set environment variables which will be used by gcloud commands. `daemonset.yaml` will
create the Kubernetes Service Account `observiq-otel-collector` in the namespace `default. If you
modify these values, be sure to reflect those changes here.

```bash
PROJECT="myproject"
KSA="observiq-otel-collector"
GSA="observiq-otel-collector"
NAMESPACE=default
```

NOTE: The Google Service account must be created in the project that the cluster is running in. If you wish
to send logs to a different project, you can grant the Google Service Account permission to that project.

**Create the GSA**

```bash
gcloud --project "${PROJECT}" \
  iam service-accounts create "${GSA}"
```

**Grant GSA logging.logWriter role**

```bash
gcloud projects add-iam-policy-binding "${PROJECT}" \
  --member "serviceAccount:${GSA}@${PROJECT}.iam.gserviceaccount.com" \
  --role "roles/logging.logWriter"
```

**Bind the GSA with KSA**

```bash
gcloud \
  --project "${PROJECT}" iam service-accounts add-iam-policy-binding \
    "${GSA}@${PROJECT}.iam.gserviceaccount.com" \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:${PROJECT}.svc.id.goog[${NAMESPACE}/${KSA}]"
```

**Set KSA Annotation**

Modify `daemonset.yaml`'s ServiceAccount configuration. Update the `iam.gke.io/gcp-service-account`
annotation to point to the correct Google Service Account email.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: observiq-otel-collector
  namespace: default
  annotations:
    # example: observiq-otel-collector@myproject.iam.gserviceaccount.com
    iam.gke.io/gcp-service-account: <gsa email>
```

## Deploy

After following the prerequisites instructions, simply deploy the daemonset with kubectl:

```bash
kubectl apply -f daemonset.yaml
```

NOTE: The example `daemonset.yaml` has the following resource configuration. See [the docs](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-resource-requests) for more information on Autopilot resource allocation.

```yaml
resources:
  # Default values for Daemonset's deployed to GKE Autopilot.
  limits:
    cpu: 50m
    ephemeral-storage: 100Mi
    memory: 100Mi
  requests:
    cpu: 50m
    ephemeral-storage: 100Mi
    memory: 100Mi
```
