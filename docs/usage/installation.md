### Installation

```bash
export KUBECONFIG=<path-to-management-kubeconfig>
```

```bash
helm install hmc oci://ghcr.io/mirantis/hmc/charts/hmc --version <hmc-version> -n hmc-system --create-namespace
```


#### Extended Management configuration

By default, the Hybrid Container Cloud is being deployed with the following configuration:

```yaml
apiVersion: hmc.mirantis.com/v1alpha1
kind: Management
metadata:
  name: hmc
spec:
  core:
    capi:
      template: cluster-api
    hmc:
      template: hmc
  providers:
  - template: k0smotron
  - config:
      configSecret:
       name: aws-variables
    template: cluster-api-provider-aws
```

There are two options to override the default management configuration of HMC:

1. Update the `Management` object after the HMC installation using `kubectl`:

    `kubectl --kubeconfig <path-to-management-kubeconfig> edit management`

2. Deploy HMC skipping the default `Management` object creation and provide your own `Management`
configuration:

   * Create `management.yaml` file and configure core components and providers.
   See [Management API](api/v1alpha1/management_types.go).

   * Specify `--create-management=false` controller argument and install HMC:

    If installing using `helm` add the following parameter to the `helm install` command:

    `--set="controller.createManagement=false"`

   * Create `hmc` `Management` object after HMC installation:

    `kubectl --kubeconfig <path-to-management-kubeconfig> create -f management.yaml`


## Cleanup

1. Remove the Management object:

`kubectl delete management.hmc hmc`

> NOTE:
> Make sure you have no HMC ManagedCluster objects left in the cluster prior to Management deletion

2. Remove the `hmc` Helm release:

`helm uninstall hmc -n hmc-system`

3. Remove the `hmc-system` namespace:

`kubectl delete ns hmc-system`
