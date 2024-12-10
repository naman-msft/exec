---
title: Create a Highly Available PostgreSQL Cluster on Azure Kubernetes Service (AKS) using Terraform
description: This tutorial shows how to create a Highly Available PostgreSQL cluster on AKS using the CloudNativePG operator
author: russd2357,kenkilty
ms.author: rdepina,kenkilty
ms.topic: article
ms.date: 06/26/2024
ms.custom: innovation-engine, linux-related content
---
# Create a Highly Available PostgreSQL Cluster on Azure Kubernetes Service (AKS) using Terraform.

In this guide, you will deploy a highly-available PostgreSQL cluster that spans multiple Azure availability zones. You will walk through the steps required to set up the PostgreSQL cluster running on [Azure Kubernetes Service](https://learn.microsoft.com/en-us/azure/aks/what-is-aks) (AKS) and perform basic Postgres operations such as backup and restore. 


## Installing Terraform

1. Update Package Index
Before installing any software, it’s a good practice to update your package index. This ensures that you have the latest information about available packages

```bash
sudo apt-get update
```


2. Install Required Packages
You need wget to download files from the internet and unzip to extract the downloaded files. Install them using the following command:

```bash
sudo apt-get install -y wget unzip
```


3. Download Terraform
Use wget to download the latest version of Terraform. You can find the latest version on the Terraform releases page. For example, to download version 1.5.0:

```bash
wget https://releases.hashicorp.com/terraform/1.5.0/terraform_1.5.0_linux_amd64.zip
```

4. Unzip the Downloaded File
After downloading, you need to extract the Terraform binary from the zip file:

```bash
unzip terraform_1.5.0_linux_amd64.zip
```


5. Move Teffaform to a Directory in Your PATH
To make Terraform accessible from anywhere in your terminal, move it to /usr/local/bin:

```bash
sudo mv terraform /usr/local/bin/
```


6. Verify the Installation
Finally, check if Terraform is installed correctly by checking its version:

```bash
terraform -v
```

Results:
<!-- expected_similarity=0.3 -->
```output
Terraform v1.5.0
```


## Creating a Highly Available PostgreSQL Cluster on Azure Kubernetes Service (AKS) Using Terraform

1. Create a Terraform Configuration File Create a file named main.tf with the following content:

```text
provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "rg" {
  name     = "pg-ha-rg"
  location = "West Europe"
}

resource "azurerm_kubernetes_cluster" "aks" {
  name                = "pg-ha-aks"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = "pgha"

  agent_pool_profile {
    name       = "agentpool"
    count      = 3
    vm_size    = "Standard_DS2_v2"  # SKU for AKS
    os_type    = "Linux"
    mode       = "System"
  }

  identity {
    type = "SystemAssigned"
  }

  role_based_access_control {
    enabled = true
  }
}

resource "azurerm_postgresql_server" "pg_server" {
  name                = "pg-ha-server"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  version             = "11"
  sku {
    name     = "B_Gen5_2"  # SKU for PostgreSQL
    tier     = "Basic"
    capacity = 2
  }
  storage_profile {
    storage_mb = 5120
  }
  administrator_login          = "pgadmin"
  administrator_login_password = "YourPassword123!"
  ssl_enforcement_enabled      = true
}

resource "azurerm_postgresql_database" "pg_database" {
  name                = "mydatabase"
  resource_group_name = azurerm_resource_group.rg.name
  server_name         = azurerm_postgresql_server.pg_server.name
  charset             = "UTF8"
  collation           = "English_United States.1252"
}
```


2. Initialize Terraform Run the following command to initialize your Terraform configuration:

```bash
terraform init
```

Results:
<!-- expected_similarity=0.3 -->
```output
Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/azurerm versions matching ">= 2.0.0"...
- Installing hashicorp/azurerm v2.0.0...
- Installed hashicorp/azurerm v2.0.0 (signed by HashiCorp)

Terraform has been successfully initialized!
```


3. Validate the Configuration Check if your configuration is valid:

```bash
terraform validate
```

Results:
<!-- expected_similarity=0.3 -->
```output
Success! The configuration is valid.
```


4. Plan the Deployment Generate an execution plan:

```bash
terraform plan
```

Results:
<!-- expected_similarity=0.3 -->
```output
Terraform will perform the following actions:

  # azurerm_kubernetes_cluster.aks will be created
  + resource "azurerm_kubernetes_cluster" "aks" {
      ...
    }

  # azurerm_postgresql_server.pg_server will be created
  + resource "azurerm_postgresql_server" "pg_server" {
      ...
    }

Plan: 3 to add, 0 to change, 0 to destroy.
```


5. Apply the Configuration Deploy the resources:

```bash
terraform apply -auto-approve
```

Results:
<!-- expected_similarity=0.3 -->
```output
azurerm_resource_group.rg: Creating...
azurerm_resource_group.rg: Creation complete after 5s [id=/subscriptions/.../resourceGroups/pg-ha-rg]
azurerm_kubernetes_cluster.aks: Creating...
azurerm_postgresql_server.pg_server: Creating...
...
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
```


6. Verify the Deployment Check the status of the AKS cluster:

```bash
az aks show --resource-group pg-ha-rg --name pg-ha-aks --output table
```

Results:
<!-- expected_similarity=0.3 -->
```output
Name         ResourceGroup   Location    KubernetesVersion    ProvisioningState
-----------  --------------- ----------- -------------------- -------------------
pg-ha-aks   pg-ha-rg       West Europe     1.20.7               Succeeded
```


7. Connect to PostgreSQL To connect to your PostgreSQL server, you can use the following command:

```bash
psql "host=pg-ha-server.postgres.database.azure.com dbname=mydatabase user=pgadmin@pg-ha-server password=YourPassword123! sslmode=require"
```

Results:
<!-- expected_similarity=0.3 -->
```output
psql (12.3)
Type "help" for help.

mydatabase=#
```

8. Deploy a Sample Application To test the PostgreSQL setup, you can deploy a simple application. Create a file named app-deployment.yaml with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pg-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: pg-app
  template:
    metadata:
      labels:
        app: pg-app
    spec:
      containers:
      - name: pg-app
        image: postgres:11
        env:
        - name: POSTGRES_DB
          value:
```

## Steps to Test Application
1. Expose the Application First, you need to create a service to expose your application. Create a file named app-service.yaml with the following content:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: pg-app-service
spec:
  type: LoadBalancer
  ports:
    - port: 5432
      targetPort: 5432
  selector:
    app: pg-app
```

Apply this configuration to your AKS cluster:

```bash
kubectl apply -f app-service.yaml
```

Results:
<!-- expected_similarity=0.3 -->
```output
service/pg-app-service created
```

2. Check the Status of the Service After exposing the application, check the status of the service to get the external IP address:

```bash
kubectl get services
```

Results:
<!-- expected_similarity=0.3 -->
```output
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
pg-app-service     LoadBalancer   10.0.0.1       <pending>       5432:XXXXX/TCP   1m
```

Wait a few moments until the EXTERNAL-IP is assigned. It may take a couple of minutes.


3. Connect to the Application Once the external IP is assigned, you can connect to the PostgreSQL database using the following command. Replace <EXTERNAL-IP> with the actual external IP address you obtained from the previous step:

```bash
psql "host=<EXTERNAL-IP> dbname=mydatabase user=pgadmin@pg-ha-server password=YourPassword123! sslmode=require"
```

Results:
<!-- expected_similarity=0.3 -->
```output
psql (12.3)
Type "help" for help.

mydatabase=#
```

4. Clean Up Resources 
When done, destroy the resources:

```bash
terraform destroy -auto-approve
```

Results:
<!-- expected_similarity=0.3 -->
```output
Results:
<!-- expected_similarity=0.3 -->
```output
psql (12.3)
Type "help" for help.

mydatabase=#
```



To learn more about AKS and walk through a complete code-to-deployment example, continue to the Kubernetes cluster tutorial.

> [!div class="nextstepaction"]
> [AKS tutorial][aks-tutorial]

<!-- LINKS - external -->
[kubectl]: https://kubernetes.io/docs/reference/kubectl/
[kubectl-apply]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply
[kubectl-get]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#get

<!-- LINKS - internal -->
[kubernetes-concepts]: ../concepts-clusters-workloads.md
[aks-tutorial]: ../tutorial-kubernetes-prepare-app.md
[azure-resource-group]: ../../azure-resource-manager/management/overview.md
[az-aks-create]: /cli/azure/aks#az-aks-create
[az-aks-get-credentials]: /cli/azure/aks#az-aks-get-credentials
[az-aks-install-cli]: /cli/azure/aks#az-aks-install-cli
[az-group-create]: /cli/azure/group#az-group-create
[az-group-delete]: /cli/azure/group#az-group-delete
[kubernetes-deployment]: ../concepts-clusters-workloads.md#deployments-and-yaml-manifests
[aks-solution-guidance]: /azure/architecture/reference-architectures/containers/aks-start-here?toc=/azure/aks/toc.json&bc=/azure/aks/breadcrumb/toc.json
[baseline-reference-architecture]: /azure/architecture/reference-architectures/containers/aks/baseline-aks?toc=/azure/aks/toc.json&bc=/azure/aks/breadcrumb/toc.json