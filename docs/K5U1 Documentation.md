# K5U1 Documentation
Melvin Danielsson

## May 12:
* I copied the code from the assignment into my code.
* I created a repo.
* I posted a ruleset.
* I created a ci.yml file for CI Pipeline

## May 14:
* I updated my ci.yml file because it had some errors.
* Tried merging a branch with failed tests and it wasn't possible to merge.
* I looked into how the API key works in the code.
* I realised if I change the API key value in `appsettings.json` I actually get a message that it's "secure" even though it really isn't.
* I realised that it says secure because the logic for checking whether it's secure or not seems quite lazy.

## May 16:
* A rootless container reduces the attack surface because it ensures you have exactly as much permission as needed and nothing more.
* Created a container registry in Azure.
* I chose Azure Container Apps because it is better suited for this specific use case.
* I got Azure Container Registry fully working — you can now access the app via Azure and it works.
* Got Key Vault working.
* I assigned roles to myself and to the Container App.
* Got Managed Identity working and now understand what it means.

## May 17:
* Created a CD pipeline as well, so that it also handles deployment.
* Created a docs folder with an ADR file.

## May 18:
* FINALLY got the CI/CD pipeline working (dopamine rush).
* Tested my pipeline and think it's really cool that it automatically deploys the code to Azure.
