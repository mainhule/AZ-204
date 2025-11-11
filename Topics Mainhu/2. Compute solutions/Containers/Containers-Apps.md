# Azure Container Apps

Volledig beheerd, serverless containerplatform met auto-scaling.

## Use Cases

✅ API endpoints deployen  
✅ Microservices hosten  
✅ Event-driven processing afhandelen  
✅ Background processing applicaties draaien

---

## Beperkingen

❌ **Kan geen privileged containers draaien** (geen root)

❌ **Alleen linux/amd64 container images** zijn vereist

⚠️ **State persist niet** binnen een container vanwege regelmatige restarts  
→ Gebruik externe caches voor in-memory cache vereisten

---

## Webhooks

Een webhook kan worden gebruikt om Azure Container Apps te notificeren wanneer een nieuwe image naar ACR is gepusht, wat automatische deployment van de updated container triggert.

---

## Environments

### Wanneer meerdere environments gebruiken?

**🌐 Verschillende Virtual Networks:**  
VNet is scoped op environment level.

**📊 Verschillende Log Analytics workspaces:**  
Diagnostics settings zijn gekoppeld aan environment.

**🗺️ Verschillende regionale deployments:**  
Apps in verschillende Azure regio's - een environment is region-bound.

**🔐 Verschillende managed identities voor environment-level operaties:**  
Alleen als je environment-scope identity gebruikt (zeldzaam).

---

## Authentication

De platform's authentication en authorization middleware component draait als een **sidecar** container op elke application replica, en screent alle inkomende HTTPS requests voordat ze je applicatie bereiken.

### Vereisten

✅ `allowInsecure` moet **disabled** zijn in ingress config

✅ Vereist een identity provider

✅ Gespecificeerde provider binnen app settings

**📚 Zie meer:** App Service Web Apps documentatie

---

## Managed Identities

**Hoofd topic:** Managed Identities

⚠️ **Niet ondersteund in scaling rules**

---

## Scaling

Scaling wordt gestuurd door drie verschillende categorieën triggers:

### 🌐 HTTP Trigger
Op basis van het aantal concurrent HTTP requests naar je revision.

### 🔌 TCP Trigger
Op basis van het aantal concurrent TCP connections naar je revision.

### ⚙️ Custom Triggers
Op basis van:
- **CPU** (_kan niet naar 0 schalen_)
- **Memory** (_kan niet naar 0 schalen_)
- **Event-driven data sources:**
  - Azure Service Bus
  - Azure Event Hubs
  - Apache Kafka
  - Redis

---

## Replicas

Wanneer een container app revision uitschaalt, worden nieuwe instances van de revision on-demand gemaakt. Deze instances worden **replicas** genoemd.

**Default:** 0-10 replicas

---

## Scale Rules

Het toevoegen of bewerken van scaling rules creëert een nieuwe revision van de container app.

In **"multiple revisions" mode:**
- Nieuwe scale trigger maakt nieuwe revision
- Oude revision blijft beschikbaar met oude scale rules

### Voorbeeld

```bash
az containerapp create \
 --name myapp \
 --resource-group myResourceGroup \
 --environment myEnvironment \
 
 # Replica limieten
 --min-replicas 0 \
 --max-replicas 5 \

 # HTTP Scaling Rule
 --scale-rule-name http-rule-name \
 --scale-rule-type http \
 --scale-rule-http-concurrency 100 \

 # TCP Scaling Rule
 --scale-rule-name tcp-rule-name \
 --scale-rule-type tcp \
 --scale-rule-tcp-concurrency 100 \

 # Custom Scaling rule (Service Bus voorbeeld)
 --secrets "connection-string-secret=<SERVICE_BUS_CONNECTION_STRING>" \
 --scale-rule-auth "connection=connection-string-secret" \
 --scale-rule-name servicebus-rule-name \
 --scale-rule-type azure-servicebus \
 --scale-rule-metadata "queueName=my-queue" "namespace=service-bus-namespace" "messageCount=5"
```

---

## Zonder Scale Rule

⚠️ **Zonder scale rule:**
- Default (HTTP, 0-10 replicas) is van toepassing op je app
- Maak een rule of stel `minReplicas` in op 1+ als ingress is uitgeschakeld
- Zonder `minReplicas` of custom rule kan je app naar nul schalen en **niet starten**

---

## Revisions

Onveranderlijke snapshots van een container app version.

### Automatische Creatie

🎯 Eerste revision wordt **automatisch gemaakt** bij deployment

🔄 Nieuwe revisions worden gemaakt bij **revision scope changes**

📦 Tot **100 revisions** kunnen worden bewaard voor historie

### Revision Modes

**🔵 Single Revision Mode:**
- Houdt oude revision actief tot nieuwe klaar is
- Traffic switcht automatisch

**🟣 Multiple Revision Mode:**
- Je controleert revision lifecycle en traffic distributie (via ingress)
- Traffic switcht alleen naar laatste revision wanneer deze klaar is

### Revision Labels

Direct traffic naar specifieke revisions. Een label biedt een unieke URL die je kunt gebruiken om traffic naar de revision te routeren waaraan het label is toegewezen.

---

## Scopes

### 🔄 Revision-Scope Changes

Triggeren een nieuwe revision bij deployment via `az containerapp update`.

**Trigger:** Wijzigingen aan `properties.template`

**Voorbeelden:**
- Version suffix
- Container configuratie
- Scaling rules

**Effect:** Wijzigingen zijn niet van toepassing op andere revisions

### 🌍 Application-Scope Changes

Globaal toegepast op alle revisions. **Geen nieuwe revision** wordt gemaakt.

**Trigger:** Wijzigingen aan `properties.configuration`

**Voorbeelden:**
- Secrets
- Revision mode
- Ingress
- Credentials
- DAPR settings

---

## Secrets

Secrets zijn scoped op **application level** (`az containerapp create`), buiten specifieke revisions.

### Belangrijk

✅ Secrets gedefinieerd op application level

❌ **Toevoegen, verwijderen, of wijzigen genereert geen nieuwe revisions**

⚠️ **Apps moeten worden herstart** om updates te reflecteren

### Secrets Definiëren

**Gewone secrets:**
```bash
--secrets "queue-connection-string=<CONNECTION_STRING>"
```

**Secrets vanuit Key Vault:**
```bash
--secrets "kv-connection-string=keyvaultref:<KEY_VAULT_SECRET_URI>,identityref:<USER_ASSIGNED_IDENTITY_ID>"
```

### Secrets Mounten in Volume

```bash
--secret-volume-mount "/mnt/secrets"
```

**Effect:** Secret name = bestandsnaam, secret value = inhoud

### Secrets in Environment Variables

```bash
--env-vars "QueueName=myqueue" "ConnectionString=secretref:queue-connection-string"
```

**Let op het `secretref:` prefix!**

---

## Logging

### Log Types

**System Logs:**  
Op container app level

**Console Logs:**  
Van `stderr` en `stdout` messages binnen container app

### Query Logs met Log Analytics

```bash
az monitor log-analytics query \
    --workspace $WORKSPACE_CUSTOMER_ID \
    --analytics-query "ContainerAppConsoleLogs_CL | where ContainerAppName_s == 'album-api' | project Time=TimeGenerated, AppName=ContainerAppName_s, Revision=RevisionName_s, Container=ContainerName_s, Message=Log_s, LogLevel_s | take 5" \
    --out table
```

**Beschikbare tables:**
- `ContainerAppConsoleLogs_CL`
- `ContainerAppSystemLogs_CL`
