---
title: 'Tutorial: Implement RAG on Azure Cognitive Services with a Chat Interface'
description: Learn how to implement Retrieval-Augmented Generation (RAG) using Azure Cognitive Services, LangChain, ChromaDB, and Chainlit, and deploy it in Azure Container Apps.
ms.topic: tutorial
ms.date: 10/10/2023
author: GitHubCopilot
ms.author: GitHubCopilot
ms.custom: innovation-engine
---

# Tutorial: Implement RAG on Azure Cognitive Services with a Chat Interface

This tutorial will guide you through the steps to implement Retrieval-Augmented Generation (RAG) using Azure Cognitive Services, LangChain, ChromaDB, and Chainlit, and deploy it in Azure Container Apps.

## Prerequisites

- An Azure account with an active subscription.
- Azure CLI installed on your local machine.
- Docker installed on your local machine.
- Python 3.9 or higher installed on your local machine.

## Step 1: Create an Azure OpenAI Service

1. **Create a Resource Group**

   Set the following environment variables:

   ```bash
   export RANDOM_SUFFIX=$(openssl rand -hex 3)
   export RESOURCE_GROUP="myResourceGroup$RANDOM_SUFFIX"
   export LOCATION="eastus"
   ```

   Create the resource group:

   ```azurecli-interactive
   az group create --name $RESOURCE_GROUP --location $LOCATION
   ```

   Results:

   <!-- expected_similarity=0.3 -->

   ```JSON
   {
     "id": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/myResourceGroupxxx",
     "location": "eastus",
     "managedBy": null,
     "name": "myResourceGroupxxx",
     "properties": {
       "provisioningState": "Succeeded"
     },
     "tags": null,
     "type": "Microsoft.Resources/resourceGroups"
   }
   ```

2. **Create an Azure OpenAI Service**

   Set the following environment variable:

   ```bash
   export OPENAI_SERVICE_NAME="myOpenAIService$RANDOM_SUFFIX"
   ```

   Create the Azure OpenAI Service:

   ```azurecli-interactive
   az cognitiveservices account create \
     --name $OPENAI_SERVICE_NAME \
     --resource-group $RESOURCE_GROUP \
     --kind OpenAI \
     --sku S0 \
     --location $LOCATION \
     --yes
   ```

## Step 2: Set Up LangChain, ChromaDB, and Prepare the Data

1. **Create a Working Directory**

   ```bash
   mkdir rag-chat-app
   cd rag-chat-app
   ```

2. **Create a Virtual Environment**

   ```bash
   python3 -m venv venv
   source venv/bin/activate
   ```

3. **Create a `requirements.txt` File**

   ```plaintext
   langchain
   chromadb
   chainlit
   openai
   tiktoken
   ```

4. **Install the Dependencies**

   ```bash
   pip install -r requirements.txt
   ```

5. **Prepare the Data**

   Create a `documents.txt` file with sample content:

   ```bash
   echo "Azure Cognitive Services provide AI capabilities for developers to build intelligent applications." > documents.txt
   ```

6. **Create an `app.py` File**

   ```python
   import os
   from langchain.document_loaders import TextLoader
   from langchain.indexes import VectorstoreIndexCreator
   from langchain.chains import ConversationalRetrievalChain
   from langchain.embeddings import OpenAIEmbeddings
   from langchain.llms import OpenAI
   import chainlit as cl

   # Set Azure OpenAI API credentials
   os.environ["OPENAI_API_TYPE"] = "azure"
   os.environ["OPENAI_API_KEY"] = os.getenv("OPENAI_API_KEY")
   os.environ["OPENAI_API_BASE"] = os.getenv("OPENAI_API_BASE")
   os.environ["OPENAI_API_VERSION"] = "2023-03-15-preview"

   # Load documents
   loader = TextLoader('documents.txt')
   documents = loader.load()

   # Create index
   index = VectorstoreIndexCreator().from_loaders([loader])

   # Create conversational retrieval chain
   retriever = index.vectorstore.as_retriever()
   qa_chain = ConversationalRetrievalChain.from_llm(
       llm=OpenAI(temperature=0),
       retriever=retriever
   )

   # Initialize conversation history
   history = []

   @cl.on_message
   async def main(message):
       global history
       result = qa_chain({"question": message, "chat_history": history})
       history.append((message, result['answer']))
       await cl.Message(content=result['answer']).send()
   ```

7. **Set Environment Variables**

   ```bash
   export OPENAI_API_KEY="<Your Azure OpenAI Key>"
   export OPENAI_API_BASE="https://$OPENAI_SERVICE_NAME.openai.azure.com/"
   ```

## Step 3: Test the Application Locally

Run the Chainlit app locally:

```bash
chainlit run app.py -w
```

Results:

<!-- expected_similarity=0.3 -->

```log
[20:15:30] INFO:     Chainlit server is running at http://localhost:8000
[20:15:30] INFO:     Running app.py
```

Open your browser and navigate to `http://localhost:8000` to interact with your chat app.

## Step 4: Containerize the Application

1. **Create a `Dockerfile`**

   ```dockerfile
   FROM python:3.9-slim

   WORKDIR /app

   COPY . /app

   RUN pip install --no-cache-dir -r requirements.txt

   EXPOSE 8000

   CMD ["chainlit", "run", "app.py", "-w", "--host", "0.0.0.0", "--port", "8000"]
   ```

2. **Build the Docker Image**

   ```bash
   docker build -t mychainlitapp .
   ```

3. **Test the Docker Image Locally**

   ```bash
   docker run -p 8000:8000 mychainlitapp
   ```

## Step 5: Push the Docker Image to Azure Container Registry

1. **Create Azure Container Registry**

   Set the following environment variable:

   ```bash
   export ACR_NAME="myContainerRegistry$RANDOM_SUFFIX"
   ```

   Create the registry:

   ```azurecli-interactive
   az acr create --resource-group $RESOURCE_GROUP --name $ACR_NAME --sku Basic
   ```

   Results:

   <!-- expected_similarity=0.3 -->

   ```JSON
   {
     "adminUserEnabled": false,
     "loginServer": "mycontainerregistryxxx.azurecr.io",
     "name": "myContainerRegistryxxx",
     "resourceGroup": "myResourceGroupxxx",
     "location": "eastus",
     ...
   }
   ```

2. **Login to Azure Container Registry**

   ```azurecli-interactive
   az acr login --name $ACR_NAME
   ```

3. **Tag and Push the Docker Image**

   ```bash
   export ACR_LOGIN_SERVER=$(az acr show --name $ACR_NAME --query loginServer -o tsv)
   docker tag mychainlitapp $ACR_LOGIN_SERVER/mychainlitapp:v1
   docker push $ACR_LOGIN_SERVER/mychainlitapp:v1
   ```

## Step 6: Deploy to Azure Container Apps

1. **Register the Container Apps Extension**

   ```azurecli-interactive
   az extension add --name containerapp --upgrade
   ```

2. **Create a Container Apps Environment**

   Set the following environment variable:

   ```bash
   export CONTAINERAPPS_ENVIRONMENT="myContainerAppEnv$RANDOM_SUFFIX"
   ```

   Create the environment:

   ```azurecli-interactive
   az containerapp env create --name $CONTAINERAPPS_ENVIRONMENT --resource-group $RESOURCE_GROUP --location $LOCATION
   ```

3. **Deploy the Container App**

   Set the following environment variable:

   ```bash
   export CONTAINER_APP_NAME="myChainlitApp$RANDOM_SUFFIX"
   ```

   Deploy the app:

   ```azurecli-interactive
   az containerapp create \
     --name $CONTAINER_APP_NAME \
     --resource-group $RESOURCE_GROUP \
     --environment $CONTAINERAPPS_ENVIRONMENT \
     --image $ACR_LOGIN_SERVER/mychainlitapp:v1 \
     --target-port 8000 \
     --ingress external \
     --registry-server $ACR_LOGIN_SERVER \
     --cpu 0.5 --memory 1.0Gi
   ```

## Step 7: Test the Deployment

1. **Get the URL of the Container App**

   ```azurecli-interactive
   az containerapp show --name $CONTAINER_APP_NAME --resource-group $RESOURCE_GROUP --query properties.configuration.ingress.fqdn -o tsv
   ```

2. **Test the Chat App**

   Open the URL in your browser to interact with your RAG-enabled chat app.

By following these steps, you have successfully implemented Retrieval-Augmented Generation using Azure Cognitive Services with a chat interface and deployed it to Azure Container Apps.