---
title: 'Tutorial: Create and manage a Virtual Machine Scale Set with the Azure CLI'
description: Learn how to use the Azure CLI to create a Virtual Machine Scale Set, along with some common management tasks such as how to start and stop an instance or change the scale set capacity.
ms.topic: tutorial
ms.date: 06/14/2024
author: ju-shim
ms.author: jushiman
ms.custom: devx-track-azurecli, innovation-engine
---

# Tutorial: Create and manage a Virtual Machine Scale Set with the Azure CLI

A Virtual Machine Scale Set allows you to deploy and manage a set of virtual machines. Throughout the lifecycle of a Virtual Machine Scale Set, you may need to run one or more management tasks. In this tutorial, you learn how to:

- Create a resource group
- Create a Virtual Machine Scale Set
- Scale out and in
- Stop, Start, and restart VM instances

## Create a resource group

Start by creating a resource group to hold your Virtual Machine Scale Set and related resources. This example creates a resource group named **myResourceGroup$RANDOM_SUFFIX** in the **centralindia** region.

```bash
export RANDOM_SUFFIX=$(openssl rand -hex 3)
export REGION="centralindia"
export RESOURCE_GROUP="myResourceGroup$RANDOM_SUFFIX"
az group create --name $RESOURCE_GROUP --location $REGION
```

### Results:

<!-- expected_similarity=0.3 -->

```json
{
  "id": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/myResourceGroupxxx",
  "location": "centralindia",
  "managedBy": null,
  "name": "myResourceGroupxxx",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null,
  "type": "Microsoft.Resources/resourceGroups"
}
```

## Create a scale set

Next, create a Virtual Machine Scale Set. This example uses a flexible orchestration mode and generates a new SSH key pair if one does not already exist. Adjust the `--image` value to one of the valid Azure Marketplace images (e.g., `Ubuntu2204`), and use a VM SKU size that is currently available, such as `Standard_DS1_v2`.

```bash
export SCALE_SET_NAME="myScaleSet"
az vmss create \
  --resource-group $RESOURCE_GROUP \
  --name $SCALE_SET_NAME \
  --orchestration-mode flexible \
  --image Ubuntu2204 \
  --vm-sku Standard_DS1_v2 \
  --admin-username azureuser \
  --generate-ssh-keys
```

### Results:

<!-- expected_similarity=0.3 -->

```json
{
  "id": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/myResourceGroupxxx/providers/Microsoft.Compute/virtualMachineScaleSets/myScaleSet",
  "location": "centralindia",
  "name": "myScaleSet",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "type": "Microsoft.Compute/virtualMachineScaleSets"
}
```

## Scale the scale set

Change the number of VM instances in the scale set. This example scales the Virtual Machine Scale Set to 3 instances.

```bash
az vmss scale \
  --resource-group $RESOURCE_GROUP \
  --name $SCALE_SET_NAME \
  --new-capacity 3
```

### Verify the instances:

```bash
az vmss list-instances --resource-group $RESOURCE_GROUP --name $SCALE_SET_NAME --output table
```

### Results:

```text
InstanceId      ResourceGroup      Location       InstanceState
-------------  -----------------  --------------  --------------
1               myResourceGroup    centralindia    Succeeded
2               myResourceGroup    centralindia    Succeeded
3               myResourceGroup    centralindia    Succeeded
```

## Stop and deallocate VM instances

To stop all instances of the Virtual Machine Scale Set, run the following command:

```bash
az vmss stop \
  --resource-group $RESOURCE_GROUP \
  --name $SCALE_SET_NAME
```

To deallocate specific instances and save costs, specify the instance ID instead of the instance name:

```bash
export INSTANCE_ID="1"
az vmss deallocate \
  --resource-group $RESOURCE_GROUP \
  --name $SCALE_SET_NAME \
  --instance-ids $INSTANCE_ID
```

## Start VM instances

To restart all instances in the scale set:

```bash
az vmss start \
  --resource-group $RESOURCE_GROUP \
  --name $SCALE_SET_NAME
```

Alternatively, start a specific instance using the instance ID:

```bash
az vmss start \
  --resource-group $RESOURCE_GROUP \
  --name $SCALE_SET_NAME \
  --instance-ids $INSTANCE_ID
```

## Restart VM instances

To restart all instances in the scale set:

```bash
az vmss restart \
  --resource-group $RESOURCE_GROUP \
  --name $SCALE_SET_NAME
```

To restart a specific instance using the instance ID:

```bash
az vmss restart \
  --resource-group $RESOURCE_GROUP \
  --name $SCALE_SET_NAME \
  --instance-ids $INSTANCE_ID
```

## Summary

In this tutorial, you:

- Created a resource group
- Deployed and scaled a Virtual Machine Scale Set
- Managed VM instances in the scale set, including stopping, starting, and restarting

You can use these commands to manage scale sets programmatically or through the Azure CLI.
