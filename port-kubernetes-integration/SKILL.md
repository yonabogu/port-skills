---
name: port-kubernetes-integration
description: >
  Port.io Kubernetes integration expert. Use when configuring Port's K8s exporter/Ocean integration:
  mapping Kubernetes workloads (Deployments, ReplicaSets, Pods, StatefulSets, DaemonSets), Nodes,
  Namespaces, CRDs, Services, Ingresses, and ConfigMaps into Port entities; writing K8s exporter
  mapping files; handling K8s-specific JQ patterns for labels and annotations; and setting up the
  Helm chart for the K8s exporter. Essential when the user mentions Port Kubernetes integration,
  K8s exporter, syncing cluster data to Port, or mapping K8s resources.
license: Apache-2.0
metadata:
  author: port-io
  version: "1.0"
compatibility: Designed for use with Port.io (port.io) and Kubernetes. Requires Helm.
---

# Port Kubernetes Integration

Port's Ocean-based K8s integration continuously syncs Kubernetes resources into your catalog.

## Supported Resource Kinds

| Kind | K8s Resource |
|---|---|
| `Deployment` | apps/v1 Deployments |
| `ReplicaSet` | apps/v1 ReplicaSets |
| `Pod` | v1 Pods |
| `StatefulSet` | apps/v1 StatefulSets |
| `DaemonSet` | apps/v1 DaemonSets |
| `Job` | batch/v1 Jobs |
| `CronJob` | batch/v1 CronJobs |
| `Service` | v1 Services |
| `Ingress` | networking.k8s.io/v1 Ingresses |
| `ConfigMap` | v1 ConfigMaps |
| `Namespace` | v1 Namespaces |
| `Node` | v1 Nodes |

Any CRD can also be mapped using its fully-qualified `apiVersion/kind`.

## Installation (Helm)

```bash
helm repo add port-labs https://port-labs.github.io/helm-charts
helm repo update

helm install port-k8s-exporter port-labs/port-k8s-exporter \
  --namespace port-k8s-exporter --create-namespace \
  --set secret.secrets.portClientId=$PORT_CLIENT_ID \
  --set secret.secrets.portClientSecret=$PORT_CLIENT_SECRET \
  --set portBaseUrl=https://api.port.io \
  --set stateKey=k8s-exporter \
  -f values.yaml
```

## values.yaml — Mapping Configuration

```yaml
config:
  resources:
    - kind: Deployment
      selector:
        query: .metadata.namespace != "kube-system"
      port:
        entity:
          mappings:
            identifier: .metadata.name + "-" + .metadata.namespace
            title: .metadata.name
            blueprint: '"workload"'
            properties:
              namespace:     .metadata.namespace
              cluster:       env.CLUSTER_NAME        # injected via Helm value
              image:         .spec.template.spec.containers[0].image
              replicas:      .spec.replicas
              available:     .status.availableReplicas
              labels:        .metadata.labels
              annotations:   .metadata.annotations
              created_at:    .metadata.creationTimestamp
              app_version:   '.metadata.labels."app.kubernetes.io/version" // null'
            relations:
              namespace: .metadata.namespace
              service: '.metadata.labels."app.kubernetes.io/name" // .metadata.labels.app // null'

    - kind: Namespace
      selector:
        query: "true"
      port:
        entity:
          mappings:
            identifier: .metadata.name
            title: .metadata.name
            blueprint: '"namespace"'
            properties:
              labels: .metadata.labels
              status: .status.phase

    - kind: Node
      selector:
        query: "true"
      port:
        entity:
          mappings:
            identifier: .metadata.name
            title: .metadata.name
            blueprint: '"node"'
            properties:
              os:       '.status.nodeInfo.osImage'
              kernel:   '.status.nodeInfo.kernelVersion'
              cpu:      '.status.capacity.cpu | tonumber'
              memory:   '.status.capacity.memory'
              ready:    '[.status.conditions[] | select(.type=="Ready")] | .[0].status == "True"'
```

## Mapping CRDs

Any CRD is mapped by specifying its `apiVersion` and `kind`:

```yaml
- kind: argoproj.io/v1alpha1/Application
  selector:
    query: "true"
  port:
    entity:
      mappings:
        identifier: .metadata.name
        title: .metadata.name
        blueprint: '"argocdApp"'
        properties:
          sync_status:   .status.sync.status
          health_status: .status.health.status
          repo_url:      .spec.source.repoURL
          target_rev:    .spec.source.targetRevision
          destination:   .spec.destination.server
```

## Useful K8s JQ Patterns

```jq
# Get container image
.spec.template.spec.containers[0].image

# Get all container images
[.spec.template.spec.containers[].image]

# Check if deployment is fully available
.status.availableReplicas == .spec.replicas

# Get label by key (keys with dots need bracket syntax)
.metadata.labels["app.kubernetes.io/name"]
# Or in YAML mapping (use string escape):
'.metadata.labels."app.kubernetes.io/name"'

# Get annotation
.metadata.annotations."kubectl.kubernetes.io/last-applied-configuration"

# Node ready status
'[.status.conditions[] | select(.type=="Ready")] | .[0].status'

# Pod phase
.status.phase

# Resource requests/limits
.spec.containers[0].resources.requests.cpu
.spec.containers[0].resources.limits.memory
```

## Gotchas

- K8s resource names are namespace-scoped. Use `name + "-" + namespace` as the identifier to avoid collisions.
- Label keys with dots (e.g., `app.kubernetes.io/name`) need bracket notation in JQ: `.metadata.labels["app.kubernetes.io/name"]`.
- The exporter runs in the cluster — it uses in-cluster RBAC to read resources. Ensure the ServiceAccount has the right ClusterRole.
- `env.CLUSTER_NAME` is available in mapping JQ if set as a Helm value, allowing you to tag entities with the source cluster.
- CRD mappings use the full `apiVersion/kind` path as the `kind` value in the config.
