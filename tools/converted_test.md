---
title: 'Tutorial: Create & manage a Virtual Machine Scale Set – Azure CLI'
description: Learn how to use the Azure CLI to create a Virtual Machine Scale Set, along with some common management tasks such as how to start and stop an instance, or change the scale set capacity.
ms.topic: tutorial
ms.date: 06/14/2024
author: ju-shim
ms.author: jushiman
ms.custom: innovation-engine, mimckitt, devx-track-azurecli
---
```

# Tutorial: Create and manage a Virtual Machine Scale Set with the Azure CLI

```bash
# Section: Create a Resource Group
# Random suffix ensures uniqueness for resource group names; corrected consistent use of variable
export RANDOM_SUFFIX=$(openssl rand -hex 3)
export REGION="eastus"
export RESOURCE_GROUP_NAME="MyResourceGroup$RANDOM_SUFFIX"

az group create --name $RESOURCE_GROUP_NAME --location $REGION
```

<!-- expected_similarity=0.3 -->

```JSON
{
  "id": "/subscriptions/xxxxx-xxxxx-xxxxx-xxxxx/resourceGroups/MyResourceGroupXXX",
  "location": "eastus",
  "managedBy": null,
  "name": "MyResourceGroupXXX",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null,
  "type": "Microsoft.Resources/resourceGroups"
}
```

---

```bash
# Section: Create a Scale Set
# Updated variable usage for consistency
export SCALE_SET_NAME="MyScaleSet$RANDOM_SUFFIX"
export IMAGE="UbuntuLTS" # Replace with a valid image SKU as needed
export ADMIN_USERNAME="azureuser"

az vmss create \
  --resource-group $RESOURCE_GROUP_NAME \
  --name $SCALE_SET_NAME \
  --orchestration-mode flexible \
  --image $IMAGE \
  --admin-username $ADMIN_USERNAME \
  --generate-ssh-keys
```

<!-- expected_similarity=0.3 -->

```JSON
{
  "provisioningState": "Succeeded",
  "id": "/subscriptions/xxxxx-xxxxx-xxxxx-xxxxx/resourceGroups/MyResourceGroupXXX/providers/Microsoft.Compute/virtualMachineScaleSets/MyScaleSetXXX",
  "name": "MyScaleSetXXX",
  "type": "Microsoft.Compute/virtualMachineScaleSets",
  "location": "eastus",
  "properties": {
    "orchestrationMode": "Flexible",
    "overprovision": false,
    "...additional-metadata..."
  }
}
```

---

```bash
# Section: List VM Instances in the Scale Set
# Using updated resource group variable
az vm list --resource-group $RESOURCE_GROUP_NAME --output table
```

<!-- expected_similarity=0.3 -->

```text
Name                 ResourceGroup            Location    Zones
-------------------  -----------------------  ----------  -------
MyScaleSetXXX_instance1  MyResourceGroupXXX       eastus
MyScaleSetXXX_instance2  MyResourceGroupXXX       eastus
```

---

```bash
# Section: Change the Capacity of the Scale Set
az vmss scale \
  --resource-group $RESOURCE_GROUP_NAME \
  --name $SCALE_SET_NAME \
  --new-capacity 3
```

<!-- expected_similarity=0.3 -->

```JSON
{
  "count": 3,
  "provisioningState": "Updating"
}
```

---

```bash
# Section: Stop all VM Instances in the Scale Set
az vmss stop \
  --resource-group $RESOURCE_GROUP_NAME \
  --name $SCALE_SET_NAME
```

---

```bash
# Section: Start all VM Instances in the Scale Set
az vmss start \
  --resource-group $RESOURCE_GROUP_NAME \
  --name $SCALE_SET_NAME
```

---

```bash
# Section: Restart all VM Instances in the Scale Set
az vmss restart \
  --resource-group $RESOURCE_GROUP_NAME \
  --name $SCALE_SET_NAME
```

---

```bash
# Section: Deallocate a Specific VM Instance
# Replace <INSTANCE_NAME> with an actual instance name from the scale set
export INSTANCE_NAME="<INSTANCE_NAME>"
az vm deallocate \
  --resource-group $RESOURCE_GROUP_NAME \
  --name $INSTANCE_NAME
```

---

```bash
# Section: Delete the Resource Group
# Resource group deletion, which removes all associated resources, is demonstrated but commented out for safety
# az group delete --name $RESOURCE_GROUP_NAME --no-wait --yes
```

By following this guide, you can easily create, manage, and perform basic operations on a Virtual Machine Scale Set using the Azure CLI.
