---
layout: post
comments: true
title: Creating Azure Automation webhooks with Terraform
tags:
  - Terraform
  - Azure Automation
---

Azure automation runbooks are great for running scripts in the cloud, either in
response to events or on a schedule.

Terraform has support for creating automation accounts and runbooks, however
you cannot add a webhook to a runbook, which is required if you want to trigger
it from another system.

I suspect the reason for this is that the Azure API does not allow the webhook
URI to be retrieved once it has been created and Terraform relies on being able
to discover the existing state of a resource so it knows if anything needs to be
changed.

Fortunately a webhook can be created using an ARM template, which can be
deployed from Terraform using the `azurerm_template_deployment` resource.

```hcl
resource "azurerm_template_deployment" "template_deployment" {
  name                = "test_webhook"
  resource_group_name = "${azurerm_resource_group.test_resource_group.name}"
  deployment_mode     = "Incremental"
  template_body = <<DEPLOY
{
  "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "name": "${azurerm_automation_account.test_account.name}/testwebhook",
      "type": "Microsoft.Automation/automationAccounts/webhooks",
      "apiVersion": "2015-10-31",
      "properties": {
        "isEnabled": true,
        "uri": "${local.webhook}",
        "expiryTime": "2028-01-01T00:00:00.000+00:00",
        "parameters": {},
        "runbook": {
          "name": "${azurerm_automation_runbook.test_runbook.name}"
        }
      }
    }
  ]
}
DEPLOY
}
```

There is still one more thing required which is the webhook URI (`${local.webhook}` in the example).
To create the webhook you need to pass in a valid URI which needs to adhere to a certain format.
The API exposes a method for generating webhook URIs, but they all follow a standard format
so we can just use Terraform's random_string resource to generate one for us.

```hcl
resource "random_string" "token1" {
  length  = 10
  upper   = true
  lower   = true
  number  = true
  special = false
}

resource "random_string" "token2" {
  length  = 31
  upper   = true
  lower   = true
  number  = true
  special = false
}

locals {
  webhook = "https://s9events.azure-automation.net/webhooks?token=%2b${random_string.token1.result}%2b${random_string.token2.result}%3d"
}
```

Now that the webhook is defined in Terraform it can be passed into other
resources as well, such as adding it to a Bitbucket or Github repo.

Remember that anyone who knows this webhook will be able to trigger your
runbook, so ensure that your Terraform state is somewhere secure.