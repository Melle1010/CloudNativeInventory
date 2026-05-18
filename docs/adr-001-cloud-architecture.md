# ADR-001: Cloud Architecture and Security for CloudNativeInventory API

**Date:** 2026-05-17  
**Status:** Approved  
**Author:** Melvin Danielsson

---

## Context

CloudNativeInventory API is a .NET 9 application that manages inventory data and integrates with an external system via a secret API key (`VendorApiKey`). The application needed to be brought closer to production-ready through containerization, CI/CD, and secure configuration management in Azure.

Three key decisions needed to be made:
1. Which Azure service should host the container?
2. How should Docker images be stored and distributed?
3. How should secrets be managed securely?

---

## Decision 1: Azure Container Apps (instead of Azure App Service)

### Alternatives Considered
- **Azure App Service** – Microsoft's traditional platform for web applications with container support.
- **Azure Container Apps** – A container-native platform built specifically for containerized workloads.

### Decision
I chose **Azure Container Apps**.

### Rationale
Azure Container Apps is built from the ground up for containers and offers simpler, more natural integration with Azure Container Registry via Managed Identity, without any extra configuration. The platform has a generous free tier based on actual consumption (Consumption plan), which means low cost for a project with sporadic traffic. App Service requires more configuration for containers and is better suited for traditional web applications. Container Apps also supports direct integration with Managed Identity for Key Vault access, which is central to the security architecture of this project.

### Consequences
- Simpler deployment pipeline since Azure Container Apps natively understands container images.
- A built-in Log Analytics workspace is created automatically, providing logging without additional configuration.
- Scales to zero when the app is not in use, minimizing costs.

---

## Decision 2: Azure Container Registry (instead of Docker Hub)

### Alternatives Considered
- **Docker Hub** – The most well-known public container registry.
- **Azure Container Registry (ACR)** – Microsoft's private container registry, integrated with the Azure ecosystem.

### Decision
I chose **Azure Container Registry** with the Basic SKU.

### Rationale
ACR is a private registry, meaning Docker images are not publicly accessible — unlike Docker Hub, where images are public by default. More importantly, ACR integrates directly with Azure Managed Identity, allowing Container Apps to pull images from ACR without passwords or stored credentials. Docker Hub would require credentials to be stored as secrets in the pipeline, increasing the attack surface. The Basic SKU is sufficient for this project's needs (one image, low traffic) and costs very little.

### Consequences
- Images are private and access is controlled via Azure RBAC.
- No manual password management is required for image pulls in production.
- The CI/CD pipeline uses `az acr build` to build and push images directly in Azure, without exposing registry credentials in pipeline logs.

---

## Decision 3: Azure Key Vault with Managed Identity (instead of hardcoded secrets)

### Alternatives Considered
- **Hardcoded secrets in appsettings.json** – Simple but extremely insecure; secrets end up in version control.
- **Environment variables in the Container App** – Better than hardcoding, but secrets are visible in the Azure Portal in plain text.
- **Azure Key Vault with Managed Identity** – Secrets are stored encrypted in a dedicated vault and retrieved without passwords via identity-based access.

### Decision
I chose **Azure Key Vault with System-Assigned Managed Identity**.

### Rationale
Hardcoded secrets in code or version-controlled files are one of the most common and serious security vulnerabilities (OWASP Top 10). Key Vault separates secrets entirely from code and version control. With Managed Identity, no stored credentials are needed at all — Azure handles authentication internally. This implements the principle of least privilege: the Container App has been assigned the **Key Vault Secrets Officer** role, giving it access to read secrets but not to administer the Key Vault or other Azure resources.

In `Program.cs`, `DefaultAzureCredential` is used, which automatically detects and uses Managed Identity in the production environment, while local developers continue using `appsettings.json` without any changes.

### Consequences
- `VendorApiKey` is never present in plain text in code, version-controlled files, or pipeline logs.
- Secret rotation can be done in Key Vault without modifying or redeploying the application.
- Access can be tracked and audited via Key Vault logs.
- The principle of least privilege is met: the app can only read secrets, not modify them.

---

## Decision 4: GitHub Actions for CI/CD

### Rationale
GitHub Actions is directly integrated with the GitHub repository and requires no separate CI/CD server. The pipeline runs `dotnet restore`, `dotnet build`, and `dotnet test` on every push and pull request, ensuring that broken code never reaches production. When tests pass on the main/master branch, a new Docker image is built, tagged with the commit SHA for traceability, and automatically deployed to Container Apps. Using the commit SHA as the image tag means every deployment is directly traceable to a specific code change.

---

## Summary of Infrastructure Decisions

| Component | Choice | Alternative | Main Reason |
|---|---|---|---|
| Hosting | Azure Container Apps | App Service | Container-native, cost, Managed Identity |
| Registry | Azure Container Registry | Docker Hub | Private, passwordless integration |
| Secrets | Azure Key Vault | Environment variables | Separation, encryption, least privilege |
| CI/CD | GitHub Actions | Azure DevOps | Direct GitHub integration, simplicity |
| Auth model | Managed Identity | Service Principal | No stored credentials |
