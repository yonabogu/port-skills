# Port Kubernetes Integration — Reference

## Minimal ClusterRole for the Exporter

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: port-k8s-exporter
rules:
  - apiGroups: ["", "apps", "batch", "networking.k8s.io"]
    resources:
      - namespaces
      - nodes
      - pods
      - deployments
      - replicasets
      - statefulsets
      - daemonsets
      - jobs
      - cronjobs
      - services
      - ingresses
      - configmaps
    verbs: ["get", "list", "watch"]
  # Add custom resource groups for CRDs:
  - apiGroups: ["argoproj.io"]
    resources: ["applications", "appprojects"]
    verbs: ["get", "list", "watch"]
```

## Helm values for multi-cluster setup

```yaml
# values-prod.yaml
config:
  resources:
    - kind: Deployment
      selector:
        query: "true"
      port:
        entity:
          mappings:
            identifier: '.metadata.name + "-" + .metadata.namespace + "-prod"'
            title: .metadata.name
            blueprint: '"workload"'
            properties:
              cluster: '"production"'
              namespace: .metadata.namespace
```

## Common Blueprint: workload

```json
{
  "identifier": "workload",
  "title": "Workload",
  "schema": {
    "properties": {
      "namespace":   { "type": "string", "title": "Namespace" },
      "cluster":     { "type": "string", "title": "Cluster" },
      "image":       { "type": "string", "title": "Image" },
      "replicas":    { "type": "number", "title": "Desired Replicas" },
      "available":   { "type": "number", "title": "Available Replicas" },
      "labels":      { "type": "object", "title": "Labels" }
    }
  },
  "calculationProperties": {
    "is_healthy": {
      "type": "boolean",
      "calculation": ".properties.available == .properties.replicas and .properties.replicas > 0"
    }
  },
  "relations": {
    "service":    { "title": "Service",    "target": "microservice", "required": false, "many": false },
    "namespace":  { "title": "Namespace",  "target": "namespace",    "required": false, "many": false }
  }
}
```
