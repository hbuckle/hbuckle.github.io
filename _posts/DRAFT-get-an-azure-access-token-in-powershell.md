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

```powershell
Login-AzureRmAccount
$context = Get-AzureRmContext
$tokens = $context.TokenCache.ReadItems()
$tokens | sort -Descending ExpiresOn | select DisplayableId,ExpiresOn | Format-Table
($tokens | sort -Descending ExpiresOn | select -First 1).AccessToken
```