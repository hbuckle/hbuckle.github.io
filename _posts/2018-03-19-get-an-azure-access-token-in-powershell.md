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
The main thing you need to be able to use the API is a bearer token. There are a few different
ways to get one, however if you have the AzureRM PowerShell modules installed then
you already have a bearer token.

To retrieve it, first run `Login-AzureRmAccount` or `Connect-AzureRmAccount`

Next run the following code to view the cached tokens.

```powershell
$context = Get-AzureRmContext
$tokens = $context.TokenCache.ReadItems()
$tokens | sort -Descending ExpiresOn | select DisplayableId,ExpiresOn | Format-Table
```

You can then select the most recent token and use it in your API calls.
PowerShellCore Invoke-RestMethod has an `Authentication` parameter for using bearer tokens,
for Windows PowerShell you will need to construct the header separately.

```powershell
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