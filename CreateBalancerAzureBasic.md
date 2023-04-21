## Crear una VM con una llave SSH existente
export RESOURCE_GROUP_NAME=myResourceGroup
export LOCATION=eastus
export VM_NAME=VM0
export VM_IMAGE=UbuntuLTS
export ADMIN_USERNAME=azureuser

az vm create \
  --resource-group $RESOURCE_GROUP_NAME \
  --name $VM_NAME \
  --image $VM_IMAGE \
  --admin-username $ADMIN_USERNAME \
  --ssh-key-values azurekeyPublic.pem \
  --public-ip-sku Standard \
  --size Standard_B2s \
  --ultra-ssd-enabled false \
  --nic-delete-option delete \
  --os-disk-delete-option delete \


## Se creó de tipo Standard DS1 v2 (1 vcpu, 3.5 GiB memory) en East US
## Para conectarse lo puedes hacer como: 
ssh azureuser@${IP} -i azurekey.pem

## Eliminar VM
az vm delete \
    --resource-group $RESOURCE_GROUP_NAME \
    --name $VM_NAME \
    --force-deletion none

## Ejecutar un script de automatización
### Crear archivo ConfigApache.json con el siguiente contenido
{
  "commandToExecute": "apt-get -y udpate && apt-get upgrade && apt-get install -y apache2"
} 

$ az vm extension set --resource-group $RESOURCE_GROUP_NAME --vm-name $VM_NAME --name customScript --publisher Microsoft.Azure.Extensions --version 2.1 --settings ./ConfigApache.json

## Crear balanceador por CLI
### Crear grupo de recursos
az group create --name Balancer --location eastus
### Crear una IP Pública
az network public-ip create --resource-group Balancer --name publicIP --sku Standard
### Crear Balanceador
az network lb create --resource-group Balancer --name LoadBalancer --sku Basic --public-ip-address publicIP --frontend-ip-name frontEndIP --backend-pool-name backEndPool
### Crear prueba de monitoreo o sanidad para decidir si le pasa tráfico
az network lb probe  create -g Balancer --lb-name LoadBalancer --name HealthProbe --protocol tcp --port 80
### Crear regla de entrada y salida del balanceo
az network lb rule create -g Balancer --lb-name LoadBalancer --name LoadBalancerRule --protocol tcp --frontend-port 80  --backend-port 80 --frontend-ip-name FrontEndIP --backend-pool-name BackEndPool --probe-name HealthProbe

## Crear red virtual (Virtual Network o VNet)
az network vnet create -g Balancer -n VNet --subnet-name SubNet
### Crear un Network Security Group (NSG)
az network nsg create -g Balancer -n NetworkSecurityGroup 
### Asociar NSG y VNet
az network nsg rule create -g Balancer --nsg-name NetworkSecurityGroup --name NetworkSecurytyGroupRule --priority 1001 --protocol tcp --destination-port-range 80

## Crear interfaces de red o NICs para las máquinas virtuales
for i in `seq 1 3`; do
az network nic create  -g Balancer --name Nic$i --vnet-name VNet --subnet SubNet --network-security-group NetworkSecurityGroup --lb-name LoadBalancer --lb-address-pools BackEndPool 
done
### Crear zona de disponibilidad
az vm availability-set create -g Balancer --name AvailabilitySet

## Crear máquinas virtuales con las NICs creadas 
for i in `seq 1 3`; do
az vm create \
  --resource-group $RESOURCE_GROUP_NAME \
  --name $VM_NAME$i \
  --image $VM_IMAGE \
  --admin-username $ADMIN_USERNAME \
  --ssh-key-values azurekeyPublic.pem \
  --availability-set AvailabilitySet \
  --nics Nic$i \
  --size Standard_B2s \
  --ultra-ssd-enabled false \
  --os-disk-delete-option delete \
  --custom-data cloud-init.txt \
  --no-wait
done

## Obtener IP pública relacionada al balanceador
az network public-ip show -g Balancer --name publicIP --query [ipAddress] --output tsv

## Eliminar todos los recursos
az group delete --name Balancer