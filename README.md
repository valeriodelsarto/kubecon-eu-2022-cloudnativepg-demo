# Cloudnative-PG AKS/EKS Demo

## Concepts

Azure: https://azure.github.io/azure-workload-identity/docs/introduction.html

AWS: https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html

## Azure environment

### Prerequisites - Enabling Azure AKS OIDC Support (Preview)

```bash
az feature register --name EnableOIDCIssuerPreview --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.ContainerService
az extension add --name aks-preview
az extension update --name aks-preview
```

### Prerequisites - Installing Azure AD Workload CLI (azwi)

```bash
brew install Azure/azure-workload-identity/azwi
```

### Creating the AKS Cluster

```bash
az group create -l westeurope -n cloudnativepg-kubecon-demo
az network vnet create -l westeurope --address-prefixes 10.0.0.0/16 -n cloudnativepg-kubecon-demo -g cloudnativepg-kubecon-demo --subnet-name cloudnativepg-kubecon-demo --subnet-prefixes 10.0.0.0/16
az aks create -l westeurope -g cloudnativepg-kubecon-demo -n cloudnativepg-kubecon-demo-aks -y \
    --enable-managed-identity \
    --kubernetes-version 1.23.5 \
    --node-vm-size Standard_DS2_v2 \
    --zones 1 2 3 \
    --node-count 1 \
    --enable-cluster-autoscaler \
    --max-count 3 \
    --min-count 1 \
    --network-plugin azure \
    --vnet-subnet-id /subscriptions/<YOUR_SUBSCRIPTION_ID>/resourceGroups/cloudnativepg-kubecon-demo/providers/Microsoft.Network/virtualNetworks/cloudnativepg-kubecon-demo/subnets/cloudnativepg-kubecon-demo \
    --docker-bridge-address 172.17.0.1/16 \
    --dns-service-ip 10.1.0.10 \
    --service-cidr 10.1.0.0/16 \
    --enable-oidc-issuer
az aks get-credentials -g cloudnativepg-kubecon-demo --name cloudnativepg-kubecon-demo-aks --overwrite-existing
```

### Creating the Azure Storage Account

```bash
az storage account create -l westeurope --name cloudnativepgkubecondemo -g cloudnativepg-kubecon-demo \
  --sku Standard_LRS
```

### Configuring the AKS Cluster with Azure AD Workload Identity

```bash
export AZURE_TENANT_ID="$(az account show -s $(az account show --query id -otsv) --query tenantId -otsv)"
helm repo add azure-workload-identity https://azure.github.io/azure-workload-identity/charts
helm repo update
helm install workload-identity-webhook azure-workload-identity/workload-identity-webhook \
   --namespace azure-workload-identity-system \
   --create-namespace \
   --set azureTenantID="${AZURE_TENANT_ID}"

export APPLICATION_NAME="cloudnative-pg"
export SERVICE_ACCOUNT_NAMESPACE="default"
export SERVICE_ACCOUNT_NAME="cloudnative-pg"
export SERVICE_ACCOUNT_ISSUER="$(az aks show -g cloudnativepg-kubecon-demo --name cloudnativepg-kubecon-demo-aks --query "oidcIssuerProfile.issuerUrl" -otsv)"
azwi serviceaccount create phase app --aad-application-name "${APPLICATION_NAME}"
export APPLICATION_CLIENT_ID="$(az ad sp list --display-name "${APPLICATION_NAME}" --query '[0].appId' -otsv)"
export STORAGE_ACCOUNT_ID="$(az storage account show --name cloudnativepgkubecondemo -g cloudnativepg-kubecon-demo --query id -otsv)"
az role assignment create --role "Storage Blob Data Contributor" --assignee "$APPLICATION_CLIENT_ID" --scope "$STORAGE_ACCOUNT_ID"

azwi serviceaccount create phase sa \
  --aad-application-name "${APPLICATION_NAME}" \
  --service-account-namespace "${SERVICE_ACCOUNT_NAMESPACE}" \
  --service-account-name "${SERVICE_ACCOUNT_NAME}"

azwi serviceaccount create phase federated-identity \
  --aad-application-name "${APPLICATION_NAME}" \
  --service-account-namespace "${SERVICE_ACCOUNT_NAMESPACE}" \
  --service-account-name "${SERVICE_ACCOUNT_NAME}" \
  --service-account-issuer-url "${SERVICE_ACCOUNT_ISSUER}"
```

### Deploying CloudNativePG operator `dev/cnpg-131` branch on Azure

```bash
kubectx cloudnativepg-kubecon-demo-aks
kustomize build config/default \
  | sed 's/controller:latest/ghcr.io\/cloudnative-pg\/cloudnative-pg-testing:pr-132/g' \
  | k apply -f -
```

### Creating a PG cluster on AKS

```bash
cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: cloudnative-pg
spec:
  instances: 3
  storage:
    storageClass: managed-premium
    size: 1Gi
  backup:
    barmanObjectStore:
      destinationPath: https://cloudnativepgkubecondemo.blob.core.windows.net/cloudnative-pg/
      azureCredentials:
        inheritFromAzureAD: true
    retentionPolicy: "30d"
EOF
```

### Creating a PG cluster backup on AKS

```bash
cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: backup-cloudnative-pg
spec:
  cluster:
    name: cloudnative-pg
EOF
```

## AWS environment

### Creating the EKS Cluster

```bash
eksctl create cluster -f kubecon/cloudnativepg-kubecon-demo-eks.yaml
```

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: cloudnativepg-kubecon-demo-eks
  region: eu-west-1
  version: "1.22"

iam:
  withOIDC: true
  serviceAccounts:
  - metadata:
      name: cloudnative-pg
      namespace: default
      labels: {aws-usage: "cloudnative-pg"}
    attachPolicy:
      Version: "2012-10-17"
      Statement:
      - Effect: Allow
        Action:
        - "s3:AbortMultipartUpload"
        - "s3:DeleteObject"
        - "s3:GetObject"
        - "s3:ListBucket"
        - "s3:PutObject"
        - "s3:PutObjectTagging"
        Resource:
        - "arn:aws:s3:::cloudnativepg-kubecon-demo"
        - "arn:aws:s3:::cloudnativepg-kubecon-demo/*"

managedNodeGroups:
  - name: cloudnativepg-kubecon-demo-eks-ng
    minSize: 1
    maxSize: 3
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/cloudnativepg-kubecon-demo-eks: "owned"
    desiredCapacity: 1

addons:
- name: vpc-cni
  version: latest
  attachPolicyARNs:
    - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
- name: coredns
  version: latest
- name: kube-proxy
  version: latest
```

### Creating the S3 Bucket

```bash
aws s3api create-bucket --bucket cloudnativepg-kubecon-demo --region eu-west-1 \
  --create-bucket-configuration LocationConstraint=eu-west-1 --no-cli-pager
```

### Installing the CloudNativePG operator on AWS

```bash
kubectx valerio.delsarto@enterprisedb.com@cloudnativepg-kubecon-demo-eks.eu-west-1.eksctl.io
kubectl apply -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/main/releases/cnpg-1.15.0.yaml
```

### Creating a PG cluster on EKS

```bash
cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: cloudnative-pg
spec:
  instances: 3
  storage:
    storageClass: gp2
    size: 1Gi
  backup:
    barmanObjectStore:
      destinationPath: s3://cloudnativepg-kubecon-demo/
      s3Credentials:
        inheritFromIAMRole: true
    retentionPolicy: "30d"
EOF
```

### Creating a PG cluster backup on EKS

```bash
cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: backup-cloudnative-pg
spec:
  cluster:
    name: cloudnative-pg
EOF
```

### Links

[Azure Portal Resource Group](https://portal.azure.com/#@<YOUR_TENANT_NAME>.onmicrosoft.com/resource/subscriptions/<YOUR_SUBSCRIPTION_ID>/resourceGroups/cloudnativepg-kubecon-demo/overview)

[Azure Portal Storage Account IAM](https://portal.azure.com/#@<YOUR_TENANT_NAME>.onmicrosoft.com/resource/subscriptions/<YOUR_SUBSCRIPTION_ID>/resourceGroups/cloudnativepg-kubecon-demo/providers/Microsoft.Storage/storageAccounts/cloudnativepgkubecondemo/iamAccessControl)

[AWS EKS cluster](https://eu-west-1.console.aws.amazon.com/eks/home?region=eu-west-1#/clusters/cloudnativepg-kubecon-demo-eks)

[AWS S3 bucket](https://s3.console.aws.amazon.com/s3/buckets/cloudnativepg-kubecon-demo?region=eu-west-1&tab=objects)

### Removing Azure and AWS resources

```bash
# Azure
kubectx cloudnativepg-kubecon-demo-aks
kubectl delete cluster cloudnative-pg
azwi serviceaccount delete --aad-application-name "${APPLICATION_NAME}" \
  --service-account-namespace "${SERVICE_ACCOUNT_NAMESPACE}" \
  --service-account-name "${SERVICE_ACCOUNT_NAME}" \
  --service-account-issuer-url "${SERVICE_ACCOUNT_ISSUER}" \
  --skip-phases role-assignment
az aks delete -g cloudnativepg-kubecon-demo -n cloudnativepg-kubecon-demo-aks -y
az network vnet delete -g cloudnativepg-kubecon-demo -n cloudnativepg-kubecon-demo
az storage account delete -n cloudnativepgkubecondemo -g cloudnativepg-kubecon-demo -y
az group delete -n cloudnativepg-kubecon-demo -y

# AWS
kubectx valerio.delsarto@enterprisedb.com@cloudnativepg-kubecon-demo-eks.eu-west-1.eksctl.io
kubectl delete cluster cloudnative-pg
eksctl delete cluster -f kubecon/cloudnativepg-kubecon-demo-eks.yaml
aws s3 rm s3://cloudnativepg-kubecon-demo --recursive
aws s3api delete-bucket --bucket cloudnativepg-kubecon-demo --region eu-west-1
```
