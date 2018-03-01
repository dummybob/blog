---
title: "Saving Cloud Dollars: Consumption usage details in Azure"
description: "Using Octopus to notify if an Azure Resource group exceeds cost limits"
author: lawrence.wilson@octopus.com
visibility: private
tags:
 - Get-AzureRmConsumptionUsageDetail
---

Have you ever deployed an expensive Virtual Machine into Azure for a quick 10 minute test only to come back 2 months later when you realise it's been running the whole time? This blog post shows how you can use Octopus to notify you via Slack if a resource has reached a spending limit - which you can specify using resource group tags. Since Octopus has the ability to authenticate to many different cloud platforms, and deploy resources to them, it naturally has the ability to get useful data out too. This makes Octopus a great candidate to run the scripts we need to see how much our resources are costing so that we can spot ones which are costing above the expected ammount .

### Scenario
I love a good  scenario, so without further adue, please meet OctoFX; A fictional firm of highly skilled developers who need to run tests against resources in many different cloud platforms. They have chosen to use Octopus to run these scripts because it allows them to follow a consistent approach irrespective of which cloud platform they're using. In this scenario, we will be defining a tag called NotifyCostLimit which, when this limit is hit, Octopus will send out slack notifications. If there is no NotifyCostLimit applied, the default value of $100 will be assumed

This blog entry only covers the steps taken to query Microsoft Azure, but you can apply a similar approach with AWS as well. So, let's get it done! If you would like to see the process that the script takes, you can follow along with each of the following objectives, alternatively you can skip through to the next section to see the completed script.  

### Limitations
A few key thing to note here:
- The cmdlet ([Get-AzureRmConsumptionUsageDetail](https://docs.microsoft.com/en-us/powershell/module/azurerm.consumption/get-azurermconsumptionusagedetail?view=azurermps-5.4.0)) used to retrieve the cost items can only get Resource Manager details, this means that we won't be seeing any Service Manager Resources costs at all.
- It might take up to 2 weeks for the consumption details to be available and we should consider that the earliest data entry we see might be from two weeks ago, 
- Cost figures do not include tax.

### Objective 1: Get all cost items for all subscriptions in Azure
The main focus of the script segment below is to retrieve consumption usage details. This can be acomplished by using the [Get-AzureRmConsumptionUsageDetail](https://docs.microsoft.com/en-us/powershell/module/azurerm.consumption/get-azurermconsumptionusagedetail?view=azurermps-5.4.0) cmdlet from Azure Powershell. Here we provide two dates, StartDate (today minus 30 days) and the EndDate (today)

```PowerShell
#These will be Octopus Variables in the completed script:
$SubscriptionId = "00000000-0000-0000-0000-000000000000"
$DateRangeInDays = 30
$DefaultNotifyCostLimit = 100

$now = get-Date
$SubConsumptionUsage = Get-AzureRmConsumptionUsageDetail -StartDate $($now.Date.AddDays(-$DateRangeInDays)) -EndDate $($now.Date)
```

### Objective 2: Find all of the resource group names which existed during this billing period.
Now, we take each line from the consumption usage details to trim off the beginning part of each Instance ID. Then, we strip out everything after the resource group name. In the end, we are finished with an object of resource group names. Since resource groups are case insensitive, I have made each resource group name lower-case.

```PowerShell
$SubIdPrefix = "/subscriptions/" + $SubscriptionId 
$RgIdPrefix = $SubIdPrefix + "/resourceGroups/"

$resourceGroupName = @()
$resourceGroups = @()

foreach ($line in $SubConsumptionUsage) {
    if ($line.InstanceId -ne $null ) {
        $thisRgName = $($line.InstanceId.ToLower()).Replace($RgIdPrefix.ToLower(),"")
        $toAdd = $thisRgName.Split("/")[0]
        $toAdd = $toAdd.ToString()
        $toAdd = $toAdd.ToLower()
        $toAdd = $toAdd.Trim()

        if ($resourceGroups.Name -notcontains $toAdd) {
            $resourceGroupName = [PSCustomObject]@{
            Name = $toAdd
            }
            $resourceGroups += $resourceGroupName
        }
    }
}
```

### Objective 3: Calculate the cost of each Resource Group, then flag if there is a problem.
We only care about the current resource groups now, because we aren't going to perform any actions against an already deleted resource group. For each resource group name encoutered we filter the details by resource group and add all the costs together. This becomes the total cost of each resource group. 

```PowerShell
$rgIndexId = 0
$currentResourceGroups = Get-AzureRmResourceGroup

foreach ($rg in $RgTotal) {
    
    $thisRg = $null

    $RgIdPrefix = $SubIdPrefix + "/resourceGroups/" + $rg.Name
    $ThisRgCost = $null
    $playgroundItem | ? { if ( $_.InstanceId -ne $null) { $($_.InstanceId.ToLower()).StartsWith($RgIdPrefix.ToLower()) } } |  ForEach-Object { $ThisRgCost += $_.PretaxCost   }

    $toaddCost = [math]::Round($ThisRgCost,2)

    $RgTotal[$rgIndexId] | Add-Member -MemberType NoteProperty -Name "Cost" -Value $toaddCost

    if ($currentResourceGroups.ResourceGroupName -contains $rg.Name) {
        
        $thisMyRgNameStuff = Get-AzureRmResourceGroup -Name $($rg.Name)

        $RgTotal[$rgIndexId] | Add-Member -MemberType NoteProperty -Name "OwnerContact" -Value $($thisMyRgNameStuff.tags.OwnerContact)
        

    }

    $rgIndexId ++

}

```

### Objective 4: Filter the items whose cost is higher than expected
As state, we will use the `where-object` (which is shortened to a `?` to filter each gost which is greater-than or equal to(`ge`) the cost notification tag `NotifyCostLimit`
```PowerShell
$expensiveGroups = $resourceGroups | ? {
    if ($_.NotifyCostLimit -ne $Null) {

        $_.Cost -ge $_.NotifyCostLimit

    }
    else {

        $_.Cost -ge $DefaultNotifyCostLimit

    }
}


```
### Objective 5: Notify the correct people using a Slack notification.
Finally, we will be reminding people that they still have a test resource which has reached the cost limit for that resource.

```PowerShell
function new-SlackMessage ( $resourceGroup ) {

    $orange = "#F9812A"
    $attachments = @{}
    $fieldNumber = 0
    $username = "Octopus Cost Reminder"
    $IconUrl = "https://octopus.com/images/company/Logo-Blue_140px_rgb.png"
    $payload = @{
    #channel = "General";
    username = $username;
    icon_url = $IconUrl;
    attachments = @(
    );
    }


    $state = "Current cost: $($resourceGroup.Cost)"
    $description = "ResourceGroup Name: $($resourceGroup.Name)"
    $ownerContact = "OwnerContact: $($resourceGroup.OwnerContact)"
    $colour = $orange

    if ($fieldNumber -eq 0){
        $pretext = "Reminder: I found a Resource Groups in Azure which has spiked above expected cost"
    }
    else {
        $pretext = $null
    }

    $thisFallbackMessage = "$description has spiked above expected cost (Current Cost: $state)"

    #Some results can have multiple fields:
    $fields = @(
        @{
        title = $description;
        value = $state;
        });
    

    $thisAttachment = @{
        fallback = $thisFallbackMessage;
        color = $colour;
        pretext = $pretext;
        fields = $fields;
    }

    $payload.attachments += $thisAttachment 

    return $payload

}

function New-SlackNotification ($hook, $group)
{

    $message = New-SlackMessage -resourceGroup $group
    Invoke-Restmethod -Method POST -Body ($message | ConvertTo-Json -Depth 4) -Uri $hook

}

foreach ($group in $expensiveGroups) {

    New-SlackNotification -hook $soackHook -group $group


}
```

## Configure it in Octopus
Create your new Project in Octopus, mine is called `Cloud Cost` then, define these 4 project variables:
- SubscriptionId (Type: String) The Subscription Id which can be retrieved from Azure PowerShell by running `Get-AzureRmSubscription`
- DateRangeInDays (Type: Integer) The script takes the current day and counts backwards by this many days to form a range to check usage cost
- DefaultNotifyCostLimit (Type: Integer) If a resource group isn't tagged with a `NotifyCostLimit` Octopus will default to this value (Put an int here
- SlackHook (Type: String) the full URL of your slack hook, eg: (https://hooks.slack.com/services/XXXXXXXXX/XXXXXXXXX/XXXXXXXXXXXXXXX)

### Create your first Step (Azure PowerShell Script)
Below is the full script you will need to paste (WIP):

```PowerShell#Objective 1: Get all cost items for all subscriptions in Azure
$SubscriptionId  = "95bf77d2-64b1-4ed2-9de1-b5451e3881f5"
$DateRangeInDays = 30
$DefaultNotifyCostLimit = 100

$now = get-Date
$SubConsumptionUsage = Get-AzureRmConsumptionUsageDetail -StartDate $($now.Date.AddDays(-$DateRangeInDays)) -EndDate $($now.Date)


#Objective 2: Find all of the resource group names which existed during this billing period.
$SubIdPrefix = "/subscriptions/" + $SubscriptionId 
$RgIdPrefix = $SubIdPrefix + "/resourceGroups/"

$resourceGroupName = @()
$resourceGroups = @()

foreach ($line in $SubConsumptionUsage) {
    if ($line.InstanceId -ne $null ) {
        $thisRgName = $($line.InstanceId.ToLower()).Replace($RgIdPrefix.ToLower(),"")
        $toAdd = $thisRgName.Split("/")[0]
        $toAdd = $toAdd.ToString()
        $toAdd = $toAdd.ToLower()
        $toAdd = $toAdd.Trim()

        if ($resourceGroups.Name -notcontains $toAdd) {
            $resourceGroupName = [PSCustomObject]@{
            Name = $toAdd
            }
            $resourceGroups += $resourceGroupName
        }
    }
}

# Objective 3: Calculate the cost of each Resource Group, then flag if there is a problem.
$currentResourceGroups = Get-AzureRmResourceGroup
$rgIndexId = 0
foreach ($rg in $resourceGroups) {
    
    #$thisRg = $null
    $RgIdPrefix = $SubIdPrefix + "/resourceGroups/" + $rg.Name
    $ThisRgCost = $null
    $playgroundItem | ? { if ( $_.InstanceId -ne $null) { $($_.InstanceId.ToLower()).StartsWith($RgIdPrefix.ToLower()) } } |  ForEach-Object { $ThisRgCost += $_.PretaxCost   }
    $toaddCost = [math]::Round($ThisRgCost,2)
    $resourceGroups[$rgIndexId] | Add-Member -MemberType NoteProperty -Name "Cost" -Value $toaddCost

    
    
    if ($currentResourceGroups.ResourceGroupName -contains $rg.Name) {
        
        $thisMyRgNameStuff = Get-AzureRmResourceGroup -Name $($rg.Name)
        $resourceGroups[$rgIndexId] | Add-Member -MemberType NoteProperty -Name "OwnerContact" -Value $($thisMyRgNameStuff.tags.OwnerContact)

    }

    $rgIndexId ++

}

# Objective 4: Filter the items whose cost is higher than the allowed limit.
$expensiveGroups = $resourceGroups | ? {
    if ($_.NotifyCostLimit -ne $Null) {

        $_.Cost -ge $_.NotifyCostLimit

    }
    else {

        $_.Cost -ge $DefaultNotifyCostLimit

    }
}


```

### Setting up the Octopus Service Principal Account
We first need to ensure that Octopus has a Service Principal Account configured to be able to see into your Azure Subscription. Please feel free to check out our documentation on [creating an Azure service principal account](https://g.octopushq.com/AzureServicePrincipalAccount) for more information on how to perform that task. For the purposes of this account you need to ensure that it has at least `Reader` access to the subscription.


### Create your first Step (Azure PowerShell Script)
The properties of this step are:
- Name: Resource gropus exceeding limit
```PowerShell
#Objective 1: Get all cost items for all subscriptions in Azure
$now = get-Date
$SubConsumptionUsage = Get-AzureRmConsumptionUsageDetail -StartDate $($now.Date.AddDays(-$DateRangeInDays)) -EndDate $($now.Date)

#Objective 2: Find all of the resource group names which existed during this billing period.
$SubIdPrefix = "/subscriptions/" + $SubscriptionId 
$RgIdPrefix = $SubIdPrefix + "/resourceGroups/"

$resourceGroupName = @()
$resourceGroups = @()

foreach ($line in $SubConsumptionUsage) {
    if ($line.InstanceId -ne $null ) {
        $thisRgName = $($line.InstanceId.ToLower()).Replace($RgIdPrefix.ToLower(),"")
        $toAdd = $thisRgName.Split("/")[0]
        $toAdd = $toAdd.ToString()
        $toAdd = $toAdd.ToLower()
        $toAdd = $toAdd.Trim()

        if ($resourceGroups.Name -notcontains $toAdd) {
            $resourceGroupName = [PSCustomObject]@{
            Name = $toAdd
            }
            $resourceGroups += $resourceGroupName
        }
    }
}

# Objective 3: Calculate the cost of each Resource Group, then flag if there is a problem.
$currentResourceGroups = Get-AzureRmResourceGroup
$rgIndexId = 0
foreach ($rg in $resourceGroups) {
    
    #$thisRg = $null
    $RgIdPrefix = $SubIdPrefix + "/resourceGroups/" + $rg.Name
    $ThisRgCost = $null
    $playgroundItem | ? { if ( $_.InstanceId -ne $null) { $($_.InstanceId.ToLower()).StartsWith($RgIdPrefix.ToLower()) } } |  ForEach-Object { $ThisRgCost += $_.PretaxCost   }
    $toaddCost = [math]::Round($ThisRgCost,2)
    $resourceGroups[$rgIndexId] | Add-Member -MemberType NoteProperty -Name "Cost" -Value $toaddCost
	
    if ($currentResourceGroups.ResourceGroupName -contains $rg.Name) {
        
        $thisMyRgNameStuff = Get-AzureRmResourceGroup -Name $($rg.Name)
        $resourceGroups[$rgIndexId] | Add-Member -MemberType NoteProperty -Name "OwnerContact" -Value $($thisMyRgNameStuff.tags.OwnerContact)

    }

    $rgIndexId ++

}

# Objective 4: Filter the items whose cost is higher than the allowed limit.
if ($NotifyCostLimit -eq $null) {

    $NotifyCostLimit = $DefaultNotifyCostLimit

}

$expensiveGroups = $resourceGroups | ? {$_.Cost -ge $NotifyCostLimit}
Set-OctopusVariable -name "expensiveGroups" -value $resourceGroups


```

### Create your second Step (Azure PowerShell Script)
Below is the full script you will need to paste (WIP):
```PowerShell
#slack-notify 

#Here is the list to notify:
$expensiveGroups = $OctopusParameters["Octopus.Action[Resource gropus exceeding limit].Output.expensiveGroups"]


```