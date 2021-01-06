# Azure AKS and Key vault integration based on service principal for retreiving secrets in key vault

# Global variables

```
export suffix=001
export AKS_GRP=aks-$suffix
export AZ_KV_GRP=$AKS_GRP
export LOC=westeurope
export AKS_NAME=aks-$suffix
export AZ_KV_NAME=kv-$suffix
export SP_NAME=az-aks-kv-sp-$suffix
export SP_AKS_SECRET=secrets-store-creds
export TENANTID=$(az account show --query tenantId)
export SUBID=$(az account show --query id)
```

# Aks Group create 

``` 
$ az group create -n AKS_GRP -l $LOC
``` 

# create the service principal

```
SPObj=$(az ad sp create-for-rbac --name $SP_NAME --skip-assignment)
```

# Keep the appid and secret in a safe place,we shall need them later. If jq not installed in the machine, install it.

```
export SP_CLIENT_ID=echo $SPObj | jq -r '.appId'
export SP_CLIENT_SECRET=echo $SPObj | jq -r '.password'
```

# Aks Cluster create 

``` 
$ az aks create -n $AKS_NAME -g $AKS_GRP --node-count 2 --network-plugin azure --generate-ssh-keys 
``` 

# Get AKS credentials to work with kubectl 
``` 
az aks get-credentials --name $AKS_NAME -g $AKS_GRP
``` 

# Make sure kubernetes version is >= 1.16 

```
kubectl get nodes -o wide 
``` 

# Helm repo add for csi-secrets-store-driver (optional)
``` 
$ helm repo add secrets-store-csi-driver https://raw.githubusercontent.com/kubernetes-sigs/secrets-store-csi-driver/master/charts 
$ helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver 
``` 

# Helm repo for cs-secret-store-driver-azure-provider. 
***This helm repo provide both the csi-secrets-store-driver as well as csi-secrets-store-driver-azure-provider as well***

``` 
$ helm repo add csi-secrets-store-provider-azure https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/master/charts 
$ helm install csi-secrets-store-provider-azure/csi-secrets-store-provider-azure --generate-name 

``` 

# Create azure key vault 

``` 
az keyvault create --name $AZ_KV_NAME --resource-group $AZ_KV_GRP
``` 

# Assign service principal a reader role for the keyvault

```
az role assignment create --role Reader --assignee $SP_CLIENT_ID --scope /subscriptions/$SUBID/resourcegroups/$AZ_KV_GRP/providers/Microsoft.KeyVault/vaults/$AZ_KV_NAME

```

# Then assign get policies for the secrets && create two key vault secrets with name as dbusername and dbpassword. Provide some secret values

```
az keyvault set-policy -n $AZ_KV_NAME --secret-permissions get --spn $SP_CLIENT_ID
```

# Create secret based on the appid and appsecret of the service principal

```
kubectl create secret generic $SP_AKS_SECRET --from-literal clientid=$SP_CLIENT_ID --from-literal clientsecret=$SP_CLIENT_SECRET
```

# Populate all the respective details in storageproviderclass yaml file && run 

```
kubectl apply -f azure-kv-provider.yaml
```

# Lastly deploy a sample test nginx deployment to test

```
kubectl apply -f nginx-deployment.yaml 
```

# wait for nginx pod to be up and running and then test the secrets are mounted or not

```
kubectl exec nginx-app -- ls /mnt/secrets/DB_PASSWORD
kubectl exec nginx-app -- cat /mnt/secrets/DB_PASSWORD
```

