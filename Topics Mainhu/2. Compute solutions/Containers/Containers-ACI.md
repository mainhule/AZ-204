# Azure Container Instances (ACI)

Simpele containerdeployment zonder VM's te hoeven beheren. Ondersteunt containers tot 15GB.

## ❌ Belangrijke Beperkingen

**Geen scaling ondersteuning!**
- Voor scaling gebruik je Container Apps

**IP Adres:**
- Kan veranderen bij restart
- Vermijd hardcoded IP adressen
- Voor stabiele public IP: overweeg Application Gateway

**Maximum:**
- 15GB per container

---

## Gebruik Voor

✅ Eenvoudige, kortstondige taken  
✅ Ontwikkeling en testen  
✅ Batch processing  
✅ Quick deployments zonder infrastructuur management

---

## Volume Opties

### ✅ Azure File Share
- **Alleen** voor Linux containers
- **Alleen** als root
- Vereist bestaand storage account en account key

**Configuratie parameters:**
```bash
--os-type Linux
--azure-file-volume-account-name <storage-account>
--azure-file-volume-account-key <key>
--azure-file-volume-mount-path /mnt/data
--azure-file-volume-share-name myshare
```

### ✅ Secret Volumes
- **Beperkt** tot Linux containers
- Maakt bestanden aan met secret name als bestandsnaam en secret value als inhoud

**Configuratie:**
```bash
--secrets mysecret1="My first secret FOO" mysecret2="My second secret BAR"
--secrets-mount-path /mnt/secrets
```

Dit maakt `mysecret1` en `mysecret2` bestanden aan in `/mnt/secrets`.

### ❌ Blob Storage
**Geen directe ondersteuning** omdat het SMB support mist.

---

## Environment Variables

### Public Environment Variables
```bash
--environment-variables 'PUBLIC_ENV_VAR'='my-exposed-value'
```

Format kan zijn:
- `'key'='value'`
- `key=value`
- `'key=value'`

### Secure Environment Variables
```bash
--secure-environment-variables 'SECRET_ENV_VAR'='my-secret-value'
```

⚠️ Niet zichtbaar in je container's properties

---

## Networking

### Public DNS Name

Moet uniek zijn - toegankelijk via:
```
<dns-label>.<region>.azurecontainer.io
```

**Configuratie:**
```bash
--dns-name-label mydnsname
--ip-address public
```

---

## Restart Policy

```bash
--restart-policy {Always, Never, OnFailure}
```

- **Always** (default): Container blijft herstarten
- **Never**: Draait maar één keer (status bij stop: Terminated)
- **OnFailure**: Herstart alleen bij failures

---

## CLI Commando's - Basis

### Container maken en deployen

```bash
# Resource group maken
az group create --name myResourceGroup --location eastus

# Container maken van image
az container create \
    --name mycontainer \
    --image myimage:v1 \
    --resource-group myResourceGroup

# Container status verifiëren
az container show \
    --name mycontainer \
    --resource-group myResourceGroup \
    --query "{FQDN:ipAddress.fqdn,ProvisioningState:provisioningState}" \
    --out table
```

### Container Logs

```bash
# Logs bekijken
az container logs \
    --name mycontainer \
    --resource-group myResourceGroup

# Real-time monitoring
az container attach \
    --name mycontainer \
    --resource-group myResourceGroup
```

---

## Deployment Opties

### 📦 Van Image (Simple)

```bash
az container create \
    --name mycontainer \
    --image nginx:latest \
    --resource-group myResourceGroup \
    --dns-name-label myuniquedns \
    --ip-address public
```

### 📋 Van YAML File

Inclusief container groups:

```bash
az container create \
    --name mycontainer \
    --file deploy.yml \
    --resource-group myResourceGroup
```

### 🔧 ARM Template

Voor deployment van extra Azure service resources (bijvoorbeeld Azure Files share).

Geen specifiek voorbeeld, maar goed om te weten dat deze optie bestaat.

---

## Container Groups

Containers gebruiken één host machine en delen:
- Lifecycle
- Resources
- Network (delen external IP, ports, DNS)
- Storage volumes

**⚠️ Let op voor Windows:**
- Alleen single-instance deployment toegestaan

**Resources:**
De resources toegewezen aan de host zijn de som van alle aangevraagde resources.
