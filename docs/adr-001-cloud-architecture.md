# ADR-001: Molnarkitektur och säkerhet för CloudNativeInventory API

**Datum:** 2026-05-17  
**Status:** Godkänd  
**Författare:** Melvin Danielsson

---

## Kontext

CloudNativeInventory API är en .NET 9-applikation som hanterar lagerdata och integrerar mot ett externt system via en hemlig API-nyckel (`VendorApiKey`). Applikationen behövde göras produktionsnära genom containerisering, CI/CD och säker konfigurationshantering i Azure.

Tre centrala beslut behövde fattas:
1. Vilken Azure-tjänst ska hosta containern?
2. Hur ska Docker-images lagras och distribueras?
3. Hur ska hemligheter hanteras säkert?

---

## Beslut 1: Azure Container Apps (istället för Azure App Service)

### Alternativ som övervägdes
- **Azure App Service** – Microsofts traditionella plattform för webbapplikationer med stöd för containers.
- **Azure Container Apps** – En container-nativ plattform byggd specifikt för containeriserade arbetsbelastningar.

### Beslut
Jag valde **Azure Container Apps**.

### Motivering
Azure Container Apps är byggt från grunden för containers och erbjuder en enklare och mer naturlig integration med Azure Container Registry via Managed Identity, utan behov av extra konfiguration. Plattformen har en generös gratistier baserad på faktisk förbrukning (Consumption-plan), vilket innebär låg kostnad för ett projekt med sporadisk trafik. App Service kräver mer konfiguration för containers och är mer lämpat för traditionella webbapplikationer. Container Apps stödjer också direkt integration med Managed Identity för Key Vault-åtkomst, vilket är centralt för säkerhetsarkitekturen i detta projekt.

### Konsekvenser
- Enklare deployment-pipeline eftersom Azure Container Apps nativt förstår container-images.
- Inbyggd Log Analytics workspace skapas automatiskt, vilket ger loggning utan extra konfiguration.
- Skalning till noll när appen inte används, vilket minimerar kostnader.

---

## Beslut 2: Azure Container Registry (istället för Docker Hub)

### Alternativ som övervägdes
- **Docker Hub** – Den mest välkända publika container-registret.
- **Azure Container Registry (ACR)** – Microsofts privata container-register, integrerat med Azure-ekosystemet.

### Beslut
Jag valde **Azure Container Registry** med Basic SKU.

### Motivering
ACR är ett privat register vilket innebär att Docker-images inte är publikt tillgängliga, till skillnad från Docker Hub där images är publika som standard. Viktigare är att ACR integrerar direkt med Azure Managed Identity – Container Apps kan hämta images från ACR utan lösenord eller lagrade credentials. Docker Hub skulle kräva att credentials lagrades som hemligheter i pipeline, vilket ökar attackytan. Basic SKU räcker för projektets behov (en image, låg trafik) och kostar minimalt.

### Konsekvenser
- Images är privata och åtkomst styrs via Azure RBAC.
- Ingen manuell lösenordshantering krävs för image-pulls i produktion.
- CI/CD-pipelinen använder `az acr build` för att bygga och pusha images direkt i Azure, utan att exponera registry-credentials i pipeline-loggar.

---

## Beslut 3: Azure Key Vault med Managed Identity (istället för hårdkodade hemligheter)

### Alternativ som övervägdes
- **Hårdkodade hemligheter i appsettings.json** – Enkelt men extremt osäkert; hemligheter hamnar i versionshantering.
- **Environment variables i Container App** – Bättre än hårdkodning men hemligheter syns i Azure Portal i klartext.
- **Azure Key Vault med Managed Identity** – Hemligheter lagras krypterat i ett dedikerat valv och hämtas utan lösenord via identitetsbaserad åtkomst.

### Beslut
Jag valde **Azure Key Vault med System-Assigned Managed Identity**.

### Motivering
Hårdkodade hemligheter i kod eller versionshanterade filer är en av de vanligaste och allvarligaste säkerhetsbristerna (OWASP Top 10). Key Vault separerar hemligheter helt från koden och versionhanteringen. Med Managed Identity behövs inga lagrade credentials överhuvudtaget – Azure hanterar autentiseringen internt. Detta implementerar principen om minsta behörighet: Container App:en har tilldelats rollen **Key Vault Secrets Officer** vilket ger den tillgång att läsa hemligheter, men inte administrera Key Vault eller andra Azure-resurser.

I `Program.cs` används `DefaultAzureCredential` som automatiskt identifierar och använder Managed Identity i produktionsmiljön, medan lokala utvecklare fortsätter använda `appsettings.json` utan förändring.

### Konsekvenser
- `VendorApiKey` finns aldrig i klartext i kod, versionshanterade filer eller pipeline-loggar.
- Rotering av hemligheter kan göras i Key Vault utan att ändra eller redeployera applikationen.
- Åtkomst kan spåras och auditeras via Key Vault-loggar.
- Principen om minsta behörighet uppfylls: appen kan bara läsa hemligheter, inte modifiera dem.

---

## Beslut 4: GitHub Actions för CI/CD

### Motivering
GitHub Actions är direkt integrerat med GitHub-repot och kräver ingen separat CI/CD-server. Pipelinen kör `dotnet restore`, `dotnet build` och `dotnet test` vid varje push och pull request, vilket säkerställer att trasig kod aldrig når produktion. Vid godkända tester på main/master-branchen byggs en ny Docker-image, taggas med commit-SHA för spårbarhet, och deployar automatiskt till Container Apps. Commit-SHA som image-tagg innebär att varje deployment är direkt spårbar till en specifik kodändring.

---

## Sammanfattning av infrastrukturbeslut

| Komponent | Val | Alternativ | Huvudskäl |
|---|---|---|---|
| Hosting | Azure Container Apps | App Service | Container-nativt, kostnad, Managed Identity |
| Registry | Azure Container Registry | Docker Hub | Privat, lösenordslös integration |
| Hemligheter | Azure Key Vault | Environment variables | Separation, kryptering, minsta behörighet |
| CI/CD | GitHub Actions | Azure DevOps | Direkt GitHub-integration, enkelt |
| Auth-modell | Managed Identity | Service Principal | Inga lagrade credentials |
