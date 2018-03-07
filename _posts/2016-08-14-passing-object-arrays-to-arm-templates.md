---
layout: post
title: Passing object arrays to ARM templates
---

Recently I've been getting into ARM deployment templates to create infrastructure in Azure. There's loads of documentation out there on using these templates,
but basically you can create a central repository of templates for all the different bits of your infrastructure (VMs, load balancers, availability sets etc)
and then call these templates from a deployment template.
This allows you to have a standard set of templates for all the things you can deploy and then pick and choose the bits you need for a particular deployment,
passing in parameters where appropriate.

This all works very nicely, but I ran into an issue when trying to pass in parameters for the network security group template, specifically the security rules.
Here is an example of an NSG template:

```json
{
  "apiVersion": "2015-06-15",
  "type": "Microsoft.Network/networkSecurityGroups",
  "name": "[parameters('frontEndNSGName')]",
  "location": "[resourceGroup().location]",
  "tags": {
    "displayName": "NSG - Front End"
  },
  "properties": {
    "securityRules": [
      {
        "name": "rdp-rule",
        "properties": {
          "description": "Allow RDP",
          "protocol": "Tcp",
          "sourcePortRange": "*",
          "destinationPortRange": "3389",
          "sourceAddressPrefix": "Internet",
          "destinationAddressPrefix": "*",
          "access": "Allow",
          "priority": 100,
          "direction": "Inbound"
        }
      },
      {
        "name": "web-rule",
        "properties": {
          "description": "Allow WEB",
          "protocol": "Tcp",
          "sourcePortRange": "*",
          "destinationPortRange": "80",
          "sourceAddressPrefix": "Internet",
          "destinationAddressPrefix": "*",
          "access": "Allow",
          "priority": 101,
          "direction": "Inbound"
        }
      }
    ]
  }
}
```

You can see the securityRules property is expecting an array to be passed in as indicated by the square brackets, but each item in the array is itself an object with its own set of properties.
I created my deployment template to look like this:

```json
{
  "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "nsgName": {
      "type": "string"
    },
    "nsgRules": {
      "type": "array",
      "defaultValue": []
    }
  },
  "resources": [
    {
      "type": "Microsoft.Network/networkSecurityGroups",
      "name": "[parameters('nsgName')]",
      "apiVersion": "2015-06-15",
      "location": "[ResourceGroup().location]",
      "properties": {
        "securityRules": "[parameters('nsgRules')]"
      }
    }
  ]
}
```

Rather than filling up the template with all the rule definitions I created a separate json file with the rules:

```json
[
  {
    "name": "deny-internet",
    "properties": {
      "protocol": "*",
      "sourcePortRange": "*",
      "destinationPortRange": "*",
      "sourceAddressPrefix": "*",
      "destinationAddressPrefix": "internet",
      "access": "Deny",
      "priority": 4000,
      "direction": "Outbound"
    }
  },
]
```

And then passed it in to the template in PowerShell

```powershell
$nsgRules = Get-Content "$PSScriptRoot\nsgRules.json" -Raw | ConvertFrom-Json
$templateParameters = @{nsgName = "myNsg"; nsgRules = $nsgRules}

$params = @{
  Name = "hbdeploy";
  ResourceGroupName = "hbtest";
  TemplateFile = "$PSScriptRoot\azuredeploy.json";
  TemplateParameterObject = $templateParameters
}
New-AzureRmResourceGroupDeployment @params -Verbose
```

But this didn't work! I kept getting errors that the $nsgRules object couldn't be deserialized. This is because the ConvertFrom-Json cmdlet returns an Object[] of PSCustomObjects,
while the New-AzureRmResourceGroupDeployment requires a normal .NET type rather than a primitive or collection type.

The solution is to use the Newtonsoft.Json .NET library to deserialize the nsgRules.json file:

```powershell
Add-Type -Path "C:\Source\Newtonsoft.Json.6.0.1\lib\net45\Newtonsoft.Json.dll"
$jsonContent = Get-Content "$PSScriptRoot\nsgrules.json" -Raw
$nsgRules= [Newtonsoft.Json.JsonConvert]::DeserializeObject($jsonContent.ToString())
```

This returns a Newtonsoft.Json.JArray object, which can be passed to the template succesfully.

One more thing to note is that the API will only work with version 6.0.x of the Newtonsoft.Json library.
When I tried with newer versions the array was accepted by the deployment but the rules were all blank (see the last comment here https://github.com/Azure/azure-sdk-for-net/issues/1777).