# Azure Web App, PostgreSQL and Storage Account via Crossplane

This repository contains the Crossplane code to create resources in Azure to host a webapp on Azure solution for developers.
Everything is created using Crossplane and the Azure provider from Upbound.

## Install

Follow these instructions to install Crossplane and the Azure provider on your AKS cluster.

```bash
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
LOCATION=westeurope
CLUSTER_NAME=aks-cluster
RESOURCE_GROUP_NAME=aks-cluster-rg

# Create a new resource group
az group create \
--name $RESOURCE_GROUP_NAME \
--location $LOCATION

# Create a new AKS cluster (aka our "control-cluster" to host Crossplane)
az aks create \
--resource-group $RESOURCE_GROUP_NAME \
--name $CLUSTER_NAME \
--node-count 2 \
--node-vm-size Standard_D2s_v6 \
--enable-managed-identity \
--ssh-access disabled \
--location $LOCATION

# Get the AKS credentials
az aks get-credentials \
--resource-group $RESOURCE_GROUP_NAME \
--name $CLUSTER_NAME

# Install Crossplane
helm repo add \
crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane \
crossplane-stable/crossplane \
--namespace crossplane-system \
--create-namespace

# Install the Crossplane Providers for Azure
kubectl crossplane install provider upbound/provider-azure-web:v1.11.2
kubectl crossplane install provider upbound/provider-azure-dbforpostgresql:v1.11.2
kubectl crossplane install provider upbound/provider-azure-storage:v1.11.2

# Create a Kubernetes secret for Azure
az ad sp create-for-rbac \
--sdk-auth \
--role Owner \
--scopes /subscriptions/$SUBSCRIPTION_ID > azure-credentials.json

kubectl create secret \
generic azure-secret \
-n crossplane-system \
--from-file=creds=./azure-credentials.json

cat <<EOF | kubectl apply -f -
apiVersion: azure.upbound.io/v1beta1
metadata:
  name: default
kind: ProviderConfig
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: azure-secret
      key: creds
EOF

# Install our resources
kubectl apply -f xrd/compositeazureservice.yaml
kubectl apply -f compositions/azureservice-composition.yaml

# Install a sample application
kubectl apply -f examples/first-service-claim.yaml
kubectl apply -f examples/second-service-claim.yaml
```

## Monitor status of resources

You can monitor the status of the resources created by Crossplane using the following commands:

```bash
# View the status of the application
kubectl get resourcegroups.azure.upbound.io
kubectl get serviceplans.web.azure.upbound.io
kubectl get windowswebapps.web.azure.upbound.io
kubectl get accounts.storage.azure.upbound.io
kubectl get flexibleservers.dbforpostgresql.azure.upbound.io
```

## Tear down

To tear down the resources created by this repository, run the following command:

```bash
# Delete the application - uncomment the following line
kubectl delete -f examples/first-service-claim.yaml
kubectl delete -f examples/second-service-claim.yaml

# Delete the Crossplane resources
kubectl delete -f compositions/azureservice-composition.yaml
kubectl delete -f xrd/compositeazureservice.yaml

# Delete the AKS cluster
az aks delete \
--name $CLUSTER_NAME \
--resource-group $RESOURCE_GROUP_NAME \
--yes \
--no-wait

# Delete the resource group
az group delete \
--name $RESOURCE_GROUP_NAME \
--yes \
--no-wait

# Remove entries from local kubeconfig
kubectl config delete-context $CLUSTER_NAME
kubectl config delete-cluster $CLUSTER_NAME
```
