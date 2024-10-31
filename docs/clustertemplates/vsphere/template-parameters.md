# vSphere cluster template parameters

### Image template

You can use pre-buit image templates from [CAPV project](https://github.com/kubernetes-sigs/cluster-api-provider-vsphere/blob/main/README.md#kubernetes-versions-with-published-ovas)
or build your own.

When building your own image make sure that vmware tools and cloud-init are
installed and properly configured.

You can follow [official open-vm-tools guide](https://docs.vmware.com/en/VMware-Tools/11.0.0/com.vmware.vsphere.vmwaretools.doc/GUID-C48E1F14-240D-4DD1-8D4C-25B6EBE4BB0F.html)
on how to correctly install vmware-tools.

When setting up cloud-init you can refer to [official docs](https://cloudinit.readthedocs.io/en/latest/index.html)
and specifically [vmware datasource docs](https://cloudinit.readthedocs.io/en/latest/reference/datasources/vmware.html)
for extended information regarding cloud-init on vSphere.

### vSphere network

When creating network make sure that it has DHCP service.

Also make sure that the part of your network is out of DHCP range (e.g. network
172.16.0.0/24 with DHCP range 172.16.0.100-172.16.0.254). This is needed to make
sure that LB services will not create any IP conflicts in the network.

## ManagedCluster parameters

To deploy managed cluster a number of parameters should be passed to the
`ManagedCluster` object.

### Parameter list

The following is the list of vSphere specific parameters, which are _required_
for successful cluster creation.

| Parameter                           | Example                               | Description                        |
|-------------------------------------|---------------------------------------|------------------------------------|
| `.spec.config.vsphere.server`       | `vcenter.example.com`                 | Address of the vSphere server      |
| `.spec.config.vsphere.thumbprint`   | `"00:00:00"`                          | Certificate thumbprint             |
| `.spec.config.vsphere.datacenter`   | `DC`                                  | Datacenter name                    |
| `.spec.config.vsphere.datastore`    | `/DC/datastore/DS`                    | Datastore path                     |
| `.spec.config.vsphere.resourcePool` | `/DC/host/vCluster/Resources/ResPool` | Resource pool path                 |
| `.spec.config.vsphere.folder`       | `/DC/vm/example`                      | vSphere folder path                |

_You can check [machine parameters](machine-parameters.md) for machine specific
parameters._

To obtain vSphere certificate thumbprint you can use the following command:

```bash
curl -sw %{certs} https://vcenter.example.com | openssl x509 -sha256 -fingerprint -noout | awk -F '=' '{print $2}'
```

## Example of ManagedCluster CR

With all above parameters provided your `ManagedCluster` can look like this:

```yaml
apiVersion: hmc.mirantis.com/v1alpha1
kind: ManagedCluster
metadata:
  name: cluster-1
spec:
  template: vsphere-standalone-cp-0-0-2
  credential: vsphere-credential
  config:
    clusterIdentity:
      name: vsphere-cluster-identity
    vsphere:
      server: vcenter.example.com
      thumbprint: "00:00:00"
      datacenter: "DC"
      datastore: "/DC/datastore/DC"
      resourcePool: "/DC/host/vCluster/Resources/ResPool"
      folder: "/DC/vm/example"
    controlPlaneEndpointIP: "172.16.0.10"

    controlPlane:
      ssh:
        user: ubuntu
        publicKey: |
          ssh-rsa AAA...
      rootVolumeSize: 50
      cpus: 2
      memory: 4096
      vmTemplate: "/DC/vm/template"
      network: "/DC/network/Net"

    worker:
      ssh:
        user: ubuntu
        publicKey: |
          ssh-rsa AAA...
      rootVolumeSize: 50
      cpus: 2
      memory: 4096
      vmTemplate: "/DC/vm/template"
      network: "/DC/network/Net"
```


## SSH

Currently SSH configuration on vSphere expects that user is already created
during template creation. Because of that you must pass username along with SSH
public key to configure SSH access.


SSH public key can be passed to `.spec.config.ssh.publicKey` (in case of
hosted CP) parameter or `.spec.config.controlPlane.ssh.publicKey` and
`.spec.config.worker.ssh.publicKey` parameters (in case of standalone CP) of the
`ManagedCluster` object.

SSH public key must be passed literally as a string.

Username can be passed to `.spec.config.controlPlane.ssh.user`,
`.spec.config.worker.ssh.user` or `.spec.config.ssh.user` depending on you
deployment model.

## VM resources

The following parameters are used to define VM resources:

| Parameter         | Example | Description                                                          |
|-------------------|---------|----------------------------------------------------------------------|
| `.rootVolumeSize` | `50`    | Root volume size in GB (can't be less than one defined in the image) |
| `.cpus`           | `2`     | Number of CPUs                                                       |
| `.memory`         | `4096`  | Memory size in MB                                                    |

The resource parameters are the same for hosted and standalone CP deployments,
but they are positioned differently in the spec, which means that they're going to:

- `.spec.config` in case of hosted CP deployment.
- `.spec.config.controlPlane` in in case of standalone CP for control plane
  nodes.
- `.spec.config.worker` in in case of standalone CP for worker nodes.

## VM Image and network

To provide image template path and network path the following parameters must be
used:

| Parameter     | Example           | Description         |
|---------------|-------------------|---------------------|
| `.vmTemplate` | `/DC/vm/template` | Image template path |
| `.network`    | `/DC/network/Net` | Network path        |

As with resource parameters the position of these parameters in the
`ManagedCluster` depends on deployment type and these parameters are used in:

- `.spec.config` in case of hosted CP deployment.
- `.spec.config.controlPlane` in in case of standalone CP for control plane
  nodes.
- `.spec.config.worker` in in case of standalone CP for worker nodes.
