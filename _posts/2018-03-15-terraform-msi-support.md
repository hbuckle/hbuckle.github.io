---
layout: post
comments: true
title: Terraform MSI Support - provision a build agent to auto login to Azure
tags:
  - MSI
  - Azure
  - Authentication
  - Buildkite
categories:
  - Terraform
---

The good people at Hashicorp have recently merged a couple of
my pull requests into the Terraform AzureRM provider to enable
support for [Managed Service Identity](https://docs.microsoft.com/en-us/azure/active-directory/managed-service-identity/overview)

To recap, MSI allows an Azure virtual machine to retrieve an access
token for the Azure API so it can manage resources.

So what cool things can we do with this? Here's an example of creating
a VM to use as a Terraform CI build server.

I'm going to use the [buildkite](https://buildkite.com) agent, which is
a neat little CI tool that can be deployed in a container. To get
the agent up and running I'll deploy a CoreOS VM and configure it
to run the buildkite container as a service.

```terraform
data "ignition_systemd_unit" "systemd_unit" {
  name    = "bkagent.service"
  enabled = true
  content = "${file("${path.module}/bkagent.service")}"
}

data "ignition_config" "config" {
  systemd = [
    "${data.ignition_systemd_unit.systemd_unit.id}",
  ]
}

resource "azurerm_virtual_machine" "virtual_machine" {
  name                  = "coreos"
  location              = "${azurerm_resource_group.resource_group.location}"
  resource_group_name   = "${azurerm_resource_group.resource_group.name}"
  network_interface_ids = ["${azurerm_network_interface.network_interface.id}"]
  vm_size               = "Standard_DS1_v2"

  identity = {
    type = "SystemAssigned"
  }

  storage_image_reference {
    publisher = "CoreOS"
    offer     = "CoreOS"
    sku       = "Stable"
    version   = "latest"
  }

  storage_os_disk {
    name              = "coreos"
    caching           = "ReadWrite"
    create_option     = "FromImage"
    managed_disk_type = "Premium_LRS"
  }

  os_profile {
    computer_name  = "coreos"
    admin_username = "azureuser"
    admin_password = "${var.password}"

    custom_data = "${data.ignition_config.config.rendered}"
  }

  os_profile_linux_config {
    disable_password_authentication = false
  }
}

resource "azurerm_virtual_machine_extension" "virtual_machine_extension" {
  name                 = "coreos"
  location             = "${azurerm_resource_group.resource_group.location}"
  resource_group_name  = "${azurerm_resource_group.resource_group.name}"
  virtual_machine_name = "${azurerm_virtual_machine.virtual_machine.name}"
  publisher            = "Microsoft.ManagedIdentity"
  type                 = "ManagedIdentityExtensionForLinux"
  type_handler_version = "1.0"

  settings = <<SETTINGS
    {
        "port": 50342
    }
SETTINGS
}

data "azurerm_subscription" "subscription" {}

data "azurerm_builtin_role_definition" "builtin_role_definition" {
  name = "Contributor"
}

resource "azurerm_role_assignment" "role_assignment" {
  scope              = "${data.azurerm_subscription.subscription.id}"
  role_definition_id = "${data.azurerm_subscription.subscription.id}${data.azurerm_builtin_role_definition.builtin_role_definition.id}"
  principal_id       = "${lookup(azurerm_virtual_machine.virtual_machine.identity[0], "principal_id")}"
}
```

To enable MSI we set the `identity` property on the VM to `SystemAssigned`
and then apply the `ManagedIdentityExtensionForLinux` VM extension.

The `principal_id` is then retrieved from the VM and used to create a new
`role_assignment`. For this example we just give the VM contributor rights
to the subscription it is in.

To get the buildkite agent running the CoreOS Ignition resource
is used to set up a systemd unit to run the agent in a container.
The bkagent.service file looks like this:

```
[Unit]
Description=Buildkite Agent Container
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=0
Restart=always
ExecStartPre=-/usr/bin/docker kill bkagent
ExecStartPre=-/usr/bin/docker rm bkagent
ExecStart=/usr/bin/docker run --name bkagent --network=host -e BUILDKITE_AGENT_TOKEN=<REPLACE WITH TOKEN> buildkite/agent:3

[Install]
WantedBy=multi-user.target
```

This will give us a buildkite agent connected to our account. To run Terraform
on the agent set some environment variables in the `pipeline.yml` file:

```yaml
env:
  ARM_USE_MSI: true
  ARM_MSI_ENDPOINT: http://localhost:50342/oauth2/token
  ARM_TENANT_ID: <my tenant>
  ARM_SUBSCRIPTION_ID: <my subscription>
```

You can now run Terraform on this agent and it will automatically log in using
the VM's managed service identity. You could hand the agent over to a dev team to
use for deployments without needing to configure or provide credentials for them - 
everything is handled entirely through Terraform and abstracted away.
The same approach will work for VSTS, or Jenkins, or any other agent based CI tool.

You could also run the Azure CLI on the agent using `az login --msi` or write
your own code to call the MSI endpoint and get a token for the API - there's all
sorts of interesting possibilities.