# Templates system

By default, Hybrid Container Cloud delivers a set of default `Template` objects. You can also build your own templates
and use them for deployment.

## Custom deployment Templates

> At the moment all `Templates` should reside in the `hmc-system` namespace. But they can be referenced
> by `ManagedClusters` from any namespace.

Here are the instructions on how to bring your own Template to HMC:

1. Create a [HelmRepository](https://fluxcd.io/flux/components/source/helmrepositories/) object containing the URL to the
external Helm repository. Label it with `hmc.mirantis.com/managed: "true"`.
2. Create a [HelmChart](https://fluxcd.io/flux/components/source/helmcharts/) object referencing the `HelmRepository` as a
`sourceRef`, specifying the name and version of your Helm chart. Label it with `hmc.mirantis.com/managed: "true"`.
3. Create a `Template` object in `hmc-system` namespace referencing this helm chart in `spec.helm.chartRef`.
`chartRef` is a field of the
[CrossNamespaceSourceReference](https://fluxcd.io/flux/components/helm/api/v2/#helm.toolkit.fluxcd.io/v2.CrossNamespaceSourceReference) kind.

Here is an example of a custom `Template` with the `HelmChart` reference:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: custom-templates-repo
  namespace: hmc-system
  labels:
    hmc.mirantis.com/managed: "true"
spec:
  insecure: true
  interval: 10m0s
  provider: generic
  type: oci
  url: oci://ghcr.io/external-templates-repo/charts
```

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmChart
metadata:
  name: custom-template-chart
  namespace: hmc-system
  labels:
    hmc.mirantis.com/managed: "true"
spec:
  interval: 5m0s
  chart: custom-template-chart-name
  reconcileStrategy: ChartVersion
  sourceRef:
    kind: HelmRepository
    name: custom-templates-repo
  version: 0.2.0
```

```yaml
apiVersion: hmc.mirantis.com/v1alpha1
kind: Template
metadata:
  name: os-k0smotron
  namespace: hmc-system
spec:
  type: deployment
  providers:
    infrastructure:
      - openstack
    bootstrap:
      - k0s
    controlPlane:
      - k0smotron
  helm:
    chartRef:
      kind: HelmChart
      name: custom-template-chart
      namespace: default
```

The `Template` should follow the rules mentioned below:

1. `spec.type` should be `deployment` (as an alternative, the referenced helm chart may contain the
`hmc.mirantis.com/type: deployment` annotation in `Chart.yaml`).
2. `spec.providers` should contain the list of required Cluster API providers: `infrastructure`, `bootstrap` and
`controlPlane`. As an alternative, the referenced helm chart may contain the specific annotations in the `Chart.yaml` (value is a list of providers divided by comma). These fields are only used for validation. For example:

    `Template` spec:

    ```yaml
    spec:
      providers:
        infrastructure:
          - aws
        bootstrap:
          - k0s
        controlPlane:
          - k0smotron
    ```

    `Chart.yaml`:

    ```bash
    annotations:
      hmc.mirantis.com/infrastructure-providers: aws
      hmc.mirantis.com/controlplane-providers: k0smotron
      hmc.mirantis.com/bootstrap-providers: k0s
    ```

## Compatibility attributes

Each of the `*Template` resources has compatibility versions attributes, including exact versions or version constraints.
Both must be set in the Semantic Version format. Each attribute can be set either via the corresponding `.spec` fields or via the annotations.
Values set via the `.spec` have precedence over the values set via the annotations.

1. The `ProviderTemplate` resource has dedicated fields to set an exact compatible `CAPI` version along
with exact compatibility versions for each of the `infrastructure`, `bootstrap`
and `controlPlane` providers type.
Given compatibility values will be then set accordingly in the `.status` field.

    Example with the `.spec`:

    ```yaml
    apiVersion: hmc.mirantis.com/v1alpha1
    kind: ProviderTemplate
    # ...
    spec:
      capiVersion: 1.7.3 # exact version must be set
      providers:
        bootstrap:
        - name: k0s
          versionOrConstraint: 1.0.0 # exact version must be set
        controlPlane:
        - name: k0smotron
          versionOrConstraint: 1.34.0 # exact version must be set
        infrastructure:
        - name: aws
          versionOrConstraint: 1.2.3 # exact version must be set
    ```

    Example with the `annotations` in the `Chart.yaml`:

    ```yaml
    hmc.mirantis.com/capi-version: 1.7.3
    hmc.mirantis.com/bootstrap-providers: k0s 1.0.0 # space-separated comma-separated list, e.g. k0s 1.0.0, k0s 1.0.1
    hmc.mirantis.com/controlplane-providers: k0smotron 1.34.0 # space-separated comma-separated list, e.g. k0s 1.0.0, k0smotron 1.34.5
    hmc.mirantis.com/infrastructure-providers: aws 1.2.3 # space-separated comma-separated list
    ```

1. The `ClusterTemplate` resource has dedicated fields to set an exact compatible Kubernetes version
along with compatibility constrainted versions for each of the `infrastructure`, `bootstrap`
and `controlPlane` providers type to match against the related `ProviderTemplate` objects.
Given compatibility values will be then set accordingly in the `.status` field.

    Example with the `spec`:

    ```yaml
    apiVersion: hmc.mirantis.com/v1alpha1
    kind: ClusterTemplate
    # ...
    spec:
      k8sVersion: 1.30.0 # exact version must be set
      providers:
        bootstrap:
        - name: k0s
          versionOrConstraint: ">=1.0.0" # version constraints must be set
        controlPlane:
        - name: k0smotron
          versionOrConstraint: "^1.34.0" # version constraints must be set
        infrastructure:
        - name: aws
          versionOrConstraint: "~1.2.3" # version constraints must be set
    ```

    Example with the `annotations` in the `Chart.yaml`:

    ```yaml
    hmc.mirantis.com/k8s-version: 1.30.0
    hmc.mirantis.com/bootstrap-providers: k0s >=1.0.0 # space-separated comma-separated list
    hmc.mirantis.com/controlplane-providers: k0smotron ^1.34.0 # space-separated comma-separated list, e.g. k0s >=1.0.0, k0smotron ~1.34.0
    hmc.mirantis.com/infrastructure-providers: aws ~1.2.3 # space-separated comma-separated list
    ```

1. The `ClusterTemplate` resource has dedicated fields to set an compatibility constrainted
Kubernetes version to match against the related `ClusterTemplate` objects.
Given compatibility values will be then set accordingly in the `.status` field.

    Example with the `spec`:

    ```yaml
    apiVersion: hmc.mirantis.com/v1alpha1
    kind: ServiceTemplate
    # ...
    spec:
      k8sConstraint: "^1.30.0" # version constraints must be set
    ```

    Example with the `annotations` in the `Chart.yaml`:

    ```yaml
    hmc.mirantis.com/k8s-version-constraint: ^1.30.0
    ```

### Compatibility attributes enforcement

The aforedescribed attributes are being checked and enforced in the `validatingwebhook` sticking
to the following rules:

* both the exact and constraint version of the same type (e.g. `k8sVersion` and `k8sConstraint`) must
be set otherwise no check is performed;
* if a `ProviderTemplate` object's exact providers versions do not satisfy the constraints
from the related `ClusterTemplate` object, the updates to the `ManagedCluster` object will be blocked;
* if a `ClusterTemplate` object's exact kubernetes version does not satisfy the constraint
from the related `ServiceTemplate` object, the updates to the `ManagedCluster` object will be blocked.

## Remove Templates shipped with HMC

If you need to limit the cluster templates that exist in your HMC installation, follow the instructions below:

1. Get the list of `deployment` Templates shipped with HMC:

    ```bash
    kubectl get templates -n hmc-system -l helm.toolkit.fluxcd.io/name=hmc-templates  | grep deployment
    ```

    Example output:

    ```bash
    aws-hosted-cp              deployment   true
    aws-standalone-cp          deployment   true
    ```

2. Remove the templates from the list:

    ```bash
    kubectl delete template -n hmc-system <template-name>
    ```
<!---
TODO: document `--create-template=false` controller flag once the templates are limited to deployment templates only.
-->
