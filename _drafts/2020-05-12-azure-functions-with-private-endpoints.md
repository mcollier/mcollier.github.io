---
layout: post
title:  "Azure Functions with Private Endpoints"
date:   2020-05-12 09:00:00 -0500
categories: [azure-functions]
description: Setting up Azure Functions to work with private endpoints.
tags: [azure-functions, networking, virtual-network, azure-bastion, private-endpoints]
comments: true
---

This post will detail my personal experience configuring an Azure Function to work with Azure resources using [private endpoints](https://docs.microsoft.com/azure/private-link/private-endpoint-overview).  Using private endpoints provides virtual network-based controls on accessing Azure resources.  In short, by using private endpoints, Azure Functions will communicate with designated resources using a resource specific private, non-routable IP address (e.g. 10.0.0/24 address space).  The example I'll show in this post uses private endpoints for Azure Storage and CosmosDB.

The sample shown in this post, and [accompanying GitHub repository](https://github.com/mcollier/azure-functions-private-storage), will discuss the following key concepts necessary to use private endpoints with Azure Functions:

- Azure Function with blob trigger and CosmosDB output binding
- Virtual network
- Virtual network (VNET) integration
- Configuring private endpoints for Azure resources
  - Azure Storage private endpoints
  - Azure Cosmos DB private endpoints
- Using private Azure DNS zones

Additionally, the sample will use an Azure VM and Azure Bastion in order to access Azure resources for which access is restricted to the virtual network.

## Architecture Overview

The following diagram shows the high-level architecture of the solution to be created:

![Architecture overview](/assets/azure-functions-with-private-endpoints/high-level-architecture.jpg)

## Deployment

In order to get started with this sample, you'll need an Azure subscription.  If you don't have one, you can get a free Azure account at [https://azure.microsoft.com/free/](https://azure.microsoft.com/free/).

### Prerequisites

- [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli)
- [Azure Function Core Tools](https://docs.microsoft.com/azure/azure-functions/functions-run-local)

### Resource Manager template

The template will provision all the necessary Azure resources. The template will also create the application settings needed by the Azure Function sample code.  The Azure CLI can be used to deploy the template:

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

### Sample Function code

The function can be published manually by using the Azure Function Core Tools:

```azurecli
func azure functionapp publish <function-app-name>
```

## Azure Function

The function used in this sample is based on a simplified concept of processing census data.  Many of the Azure resources used in this sample will be related to census data.  At a high level, the function process is as follows:

- trigger on a new CSV-formatted file being available in a specific Azure Storage blob container
- convert the CSV file to JSON
- save the JSON document to CosmosDB via an output binding

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

### Azure Function Premium plan

The Azure Function app provisioned in this sample uses an [Azure Functions Premium plan](https://docs.microsoft.com/azure/azure-functions/functions-premium-plan#features). The Premium plan is used to enable virtual network integration. Virtual network integration is significant in this sample as the storage accounts used by the function app can only be accessed via private endpoints within the virtual network.

### Azure Function Configuration

There are a few important details about the configuration of the function.

#### Run from Package

The function is configured to [run from a deployment package](https://docs.microsoft.com/azure/azure-functions/run-functions-from-deployment-package). As such, the package is persisted in an Azure File share referenced by the [WEBSITE_CONTENTAZUREFILECONNECTIONSTRING](https://docs.microsoft.com/azure/azure-functions/functions-app-settings#website_contentazurefileconnectionstring) application setting.

#### Virtual Network Triggers

[Virtual network trigger support must be enabled](https://docs.microsoft.com/azure/azure-functions/functions-networking-options#premium-plan-with-virtual-network-triggers) in order for the function to trigger based on resources using a private endpoint.  Virtual network trigger support can be enabled via the Azure portal, the Azure CLI, or via an ARM template (as done in this sample).

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

When using VNet Integration, the function app will use the same DNS server that is configured for the virtual network.  To work with a private endpoint, this default configuration will need to be overridden.  In order to make [calls to a resource using a private endpoint](https://docs.microsoft.com/azure/azure-functions/functions-networking-options#azure-dns-private-zones), it is necessary to integrate with Azure DNS Private Zones.

Private endpoints automatically create Azure DNS Private Zones.  The Azure DNS Private Zone contains the details on how to route requests to the private IP address for the designated Azure service.  Therefore, it is necessary to configure the app to use a specific Azure DNS server, and also route all network traffic into the virtual network. This is accomplished by setting the following application settings:

| Name | Value |
|------|-------|
| WEBSITE_DNS_SERVER | 168.63.129.16 |
| WEBSITE_VNET_ROUTE_ALL | 1 |

## Virtual Network

Azure resources in this sample either integrate with or are placed within a virtual network. The use of private endpoints keeps network traffic contained with the virtual network.

The sample uses four subnets:

- Subnet for Azure Function virtual network integration. This subnet is delegated to the function.
- Subnet for private endpoints. Private IP addresses are allocated from this subnet.
- Subnet for the virtual machine. The template creates a VM which is placed within this subnet.
- Subnet for the [Azure Bastion host](https://docs.microsoft.com/azure/bastion/bastion-create-host-portal).

## Virtual Network (VNet) Integration

As previously mentioned, the sample uses an Azure Function Premium plan which enables support for VNet Integration.  By using VNet Integration, the function is able to access the following:

- resources in a virtual network in the same region as the function app
- resources secured with a service endpoint or private endpoint
- resources in the integrated virtual network

For this sample specifically, VNet Integration enables the function to access Azure Storage and CosmosDB via the configured private endpoints.

For more information on using Azure Functions with virtual network integration, please refer to [https://docs.microsoft.com/azure/azure-functions/functions-networking-options#regional-virtual-network-integration](https://docs.microsoft.com/azure/azure-functions/functions-networking-options#regional-virtual-network-integration)

## Azure Storage Private Endpoints

One of the key scenarios in this sample is the use of Azure Storage private endpoints with the storage account required for use with Azure Functions. Azure Functions uses a storage account for metadata related to the runtime and various triggers. The storage account is referenced via the [AzureWebJobsStorage](https://docs.microsoft.com/azure/azure-functions/functions-app-settings#azurewebjobsstorage) application setting. The _AzureWebJobsStorage_ account will be configured for access via private endpoints.

It is important to note that it is currently not possible to use the storage account referenced by _AzureWebJobsStorage_ for the Azure Function's application code __[if that storage account uses any virtual network restrictions](https://docs.microsoft.com/azure/azure-functions/functions-networking-options#restrict-your-storage-account-to-a-virtual-network)__. Therefore, another Azure Storage account, without virtual network restrictions, is needed.

The sample will use three Azure storage related application settings:

| Name | Description |
|------|-------|-------------|
| CensusResultsAzureStorageConnection | The connection string for an Azure Storage account which uses a private endpoint.  For the purposes of this sample, this storage account is separate from the _AzureWebJobsStorage_ account.   |
| [AzureWebJobsStorage](https://docs.microsoft.com/azure/azure-functions/functions-app-settings#azurewebjobsstorage) | The connection string for an Azure Storage account which uses a private endpoint | TODO |
| [WEBSITE_CONTENTAZUREFILECONNECTIONSTRING](https://docs.microsoft.com/azure/azure-functions/functions-app-settings#website_contentazurefileconnectionstring) | The connection string references an Azure Storage account which contains an Azure File share used to store the application content/code. This storage account __does not__ use a private endpoint or any other virtual network restrictions. |

When using private endpoints for Azure Storage, it is necessary to create a private endpoint for each Azure Storage service (table, blob, queue, or file).  Therefore, this samples sets up 5 private endpoints related to Azure Storage.

1. A private endpoint for the Azure Storage __queue__ referenced by the _AzureWebJobsStorage_ application setting.
2. A private endpoint for the Azure Storage __blob__ storage referenced by the _AzureWebJobsStorage_ application setting.
3. A private endpoint for the Azure Storage __table__ storage referenced by the _AzureWebJobsStorage_ application setting.
4. A private endpoint for the Azure Storage __file__ storage referenced by the _AzureWebJobsStorage_ application setting.
5. A private endpoint for __blob__ storage referenced by the _CensusResultsAzureStorageConnection_ application setting.  This is the only private endpoint related to the _CensusResultsAzureStorageConnection_ application setting.

## Azure Cosmos DB Private Endpoints

As mentioned previously, a CosmosDB output binding is used to save data to a CosmosDB collection.  A [CosmosDB private endpoint](https://docs.microsoft.com/azure/cosmos-db/how-to-configure-private-endpoints) is created, and the function communicates with CosmosDB via the private endpoint.

Much like Azure Storage, CosmosDB contains different services and a private endpoint is needed for each.  Meaning, there is a private endpoint for the SQL protocol, and another private endpoint for the Mongo protocol, etc.

It is important to note that value for the "groupIds" in the ARM template below is case sensitive.  In most cases, ARM templates are not case sensitive. In this case, the value _must_ be "Sql".  If it is "sql", you'll receive an "InternalServerError" error.

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

I've submitted an Azure Feedback request at [https://feedback.azure.com/forums/217313-networking/suggestions/40247743-private-endpoint-groupid-should-be-case-insensitiv](https://feedback.azure.com/forums/217313-networking/suggestions/40247743-private-endpoint-groupid-should-be-case-insensitiv).  Please vote it up if you'd like to see this changed to be case insensitive.

## Private Azure DNS Zones

By default, Azure services have DNS configuration to know how to connect to other Azure services over a public endpoint.  However, when using a private endpoint, the connection isn't made over the public endpoint.  It's made using a private IP address allocated specifically for that Azure resource.

While the connection string remains the same, the private endpoint creates an alias in a subdomain prefixed with "privatelink".  For example, blobs in an Azure Storage account may have a public DNS name of contoso.blob.core.windows.net.  Yet, from within the virtual network, a private DNS zone is automatically created which corresponds to contoso.privatelink.blob.core.windows.net.  A DNS A record is created for each private IP address associated with the private endpoint.  Clients within the virtual network would resolve the connection to the storage account as follows:

| Name | Type | Value |
|------|------|-------|
| contoso.blob.core.windows.net | CNAME | contoso.privatelink.blob.core.windows.net |
| contoso.privatelink.blob.core.windows.net | A | 10.100.1.0 |

For me, setting up the Azure Private DNS Zones was one of the more challenging parts of this exercise. Fortunately, there is new API available that makes this process __much__ easier!  The `privateDnsZoneGroups` sub-type handles configuring the DNS zone, obtaining the private IP address for the configured service, and setting up the corresponding DNS A record.

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

The following DNS zones are created in this sample:

- privatelink.queue.core.windows.net
- privatelink.blob.core.windows.net
- privatelink.table.core.windows.net
- privatelink.file.core.windows.net
- privatelink.documents.azure.com

## Summary

## Resources

- [GitHub repository](https://github.com/mcollier/azure-functions-private-storage)
