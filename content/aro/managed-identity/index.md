# ARO Managed Identity Install Guide

The documentation to support installing a new ARO cluster using Managed Identity is spread across a few pages with some inconsistency, this guide attempts to streamline those instructions somewhat. This document does assume some familiarity with ARO and doing ARO installations and is primarily focused on making the additional steps required for building a cluster with Azure Managed Identity easier to accomplish.

## References

1. [The official Microsoft documentation](https://learn.microsoft.com/en-us/azure/openshift/howto-create-openshift-cluster) - Covers all of the steps here but less streamlined and with a reliance on hard-coded values that may need to be changed based on environment
1. [The ARO Managed Identity Blog Post](https://www.redhat.com/en/blog/managed-identity-workload-identity-azure) - More of an overview and an explanation of how Managed Identity works, this article is potentially good background

## Prerequisites

Refer to the ['Prerequisites'](https://learn.microsoft.com/en-us/azure/openshift/howto-create-openshift-cluster#prerequisites) section of the official documentation, it's complete and clear.

## VNet and Subnets

Managed Identity clusters are identical to standard clusters in terms of networking requirements. The script examples below show using the common `10.0.0.0/22` with two `/23` subnets which aligns with the examples from the official documentation, but the chosen CIDR range and subnets sizes for clusters need to take into account interconnection with other VNets and/or on-premises environments as well as sizing around the number of eventual nodes the cluster might contain, as well as adequate address space for any other Azure services that might be on the same VNet. Existing network creation scripts or automation you may have developed should work with ARO MI just the same as they did with standard ARO.

## Create Identities and Assign Roles

There are a few managed identities that need to be created, and each identity needs to be assigned to a managed Role that dictates what it's allowed to do. These roles are build and maintained by RH/MSFT. These are also assigned at a specific scope (typically the subnets the cluster will reside on) to keep those permissions as narrow as possible.

The documentation shows the creation and assignment of these identities with names that are not specific to any individual cluster, the scripts below prepend the identity names for each with the cluster name (which is not the same as the cluster DNS name), which should make figuring out which identities are in use by which cluster easier down the road.

First, there are a number of variables that need to be set.

```bash
# The name of the Azure Resource Group that will hold the Cluster
export RESOURCEGROUP=test-aro-rg

# The ID of the Azure Subscription in which the resources will be created
export SUBSCRIPTION=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
                    
# The name of the cluster
export ARO_CLUSTER_NAME=test-aro

# The setting of a custom DNS name for the cluster is still optional (but recommended) for ARO with Managed Identity
export CLUSTER_DNS_NAME=test-aro.example.domain

```

## VNet and Subnet Creation

```bash

# The name and CIDR ranges should be modified to fit your environment

VNET_NAME=test-aro-vnet
VNET_CIDR="10.66.0.0/22"
CP_SUBNET_NAME=control-plane-subnet
CP_SUBNET_CIDR="10.66.0.0/23"
WORKER_SUBNET_NAME=worker-subnet
WORKER_SUBNET_CIDR="10.66.2.0/23"

az network vnet create --resource-group $RESOURCEGROUP --name $VNET_NAME --address-prefixes $VNET_CIDR
az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name $VNET_NAME --name $CP_SUBNET_NAME --address-prefixes $CP_SUBNET_CIDR
az network vnet subnet create --resource-group $RESOURCEGROUP --vnet-name $VNET_NAME --name $WORKER_SUBNET_NAME --address-prefixes $WORKER_SUBNET_CIDR
az network vnet subnet update --name $CP_SUBNET_NAME --resource-group $RESOURCEGROUP --vnet-name $VNET_NAME --private-link-service-network-policies Disabled

```

```bash

# Create the core cluster identity, and save it's ID for later use
az identity create --resource-group $RESOURCEGROUP --name "${ARO_CLUSTER_NAME}-cluster"
export MI_ARO_CLUSTER_ID=$(az identity show --resource-group $RESOURCEGROUP --name "${ARO_CLUSTER_NAME}-cluster" --query principalId -o tsv)

# Create the operator identities
az identity create --resource-group $RESOURCEGROUP --name "${ARO_CLUSTER_NAME}-cloud-controller-manager"
az identity create --resource-group $RESOURCEGROUP --name "${ARO_CLUSTER_NAME}-ingress"
az identity create --resource-group $RESOURCEGROUP --name "${ARO_CLUSTER_NAME}-machine-api"
az identity create --resource-group $RESOURCEGROUP --name "${ARO_CLUSTER_NAME}-disk-csi-driver"
az identity create --resource-group $RESOURCEGROUP --name "${ARO_CLUSTER_NAME}-cloud-network-config"
az identity create --resource-group $RESOURCEGROUP --name "${ARO_CLUSTER_NAME}-image-registry"
az identity create --resource-group $RESOURCEGROUP --name "${ARO_CLUSTER_NAME}-file-csi-driver"
az identity create --resource-group $RESOURCEGROUP --name "${ARO_CLUSTER_NAME}-aro-operator"

# Retrieve and assign the built-in Azure Red Hat OpenShift Federated Credential Role
export ARO_FED_CRED_ROLE_ID=$(az role definition list --name "Azure Red Hat OpenShift Federated Credential" --query "[0].id" -o tsv)

# Assign the roles scoped to each identity to the cluster identity
az role assignment create \
  --assignee-object-id $MI_ARO_CLUSTER_ID \
  --assignee-principal-type ServicePrincipal \
  --role "$ARO_FED_CRED_ROLE_ID" \
  --scope "/subscriptions/$SUBSCRIPTION/resourcegroups/$RESOURCEGROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/${ARO_CLUSTER_NAME}-aro-operator"

az role assignment create \
  --assignee-object-id $MI_ARO_CLUSTER_ID \
  --assignee-principal-type ServicePrincipal \
  --role "$ARO_FED_CRED_ROLE_ID" \
  --scope "/subscriptions/$SUBSCRIPTION/resourcegroups/$RESOURCEGROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/${ARO_CLUSTER_NAME}-cloud-controller-manager" 

az role assignment create \
  --assignee-object-id ${MI_ARO_CLUSTER_ID} \
  --assignee-principal-type ServicePrincipal \
  --role "$ARO_FED_CRED_ROLE_ID" \
  --scope "/subscriptions/$SUBSCRIPTION/resourcegroups/$RESOURCEGROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/${ARO_CLUSTER_NAME}-ingress" 

az role assignment create \
  --assignee-object-id ${MI_ARO_CLUSTER_ID} \
  --assignee-principal-type ServicePrincipal \
  --role "$ARO_FED_CRED_ROLE_ID" \
  --scope "/subscriptions/$SUBSCRIPTION/resourcegroups/$RESOURCEGROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/${ARO_CLUSTER_NAME}-machine-api" 

az role assignment create \
  --assignee-object-id ${MI_ARO_CLUSTER_ID} \
  --assignee-principal-type ServicePrincipal \
  --role "$ARO_FED_CRED_ROLE_ID" \
  --scope "/subscriptions/$SUBSCRIPTION/resourcegroups/$RESOURCEGROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/${ARO_CLUSTER_NAME}-disk-csi-driver" 

az role assignment create \
  --assignee-object-id ${MI_ARO_CLUSTER_ID} \
  --assignee-principal-type ServicePrincipal \
  --role "$ARO_FED_CRED_ROLE_ID" \
  --scope "/subscriptions/$SUBSCRIPTION/resourcegroups/$RESOURCEGROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/${ARO_CLUSTER_NAME}-cloud-network-config" 

az role assignment create \
  --assignee-object-id ${MI_ARO_CLUSTER_ID} \
  --assignee-principal-type ServicePrincipal \
  --role "$ARO_FED_CRED_ROLE_ID" \
  --scope "/subscriptions/$SUBSCRIPTION/resourcegroups/$RESOURCEGROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/${ARO_CLUSTER_NAME}-image-registry" 

az role assignment create \
  --assignee-object-id ${MI_ARO_CLUSTER_ID} \
  --assignee-principal-type ServicePrincipal \
  --role "$ARO_FED_CRED_ROLE_ID" \
  --scope "/subscriptions/$SUBSCRIPTION/resourcegroups/$RESOURCEGROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/${ARO_CLUSTER_NAME}-file-csi-driver"

# assign vnet-level permissions for operators that require it, and subnets-level permission for operators that require it 

az role assignment create \
  --assignee-object-id "$(az identity show --resource-group $RESOURCEGROUP --name ${ARO_CLUSTER_NAME}-cloud-controller-manager --query principalId -o tsv)" \
  --assignee-principal-type ServicePrincipal \
  --role "$(az role definition list --name 'Azure Red Hat OpenShift Cloud Controller Manager' --query '[0].id' -o tsv)" \
  --scope "/subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCEGROUP/providers/Microsoft.Network/virtualNetworks/$VNET_NAME/subnets/$CP_SUBNET_NAME"

az role assignment create \
  --assignee-object-id "$(az identity show --resource-group $RESOURCEGROUP --name ${ARO_CLUSTER_NAME}-cloud-controller-manager --query principalId -o tsv)" \
  --assignee-principal-type ServicePrincipal \
  --role "$(az role definition list --name 'Azure Red Hat OpenShift Cloud Controller Manager' --query '[0].id' -o tsv)" \
  --scope "/subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCEGROUP/providers/Microsoft.Network/virtualNetworks/$VNET_NAME/subnets/$WORKER_SUBNET_NAME" 

az role assignment create \
  --assignee-object-id "$(az identity show --resource-group $RESOURCEGROUP --name ${ARO_CLUSTER_NAME}-ingress --query principalId -o tsv)" \
  --assignee-principal-type ServicePrincipal \
  --role "$(az role definition list --name 'Azure Red Hat OpenShift Cluster Ingress Operator' --query '[0].id' -o tsv)" \
  --scope "/subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCEGROUP/providers/Microsoft.Network/virtualNetworks/$VNET_NAME/subnets/$CP_SUBNET_NAME"

az role assignment create \
    --assignee-object-id "$(az identity show --resource-group $RESOURCEGROUP --name ${ARO_CLUSTER_NAME}-ingress --query principalId -o tsv)" \
    --assignee-principal-type ServicePrincipal \
    --role "$(az role definition list --name 'Azure Red Hat OpenShift Cluster Ingress Operator' --query '[0].id' -o tsv)" \
    --scope "/subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCEGROUP/providers/Microsoft.Network/virtualNetworks/$VNET_NAME/subnets/$WORKER_SUBNET_NAME"

az role assignment create \
    --assignee-object-id "$(az identity show --resource-group $RESOURCEGROUP --name ${ARO_CLUSTER_NAME}-machine-api --query principalId -o tsv)" \
    --assignee-principal-type ServicePrincipal \
    --role "$(az role definition list --name 'Azure Red Hat OpenShift Machine API Operator' --query '[0].id' -o tsv)" \
    --scope "/subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCEGROUP/providers/Microsoft.Network/virtualNetworks/$VNET_NAME/subnets/$CP_SUBNET_NAME" 

az role assignment create \
    --assignee-object-id "$(az identity show --resource-group $RESOURCEGROUP --name ${ARO_CLUSTER_NAME}-machine-api --query principalId -o tsv)" \
    --assignee-principal-type ServicePrincipal \
    --role "$(az role definition list --name 'Azure Red Hat OpenShift Machine API Operator' --query '[0].id' -o tsv)" \
    --scope "/subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCEGROUP/providers/Microsoft.Network/virtualNetworks/$VNET_NAME/subnets/$WORKER_SUBNET_NAME" 

az role assignment create \
    --assignee-object-id "$(az identity show --resource-group $RESOURCEGROUP --name ${ARO_CLUSTER_NAME}-cloud-network-config --query principalId -o tsv)" \
    --assignee-principal-type ServicePrincipal \
    --role "$(az role definition list --name 'Azure Red Hat OpenShift Network Operator' --query '[0].id' -o tsv)" \
    --scope "/subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCEGROUP/providers/Microsoft.Network/virtualNetworks/$VNET_NAME" 

az role assignment create \
    --assignee-object-id "$(az identity show --resource-group $RESOURCEGROUP --name ${ARO_CLUSTER_NAME}-file-csi-driver --query principalId -o tsv)" \
    --assignee-principal-type ServicePrincipal \
    --role "$(az role definition list --name 'Azure Red Hat OpenShift File Storage Operator' --query '[0].id' -o tsv)" \
    --scope "/subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCEGROUP/providers/Microsoft.Network/virtualNetworks/$VNET_NAME/subnets/$CP_SUBNET_NAME" 

az role assignment create \
    --assignee-object-id "$(az identity show --resource-group $RESOURCEGROUP --name ${ARO_CLUSTER_NAME}-file-csi-driver --query principalId -o tsv)" \
    --assignee-principal-type ServicePrincipal \
    --role "$(az role definition list --name 'Azure Red Hat OpenShift File Storage Operator' --query '[0].id' -o tsv)" \
    --scope "/subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCEGROUP/providers/Microsoft.Network/virtualNetworks/$VNET_NAME/subnets/$WORKER_SUBNET_NAME" 

az role assignment create \
    --assignee-object-id "$(az identity show --resource-group $RESOURCEGROUP --name ${ARO_CLUSTER_NAME}-aro-operator --query principalId -o tsv)" \
    --assignee-principal-type ServicePrincipal \
    --role "$(az role definition list --name 'Azure Red Hat OpenShift Service Operator' --query '[0].id' -o tsv)" \
    --scope "/subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCEGROUP/providers/Microsoft.Network/virtualNetworks/$VNET_NAME/subnets/$CP_SUBNET_NAME" 

az role assignment create \
    --assignee-object-id "$(az identity show --resource-group $RESOURCEGROUP --name ${ARO_CLUSTER_NAME}-aro-operator --query principalId -o tsv)" \
    --assignee-principal-type ServicePrincipal \
    --role "$(az role definition list --name 'Azure Red Hat OpenShift Service Operator' --query '[0].id' -o tsv)" \
    --scope "/subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCEGROUP/providers/Microsoft.Network/virtualNetworks/$VNET_NAME/subnets/$WORKER_SUBNET_NAME" 

az role assignment create \
    --assignee-object-id "$(az ad sp list --display-name "Azure Red Hat OpenShift RP" --query '[0].id' -o tsv)" \
    --assignee-principal-type ServicePrincipal \
    --role "$(az role definition list --name 'Network Contributor' --query '[0].id' -o tsv)" \
    --scope "/subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCEGROUP/providers/Microsoft.Network/virtualNetworks/$VNET_NAME"

```


## Create the ARO cluster

```bash

# The '--version' flag is not required (or version specific) when using Managed Identity

az aro create \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER_NAME \
  --vnet $VNET_NAME \
  --master-subnet $CP_SUBNET_NAME \
  --worker-subnet $WORKER_SUBNET_NAME \
  --version 4.16.30 \
  --domain $CLUSTER_DNS_NAME \
  --enable-managed-identity \
  --assign-cluster-identity ${ARO_CLUSTER_NAME}-cluster \
  --assign-platform-workload-identity file-csi-driver ${ARO_CLUSTER_NAME}-file-csi-driver \
  --assign-platform-workload-identity cloud-controller-manager ${ARO_CLUSTER_NAME}-cloud-controller-manager \
  --assign-platform-workload-identity ingress ${ARO_CLUSTER_NAME}-ingress \
  --assign-platform-workload-identity image-registry ${ARO_CLUSTER_NAME}-image-registry \
  --assign-platform-workload-identity machine-api ${ARO_CLUSTER_NAME}-machine-api \
  --assign-platform-workload-identity cloud-network-config ${ARO_CLUSTER_NAME}-cloud-network-config \
  --assign-platform-workload-identity aro-operator ${ARO_CLUSTER_NAME}-aro-operator \
  --assign-platform-workload-identity disk-csi-driver ${ARO_CLUSTER_NAME}-disk-csi-driver \
  --pull-secret $(cat $OCP_PULL_SECRET)

```

Post-install steps are identical to any other ARO cluster, typically starting with making sure your DNS resolves to the API and default Ingress IPs correctly.