---
# layout: post
title: OneLake Security in Microsoft Fabric — What You Need to Know
description: >-
    The next evolution of OneLake security.
author: clyde
date: 2025-04-13
categories: [Microsoft Fabric]
tag: [Fabric Security]
# pin: true
image:
  path: /commons/2025-04-13-onelake-security-in-microsoft-fabric-header.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  #alt: Your alt-text here.
---
If you're new to Fabric, think of it like this: It’s a unified platform for data and AI — and at the heart of it is OneLake — your single, massive data lake for your whole organization.

Now, OneLake Security makes sure that no matter where your data is used — Power BI, SQL, Notebooks — the security rules stay consistent. ✅ One copy of data. One set of access rules. No duplicates. No confusion.

### OneLake Data Access Interface

You start in your Lakehouse. Click “Manage OneLake Data Access” — and you’re in the role manager.

Every data item starts with a default role.This gives workspace users with write access the ability to work with everything.
But you can customize this completely.

### Role Creation

Just give it a name, then pick the data you want this role to access — tables, schemas, folders, even shortcuts.

This is a grant-by-assignment model — Users don’t get access by default. You explicitly give access by assigning them to a role.

You can add individual users, user groups, service principals, or managed identities.
Once you’re happy, click Create.

### Column Level and Row Level Security Options

Now, let’s talk fine-grained control.
Need to hide specific columns? Turn on Column-Level Security.
Want to limit rows based on department, region, or user role? Use Row-Level Security with T-SQL.

You can combine these into one powerful role, or break them out into specialized ones.

### Example Setup of Multiple Roles

Here’s a real-world example:
Suppliers Readers → Can only see the Suppliers table.
Customer Data Readers → Can view Customers and Orders, but with sensitive columns hidden.
HR Data U.S. → Can only see U.S. employee rows.

You can even apply roles to shortcuts — and the security still holds. 🔐 Security travels with the data.

### Power BI, SQL Endpoint and Notebooks in action

The cool part? These roles work across every tool.

- In Power BI: only allowed data appears in reports
- In Notebooks: data science work respects your role
- In SQL: users can query, but only what they’re permitted to see

Even if someone tries to browse OneLake directly — 🔒 Access is still locked down.

### Summary

OneLake Security brings cross-engine, centralized data protection to your lake. No more managing security in five different places. One place. One definition. OneLake.

It’s now in preview — go check it out and give your data the protection it deserves!

📌 Links to learn more:

- [Microsoft Fabric updates blog](https://blog.fabric.microsoft.com/en-us/blog/the-next-evolution-of-onelake-security-enters-early-preview?ft=All)
- [OneLake security documentation](https://learn.microsoft.com/en-us/fabric/onelake/security/get-started-security)
