# AKS autoscaler with calico and kubernetes version 1.20.5

RESOURCE_GROUP_NAME=calicotest
CLUSTER_NAME=calicoscale
LOCATION=canadaeast

# Create a resource group
az group create --name $RESOURCE_GROUP_NAME --location $LOCATION

# Create a virtual network and subnet
az network vnet create \
    --resource-group $RESOURCE_GROUP_NAME \
    --name myVnet \
    --address-prefixes 10.0.0.0/8 \
    --subnet-name myAKSSubnet \
    --subnet-prefix 10.240.0.0/16

# Create a service principal and read in the application ID
SP=$(az ad sp create-for-rbac --output json)
SP_ID=$(echo $SP | jq -r .appId)
SP_PASSWORD=$(echo $SP | jq -r .password)

# Wait 15 seconds to make sure that service principal has propagated
echo "Waiting for service principal to propagate..."
sleep 15

# Get the virtual network resource ID
VNET_ID=$(az network vnet show --resource-group $RESOURCE_GROUP_NAME --name myVnet --query id -o tsv)

# Assign the service principal Contributor permissions to the virtual network resource
az role assignment create --assignee $SP_ID --scope $VNET_ID --role Contributor

# Get the virtual network subnet resource ID
SUBNET_ID=$(az network vnet subnet show --resource-group $RESOURCE_GROUP_NAME --vnet-name myVnet --name myAKSSubnet --query id -o tsv)

# Create the Cluster

az feature register --namespace "Microsoft.ContainerService" --name "EnableAKSWindowsCalico"

# Check:
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/EnableAKSWindowsCalico')].{Name:name,State:properties.state}"

# Refresh the registration:
az provider register --namespace Microsoft.ContainerService


az aks create \
    --resource-group $RESOURCE_GROUP_NAME \
    --name $CLUSTER_NAME \
    --node-count 1 \
    --generate-ssh-keys \
    --service-cidr 10.0.0.0/16 \
    --dns-service-ip 10.0.0.10 \
    --docker-bridge-address 172.17.0.1/16 \
    --vnet-subnet-id $SUBNET_ID \
    --service-principal $SP_ID \
    --client-secret $SP_PASSWORD \
    --vm-set-type VirtualMachineScaleSets \
    --kubernetes-version 1.20.5 \
    --network-plugin azure \
    --network-policy calico \
	--load-balancer-sku standard \
	--enable-cluster-autoscaler \
	--cluster-autoscaler-profile scan-interval=30s \
    --min-count 1 \
    --max-count 3
	
# Connect to the cluster
az aks get-credentials --resource-group $RESOURCE_GROUP_NAME --name $CLUSTER_NAME

# Upgrade the aks extension
az extension add -n aks-preview
az extension update -n aks-preview

# Reconfigure a more agressive environment 
az aks nodepool update --update-cluster-autoscaler --min-count 1 --max-count 10 -g $RESOURCE_GROUP_NAME -n nodepool1 --cluster-name $CLUSTER_NAME

az aks update -g $RESOURCE_GROUP_NAME -n $CLUSTER_NAME --cluster-autoscaler-profile scale-down-delay-after-add=3m scale-down-unneeded-time=3m scale-down-unneeded-time=1m scale-down-unready-time=3m skip-nodes-with-system-pods=false skip-nodes-with-local-storage=false --min-count 1 --max-count 10

az aks update --resource-group $RESOURCE_GROUP_NAME --name $CLUSTER_NAME --cluster-autoscaler-profile ""
kubectl scale deployment akstsdpl --replicas=500

# wait some minutes to allow the pods to be created
kubectl scale deployment akstsdpl --replicas=1

# Wait the scale down. It will take a couple of minutes
while true; do kubectl get pod -o wide|grep -i running|wc -l; sleep 5;kubectl get nodes -o wide; done
