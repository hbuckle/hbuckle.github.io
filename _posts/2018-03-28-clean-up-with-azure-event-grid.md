---
layout: post
comments: true
title: Perform clean up actions with Azure Event Grid
tags:
  - PowerShell
  - Azure Automation
  - Azure Event Grid
categories:
  - Azure
---

If you're doing any serious work in the cloud chances are you are creating
and destroying a lot of virtual machines. This can present a few challenges
around ensuring you clean up after yourself - a VM can leave behind various
other things that need to be removed such as DNS records, Active Directory
objects, entries in configuration management systems etc.

There are various ways to trigger cleanup actions when something is deleted
in Azure but today we're going to look at using the newly released
[Event Grid](https://docs.microsoft.com/en-us/azure/event-grid/) to monitor
a subscription and trigger an Azure Automation runbook whenever a VM is
deleted.

It's worth noting that you don't actually create an Event Grid, it simply
exists at the Azure platform level. Instead you create `Event Subscriptions`
that hook into the Event Grid. These subscriptions can be scoped at different
levels, for example you can create a subscription that only monitors a
certain resource group, or only looks for events from a certain resource
type such as storage accounts. For this example we will create one that
monitor deletion events from an Azure subscription.

Before an event subscription can be created it needs an endpoint to send events to.
Our endpoint will be the webhook for an Azure automation runbook which can
be created using the documentation [here](https://docs.microsoft.com/en-us/azure/automation/automation-webhooks).

Once you have the webhook URL create the event subscription using the azure cli

```
az eventgrid event-subscription create --name mysubscription --endpoint <webhook url> --included-event-types Microsoft.Resources.ResourceDeleteSuccess
```

This will create an event subscription that triggers the runbook whenever a
resource is deleted from the Azure subscription. You can then perform further
filtering inside the runbook, for example if you only want to take action
when a virtual machine is deleted you can use the following code in the runbook

```powershell
param(
  [parameter(Mandatory = $false)]
  [object]$WebhookData
)
$ErrorActionPreference = "Stop"

$RequestBody = $WebhookData.RequestBody | ConvertFrom-Json
$Data = $RequestBody.data
if ($Data.operationName -match "Microsoft.Compute/virtualMachines/delete" -and $Data.status -match "Succeeded") {
  $vmname = $Data.resourceUri.split('/') | Select-Object -Last 1
  # perform cleanup actions here
}
```

Event Grid is a great way to get notifications and take action on
Azure Events, and it's built into the Azure fabric, so there's very
little setup required. You can also create your own custom events
and event subscribers.