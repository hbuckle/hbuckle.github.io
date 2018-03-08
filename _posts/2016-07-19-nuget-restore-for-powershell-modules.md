---
layout: post
comments: true
title: NuGet restore for PowerShell modules
tags:
  - PowerShell
  - NuGet
categories:
  - PowerShell
---

The idea of continuous deployment for PowerShell modules has been picking up steam recently -
Powershell.org are offering a build server and there's a great overview here on setting up a pipeline using AppVeyor.

At my work we use BitBucket and Bamboo, which integrate nicely, and I have a set of Bamboo plans to deploy
my modules to our internal repository whenever I check code into BitBucket.

One problem I had was with modules that rely on external DLLs - these DLLs need to be deployed along with the module,
but I don't want to be keeping them in my source control repository. What I needed was a way to automatically
download the required assemblies to the Bamboo server during the build process. Fortunately this capability
already exists in the form of NuGet, which is used by Visual Studio
(and is also the same technology that underpins PowerShellGet).

If you create a project in Visual Studio and add some packages to it from NuGet then you will get a
packages.config file added to the project which can be used to restore those same packages on another
machine, and we can use the same method for PowerShell. Below is an example packages.config file,
as you can see it is just xml listing the package ID, version and target framework.

```xml
<?xml version="1.0" encoding="utf-8"?>
<packages>
  <package id="Newtonsoft.Json" version="8.0.3" targetFramework="net45" />
  <package id="Newtonsoft.Json.Schema" version="2.0.2" targetFramework="net45" />
</packages>
```

You can then use the nuget.exe tool with the restore parameter to download the packages.

However, seeing as this was designed for Visual Studio we need to do a bit more work to get it to work with our module.
The main issue is that nuget.exe restore will pull down the whole NuGet package, which includes versions of the DLLs
for all platforms and a bunch of other files that we don't need. I ended up writing a short function that looks at
the packages.config file and copies only the required DLLs to the module directory.

```powershell
function Restore-NugetPackages {
  [CmdletBinding()]
  Param(
    [ValidateNotNullOrEmpty()]
    [string]$ModuleDirectory
  )
  Set-StrictMode -Version Latest
  $ErrorActionPreference = "Stop"
  $config = Join-Path $ModuleDirectory "packages.config"
  if (-not(Test-Path $config)) {
    Return
  }
  $tempDir = Join-Path $env:TEMP "pspackagerestore"
  Remove-Item -Path $tempDir -Confirm:$false -Force -ErrorAction SilentlyContinue -Recurse
  $null = New-Item -Path $tempDir -ItemType Directory
  if (-not(Test-Path "$env:Temp\nuget.exe")) {
    Invoke-WebRequest -Uri "https://dist.nuget.org/win-x86-commandline/latest/nuget.exe" -OutFile "$env:Temp\nuget.exe"
  }
  & "$env:Temp\nuget.exe" restore $config -PackagesDirectory $tempDir -Source "https://api.nuget.org/v3/index.json"
  $xml = [xml](Get-Content $config -raw)
  foreach ($package in $xml.packages.package) {
    $assemblyFolder = Join-Path $tempDir "$($package.id).$($package.version)\lib\$($package.targetFramework)"
    Get-ChildItem $assemblyFolder | Copy-Item -Destination $ModuleDirectory
  }
  Remove-Item -Path $tempDir -Confirm:$false -Force -ErrorAction SilentlyContinue -Recurse
}
```