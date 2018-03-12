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

When a server is deployed in Azure you often need to set it up in some way. A common way
to do this is by using the custom script extension for [Windows](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/extensions-customscript)
or [Linux](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/extensions-customscript).

However these do have a couple of disadvantages, the first being that you have to publish your
script somewhere, which you may not want to do depending on the content.
Second the VM needs internet access to actually download the script, so it won't work
if you are behind a proxy server.

Recently a new command appeared in AzureRM PowerShell called [Invoke-AzureRmVMRunCommand](https://docs.microsoft.com/en-us/powershell/module/azurerm.compute/invoke-azurermvmruncommand?view=azurermps-5.4.0)
that allows commands to be run against a VM without downloading a script from the internet.

Under the covers this is using a new VM extension published by Microsoft.CPlat.Core called
RunCommandWindows or RunCommandLinux

```
>az vm extension image list -l northeurope -p Microsoft.CPlat.Core -o tsv
RunCommandLinux         Microsoft.CPlat.Core    1.0.0
RunCommandWindows       Microsoft.CPlat.Core    1.0.0
RunCommandWindows       Microsoft.CPlat.Core    1.0.1
```

So what if you want to use this extension from ARM or Terraform without calling PowerShell?
The extension doesn't appear to be documented anywhere so a little reverse engineering is necessary.
If we decompile the RunCommandExtension.exe that is created on the VM and look at the PublicSettings
class we can see it has a parameter called `script` that expects a list of strings.

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
```

What this means is that in the `settings` part of the extension you pass the script as an
array, with each line as a separate element.

```json
{
  "script": [
    "$date = get-date",
    "write-output $date.ToShortDateString()"
  ]
}
```

```c#

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