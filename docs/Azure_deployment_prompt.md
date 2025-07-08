# Azure 3-Tier Application Deployment Prompt (Azure Container Apps & Azure Dev CLI)

This document provides a structured prompt for GitHub Copilot (Agent Mode) to scaffold a production-ready Azure Developer CLI (azd) based deployment of a three-tier application. The architecture consists of a web frontend, a REST API backend (both in .NET Core), and an Azure SQL database. The primary deployment target is Azure Container Apps, with notes on alternatives using Azure Web Apps and Azure Kubernetes Service (AKS). Infrastructure-as-Code is done with Azure Bicep, and the deployment workflow is managed via azd and GitHub Actions CI/CD. Secrets are stored in Azure Key Vault and accessed securely using Managed Identities and OpenID Connect (OIDC) for GitHub Actions (avoiding long-lived cloud credentials). Follow the steps and modular sections below to have Copilot generate the necessary project structure, configuration, and code. Each section outlines specific files or configurations to create, with best practices and optional enhancements.

## Project Structure Setup

Begin by setting up the base project layout and configuration:

- Folder Layout: Create a repository structure with separate folders for each tier and infrastructure:

```md
/api       - .NET Core Web API (backend service)
/web       - .NET Core Web App (frontend service)
/infra     - Infrastructure as Code (Bicep modules)
   ├─ main.bicep               (orchestrator template)
   ├─ modules/
       ├─ containerEnv.bicep   (Azure Container Apps environment + Log Analytics)
       ├─ containerApp.bicep   (parametric module for a Container App instance)
       ├─ azureSql.bicep       (Azure SQL Server + Database)
       ├─ keyVault.bicep       (Azure Key Vault for secrets)
       ├─ acr.bicep            (Azure Container Registry)
       ├─ (optional) aks.bicep (AKS cluster, for the AKS variant)
       └─ (optional) webapp.bicep (App Service plan & Web App, for the Web Apps variant)
```

azure.yaml - Azure Developer CLI project configuration
This structure separates the application code (api, web) from the infrastructure definition (infra). The Bicep files are organized into modules for clarity and reuse (each major resource in its own file) and a main template that composes them.

- Azure Developer CLI Configuration (azure.yaml): Define an azure.yaml at the repository root to map the services to Azure resources. For example:

```yaml
name: MyThreeTierApp    # project name
services:
  web:
    project: ./web
    language: dotnet
    host: containerapp
    docker:               # container build context (if needed)
      context: .
      dockerfile: Dockerfile
  api:
    project: ./api
    language: dotnet
    host: containerapp
    docker:
      context: .
      dockerfile: Dockerfile
```

In the above:

- We declare two services, "web" and "api", pointing to the respective project folders.
- host: containerapp specifies that these will be deployed to Azure Container Apps (serverless container environment).
- language: dotnet (or csharp) tells azd the tech stack.
- The optional docker section can hint how to build the container (though we will handle containerization in CI).

Azure Developer CLI uses this file to understand the project structure and deployment targets. For instance, listing multiple services under services: (web and api) with their paths and host types is supported as shown in Microsoft’s sample. The azure.yaml can also include a pipeline section, but since we will craft custom GitHub Actions, we can omit pipeline config or just note it for completeness.

- Azure Dev CLI environment: By default, azd will create an environment (e.g., "dev") and an Azure resource group for you. You can configure an environment name when running azd (for example, via azd env set AZURE_ENV_NAME dev). The environment name and project name typically combine to form resource names. Ensure the azure.yaml has a unique name and consider specifying resourceGroup or let azd default to {name}-{env} for the RG.

With this structure in place, Copilot should scaffold empty projects for the API and Web (e.g., basic .NET minimal API and a simple Razor Pages or MVC app) and placeholder Bicep files in infra/. Next, we’ll fill in the details of containerization and infrastructure.

## Containerization (Dockerfiles & ACR)

Each application (API and Web) needs a Dockerfile for containerization:

- Dockerfile for Web and API: Use a multi-stage Docker build to produce lean production images. For .NET 6/7/8/9, the first stage uses the .NET SDK to build/publish the app, and the second stage uses the lighter ASP.NET runtime image. For example, a Dockerfile (the same pattern can be used in both api/ and web/ folders):

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src
COPY . ./
RUN dotnet restore
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS final
WORKDIR /app
COPY --from=build /app ./
# (Optional: expose ports if not using Azure-managed ingress)
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

This multi-stage build ensures the final image only contains the published app and the ASP.NET runtime, not the SDK or source files (optimizing image size). Make sure to replace MyApp.dll with the actual project DLL name, and adjust paths if needed (e.g., if the project is in a subfolder).

- Azure Container Registry (ACR): The application images will be stored in an Azure Container Registry. In our Bicep infrastructure (detailed in the next section), we will provision an ACR resource. For now, note that the ACR name and login server (e.g., myregistry.azurecr.io) will be needed for pushing images. Decide on a unique ACR name (global across Azure). We will use Managed Identities for access, meaning we do not need to enable the admin user or store credentials for ACR in GitHub. Instead, our Azure Container Apps will authenticate to ACR via their identities, and our GitHub Actions will use OIDC to push to ACR.
- Image Tags: We will tag images with a unique identifier per build (for example, the Git short SHA). This ensures each deployment uses a fresh image and allows rollbacks if needed. For instance, a tag format could be ${{ github.sha[:7] }} (first 7 characters of the commit hash) or a build number.
- Pushing to ACR: In the CI/CD pipeline, after building the images, use Docker or Azure CLI to push them:
  - Log in to ACR. With Azure CLI authenticated (via OIDC), you can az acr login --name <RegistryName> or use Docker login with ACR credentials. Since we prefer not to store credentials, we will rely on Azure CLI being logged in (via the GitHub OIDC login) to grant push rights.
  - Push the images: e.g. docker push myregistry.azurecr.io/web:${{ github.sha[:7] }} and similarly for the API image.
  - Alternatively, use Azure CLI to build and push in one step with Azure Container Registry Tasks (e.g., az acr build), but standard Docker build/push is straightforward here.
Copilot should generate the Dockerfiles for each project and ensure they work for building the .NET apps. Next, we set up the infrastructure as code in Bicep, tying everything together.

## Infrastructure as Code (Azure Bicep)

Using Azure Bicep, we'll define all the Azure resources required for the 3-tier application. The goal is to have a reusable, azd-compatible Bicep setup. We will create a main Bicep template that orchestrates module deployments for clarity. Key infrastructure components include:

- Azure Container Apps Environment: Before deploying containerized apps, we need a Container Apps Environment (a secure enclave for container apps in a region). This environment requires a Log Analytics Workspace for logging. We will create:
  - Log Analytics Workspace: to collect logs and metrics from container apps.
  - Container App Environment: associated with the Log Analytics workspace. In Bicep, this is a resource of type Microsoft.App/managedEnvironments. We will pass the workspace customer ID and key to configure it.
- Azure Container Apps (Web & API): Define two container app resources (type Microsoft.App/containerApps):
  - Web Container App: This will run the frontend container. Enable ingress with external: true so that it has a public URL. Set targetPort (matching the port the web container listens on, e.g., 80 or 5000) and perhaps transport: http. We will allow Ingress for the web app so it’s accessible publicly.
  - API Container App: This will run the backend API. We configure this with internal ingress only (external: false) – meaning it is only reachable within the Container Apps Environment (for security). The web frontend can reach it over the environment's internal network. Alternatively, one could use Dapr for service-to-service communication, but a simpler approach is internal HTTP ingress.
  - Both container apps will reference images in ACR. In Bicep, we supply the container image name including the ACR login server and tag (e.g., myregistry.azurecr.io/web:<tag>).
  - Scaling: Optionally configure scale rules. For simplicity, we might set minReplicas: 1 and maxReplicas: say 3 for each app, or utilize default scale-to-zero for the API if no traffic. Azure Container Apps supports CPU/Memory and event-driven scaling (including HTTP concurrency). In scaffold, include a basic static scaling config or leave default.
  - Managed Identity: Enable a system-assigned managed identity on each Container App. This is critical for pulling images from ACR and accessing secrets. In Bicep, add:

```json
identity: {
  type: 'SystemAssigned'
}
```

for each container app resource. This instructs Azure to create an identity (an Entra ID service principal) tied to the app.

- Azure Container Registry (ACR): Define an Microsoft.ContainerRegistry/registries resource. Use at least the Basic SKU (sufficient for most needs). We will not enable the admin user (to avoid static credentials). Instead, the CI pipeline and container apps will use Azure AD authentication. The ACR resource can have its own system-assigned identity, but that is not required for our scenario. We do need to ensure the container apps can pull from ACR:
  - Grant the container apps' managed identities AcrPull permission on the registry. This can be done with a Role Assignment resource in Bicep, or alternatively by enabling adminUserEnabled: true and storing credentials in Key Vault (not preferred). We choose the managed identity approach: for each container app identity, create a role assignment of role "AcrPull" on the ACR. For example:

```bicep
resource acrPullRole_web 'Microsoft.Authorization/roleAssignments@2020-04-01-preview' = {
  scope: acrResource
  name: guid(acrResource.id, webContainerApp.identity.principalId, 'AcrPull')
  properties: {
    roleDefinitionId: resourceId('Microsoft.Authorization/roleDefinitions', '7f951dda-4ed3-4680-a7ca-43fe172d538d')  # AcrPull role ID
    principalId: webContainerApp.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

Do similarly for the API app. This assignment ensures the container apps can authenticate to ACR using their identities. (Note: The Azure CLI az containerapp create automatically does this assignment for you when using managed identity and ACR, but in Bicep we explicitly add it.)

- Azure SQL Database: Provision an Azure SQL server and a database for the application:
  - SQL Server: Use Microsoft.Sql/servers with an admin username and password (for initial setup). The admin credentials will be stored securely (more on that in Secrets Management). Optionally enable Azure AD admin if planning to use managed identity for DB access (advanced scenario).
  - SQL Database: Use Microsoft.Sql/servers/databases to create a database on the server. A basic or serverless SKU is fine for development (e.g., GP_S_Gen5_2 for GeneralPurpose, serverless). For production, choose appropriate tier.
  - We won’t expose the SQL publicly; ensure allowAzureServices or proper firewall settings if needed (or integrate with private networks in a real scenario).
  - Output the database connection string or at least the necessary info (Server name, DB name) to store as a secret.
- Azure Key Vault: Provision a Key Vault to hold secrets such as the SQL connection string (and potentially other app secrets or certificates). Use SKU Standard. In Bicep:

```bicep
resource kv 'Microsoft.KeyVault/vaults@2022-07-01' = {
  name: keyVaultName
  location: resourceGroup().location
  properties: {
    sku: { name: 'standard', family: 'A' }
    tenantId: tenant().tenantId
    enableSoftDelete: true
    enabledForTemplateDeployment: true  // allow bicep to retrieve secrets if needed
    accessPolicies: []  // We'll use RBAC or separate policy setting below
  }
}
```

We can leave accessPolicies empty to use Azure RBAC for secret management. Alternatively, we can add an access policy for the web and api managed identities here (if not using RBAC roles):

```bicep
accessPolicies: [
  {
    tenantId: tenant().tenantId
    objectId: webContainerApp.identity.principalId
    permissions: { secrets: [ 'get', 'list' ] }
  }
  {
    tenantId: tenant().tenantId
    objectId: apiContainerApp.identity.principalId
    permissions: { secrets: [ 'get', 'list' ] }
  }
]
```

This grants the container apps permission to retrieve secrets from Key Vault. (If using Azure RBAC for Key Vault, you would assign the role "Key Vault Secrets User" to the identities instead.)

- Dependencies and Outputs: Use the main Bicep (infra/main.bicep) to orchestrate module deployments in the correct order:

```bicep
// Example structure in main.bicep (simplified)
targetScope = 'resourceGroup'

@parameter()
param environmentName string  // e.g., "dev"
@parameter()
param location string = resourceGroup().location

// Global resources
module acr './modules/acr.bicep' = {
  name: 'acr'
  params: { location: location, registryName: 'myacr${uniqueString(resourceGroup().id)}' }
}
module log './modules/logAnalytics.bicep' = {
  name: 'log'
  params: { location: location, workspaceName: '${environmentName}-logs' }
}
module kv './modules/keyVault.bicep' = {
  name: 'kv'
  params: { location: location, vaultName: '${environmentName}-kv' }
}
module sql './modules/azureSql.bicep' = {
  name: 'sql'
  params: {
    location: location
    serverName: '${environmentName}-sql'
    adminUser: 'sqladmin'
    adminPassword: '<--use secure param or generate-->'  // use @secure() for real
  }
}
module env './modules/containerEnv.bicep' = {
  name: 'container-env'
  params: {
    location: location
    envName: '${environmentName}-apps-env'
    logsWorkspaceId: log.outputs.workspaceId
    logsPrimaryKey: log.outputs.primaryKey  // from logAnalytics module
  }
}
module apiApp './modules/containerApp.bicep' = {
  name: 'api-app'
  params: {
    name: 'api-service'
    environmentId: env.outputs.envId
    image: '${acr.outputs.loginServer}/api:${param_imageTag}'  // param_imageTag passed in or output from pipeline
    cpu: 0.5
    memory: '1Gi'
    ingress: 'internal'
    vaultName: kv.outputs.vaultName
  }
  dependsOn: [ acr, env, kv, sql ]  // ensure infra is up
}
module webApp './modules/containerApp.bicep' = {
  name: 'web-app'
  params: {
    name: 'web-ui'
    environmentId: env.outputs.envId
    image: '${acr.outputs.loginServer}/web:${param_imageTag}'
    cpu: 0.5
    memory: '1Gi'
    ingress: 'external'  // external ingress for web
    vaultName: kv.outputs.vaultName
  }
  dependsOn: [ acr, env, kv, sql ]
}
```

In the above pseudocode:

- We use modules for each major resource group. The containerApp.bicep module would define a container app, including its registry settings and environment variables.
- We pass the Log Analytics workspace ID and key from the log module to the container environment module (the environment needs the workspace key for diagnostics).
- We ensure that container apps depend on prior resources (so that ACR, Key Vault, etc., exist when the container apps are created).
- The container image name and tag (param_imageTag) is parameterized, allowing us to update the image version on each deployment easily.
- We might output from main the URL of the web frontend (webApp.outputs.url if available) or the FQDN of the container app, so that azd can display it after deployment.

Modular Bicep Design: This approach keeps templates manageable. An example from the community similarly breaks down Container Apps deployments into modules for Log Analytics, Container Env, ACR, etc.. Copilot should generate these Bicep files with proper resource definitions and parameters. Focus on minimal viable config (we can fine-tune SKUs and settings as needed). Use the @secure() decorator for sensitive parameters like passwords so they aren't logged.

- Alternate Modules (Web App / AKS): In the infra/modules directory, include optional Bicep files for alternative deployment targets:
  - webapp.bicep (and maybe appServicePlan.bicep): This would provision an Azure App Service Plan (Linux) and two Web Apps (one for frontend, one for API) configured to run container images from ACR. The Web Apps would also have managed identities for secrets and ACR access. These modules would be used if we switch host: appservice in azure.yaml for example.
  - aks.bicep: This would provision an AKS cluster. A production-grade AKS setup is complex, but for scaffolding, we can create a basic AKS cluster (e.g., with a default node pool). Additionally:
    - Deploy the web and API containers to AKS (likely via Kubernetes manifests or Azure Container Instances). For simplicity, we might not fully automate AKS deployment of apps here, but the Bicep can output credentials or set up a basic internal load balancer service for API and an external ingress for web. It’s noted as an optional path for advanced use.
  - These optional modules are not invoked by default in main.bicep unless the user chooses to. Clearly mark them as alternatives. For instance, comments or a separate section can explain how to swap Azure Container Apps with Web Apps or AKS if needed.

With the infrastructure defined, our deployment will create all necessary Azure resources. Next, we manage application secrets and configuration.

## Secrets Management (Azure Key Vault & Config)

Proper secrets management is vital. We use Azure Key Vault to store sensitive configuration and use managed identities to fetch them at runtime, avoiding hardcoding secrets.

- Storing Secrets in Key Vault: After infrastructure provisioning, we will have certain sensitive values to store:
  - SQL Database Connection String: Construct a connection string for the DB (including server, database, and credentials or Azure AD info). Store this as a secret in Key Vault (e.g., secret name "ConnectionString"). If using SQL admin/password, include those. For example, after Bicep deployment, one could run:

  ```bash
  keyvault secret set --vault-name <Name> --name "ConnectionString" \
  alue "Server=tcp:<sqlserver>.database.windows.net,1433;Database=<dbname>;User   <admin>;Password=<password>;Encrypt=true;"
  ```
  
   (a real pipeline, you might automate this with Azure CLI, or generate password   Bicep and output to KV via reference.)
  - Other App Secrets: If the app needs any API keys or configuration secrets, also put them in Key Vault. During scaffold, you might not have additional secrets, but we include Key Vault for future use.
- Managed Identity Access: Both the Web and API container apps use system-assigned identities. We have given these identities permission to Get secrets from Key Vault (either via accessPolicies or via an RBAC assignment like "Key Vault Secrets User"). This means the running application can call Key Vault to retrieve secrets without any embedded credentials. No secrets are stored in code or in environment variables in plain text; instead, the app will use Azure SDK or REST calls to fetch what it needs at startup. According to Azure’s guidance, using managed identities allows secure access to Key Vault and other resources without managing credentials in the app.
- Application Configuration: To enable the app code to use Key Vault:
  - Use Azure SDK (e.g., Azure.Identity) to acquire a SecretClient with DefaultAzureCredential (which will pick up the managed identity).
  - Fetch the secret by name. For example, in .NET you might use DefaultAzureCredential and SecretClient to get the connection string at startup, then use it to configure the database context.
  - Alternatively, you could inject the secret as an environment variable to the container app by having Bicep reference the Key Vault value at deploy time. For instance, Bicep can do:

  ```bicep
  resource dbConnSecret 'Microsoft.KeyVault/vaults/secrets@2022-07-01' existing = {
    name: 'ConnectionString'
    parent: kv
  }
  // inside containerApp properties:
  configuration: {
    secrets: [
      { name: 'DB_CONN_STR', value: dbConnSecret.getSecret().value }  //   pseudo-code for illustration
    ]
    env: [
      { name: 'ConnectionString', value: secretRef('DB_CONN_STR') }
    ]
  }
  ```

  However, this approach injects the secret at deployment time (and requires Key Vault enabled for template deployment). It might be simpler to let the app pull it at runtime as described above. For scaffolding, mention both possibilities:
  - Option A: Retrieve secrets in code via managed identity (preferred for dynamic secrets or when rotation is a concern).
  - Option B: Inject secrets as environment variables via Bicep (easier initial setup, but the secret value is present in deployment logs – mitigated by using @secure() parameters and Key Vault references).
- Connection Strings & Entity Framework: If using EF Core in the .NET apps, you can configure the connection string after retrieving it from Key Vault. Mark this as TODO in the prompt code if not implementing fully.

By storing secrets in Azure Key Vault and using identities, we adhere to best practices: keys and passwords are not stored in source control or GitHub secrets, and the apps can securely fetch what they need at runtime. Copilot should scaffold code (or at least comments) in the app to illustrate retrieving the connection string.

## CI/CD Pipelines (GitHub Actions)

Set up continuous integration and deployment using GitHub Actions. We will have two workflows: one for build/test and one for deploy. Using OIDC authentication, the workflows will securely deploy to Azure without storing secrets.

### build-test.yml (CI Build Pipeline)

Purpose: On each pull request or push to the main branch (except deployment events), validate the code by running tests, linters, and ensuring the containers build successfully. This pipeline does not deploy or push images. Key steps in build-test.yml:

- Trigger: e.g., on pull_request for main, or pushes to feature branches. This ensures every change is validated.
- Jobs: Use a job running on ubuntu-latest (which has dotnet and docker available).
- Checkout code: Use actions/checkout@v3.
- Set up .NET: Use actions/setup-dotnet@v3 to install the required SDK (e.g., .NET 7 or 8 or 9, matching the project).
- Run tests: Restore and run dotnet test for the solution. Ensure both api and web have tests (if present) run.
- Lint/format (optional): If using linters or formatters, run them.
- Build containers: Do a Docker build for both images to ensure Dockerfiles and app compilation is correct. Tag with a temporary tag (like :ci-test).
  - You can use Docker layer caching if needed, but for simplicity just build to verify. Since we are not pushing in this workflow, we don't need to login to ACR.
  - This step catches any issues in Dockerfile or build process early.
- No push, no deploy: The images built can be discarded after the job.
By having this separate CI workflow, we ensure quality before merging code. It’s faster and isolates build/test from the deployment steps.

## deploy.yml (CD Deploy Pipeline)

Purpose: On push to main (or on a release tag), build and deploy the application to Azure. This will produce container images, push them to ACR, then update the Azure resources via azd/Bicep. Key steps in deploy.yml:

- Trigger: Typically on push to main branch (after PR merge) and/or on version tag. Optionally, protect it behind manual approvals or environment protection rules.
- Permissions: Important: add permissions: { id-token: write, contents: read } at the workflow or job level. This is required to use GitHub’s OIDC token for Azure login.
- Azure Login (OIDC): Use the official Azure login action:

```yaml
- name: Azure Login
  uses: azure/login@v1
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    allow-no-subscriptions: true
```

This action will request an OIDC token from GitHub and exchange it for an Azure access token, logging in as the configured service principal. The credentials (clientId/tenantId/subscriptionId) are not secrets in the traditional sense; they can be stored as repository or organization secrets or in GitHub Environments. Because we set up a federated identity, no password or secret is needed. Using OIDC is more secure and avoids manual secret management.

- Checkout and setup: Similar to CI, check out the code and set up .NET SDK (and any other tool like Azure CLI if needed).
- Build apps: Run dotnet publish or similar to compile the apps (this might be optional if Docker build does it, but it can speed up Docker by having artifacts ready).
- Docker build: Build both web and api images, tagging them with the short SHA or a version:

```bash
docker build -t myregistry.azurecr.io/web:${{ github.sha[:7] }} ./web
docker build -t myregistry.azurecr.io/api:${{ github.sha[:7] }} ./api
```

Ensure the Docker context and paths are correct. Alternatively, use Docker Compose or parallel build jobs.

- Push to ACR: Now push the images:

```bash
az acr login --name <ACR_NAME>
docker push myregistry.azurecr.io/web:${{ github.sha[:7] }}
docker push myregistry.azurecr.io/api:${{ github.sha[:7] }}
```

The azure/login action already logged us into Azure, so az acr login will authenticate using that session (no separate credentials). After this, images are in Azure Container Registry.

- Deploy to Azure: With images available and tagged, deploy the Azure resources with the updated image references. We have a couple of options:
  - Using azd: Since this project is azd-enabled, we could run azd deploy which will apply any infra changes and deploy the app code. However, for container apps, azd deploy might attempt to build/push images itself if it detects changes. To avoid double-building, we can invoke infrastructure deployment directly.
  - Bicep deployment: Run an Azure CLI deployment command to update the container apps with new image tags. For example:

  ```yaml
  - name: Deploy Bicep (update container images)
    uses: azure/cli@v1
    with:
      azcliversion: 2.50.0
      scriptType: bash
      script: |
        az deployment group create \
          --resource-group $AZURE_RG \
          --template-file infra/main.bicep \
          --parameters imageTag='${{ github.sha[:7] }}' environmentName='prod'
  ```

  Here we pass the new imageTag parameter (and any other required params like environmentName, etc.). This will update the Azure Container Apps to use the newly pushed image tags. Since the container app definitions reference the images by tag, this triggers a new revision deployment.
  - Output the Web URL: After deployment, optionally use Azure CLI or azd to fetch the frontend URL. E.g., az containerapp show -n web-ui -g $AZURE_RG --query properties.configuration.ingress.fqdn. Azd might output this automatically. We can log it or set it as an output of the job for visibility.
- Post-deployment tests (optional): Optionally, once deployed, run a small script to test that the web app is responding (and maybe that API is reachable). This could be a simple curl against the health endpoint.

By separating build and deploy, we ensure clarity in the pipeline. The use of OIDC means we did not use any stored Azure secrets – the GitHub Action obtained a token for the Azure service principal on-the-fly. In our pipelines, no manual secret for Azure credentials is needed, aligning with modern security practices.

## Deployment Prep Script (OIDC and Resource Setup)

To use OIDC and to prepare Azure resources for the first deployment, consider a one-time setup script. This is not part of the automated pipeline, but a manual step to configure Azure and GitHub: Script: scripts/setup-oidc.sh (for example):

```bash
#!/usr/bin/env bash
# Variables (edit these)
RESOURCE_GROUP="my-azd-app-rg"
LOCATION="eastus"
SUBSCRIPTION_ID="<your subscription>"

# 1. Create resource group (if not already created by azd)
az group create -n $RESOURCE_GROUP -l $LOCATION

# 2. Pre-create Key Vault and ACR (optional, azd/bicep can do this too)
az acr create -n "<ACR_NAME>" -g $RESOURCE_GROUP --sku Basic
az keyvault create -n "<KeyVaultName>" -g $RESOURCE_GROUP --location $LOCATION --enable-rbac-authorization true

# 3. Create Azure AD app for GitHub Actions OIDC
APP_NAME="GithubOIDC-MyApp"
az ad app create --display-name "$APP_NAME" --allow-public-client false
APP_ID=$(az ad app list --display-name "$APP_NAME" --query "[0].appId" -o tsv)
# Create service principal for the app
az ad sp create --id $APP_ID
# Assign role (e.g., contributor on the resource group) to the service principal
az role assignment create --assignee $APP_ID --role Contributor --scope /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP

# 4. Add federated credentials to the AAD app (replace org/repo and workflow details accordingly)
az ad app federated-credential create --id $APP_ID --parameters '{
  "name": "GitHubActionsOIDC",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<GITHUB_ORG>/<REPO>:ref:refs/heads/main",
  "description": "OIDC federated credential for GitHub Actions (main branch)",
  "audiences": ["api://AzureADTokenExchange"]
}'
echo "Client ID: $APP_ID"
```

Explanation:

- We ensure a resource group exists.
- (Optional) We create ACR and Key Vault ahead of time. This can speed up first deployment and ensure those resources exist for pushing images (especially if our first pipeline run needs them). Our Bicep will detect they exist and skip or update as needed (to avoid conflicts, you might skip this and let Bicep handle everything).
- We register an Azure AD application that will represent our GitHub Actions. We then create a service principal and give it Contributor rights on the resource group (scope can be narrowed further if needed, e.g., only certain resources).
- We then configure a federated credential on the AD app to trust GitHub OIDC tokens from our repository. In the above JSON, adjust the subject to match your repo and branch or environment. This example trusts any workflow run on the main branch of GITHUB_ORG/REPO. You could also set repo:<org>/<repo>:pull_request or target specific environments. This setup corresponds to what GitHub’s docs describe for Azure federated identity.
- After running this script, note the Client ID (the app’s ID). That will be used as the AZURE_CLIENT_ID in GitHub Actions. The tenant ID and subscription ID are known to you. These can be stored as GitHub Actions secrets or directly in the workflow (since Client ID and Tenant ID are not highly sensitive by themselves).

This script need only be run once. After that, GitHub Actions should be able to login via OIDC (as long as the workflow JWT meets the conditions). The Azure Developer CLI (azd pipeline config) can also automate some of this (it creates a service principal and sets up OIDC by default), but the manual approach gives us visibility and control.

## Deployment options (Optional Variations)

- Azure Web Apps (App Service) Alternative: Instead of container apps, you can deploy the Web and API to Azure App Service (Web Apps for Containers).
  - You would need an App Service Plan (Linux) resource, and two Web App resources. Each Web App can be configured to pull the container image from ACR. In Bicep, Microsoft.Web/sites resource with siteConfig specifying the image and registry. For example, set containerSettings with imageName, registryUrl, and enable managed identity to access ACR (the Web App's identity can also be granted AcrPull on ACR).
  - Azure Developer CLI supports host: appservice. If using azd, simply changing the host in azure.yaml and providing an appropriate Bicep module for App Service could enable this path. The Key Vault integration remains similar (Web App can use its identity for Key Vault as well).
  - Note: App Service provides built-in integration for Key Vault references in app settings, which is another way to load secrets. But using managed identity to fetch secrets at runtime is still a good approach.
  - We include a infra/modules/webapp.bicep as a reference implementation. If switching, disable containerapp modules and enable the webapp ones.
- Azure Kubernetes Service (AKS) Alternative: For scenarios requiring full control over Kubernetes:
  - Provision an AKS cluster (e.g., with a default nodepool). This can be done via Bicep (Microsoft.ContainerService/managedClusters).
  - Container deployment to AKS can be handled via Kubernetes manifests or Helm charts. In an azd context, host: aks implies you might supply Kubernetes deployment YAMLs in the project that azd will apply. Copilot can scaffold simple deployment and service YAMLs for the web and api.
    - For example, a Deployment for the web app with a LoadBalancer Service for external access, and a Deployment for the API with an Internal load balancer Service (or no Service if only accessed by web within cluster).
    - Optionally use an Ingress controller (e.g., NGINX or Azure Application Gateway) to route traffic, especially if using a single endpoint.
  - The container images would still come from ACR, so you'd grant the AKS cluster’s kubelet identity pull access to ACR (if using managed identity, Azure can do this automatically when linking ACR).
  0 Managed Identity in AKS: you can use a user-assigned identity with Azure AD Workload Identity or the older Pod Identity to allow pods to access Key Vault. This is more complex, so in a simple scaffold one might use Kubernetes secrets (populated with values from Key Vault via pipeline) or mount an Azure Key Vault FlexVolume/CSIDriver. This goes beyond scope, but worth noting.
  - In our Bicep optional module, we could output the kubeconfig or credentials for reference. The actual app deployment to AKS might be outside Bicep (applied via kubectl in pipeline).
  - Important: AKS brings significant complexity (cluster operations, networking). Use it only if needed for advanced scenarios. Azure Container Apps often suffices for containerized web/API with minimal ops overhead (Container Apps are serverless and handle many concerns automatically).
- AI Services (Optional): If the app needs Azure AI services (Cognitive Search, etc.), those could be added as additional modules in infra and accessed via identity. Not in scope for now, but the template can be extended.

The prompt instructs Copilot to scaffold code and config primarily for Container Apps. The alternative sections (Web Apps, AKS) should be generated as commented-out or separate files, not active unless chosen. Clearly label these sections in the output as optional.

## Database Migration and Initialization

Finally, address how to set up the database schema:

- Entity Framework Core Migrations: Since we have a .NET backend with a SQL database, we likely use EF Core Code-First migrations or a DACPAC for schema.
  - If using EF Core: You can include a step in the deployment to apply migrations. For example, after deploying the containers, run dotnet ef database update or have the API execute context.Database.Migrate() on startup. Running migrations at startup can be convenient for development, but use caution in production (consider a dedicated migration step or tool).
  - Startup Migration Script: Copilot could scaffold code in the API Program.cs to detect if the DB is empty and run Migrate(). Ensure that the connection string is loaded (from Key Vault as discussed). Wrap it in try/catch and logs.
  - Alternatively, generate SQL migration scripts (dotnet ef migrations script) and apply them out-of-band. This could be a separate job in the pipeline that runs an Azure CLI script or SQL command. Simplicity favors the automatic approach for now.
  - Sample:

   ```cs
   using var scope = app.Services.CreateScope();
   var db = scope.ServiceProvider.GetRequiredService<MyDbContext>();
   db.Database.Migrate();
   ```

   This will apply pending migrations on app start.
- Seed Data (if any): If the app requires some seed data, you can similarly perform seeding after migration in the startup.
- Connection resiliency: Ensure the connection string has Encrypt=true;TrustServerCertificate=false; and possibly Connection Timeout set appropriately. Azure SQL might need a moment to be accessible if just created, so the app might retry on first connection.
- Credentials for DB: We used an admin user for simplicity. In production, you might create a dedicated DB user with limited permissions. As an advanced step, one can enable Managed Identity authentication for the DB:
  - This involves setting an Azure AD admin on the SQL server (could be the same service principal used by GitHub or a separate AAD user).
  - Then create a user in the database mapped to the web or api's managed identity (using CREATE USER [<identity-name>] FROM EXTERNAL PROVIDER in SQL).
  - And grant that user db_datareader/db_datawriter or appropriate roles. Then the app could use Active Directory Default auth mode (no password) to connect. This avoids storing a DB password at all. This is a more complex setup but worth mentioning as a future improvement for full passwordless deployment.
  - For now, using the admin/password in a secret is acceptable for a scaffold, with a TODO to implement AAD auth later.

## Conclusion

The prompt above is organized into steps that GitHub Copilot can follow to scaffold the entire project. It provides a comprehensive Azure-based architecture:

- Azure Container Apps for hosting, offering a fully managed serverless container experience (with autoscaling, ingress management, and microservices features out-of-the-box).
- Infrastructure-as-Code with Bicep, broken into logical modules for readability.
- Secure secret management with Key Vault and Managed Identities, so no credentials are exposed in code or repo.
- Continuous deployment using GitHub Actions with OIDC, eliminating the need for stored cloud credentials.
- Alternative deployment options (App Service, AKS) are noted for flexibility.

Using this structured prompt, Copilot (in Agent Mode) should be able to generate a production-ready Azure Developer CLI template repository that implements this architecture following best practices. All necessary files (Dockerfiles, Bicep templates, Azure configurations, CI/CD YAMLs, and basic .NET app code) will be scaffolded, after which you can customize the application logic and settings as needed.
