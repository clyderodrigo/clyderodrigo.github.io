---
# layout: post
title: Exploring Infrastructure as Code (IaC): My Initial Steps with Azure Bicep
description: >-
    Intro to Bicep, it's advantages over ARM templates, and setting up the dev environment. 
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

> *“Simplicity is the ultimate sophistication” —Leonardo Da Vinci, and probably every engineer at some point.*

I'm not going into this as an infrastructure pro.
I'm not from a DevOps background. Heck.. until a year and a half ago I wasn't even in development (I was in HR. Long story..).

Anyway... last week at work, I deployed infrastructure for an analytics platform solution with Azure DevOps pipelines. 
And I found it all very intriguing.

So I'm diving into this as an aspiring Data (and AI) engineering pro —exploring the technical verticals of what I'm seeing on-the-job.
It's not just about surviving the next sprint, right? 🌚 (Sometimes, yes it is.)

## Prebrief: Analytics Platform & Business Intelligence
**Business Intelligence (BI)** is a enterprise practice. It's about having the right info at the right time to make the right decisions. And it's current evolution puts BI tools and analytics insights in the hands of every decision maker, not just data specialists. **Analytics** surfaces meaningful patterns in data using business semantic modelling and advanced modelling(AI/ML) techniques. Insights from Analytics can be descriptive, diagnostic, predictive and prescriptive in nature - ranging from interpretations to recommendations. 

The unified **Analytics Platform** is an automated solution we build for enterprises that removes the complexity surrounding collecting, pre-processing and serving business data from across various sources such as databases, spreadsheets, and cloud services - so that users can focus on getting analytics insights into their business operations. 
Unlike reporting tools that are locked into a given system, the analytics platform is in full business control and adaptive:
- New data sources can be connected as needs evolve
- Analytics consumers can perform self-service analysis with dashboards, reporting, alerts and AI agents 
- Analytics authors can build new dashboards and reporting with ease


## Why Infrastructure-as-Code matters
In data & analytics engineering, the default and expected focus is on ETL pipelines, data lake/lakehouse/warehouse storage, data modelling and dashboards. These are what fundamentally bring an Analytics Platform's operations to life. But if the layout of components of the analytics platform isn't repeatable and extensible (not forgetting secure), well then you'll find yourself clicking your way through provisioning Azure resources for your next projects and you'll find that each one is duct taped together a little differently- in ways that might cause problems later on.
IaC solves this. Infrastructure setups are codified, version controlled and deployable across environments with no drama.

Production-grade Azure IaC is like a machine with many cogs working together to get things moving.
- Azure Bicep
- Modular and extensible infrastructure-as-code
- Deployments with environment separation
- CI/CD automation through Azure DevOps
- Alignment with Microsoft’s Cloud Adoption Framework & recommendations
- Policy-as-Code (Governance-as-Code)


## What is Azure Bicep and why did I start there?

To quote Microsoft, "Bicep is a domain-specific language that uses declarative syntax to deploy Azure resources. In a Bicep file, you define the infrastructure you want to deploy to Azure and then use that file throughout the development lifecycle to repeatedly deploy that infrastructure. Your resources are deployed in a consistent manner."

It simplifies Azure resource deployments. And it's a clean, readable next-gen upgrade to ARM templates.

What makes Bicep tick:

* **Native to Azure**
* Reads like code
* Built for modularity and reuse

While 'Terraform' exists as a cross-platform alternative, Bicep is a no brainer for Azure IaC.

[Bicep Overview – Microsoft Docs](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview)


## Installing Bicep Locally

If you've got the Azure CLI installed (which you should if you’re working in Azure), run:

```Azure CLI
az bicep install
```

Done. Verify it:

```Azure CLI
bicep --version
```

> *Quick Tip:* Install the Bicep extension on VS Code for autocomplete and issue flags.

[Install Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/install)

## Writing a Test Bicep File

> *In this post's scope, we deploy code to a resource group.*

I'm writing a bicep file that provisions a Storage Account. Here’s what my Bicep file (`main.bicep`) looks like:

```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2021-04-01' = {
  name: 'devcr46azdemost'
  location: resourceGroup().location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
}
```

It is readable, declarative, and tweakable. This made sense. 

## Step 3: Deploying It to Azure

I already have a resource group called dev-cr46-az-demo-rg, so this is my deployment command:

```Azure CLI
az deployment group create \
  --resource-group dev-cr46-az-demo-rg \
  --template-file "$HOME/Dev/main.bicep"
```

And... we should be good to go! Run this script—the resource group and storage account gets deployed from code. 
No clicking around Azure Portal. No manual errors. Just repeatable, testable infrastructure with Bicep and Azure CLI.

This bicep demo is the first cog in the machine takes us to production-grade Azure IaC.

## References

* [Bicep Overview – Microsoft Docs](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview)
* [Install Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/install)
* [VS Code Bicep Extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-bicep)








