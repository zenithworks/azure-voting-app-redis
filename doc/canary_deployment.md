[Ronai doc](https://microsoft.sharepoint.com/:w:/t/ReleaseManagement/RM/ERqt4k3Sd_FBr4jjw-omnK8BFtPB-3OSiAhmeKYcayKAgw?e=3TcaKU&CID=69E5F544-26D0-4672-BB5F-3AB4FD2C380E&wdLOR=c94A93656-AAF3-469F-A6E2-EB27242C1306) 
<br />
[release notes](https://github.com/MicrosoftDocs/azure-devops-docs-pr/blob/releasenotes/sprint-160-update/release-notes/2019/_shared/pipelines/sprint-160-update.md)
<br />
[main doc toc](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/deployment-jobs?view=azure-devops)
[azure devops schema](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=azure-devops&tabs=example)

## ----------

---
title: Deployment jobs
description: Deploy to resources within an environment
ms.prod: devops
ms.technology: devops-cicd
ms.topic: conceptual
ms.assetid: fc825338-7012-4687-8369-5bf8f63b9c10
ms.manager: mijacobs
ms.author: ronai
author: RoopeshNair
ms.date: 5/2/2019
monikerRange: 'azure-devops'
---

# Deployment jobs

[!INCLUDE [version-team-services](../_shared/version-team-services.md)]

> [!NOTE]
>
> To use deployment jobs, [make sure the Multi-stage pipelines experience is turned on](../../project/navigation/preview-features.md).

In YAML pipelines, we recommend that you put your deployment steps in a deployment job. A deployment job is a special type of [job](phases.md) that is a collection of steps to be run sequentially against the environment.

## Overview

Using deployment job provides some benefits:

 - **Deployment history**: You get end-to-end deployment history across pipelines down to a specific resource and status of the deployments for auditing.
 - **Apply deployment strategy**: Define how your application is rolled-out.

   > [!NOTE] 
   > At the moment we offer only *runOnce* and *canary* strategy. Additional strategies like rolling, blue-green are on our roadmap.

## Schema

Here's the full syntax to specify a deployment job: 

```YAML
jobs:
- deployment: string   # name of the deployment job, A-Z, a-z, 0-9, and underscore
  displayName: string  # friendly name to display in the UI
  pool:                # see pool schema
    name: string
    demands: string | [ string ]
  dependsOn: string 
  condition: string 
  continueOnError: boolean                # 'true' if future jobs should run even if this job fails; defaults to 'false'
  timeoutInMinutes: nonEmptyString        # how long to run the job before automatically cancelling
  cancelTimeoutInMinutes: nonEmptyString  # how much time to give 'run always even if cancelled tasks' before killing them
  variables: { string: string } | [ variable | variableReference ]  
  environment: string  # target environment name and optionally a resource-name to record the deployment history; format: <environment-name>.<resource-name>
  strategy: [ deployment strategy ]
    runOnce:
      deploy:
        displayName: string                 # friendly name to display in the UI
        steps:
        - script: [ script | bash | pwsh | powershell | checkout | task | templateReference ]
```

Here is the syntax of the deployment strategies supported:

### RunOnce deployment strategy:

RunOnce is the simplest deployment strategy wherein the 'deploy' job is run only once to execute the steps it contains. We would recommned this strategy, if you're just getting started with deployment jobs.

```YAML
strategy: 
    runOnce:
      deploy:
        displayName: string                 # friendly name to display in the UI
        steps:
        - script: [ script | bash | pwsh | powershell | checkout | task | templateReference ]
```

### Canary deployment strategy:

Canary deployment strategy is an advance deployment strategy which helps in mitigating the risk involved in rolling new version of application. Using this you can reduce the risk by slowly rolling out the change to a small subset of users. As you gain more confidence in the new version, you can start releasing it to more servers in your infrastructure and routing more users to it.


```YAML
strategy: 
    canary:
      increments: [ number ]
      pre-deploy:
        displayName: string                 # friendly name to display in the UI
        steps:
        - script: [ script | bash | pwsh | powershell | checkout | task | templateReference ]
      deploy:
        displayName: string                 # friendly name to display in the UI
        steps:
        - script: [ script | bash | pwsh | powershell | checkout | task | templateReference ]
      postRouteTraffic:
        displayName: string                 # friendly name to display in the UI
        steps:
        - script: [ script | bash | pwsh | powershell | checkout | task | templateReference ]
      On:
        failure:
            displayName: string                 # friendly name to display in the UI
            steps:
            - script: [ script | bash | pwsh | powershell | checkout | task | templateReference ]
        success:
            displayName: string                 # friendly name to display in the UI
            steps:
            - script: [ script | bash | pwsh | powershell | checkout | task | templateReference ]
```





## Examples

### RunOnce Deployment strategy

The following example YAML snippet showcases a simple use of a deploy job using runOnce deployment strategy - 

```YAML
jobs:
  # track deployments on the environment
- deployment: DeployWeb
  displayName: deploy Web App
  pool:
    vmImage: 'Ubuntu-16.04'
  # creates an environment if it doesn't exist
  environment: 'smarthotel-dev'
  strategy:
    # default deployment strategy, more coming...
    runOnce:
      deploy:
        steps:
        - script: echo my first deployment
```

In the above example, with each run of this job, deployment history is recorded against the "smarthotel-dev" environment.

> [!NOTE]
> - Currently only Kubernetes resources are supported within an environment, with support for VMs and other resources on the roadmap.
> - It is also possible to create an environment with empty resources and use that as an abstract shell to record deployment history as shown in the previous example.

The following example snippet demonstrates how a pipeline can refer an environment and a resource within the same to be used as the target for a deployment job.

```YAML
jobs:
- deployment: DeployWeb
  displayName: deploy Web App
  pool:
    vmImage: 'Ubuntu-16.04'
  # records deployment against bookings resource - Kubernetes namespace
  environment: 'smarthotel-dev.bookings'
  strategy: 
    runOnce:
      deploy:
        steps:
          # No need to explicitly pass the connection details
        - task: KubernetesManifest@0
          displayName: Deploy to Kubernetes cluster
          inputs:
            action: deploy
            namespace: $(k8sNamespace)
            manifests: |
              $(System.ArtifactsDirectory)/manifests/*
            imagePullSecrets: |
              $(imagePullSecret)
            containers: |
              $(containerRegistry)/$(imageRepository):$(tag)
```

This approach has the following benefits:
- Records deployment history on a specific resource within the environment as opposed to recording the history on all resources within the environment.
- Steps in the deployment job **automatically inherit** the connection details of the resource (in this case, a kubernetes namespace: `smarthotel-dev.bookings`), since the deployment job is linked to the environment. 
This is particularly useful in the cases where the same connection detail is to be set for multiple steps of the job.




### Canary deployment strategy

In the following example, the canary strategy for AKS will first deploy the changes with 10% pods followed by 20% while monitoring the health during postRouteTraffic. If all goes well, it will promote to 100%.  

```YAML
jobs: 
- deployment: 
  environment: musicCarnivalProd 
  pool: 
    name: musicCarnivalProdPool  
  strategy:                  
    canary:      
      increments: [10,20]  
      preDeploy:                                     
        steps:           
        - script: initialize, cleanup....   
      deploy:             
        steps: 
        - script: echo deploy updates... 
        - task: KubernetesManifest@0 
          inputs: 
            action: $(strategy.action)       
            namespace: 'default' 
            strategy: $(strategy.name) 
            percentage: $(strategy.increment) 
            manifests: 'manifest.yml' 
      postRouteTaffic: 
        pool: server 
        steps:           
        - script: echo monitor application health...   
      on: 
        failure: 
          steps: 
          - script: echo clean-up, rollback...   
        success: 
          steps: 
          - script: echo checks passed, notify... 
```