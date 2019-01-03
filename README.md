# Build a real-time serverless app with Azure Functions

## Introduction

In this tutorial, you'll be building a serverless web app for displaying airline flight prices. The flight prices on the web page updates in real-time as prices are changed in the database. You'll then add authentication to the application and allow authenticated users to update the prices.

### Technologies used

* [Azure Storage](https://azure.microsoft.com/services/storage/?WT.mc_id=serverlesschatlab-tutorial-antchu)
    - Host the static website for the frontend
* [Azure Functions](https://azure.microsoft.com/services/functions/?WT.mc_id=serverlesschatlab-tutorial-antchu)
    - Backend API for retrieving and modifying prices
    - Logic to broadcast pricing changes in real-time
* [Azure Cosmos DB](https://azure.microsoft.com/services/cosmos-db/?WT.mc_id=serverlesschatlab-tutorial-antchu)
    - Store flight prices
    - Trigger Azure Functions when data changes
* [Azure SignalR Service](https://azure.microsoft.com/services/signalr-service/?WT.mc_id=serverlesschatlab-tutorial-antchu)
    - Broadcast flight price updates to connected chat clients

### Prerequisites

<!-- >> If you are using a prepared virtual machine for this lab, all prerequisites should already be installed. -->

The following software is required to build this tutorial.

* [Node.js](https://nodejs.org/en/download/) (Version 10.x)
* [.NET SDK](https://www.microsoft.com/net/download?WT.mc_id=serverlesschatlab-tutorial-antchu) (Version 2.x, required for Functions extensions)
* [Azure Functions Core Tools](https://github.com/Azure/azure-functions-core-tools) (Version 2)
* [Visual Studio Code](https://code.visualstudio.com/?WT.mc_id=serverlesschatlab-tutorial-antchu) (VS Code) with the following extensions
    * [Azure Tools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-node-azure-pack&WT.mc_id=serverlesschatlab-tutorial-antchu)
        - Includes Azure Functions and Azure Storage extensions
* [http-server](https://www.npmjs.com/package/http-server)
    * For running frontend web app locally
    * With Node installed, run the command `npm install http-server -g`
* [Postman](https://www.getpostman.com/) (optional, for testing Azure Functions)

---

## Create Azure resources

You will build and test the Azure Functions app locally. The app will access some services in Azure that need to be created ahead of time.


### Log in to the Azure portal

1. Go to the [Azure portal](https://portal.azure.com/) and log in with your credentials.


### Create an Azure Cosmos DB account

1. Click on the **Create a resource** (**+**) button for creating a new Azure resource.

1. Under **Databases**, select **Azure Cosmos DB**.

1. Enter the following information.

    | Name | Value |
    |---|---|
    | Subscription | Select the subscription you want to use |
    | Resource group | Create new - enter a new resource group name |
    | Account name | A unique name for the Cosmos DB account |
    | API | Core (SQL) |
    | Location | Select a location close to you |
    | Geo-redundancy | Disable |
    | Multi-region writes | Disable |
    
    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/create-cosmosdb-screenshot.png)

1. Click **Review and create**, then **Create**.


### Create a Storage account

1. Click on the **Create a resource** (**+**) button for creating a new Azure resource.

1. Under **Storage**, select **Storage account**.

1. Enter the following information.

    | Name | Value |
    |---|---|
    | Subscription | Select the same subscription as the Cosmos DB account |
    | Resource group | Select the same resource group as the Cosmos DB account |
    | Storage account name | A unique name for the Storage account |
    | Location | Select a location close to you |
    | Performance | Standard |
    | Account kind | StorageV2 (general purpose v2) |
    | Replication | Locally-redundant storage (LRS) |
    | Access tier (default) | Hot |
    
1. Click **Review and create**, then **Create**.


### Create an Azure SignalR Service instance

1. Click on the **Create a resource** (**+**) button for creating a new Azure resource.

1. Search for **SignalR Service** and select it. Click **Create**.

1. Enter the following information.

    | Name | Value |
    |---|---|
    | Resource name | A unique name for the SignalR Service instance |
    | Subscription | Select the same subscription as the other resources |
    | Resource group | Select the same resource group as the resources |
    | Location | Select a location close to you |
    | Pricing Tier | Free |

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/create-signalr-screenshot.png)
    
1. Click **Create**.


### Create a database and collection in Cosmos DB

Before creating the application, you will initialize the Cosmos DB account with a database and a collection. You will also seed it with some data.

1. In the Azure portal, open the Cosmos DB account that you created.

1. Select **Data Explorer**, create a database and collection.
    1. Click **New Collection**.
    1. Select **Create new** and enter **flightsdb** as the database id.
    1. Select **Provision throughput** and enter a throughput value of **400**.
    1. In **Collection id**, enter **flights**.
    1. Enter a partition key of **/id**.
    1. Click **OK** to create the database.

1. In the Data Explorer, navigate to the newly created **flights** collection.

1. Select the collection's **Documents**.

1. Click on **New Document** and enter the following as the body of the document:

    ```json
    {
        "id": "68a912fd-fed7-aae1-9d7d-af3c4c9cfdaa",
        "from": "SEA",
        "to": "LAX",
        "price": 300
    }
    ```

1. Click **Save** to create the document in the collection.

---


## Initialize a function app

Now that the Azure resources are created and Cosmos DB has been initialized, It's time to create the function app. You will create and run the function app locally on your computer. It will use the resources you have created in Azure.

Later on in the tutorial, you will deploy the function app to Azure.


### Create a new Azure Functions project

1. In a new VS Code window, use `File > Open Folder` in the menu to create and open an empty folder in an appropriate location. This will be the main project folder for the application that you will build.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/vscode-new-folder-screenshot.png)

1. Using the Azure Functions extension in VS Code, initialize a Function app in the main project folder.
    1. Open the Command Palette in VS Code by selecting `View > Command Palette` from the menu (shortcut `Ctrl-Shift-P`, macOS: `Cmd-Shift-P`).
    1. Search for the "Azure Functions: Create New Project" command and select it.
    1. The main project folder should appear. Select it (or use "Browse" to locate it).
    1. In the prompt to choose a language, select **JavaScript**.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/vscode-new-function-project-screenshot.png)



### Install function app extensions

This tutorial uses Azure Functions bindings to interact with Azure Cosmos DB and Azure SignalR Service. These bindings are available as extensions that need to be installed using the Azure Functions Core Tools CLI before they can be used.

1. Open a terminal in VS Code by selecting `View > Integrated Terminal` from the menu (Ctrl-`).

1. Ensure the main project folder is the current directory.

1. Install the Cosmos DB function app extension.
    ```
    func extensions install -p Microsoft.Azure.WebJobs.Extensions.CosmosDB -v 3.0.2
    ```

1. Install the SignalR Service function app extension.
    ```
    func extensions install -p Microsoft.Azure.WebJobs.Extensions.SignalRService -v 1.0.0-preview1-10025
    ```

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/vscode-install-func-extensions-screenshot.png)


### Configure application settings

When running and debugging the Azure Functions runtime locally, application settings are read from **local.settings.json**. Update this file with the connection strings of the Cosmos DB account and the SignalR Service instance that you created earlier.

> Note that this file is excluded from source control by default. You will use environment variables and other settings in Azure to configure your function app in the cloud.

1. In VS Code, select **local.settings.json** in the Explorer pane to open it.

1. Replace the file's contents with the following.
    ```json
    {
      "IsEncrypted": false,
      "Values": {
        "AzureWebJobsStorage": "<storage-connection-string>",
        "AzureSignalRConnectionString": "<signalr-connection-string>",
        "AzureWebJobsCosmosDBConnectionString": "<cosmosdb-connection-string>",
        "WEBSITE_NODE_DEFAULT_VERSION": "10.6.0",
        "FUNCTIONS_WORKER_RUNTIME": "node"
      },
      "Host": {
        "LocalHttpPort": 7071,
        "CORS": "http://localhost:8080",
        "CORSCredentials": true
      }
    }
    ```

    * Enter the Azure Storage account connection string into a setting named `AzureWebJobsStorage`. Obtain the value from the **Keys** page in the Storage account in the Azure portal; either than primary or secondary connection string can be used. This setting is required by Azure Functions runtime to coordinate execution between multiple function app instances.
    * Enter the Azure SignalR Service connection string into a setting named `AzureSignalRConnectionString`. Obtain the value from the **Keys** page in the Azure SignalR Service resource in the Azure portal; either than primary or secondary connection string can be used.
    * Enter the Azure Cosmos DB connection string into a setting named `CosmosDBConnectionString`. Obtain the value from the **Keys** page in the Cosmos DB account in the Azure portal; either than primary or secondary connection string can be used.
    * The `WEBSITE_NODE_DEFAULT_VERSION` setting is not used locally, but is required when deployed to Azure.
    * The `Host` section configures the port and CORS settings for the local Functions host.

1. Save the file.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/vscode-localsettings-screenshot.png)

---

## Create an Azure Function to retrieve flight prices

### GetFlights function

#### Create the function

1. Open the VS Code command palette (`Ctrl-Shift-P`, macOS: `Cmd-Shift-P`).

1. Search for and select the **Azure Functions: Create Function** command.

1. When prompted, provide the following information.

    | Name | Value |
    |---|---|
    | Function app folder | select the main project folder |
    | Template | HTTP Trigger |
    | Name | GetFlights |
    | Authorization level | Anonymous |
    
    A folder named **GetFlights** is created that contains the new function. A JavaScript Azure Function consists of two files: **function.json** and **index.js**.

1. Open **GetFlights/function.json** to configure bindings for the function. Modify the content of the file to the following.
    ```json
    {
      "disabled": false,
      "bindings": [
        {
          "authLevel": "anonymous",
          "type": "httpTrigger",
          "direction": "in",
          "name": "req"
        },
        {
          "type": "http",
          "direction": "out",
          "name": "res"
        },
        {
          "type": "cosmosDB",
          "name": "flights",
          "ConnectionStringSetting": "AzureWebJobsCosmosDBConnectionString",
          "databaseName": "flightsdb",
          "collectionName": "flights",
          "direction": "in"
        }
      ]
    }
    ```
    This adds a Cosmos DB input binding that retrieves documents from a collection named `flights` from a database named `flightsdb`. `AzureWebJobsCosmosDBConnectionString` is the name of the application setting that you configured earlier that contains the connection string to Cosmos DB.

1. Save the file.

1. Open **GetFlights/index.js** to view the body of the function. Modify the content of the file to the following.
    ```javascript
    module.exports = async function (context, req, flights) {
      context.res.body = flights;
    };
    ```
    Using the Cosmos DB input binding that you configured, Azure Functions retrieves all the documents in the specified Cosmos DB collection and passes them to the function in the `flights` argument as an array of objects. This function simply returns them directly as the HTTP response body. If you had any business logic that needs to run before returning the results, you can add them to the function.

1. Save the file.


#### (Optional) Test the function

1. To run the function app locally, press **F5** in VS Code. If this is the first time, the Azure Functions host will start in the VS Code integrated terminal.

1. When the functions runtime is successfully started, the terminal output will display a URL for the local **CreateMessage** endpoint (by default, it is `http://localhost:7071/api/GetFlights`).

1. Open Postman. Postman is an application to send HTTP requests.

1. Select `File > Import` from the menu.

1. Choose **Import from link** and paste in
    ```
    https://raw.githubusercontent.com/Azure-Samples/functions-serverless-chat-app-tutorial/master/requests/SignalRChat.postman_collection.json
    ```
    This loads a collection of HTTP requests for testing the function app locally. Click on the **Collections** tab in Postman to see it.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/postman-import-screenshot.png)

1. In the **Serverless flights** collection, select the **Get flights** request.

1. Confirm the URL matches the one outputted by the function host and there is JSON message in the request body.

1. Click **Send**. The function app should return an HTTP status of 200. The body should contain the flight(s) data you added to Cosmos DB earlier.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/postman-test-createmessage-screenshot.png)

1. In the Azure portal, open the Cosmos DB account resource you created earlier.

1. Using the Cosmos DB Data Explorer, locate the **messages** collection in the **chat** database. The message sent from Postman should appear as a document in the collection.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/cosmosdb-data-explorer-screenshot.png)

1. Click the **Disconnect** button to stop the function host and stop debugging.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/vscode-disconnect-functions-debug-screenshot.png)


## Create and run the frontend single page application

The flight application's UI is a simple single page application (SPA) created with Vue JavaScript framework. It will be hosted separately from the function app. Locally, you will run the web interface using the Live Server VS Code extension.

1. In VS Code, create a new folder named **content** at the root of the main project folder.

1. In the **content** folder, create a new file named **index.html**.

1. Copy and paste the content of **[index.html](https://raw.githubusercontent.com/Azure-Samples/functions-serverless-chat-app-tutorial/master/snippets/index.html)**.

1. Save the file.

1. Press **F5** to run the function app locally and attach a debugger.

1. Run the app using http-server that you installed earlier.
    1. In a terminal, change into the **content** folder containing **index.html**.
    1. Run `http-server`. The server should start and binds to **http://localhost:8080** by default.

    > If you are using a different web server or http-server uses a different port, you may need to update your `CORS` entry in **local.settings.json** and restart the function app.

1. Open a browser and navigate to **http://localhost:8080/**. The app should appear and populated with the correct data from Cosmos DB.

Next, you will integrate Azure SignalR Service into your application to allow any changes in data to appear in real-time.


---

## Display updates in real-time with Azure SignalR Service

Azure SignalR Service provides real-time messaging capabilities to supported clients, including web browsers. You will use Azure Functions to integrate with SignalR Service to broadcast database updates in real-time to connected browsers.


### SignalR negotiate function

The single page application uses the SignalR client SDK to connect to SignalR Service. The SDK needs to retrieve the connection information required to connect to the service. You will create an HTTP endpoint to return this information using an Azure Function. By convention, this endpoint must be called **negotiate**.

#### Create the function

1. Open the VS Code command palette (`Ctrl-Shift-P`, macOS: `Cmd-Shift-P`).

1. Search for and select the **Azure Functions: Create Function** command.

1. When prompted, provide the following information.

    | Name | Value |
    |---|---|
    | Function app folder | select the main project folder |
    | Template | HTTP Trigger |
    | Name | negotiate |
    | Authorization level | Anonymous |
    
    A folder named **negotiate** is created that contains the new function.

1. Open **negotiate/function.json** to configure bindings for the function. Modify the content of the file to the following. This adds an input binding that generates valid credentials for a client to connect to an Azure SignalR Service hub named `flights`.
    ```json
    {
        "disabled": false,
        "bindings": [
            {
                "authLevel": "anonymous",
                "type": "httpTrigger",
                "direction": "in",
                "name": "req"
            },
            {
                "type": "http",
                "direction": "out",
                "name": "res"
            },
            {
                "type": "signalRConnectionInfo",
                "name": "connectionInfo",
                "hubName": "flights",
                "direction": "in"
            }
        ]
    }
    ```

1. Open **negotiate/index.js** to view the body of the function. Modify the content of the file to the following.

    ```javascript
    module.exports = async function (context, connectionInfo) {
        context.res.body = connectionInfo;
    };
    ```

    This function takes the SignalR connection information from the input binding and returns it to the client in the HTTP response body.


#### (Optional) Test the function

1. To run the function app locally, press **F5** in VS Code. If this is the first time, the Azure Functions host will start in the VS Code integrated terminal.

1. When the functions runtime is successfully started, the terminal output will display URLs for the local endpoints, including **negotiate** (by default, it is `http://localhost:7071/api/negotiate`).

1. (See above to open Postman and import a collection) In Postman, in the **Serverless flights** collection, select the **Get SignalR negotiate** request.

1. Confirm the URL matches the one outputted by the function host and the HTTP method is **GET**.

1. Click **Send**. The function app should return connection information for SignalR Service.

1. Press the **Disconnect** button to disconnect the debugger from the function host.


### Broadcast Cosmos DB changes to all clients

#### Create the CosmosTrigger function

1. Open the VS Code command palette (`Ctrl-Shift-P`, macOS: `Cmd-Shift-P`).

1. Search for and select the **Azure Functions: Create Function** command.

1. When prompted, provide the following information.

    | Name | Value |
    |---|---|
    | Function app folder | select the main project folder |
    | Template | Azure Cosmos DB Trigger |
    | Name | CosmosTrigger |
    | App setting for your account Cosmos DB account | AzureWebJobsCosmosDBConnectionString |
    | Database name | flightsdb |
    | Collection name | flights |
    | Collection name for leases | leases |
    | Create lease collection of not exists | true |
    | Authorization level | Anonymous |
    
    A folder named **CosmosTrigger** is created that contains the new function.

1. Open **CosmosTrigger/function.json** to configure bindings for the function. Modify the content of the file to the following. This adds an input binding that generates valid credentials for a client to connect to an Azure SignalR Service hub named `flights`.
    ```json
    {
      "disabled": false,
      "bindings": [
        {
          "type": "cosmosDBTrigger",
          "name": "documents",
          "direction": "in",
          "databaseName": "flightsdb",
          "collectionName": "flights",
          "createLeaseCollectionIfNotExists": true,
          "ConnectionStringSetting": "AzureWebJobsCosmosDBConnectionString",
          "feedPollDelay": 1000
        },
        {
          "type": "signalR",
          "name": "signalRMessages",
          "hubName": "flights",
          "direction": "out"
        }
      ]
    }
    ```

    This adds the a SignalR output binding to the function that will be used to send updates to SignalR Service.

1. Open **CosmosTrigger/index.js** to view the body of the function. Modify the content of the file to the following.

    ```javascript
    module.exports = async function (context, documents) {
      context.bindings.signalRMessages =
        documents.map(flight => ({
          target: 'flightUpdated',
          arguments: [ flight ]
        }));
    };
    ```

    This function is triggered whenever one or more documents are updated in the Cosmos DB collection. Each document contains a flight whose data was updated. The function uses the SignalR output binding to raise an event named **flightUpdated** on each client that is connected to SignalR Service, passing the updated flight as the argument.


#### Test the app

1. Press **F5** to run the function app locally and attach a debugger.

1. If you shut down http-server that you started earlier, restart it in the **content** directory.

1. Browse to **http://localhost:8080/**. The app appears with the current flight prices.

1. Go to the Azure portal and navigate to the Data Explorer of the Cosmos DB account. Locate the document containing the flight information that is currently displayed in the app.

1. Modify the price of the flight in the Data Explorer and save it. The new price should change in the browser without refreshing the app.


---

## Deploy to Azure

You have been running the function app and the frontend locally. You will now deploy them to Azure.


### Log into Azure with VS Code

1. Open the VS Code command palette (`Ctrl-Shift-P`, macOS: `Cmd-Shift-P`).

1. Search for and select the **Azure: Sign in** command.

1. Follow the instructions to complete the sign in process in your browser.


### Deploy the function app

1. Open the VS Code command palette (`Ctrl-Shift-P`, macOS: `Cmd-Shift-P`).

1. Search for and select the **Azure Functions: Deploy to Function App** command.

1. When prompted, provide the following information.

    | Name | Value |
    |---|---|
    | Folder to deploy | Select the main project folder |
    | Subscription | Select your subscription |
    | Function app | Select **Create New Function App** |
    | Function app name | Enter a unique name |
    | Resource group | Select the same resource group as your other resources in this tutorial |
    | Storage account | Select the account you created earlier |
    | Location | Select a location close to you |
    
    A new function app is created in Azure and the deployment begins. The Azure Functions VS Code extension will first create the Azure resources; then it will deploy the function app. Wait for deployment to complete.


### Upload function app local settings

1. Open the VS Code command palette (`Ctrl-Shift-P`, macOS: `Cmd-Shift-P`).

1. Search for and select the **Azure Functions: Upload local settings** command.

1. When prompted, provide the following information.

    | Name | Value |
    |---|---|
    | Local settings file | local.settings.json |
    | Subscription | Select your subscription |
    | Function app | Select the previously deployed function app |
    | Function app name | Enter a unique name |

Local settings are uploaded to the function app in Azure. If prompted to overwrite existing settings, select **Yes to all**.

![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/vscode-update-function-settings-screenshot.png)



### Configure static websites in Storage

The Storage account that you're using for this tutorial is the perfect place to host the frontend in Azure. First, you need to configure it to host a static website.

1. Open the VS Code command palette (`Ctrl-Shift-P`, macOS: `Cmd-Shift-P`).

1. Search for and select the **Azure Storage: Configure static website** command.

1. Select your subscription and storage account. The browser will open directly to the page for configuring this setting.

1. Select **Enabled**.

1. Enter `index.html` as the **Index document name**.

1. Click **Save**.

1. The static websites **primary endpoint** appears on the screen. Note this value as it will be required to configure cross origin resource sharing (CORS) in the Azure Function app.


### Enable function app cross origin resource sharing (CORS)

Although there is a CORS setting in **local.settings.json**, it is not propagated to the function app in Azure. You need to set it separately.

#### Add CORS origin

1. Open the VS Code command palette (`Ctrl-Shift-P`, macOS: `Cmd-Shift-P`).

1. Search for and select the **Azure Functions: Open in portal** command.

1. Select the subscription and function app name to open the function app in the Azure portal.

1. Under the **Platform features** tab, select **CORS**.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/functions-platform-features-screenshot.png)


1. Add an entry with the static website **primary endpoint** as the value.

1. Click **Save** to persist the CORS settings.

#### Enable CORS credentials support

In order for the SignalR JavaScript SDK to function and to add authentication to the application later in the tutorial, support for credentials in CORS must be enabled.

Currently, this feature can only be enabled using the Azure command line interface (CLI) or REST APIs. You will execute a command in Azure Cloud Shell to enable this feature. This tutorial will be updated once the portal supports this feature.

1. In the Azure portal, click the **Cloud Shell** button to open a Cloud Shell terminal in the browser.

1. Execute the following command, replacing `<>` with valid values based on resources you have created.

    ```
    az resource update --resource-group <resource_group_name> --parent sites/<function_app_name> --name web --namespace Microsoft.Web --resource-type config --set properties.cors.supportCredentials=true --api-version 2015-06-01
    ```

1. Once completed, CORS credentials support is enabled in the function app.

> Note: There is currently a bug in the Azure portal, where updating the CORS origins like you did in the previous step will disable CORS credentials support. Rerun the above command if this happens. A fix is rolling out in late January.


### Update function app URL in the frontend

1. In the Azure portal, navigate to the function app's overview page.

1. Copy the function app's URL.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/functions-get-url-screenshot.png)

1. In VS Code, open **index.html** and replace the value of `window.apiBaseUrl` with the function app's URL.

1. Save the file.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/vscode-paste-function-url-screenshot.png)


### Deploy frontend to Storage

1. Open the VS Code command palette (`Ctrl-Shift-P`, macOS: `Cmd-Shift-P`).

1. Search for and select the **Azure Storage: Deploy to static website** command.

1. Select the subscription and Storage account.

1. When prompted for a folder, select **content**.


...