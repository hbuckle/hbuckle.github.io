---
layout: post
comments: true
title: Use the Azure RunCommand VM extension
tags:
  - Azure
  - Terraform
categories:
  - Azure
---

```hcl
resource "azurerm_virtual_machine_extension" "test" {
  name                       = "cplattest"
  location                   = "${azurerm_resource_group.resource_group.location}"
  resource_group_name        = "${azurerm_resource_group.resource_group.name}"
  virtual_machine_name       = "${azurerm_virtual_machine.virtual_machine.name}"
  publisher                  = "Microsoft.CPlat.Core"
  type                       = "RunCommandWindows"
  type_handler_version       = "1.0"
  auto_upgrade_minor_version = true

  settings = <<SETTINGS
    {
        "script": [
          "powershell -command write-output hello"
        ]
    }
SETTINGS
}
```

```c#
[DataContract]
public class PublicSettings
{
  [DataMember(Name = "script")]
  public List<string> Script
  {
    get;
    set;
  }

  public override string ToString()
  {
    return "Script: [" + ((this.Script != null) ? string.Join("\n", this.Script) : string.Empty) + "]";
  }
}

[DataContract]
public class ProtectedSettings
{
  [DataMember(Name = "parameters")]
  public List<ParameterDefinition> Parameters
  {
    get;
    set;
  }

  public override string ToString()
  {
    return "Parameters: [" + ((this.Parameters != null) ? string.Join("\n", this.Parameters) : string.Empty) + "]";
  }
}

[DataContract]
public class ParameterDefinition
{
  [DataMember(Name = "name")]
  public string Name;

  [DataMember(Name = "value")]
  public string Value;

  public override string ToString()
  {
    return "Name: " + this.Name + ", Value: " + this.Value;
  }
}
```