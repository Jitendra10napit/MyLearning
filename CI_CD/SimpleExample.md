Here's the simplest complete pipeline: a **.NET Web API** goes through **build**, then deploy to **Dev**, then deploy to **Prod** on **Azure App Service**.

## The pipeline (`azure-pipelines.yml`)

```yaml
# 1. TRIGGER – run when code is merged to main
trigger:
  - main

# 2. AGENT – machine that runs the pipeline
pool:
  vmImage: 'ubuntu-latest'

# 3. VARIABLES
variables:
  buildConfiguration: 'Release'

stages:

# ================= STAGE 1: BUILD (CI) =================
- stage: Build
  jobs:
  - job: Build
    steps:
    - task: UseDotNet@2                 # install .NET SDK
      inputs:
        version: '10.x'

    - script: dotnet restore            # download NuGet packages
      displayName: 'Restore'

    - script: dotnet build -c $(buildConfiguration) --no-restore
      displayName: 'Build'

    - script: dotnet test -c $(buildConfiguration) --no-build
      displayName: 'Run Unit Tests'

    - script: dotnet publish src/MyApi -c $(buildConfiguration) -o $(Build.ArtifactStagingDirectory)
      displayName: 'Publish'

    - publish: $(Build.ArtifactStagingDirectory)   # save output as artifact
      artifact: drop

# ================= STAGE 2: DEPLOY TO DEV (CD) =================
- stage: Deploy_Dev
  dependsOn: Build
  jobs:
  - deployment: DeployDev
    environment: 'dev'                  # no approval
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            inputs:
              azureSubscription: 'azure-connection'
              appName: 'myapi-dev'
              package: '$(Pipeline.Workspace)/drop'

# ================= STAGE 3: DEPLOY TO PROD (CD) =================
- stage: Deploy_Prod
  dependsOn: Deploy_Dev
  jobs:
  - deployment: DeployProd
    environment: 'prod'                 # approval required (set in Azure DevOps)
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            inputs:
              azureSubscription: 'azure-connection'
              appName: 'myapi-prod'
              package: '$(Pipeline.Workspace)/drop'
```

## Flow in one line

```
Code merged → Restore → Build → Test → Publish artifact → Deploy Dev → (Approval) → Deploy Prod
```

## Key terms to know

| Term | Simple meaning |
|---|---|
| **Trigger** | When the pipeline starts (a merge to `main`) |
| **Pool / Agent** | The machine that runs the steps |
| **Stage** | A big phase: Build, Dev, Prod |
| **Job** | A group of steps that runs on one agent |
| **Step / Task** | A single action: build, test, deploy |
| **Artifact** | The build output (`drop`), built **once** and reused in every stage |
| **Environment** | Dev/Prod targets where you set **approvals** |
| **Service connection** | `azure-connection`, the secure login from the pipeline to Azure |

## How to speak about it (about 30 seconds)

"Our pipeline has two parts. **CI**: when code is merged to main, it restores packages, builds, runs unit tests, publishes the API and saves the output as an **artifact**. **CD**: the **same artifact** is deployed to Dev automatically, and after a **manual approval** it's deployed to Prod. Approvals are configured on the Azure DevOps **environment**, and the pipeline connects to Azure through a **service connection**, so no passwords are kept in the YAML. Because we build once and deploy the same artifact everywhere, what we tested in Dev is exactly what goes to Prod."

**If they ask "what would you add next?":** code coverage and a SonarQube scan, a Docker image instead of a zip, a staging slot with a swap for zero downtime, and a health check after deployment.
