# azure-end2end-encryption
Azure end2end encryption with api gateway
https://docs.microsoft.com/en-us/learn/modules/end-to-end-encryption-with-app-gateway

## Resource Groupe
```shell

export rgName=az-endtoend-encryption

az group create --name $rgName --location eastus
```

## Shipping Portal App
```shell

git clone https://github.com/MicrosoftDocs/mslearn-end-to-end-encryption-with-app-gateway shippingportal

cd shippingportal

bash setup-infra.sh

echo https://"$(az vm show \
  --name webservervm1 \
  --resource-group $rgName \
  --show-details \
  --query [publicIps] \
  --output tsv)"

```

## Configure backend pool for encryption
```shell
#get the private IP address of the virtual machine that's acting as the web server.
privateip="$(az vm list-ip-addresses \
  --resource-group $rgName \
  --name webservervm1 \
  --query "[0].virtualMachine.network.privateIpAddresses[0]" \
  --output tsv)"

Set up the backend pool for Application Gateway by using the private IP address of the virtual machine.
az network application-gateway address-pool create \
  --resource-group $rgName \
  --gateway-name gw-shipping \
  --name ap-backend \
  --servers $privateip

#Upload the certificate for the VM in the backend pool
az network application-gateway root-cert create \
  --resource-group $rgName \
  --gateway-name gw-shipping \
  --name shipping-root-cert \
  --cert-file server-config/shipping-ssl.crt

#Configure the HTTP settings to use the certificate.
az network application-gateway http-settings create \
  --resource-group $rgName \
  --gateway-name gw-shipping \
  --name https-settings \
  --port 443 \
  --protocol Https \
  --host-name $privateip

#Run the following commands to set the trusted certificate for the backend pool to the certificate installed on the backend VM.
export rgID="$(az group show --name $rgName --query id --output tsv)"

az network application-gateway http-settings update \
    --resource-group $rgName \
    --gateway-name gw-shipping \
    --name https-settings \
    --set trustedRootCertificates='[{"id": "'$rgID'/providers/Microsoft.Network/applicationGateways/gw-shipping/trustedRootCertificates/shipping-root-cert"}]'

```

## Setup App Gateway
```shell
#create a new frontend port (443) for the gateway.
az network application-gateway frontend-port create \
    --resource-group $rgName \
    --gateway-name gw-shipping  \
    --name proxy-ssl \
    --port 443

#Upload the SSL certificate for Application Gateway. 
az network application-gateway ssl-cert create \
   --resource-group $rgName \
   --gateway-name gw-shipping \
   --name appgateway-cert \
   --cert-file server-config/appgateway.pfx \
   --cert-password somepassword

#create a new listener that accepts incoming traffic on port 443
az network application-gateway http-listener create \
  --resource-group $rgName \
  --gateway-name gw-shipping \
  --name https-listener \
  --frontend-port proxy-ssl \
  --ssl-cert appgateway-cert

#create a rule that directs traffic received through the new listener to the backend pool.
az network application-gateway rule create \
    --resource-group $rgName \
    --gateway-name gw-shipping \
    --name https-rule \
    --address-pool ap-backend \
    --http-listener https-listener \
    --http-settings https-settings \
    --rule-type Basic

echo https://$(az network public-ip show \
  --resource-group $rgName \
  --name appgwipaddr \
  --query ipAddress \
  --output tsv)
```