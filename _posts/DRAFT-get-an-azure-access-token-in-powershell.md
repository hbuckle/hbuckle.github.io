---
layout: post
comments: true
title: Get an Azure access token in PowerShell
tags:
  - PowerShell
  - Azure
  - Authentication
categories:
  - PowerShell
---

You can do most things in Azure using the AzureRM PowerShell commands or the Azure CLI.
Occasionally though there is a new feature that is only available through the REST API.

```powershell
Login-AzureRmAccount
$context = Get-AzureRmContext
$tokens = $context.TokenCache.ReadItems()
$tokens | sort -Descending ExpiresOn | select DisplayableId,ExpiresOn | Format-Table
$token = ($tokens | sort -Descending ExpiresOn | select -First 1).AccessToken

#powershellcore
$securetoken = $token | ConvertTo-SecureString -AsPlainText -Force
Invoke-RestMethod -Method Get -Uri $resourceUri -Token $tokensecure -Authentication Bearer -ContentType "application\json"

#windowspowershell
$authHeader = @{
  "Authorization" = "Bearer $token"
}
Invoke-RestMethod -Method Get -Uri $resourceUri -Headers $authHeader -ContentType "application\json"
```

```bash
az login
az account get-access-token
```