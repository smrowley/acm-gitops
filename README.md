# Configure ACM for GitOps config management

## Prerequisites

1. Install the ACM operator
1. Install the GitOps operator

## Configuration Steps

1. Import/create clusters
1. Create a `ManagedClusterSet`:

    ```yaml
    apiVersion: cluster.open-cluster-management.io/v1alpha1
    kind: ManagedClusterSet
    metadata:
      name: sample
    spec: {}
    ```

1. Add managed clusters to `ManagedClusterSet` by labeling the `ManagedClusters` you want in the set with the `cluster.open-cluster-management.io/clusterset` label. The value needs to match the `ManagedClusterSet` name:

    ```
    oc label managedcluster cluster1 cluster.open-cluster-management.io/clusterset=sample

    oc label managedcluster cluster2 cluster.open-cluster-management.io/clusterset=sample
    ```

1. Create binding for `ManagedClusterSet`. Applications and policies that are created in the same namespace can only access managed clusters that are included in the bound managed cluster set resource:

    ```yaml
    apiVersion: cluster.open-cluster-management.io/v1alpha1
    kind: ManagedClusterSetBinding
    metadata:
      name: sample
      namespace: openshift-gitops
    spec:
      clusterSet: sample
    ```

1. Create a `Placement` in the same namespace as the `ManagedClusterSetBinding`. The `Placement` defines eligible groups of clusters given a label selector:

    ```yaml
    apiVersion: cluster.open-cluster-management.io/v1alpha1
    kind: Placement
    metadata:
      name: sample
      namespace: openshift-gitops
    spec:
      predicates:
      - requiredClusterSelector:
          labelSelector:
            matchExpressions:
            - key: vendor
              operator: "In"
              values:
              - OpenShift
    ```

    _Note: need to confirm, but I believe Placements allows subdividing of ManagedClusterSets for targeting config and policies._

1. Create `GitOpsCluster` resource:

    ```yaml
    apiVersion: apps.open-cluster-management.io/v1beta1
    kind: GitOpsCluster
    metadata:
      name: argo-acm-importer
      namespace: openshift-gitops
    spec:
      argoServer:
        cluster: local-cluster #must target the hub cluster
        argoNamespace: openshift-gitops
      placementRef:
        kind: Placement
        apiVersion: cluster.open-cluster-management.io/v1alpha1
        name: sample
        namespace: openshift-gitops
    ```

1. Create `ApplicationSet` resource. This provides a template for creating `Application` resources for ArgoCD to pick up. The `clusterDecisionResource` attribute looks at the corresponding `PlacementDecision` to determine which clusters are eligible for the `Applications`:

    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: ApplicationSet
    metadata:
      name: sample
      namespace: openshift-gitops
    spec:
      generators:
        - clusterDecisionResource:
            configMapRef: acm-placement
            labelSelector:
            matchLabels:
                cluster.open-cluster-management.io/placement: sample
            requeueAfterSeconds: 180
      template:
        metadata:
        name: 'sample-{{name}}'
        spec:
        project: default
        source:
            repoURL: 'https://github.com/redhat-developer-demos/openshift-gitops-examples'
            targetRevision: main
            path: apps/bgd/overlays/bgd
        destination:
            server: '{{server}}'
            namespace: bgd
        syncPolicy:
            automated:
            prune: true
            selfHeal: true
        syncOptions:
            - CreateNamespace=true
    ```

### Errata

The ACM wizard for creating `ApplicationSets` also specified creating this `Channel` resource. It's not necessary, but essentially "saves" source repos for the ACM UI to allow easy selection.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ggithubcom-redhat-developer-demos-openshift-gitops-examples-ns
---
apiVersion: apps.open-cluster-management.io/v1
kind: Channel
metadata:
  name: ggithubcom-redhat-developer-demos-openshift-gitops-examples
  namespace: ggithubcom-redhat-developer-demos-openshift-gitops-examples-ns
spec:
  type: Git
  pathname: 'https://github.com/redhat-developer-demos/openshift-gitops-examples'

```
