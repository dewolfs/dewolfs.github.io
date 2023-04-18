---
layout: post
title: 'From zero to hero with Microsoft Graph PowerShell SDK'
date: 2023-04-18 02:30:20 +0300
description: 'graphapi' # Add post description (optional)
img: 2023-04-18-graphapi.jpg # Add image post (optional)
fig-caption: Photo By Caleb White on Unsplash # Add figcaption (optional)
tags: [ azure, azuread, graphapi, api, security, powershell ]
---

# 1. Introduction

When adopting the **[subscription vending](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/subscription-vending)** mechanism, one of the activities is to setup permissions for product teams and the functionality to allow provisioning of Infrastructure-As-Code using a service principal.  To make this happen, we want to build the required automation to do this in a repeatable and scalable manner.

By using the **[Microsoft Graph PowerShell SDK](https://learn.microsoft.com/en-us/powershell/microsoftgraph/overview?view=graph-powershell-1.0)** we can take advantage of a secured, unified API to connect to Azure Active Directory.  

We are going to create Azure AD applications, obtaining an access token and calling APIs with application permissions.

The following images shows what we will implement in this blogpost:

![design]({{site.baseurl}}/assets/img/2023-04-18-design.png)

1. We have three (*[createSPN.ps1](https://github.com/dewolfs/graph-api-powershell-sdk/blob/main/scripts/createSPN.ps1)*, *[createAADGroup.ps1](https://github.com/dewolfs/graph-api-powershell-sdk/blob/main/scripts/createAADGroup.ps1)*, *[addOwnerToAADGroup.ps1](https://github.com/dewolfs/graph-api-powershell-sdk/blob/main/scripts/addOwnertoAADGroup.ps1)*) separate PowerShell scripts that are triggered via an Azure DevOps pipeline.
2. Authenticating to the Microsoft Graph is taking place via separate (*spn-prd-graph-applications, spn-prd-graph-groups*) Application registrations.
3. Each Application Registration has its own secret stored in an Azure KeyVault.
4. IAM has been configured on the secrets in such a way that:
   - spn-prd-graph-applications can read only it's own secret
   - spn-prd-graph-groups can read only it's own secret
5. The Microsoft Graph PowerShell SDK is a wrapper to easily execute operational tasks in Azure Active Directory
6. The Azure DevOps pipeline authenticates (with two dedicated service connections) to the Azure KeyVault via two Application Registrations to retrieve the secret that is used as part of the PowerShell script.

# 2. Application Registration

First, we create the two dedicated Application Registrations (*spn-prd-graph-applications, spn-prd-graph-groups*) required for our scenario.  This is a straightforward task in the **[Microsoft Entra Portal](https://entra.microsoft.com/)**: Applications > New registration > Fill in a name and click on Register.

## 2.1 API Permissions  

Next, we need to assign the necessary API permissions to the Application Registration.  Open the properties of the Application Registration, click on API permissions > Add permission and select Microsoft Graph.  

![api-permissions2]({{site.baseurl}}/assets/img/2023-04-18-apipermissions2.png)

The Microsoft Graph allows us to define permissions in two different ways:  

- **Delegated permissions** are used by apps that *have a signed-in user present*.  For these apps, either the user or an administrator consents to the permissions that the app requests and the app can act as the signed-in user when making calls to Microsoft Graph.  Some delegated permissions can be consented by non-administrative users, but some higher-privileged permissions require administrator consent. 
- **Application permissions** are used by apps that run *without a signed-in* user present.  For example, apps that run as background services or daemons.  Application permissions can only be consented by an administrator. 

For our scenario, we need to pick **Application permissions**.  Look in the list for the API permission you need.

![api-permissions3]({{site.baseurl}}/assets/img/2023-04-18-apipermissions3.png)

The following table shows what API permissions are set for the two Application Registrations we use to authenticate to the Microsoft Graph API.

| **Application** App registration (spn-prd-graph-applications) | **Groups** App registration (spn-prd-graph-groups) |
| Purpose: create new App registrations | Purpose: create new AAD Groups, assign Owner |
| -------- | ------- |
| ![api-permissions]({{site.baseurl}}/assets/img/2023-04-18-spn-app-premissions.png) | ![api-permissions]({{site.baseurl}}/assets/img/2023-04-18-spn-grp-premissions.png) |
| Application.ReadWrite.OwnedBy | Group.ReadWrite.All, Directory.ReadWrite.All |

## 2.2 Secret

Multiple options exist for authenticating to the Microsoft Graph API.  We will be using a client secret.  Open the properties of the Application Registration > Certificates & Secrets and click on **New client secret**.  Select how long the secret should be valid and save it in a temporarily place because we will need it later.  

![spncreate]({{site.baseurl}}/assets/img/2023-04-18-new-secrets.png)

# 3. Automation and DevOps

## 3.1 Retrieve secrets  

The PowerShell scripts create and/or modify resources in Azure Active Directory.  Before this activity can take place, we need to authenticate to the Graph API.  We need to store the client secret of the Application Registration we created before inside our Azure KeyVault.

![kv-secrets]({{site.baseurl}}/assets/img/2023-04-18-secrets.png)

Our KeyVault has the Azure role-based access control permission model defined so that we can set granular permissions.

![kv-iam]({{site.baseurl}}/assets/img/2023-04-18-secret-iam.png)

The Azure built-in role '**[Key Vault Secrets User](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide?tabs=azure-cli#azure-built-in-roles-for-key-vault-data-plane-operations)**' was assigned to the Application Registration on the individual secret.  The following table indicates which role is assigned to which Application registration at what scope.

| Role | Assign access to | Scope |
| -------- | ------- | ------- |
| Key Vault Secrets User| spn-prd-graph-applications | secret-spn-prd-graph-applications |
| Key Vault Secrets User| spn-prd-graph-groups| secret-spn-prd-graph-groups |

## 3.2 Azure DevOps Service Connection

In Azure DevOps, we define a service connection for each Application Registration.  

![serv-conn]({{site.baseurl}}/assets/img/2023-04-18-serv-conn.png)

This is required to retrieve the client secret that is stored in the Azure KeyVault.  We have named them accordingly so that it's easy to distinguish.  Each service connection is using it's own Application Registration object ID and client secret.

We are using the Application Registration not only to authenticate to Microsoft Graph but also to Microsoft Azure, keeping least privileged principle in mind.

## 3.3 Azure DevOps Pipeline

The parent-pipeline (*[00-pipeline.yml](https://github.com/dewolfs/graph-api-powershell-sdk/blob/main/pipeline/00-pipeline.yml)*) calls the child-pipelines (*[01-applications.yml](https://github.com/dewolfs/graph-api-powershell-sdk/blob/main/pipeline/01-applications.yml)*, *[02-groups.yml](https://github.com/dewolfs/graph-api-powershell-sdk/blob/main/pipeline/02-groups.yml)*, *[03-groupowners.yml](https://github.com/dewolfs/graph-api-powershell-sdk/blob/main/pipeline/03-groupowners.yml)*) to execute the individual tasks.  The complete pipeline looks like this:  

![pipeline]({{site.baseurl}}/assets/img/2023-04-18-pipeline.png)

To retrieve the secret from the Azure KeyVault, we use the **AzureKeyVault@2** task in our pipeline:

```bash
- task: AzureKeyVault@2
  displayName: 'Get secrets from keyvault'
  inputs:
    azureSubscription: $(azureSubscriptionApp)
    KeyVaultName: $(keyVaultName)
    SecretsFilter: 'secret-spn-prd-graph-applications'
    RunAsPreJob: true
```

Luckily The Azure DevOps hosted agents have the **[Microsoft.Graph PowerShell modules pre-installed](https://github.com/actions/runner-images/blob/main/images/linux/Ubuntu2204-Readme.md#powershell-tools)**.

## 3.4 PowerShell scripts

The image below demonstrate how you can start making authenticated calls to the Microsoft Graph API.  The clientId and tenantId are defined in the variable file (*[aad-variables.yml](https://github.com/dewolfs/graph-api-powershell-sdk/blob/main/input/aad-variables.yml)*) and imported at pipeline runtime.  Thanks to the task **AzureKeyVault@2**, we can use the clientSecret as environment variable in our PowerShell scripts.  

![authn]({{site.baseurl}}/assets/img/2023-04-18-authn.png)

## 3.4.1 Create SPN

- **Input:**  The pipeline parameters *environment* and *uniqueName* are used to construct the correct name of the Application registration and Enterprise Application.

- **Output:**
![spn-output]({{site.baseurl}}/assets/img/2023-04-18-spn-output.png)

## 3.4.2 Create Groups 

- **Input:**  The pipeline parameters *environment* and *uniqueName* are used to construct the correct name of the Azure AD security groups.

- **Output:**
![group-output]({{site.baseurl}}/assets/img/2023-04-18-group-output.png)

## 3.4.3 Set Group Owners

- **Input:**  The file *[azure-subs-01-groupOwners.json](https://github.com/dewolfs/graph-api-powershell-sdk/blob/main/input/azure-subs-01-groupOwners.json)* stored in the repository is used as input file to declare which user needs to be set as Owner to which group(s).  
![owners-input]({{site.baseurl}}/assets/img/2023-04-18-owners-input.png)

- **Output:** 
![owners-output]({{site.baseurl}}/assets/img/2023-04-18-owners-output.png)

# 4. Tools

## 4.1 Graph API X-RAY browser plugin

When looking through the Graph API reference material it can be difficult to figure out which API call matches individual actions.
The **[Microsoft Graph X-Ray browser plugin](https://graphxray.merill.net)** shows used PowerShell, JS, Java, C#, Objective-C, and Go commands to understand and reuse code in your project without having to deep dive into the APIs.

![x-ray]({{site.baseurl}}/assets/img/2023-04-18-x-ray.png)

## 4.2 Graph Explorer tool

**[Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer)** is a developer tool that lets you learn about Microsoft Graph APIs. Use Graph Explorer to try the APIs on the default sample tenant to explore capabilities, or sign in to your own tenant and use it as a prototyping tool to fulfill your app scenarios. This tool includes helpful features such as code snippets (C#, Java, JavaScript, Go and PowerShell), Microsoft Graph Toolkit and adaptive cards integration.

![explorer]({{site.baseurl}}/assets/img/2023-04-18-explorer.png)

*The configuration we used in this post can be found on <https://github.com/dewolfs/graph-api-powershell-sdk>.*