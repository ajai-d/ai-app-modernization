# Infrastructure as Code generation and deployment

In this example, we will use GitHub Copilot agent mode to generate Infrastructure as Code artifacts (Bicep) for a .NET application and deploy to Azure.

## Setup

- Open GitHub Copilot (Ctrl-I) and select Agent mode.
- Open the [.github/prompts/deployment.prompt.md](.github/prompts/deployment.prompt.md) file
- Tell agent mode this: Review my app code and create infrastructure as code artifacts for deployment. Then deploy it to Azure following the prompt instructions in deployment.prompt.md.