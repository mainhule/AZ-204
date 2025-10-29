# Azure Container Registry (ACR)

Een privé Docker registry voor het opslaan en beheren van container images.

**Endpoint formaat:** `<registry>.azurecr.io/<repository>/<image>:<tag>`

**SKU's:**
- Basic: 10GB
- Standard: 100GB (⭐ Productie)
- Premium: 500GB + extra features

**Authenticatie opties:**
1. Individueel (Entra ID)
2. Service Principal
3. Managed Identity
4. Admin User (niet aanbevolen)

**ACR Tasks:**
- Quick task: build & push
- Automated task: triggers op Git commits
- Multi-step task: complexe workflows

**Belangrijkste CLI commando's:**
```bash
az acr create --name myregistry --sku Standard
az acr login --name myregistry
az acr build --registry myregistry --image myapp:v1 .
```
