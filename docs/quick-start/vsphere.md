# vSphere Quick Start

## Prerequisites

1. `kubectl` CLI installed locally.
2. vSphere instance version `6.7.0` or higher.
3. vSphere account with appropriate privileges.
4. Image template.
5. vSphere network with DHCP enabled.

### vSphere privileges

To function properly the user assigned to vSphere provider should be able to
manipulate vSphere resources. The following is the general overview of the
required privileges:

- `Virtual machine` - full permissions are required
- `Network` - `Assign network` is sufficient
- `Datastore` - it should be possible for user to manipulate virtual machine
  files and metadata

In addition to that specific CSI driver permissions are required see
[the official doc](https://docs.vmware.com/en/VMware-vSphere-Container-Storage-Plug-in/2.0/vmware-vsphere-csp-getting-started/GUID-0AB6E692-AA47-4B6A-8CEA-38B754E16567.html)
to get more information on CSI specific permissions.


## Cluster Identity

To provide credentials for CAPI vSphere provider (CAPV) the
`VSphereClusterIdentity` resource must be created. This should be done before
provisioning any clusters.

To create cluster identity you'll only need username and password for your
vSphere instance.

The example of the objects can be found below:

**Secret**:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vsphere-cluster-identity-secret
  namespace: hmc-system
stringData:
  username: user
  password: Passw0rd
```

**VsphereClusterIdentity**:

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: VSphereClusterIdentity
metadata:
  name: vsphere-cluster-identity
spec:
  secretName: vsphere-cluster-identity-secret
  allowedNamespaces:
    selector:
      matchLabels: {}
```

To be used for the cluster creation.`VsphereClusterIdentity` then should be
referenced in the `Credential` object.