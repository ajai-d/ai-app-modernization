---
mode: 'agent'
---

Create Bicep artifacts for deploying the example code to Azure.

Structure the Bicep files to deploy the code in a way that is easy to understand and maintain.

If possible, use Platform as a Service (PaaS) resources such as Azure Container Apps to deploy the code.

Use rg-ShipAppsFaster as the resource group name.

Use East US, East US 2, West US, or West US 2 as the region if possible. 

Create an Azure container registry to store the app image.

Create a Dockerfile for the .NET app that builds the app and runs it in a container.

Build and push the .NET app container image to the Azure Container Registry.

Use the Bicep file to deploy the app, SQL database, and other necessary resources.

Ensure the Bicep file is modular and reusable for different environments.

Run test deployments in the region to ensure the code works as expected.

Check for Azure quota in a given region before deploying.

Build the container image, push it to Azure Container Registry, and deploy the app using the Bicep file without asking the user to input any parameters.

Ensure the deployment is fully automated and does not require manual intervention.