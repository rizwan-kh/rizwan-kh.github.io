title: "(Azure DevOps) ADF CI CD using Azure Pipeline"
date: 2023-06-06T21:28:21-05:00
draft: false
toc: false
images:
tags:
  - azure devops
  - pipelines
  - adf
---
![azure](/azure-pipelines.png)

# Automating Azure Data Factory deployments using Azure DevOps CI/CD

## Introduction

In the ever-evolving landscape of data engineering, having a streamlined and automated process for deploying changes to Azure Data Factory (ADF) is crucial. Azure DevOps provides a robust platform for implementing Continuous Integration and Continuous Deployment (CI/CD) pipelines for ADF, allowing teams to deliver changes efficiently and maintain a reliable data processing workflow.

## Why CI/CD for ADF?

Azure Data Factory is a cloud-based data integration service that allows you to create, schedule, and manage data pipelines. As data engineering projects grow in complexity, manual deployment processes become error-prone and time-consuming. CI/CD for ADF brings several benefits:

- **Efficiency:** Automated deployments eliminate manual errors and reduce deployment time.
  
- **Consistency:** Ensures consistency across development, testing, and production environments.
  
- **Version Control:** Integrates seamlessly with version control systems for better code management.

## Tools of the Trade

To implement ADF CI/CD with Azure DevOps, we leverage two key PowerShell modules:

1. **Az.DataFactory Module:** This module allows us to manage Azure Data Factory resources using PowerShell commands.

2. **[azure.datafactory.tools Module](https://github.com/Azure-Player/azure.datafactory.tools):** This custom module provides cmdlets for deploying and managing Azure Data Factory artifacts.

## Implementing CI/CD with Azure DevOps

### 1. Setting Up the Pipeline

In your Azure DevOps project, create a new pipeline with the necessary triggers and configurations.

### 2. Folder Structure

We have a separate template repository for the pipeline code which has the shown folder structure.
| S.No  | Filename                                                      | Filetype  | Description|
|------ |-------------------------------------------------------------- |---------- |------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1     | README.md                                | markdown       | README markdown file|
| 2     | adf/build-template.yml                   | YAML           | Validate and Build Template YAML file for ADF|
| 3     | adf/deploy-template.yml                  | YAML           | Publish Template YAML file for ADF|

#### Validate Template - ADF
The Validate/Build template process involves installing the PowerShell module azure.datafactory.tools and then validating the Azure Data Factory (ADF) module. It consists of two stages: incremental and full-deploy. The decision of which stage to execute is determined by the value of the Azure Pipelines Library variable fulldeploy. If the value is true, the job BuildAndPublishFull is executed, which deploys the full ADF code available on the Git repository. If the value is false, the job BuildAndPublishIncremental is executed, which promotes only the files that have changed between the last two commits.

#### Publish Template - ADF
The Publish template process involves installing the PowerShell module azure.datafactory.tools and downloading the artifacts published in the Build/Validate stage. In this stage, dynamic variables are replaced based on the environment name, using the deployment/config-<env>.csv file for variable replacement. After the variable replacement, the ADF files are published.


### 3. CI/CD Triggers

Set up CI triggers to automatically run the pipeline on code commits to your ADF repository. This ensures that any changes are validated and deployed promptly.

### 4. Incremental and Full Deployment

This logic is implemented in the validate script to perform either incremental or full deployment based on a flag. This flexibility allows for targeted updates or complete overhauls, depending on your requirements.

## Template Pipeline Scripts

```yml
# build-template.yml
jobs:
- job: ValidateADFCode
  steps:
  - powershell: |
      Install-Module Az.DataFactory -MinimumVersion "1.10.0" -Force
      Install-Module -Name "azure.datafactory.tools" -Force
      Import-Module -Name "azure.datafactory.tools" -Force
      $r = Test-AdfCode -RootFolder $pwd.Path
      if ($r.ErrorCount -gt 0) {
          throw "Error in Validation - $($r.ErrorCount) error(s) found. Cancelling the pipeline!"
      }
    displayName: 'Validate Azure Data Factory Code'

- job: BuildAndPublishIncremental
  dependsOn: ValidateADFCode
  condition: and(succeeded(), eq(variables.fulldeploy, false)) # only publish artifacts for incremental changes
  steps:
  - checkout: git://x-ops/adf-devops-pipelines@main
  - checkout: self
    fetchDepth: 0
  - script: |
      echo "Incremental changes only"
      echo "connectedServiceNameARM: $(connectedServiceNameARM)"
      echo "dataFactoryName: $(dataFactoryName)"
      echo "environment: $(environment)"
      echo "location: $(location)"
      echo "resourceGroupName: $(resourceGroupName)"
      cd $(Build.Repository.Name)
      git status
      git branch
      diff=$(git diff HEAD..HEAD~ --name-only)
      mkdir -p adf
      for file in $diff; do 
        echo $file >> changedFiles.txt;
        ## copy the files along with directory to a folder and publish that as artifact
        cp --parents $file adf/.
      done
      ls -lrt adf/
      cat changedFiles.txt
  - publish: $(System.DefaultWorkingDirectory)/$(Build.Repository.Name)/adf
    artifact: ADFJson

- job: BuildAndPublishFull
  dependsOn: ValidateADFCode
  condition: and(succeeded(), eq(variables.fulldeploy, true)) # publish full adf artifacts
  steps:
  - checkout: git://x-ops/adf-devops-pipelines@main
  - checkout: self
    fetchDepth: 0
  - script: |
      echo "Full Publish"
      echo "connectedServiceNameARM: $(connectedServiceNameARM)"
      echo "dataFactoryName: $(dataFactoryName)"
      echo "environment: $(environment)"
      echo "location: $(location)"
      echo "resourceGroupName: $(resourceGroupName)"
      cd $(Build.Repository.Name)
      git status
  - publish: $(System.DefaultWorkingDirectory)/$(Build.Repository.Name)
    artifact: ADFJson
```

```yml
# deploy-template.yml
jobs:
- deployment: Deploy
  displayName: "Publishing to ${{ parameters.environment }} environment"
  environment: ${{ parameters.environment }}
  strategy:
    runOnce:
      deploy:
        steps:
        - powershell: |
            Install-Module Az.DataFactory -MinimumVersion "1.10.0" -Force
            Install-Module -Name "azure.datafactory.tools" -Force
            Import-Module -Name "azure.datafactory.tools" -Force
          displayName: 'Installing PowerShell Module'

        - task: AzurePowerShell@5
          displayName: 'Deploy Azure DataFactory'
          inputs:
            connectedServiceNameARM: $(connectedServiceNameARM)
            ScriptType: InlineScript
            Inline: |
              $opt = New-AdfPublishOption
              $opt.DeleteNotInSource = $$(deleteNotInSource)
              Publish-AdfV2FromJson -RootFolder "$(Pipeline.Workspace)/ADFJson/" -ResourceGroupName "$(resourceGroupName)" -DataFactoryName "$(dataFactoryName)" -Location "$(location)" -Stage "$(environment)" -Option $opt
            FailOnStandardError: true
            azurePowerShellVersion: LatestVersion
```

```yml
# azure-pipelines.yml
# this file will be in the ADF Repo
pool:  
  vmImage: 'ubuntu-latest'
    
resources:
  repositories:
  - repository: templates
    type: git    
    name: x-ops/adf-devops-pipelines
    ref: main
    
variables:
- group: adf-env-vg

stages:
- stage: BuildValidate
  displayName: Build and Validate ADF  
  jobs:  
  - template: adf/build-template.yml@templates

- stage: Publish2Dev
  displayName: 'Publish ADF to Development'
  dependsOn: 'BuildValidate'
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main')) # refer to main or tags
  jobs:
  - template: adf/adf-deploy-template.yml@templates
    parameters:
      environment: 'dev-env'
  variables:
  - group: adf-env-dev-vg

- stage: Publish2Prod
  displayName: 'Publish ADF to Production'
  dependsOn: 
  - BuildValidate
  - Publish2Dev
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main')) # refer to main or tags
  jobs:
  - template: adf/deploy-template.yml@templates
    parameters:
      environment: 'prod-env'
  variables:
  - group: adf-env-prod-vg
```

## Conclusion

Automating CI/CD for Azure Data Factory using Azure DevOps and PowerShell provides a robust solution for managing data engineering workflows. With the power of version control, scripted deployments, and automated testing, teams can enhance collaboration and deliver high-quality data pipelines.

Start leveraging the benefits of ADF CI/CD today, and witness the transformation in your data engineering practices.

Happy Deploying!

---