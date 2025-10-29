# Azure Container Apps

Volledig beheerd, serverless containerplatform met auto-scaling.

**Use cases:**
- API endpoints
- Microservices
- Event-driven processing
- Background processing

**Belangrijke concepten:**

## Scaling
- HTTP: Op basis van gelijktijdige requests
- TCP: Op basis van connections
- Custom: CPU, geheugen, Service Bus, Event Hubs, etc.
- Default: 0-10 replicas (kan naar 0 schalen)

## Revisions
- Onveranderlijke snapshots van je app
- Single mode: Oude blijft actief tot nieuwe klaar is
- Multiple mode: Je controleert traffic verdeling
- Tot 100 revisions bewaren

## Secrets
- App-level (niet revision-level)
- Key Vault integratie mogelijk
- Mount als volumes of environment variables

## Disaster Recovery
- Manual: Wacht op regio herstel
- Resilient: Deploy naar meerdere regio's + traffic manager

## Dapr Integration
- Service-to-service communicatie
- State management
- Pub/sub messaging
- External bindings
- Actor model
- Observability

**Belangrijkste CLI commando's:**
```bash
az containerapp env create --name prod
az containerapp up --name myapp --environment prod
az containerapp create --name myapp --image myapp:v1
```
