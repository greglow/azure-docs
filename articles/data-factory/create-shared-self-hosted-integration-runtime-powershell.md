---
title: Create a shared self-hosted integration runtime in Azure Data Factory with PowerShell | Microsoft Docs
description: Learn how to create a shared self-hosted integration runtime in Azure Data Factory, which lets multiple data factories access the integration runtime.
services: data-factory
documentationcenter: ''
author: nabhishek
manager: craigg

ms.service: data-factory
ms.workload: data-services
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: conceptual
ms.date: 10/31/2018
ms.author: abnarain
---
# Create a shared self-hosted integration runtime in Azure Data Factory with PowerShell

This step-by-step guide shows you how to create a shared self-hosted integration runtime (IR) in Azure Data Factory using Azure PowerShell. Then you can use the shared self-hosted integration runtime in another data factory. In this tutorial, you do the following steps: 

1. Create a data factory. 
1. Create a self-hosted integration runtime.
1. Share the self-hosted integration runtime with other data factories.
1. Create a linked integration runtime.
1. Revoke the sharing.

## Prerequisites 

- **Azure subscription**. If you don't have an Azure subscription, [create a free account](https://azure.microsoft.com/free/) before you begin. 

- **Azure PowerShell**. Follow the instructions in [Install Azure PowerShell on Windows](/powershell/azure/install-azurerm-ps). You use PowerShell to run a script to create a self-hosted integration runtime that can be shared with other data factories. 

> [!NOTE]
> For a list of Azure regions in which Data Factory is currently available, select the regions that interest you on the following page: [Products available by region](https://azure.microsoft.com/global-infrastructure/services/?products=data-factory).

## Create a data factory

1. Launch the Windows PowerShell ISE.

1. Create variables.

    Copy and paste the following script and replace the variables (SubscriptionName, ResourceGroupName, etc.) with actual values. 

    ```powershell
    # If input contains a PSH special character, e.g. "$", precede it with the escape character "`" like "`$". 
    $SubscriptionName = "[Azure subscription name]" 
    $ResourceGroupName = "[Azure resource group name]" 
    $DataFactoryLocation = "EastUS" 

    # Shared Self-hosted integration runtime information. This is a Data Factory compute resource for running any activities 
    # Data factory name. Must be globally unique 
    $SharedDataFactoryName = "[Shared Data factory name]" 
    $SharedIntegrationRuntimeName = "[Shared Integration Runtime Name]" 
    $SharedIntegrationRuntimeDescription = "[Description for Shared Integration Runtime]"

    # Linked integration runtime information. This is a Data Factory compute resource for running any activities
    # Data factory name. Must be globally unique
    $LinkedDataFactoryName = "[Linked Data factory name]"
    $LinkedIntegrationRuntimeName = "[Linked Integration Runtime Name]"
    $LinkedIntegrationRuntimeDescription = "[Description for Linked Integration Runtime]"
    ```

1. Sign in and select a subscription.

    Add the following code to the script to sign in and select your Azure subscription:

    ```powershell
    Connect-AzureRmAccount
    Select-AzureRmSubscription -SubscriptionName $SubscriptionName
    ```

1. Create a resource group and a Data Factory.

    *(This step is optional. If you already have a data factory, skip this step.)* 

    Create an [Azure resource group](../azure-resource-manager/resource-group-overview.md) using the [New-AzureRmResourceGroup](/powershell/module/azurerm.resources/new-azurermresourcegroup) command. A resource group is a logical container into which Azure resources are deployed and managed as a group. The following example creates a resource group named `myResourceGroup` in the WestEurope location. 

    ```powershell
    New-AzureRmResourceGroup -Location $DataFactoryLocation -Name $ResourceGroupName
    ```

    Run the following command to create a data factory: 

    ```powershell
    Set-AzureRmDataFactoryV2 -ResourceGroupName $ResourceGroupName `
                             -Location $DataFactoryLocation `
                             -Name $SharedDataFactoryName
    ```

## Create a self-hosted integration runtime

*(This step is optional. If you already have the self-hosted integration runtime that you want to share with other data factories, skip this step.)*

Run the following command to create a self-hosted integration runtime:

```powershell
$SharedIR = Set-AzureRmDataFactoryV2IntegrationRuntime `
    -ResourceGroupName $ResourceGroupName `
    -DataFactoryName $SharedDataFactoryName `
    -Name $SharedIntegrationRuntimeName `
    -Type SelfHosted `
    -Description $SharedIntegrationRuntimeDescription
```

### Get the integration runtime authentication key and register a node

Run the following command to get the authentication key for the self-hosted integration runtime:

```powershell
Get-AzureRmDataFactoryV2IntegrationRuntimeKey `
    -ResourceGroupName $ResourceGroupName `
    -DataFactoryName $SharedDataFactoryName `
    -Name $SharedIntegrationRuntimeName
```

The response contains the authentication key for this self-hosted integration runtime. You use this key when you register the integration runtime node.

### Install and register the self-hosted integration runtime

1. Download the self-hosted integration runtime installer from [Azure Data Factory Integration Runtime](https://aka.ms/dmg).

2. Run the installer to install the self-hosted integration on a local computer.

3. Register the new self-hosted integration with the authentication key that you retrieved in a previous step.

## Share the self-hosted integration runtime with another data factory

### Create another data factory

*(This step is optional. If you already have the data factory with which you want to share, skip this step.)*

```powershell
$factory = Set-AzureRmDataFactoryV2 -ResourceGroupName $ResourceGroupName `
    -Location $DataFactoryLocation `
    -Name $LinkedDataFactoryName
```
### Grant permission

Grant permission to the Data Factory that needs to access the self-hosted integration runtime you created and registered.

> [!IMPORTANT]
> Do not skip this step!

```powershell
New-AzureRMRoleAssignment `
    -ObjectId $factory.Identity.PrincipalId ` #MSI of the Data Factory with which it needs to be shared
    -RoleDefinitionId 'b24988ac-6180-42a0-ab88-20f7382dd24c' ` #This is the Contributor role
    -Scope $SharedIR.Id
```

## Create a linked self-hosted integration runtime

Run the following command to create a linked self-hosted integration runtime:

```powershell
Set-AzureRmDataFactoryV2IntegrationRuntime `
    -ResourceGroupName $ResourceGroupName `
    -DataFactoryName $LinkedDataFactoryName `
    -Name $LinkedIntegrationRuntimeName `
    -Type SelfHosted `
    -SharedIntegrationRuntimeResourceId $SharedIR.Id `
    -Description $LinkedIntegrationRuntimeDescription
```

Now you can use this linked integration runtime in any linked service. The linked integration runtime is using the shared integration runtime to run activities.

## Revoke integration runtime sharing from a data factory

To revoke the access of a data factory from accessing the shared integration runtime, you can run the following command:

```powershell
Remove-AzureRMRoleAssignment `
    -ObjectId $factory.Identity.PrincipalId `
    -RoleDefinitionId 'b24988ac-6180-42a0-ab88-20f7382dd24c' `
    -Scope $SharedIR.Id
```

To remove the existing linked integration runtime, you can run the following command against the
shared integration runtime:

```powershell
Remove-AzureRmDataFactoryV2IntegrationRuntime `
    -ResourceGroupName $ResourceGroupName `
    -DataFactoryName $SharedDataFactoryName `
    -Name $SharedIntegrationRuntimeName `
    -Links `
    -LinkedDataFactoryName $LinkedDataFactoryName
```

## Next steps

- Review integration runtime concepts in [Integration runtime in Azure Data Factory](concepts-integration-runtime.md).

- Learn how to create a self-hosted integration runtime in the Azure portal in [Create and configure a self-hosted integration runtime](create-self-hosted-integration-runtime.md).
