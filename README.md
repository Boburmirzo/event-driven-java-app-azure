---
page_type: sample
languages:
- java
products:
- azure
- azure-functions
- azure-event-hubs
- azure-cosmos-db
description: "Shows how to use build a real-time event-driven Java solution in Azure."
urlFragment: "sample"
---

# Build a real-time event-driven Java solution in Azure

This repo shows wit the simple example how to build serverless Java solutions using the power of serverless event-driven computing, Azure Functions, and send event-based telemetric data in real time to Azure Cosmos DB.

This sample accompanies [Tutorial: Create an Azure function in Java with an Event Hub trigger and Cosmos DB output binding](https://docs.microsoft.com/azure/azure-functions/functions-event-hub-cosmos-db).

The commands below use a Bash environment. Equivalent commands for the Windows Cmd environment are provided in the tutorial.

## Prerequisites

* [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli)
* [Java Developer Kit](https://aka.ms/azure-jdks), version 8
* [Maven](https://maven.apache.org)
* [Azure Functions Core Tools](https://www.npmjs.com/package/azure-functions-core-tools)

## Setup

Navigate to the folder containing the repo and run the following commands. Replace the `<value>` placeholders with appropriate resource names and location.

``` bash
RESOURCE_GROUP=<value>
EVENT_HUB_NAMESPACE=<value>
EVENT_HUB_NAME=<value>
EVENT_HUB_AUTHORIZATION_RULE=<value>
COSMOS_DB_ACCOUNT=<value>
STORAGE_ACCOUNT=<value>
FUNCTION_APP=<value>
LOCATION=<value>

az group create \
    --name $RESOURCE_GROUP \
    --location $LOCATION

az eventhubs namespace create \
    --resource-group $RESOURCE_GROUP \
    --name $EVENT_HUB_NAMESPACE
az eventhubs eventhub create \
    --resource-group $RESOURCE_GROUP \
    --name $EVENT_HUB_NAME \
    --namespace-name $EVENT_HUB_NAMESPACE \
    --message-retention 1
az eventhubs eventhub authorization-rule create \
    --resource-group $RESOURCE_GROUP \
    --name $EVENT_HUB_AUTHORIZATION_RULE \
    --eventhub-name $EVENT_HUB_NAME \
    --namespace-name $EVENT_HUB_NAMESPACE \
    --rights Listen Send

az cosmosdb create \
    --resource-group $RESOURCE_GROUP \
    --name $COSMOS_DB_ACCOUNT
az cosmosdb sql database create \
    --resource-group $RESOURCE_GROUP \
    --account-name $COSMOS_DB_ACCOUNT \
    --name TelemetryDb
az cosmosdb sql container create \
    --resource-group $RESOURCE_GROUP \
    --account-name $COSMOS_DB_ACCOUNT \
    --database-name TelemetryDb \
    --name TelemetryInfo \
    --partition-key-path '/temperatureStatus'

az storage account create \
    --resource-group $RESOURCE_GROUP \
    --name $STORAGE_ACCOUNT \
    --sku Standard_LRS
az functionapp create \
    --resource-group $RESOURCE_GROUP \
    --name $FUNCTION_APP \
    --storage-account $STORAGE_ACCOUNT \
    --consumption-plan-location $LOCATION \
    --runtime java \
    --functions-version 2

AZURE_WEB_JOBS_STORAGE=$( \
    az storage account show-connection-string \
        --name $STORAGE_ACCOUNT \
        --query connectionString \
        --output tsv)
echo $AZURE_WEB_JOBS_STORAGE
EVENT_HUB_CONNECTION_STRING=$( \
    az eventhubs eventhub authorization-rule keys list \
        --resource-group $RESOURCE_GROUP \
        --name $EVENT_HUB_AUTHORIZATION_RULE \
        --eventhub-name $EVENT_HUB_NAME \
        --namespace-name $EVENT_HUB_NAMESPACE \
        --query primaryConnectionString \
        --output tsv)
echo $EVENT_HUB_CONNECTION_STRING
COSMOS_DB_CONNECTION_STRING=$( \
    az cosmosdb keys list \
        --resource-group $RESOURCE_GROUP \
        --name $COSMOS_DB_ACCOUNT \
        --type connection-strings \
        --query connectionStrings[0].connectionString \
        --output tsv)
echo $COSMOS_DB_CONNECTION_STRING

az functionapp config appsettings set \
    --resource-group $RESOURCE_GROUP \
    --name $FUNCTION_APP \
    --settings \
        AzureWebJobsStorage=$AZURE_WEB_JOBS_STORAGE \
        EventHubConnectionString=$EVENT_HUB_CONNECTION_STRING \
        CosmosDBConnectionString=$COSMOS_DB_CONNECTION_STRING

mvn archetype:generate --batch-mode \
    -DarchetypeGroupId=com.microsoft.azure \
    -DarchetypeArtifactId=azure-functions-archetype \
    -DappName=$FUNCTION_APP \
    -DresourceGroup=$RESOURCE_GROUP \
    -DgroupId=com.example \
    -DartifactId=telemetry-functions

cd telemetry-functions
rm -r src/test

func azure functionapp fetch-app-settings $FUNCTION_APP
```

or you can use the following cmds with resource names prepopulated for the demo:

```bash
## Create a new Resource Group
az group create --name java-event-based --location westeurope

## Create a new eventhubs namespace
az eventhubs namespace create --resource-group java-event-based --name device-event-telemetry-ns

## Create a new evenhub inside device-event-telemetry-ns namespace
az eventhubs eventhub create --resource-group java-event-based --name device-event-hub --namespace-name device-event-telemetry-ns --message-retention 1
 
az eventhubs eventhub authorization-rule create --resource-group java-event-based --name device-event-receive-ar --eventhub-name device-event-hub --namespace-name device-event-telemetry-ns --rights Listen Send

az cosmosdb create --resource-group java-event-based --name events-data-account

az cosmosdb sql database create --resource-group java-event-based --account-name events-data-account --name TelemetryDb
 
az cosmosdb sql container create --resource-group java-event-based --account-name events-data-account --database-name TelemetryDb --name TelemetryInfo --partition-key-path '/temperatureStatus'

## Storage Account

az storage account create --resource-group java-event-based --name eventsstoragejava --sku Standard_LRS

## Create a new Function APP
 
az functionapp create --resource-group java-event-based --name event-trigger-java-func-app --storage-account eventsstoragejava --consumption-plan-location westeurope --runtime java --functions-version 3

## Get the storage account connection string and set it to env variable
SET AZURE_WEB_JOBS_STORAGE=$(az storage account show-connection-string --name eventsstoragejava --query connectionString --output tsv)

echo $AZURE_WEB_JOBS_STORAGE
  
echo $AZURE_WEB_JOBS_STORAGE EVENT_HUB_CONNECTION_STRING=$(az eventhubs eventhub authorization-rule keys list --resource-group java-event-based --name device-event-receive-ar --eventhub-name device-event-hub --namespace-name device-event-telemetry-ns --query primaryConnectionString --output tsv)
  
echo $EVENT_HUB_CONNECTION_STRING 

COSMOS_DB_CONNECTION_STRING=$(az cosmosdb keys list --resource-group java-event-based --name events-data-account --type connection-strings --query connectionStrings[0].connectionString --output tsv)

echo $COSMOS_DB_CONNECTION_STRING

## Update the function app's settings with all connection strings
az functionapp config appsettings set --resource-group java-event-based --name event-trigger-java-func-app --settings AzureWebJobsStorage=$AZURE_WEB_JOBS_STORAGE EventHubConnectionString=$EVENT_HUB_CONNECTION_STRING CosmosDBConnectionString=$COSMOS_DB_CONNECTION_STRING

mvn archetype:generate --batch-mode -DarchetypeGroupId=com.microsoft.azure -DarchetypeArtifactId=azure-functions-archetype -DappName=event-trigger-java-func-app -DresourceGroup=java-event-based -DgroupId=com.example -DartifactId=telemetry-functions

func azure functionapp fetch-app-settings event-trigger-java-func-app

mvn archetype:generate --batch-mode -DarchetypeGroupId=com.microsoft.azure -DarchetypeArtifactId=azure-functions-archetype -DappName=event-trigger-java-func-app -DresourceGroup=java-event-based -DgroupId=com.example -DartifactId=telemetry-functions

```

## Running the sample

Run the sample locally:

``` bash
mvn clean package
mvn azure-functions:run
```

Deploy to Azure:

```bash
mvn azure-functions:deploy
```

Clean up Azure resources when you are finished:

``` bash
az group delete --name $RESOURCE_GROUP
```

## Key concepts

For details, see [Tutorial: Create an Azure function in Java with an Event Hub trigger and Cosmos DB output binding](https://docs.microsoft.com/azure/azure-functions/functions-eventhub-cosmosdb).
