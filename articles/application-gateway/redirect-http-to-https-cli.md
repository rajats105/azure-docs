---
title: Create an application gateway with a certificate - Azure CLI | Microsoft Docs
description: Learn how to create an application gateway and add a certificate for SSL termination using the Azure CLI.
services: application-gateway
author: vhorne
manager: jpconnock
editor: tysonn

ms.service: application-gateway
ms.topic: article
ms.workload: infrastructure-services
ms.date: 7/14/2018
ms.author: victorh

---
# Create an application gateway with HTTP to HTTPS redirection using the Azure CLI

You can use the Azure CLI to create an [application gateway](overview.md) with a certificate for SSL termination. A routing rule is used to redirect HTTP traffic to the HTTPS port in your application gateway. In this example, you also create a [virtual machine scale set](../virtual-machine-scale-sets/virtual-machine-scale-sets-overview.md) for the backend pool of the application gateway that contains two virtual machine instances.

In this article, you learn how to:

> [!div class="checklist"]
> * Create a self-signed certificate
> * Set up a network
> * Create an application gateway with the certificate
> * Add a listener and redirection rule
> * Create a virtual machine scale set with the default backend pool

If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/?WT.mc_id=A261C142F) before you begin.

[!INCLUDE [cloud-shell-try-it.md](../../includes/cloud-shell-try-it.md)]

If you choose to install and use the CLI locally, this quickstart requires that you are running the Azure CLI version 2.0.4 or later. To find the version, run `az --version`. If you need to install or upgrade, see [Install Azure CLI](/cli/azure/install-azure-cli).

## Create a self-signed certificate

For production use, you should import a valid certificate signed by a trusted provider. For this tutorial, you create a self-signed certificate and pfx file using the openssl command.

```azurecli-interactive
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout privateKey.key -out appgwcert.crt
```

Enter values that make sense for your certificate. You can accept the default values.

```azurecli-interactive
openssl pkcs12 -export -out appgwcert.pfx -inkey privateKey.key -in appgwcert.crt
```

Enter the password for the certificate. In this example, *Azure123456!* is being used.

## Create a resource group

A resource group is a logical container into which Azure resources are deployed and managed. Create a resource group using [az group create](/cli/azure/group).

The following example creates a resource group named *myResourceGroupAG* in the *eastus* location.

```azurecli-interactive 
az group create --name myResourceGroupAG --location eastus
```

## Create network resources

Create the virtual network named *myVNet* and the subnet named *myAGSubnet* using [az network vnet create](/cli/azure/network/vnet). You can then add the subnet named *myBackendSubnet* that's needed by the backend servers using [az network vnet subnet create](/cli/azure/network/vnet/subnet). Create the public IP address named *myAGPublicIPAddress* using [az network public-ip create](/cli/azure/network/public-ip).

```azurecli-interactive
az network vnet create \
  --name myVNet \
  --resource-group myResourceGroupAG \
  --location eastus \
  --address-prefix 10.0.0.0/16 \
  --subnet-name myAGSubnet \
  --subnet-prefix 10.0.1.0/24
az network vnet subnet create \
  --name myBackendSubnet \
  --resource-group myResourceGroupAG \
  --vnet-name myVNet \
  --address-prefix 10.0.2.0/24
az network public-ip create \
  --resource-group myResourceGroupAG \
  --name myAGPublicIPAddress
```

## Create the application gateway

You can use [az network application-gateway create](/cli/azure/network/application-gatewaywork_application_gateway_create) to create the application gateway named *myAppGateway*. When you create an application gateway using the Azure CLI, you specify configuration information, such as capacity, sku, and HTTP settings. 

The application gateway is assigned to *myAGSubnet* and *myAGPublicIPAddress* that you previously created. In this example, you associate the certificate that you created and its password when you create the application gateway. 

```azurecli-interactive
az network application-gateway create \
  --name myAppGateway \
  --location eastus \
  --resource-group myResourceGroupAG \
  --vnet-name myVNet \
  --subnet myAGsubnet \
  --capacity 2 \
  --sku Standard_Medium \
  --http-settings-cookie-based-affinity Disabled \
  --frontend-port 443 \
  --http-settings-port 80 \
  --http-settings-protocol Http \
  --public-ip-address myAGPublicIPAddress \
  --cert-file appgwcert.pfx \
  --cert-password "Azure123456!"

```

 It may take several minutes for the application gateway to be created. After the application gateway is created, you can see these new features of it:

- *appGatewayBackendPool* - An application gateway must have at least one backend address pool.
- *appGatewayBackendHttpSettings* - Specifies that port 80 and an HTTP protocol is used for communication.
- *appGatewayHttpListener* - The default listener associated with *appGatewayBackendPool*.
- *appGatewayFrontendIP* - Assigns *myAGPublicIPAddress* to *appGatewayHttpListener*.
- *rule1* - The default routing rule that is associated with *appGatewayHttpListener*.

## Add a listener and redirection rule

### Add the HTTP port

You can use [az network application-gateway frontend-port create](/cli/azure/network/application-gateway/frontend-portwork_application_gateway_frontend_port_create) to add the HTTP port to the application gateway.

```azurecli-interactive
az network application-gateway frontend-port create \
  --port 80 \
  --gateway-name myAppGateway \
  --resource-group myResourceGroupAG \
  --name httpPort
```

### Add the HTTP listener

You can use [az network application-gateway http-listener create](/cli/azure/network/application-gateway/http-listenerwork_application_gateway_http_listener_create) to add the listener named *myListener* to the application gateway.

```azurecli-interactive
az network application-gateway http-listener create \
  --name myListener \
  --frontend-ip appGatewayFrontendIP \
  --frontend-port httpPort \
  --resource-group myResourceGroupAG \
  --gateway-name myAppGateway
```

### Add the redirection configuration

Add the HTTP to HTTPS redirection configuration to the application gateway using [az network application-gateway redirect-config create](/cli/azure/network/application-gateway/redirect-configwork_application_gateway_redirect_config_create).

```azurecli-interactive
az network application-gateway redirect-config create \
  --name httpToHttps \
  --gateway-name myAppGateway \
  --resource-group myResourceGroupAG \
  --type Permanent \
  --target-listener appGatewayHttpListener \
  --include-path true \
  --include-query-string true
```

### Add the routing rule

Add the routing rule named *rule2* with the redirection configuration to the application gateway using [az network application-gateway rule create](/cli/azure/network/application-gateway/rulework_application_gateway_rule_create).

```azurecli-interactive
az network application-gateway rule create \
  --gateway-name myAppGateway \
  --name rule2 \
  --resource-group myResourceGroupAG \
  --http-listener myListener \
  --rule-type Basic \
  --redirect-config httpToHttps
```

## Create a virtual machine scale set

In this example, you create a virtual machine scale set named *myvmss* that provides servers for the backend pool in the application gateway. The virtual machines in the scale set are associated with *myBackendSubnet* and *appGatewayBackendPool*. To create the scale set, you can use [az vmss create](/cli/azure/vmss#az-vmss-create).

```azurecli-interactive
az vmss create \
  --name myvmss \
  --resource-group myResourceGroupAG \
  --image UbuntuLTS \
  --admin-username azureuser \
  --admin-password Azure123456! \
  --instance-count 2 \
  --vnet-name myVNet \
  --subnet myBackendSubnet \
  --vm-sku Standard_DS2 \
  --upgrade-policy-mode Automatic \
  --app-gateway myAppGateway \
  --backend-pool-name appGatewayBackendPool
```

### Install NGINX

```azurecli-interactive
az vmss extension set \
  --publisher Microsoft.Azure.Extensions \
  --version 2.0 \
  --name CustomScript \
  --resource-group myResourceGroupAG \
  --vmss-name myvmss \
  --settings '{ "fileUris": ["https://raw.githubusercontent.com/Azure/azure-docs-powershell-samples/master/application-gateway/iis/install_nginx.sh"],
  "commandToExecute": "./install_nginx.sh" }'
```

## Test the application gateway

To get the public IP address of the application gateway, you can use [az network public-ip show](/cli/azure/network/public-ip). Copy the public IP address, and then paste it into the address bar of your browser.

```azurepowershell-interactive
az network public-ip show \
  --resource-group myResourceGroupAG \
  --name myAGPublicIPAddress \
  --query [ipAddress] \
  --output tsv
```

![Secure warning](./media/redirect-http-to-https-cli/application-gateway-secure.png)

To accept the security warning if you used a self-signed certificate, select **Details** and then **Go on to the webpage**. Your secured NGINX site is then displayed as in the following example:

![Test base URL in application gateway](./media/redirect-http-to-https-cli/application-gateway-nginxtest.png)

## Next steps

In this tutorial, you learned how to:

> [!div class="checklist"]
> * Create a self-signed certificate
> * Set up a network
> * Create an application gateway with the certificate
> * Add a listener and redirection rule
> * Create a virtual machine scale set with the default backend pool


