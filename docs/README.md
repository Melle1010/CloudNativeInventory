# CloudNativeInventory API

Ett .NET 9 Web API för lagerhantering, byggt för att demonstrera containerisering, CI/CD och säker konfigurationshantering i Azure.

## Azure-tjänster

| Tjänst | Användning |
|---|---|
| Azure Container Registry | Lagrar Docker-images privat |
| Azure Container Apps | Kör applikationen i produktion |
| Azure Key Vault | Lagrar hemligheter säkert |
| Log Analytics Workspace | Loggning och övervakning |

För motivering av dessa val, se [docs/adr-001-cloud-architecture.md](docs/adr-001-cloud-architecture.md).

Här är min dokumentation också [docs/K5U1%20Documentation.md](docs/K5U1%20Documentation.md).
---

## Köra API:t lokalt

### Förutsättningar
- .NET 9 SDK
- Docker Desktop

### Steg

1. Klona repot:
```bash
git clone https://github.com/Melle1010/CloudNativeInventory.git
cd CloudNativeInventory
```

2. Kör API:t:
```bash
dotnet run --project CloudNativeInventory.Api
```

3. Testa att det fungerar:
```
GET http://localhost:5000/api/inventory
```

Du ska få tillbaka en JSON-lista med en laptop.

### Hemligheter lokalt

Lokalt läser applikationen `VendorApiKey` från `appsettings.json`. Det värdet är en platshållare och används bara under utveckling – det är aldrig en riktig nyckel. I produktion hämtas nyckeln från Azure Key Vault via Managed Identity, och `appsettings.json` används inte.

Checka aldrig in riktiga hemligheter i `appsettings.json`.

### Köra med Docker lokalt

Bygg och starta containern:
```bash
docker build -t inventory-api -f CloudNativeInventory.Api/Dockerfile CloudNativeInventory.Api
docker run -p 8080:8080 inventory-api
```

Testa:
```
GET http://localhost:8080/api/inventory
```

---

## CI/CD-pipeline

Pipelinen finns i `.github/workflows/ci.yml` och består av två jobb.

### Vad som triggar pipelinen

| Händelse | CI (build + test) | CD (deploy) |
|---|---|---|
| Push till master | ✅ | ✅ |
| Pull request mot master | ✅ | ❌ |

Deploy körs alltså bara när kod mergas till master – aldrig vid öppna pull requests.

### Steg i CI-jobbet (build-and-test)

1. Checka ut koden
2. Sätt upp .NET 9
3. `dotnet restore` – hämtar NuGet-paket
4. `dotnet build` – kompilerar projektet
5. `dotnet test` – kör alla tester i `CloudNativeInventory.Tests`

Om något steg misslyckas stannar pipelinen och deploy körs inte.

### Steg i CD-jobbet (deploy)

1. Checka ut koden
2. Logga in i Azure med `AZURE_CREDENTIALS` (GitHub Secret)
3. Logga in i Azure Container Registry
4. Bygg Docker-image och tagga med commit-SHA och `latest`
5. Pusha imagen till ACR
6. Uppdatera Container App med den nya imagen

Varje deploy taggas med commit-SHA (`github.sha`) för full spårbarhet – du kan alltid se exakt vilken kod som kör i produktion.

### GitHub Secrets som krävs

| Secret | Beskrivning |
|---|---|
| `AZURE_CREDENTIALS` | JSON-credentials för Azure Service Principal |

---

## Deploy och verifiering

### Automatisk deploy

Deploy sker automatiskt när du pushar eller mergar till `master`. Du behöver inte göra något manuellt.

### Manuell deploy (om det behövs)

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

### Verifiera att Key Vault fungerar i produktion

När appen är deployad, testa denna endpoint:

```
GET https://<din-app-url>.azurecontainerapps.io/api/inventory/system/verify-integration
```

Förväntat svar vid korrekt konfiguration:
```json
{
  "status": "Secured",
  "message": "Hemlighet laddades framgångsrikt via säker konfiguration."
}
```

Om du istället får `"status": "Unsecured"` betyder det att appen inte kan nå Key Vault – kontrollera att Managed Identity är aktiverad och att den har rätt roll i Key Vault.

### Övriga endpoints

```
GET /api/inventory                        Hämtar alla produkter
GET /api/inventory/system/verify-integration  Verifierar Key Vault-integration
```

---

## Arkitekturbeslut

Se [docs/adr-001-cloud-architecture.md](docs/adr-001-cloud-architecture.md) för fullständig motivering av:

- Val av Azure Container Apps över App Service
- Val av Azure Container Registry över Docker Hub
- Val av Key Vault med Managed Identity över hårdkodade hemligheter
- Val av GitHub Actions för CI/CD
