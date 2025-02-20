---
published: true
type: workshop
title: Build an Intelligent App
short_title: Build an Intelligent App
description: In this workshop you will learn how to build an intelligent app leveraging Azure services
level: intermediate # Required. Can be 'beginner', 'intermediate' or 'advanced'
navigation_numbering: false
authors: # Required. You can add as many authors as needed
  - Fethi Dilmi
  - Louis-Guillaume Morand

contacts: # Required. Must match the number of authors
  - "@fethidilmi"
  - "@lgmorand"
duration_minutes: 400
tags: azure, ipaas, IA, openai, cosmosdb, bing
navigation_levels: 3
---

# Introduction

## Build and Modernize AI Applications Hackathon

CosmicWorks has big plans for their retail site. They're eager to launch a POC of a simple chat interface where users can interact with a virtual agent to find product and account information.

The scenario centers around a consumer retail "Intelligent Agent" that allows users to ask questions on vectorized product, customer and sales order data stored in a database. The data in this solution is the Cosmic Works sample for Azure Cosmos DB. This data is an adapted subset of the Adventure Works 2017 dataset for a retail Bike Shop that sells bicycles, biking accessories, components and clothing.

<div class="tip" data-title="Tip">

> BUT you can bring your own data instead.

</div>

This hackathon will challenge you and your team to launch a POC of a chat interface where users can interact with a virtual agent to find product and account information. Through the course of the hackathon, you will modify an existing application to do the following:

- Store the chat messages in an **Azure Cosmos DB database**, grouped by chat sessions
- Use **Azure OpenAI Service** to create vector embeddings and chat completions
- Use **Azure Cognitive Search** as a vector database to search for product and account information by the vector embeddings
- Load up existing product and account information into Azure Cosmos DB and the Azure Cognitive Search vector index
- Create a process that manages the conversation flow, vectorization, search, data handling, and response generation
- Externally manage system prompts

## Prerequisites

- Azure Subscription
- Visual Studio (Communitity Edition is fine for this lab)
- .NET 7 SDK
- Docker Desktop
- Azure CLI 2.69.0
- Helm v3.17.0 or greater - [https://helm.sh/](https://helm.sh/) (for AKS)

## Setting up your development environment

The following steps will guide you through the process needed to being the hackathon.

### Clone this repo

Clone this repository and change to the `main` branch

```pwsh
git clone https://github.com/lgmorand/tiger
cd .\tiger\
git checkout main
```

### Deploy the Azure core services

1. Open the PowerShell command line and navigate to the directory where you cloned the repo.
2. Navigate into the `code\VectorSearchAiAssistant` folder.
3. Run the following PowerShell script to provision the infrastructure and deploy the API and frontend. Provide the name of a NEW resource group that will be created. This will provision all of the required infrastructure, deploy the API and web app services into Azure Kubernetes Service (AKS) if using the deployAks flag below or Azure Container Apps (ACA), and import data into Cosmos DB.

```pwsh
./scripts/Starter-Deploy.ps1  -resourceGroup <resource-group-name> -location <location> -subscription <subscription-id> -deployAks 1
```

<div class="tip" data-title="Note">

>**NOTE**:
>
>If `<resource-group-name>` already exists, your user must have `Owner` permissions on the resource group.
>If `<resource-group-name>` does not exist exists, the deployment script will create it. In this case, your user must have `Owner` permissions on the subscription in which the resource group will be created.

</div>

> **NOTE**:
>
>By default, the deployment script will attempt to create new Azure Open AI model deployments for the `gpt-35-turbo` and `text-embedding-ada-002` models. If you already have deployments for these models, you can skip the deployment by passing the following parameters to the script:

>```pwsh
>-openAiName <open-ai-name> -openAiRg <open-ai-resource-group> -openAiCompletionsDeployment <completions-deployment-name> -openAiEmbeddingsDeployment <embeddings-deployment-name> -stepDeployOpenAi $false
>```
>In case you will defer the Open AI deployment to the script, make sure have enough Tokens Per Minute (TPM) in thousands quota available in your subscription. By default, the script will attempt to set a value of 120K for each deployment. In case you need to change this value, you can edit lines 22 and 29 in the `code\VectorSearchAiAssistant\scripts\Deploy-OpenAi.ps1` file.

>If using your own Azure OpenAI account, it will be necessary to update the appsettings.json file in the `ChatServiceWebApi` project to use the model deployment names used in your existing Azure OpenAI account. See **Configure Local Settings section below** for more details.

### Decide on the containerization approach

The deployment script supports two types of containerization:

- [Azure Kubernetes Service - AKS](https://azure.microsoft.com/services/kubernetes-service) - this option allows you to deploy the application into an AKS cluster. This option is more complex to set up, but it provides more flexibility and control over the deployment. To use AKS, pass the following parameter to the deployment script:
  ```pwsh
  -deployAks $true
  ```
- [Azure Container Apps - ACA](https://azure.microsoft.com/products/container-apps) - this option allows you to deploy containerized applications without having to manage the underlying infrastructure and thus is an easier option to get started with.

For the purpose of this hackathon, you can whatever your preference is, either AKS or ACA.

>**NOTE**:
>
>For the reminder of this hackathon, please interpret any documentation references to `AKS` as `ACA` if you chose to use Azure Container Apps (and vice-versa).

### Verify initial deployment

1. After the command completes, navigate to resource group and obtain the name of the AKS service.
2. Execute the following command to obtain the website's endpoint:

    For ACA:

    ```pwsh
    az deployment group show -g <resource-group-name> -n cosmosdb-openai-azuredeploy -o json --query properties.outputs.webFqdn.value
    ```

    For AKS:

    ```pwsh
    az aks show -n <aks-name> -g <resource-group-name> -o tsv --query addonProfiles.httpApplicationRouting.config.HTTPApplicationRoutingZoneName
    ```

1. Browse to the website with the returned hostname.

If the website loads, you are ready to continue with the hackathon challenges. Don't worry if the website is not fully operational yet - you will get it there!

## Service Architecture

After the deployment is complete the following Azure services will be deployed.

- Azure OpenAI Service
- Azure Cosmos DB
- Azure Cognitive Search
- Azure Container Apps (or AKS)
- Azure Storage
- Azure Networking (*not pictured*)

<p align="center">
    <img src="assets/intro/architecture.png" width="100%">
</p>

## Run the solution locally using Visual Studio

You can run the website and the REST API that supports it locally. You need to first update your local configuration and then you can run the solution in the debugger using Visual Studio.

#### Configure local settings

- In the `ChatServiceWebApi` project, review the content of the `appsettings.json` file.

>**NOTE**:
>
> If you deploy using a pre-existing Azure OpenAI account. You will need to update `CompletionsDeployment` and `EmbeddingsDeployment` values to match the names used for these models in your Azure OpenAI account.

```json
{
    "Logging": {
        "LogLevel": {
            "Default": "Information",
            "Microsoft.AspNetCore": "Warning",
            "Microsoft.SemanticKernel": "Error"
        },
        "ApplicationInsights": {
            "LogLevel": {
                "Default": "Information",
                "Microsoft.AspNetCore": "Warning",
                "Microsoft.SemanticKernel": "Error"
            }
        }
    },
    "AllowedHosts": "*",
    "MSCosmosDBOpenAI": {
        "CognitiveSearch": {
            "IndexName": "vector-index",
            "MaxVectorSearchResults": 10
        },
        "OpenAI": {
            "CompletionsDeployment": "completions",
            "CompletionsDeploymentMaxTokens": 8096,
            "EmbeddingsDeployment": "embeddings",
            "EmbeddingsDeploymentMaxTokens": 8191,
            "ChatCompletionPromptName": "RetailAssistant.Default",
            "ShortSummaryPromptName": "Summarizer.TwoWords",
            "PromptOptimization": {
                "CompletionsMinTokens": 50,
                "CompletionsMaxTokens": 300,
                "SystemMaxTokens": 1500,
                "MemoryMinTokens": 1500,
                "MemoryMaxTokens": 7000,
                "MessagesMinTokens": 100,
                "MessagesMaxTokens": 200
            }
        },
        "CosmosDB": {
            "Containers": "completions, customer, product",
            "MonitoredContainers": "customer, product",
            "Database": "database",
            "ChangeFeedLeaseContainer": "leases"
        },
        "DurableSystemPrompt": {
            "BlobStorageContainer": "system-prompt"
        },
        "CognitiveSearchMemorySource": {
            "IndexName": "vector-index",
            "ConfigBlobStorageContainer": "memory-source",
            "ConfigFilePath": "ACSMemorySourceConfig.json"
        },
        "BlobStorageMemorySource": {
            "ConfigBlobStorageContainer": "memory-source",
            "ConfigFilePath": "BlobMemorySourceConfig.json"
        }
    }
}
```

  - In the `ChatServiceWebApi` project, create an `appsettings.Development.json` file with the following content (replace all `<...>` placeholders with the values from your deployment):

```json
{
    "MSCosmosDBOpenAI": {
        "CognitiveSearch": {
            "Endpoint": "https://<...>.search.windows.net",
            "Key": "<...>"
        },
        "OpenAI": {
            "Endpoint": "https://<...>.openai.azure.com/",
            "Key": "<...>"
        },
        "CosmosDB": {
            "Endpoint": "https://<...>.documents.azure.com:443/",
            "Key": "<...>"
        },
        "DurableSystemPrompt": {
            "BlobStorageConnection": "<...>"
        },
        "BlobStorageMemorySource": {
            "ConfigBlobStorageConnection": "<...>"
        },
        "CognitiveSearchMemorySource": {
            "Endpoint": "https://<...>.search.windows.net",
            "Key": "<...>",
            "ConfigBlobStorageConnection": "<...>"
        }
    }
}
```

> **NOTE**: THe `BlobStorageConnection` value can be found in the Azure Portal by navigating to the Storage Account created by the deployment (the one that has a container named `system-prompt`) and selecting the `Access keys` blade. The value is the `Connection string` for the `key1` key.

### Running in debug

To run locally and debug using Visual Studio, open the solution file to load the projects and prepare for debugging.

Before you can start debugging, you need to set the startup projects. To do this, right-click on the solution in the Solution Explorer and select `Set Startup Projects...`. In the dialog that opens, select `Multiple startup projects` and set the `Action` for the `ChatServiceWebApi` and `Search` projects to `Start`.

Also, make sure the newly created `appsettings.Development.json` file is copied to the output directory. To do this, right-click on the file in the Solution Explorer and select `Properties`. In the properties window, set the `Copy to Output Directory` property to `Copy always`..

You are now ready to start debugging the solution locally. To do this, press `F5` or select `Debug > Start Debugging` from the menu.

**NOTE**: With Visual Studio, you can also use alternate ways to manage the secrets and configuration. For example, you can use the `Manage User Secrets` option from the context menu of the `ChatWebServiceApi` project to open the `secrets.json` file and add the configuration values there.

---

# Challenge 1: The Landing Before the Launch

CosmicWorks has big plans for their retail site, but they need to start somewhere; they need a landing zone in Azure for all of their services. It will take a while to prepare their e-Commerce site to migrate to Azure, but they're eager to launch a POC of a simple chat interface where users can interact with a virtual agent to find product and account information.

They've created a simple Blazor web application for the UI elements and have asked you to to incorporate the backend plumbing to do the following:

- Store the chat history in an Azure Cosmos DB database
  - They expect the following types of messages: Session (for the chat session), Message (the user and assistant message).
  - A message should have a sender (Assistant or User), tokens (that indicates how many tokens were used), text (the text from the assistant or the user), rating (thumbs up or down) and vector (the vector embedding of the user's text).
- Source the customer and product data from the Azure Comos DB database.
- Use Azure OpenAI service to create vector embeddings and chat completions.
- Use a Azure Cognitive Search to search for relevant product and account information by the vector embeddings.
- Encapsulate the orchestration of interactions with OpenAI behind a back-end web service.
- Create a storage account to externalize prompts that will be used by your assistant.

For this challenge, you will deploy the services into the landing zone in preparation for the launch of the POC.

## Challenge

Your team must:

1. Clone the Starter repo with the Blazor web application and starter artifacts
2. Set up your development environment
3. Deploy the Azure services needed to support the chat interface

<div class="tip" data-title="Hints">

> - CosmicWorks has provided you a script to deploy the foundation of your Azure environment. See the instructions in the README.md of the repo.
> - You will need to deploy the following Azure services within a new Resource Group:
>  - Azure Cosmos DB
>  - Azure OpenAI service
>  - Azure Cognitive Search
>  - Azure Container Apps service that will host:
>    - A web service that supports running the front-end Blazor web app in a Docker container
>    - A web service that supports running the back-end web API in a Docker container
>  - Azure Storage Account (not enabled for hierarchical namespace)
>- You will load data and deploy code into these services in a later challenge.

</div>

### Success Criteria

To complete this challenge successfully, you must:

- Deploy Azure Cosmos DB with the NoSQL API. It should have a database named `database` and containers named `completions` with a partition key of `/sessionId`, `customer` with a partition key of `/customerId`, `embedding` with a partition key of `/id`, `product` with a partition key of `/categoryId`, and `leases` with a partition key of `/id`. You only need to deploy the account to a single region. Make a best guess at the RU/s for each container. You will adjust these in a later challenge based on performance and cost.
- Deploy Azure OpenAI with the following deployments:
  - `completions` with the `gpt-35-turbo` model
  - `embeddings` with the `text-embedding-ada-002` model
- Deploy Azure Cognitive Search in the basic tier.
- Deploy Azure Container Apps, Azure Container Registry and Azure Storage Account.

### Resources

- [Azure Cosmos DB](https://learn.microsoft.com/azure/cosmos-db/)
- [Azure OpenAI service](https://learn.microsoft.com/azure/cognitive-services/openai/overview)
- [Azure Cognitive Search](https://learn.microsoft.com/azure/search/)
- [Azure Container Apps](https://learn.microsoft.com/azure/container-apps/start)

## Explore Further

- [Understanding embeddings](https://learn.microsoft.com/azure/cognitive-services/openai/concepts/understand-embeddings)

---

# Challenge 2: It's All About the Payload

CosmicWorks has years of product, customer, and sales data that they exported to an Azure storage account. They want to load this data into the Azure Cosmos DB account for a future migration of their e-Commerce site to Azure, but also for the POC of the chat interface. They also want to load the data into Azure Cognitive Search so that they can search for product and account information by the vector embeddings.

One critical component of the magic that makes the CosmicWorks chat interface work is the ability to search for products and accounts by their vector embeddings. When a user types a question into the chat interface, we need to create a vector embedding for the question, then search for the most similar vector embeddings of products and accounts, and then return those similar documents. The vector embeddings for products and accounts are stored in a vector database, allowing us to return relevant context documents that get sent to Azure OpenAI's completions endpoint.

CosmicWorks has done some research, and they would like to use Microsoft Semantic Kernel as the framework the code uses to orchestrate calls to the Azure OpenAI embeddings and completions endpoints. They've provided some incomplete starter code to give you an idea of what they are looking for.

## Challenge

Your team must:

1. Implement an efficient and repeatable way to load product and customer data from the storage account into Cosmos DB. For this exercise, you only need to load the data once, but CosmicWorks wants to be able to repeat the process in the future with new data. Cosmicworks has provided the data for you to start with, listed in the resources below.
2. Create a vector index in Azure Cognitive Search. They had some ideas on this that they provided in the starter project.
3. Create a process to index product and customer data from Cosmos DB using the change feed to load the documents into an Azure Cognitive Search vector index. They have provided a starter template for you that they had created for another effort.
4. Verify that the data was loaded into Cosmos DB and Cognitive Search.

<div class="tip" data-title="Hints">

> - CosmicWorks suggest using the Azure Cosmos DB Desktop Data Migration Tool to load their sample files from their Azure Storage into your instance of Cosmos DB. They suggest you do a "Quick Install" of the tool. They have provided the `migrationsettings.template.json` in the root of the starter repo that contains the parameters you should use with this tool. You need to replace the `{{cosmosConnectionString}}` instances in the JSON file with the connection string to your deployed instance of Cosmos DB.
> - Search thru the solution for the `TODO: Challenge 2` comments and follow the instructions provided.
> - Think about how you can use the Cosmos DB Change Feed to trigger the creation of vector embeddings for new/updated products and customers.
> - Think about how you build the logic for accessing the Azure OpenAI service to perform the vector embeddings of the product and customer documents. You will use this same logic layer to perform other Azure OpenAI tasks in later challenges.

</div>

### Success Criteria

To complete this challenge successfully, you must:

- Demonstrate to your coach that you can load the data from the storage account into Cosmos DB using a method that can be repeated in the future.
- Perform a document count on each container in Cosmos DB and verify that the counts match the number of documents in the storage account.
- Generate vector embeddings of a sufficiently high dimensionality that is supported by the Azure OpenAI service as well as your vector database.
- Encapsulate the embedding logic within a service layer that can be used by other components of the CosmicWorks chat interface, as well as a future REST API service.
- Create a process that automatically generates vector embeddings for all of the products and customers in the Cosmos DB database and stores them in the vector database.
- Demonstrate to your coach a successful search for products and customers by vector embeddings. This does not necessarily have to originate from the chat interface at this point.

### Resources

- [Sample Customer Data](https://cosmosdbcosmicworks.blob.core.windows.net/cosmic-works-small/customer.json)
- [Sample Product Data](https://cosmosdbcosmicworks.blob.core.windows.net/cosmic-works-small/product.json)
- [Azure Cosmos DB Desktop Data Migration Tool](https://github.com/AzureCosmosDB/data-migration-desktop-tool)
- [Query Azure Search using Search Explorer](https://learn.microsoft.com/azure/search/search-explorer)
- [Work with data using Azure Cosmos DB Explorer](https://learn.microsoft.com/en-us/azure/cosmos-db/data-explorer)

## Explore Further

- [Understanding embeddings](https://learn.microsoft.com/azure/cognitive-services/openai/concepts/understand-embeddings)
- [Semantic Kernel](https://learn.microsoft.com/semantic-kernel/overview/)

---

# Challenge 3: Now We're Flying

With the critical components in place, we're ready to tie everything into the chat interface. When a user types a question into the chat interface, we need to create a vector embedding for the question, then search for the most similar vector embeddings for products and accounts, and return the relevant documents that get sent to Azure OpenAI's completions endpoint.

In order to return a human-friendly response to the user, we need to use the completions endpoint to generate a response based on the most relevant documents and an instructional system-level prompt. Furthermore, we need to keep a history of the user's questions and the responses that were generated so that they can reload the chat in the future.

To generate prompts for the Azure OpenAI service, the approach is to use a technique called *prompt engineering* to author prompts that are used to guide the generation of completions. Prompt engineering is an iterative process that involves authoring prompts, generating completions, and evaluating the results.

The starter solution uses Semantic Kernel to orchestrate the execution of prompts. This challenge is about experimenting with system prompts to impact how the completions work.

## Challenge

Your team must:

1. Use the Azure OpenAI service to create vector embeddings for the user prompt that is entered into the chat interface. Invoke the completions endpoint to generate a response based on the most relevant documents and some instructional system-level prompts. The system prompt should be included with every completions call, but not repeated in the chat history. Use Semantic Kernel as they stubbed out in the project to make this call.
2. Create the system prompt that defines the assistant's behavior. CosmicWorks has provided you with a starter prompt located under VectorSearchAiAssistant\SystemPrompts\RetailAssistant\Default.txt. You should add to the content in this file. The system prompt should instruct the model to do the following:
   1. Tell it that it is an intelligent assistant for a bike company.
   2. Tell it that it is responding to user questions about products, product categories, customers, and sales order information provided in JSON format embedded below.
   3. Only answer questions related to the information provided.
   4. Not to "make up" information and to respond that it does not know the answer to suggest to the user to search for it themselves.
   5. Make sure the prompt ends with "Text of relevant information:" as after that the system will inject context data and chat history.
3. Upload the system prompt file you created at the previous step to the Azure Storage Account, place it under the path `system-prompt / RetailAssistant` overwriting the file that is there.
4. Store the user's questions and the responses that were generated so the system can reload them in the future.

<div class="tip" data-title="Hints">

- CosmicWorks has provided starter code for you. Search for the two methods with `TODO: Challenge 3` and complete them as instructed.
- Think carefully about the system prompt, about how it should respond, what knowledge it is allowed to use when reasoning to create a response, what subjects it is allowed to respond to and importantly what it should not respond to.
- Have the agent reject off topic prompts from the user (such as asks to tell a joke).

</div>

### Success Criteria

To complete this challenge successfully, you must:

- Demonstrate to your coach that you can load the system prompt from the storage account.
- Interact with the assistant thru the web based chat interface.
- View the chat messages saved to the container in Cosmos DB and verify that your new User and Assistant messages are appearing.
- Try a variety of user prompts to see how the assistant responds.

### Resources

- [Intro to prompt engineering](https://learn.microsoft.com/en-us/azure/cognitive-services/openai/concepts/prompt-engineering)

## Explore Further

- [Writing Effective System Prompts](https://learn.microsoft.com/azure/cognitive-services/openai/concepts/system-message)

---

# Challenge 4: What's Your Vector, Victor?

Now it's time to see the end-to-end process in action. In this challenge, you will load new data using the data loading mechanism you created in a previous challenge, then observe the automatic vectorization in action. Once you ingest the new data, ask a question through the prompt interface and see if it returns an answer about the new data you loaded.

## Challenge

Your team must:

1. Load new data using the data loading mechanism you created in a previous challenge.
2. Ask a question through the prompt interface and see if it returns an answer about the new data you loaded.

<div class="tip" data-title="Hints">

- Cosmicworks has provided the JSON files containing the initial products and customers that you loaded into the system, take one of these and modify it to create some new products or customers and uploaded it to the storage account from where you loaded the initial data and run your data loading process.
- Experiment using prompts to elicit different response formats from the completions model:
   - Respond with a single number or with one or two words
   - Respond with a bulleted lists or formatted a certain way
   - Respond using simpler syntax (e.g. explain it like I'm five)
   - Challenge the model with prompts that require reasoning or chain of thought. For example, ask it to calculate aggregates on the data or go further and give some word problems like those you had in school.
   - Challenge the model to explain its reasoning
   - Request that the model list its sources

</div>

### Success Criteria

To complete this challenge successfully, you must:

- Show your coach the new data that you created and then your chat history showing how it responded using the new data as context.
- Try to locate the new product or customer data you loaded in the Cognitive Search Index and in Cosmos DB.

### Resources

- [Query Azure Search using Search Explorer](https://learn.microsoft.com/azure/search/search-explorer)
- [Work with data using Azure Cosmos DB Explorer](https://learn.microsoft.com/en-us/azure/cosmos-db/data-explorer)

## Explore Further

- [Prompt engineering techniques](https://learn.microsoft.com/azure/cognitive-services/openai/concepts/advanced-prompt-engineering?pivots=programming-language-chat-completions)

---

# Challenge 5: It's All About the Payload, The Sequel

What you've built so far is a pattern you can use with any type of data. You've built a process that can be used to load data from a storage account into Cosmos DB and then run a process that stores both documents and vector embeddings in Azure Cognitive Search. Now it's time to test your pattern by loading a completely new type of data.

The solution provided by Cosmicworks is specific to Product, Customer and SalesOrder data. In this challenge you will extend the solution so that it can handle any type of data in the JSON format.

## Challenge

Your team must:

1. Create a container named `sourcedata` in Cosmos DB.
2. Create a new entity for the JSON document data type in the starter project.
3. Create a new change feed processor that monitors the `sourcedata` container and works with instances of your new data type.
4. Load data of the new type into Cosmos DB.
5. Use the chat interface to ask questions about the new data type.

<div class="tip" data-title="Hints">

- With the starter solution supplied by CosmicWorks open in Visual Studio, expand the VectorSearchAiAssistant.Service project, Models, Search and take a look at Product.cs. This class is required to process the data with the Cosmos DB change feed and is also used as the schema for the document added to the Cognitive Search index. You will need to define an entity similar to this for your new type of data.
- Extend the implementation of the `ModelRegistry` class to include your newly created data type.
- Review the implementation of the change feed processor located in the same project under Services, CosmosDbService.cs to validate it is ready to use your new data type.
- In SemanticKernelRAGService.cs update the setup of the `_memoryTypes` in the SemanticKernelRAGService constructor to include your new type that will be used to initialize the Search index.

</div>

### Success Criteria

To complete this challenge successfully, you must:

- Show your coach the new data that you created and then your chat history showing how it responded using the new data as context.
- Try to locate the new product or customer data you loaded in the Cognitive Search Index and in Cosmos DB.

### Resources

- [Change feed processor in Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/change-feed-processor?tabs=dotnet)

## Explore Further

- [Reading from the change feed](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/read-change-feed)

---

# Challenge 6: The Colonel Needs a Promotion

In this challenge, you and your team will add a new capability by creating a couple of new plugins for Semantic Kernel.

You have lately become a big fan of Shakespeare's sonnets. You love how they convey details and information, and you want your copilot to provide details about your company's policies in the poetic form of Shakespearean language.

Your challenge is correctly handling requests about your company's policies (return or shipping), including when users ask to get them as poems written in Shakespearean style.

## Challenge

Your team must:

1. Create a new Semantic Kernel plugin that will retrieve only memories related to company policies.
2. Create a new Semantic Kernel plugin that will present any information as a poem written in Shakespearean language.
3. Register the plugins with Semantic Kernel.
4. Create a plan to respond to requests.
5. Replace the logic in the SemanticKernalRAGService.cs GetResponse method with one that will first make a plan to decide if your functions should be used or not, and then execute the completion request accordingly.

<div class="tip" data-title="Hints">

- You might want to try building this first in a simple console project.
- You should use the SequentialPlanner from Semantic Kernel to create and execute a plan around the prompt, so that it can choose when to invoke your plugins.
- You will have to update how you handle the completion response from the SequentialPlanner.

</div>

### Success Criteria

To complete this challenge successfully, you must:

- Show your coach an example chat where your new plugins where selected by the plan and executed to produce the completion.

### Resources

- [Semantic Kernel auto create plans with planner](https://learn.microsoft.com/semantic-kernel/ai-orchestration/planner?tabs=Csharp)
- [Semantic Kernel creating native functions](https://learn.microsoft.com/en-us/semantic-kernel/ai-orchestration/native-functions?tabs=Csharp)

## Explore Further

[Microsoft Semantic Kernel on Github](https://github.com/microsoft/semantic-kernel)

---

# Challenge 7: Getting Into the Flow

Up until now, you have used a web service (ChatServiceWebAPI) that utilizes an instance of the ChatService singleton to  orchestrate calls to Azure OpenAI and Azure Cognitive Search by using Semantic Kernel. This effectively provides the "smarts" to your AI assistant. This is not the only way that you could build these smarts.

In this challenge, you and your team will use Azure ML Prompt Flow to replace portions of the ChatService singleton.

## Challenge

Your team must:

1. Create an Azure Machine Learning Prompt Flow that re-creates the core steps of the ChatService, which are:
    - Get user query vector embedding
    - Search for context data
    - Request the completion
    - Store and return the result

<div class="tip" data-title="Hints">

- For all interactions with Azure OpenAI, you will want to use the LLM Tool in Prompt Flow.
- To search for context data from Cognitive Search you will want the Vector DB Lookup Tool.
- For storing the results back to Cosmos DB, consider using the Python Tool.

</div>

### Success Criteria

To complete this challenge successfully, you must:
- Demonstrate to your coach the PromptFlow you created in Azure Machine Learning.
- Deploy your Prompt Flow endpoint and integrate that into the solution.

### Resources

- [Get stated with Prompt Flow](https://learn.microsoft.com/en-us/azure/machine-learning/prompt-flow/get-started-prompt-flow?view=azureml-api-2#create-and-develop-your-prompt-flow)
- [Integrate LangChain in PromptFlow](https://learn.microsoft.com/en-us/azure/machine-learning/prompt-flow/how-to-integrate-with-langchain?view=azureml-api-2)
- [Deploy a flow as a managed online endpoint for real-time inference](https://learn.microsoft.com/en-us/azure/machine-learning/prompt-flow/how-to-deploy-for-real-time-inference?view=azureml-api-2)

## Explore Further

[The Prompt Flow GitHub repository](https://github.com/microsoft/promptflow)

# Teardown

When you have finished with the hackathon, simply delete the resource group that was created.