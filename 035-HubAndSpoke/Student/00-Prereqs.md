# Module 0: Pre-requisites - Ready, Set, Go!

**[Home](../README.md)** - [Next module >](./01-HubNSpoke-basic.md)

## Introduction

A smart Azure engineer always has the right tools in their toolbox. In addition, a good grasp on key foundational networking concepts.

## Description

In this module we'll be setting up all the tools we will need to complete our modules.

- Install the recommended toolset, being one of this:
  - The Powershell way (same tooling for Windows, Linux or Mac):
    - [Powershell core (7.x)](https://docs.microsoft.com/en-us/powershell/scripting/overview)
    - [Azure Powershell modules](https://docs.microsoft.com/en-us/powershell/azure/new-azureps-module-az)
    - [Visual Studio Code](https://code.visualstudio.com/): the Windows Powershell ISE might be an option here for Windows users, but VS Code is far, far better
    - [VScode Powershell extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode.PowerShell)
  - **IMPORTANT**: Although Azure CLI is also usable to complete this Hack-a-thon, in the interest of time we will standardize on using Powershell.
  - The Azure Portal way: not really recommended, but you can still the portal to fulfill most of the modules.

**NOTE:** You can use Azure Powershell and CLI on the Azure Cloud Shell, but running the commands locally along Visual Studio Code will give you a much better experience

## Success Criteria

1. You have an Azure shell at your disposal (Powershell or Azure Cloud Shell)
2. Visual Studio Code is installed.
3. Running `Connect-AzAccount` allows to authenticate to Azure
