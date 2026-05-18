# CloudNativeInventory API

A .NET 9 Web API for inventory management, built to demonstrate containerization, CI/CD, and secure configuration management in Azure.

## Azure Services

| Service | Usage |
|---|---|
| Azure Container Registry | Stores Docker images privately |
| Azure Container Apps | Runs the application in production |
| Azure Key Vault | Stores secrets securely |
| Log Analytics Workspace | Logging and monitoring |

For the reasoning behind these choices, see [ADR-001](adr-001-cloud-architecture.md).

Full documentation is also available here: [Documentation](K5U1%20Documentation.md).

---

## Running the API Locally

### Prerequisites
- .NET 9 SDK
- Docker Desktop

### Steps

1. Clone the repo:
```bash
git clone https://github.com/Melle1010/CloudNativeInventory.git
cd CloudNativeInventory
```

2. Run the API:
```bash
dotnet run --project CloudNativeInventory.Api
```

3. Verify it's working:
```
GET http://localhost:5000/api/inventory
```

You should receive a JSON list containing a laptop and an IPhone.

### Secrets Locally

Locally, the application reads `VendorApiKey` from `appsettings.json`. That value is a placeholder and is only used during development — it is never a real key. In production, the key is fetched from Azure Key Vault via Managed Identity, and `appsettings.json` is not used.

### Running with Docker Locally

Build and start the container:
```bash
docker build -t inventory-api -f CloudNativeInventory.Api/Dockerfile CloudNativeInventory.Api
docker run -p 8080:8080 inventory-api
```

Test:
```
GET http://localhost:8080/api/inventory
```

---

## CI/CD Pipeline

The pipeline is defined in `.github/workflows/ci.yml` and consists of two jobs.

### What Triggers the Pipeline

| Event | CI (build + test) | CD (deploy) |
|---|---|---|
| Push to master | ✅ | ✅ |
| Pull request against master | ✅ | ❌ |

Deploy only runs when code is merged to master — never on open pull requests.

### Steps in the CI Job (build-and-test)

1. Check out the code
2. Set up .NET 9
3. `dotnet restore` – fetches NuGet packages
4. `dotnet build` – compiles the project
5. `dotnet test` – runs all tests in `CloudNativeInventory.Tests`

If any step fails, the pipeline stops and deploy does not run.

### Steps in the CD Job (deploy)

1. Check out the code
2. Log in to Azure using `AZURE_CREDENTIALS` (GitHub Secret)
3. Log in to Azure Container Registry
4. Build Docker image and tag with commit SHA and `latest`
5. Push the image to ACR
6. Update the Container App with the new image

Every deploy is tagged with the commit SHA (`github.sha`) for full traceability — you can always see exactly which code is running in production.

### Required GitHub Secrets

| Secret | Description |
|---|---|
| `AZURE_CREDENTIALS` | JSON credentials for the Azure Service Principal |

---

## Deploy and Verification

### Automatic Deploy

Deploy happens automatically when you push or merge to `master`. No manual action is required.

### Manual Deploy (if needed)

```bash
az acr login --name melvin

docker build \
  -t melvin.azurecr.io/inventory-api:latest \
  -f CloudNativeInventory.Api/Dockerfile \
  CloudNativeInventory.Api

docker push melvin.azurecr.io/inventory-api:latest

az containerapp update \
  --name inventory-container-app \
  --resource-group inventory-rg \
  --image melvin.azurecr.io/inventory-api:latest
```

### Verifying Key Vault in Production

Once the app is deployed, test this endpoint:

```
GET https://<your-app-url>.azurecontainerapps.io/api/inventory/system/verify-integration
```

Expected response when correctly configured:
```json
{
  "status": "Secured",
  "message": "Secret was loaded successfully via secure configuration."
}
```

If you instead get `"status": "Unsecured"`, the app cannot reach Key Vault — verify that Managed Identity is enabled and that it has the correct role in Key Vault.

### Other Endpoints

```
GET /api/inventory                            Fetches all products
GET /api/inventory/system/verify-integration  Verifies Key Vault integration
```

---

## Architecture Decisions

See [docs/adr-001-cloud-architecture.md](docs/adr-001-cloud-architecture.md) for the full rationale behind:

- Choosing Azure Container Apps over App Service
- Choosing Azure Container Registry over Docker Hub
- Choosing Key Vault with Managed Identity over hardcoded secrets
- Choosing GitHub Actions for CI/CD
