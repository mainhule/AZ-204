# Compute oplossingen in Azure
Compute is het vermogen om programma’s uit te voeren, berekeningen te doen, en workloads te draaien. In Azure zijn er verschillende manieren om dat te doen, afhankelijk van je behoeften qua schaal, flexibiliteit, beheer en kosten.

## 🚀 Azure Container Apps
**Geschikt voor:** Serverless microservices op basis van containers
- ✅ Algemeen inzetbaar, Kubernetes-achtig
- ✅ Event-driven architecturen
- ✅ Langlopende processen
- 🎯 **Wanneer gebruiken:** Geen directe toegang tot Kubernetes API nodig

## 🌐 Azure App Service
**Geschikt voor:** Volledig beheerde hosting voor webapps en web-API's
- ✅ Geoptimaliseerd voor webapplicaties
- ✅ Integratie met andere Azure diensten
- 🎯 **Wanneer gebruiken:** Traditionele web apps en API's

## 📦 Azure Container Instances (ACI)
**Geschikt voor:** Eenvoudige "bouwsteen" voor containers
- ❌ Geen schaalbaarheid
- ❌ Geen load balancing
- ❌ Geen certificaten
- 🎯 **Wanneer gebruiken:** Simpele, tijdelijke containers

## ⚙️ Azure Kubernetes Service (AKS)
**Geschikt voor:** Volledig beheerde Kubernetes in Azure
- ✅ Directe toegang tot Kubernetes API
- ✅ Draait elke Kubernetes workload
- 🎯 **Wanneer gebruiken:** Volledige controle over Kubernetes nodig

## ⚡ Azure Functions
**Geschikt voor:** Event-driven applicaties met functions model
- ✅ Kortlevende functies
- ✅ Als code of container
- 🎯 **Wanneer gebruiken:** Kleine, specifieke taken

---

## 🔄 Belangrijke vergelijkingen

**Container Apps vs AKS:**
- Heb je toegang tot Kubernetes API nodig? → Kies AKS
- Anders → Container Apps

**Container Instances vs Container Apps:**
- ACI = basis bouwsteen
- Container Apps = ACI + schaalbaarheid + extra features

**Functions vs Container Apps:**
Functions: Ideaal voor: kleine, kortdurende taken die reageren op events.
Azure Container Apps: Ideaal voor: algemene workloads, microservices, en complexere toepassingen.
- Beide geschikt voor event-driven
- Functions = korte taken
- Container Apps = algemeen gebruik

## 📊 Feature vergelijking

### Schaling
- **Container Apps:** 🟢 Auto-scaling
- **Container Instances:** 🔴 Alleen handmatig
- **App Service:** 🟢 Auto-scaling
- **AKS:** 🟡 Cluster Autoscaler
- **Functions:** 🟢 Consumption-based auto-scaling

### State Management
State management verwijst naar het opslaan en beheren van data die je applicatie nodig heeft om te functioneren. 
- **Container Apps:** 🔴 Stateless
- **Container Instances:** 🔴 Stateless
- **App Service:** 🟢 Stateless & stateful
- **AKS:** 🟢 Stateless & stateful
- **Functions:** 🔴 Stateless

Voor Stateless services (Container Apps, Functions, ACI):

❌ Geen lokale file storage
❌ Geen session data in memory/disk
✅ Gebruik externe storage (Blob, Cosmos DB, Redis)
✅ Elke container instance is identiek
✅ Makkelijker te schalen

Voor Stateful services (App Service, AKS):

✅ Lokale file storage mogelijk
✅ Session data kan lokaal
⚠️ Complexer om te schalen (data moet gesynchroniseerd)
⚠️ Backup/restore belangrijker

### Resource Isolation
- **Container Apps:** 🟡 Shared
- **Container Instances:** 🟢 Dedicated
- **App Service:** 🟡 Shared/Dedicated
- **AKS:** 🟢 Dedicated
- **Functions:** 🟡 Shared

### Kosten
- **Container Apps:** 💰 Pay-as-you-go
- **Container Instances:** 💰 Pay-as-you-go
- **App Service:** 💰💰 Vast + schaalbaar
- **AKS:** 💰💰💰 Cluster + node kosten
- **Functions:** 💰 Pay-as-you-go of vast

### Multi-Region Support
- **Container Apps:** ❌ Nee
- **Container Instances:** ❌ Nee
- **App Service:** ✅ Ja
- **AKS:** ✅ Ja
- **Functions:** ✅ Ja

> ⚠️ **Let op:** Functions is geen containeroplossing.
