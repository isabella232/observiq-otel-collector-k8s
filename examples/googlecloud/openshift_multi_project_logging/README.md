# Google Cloud Multi Project Logging (OpenShift)

This example can be used to ship logs from OpenShift to multiple Google Cloud projects.

Each unique namespace will be matched with a unique Google project.

**Create Project**

```bash
oc create -f project.yaml
```

**Create Google Cloud Credential Secret**

Create a Google service account and download an application json key. Name the file `credentials.json` (The file name is important, as that is what will be referenced by the daemonset volume mount). The service account
should have the `logs writer` role for every project you wish to send logs to. This example uses a single service account which has permission to all projects.

```bash
oc -n observiq create secret generic gcp-credentials \
    --from-file=credentials.json
```

**Apply Setup**

Create the Security Context Constraints and RBAC rules.

```bash
oc apply -f setup.yaml
```

**Update Routing and Exporters**

The example configuration has a `routing` processor and a set of Google exporters.

Routing Processor:
```yaml
routing:
  attribute_source: resource
  from_attribute: k8s.namespace.name
  default_exporters:
  - logging
  table:
  - value: dev
    exporters:
      - googlecloud/dev
      - logging
  - value: stage
    exporters:
      - googlecloud/stage
      - logging
  - value: prod
    exporters:
      - googlecloud/prod
      - logging
```

Matching exporters:
```yaml
exporters:
  googlecloud/dev:
    project: multi-project-logging-dev

  googlecloud/stage:
    project: multi-project-logging-stage

  googlecloud/prod:
    project: multi-project-logging-prod

  logging:
```

1. Update the Google exporters' `project` field to match your projects. You should have a single Google exporter per project you wish to send logs to.
2. Update the routing processor to map to the exporters. In this example, we are routing based on namespace name.

**Deploy Collector Daemonset**

1. Open the file `observiq-otel-collector.yaml` with your editor of choice

2. Modify the cluster name by finding and replacing `minishift`. This is a friendly name that will allow you to search between multiple clusters.
```yaml
- name: K8S_CLUSTER
  value: minishift
```

3. Deploy the collector.
```bash
oc apply -f observiq-otel-collector.yaml
```
