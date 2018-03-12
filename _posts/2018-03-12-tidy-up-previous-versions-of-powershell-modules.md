---
layout: post
comments: true
title: Tidy up previous versions of PowerShell modules
tags:
  - PowerShellGet
  - Quick script
categories:
  - PowerShell
---

If you use PowerShellGet to install and update modules you may have noticed it doesn't
remove previous versions automatically for you.

If you're installing a lot of big modules (like the AzureRM ones) then the disk space
usage can add up, so here's a quick script to remove all but the most recent version
of all installed modules.

Note the use of Get-Package rather than Get-InstalledModule - for whatever reason
Get-Package is waaaaay faster.

```powershell
Get-Package -PackageManagementProvider PowerShellGet -AllVersions | Group-Object -Property Name | % {
  if ($_.Count -gt 1) {
    $_.Group | Sort-Object -Property Version -Descending | Select-Object -Skip 1 | % {
      Write-Output "Uninstalling $($_.Name) $($_.Version)"
      Remove-Module $_.Name -Force -ErrorAction SilentlyContinue
      Uninstall-Module $_.Name -RequiredVersion $_.Version -Force
    }
  }
}
```