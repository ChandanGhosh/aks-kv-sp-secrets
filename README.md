# Azure AKS and Key vault integration based on service principal for retreiving secrets in key vault

#### Global variables
###### *If jq not installed in the machine, install it.*

```
export SUFFIX=$(echo $RANDOM)
export AKS_GRP=aks-grp-$SUFFIX
export AZ_KV_GRP=kv-grp-$SUFFIX
export LOC=westeurope
export AKS_NAME=aks-$SUFFIX
export AZ_KV_NAME=kv-$SUFFIX
export SP_NAME=az-aks-kv-sp-$SUFFIX
export CSI_STORAGE_CLASS_NAME=nginx-csi-storage-class
export SP_AKS_SECRET=secrets-store-creds
export CSI_NAMESPACE=azure-cs-driver
export TENANTID=$(echo $(az account show) | jq -r '.tenantId')
export SUBID=$(echo $(az account show) | jq -r '.id')
```


#### create the service principal
```
SPObj=$(az ad sp create-for-rbac --name $SP_NAME --skip-assignment)
```

#### Keep the appid and secret in a safe place,we shall need them later.
```
export SP_CLIENT_ID=$(echo $SPObj | jq -r '.appId')
export SP_CLIENT_SECRET=$(echo $SPObj | jq -r '.password')
```

#### Aks Group create 
``` 
az group create -n $AKS_GRP -l $LOC --tags label=$SUFFIX
``` 

#### Aks Cluster create 

``` 
az aks create -n $AKS_NAME -g $AKS_GRP --node-count 2 --network-plugin azure --generate-ssh-keys 
``` 

#### Get AKS credentials to work with kubectl 
``` 
az aks get-credentials --name $AKS_NAME -g $AKS_GRP
``` 

#### Make sure kubernetes version is >= 1.16 

```
kubectl get nodes -o wide 
``` 

#### Create csi driver namespace
```
kubectl create ns $CSI_NAMESPACE
```

#### Helm repo for cs-secret-store-driver-azure-provider. 
##### ***This helm repo provide both the csi-secrets-store-driver as well as csi-secrets-store-driver-azure-provider as well***
``` 
helm repo add csi-secrets-store-provider-azure https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/master/charts 
helm install -n $CSI_NAMESPACE csi-secrets-store-provider-azure/csi-secrets-store-provider-azure --generate-name 
``` 

#### Make sure azure CSI driver pods are running. 
```
kubectl get all -n $CSI_NAMESPACE
```

#### Create keyvault group
```
az group create -n $AZ_KV_GRP -l $LOC --tags label=$SUFFIX
```

#### Create azure key vault 
``` 
az keyvault create --name $AZ_KV_NAME --resource-group $AZ_KV_GRP
``` 

#### Assign service principal a reader role for the keyvault
```
az role assignment create --role Reader --assignee $SP_CLIENT_ID --scope /subscriptions/$SUBID/resourcegroups/$AZ_KV_GRP/providers/Microsoft.KeyVault/vaults/$AZ_KV_NAME

```

#### Then assign get policies for the secrets && create two key vault secrets with name as *dbusername* and *dbpassword*. Provide some secret values
```
az keyvault set-policy -n $AZ_KV_NAME --secret-permissions get --spn $SP_CLIENT_ID
az keyvault secret set --name dbusername --value=dbadmin --vault-name $AZ_KV_NAME
az keyvault secret set --name dbpassword --value=supersecretpassword --vault-name $AZ_KV_NAME
```

#### Create secret based on the appid and appsecret of the service principal
```
kubectl create secret generic $SP_AKS_SECRET --from-literal clientid=$SP_CLIENT_ID --from-literal clientsecret=$SP_CLIENT_SECRET
```

#### Populate all the respective details in storageproviderclass yaml file && run 
```
envsubst < azure-kv-provider.yaml | kubectl apply -f -
```

#### Lastly deploy a sample test nginx deployment to test
```
envsubst < nginx-deployment.yaml | kubectl apply -f - 
```

#### Wait for nginx pod to be up and running and then test the secrets are mounted or not
```
kubectl exec nginx-app -- ls /mnt/secrets/
kubectl exec nginx-app -- cat /mnt/secrets/DB_PASSWORD
```

#### Clean up resources

```
az ad sp delete --id $SP_CLIENT_ID
for rg in $(az group list --tag label=$SUFFIX --query '[].name' | jq -r '.[]'); do echo "Delete Resource Group: ${rg}"; az group delete -n ${rg}; done
```
