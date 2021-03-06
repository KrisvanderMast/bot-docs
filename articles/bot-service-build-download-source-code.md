---
title: Download and redeploy bot source code | Microsoft Docs
description: Learn how to download and publish a Bot Service.
keywords: download source code, redeploy, deploy, zip file, publish
author: ivorb
ms.author: v-ivorb
manager: kamrani
ms.topic: article
ms.prod: bot-framework
ms.date: 09/20/2018
---
# Download and redeploy bot code
Azure Bot Service allows you to download the entire source project for your bot. This allows you to work on your bot locally using an IDE of your choice. Once you are done making changes, you can publish your changes back to Azure. This topic will show you how to download your bot's source code and publish the changes back to Azure. 

## Prerequisites
- Install [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/?view=azure-cli-latest)
- Install az botservice extension by using `az extension add -n botservice` command

## Download code 
To download code from the Azure portal, do the following:
1. Open the blade for the bot.
2. Under the **Bot management** section, Click **Build**.
3. Click **Download zip file**. 
4. Extract the .zip file to a local directory.
5. Navigate to the extracted folder and open the source files in your favorite IDE.

Now make changes to your sources by either editing existing source files or adding new ones to your project.

Login to Azure CLI by running the following command.
```azcli
az login
```
You will be prompted with a unique temporary auth code. To sign in, use a web browser and visit Microsoft [device login](https://microsoft.com/devicelogin), and paste the code provided by the CLI to continue.

To download code using `az` cli, use the following command:
```azcli
az bot download --name "my-bot-name" --resource-group "my-resource-group"`
```

## Publish code using Azure CLI
To publish code back to Azure using `az` cli, use the following command:
```azcli
az bot publish --name "my-bot-name" --resource-group "my-resource-group" --code-dir <path to directory> 
```

If ```code-dir``` is not provided, ```az bot publish``` will use the local directory to publish.

## Publish code using Visual Studio
In addition to the `az` cli, you can also use Visual Studio to publish your **C#** code changes back to the Azure portal.
1. In Visual Studio, right-click your project name and click **Publish...**. The **Publish** window opens.

![Azure publish](~/media/azure-bot-build/azure-csharp-publish.png)

2. Select the profile for your project.
3. Copy the password listed in the _publish.cmd_ file in your project.
4. Click **Publish**.
5. When prompted, enter the password that you copied in step 3.   

After your project configured, your project changes will be published to Azure.

## Next steps
Now that you know how to upload changes back to Azure, you can setup continuous deployment for your bot.

> [!div class="nextstepaction"]
> [Set up continuous deployment](bot-service-build-continuous-deployment.md)
