---
title: Deploy an Azure App custom container
description: Deploy to Azure App custom container from Azure Pipelines
services: vsts
ms.topic: conceptual
ms.assetid:
ms.custom: seodec18
ms.date: 09/20/2021
monikerRange: '>= tfs-2017'
---

# Deploy an Azure App custom container

You can use App Service on Linux and pipelines to automatically deploy your web app to a [custom container in Azure](/azure/app-service/quickstart-custom-container). 

Azure App Service is a managed environment for hosting web applications, REST APIs, and mobile back ends. You can develop in your favorite languages, including .NET, Python, and JavaScript. 

You'll use the [Azure Web App for Container task
](../tasks/deploy/azure-rm-web-app-containers.md) to deploy to Azure App Service in your pipeline.

To learn how to deploy to an Azure Web App without a container, see [Deploy an Azure Web App](webapp.md). 

::: moniker range="tfs-2017"

> [!NOTE]
> This guidance applies to Team Foundation Server (TFS) version 2017.3 and later.

::: moniker-end

## Prerequisites

* An Azure account with an active subscription. [Create an account for free](https://azure.microsoft.com/free/?WT.mc_id=A261C142F).
* A deployed App Service on Linux custom container. To get started, see [Run a custom container in Azure](/azure/app-service/quickstart-custom-container).

## Build and push your image

Azure Pipelines can be used to [push images](../ecosystems/containers/push-image.md) to container registries such as Azure Container Registry (ACR), Docker Hub, Google Container Registries, and others. This example pushes an image to Azure Container Registry. 

# [YAML](#tab/yaml/)

1. Sign in to your Azure DevOps organization and navigate to your project.
1. Go to **Pipelines**, and then select **New Pipeline**.
1. Select **GitHub** as the location of your source code and select your repository.

   > [!NOTE]
   > You might be redirected to GitHub to sign in. If so, enter your GitHub credentials.
   > You might be redirected to GitHub to install the Azure Pipelines app. If so, select **Approve and install**.

1. From the Configure tab, select the **Docker - Build and push an image to Azure Container Registry** task.

1. Select your **Azure Subscription**, and then select your **Container registry** from the dropdown menu.

1. Provide an **Image Name** to your container image, and then select **Validate and configure**.

   As Azure Pipelines creates your pipeline, it:

   * Creates a _Docker registry service connection_ to enable your pipeline to push images to your container registry.

   * Generates an *azure-pipelines.yml* file, which defines your pipeline.

1. Save and run your pipeline. Your YAML pipeline should look similar to this:

```yaml
- stage: Build
  displayName: Build and push stage
  jobs:  
  - job: Build
    displayName: Build job
    pool:
      vmImage: $(vmImageName)
    steps:
    - task: Docker@2
      displayName: Build and push an image to container registry
      inputs:
        command: buildAndPush
        repository: $(imageRepository)
        dockerfile: $(dockerfilePath)
        containerRegistry: $(dockerRegistryServiceConnection)
        tags: |
          $(tag)
```


::: moniker range="azure-devops-2019"

If you're an experienced pipeline user and already have a YAML pipeline to build your .NET Core app, then you might find the examples below useful.

::: moniker-end

::: moniker range="< azure-devops-2019"

YAML pipelines aren't available on TFS.

::: moniker-end

# [Classic](#tab/classic/)

::: moniker range="< azure-devops"

> [!TIP] 
> If you're new to Azure DevOps Server or TFS, then see [Create your first pipeline](../create-first-pipeline.md) before you start.

::: moniker-end

To get started: 

1. Fork this repo in GitHub, or import it into Azure Repos:

   ```
   https://github.com/Microsoft/devops-project-samples/tree/master/dotnet/aspnetcore/container/Application
   ```
2. Create a pipeline and select the **Docker** template. This selection automatically adds the tasks required to build the code in the sample repository.

3. Save the pipeline and queue a build to see it in action.

4. Create a release pipeline and select the **Empty job** for your stage. Search for the **AzureWebAppContainer** task and configure.

5. Link the build pipeline to this release pipeline as an artifact. Save the release pipeline and create a release to see it in action.

---

Now that the build pipeline is in place, you will learn a few more common configurations to customize the deployment of the Azure Container Web App.

## Deploy with Azure Web App for Container

# [YAML](#tab/yaml/)

::: moniker range=">= azure-devops-2019"

Deploy to an Azure App custom container with the [Azure Web App for Container task](../tasks/deploy/azure-rm-web-app-containers.md). 

```yaml

trigger:
- master

resources:
- repo: self

variables: 
  ## Add this under variables section in the pipeline
  azureSubscription: <Name of the Azure subscription>
  appName: <Name of the Web App>
  containerRegistry: <Name of the Azure container registry>
  dockerRegistryServiceConnection: '4fa4efbc-59af-4c0b-8637-1d5bf7f268fc'
  imageRepository: <Name of image repository>
  dockerfilePath: '$(Build.SourcesDirectory)/Dockerfile'
  tag: '$(Build.BuildId)'

  vmImageName: 'ubuntu-latest'

stages:
- stage: Build
  displayName: Build and push stage
  jobs:
  - job: Build
    displayName: Build
    pool:
      vmImage: $(vmImageName)
    steps:
    - task: Docker@2
      displayName: Build and push an image to container registry
      inputs:
        command: buildAndPush
        repository: $(imageRepository)
        dockerfile: $(dockerfilePath)
        containerRegistry: $(dockerRegistryServiceConnection)
        tags: |
          $(tag)


    ## Add the below snippet at the end of your pipeline
    - task: AzureWebAppContainer@1
      displayName: 'Azure Web App on Container Deploy'
      inputs:
        azureSubscription: $(azureSubscription)
        appName: $(appName)
        containers: $(containerRegistry)/$(imageRepository):$(tag)
```

The **Azure Web App on Container** task will pull the appropriate Docker image corresponding to the BuildId from the repository specified, and then deploy the image to your Azure App Service on Linux.

::: moniker-end

::: moniker range="< azure-devops-2019"

YAML pipelines aren't available on TFS.

::: moniker-end

# [Classic](#tab/classic/)
The simplest way to deploy to an Azure Web App Container is to use the **Azure Web App On Container Deploy** task.
This task is added to the release pipeline when you select the deployment task for Azure Web App on Container deployment.
Templates exist for apps developed in various programming languages. If you can't find a template for your language, select the generic **Azure App Service Deployment** template.

---

## Deploy to a slot

# [YAML](#tab/yaml/)

::: moniker range=">= azure-devops"

You can configure the Azure Web App container to have multiple slots. Slots allow you to safely deploy your app and test it before making it available to your customers.

The following YAML snippet shows how to deploy to a staging slot, and then swap to a production slot:

```yaml
- task: AzureWebAppContainer@1
  inputs:
    azureSubscription: '<Azure service connection>'
    appName: '<Name of the web app>'
    containers: $(containerRegistry)/$(imageRepository):$(tag)
    deployToSlotOrASE: true
    resourceGroupName: '<Name of the resource group>'
    slotName: staging

- task: AzureAppServiceManage@0
  inputs:
    azureSubscription: '<Azure service connection>'
    WebAppName: '<name of web app>'
    ResourceGroupName: '<name of resource group>'
    SourceSlot: staging
    SwapWithProduction: true
```
::: moniker-end

::: moniker range="< azure-devops-2019"

YAML pipelines aren't available on TFS.

::: moniker-end

# [Classic](#tab/classic/)

You can configure the Azure Web App for container to have multiple slots. Slots allow you to safely deploy your app and test it before making it available to your customers.
Use the option **Deploy to Slot** in the **Azure Web App Container** task to specify the slot to deploy to. You can swap the slots by using the **Azure App Service Manage** task.

---

## FAQ
### How do I find my registry credentials for the web app?

<a name="endpoint"></a>

App Service needs information about your registry and image to pull the private image. In the [Azure portal](https://portal.azure.com), go to **Container settings** from the web app and update the **Image source, Registry** and save.

![Screenshot showing Update image source and Registry in container settings.](media/webapp-linux/container-settings.png)
