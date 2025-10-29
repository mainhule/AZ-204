# Azure Container Instances (ACI)

Simpele containerdeployment zonder VM's beheren.

**Belangrijke beperkingen:**
- Geen scaling (gebruik Container Apps daarvoor)
- IP kan veranderen bij restart
- Maximaal 15GB per container

**Gebruik voor:**
- Eenvoudige, kortstondige taken
- Ontwikkeling en testen
- Batch processing

**Volume opties:**
- Azure File Share (alleen Linux, als root)
- Secret volumes
- Geen directe Blob Storage ondersteuning

**Belangrijkste CLI commando's:**
```bash
az container create --name mycontainer --image myapp:v1
az container logs --name mycontainer
```
