# Quick Start

## Prerequisites

### Required Tools

Before we begin, deploying Kubernetes clusters on Azure using Project 2A, make sure you have:

1. **Azure CLI (`az`):**  
   The `az` CLI is required to interact with Azure resources. Install it by
   following the
   [Azure CLI installation instructions](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli).

    Run the `az login` command to authenticate your session with Azure.

1. **`kubectl`:**  
   Make sure `kubectl` is installed on your local machine to manage your
   Kubernetes clusters.

    You can follow the
    [official installation guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

### Azure account access

Ensure you have access to an Azure account with the necessary permissions to
manage resources. You will need to register specific resource providers, which are listed below.

### Register resource providers

If you're using a new subscription, register these services
ensure the following resource providers are registered:

- `Microsoft.Compute`
- `Microsoft.Network`
- `Microsoft.ContainerService`
- `Microsoft.ManagedIdentity`
- `Microsoft.Authorization`

To register these providers, you can run the following commands in the Azure CLI:

```bash
az provider register --namespace Microsoft.Compute
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.ManagedIdentity
az provider register --namespace Microsoft.Authorization
```

You can follow the [official documentation guide](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/resource-providers-and-types)
to register the providers.

## Setup the Azure Environment

Before you can create a cluster on Azure, you need to set up credentials.
This involves creating an `AzureClusterIdentity` and a `Service Principal (SP)`
to let CAPZ (Cluster API Azure) communicate with Azure.

### Step 1: Find Your Subscription ID

Open your terminal and log into your Azure account:

```bash
az login
```

List all your Azure subscriptions:

```bash
az account list -o table
```

Look for the Subscription ID of the account you want to use.

Example output:

```diff
Name                     SubscriptionId                        TenantId
-----------------------  -------------------------------------  --------------------------------
My Azure Subscription    12345678-1234-5678-1234-567812345678  87654321-1234-5678-1234-12345678
```

Copy your chosen Subscription ID for the next step.

### Step 2: Create a Service Principal (SP)

The Service Principal is like a password-protected user that CAPZ will use to manage resources on Azure.

In your terminal, run the following command. Make sure to replace <Subscription ID> with the ID you copied earlier:

```bash
az ad sp create-for-rbac --role contributor --scopes="/subscriptions/<Subscription ID>"
```

You will see output like this:

```json
{
 "appId": "12345678-7848-4ce6-9be9-a4b3eecca0ff",
 "displayName": "azure-cli-2024-10-24-17-36-47",
 "password": "12~34~I5zKrL5Kem2aXsXUw6tIig0M~3~1234567",
 "tenant": "12345678-959b-481f-b094-eb043a87570a",
}
```

Copy and save these values somewhere safe, for creating
the `AzureClusterIdentity` object and it's secret:

    
    appId -> This is the Client ID
    password -> This is the Client Secret
    tenant -> This is the Tenant ID
    
Subscription ID which was used to create service principal should be the
same that will be used in the `.spec.config.subscriptionID` field of the
`ManagedCluster` object.

To use `AzureClusterIdentity` it should be referenced in the `Credential`
object. For more details check the [credential section](../credential/main.md).

> NOTE:
> Make sure to save this credentials and treat them like passwords.

Now that you have your `Subscription ID`, `Client ID`, `Client Secret`,
and `Tenant ID`, you can create the AzureClusterIdentity resource,
which will use these credentials to manage your cluster on Azure.

### Create the Azure ClusterIdentity resources

#### Step 1: Create the Secret Object

The Secret stores the clientSecret (password) from the Service Principal.

1. Save the Secret YAML into a file named `az-cluster-identity-secret.yaml`:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
   name: az-cluster-identity-secret
   namespace: hmc-system
   stringData:
   clientSecret: password # Password retrieved from the Service Principal
   type: Opaque
   ```

   Apply the YAML to your cluster using the following command:

   ```bash
   kubectl apply -f az-cluster-identity-secret.yaml
   ```

#### Step 2: Create the AzureClusterIdentity Object

This object defines the credentials CAPZ will use to manage Azure resources.

1. Save the AzureClusterIdentity YAML into a file named `az-cluster-identity.yaml`:

   ```yaml
   apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
   kind: AzureClusterIdentity
   metadata:
   labels:
      clusterctl.cluster.x-k8s.io/move-hierarchy: "true"
   name: az-cluster-identity
   namespace: hmc-system
   spec:
   allowedNamespaces: {}
   clientID: appId # AppId retrieved from the Service Principal
   clientSecret:
      name: az-cluster-identity-secret
      namespace: hmc-system
   tenantID: tenant # TennantID retrieved from the Service Principal
   type: ServicePrincipal
   ```

1. Apply the YAML to your cluster:

   ```bash
      kubectl apply -f az-cluster-identity.yaml
   ```

#### Step 3: Verify the Resources

After applying both YAML files, you can verify that the objects were
created successfully:

1. Check the Secret:

   ```bash
   kubectl get secret az-cluster-identity-secret -n hmc-system
   ```

1. Check the AzureClusterIdentity:

   ```bash
   kubectl get azureclusteridentity az-cluster-identity -n hmc-system
   ```

## Install HMC


To install HMC, first check the latest release at [https://github.com/Mirantis/hmc/tags](https://github.com/Mirantis/hmc/tags).

Then, run the following command to deploy HMC on your cluster:

```bash
kubectl apply -f https://github.com/Mirantis/hmc/releases/download/v0.0.3/install.yaml
```

Wait a moment for the pods to initialize. Once complete, you should see output
similar to the following:


```bash
kubectl get po -n hmc-system
NAMESPACE                           NAME                                                             READY   STATUS              RESTARTS        AGE
cert-manager                        cert-manager-6bcd5c585d-tdtx7                                    1/1     Running             2               14m
cert-manager                        cert-manager-cainjector-df6db5846-6npqg                          1/1     Running             1 (3m31s ago)   14m
cert-manager                        cert-manager-webhook-6ff7fbb9ff-nvfl9                            1/1     Running             2               14m
hmc-system                          helm-controller-76f675f6b7-qwvxv                                 1/1     Running             4               54m
hmc-system                          hmc-cert-manager-7c8bd964b4-l4pls                                1/1     Running             2 (4m1s ago)    54m
hmc-system                          hmc-cert-manager-cainjector-56476c46f9-fx2sk                     1/1     Running             0               54m
hmc-system                          hmc-cert-manager-webhook-69d7fccf68-nzzvq                        1/1     Running             2               54m
hmc-system                          hmc-controller-manager-855fbf9586-f44ch                          1/1     Running             5 (2m44s ago)   54m
hmc-system                          hmc-flux-check-svfzl                                             0/1     Completed           0               54m
hmc-system                          source-controller-5f648d6f5d-fkfxh                               1/1     Running             4 (2m44s ago)   54m
```
Once all the components are installed, check the clustertemplate with:

```bash
kubectl get clustertemplate -n hmc-system
NAME                          VALID
aws-eks-0-0-1                 false
aws-hosted-cp-0-0-2           false
aws-standalone-cp-0-0-2       false
azure-hosted-cp-0-0-2         false
azure-standalone-cp-0-0-2     false
vsphere-hosted-cp-0-0-2       false
vsphere-standalone-cp-0-0-2   false
```
### Enable the cluster template by

#### Step 1

#### Step 2


## Create the Management Server

### Test Management cluster status/health


## Create workload clusters

### Accessing workload cluster





## Deleting everything
  


## Azure machine parameters

Follow the [Azure machine parameters](./machine-parameters.md) guide if you want
to setup/modify the default machine parameters.

## Hosted Control Plane

For more advanced setups, follow the
[Hosted Control Plane (K0smotron) Guide](./hosted-control-plane.md) to
configure a hosted control plane.  
> NOTE:
> In this setup, all control plane components for managed clusters will
> reside within the management cluster, offering centralized control and easier management.
