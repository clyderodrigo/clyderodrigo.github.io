---
#layout: post
title: My Intro to Azure Bicep —Exploring Infrastructure as Code (IaC)
description: >-
    Deploying environments with Bicep, scaling IaC with modularization & zone design.
author: clyde
date: 2025-05-04
categories: [Azure DevOps, Infrastructure as Code]
tag: [Azure DevOps, Infrastructure as Code, Bicep, Azure CLI]
pin: true
image:
  path: https://fictal.com/wp-content/uploads/2022/03/azure-bicep-logos-1000x493.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  #alt: Your alt-text here.
---
## “Simplicity is the ultimate sophistication” —probably every engineer at some point, and Leonardo Da Vinci.

I'm NOT diving into this as a DevOps or CI/CD pro. Rather, an aspiring one — keen to explore the technical verticals of what I'm seeing on-the-job.
Heck. Until a year and a half ago I wasn't even in engineering and consulting (I was in HR. Long story..).

> The infrastructure layer of an analytics platform gets deep fast —networks, storage, services/compute, security, governance, and it branches into environments. 
{: .prompt-info }

You need consistency. And, at work I experienced deploying infrastructure for an analytics platform solution with Azure DevOps pipelines. 

Infrastructure as Code (IaC) is intriguing and powerful stuff.


## My Infrastructure Deployment @ work —for Analytics Platforms

![image1](/posts/images/20250504-0.png)

## 1| Why Infrastructure-as-Code matters

In data & analytics engineering, the default focus is on data pipelines, lake/lakehouse/warehouse storage, dimensional modelling, semantic modelling, and dashboards that bring an Analytics Platform's operations to life. If the infrastructure layer isn't repeatable and extensible (not forgetting secure), well then you'll find yourself clicking your way through provisioning Azure resources for your next projects and you'll find that each one is duct taped together a little differently- in ways that might cause problems later on.

IaC solves this. Infrastructure setups are codified, version controlled and deployable across environments with no drama.


> Production-grade Azure IaC is like a machine with many cogs working together to get things moving.
{: .prompt-info }

Production-grade Azure IaC scope:

1. Azure Bicep (or Terraform)
2. Modular and extensible infrastructure-as-code
3. Deployments with environment separation
4. CI/CD automation through Azure DevOps
5. Alignment with Microsoft’s Cloud Adoption Framework & recommendations
6. Policy-as-Code (Governance-as-Code)

## 2| What is Azure Bicep and why did I start there?

To quote Microsoft, "Bicep is a domain-specific language that uses declarative syntax to deploy Azure resources. In a Bicep file, you define the infrastructure you want to deploy to Azure and then use that file throughout the development lifecycle to repeatedly deploy that infrastructure. Your resources are deployed in a consistent manner."

It simplifies Azure resource deployments. And it's a clean, readable next-gen upgrade to ARM templates.

What makes Bicep tick:

* Native to Azure
* Reads like code
* Built for modularity and reuse

While 'Terraform' exists as a cross-platform alternative, Bicep is a no brainer for Azure IaC.

[Bicep Overview – MS Docs](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview)

🗡️

## 3| Deploying Environments with Bicep
We focus on using Bicep to deploy an environment —your IaC example will deploy just a storage account. I will install the Bicep CLI locally and write a simple bicep file to deploy a storage account on Azure.

I will employ paramterization in your code. When deploying resources across multiple environments (dev, test, prod), parameterization lets you Keep the core logic clean and reuse the same logic across environment deployments.

**Hands-On:** Installing Bicep CLI, writing a flat Bicep file and a json file for parameterization to deploy a storage account.

### 3.1| Installing Bicep Locally

If you've got the Azure CLI installed (which you should if you’re working in Azure), run:

`bash`
```bash
az bicep install
```

Done. Verify it:

```bash
bicep --version
```

![image1](/posts/images/20250504-1.png)
_Bicep installation_

> Install the Bicep extension on VS Code for autocomplete and issue flags.
{: .prompt-tip }

[Install Bicep CLI – MS Docs](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/install)

### 3.2| Writing a Bicep File

> *In this post's scope, we deploy code to a resource group.*

I'm writing a flat bicep file that provisions a Storage Account. Here’s what my Bicep file looks like:

`main.bicep`
```
resource storageAccount 'Microsoft.Storage/storageAccounts@2021-04-01' = {
  name: storageAccountName
  location: resourceGroup().location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
}
```

Now I create a basic parameter file for provisioning our dev environment resource:

`dev.parameters.json`
```
{
  "parameters": {
    "storageAccountName": {
      "value": "devcr46azdemost"
    }
  }
}
```
Similar parameter files can be created for test and production resource provisioning with different naming tiers (Examples:  'tstcr46azdemost', 'prdcr46azdemost').

Together, the bicep file and parameter file code is readable, declarative, and reconfigurable. This made sense.

![image2](/posts/images/20250504-2.png)
_Azure deployment of main.bicep_

### 3.3| Deploying it to Azure

I already have a resource group called dev-cr46-az-demo-rg, so this is how I will run the deployment via CLI:

`bash`
```bash
az deployment group create \
  --resource-group dev-cr46-az-demo-rg \
  --template-file "$HOME/Dev/main.bicep" \
  --parameters @dev.parameters.json
```

![image3](/posts/images/20250504-3.png)
_Successful Azure deployment_

And... that's it for this Infrastructure as code! Run this via CLI and the storage account gets deployed from code.
No clicking around Azure Portal. No manual errors. Just repeatable, testable infrastructure with Bicep and Azure CLI.

🗡️

## 4| Bicep Modules: Designed for Reuse

We focus on modularizing IaC. This is a best practice that moves away from writing flat bicep files and towards designing modular infrastructure that's repeatable. Bicep modules also let you separate logic by domain (network, storage, compute), keeping templates clean and readable.

**Hands-On:** Design and deploy a Bicep module for a virtual network for analytics workloads.

With modules, we move from this Bicep code structure:

```
# main.bicep with everything hardcoded
resource vnet 'Microsoft.Network/virtualNetworks@2021-02-01' = { ... }
resource storage 'Microsoft.Storage/storageAccounts@2021-04-01' = { ... }
```

To this: 

```
module vnet './modules/vnet.bicep' = {
  name: 'vnetModule'
  params: {
    name: 'analytics-vnet'
    addressPrefix: '10.10.0.0/16'
  }
}
```

What you get is a reusable building block for deploying a virtual network resource. Need to deploy several vnets? Just call the vnet module in `main.bicep` and specify the parameters to be passed on, for each vnet. Same logic. Zero duplication.


### 4.1| Create a Module File with Bicep

`vnet.bicep`
``` 
param name string
param addressPrefix string

resource vnet 'Microsoft.Network/virtualNetworks@2021-02-01' = {
  name: name
  location: resourceGroup().location
  properties: {
    addressSpace: {
      addressPrefixes: [addressPrefix]
    }
  }
}
```

### 4.2| Call Module from the Main File

`main.bicep`
```
module vnet './modules/vnet.bicep' = {
  name: 'analytics-vnet'
  params: {
    name: 'vnet-data'
    addressPrefix: '10.20.0.0/16'
  }
}
```

> Keep modules tight with one resource per module, unless tightly coupled with other resources. That way, complexity is composable, and not baked-in.
{: .prompt-tip }

With modularized code, you think like an engineer. I'm not just building for the requirement now. I'm writing for future ones as well.

🗡️


## 5| Zone Design for Stepping up your IaC

If bicep modules are reusable building blocks of resources, 'zones' are blueprints for specific environment setups.
Zones are IaC design for:
- Engineering specific environment setups of modules
- Typically includes resources for logging, identity, security
- Applies governance policies

In enterprise analytics, you deal with sensitive data, compliance requirements and heavy workloads —Zones become essential. A basic analytics-ready zone can include virtual networks, a log analytics workspace, a key vault, Role assignments and so on.

Structure your Infrastructure as code repo like this:
```
/landing-zone/
├── main.bicep
├── modules/
│   ├── vnet.bicep
│   ├── keyvault.bicep
│   ├── loganalytics.bicep
```

In main.bicep, orchestrate your modules:
```
module vnet './modules/vnet.bicep' = {
  name: 'vnet'
  params: { ... }
}

module logs './modules/loganalytics.bicep' = {
  name: 'logAnalytics'
  params: { ... }
}
```

🗡️

## 6| Wrap up
IaC isn't just deploying resources. It's engineering repeatable and extensible environments, and letting me focus on building the analytics platform.

**This is how environments for analytics are built. Not with glue—but with patterns.**

> This bicep post places 4 important cogs in the machine that drives us to production-grade Azure IaC: IaC with Azure Bicep, Parameterization, Bicep modules, and Zone design.
{: .prompt-info }

## References

* [Bicep Overview – Microsoft Docs](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview)
* [Install Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/install)
* [VS Code Bicep Extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-bicep)
* [Bicep Modules – Microsoft Docs](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/modules)
* [Bicep Parameters](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/parameters)
