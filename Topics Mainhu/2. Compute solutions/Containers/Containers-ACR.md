# Azure Container Registry (ACR)

Een privé Docker registry voor het opslaan en beheren van container images.

## Endpoint Formaat

```
<registry>.azurecr.io/<repository>/<image-or-artifact>:<tag>
```

**Repository** (ook bekend als namespace):
- Maakt het mogelijk om één registry te delen tussen meerdere groepen in je organisatie
- Kan meerdere levels diep zijn
- Optioneel

---

## SKU's

### 📦 Basic
- **Storage:** 10GB
- **Best voor:** Development en testing

### 🏢 Standard (⭐ Productie)
- **Storage:** 100GB
- **Best voor:** Productie workloads

### 💎 Premium
- **Storage:** 500GB
- **Extra features:**
  - Concurrent operations
  - High volumes (⚡)
  - Customer-Managed Key
  - Content trust voor image tag signing
  - Private link
  - Zone redundancy (min 3 zones per regio)
  - Default action (Allow/Deny) wanneer geen rules van toepassing

### Throttling

⚠️ Kan gebeuren als je de registry's limieten overschrijdt, wat tijdelijke `HTTP 429` errors veroorzaakt en retry logic of het verlagen van de request rate vereist.

⚠️ **Let op:** Hoge aantallen repositories en tags kunnen de performance beïnvloeden. Verwijder periodiek ongebruikte items.

---

## Authenticatie

### Opties Overzicht

| Type | Gebruik | Authenticatie Methode |
|------|---------|----------------------|
| **Interactive** | Developers, testers | Individual Entra ID login, Admin Account |
| **Unattended/Headless** | CI/CD pipelines | Entra ID Service Principal, Managed Identity |

### 1️⃣ Individual Login met Entra ID

**Voor:** Interactive push/pull door developers en testers

```bash
# az login levert de token - moet elke 3 uur vernieuwd worden
az acr login --name myregistry
```

⏰ **Token moet elke 3 uur vernieuwd worden**

---

### 2️⃣ Entra ID Service Principal

**Voor:** Unattended push/pull in CI/CD pipelines

#### Methode 1: Short version
```bash
az ad sp create-for-rbac \
    --name $ServicePrincipalName \
    --role AcrPush,AcrPull,AcrDelete \
    --scopes /subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.ContainerRegistry/registries/$registryName
```
Retourneert appId en password in JSON formaat.

#### Methode 2: Service principal en roles apart configureren
```bash
# Service principal maken
az ad sp create --id $ServicePrincipalName

# Role assignment maken
az role assignment create \
    --assignee $appId \
    --role AcrPush,AcrPull,AcrDelete \
    --scope /subscriptions/$subscriptionId/resourceGroups/$resourceGroup/providers/Microsoft.ContainerRegistry/registries/$registryName

# Password aanmaken (voor methode 2 wordt password niet expliciet aangemaakt)
az ad sp credential reset --name $appId

# Inloggen
az acr login --name $registryName --username $appId --password $password
```

⚠️ **Password verloopt na 1 jaar**

---

### 3️⃣ Managed Identities

**Voor:** Container instances / apps

```bash
az role assignment create \
    --assignee $managedIdentityId \
    --scope $registryName \
    --role AcrPush,AcrPull,AcrDelete
```

Container instances/apps moeten nu die managed identity gebruiken om deze ACR te benaderen (pull of push images).

---

### 4️⃣ Admin User ❌

**Niet aanbevolen** voor productie gebruik!

**Voor:** Interactive push/pull door individual developers

```bash
# Admin account heeft twee passwords die beide kunnen worden geregenereerd
az acr update -n $registryName --admin-enabled true # standaard uitgeschakeld

docker login $registryName.azurecr.io
```

---

## Roles

Beschikbare roles voor ACR:
- **AcrPull:** Pull images
- **AcrPush:** Push en pull images
- **AcrDelete:** Delete images

---

## CLI Commando's - Basis

### Registry beheren

```bash
# Resource group maken
az group create --name myResourceGroup --location eastus

# ACR maken
az acr create \
    --resource-group myResourceGroup \
    --name myregistry \
    --sku Standard

# ACR updaten
az acr update --name myregistry --tags key=value

# ACR details bekijken
az acr show --name myregistry --query "loginServer"

# Inloggen
az acr login --name myregistry
```

### Images beheren

```bash
# Repositories lijst
az acr repository list --name myregistry --output table

# Tags van repository bekijken
az acr repository show-tags \
    --name myregistry \
    --repository myimage \
    --output table
```

### Image van andere registry importeren

```bash
az acr import \
    --name myregistry \
    --source myregistry.azurecr.io/myimage:tag
```
