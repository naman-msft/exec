---
title: 快速入門：使用 Azure CLI 建立 Red Hat Enterprise Linux 虛擬機
description: 在本快速入門中，您將瞭解如何使用 Azure CLI 來建立 Red Hat Enterprise Linux 虛擬機
author: namanparikh
ms.service: virtual-machines
ms.collection: linux
ms.topic: quickstart
ms.date: 05/03/2024
ms.author: namanparikh
ms.custom: 'mvc, devx-track-azurecli, mode-api, innovation-engine, linux-related-content'
---

# 快速入門：在 Azure 上使用 Azure CLI 建立 Red Hat Enterprise Linux 虛擬機

**適用於：：** heavy_check_mark：Linux VM

[![部署至 Azure](https://aka.ms/deploytoazurebutton)](https://go.microsoft.com/fwlink/?linkid=2262692)

本快速入門說明如何使用 Azure CLI 在 Azure 中部署 Red Hat Enterprise Linux 虛擬機（VM）。 Azure CLI 可用來透過命令列或指令碼建立和管理 Azure 資源。

如尚未擁有 Azure 訂用帳戶，請在開始之前先建立[免費帳戶](https://azure.microsoft.com/free/?WT.mc_id=A261C142F)。

## 啟動 Azure Cloud Shell

Azure Cloud Shell 是免費的互動式 Shell，可讓您用來執行本文中的步驟。 它具有預先安裝和設定的共用 Azure 工具，可與您的帳戶搭配使用。 

若要開啟 Cloud Shell，只要選取程式碼區塊右上角的 [試試看]**** 即可。 您也可以移至 [https://shell.azure.com/bash](https://shell.azure.com/bash)，在另一個瀏覽器索引標籤中開啟 Cloud Shell。 選取 [複製]**** 即可複製程式碼區塊，將它貼到 Cloud Shell 中，然後選取 **Enter** 鍵加以執行。

如果您偏好在本機安裝和使用 CLI，本快速入門需要有 Azure CLI 2.0.30 版或更新版本。 執行 `az --version` 以尋找版本。 如果您需要安裝或升級，請參閱[安裝 Azure CLI]( /cli/azure/install-azure-cli)。

## 定義環境變數

第一個步驟是定義環境變數。 環境變數通常用於 Linux，以集中設定設定，以改善系統的一致性和可維護性。 建立以下環境變數以指定您在本教學課程後面建立的資源的名稱：

```bash
export RANDOM_ID="$(openssl rand -hex 3)"
export MY_RESOURCE_GROUP_NAME="myVMResourceGroup$RANDOM_ID"
export REGION="westeurope"
export MY_VM_NAME="myVM$RANDOM_ID"
export MY_USERNAME=azureuser
export MY_VM_IMAGE="RedHat:RHEL:8-LVM:latest"
```

## 使用 CLI 登入 Azure

為了使用 CLI 在 Azure 中執行命令，您需要先登入。 使用 `az login` 命令登入。

## 建立資源群組

資源群組是相關資源的容器。 所有資源都必須放置在資源群組中。 [az group create](/cli/azure/group) 命令會使用先前定義的 $MY_RESOURCE_GROUP_NAME 和 $REGION 參數來建立資源群組。

```bash
az group create --name $MY_RESOURCE_GROUP_NAME --location $REGION
```

結果：

<!-- expected_similarity=0.3 -->
```json
{
  "id": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/myVMResourceGroup",
  "location": "eastus",
  "managedBy": null,
  "name": "myVMResourceGroup",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null,
  "type": "Microsoft.Resources/resourceGroups"
}
```

## 建立虛擬機器

若要在此資源群組中建立 VM，請使用 `vm create` 命令。 

下列範例會建立 VM 並新增使用者帳戶。 `--generate-ssh-keys` 參數會使 CLI 在 `~/.ssh` 中尋找可用的 ssh 金鑰。 如果找到一個，則會使用該金鑰。 如果沒有找到，則會產生一個並儲存在 `~/.ssh` 中。 `--public-ip-sku Standard` 參數可確保機器可透過公用 IP 位址來加以存取。 最後，我們會部署最新的 `Ubuntu 22.04` 映像。

所有其他值都是使用環境變數來進行設定。

```bash
az vm create \
    --resource-group $MY_RESOURCE_GROUP_NAME \
    --name $MY_VM_NAME \
    --image $MY_VM_IMAGE \
    --admin-username $MY_USERNAME \
    --assign-identity \
    --generate-ssh-keys \
    --public-ip-sku Standard
```

建立虛擬機器和支援資源需要幾分鐘的時間。 下列範例輸出顯示 VM 建立作業成功。

結果：
<!-- expected_similarity=0.3 -->
```json
{
  "fqdns": "",
  "id": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/myVMResourceGroup/providers/Microsoft.Compute/virtualMachines/myVM",
  "location": "eastus",
  "macAddress": "00-0D-3A-10-4F-70",
  "powerState": "VM running",
  "privateIpAddress": "10.0.0.4",
  "publicIpAddress": "52.147.208.85",
  "resourceGroup": "myVMResourceGroup",
  "zones": ""
}
```

## 在 Azure 中為 Linux 虛擬機器啟用 Azure AD 登入

下列程式碼範例會部署 Linux VM，然後安裝延伸模組以為 Linux VM 啟用 Azure AD 登入。 VM 擴充功能是小型的應用程式，可在「Azure 虛擬機器」上提供部署後設定及自動化工作。

```bash
az vm extension set \
    --publisher Microsoft.Azure.ActiveDirectory \
    --name AADSSHLoginForLinux \
    --resource-group $MY_RESOURCE_GROUP_NAME \
    --vm-name $MY_VM_NAME
```

## 儲存 VM 的 IP 位址以便進行 SSH 連線

執行以下命令，將 VM 的 IP 位址儲存為環境變數：

```bash
export IP_ADDRESS=$(az vm show --show-details --resource-group $MY_RESOURCE_GROUP_NAME --name $MY_VM_NAME --query publicIps --output tsv)
```

## 透過 SSH 連線到 VM

<!--## Export the SSH configuration for use with SSH clients that support OpenSSH & SSH into the VM.
Log in to Azure Linux VMs with Azure AD supports exporting the OpenSSH certificate and configuration. That means you can use any SSH clients that support OpenSSH-based certificates to sign in through Azure AD. The following example exports the configuration for all IP addresses assigned to the VM:-->

<!--
```bash
yes | az ssh config --file ~/.ssh/config --name $MY_VM_NAME --resource-group $MY_RESOURCE_GROUP_NAME
```
-->

現在您可以在您選擇的 ssh 用戶端中執行以下命令的輸出來透過 SSH 連線到 VM：

```bash
ssh -o StrictHostKeyChecking=no $MY_USERNAME@$IP_ADDRESS
```

## 後續步驟

* [了解虛擬機器](../index.yml)
* [使用 Cloud-Init 在第一次開機時初始化 Linux VM](tutorial-automate-vm-deployment.md)
* [建立自訂的 VM 映像](tutorial-custom-images.md)
* [平衡 VM 的負載](../../load-balancer/quickstart-load-balancer-standard-public-cli.md)
