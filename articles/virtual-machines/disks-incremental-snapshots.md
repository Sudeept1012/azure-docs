---
title: Create an incremental snapshot
description: Learn about incremental snapshots for managed disks, including how to create them using the Azure portal, Azure PowerShell module, and Azure Resource Manager.
author: roygara
ms.service: storage
ms.topic: how-to
ms.date: 03/30/2022
ms.author: rogarana
ms.subservice: disks
ms.custom: devx-track-azurepowershell, ignite-fall-2021, devx-track-azurecli 
ms.devlang: azurecli
---

# Create an incremental snapshot for managed disks

**Applies to:** :heavy_check_mark: Linux VMs :heavy_check_mark: Windows VMs :heavy_check_mark: Flexible scale sets :heavy_check_mark: Uniform scale sets

[!INCLUDE [virtual-machines-disks-incremental-snapshots-description](../../includes/virtual-machines-disks-incremental-snapshots-description.md)]

## Restrictions

[!INCLUDE [virtual-machines-disks-incremental-snapshots-restrictions](../../includes/virtual-machines-disks-incremental-snapshots-restrictions.md)]

# [Azure CLI](#tab/azure-cli)

You can use the Azure CLI to create an incremental snapshot. You will need the latest version of the Azure CLI. See the following articles to learn how to either [install](/cli/azure/install-azure-cli) or [update](/cli/azure/update-azure-cli) the Azure CLI.

The following script will create an incremental snapshot of a particular disk:

```azurecli
# Declare variables
diskName="yourDiskNameHere"
resourceGroupName="yourResourceGroupNameHere"
snapshotName="desiredSnapshotNameHere"

# Get the disk you need to backup
yourDiskID=$(az disk show -n $diskName -g $resourceGroupName --query "id" --output tsv)

# Create the snapshot
az snapshot create -g $resourceGroupName -n $snapshotName --source $yourDiskID --incremental true
```

You can identify incremental snapshots from the same disk with the `SourceResourceId` property of snapshots. `SourceResourceId` is the Azure Resource Manager resource ID of the parent disk.

You can use `SourceResourceId` to create a list of all snapshots associated with a particular disk. Replace `yourResourceGroupNameHere` with your value and then you can use the following example to list your existing incremental snapshots:


```azurecli
# Declare variables and create snapshot list
subscriptionId="yourSubscriptionId"
resourceGroupName="yourResourceGroupNameHere"
diskName="yourDiskNameHere"

az account set --subscription $subscriptionId

diskId=$(az disk show -n $diskName -g $resourceGroupName --query [id] -o tsv)

az snapshot list --query "[?creationData.sourceResourceId=='$diskId' && incremental]" -g $resourceGroupName --output table
```


# [Azure PowerShell](#tab/azure-powershell)

You can use the Azure PowerShell module to create an incremental snapshot. You will need the latest version of the Azure PowerShell module. The following command will either install it or update your existing installation to latest:

```PowerShell
Install-Module -Name Az -AllowClobber -Scope CurrentUser
```

Once that is installed, login to your PowerShell session with `Connect-AzAccount`.

To create an incremental snapshot with Azure PowerShell, set the configuration with [New-AzSnapShotConfig](/powershell/module/az.compute/new-azsnapshotconfig) with the `-Incremental` parameter and then pass that as a variable to [New-AzSnapshot](/powershell/module/az.compute/new-azsnapshot) through the `-Snapshot` parameter.

```PowerShell
$diskName = "yourDiskNameHere"
$resourceGroupName = "yourResourceGroupNameHere"
$snapshotName = "yourDesiredSnapshotNameHere"

# Get the disk that you need to backup by creating an incremental snapshot
$yourDisk = Get-AzDisk -DiskName $diskName -ResourceGroupName $resourceGroupName

# Create an incremental snapshot by setting the SourceUri property with the value of the Id property of the disk
$snapshotConfig=New-AzSnapshotConfig -SourceUri $yourDisk.Id -Location $yourDisk.Location -CreateOption Copy -Incremental 
New-AzSnapshot -ResourceGroupName $resourceGroupName -SnapshotName $snapshotName -Snapshot $snapshotConfig 
```

You can identify incremental snapshots from the same disk with the `SourceResourceId` and the `SourceUniqueId` properties of snapshots. `SourceResourceId` is the Azure Resource Manager resource ID of the parent disk. `SourceUniqueId` is the value inherited from the `UniqueId` property of the disk. If you were to delete a disk and then create a new disk with the same name, the value of the `UniqueId` property changes.

You can use `SourceResourceId` and `SourceUniqueId` to create a list of all snapshots associated with a particular disk. Replace `yourResourceGroupNameHere` with your value and then you can use the following example to list your existing incremental snapshots:

```PowerShell
$resourceGroupName = "yourResourceGroupNameHere"
$snapshots = Get-AzSnapshot -ResourceGroupName $resourceGroupName

$incrementalSnapshots = New-Object System.Collections.ArrayList
foreach ($snapshot in $snapshots)
{
    
    if($snapshot.Incremental -and $snapshot.CreationData.SourceResourceId -eq $yourDisk.Id -and $snapshot.CreationData.SourceUniqueId -eq $yourDisk.UniqueId){

        $incrementalSnapshots.Add($snapshot)
    }
}

$incrementalSnapshots
```

# [Portal](#tab/azure-portal)
[!INCLUDE [virtual-machines-disks-incremental-snapshots-portal](../../includes/virtual-machines-disks-incremental-snapshots-portal.md)]

# [Resource Manager Template](#tab/azure-resource-manager)

You can also use Azure Resource Manager templates to create an incremental snapshot. You'll need to make sure the apiVersion is set to **2019-03-01** and that the incremental property is also set to true. The following snippet is an example of how to create an incremental snapshot with Resource Manager templates:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "diskName": {
      "type": "string",
      "defaultValue": "contosodisk1"
    },
  "diskResourceId": {
    "defaultValue": "<your_managed_disk_resource_ID>",
    "type": "String"
  }
  }, 
  "resources": [
  {
    "type": "Microsoft.Compute/snapshots",
    "name": "[concat( parameters('diskName'),'_snapshot1')]",
    "location": "[resourceGroup().location]",
    "apiVersion": "2019-03-01",
    "properties": {
      "creationData": {
        "createOption": "Copy",
        "sourceResourceId": "[parameters('diskResourceId')]"
      },
      "incremental": true
    }
  }
  ]
}
```
---

## Cross-region snapshot copy

You can copy incremental snapshots to any region of your choice. Azure manages the copy process removing the maintenance overhead of managing the copy process by staging a storage account in the target region. Moreover, Azure ensures that only changes since the last snapshot in the target region are copied to the target region to reduce the data footprint, reducing the recovery point objective. You can check the progress of the copy so you can know when a target snapshot is ready to restore disks in the target region. Customers are charged only for the bandwidth cost of the data transfer across the region.

:::image type="content" source="media/disks-incremental-snapshots/cross-region-snapshot.png" alt-text="Diagram of Azure orchestrated cross-region copy of incremental snapshots via the clone option." lightbox="media/disks-incremental-snapshots/cross-region-snapshot.png":::

### Restrictions

- You can copy 100 incremental snapshots in parallel at the same time per subscription per region.
- If you use the REST API, you must use version 2020-12-01 or newer of the Azure Compute REST API.

### Get started

# [Azure CLI](#tab/azure-cli)

You can use the Azure CLI to copy an incremental snapshot. You will need the latest version of the Azure CLI. See the following articles to learn how to either [install](/cli/azure/install-azure-cli) or [update](/cli/azure/update-azure-cli) the Azure CLI.

The following script will copy an incremental snapshot from one region to another:

```azurecli
subscriptionId=<yourSubscriptionID>
resourceGroupName=<yourResourceGroupName>
name=<targetSnapshotName>
sourceSnapshotResourceId=<sourceSnapshotResourceId>
targetRegion=<validRegion>

sourceSnapshotId=$(az snapshot show -n $sourceSnapshotName -g $resourceGroupName --query [id] -o tsv)

az snapshot create -g $resourceGroupName -n $targetSnapshotName --source $sourceSnapshotId --incremental --copy-start

az snapshot show -n $sourceSnapshotName -g $resourceGroupName --query [completionPercent] -o tsv
```

# [Azure PowerShell](#tab/azure-powershell)

You can use the Azure PowerShell module to copy an incremental snapshot. You will need the latest version of the Azure PowerShell module. The following command will either install it or update your existing installation to latest:

```PowerShell
Install-Module -Name Az -AllowClobber -Scope CurrentUser
```

Once that is installed, login to your PowerShell session with `Connect-AzAccount`.

The following script will copy an incremental snapshot from one region to another.

```azurepowershell
$subscriptionId="yourSubscriptionIdHere"
$resourceGroupName="yourResourceGroupNameHere"
$sourceSnapshotName="yourSourceSnapshotNameHere"
$targetSnapshotName="yourTargetSnapshotNameHere"
$targetRegion="desiredRegion"

Set-AzContext -Subscription $subscriptionId

$sourceSnapshot=Get-AzSnapshot -ResourceGroupName $resourceGroupName -SnapshotName $sourceSnapshotName

$snapshotconfig = New-AzSnapshotConfig -Location $targetRegion -CreateOption CopyStart -Incremental -SourceResourceId $sourceSnapshot.Id

New-AzSnapshot -ResourceGroupName $resourceGroupName -SnapshotName $targetSnapshotName -Snapshot $snapshotconfig

$targetSnapshot=Get-AzSnapshot -ResourceGroupName $resourceGroupName -SnapshotName $targetSnapshotName

$targetSnapshot.CompletionPercent
```

# [Portal](#tab/azure-portal)

You can also copy an incremental snapshot across regions in the Azure portal. However, you must use this specific link to access the portal, for now: https://aka.ms/incrementalsnapshot 

1. Sign in to the [Azure portal](https://aka.ms/incrementalsnapshot) and navigate to the incremental snapshot you'd like to migrate.
1. Select **Copy snapshot**.

    :::image type="content" source="media/disks-incremental-snapshots/disks-copy-snapshot.png" alt-text="Screenshot of snapshot overview, copy snapshot highlighted." lightbox="media/disks-incremental-snapshots/disks-copy-snapshot.png":::

1. For **Snapshot type** under **Instance details** select **Incremental**.
1. Change the **Region** to the region you'd like to copy the snapshot to.

    :::image type="content" source="media/disks-incremental-snapshots/disks-copy-snapshot-region-select.png" alt-text="Screenshot of copy snapshot experience, new region selected, incremental selected." lightbox="media/disks-incremental-snapshots/disks-copy-snapshot-region-select.png":::

1. Select **Review + Create** and then **Create**.

# [Resource Manager Template](#tab/azure-resource-manager)

You can also use Azure Resource Manager templates to copy an incremental snapshot. You must use version **2020-12-01** or newer of the Azure Compute REST API. The following snippet is an example of how to copy an incremental snapshot across regions with Resource Manager templates:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "name": {
            "defaultValue": "isnapshot1",
            "type": "String"
        },
        "sourceSnapshotResourceId": {
            "defaultValue": "<your_incremental_snapshot_resource_ID>",
            "type": "String"
        },
        "skuName": {
            "defaultValue": "Standard_LRS",
            "type": "String"
        },
        "targetRegion": {
            "defaultValue": "desired_region",
            "type": "String"
        }
    },
    "variables": {},
    "resources": [
        {
            "type": "Microsoft.Compute/snapshots",
            "sku": {
                "name": "[parameters('skuName')]",
                "tier": "Standard"
            },
            "name": "[parameters('name')]",
            "apiVersion": "2020-12-01",
            "location": "[parameters('targetRegion')]",
            "scale": null,
            "properties": {
                "creationData": {
                    "createOption": "CopyStart",
                    "sourceResourceId": "[parameters('sourceSnapshotResourceId')]"
                },
                "incremental": true
            },
            "dependsOn": []
        }
    ]
}

```
---

## Next steps

If you'd like to see sample code demonstrating the differential capability of incremental snapshots, using .NET, see [Copy Azure Managed Disks backups to another region with differential capability of incremental snapshots](https://github.com/Azure-Samples/managed-disks-dotnet-backup-with-incremental-snapshots).
