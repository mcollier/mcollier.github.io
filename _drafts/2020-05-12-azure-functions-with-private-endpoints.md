---
layout: post
title:  "Azure Functions with Private Endpoints"
date:   2020-05-12 09:00:00 -0500
categories: [azure-functions]
description: Setting up Azure Functions to work with private endpoints.
tags: [azure-functions, networking, virtual-network, azure-bastion, private-endpoints]
comments: true
---

As enterprises continue to adopt serverless (and Platform-as-a-Service, or PaaS) solutions, they often need a way to integrate with existing resources on a virtual network.  These existing resources could be databases, file storage, message queues or event streams, or REST APIs.  In doing so, those interactions need to take place within the virtual network.  Until relatively recently, combining serverless/PaaS offerings with traditional network access restrictions was complex, if not nearly impossible.  I feel the story is changing.  With the introduction of Azure Virtual Network service endpoints and private endpoints, it's becoming easier for enterprises to realize the benefits of serverless, while also complying with necessary virtual network access controls.

This post will detail how I am able to configure an Azure Function to work with Azure resources using [private endpoints](https://docs.microsoft.com/azure/private-link/private-endpoint-overview). By using private endpoints, I ensure that the resources are accessible only via my virtual network.  This gives me an additional level of network-based security and control.

By using private endpoints, the Azure Function app will communicate with designated resources using a resource-specific private IP address (e.g. 10.0.0/24 address space).  The example I'll show in this post uses private endpoints for Azure Storage and CosmosDB.

The sample shown in this post, and [accompanying GitHub repository](https://github.com/mcollier/azure-functions-private-storage), discusses the following key concepts necessary to use private endpoints with Azure Functions:

- Azure Function with blob trigger and CosmosDB output binding
- Azure Function Premium plan with Virtual Network Integration enabled
- Virtual network
- Configuring private endpoints for Azure resources
  - Azure Storage private endpoints
  - Azure Cosmos DB private endpoint
- Using private Azure DNS zones

Additionally, the sample uses an Azure VM and Azure Bastion in order to access Azure resources within the virtual network.  The VM and Azure Bastion setup is not discussed in this post.  If you want to learn more about using Azure Bastion, please refer to [https://docs.microsoft.com/azure/bastion/bastion-overview](https://docs.microsoft.com/azure/bastion/bastion-overview)

## Architecture Overview

The following diagram shows the high-level architecture of the solution to be created:

![Architecture overview](/assets/azure-functions-with-private-endpoints/high-level-architecture.jpg)

## Deployment

In order to get started with this sample, you'll need an Azure subscription.  If you don't have one, you can get a free Azure account at [https://azure.microsoft.com/free/](https://azure.microsoft.com/free/).

### Prerequisites

- [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli)
- [Azure Function Core Tools](https://docs.microsoft.com/azure/azure-functions/functions-run-local)

### Deploy the Resource Manager template

I've created an Azure Resource Manager (ARM) template to provision all the necessary Azure resources. The template will also create the application settings needed by the Azure Function sample code.  The Azure CLI can be used to deploy the template:

```bash
resourceGroupName="functions-private-endpoints"
location="eastus"
now=`date +%Y%m%d-%H%M%S`
deploymentName="azuredeploy-$now"

echo "Creating resource group '$resourceGroupName' in region '$location' . . ."
az group create --name $resourceGroupName --location $location

echo "Deploying main template . . ."
az deployment group create -g $resourceGroupName --template-file azuredeploy.json --parameters azuredeploy.parameters.json --name $deploymentName
```

### Deploy the code

The function can be published manually by using the Azure Function Core Tools:

```azurecli
func azure functionapp publish <function-app-name>
```

## Virtual Network

One of the first components to set up is the virtual network.  Nearly all other Azure services in this samp;le are either provisioned into the virtual network, or integrated with, the virtual network.  After all, this sample is about using private endpoints, and private endpoints go along with a virtual network (can't have one without the other).

The sample uses four subnets:

- Subnet for Azure Function virtual network integration. This subnet is delegated to the function.
- Subnet for private endpoints. Private IP addresses are allocated from this subnet.
- Subnet for the virtual machine. The template creates a VM which is placed within this subnet.
- Subnet for the [Azure Bastion host](https://docs.microsoft.com/azure/bastion/bastion-create-host-portal).

## Virtual Network (VNet) Integration

In order for the function to access resources within the virtual network, VNet Integration is needed.  The [matrix of Azure Functions networking features](https://docs.microsoft.com/azure/azure-functions/functions-networking-options#matrix-of-networking-features) shows that there are currently three options for VNet Integration:

1. Azure Functions Premium plan
2. Azure App Service Plan
3. Azure App Service Environment

As I like the purely serverless approach, I use an [Azure Function Premium plan](https://docs.microsoft.com/azure/azure-functions/functions-premium-plan).  By using an Azure Function Premium plan with VNet Integration enabled, the function is able to access Azure Storage and CosmosDB via the configured private endpoints.

Please refer to the [official documentation](https://docs.microsoft.com/azure/azure-functions/functions-networking-options#regional-virtual-network-integration) for more information on using Azure Functions with virtual network integration.

## Azure Function

The function used in this sample is based on a simplified concept of processing census data.  Many of the Azure resources used in this sample will be related to census data.  At a high level, the function logic is as follows:

1. Trigger on a new CSV-formatted file being available in a specific Azure Storage blob container
2. Convert the CSV file to JSON
3. Save the JSON document to CosmosDB via an output binding

The function is invoked via an [Azure Storage blob trigger](https://docs.microsoft.com/azure/azure-functions/functions-bindings-storage-blob-trigger).  The storage account used by the blob trigger is configured with a private endpoint.

The function assumes the file is in a CSV format, and then converts the CSV content to JSON.  The resulting JSON document is saved to an [Azure CosmosDB collection via an output binding](https://docs.microsoft.com/azure/azure-functions/functions-bindings-cosmosdb-v2-output).

```csharp
[FunctionName("CensusDataFilesFunction")]
public static async Task ProcessCensusDataFiles(
    [BlobTrigger("%ContainerName%/{blobName}", Connection = "CensusResultsAzureStorageConnection")] Stream myBlobStream,
    string blobName,
    [CosmosDB(
        databaseName: "%CosmosDbName%",
        collectionName: "%CosmosDbCollectionName%",
        ConnectionStringSetting = "CosmosDBConnection")] IAsyncCollector<string> items,
    ILogger logger)
{
    logger.LogInformation($"C# Blob trigger function processed blob of name '{blobName}' with size of {myBlobStream.Length} bytes");

    var jsonObject = await ConvertCsvToJsonAsync(myBlobStream);

    foreach (var item in jsonObject)
    {
        await items.AddAsync(JsonConvert.SerializeObject(item));
    }
}
```

### Azure Function Configuration

There are a few important details about the configuration of the function.

#### Run from Package

The function is configured to [run from a deployment package](https://docs.microsoft.com/azure/azure-functions/run-functions-from-deployment-package). As such, the package is persisted in an Azure File share referenced by the [WEBSITE_CONTENTAZUREFILECONNECTIONSTRING](https://docs.microsoft.com/azure/azure-functions/functions-app-settings#website_contentazurefileconnectionstring) application setting.

#### Virtual Network Triggers

[Virtual network trigger support must be enabled](https://docs.microsoft.com/azure/azure-functions/functions-networking-options#premium-plan-with-virtual-network-triggers) in order for the function to trigger against resources using a private endpoint.  Virtual network trigger support can be enabled via the Azure portal, the Azure CLI, or via an ARM template (as done in this sample).

```json
{
    "type": "config",
    "name": "web",
    "apiVersion": "2019-08-01",
    "dependsOn": [
        "[resourceId('Microsoft.Web/sites', variables('functionAppName'))]",
        "[resourceId('Microsoft.Network/virtualNetworks', variables('vnetName'))]"
    ],
    "properties": {
        "functionsRuntimeScaleMonitoringEnabled": true
    }
}
```

View the full ARM template [here](https://github.com/mcollier/azure-functions-private-storage/blob/master/template/azuredeploy.json#L1054-L1064).

#### Azure DNS Private Zones

When using VNet Integration, the function app uses the same DNS server that is configured for the virtual network.  To work with a private endpoint, the default configuration needs to be overridden.  In order to make [calls to a resource using a private endpoint](https://docs.microsoft.com/azure/azure-functions/functions-networking-options#azure-dns-private-zones), it is necessary to integrate with Azure DNS Private Zones.

Private endpoints automatically create Azure DNS Private Zones.  The Azure DNS Private Zone contains the details on how to route requests to the private IP address for the designated Azure service.  Therefore, it is necessary to configure the app to use a specific Azure DNS server, and also route all network traffic into the virtual network. This is accomplished by setting the following application settings:

| Name | Value |
|------|-------|
| WEBSITE_DNS_SERVER | 168.63.129.16 |
| WEBSITE_VNET_ROUTE_ALL | 1 |

## Azure Storage Private Endpoints

Azure Functions requires an Azure Storage account for persisting runtime metadata and metadata related to various triggers. The official Microsoft [documentation indicates that it is currently not possible to use Azure Functions with a storage account which uses virtual network restrictions](https://docs.microsoft.com/azure/azure-functions/functions-networking-options#restrict-your-storage-account-to-a-virtual-network).  While that's mostly true, there is a workaround.

The workaround being that it is possible to put virtual network restrictions on the Azure storage account referenced via the [AzureWebJobsStorage](https://docs.microsoft.com/azure/azure-functions/functions-app-settings#azurewebjobsstorage) application setting. However, if that is done, then a separate storage account - _one without network restrictions_ - is needed.

The other (without network restrictions) storage account needs to be referenced via the [WEBSITE_CONTENTAZUREFILECONNECTIONSTRING](https://docs.microsoft.com/azure/azure-functions/functions-app-settings#website_contentazurefileconnectionstring) application setting.  It is this storage account that will contain an Azure File share used to persist the function's application code.

Furthermore, for this sample, a third storage account is used.  This third storage account is used by the sample application code - it's where the CSV blob file will be placed.  The Function blob trigger will pick up this file and the function will do work against it.  This storage account will also use a private endpoint.

The sample will use three Azure storage related application settings:

| Name | Description | Uses a Private Endpoint? |
|------|-------|-------------------------------|
| CensusResultsAzureStorageConnection | The connection string for an Azure Storage account used by the function's blob trigger. | Yes |
| [AzureWebJobsStorage](https://docs.microsoft.com/azure/azure-functions/functions-app-settings#azurewebjobsstorage) | The connection string for an Azure Storage account required by Azure Functions. | Yes |
| [WEBSITE_CONTENTAZUREFILECONNECTIONSTRING](https://docs.microsoft.com/azure/azure-functions/functions-app-settings#website_contentazurefileconnectionstring) | The connection string references an Azure Storage account which contains an Azure File share used to store the application content/code. | No |

When using private endpoints for Azure Storage, it is necessary to create a private endpoint for each Azure Storage service (table, blob, queue, or file).  Therefore, this samples sets up 5 private endpoints related to Azure Storage.

1. A private endpoint for the Azure Storage __queue__ referenced by the _AzureWebJobsStorage_ application setting.
2. A private endpoint for the Azure Storage __blob__ storage referenced by the _AzureWebJobsStorage_ application setting.
3. A private endpoint for the Azure Storage __table__ storage referenced by the _AzureWebJobsStorage_ application setting.
4. A private endpoint for the Azure Storage __file__ storage referenced by the _AzureWebJobsStorage_ application setting.
5. A private endpoint for __blob__ storage referenced by the _CensusResultsAzureStorageConnection_ application setting.  This is the only private endpoint related to the _CensusResultsAzureStorageConnection_ application setting.

## Azure Cosmos DB Private Endpoints

As mentioned previously, a CosmosDB output binding is used to save data to a CosmosDB collection.  A [CosmosDB private endpoint](https://docs.microsoft.com/azure/cosmos-db/how-to-configure-private-endpoints) is created, and the function communicates with CosmosDB via the private endpoint.

CosmosDB supports different API (Sql, Cassandra, Mongo, Table, etc.) types, and a private endpoint is needed for each.  Meaning, there is a private endpoint for the SQL protocol, and another private endpoint for the Mongo protocol, etc.  Since I'm using the Sql API type in this sample, it is only necessary to configure a private endpoint for the Sql API.

For this sample, I set up everything with an ARM template.  It is important to note the value for the "groupIds" in the ARM template below is case sensitive.  In most cases, ARM templates are not case sensitive. In this case, since I'm using the Sql API, the value _must_ be "Sql".  If it is "sql", you'll receive an "InternalServerError" error that leaves you guessing as to what went wrong.  I learned this the hard way.

> Don't be like Mike - use "Sql".  However, if you want to see this changes, so that both "Sql" and "sql" are valid, or that a descriptive error messages is returned, please [vote up the Azure Feedback](https://feedback.azure.com/forums/217313-networking/suggestions/40247743-private-endpoint-groupid-should-be-case-insensitiv) I've submitted.

```json
{
    "type": "Microsoft.Network/privateEndpoints",
    "name": "[variables('privateEndpointCosmosDbName')]",
    "apiVersion": "2019-11-01",
    "location": "[parameters('location')]",
    "dependsOn": [
        "[resourceId('Microsoft.DocumentDB/databaseAccounts', variables('privateCosmosDbAccountName'))]",
        "[resourceId('Microsoft.Network/virtualNetworks', variables('vnetName'))]"
    ],
    "properties": {
        "subnet": {
            "id": "[resourceId('Microsoft.Network/virtualNetworks/subnets', variables('vnetName'), variables('privateEndpointSubnetName') )]"
        },
        "privateLinkServiceConnections": [
            {
                "name": "MyCosmosDbPrivateLinkConnection",
                "properties": {
                    "privateLinkServiceId": "[resourceId('Microsoft.DocumentDB/databaseAccounts', variables('privateCosmosDbAccountName'))]",
                    "groupIds": [
                        "Sql"
                    ]
                }
            }
        ]
    }
}
```

## Private Azure DNS Zones

When working with private endpoints, it is necessary to make changes your [DNS configuration](https://docs.microsoft.com/azure/private-link/private-endpoint-overview#dns-configuration).  You can either use a host file on a VM within the virtual network, a [private DNS zone](https://docs.microsoft.com/azure/dns/private-dns-privatednszone), or your own DNS server hosted within the virtual network.  I decided to use private DNS zones (because I want to manage as little infrastructure as possible).

> More information on Private Endpoint DNS configuration can be found at [https://docs.microsoft.com/azure/private-link/private-endpoint-dns#dns-configuration-scenarios](https://docs.microsoft.com/azure/private-link/private-endpoint-dns#dns-configuration-scenarios).

Azure services have DNS configuration to know how to connect to other Azure services over a public endpoint.  However, when using a private endpoint, the connection isn't made over the public endpoint.  It's made using a private IP address allocated specifically for that Azure resource.  So, the default DNS configuration will need to be overridden.

One of the nice things about working with private endpoints is that the connection string used by calling service doesn't need to change.  In other words, I can use `contoso.blob.core.windows.net` to connect to either the public endpoint or the private endpoint (for blob storage in the Contoso storage account).

This is made possible by using private DNS zones. The private endpoint creates an alias in a subdomain prefixed with "privatelink".  For example, blobs in an Azure Storage account may have a public DNS name of `contoso.blob.core.windows.net`.  A private DNS zone is created which corresponds to `contoso.privatelink.blob.core.windows.net`.  A DNS A record is created for each private IP address associated with the private endpoint.  Clients within the virtual network resolve the connection to the storage account as follows:

| Name | Type | Value |
|------|------|-------|
| `contoso.blob.core.windows.net` | CNAME | `contoso.privatelink.blob.core.windows.net` |
| `contoso.privatelink.blob.core.windows.net` | A | 10.100.1.0 |

Clients external to the virtual network continue to resolve to the public IP address of the service.

### The Zones

This sample uses private endpoints for Azure Storage and CosmosDB.  As such, private DNS zones are needed for each Azure storage service, as well as the Sql API for CosmosDB.  Meaning, five DNS zones are needed to support this sample:

- `privatelink.queue.core.windows.net`
- `privatelink.blob.core.windows.net`
- `privatelink.table.core.windows.net`
- `privatelink.file.core.windows.net`
- `privatelink.documents.azure.com`

I used the recommended zone names as specified at [https://docs.microsoft.com/azure/private-link/private-endpoint-dns#azure-services-dns-zone-configuration](https://docs.microsoft.com/azure/private-link/private-endpoint-dns#azure-services-dns-zone-configuration)

### Setting it up

For me, setting up the Azure Private DNS Zones was one of the more challenging parts of this exercise. Fortunately, there is new API available that makes this process __much__ easier!  The [privateDnsZoneGroups](https://docs.microsoft.com/azure/templates/microsoft.network/privateendpoints/privatednszonegroups) sub-type handles configuring the DNS zone, obtaining the private IP address for the configured service, and setting up the corresponding DNS A record.

Below is a snippet from the ARM template which shows using the `Microsoft.Network/privateEndpoints/privateDnsZoneGroups` type.

```json
{
    "type": "Microsoft.Network/privateEndpoints/privateDnsZoneGroups",
    "apiVersion": "2020-03-01",
    "location": "[parameters('location')]",
    "name": "[concat(variables('privateEndpointCosmosDbName'), '/default')]",
    "dependsOn": [
        "[resourceId('Microsoft.Network/privateDnsZones', variables('privateCosmosDbDnsZoneName'))]",
        "[resourceId('Microsoft.Network/privateEndpoints', variables('privateEndpointCosmosDbName'))]"
    ],
    "properties": {
        "privateDnsZoneConfigs": [
            {
                "name": "config1",
                "properties": {
                    "privateDnsZoneId": "[resourceId('Microsoft.Network/privateDnsZones', variables('privateCosmosDbDnsZoneName'))]"
                }
            }
        ]
    }
}
```

## Summary

This post outline the key components that are necessary to use private endpoints with Azure Functions.  Private endpoints allow for interaction with designated Azure resources via private IP address, thus keeping network traffic between the function and the resource confined to the virtual network.  As enterprises continue to adopt serverless, I'm currently of the opinion that using private endpoints will become increasingly common.

Using private endpoints with Azure Functions requires there to be a virtual network (with a few subnets), an Azure Functions Premium plan with VNet Integration enabled, [Azure resources to connect to which support private endpoints](https://docs.microsoft.com/azure/private-link/private-endpoint-overview#private-link-resource), and modifications to DNS configuration.  There is quite a bit of setup to get right to make it all work.  I hope that this post, along with the sample provided in my [GitHub repository](https://github.com/mcollier/azure-functions-private-storage), make it easier for someone else to get started.

## What's Next

I hope to spend some time to enhance this sample.  My initial idea list includes:

- Moving secrets to Azure Key Vault, and configuring Key Vault with a private endpoint
- Creating a Terraform script to complement the existing ARM template

What would you like to see?  Please let me know in the comments below or [file an issue on the GitHub repo](https://github.com/mcollier/azure-functions-private-storage/issues).

## Resources

- [GitHub repository](https://github.com/mcollier/azure-functions-private-storage)
