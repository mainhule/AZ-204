# [Event Grid](https://docs.microsoft.com/en-us/azure/event-grid/overview)

Event Grid is een event-driven architectuur service die events beheert en distribueert van Azure en non-Azure services naar subscribers. Het gebruikt een pub/sub (publicatie/subscription) model waardoor publishers events kunnen uitstoten zonder zich zorgen te maken over hoe deze events worden afgehandeld.

## [Concepten](https://learn.microsoft.com/en-us/azure/event-grid/concepts)

- **Events**: Wat er is gebeurd (lightweight, _maximaal 1 MB_)
- **Event sources** (Publishers): Waar het event plaatsvond
- **Topics**: Het endpoint waar publishers events naartoe sturen
- **Event subscriptions**: Het endpoint of ingebouwde mechanisme om events te routeren, soms naar meerdere handlers. Wordt ook gebruikt door handlers om events op een slimme manier te filteren
- **Event handlers** (Subscribers): De app of service die reageert op het event

## Events

Een event is de kleinste hoeveelheid informatie die volledig beschrijft wat er in het systeem plaatsvond. Ze bevatten common informatie zoals source, tijd en unieke identifier. Ze hebben ook specifieke informatie relevanter voor het type event (bijvoorbeeld storage blob creatie events bevatten details over het bestand zoals de `lastTimeModified` waarde). Elk event is beperkt tot **1 MB data**.

## [Event Sources (Publishers)](https://learn.microsoft.com/en-us/azure/event-grid/overview#event-sources)

Een event source is waar het event gebeurt. Elke event source is gerelateerd aan één of meer event types. Bijvoorbeeld: Azure Storage is de event source voor blob created events.

- **Azure Subscriptions** en **Resource Groups**: Resource management changes.
- **Container Registry**: Image push en delete events.
- **Event Hub**: File capture events.
- **Service Bus**: Messages voor active subscriptions.
- **Storage Accounts**: Blob en queue item events.
- **Media Services**: Job en media analytics events.
- **Azure IoT Hub**: Telemetry en device events.
- **Custom Topics**: Events voor apps of services.

## [Topics](https://learn.microsoft.com/en-us/azure/event-grid/concepts#topics)

Een Event Grid topic is een _endpoint_ waar een event source (publisher) events naartoe stuurt. Publishers creëren topics en subscribers besluiten welke topics ze willen volgen. Topics worden ook gebruikt voor een collectie van gerelateerde events. Er zijn verschillende types topics:

- [**System topics**](https://learn.microsoft.com/en-us/azure/event-grid/system-topics): Ingebouwde topics door Azure services zoals Azure Storage, Azure Event Hubs en Azure Service Bus. Ze worden automatisch aangemaakt in je Azure subscription zodra je bepaalde Azure resources aanmaakt. Je kunt subscriben op system topics om events van Azure resources te ontvangen.

- [**Custom topics**](https://learn.microsoft.com/en-us/azure/event-grid/custom-topics): Application en third-party topics. Worden gebruikt om events van je eigen applicatie of third-party service te publiceren.

- [**Partner topics**](https://learn.microsoft.com/en-us/azure/event-grid/partner-events-overview): Gebruikt om events van third-party services zoals Salesforce, SAP of Auth0 te ontvangen. Om een partner topic te gebruiken, moet je eerst een partner registratie aanmaken en dan autoriseren dat de partner events publiceert naar je subscription.

- **Domain topics**: Komen voor binnen Event Grid domains. Helpen bij het organiseren van events voor een groot aantal subscriptions. Ze vereenvoudigen het beheer van grote aantallen topics door ze onder één resource te groeperen. Ze zijn ideaal voor multi-tenant applicaties waar meerdere tenants hun eigen topics nodig hebben.

```sh
# System Topics worden automatisch aangemaakt zodra je Azure resources aanmaakt die events ondersteunen
# Om een subscription te maken op een system topic:
az eventgrid system-topic event-subscription create \
  --name <subscription-name> \
  --resource-group <resource-group> \
  --system-topic-name <system-topic-name> \
  --endpoint <endpoint-url>

# Maak een custom topic
az eventgrid topic create --name <topic-name> \
  --resource-group <resource-group> \
  --location <location>

# Maak een event subscription voor een custom topic
az eventgrid event-subscription create --name <subscription-name> \
  --source-resource-id <topic-resource-id> \
  --endpoint <endpoint-url>
```

## [Event Subscriptions](https://learn.microsoft.com/en-us/azure/event-grid/concepts#event-subscriptions)

Een subscription vertelt Event Grid welke events op een topic je geïnteresseerd bent te ontvangen. Je kunt events filteren op _event type_ of _subject pattern_ bij het aanmaken van een subscription.

**Vereist beheerder of bijdrager toegang op de resource** die de event source is.

### Expiration

Voor **non-Azure resources** laat Event Grid event subscriptions automatisch expireren na een bepaalde tijd (standaard 60 dagen zonder activiteit om unused endpoints op te ruimen). Event expiration wordt niet ondersteund voor Azure resources subscriptions.

## [Event Handlers (Subscribers)](https://learn.microsoft.com/en-us/azure/event-grid/event-handlers)

Een event handler (of subscriber) is een plaats waar het event wordt gestuurd voor verdere processing. De handler voert verdere actions uit om het event te verwerken. Event Grid ondersteunt meerdere handler types. Een ondersteunde Azure service of je eigen webhook kan gebruikt worden als handler. Event Grid gebruikt verschillende mechanismen om de delivery van events te garanderen, afhankelijk van het type handler.

Ondersteunde handlers:

- **Webhooks**: Azure Automation runbooks en Logic Apps
- **Azure functions**
- **Event Hubs**: Voor scenario's waar hoge throughput vereist is of de applicatie events in batches verwerkt
- **Service Bus queues en topics**: Gebruik voor enterprise messaging, wanneer events moeten worden verwerkt in een specifieke volgorde of wanneer je durable message storage nodig hebt
- **Storage Queues**: Voor scenario's waar simpele FIFO (first in, first out) processing vereist is en waar de applicatie de queue moet pollen voor nieuwe messages
- **Relay Hybrid Connections**

## [Event Grid Schema](https://learn.microsoft.com/en-us/azure/event-grid/event-schema)

Events hebben een **set van common data** (bijvoorbeeld: event time, event id, event type) en **resource-specifieke data** (bijvoorbeeld: storage account events hebben blob URL in de data object). De array bevat één of meer event objects. Er is een limiet van **1 MB** voor de totale event size. Elke event in de array is beperkt tot **1 MB**. Als een event of de array de size limiet overschrijdt, ontvang je de response **413 Payload Too Large**.

Merk op dat alle events met dezelfde `eventType` waarde dezelfde schema voor data object hebben:

```json
[
  {
    "topic": "string", // Volledige resource path naar de event source. Dit veld is niet schrijfbaar. Event Grid biedt deze waarde.
    "subject": "string", // Publisher-gedefinieerde path naar het event subject
    "id": "string", // Unieke identifier voor het event
    "eventType": "string", // Een van de geregistreerde event types voor deze event source
    "eventTime": "string", // De tijd waarop het event is gegenereerd op basis van de UTC tijd van de provider
    "data": { // Resource-specifieke data
      "object-unique-key-for-each-value": "value"
    },
    "dataVersion": "string", // De schema versie van het data object. De publisher definieert de schema versie.
    "metadataVersion": "string", // De schema versie van de event metadata. Event Grid definieert het schema van de top-level properties. Event Grid biedt deze waarde.
  }
]
```

### [CloudEvents schema](https://learn.microsoft.com/en-us/azure/event-grid/cloud-event-schema)

Naast zijn default event schema, ondersteunt Azure Event Grid native het CloudEvents v1.0 JSON schema implementatie. CloudEvents is een open specification voor het beschrijven van event data. Het vereenvoudigt interoperabiliteit door een common event schema te bieden voor zowel publiceren als consumeren van events. Dit schema biedt uniforme tooling, standaard manieren van routing en afhandeling van events, en universele manieren om het outer event schema te deserialiseren. Met een common schema kun je makkelijker werk integreren over meerdere platforms.

Zet de `eventType` waarde naar een specifieke waarde en voeg een `extension` toe:

```json
[
  {
    "specversion": "1.0",
    "type": "com.example.someevent",
    "source": "/mycontext",
    "id": "C234-1234-1234",
    "time": "2018-04-05T17:31:00Z",
    "comexampleextension1": "value",
    "comexampleothervalue": 5,
    "datacontenttype": "application/json",
    "data": {
      "appinfoA": "abc",
      "appinfoB": 123,
      "appinfoC": true
    }
  }
]
```

## [Event Delivery en Retry](https://learn.microsoft.com/en-us/azure/event-grid/delivery-and-retry)

Event Grid biedt **durable delivery**. Het probeert elk event **minstens één keer** te deliveren voor elke matching subscription onmiddellijk. Als een subscriber endpoint de receipt van een event niet bevestigt of als er een failure is, probeert Event Grid delivery opnieuw gebaseerd op een **fixed retry schedule** en **retry policy**.

Event Grid retries op de volgende errors:

- SSL certificaat validatie errors (events verzonden naar webhook endpoints die custom certificaten gebruiken)
- Connection errors
- Timeouts (standaard 60 seconden)
- HTTP status codes: 400 Bad Request, 401 Unauthorized, 404 Not Found, 408 Request Timeout, 413 Request Entity Too Large, 429 Too Many Requests, 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout

Standaard verloopt Event Grid na **24 uur** of **30 poging** (wat eerst voorkomt). Je kunt deze opties ook instellen in de subscription:

- **Maximum aantal pogingen** (1-30)
- **Event time-to-live (TTL)** (1-1440 minuten)

### Output batching

Je kunt Event Grid configureren om events te batchen voor delivery om HTTP performance te verbeteren in high-throughput scenarios. _Disabled_ by default, enabled _per subscription_. Vereist twee settings:

- **Max events per batch**: Maximum aantal events dat Event Grid per batch levert. _Overschrijdt dit nummer nooit_, maar kan minder deliveren. Stelt delivery niet uit als er minder events zijn. Moet tussen 1 en 5,000 zijn.
- **Preferred batch size in KB**: Target ceiling voor batch size in kilobytes. Kan minder deliveren als er niet genoeg events zijn om de preferred size te halen. Een batch kan groter zijn als een enkel event groter is dan de preferred size. Bijvoorbeeld: als preferred size 4 KB is en een 10-KB event naar Event Grid wordt gepusht, wordt het 10-KB event nog steeds geleverd in zijn eigen batch.

## [Filtering](https://learn.microsoft.com/en-us/azure/event-grid/event-filtering)

### Event type filtering

Standaard worden **all event types** van de event source naar het endpoint gestuurd. Je kunt beslissen om alleen bepaalde event types naar je endpoint te sturen:

```json
"filter": {
  "includedEventTypes": [
    "Microsoft.Resources.ResourceWriteFailure",
    "Microsoft.Resources.ResourceWriteSuccess"
  ]
}
```

### Subject filtering

Voor simpele filtering op subject, specificeer een **start** of **end** waarde. Bijvoorbeeld: `subject` _begint_ met `/blobServices/default/containers/testcontainer` of _eindigt_ met `.jpg`

```json
"filter": {
  "subjectBeginsWith": "/blobServices/default/containers/mycontainer/log",
  "subjectEndsWith": ".jpg"
}
```

### Advanced filtering

Filter op waarden in de data fields en specificeer de **comparison operator**. Bijvoorbeeld: alleen events waar data field `key1` is gelijk aan `value1`, `value2`, etc.

```json
"filter": {
  "advancedFilters": [
    {
      "operatorType": "NumberGreaterThanOrEquals",
      "key": "Data.Key1",
      "value": 5
    },
    {
      "operatorType": "StringContains",
      "key": "Subject",
      "values": ["container1", "container2"]
    }
  ]
}
```

Operators:

- **Number**: GreaterThan, GreaterThanOrEquals, LessThan, LessThanOrEquals, In, NotIn
- **String**: Contains, BeginsWith, EndsWith, In, NotIn
- **Boolean**: IsNullOrUndefined, IsNotNull
- **Array**: NumberIn, NumberNotIn, NumberLessThan, NumberLessThanOrEquals, NumberGreaterThan, NumberGreaterThanOrEquals, StringIn, StringNotIn, StringBeginsWith, StringEndsWith, StringContains

## [Security en Authentication](https://learn.microsoft.com/en-us/azure/event-grid/security-authentication)

### Webhook Event Delivery

Voor **push delivery** naar webhooks gebruikt Event Grid _validation handshake_ om te zorgen dat alleen geautoriseerde endpoints events ontvangen. Er zijn twee validation modes:

#### Synchronous handshake

Bij subscription creation stuurt Event Grid een `SubscriptionValidationEvent` naar het endpoint. De response moet een `validationResponse` property bevatten met de waarde van het `validationCode` veld van het event. Tijdens deze validation gebruikt Event Grid niet retry logic.

```json
[
  {
    "id": "2d1781af-3a4c-4d7c-bd0c-e34b19da4e66",
    "topic": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "subject": "",
    "data": {
      "validationCode": "512d38b6-c7b8-40c8-89fe-f46f9e9622b6",
      "validationUrl": "https://rp-eastus2.eventgrid.azure.net/eventsubscriptions/estest/validate?id=512..."
    },
    "eventType": "Microsoft.EventGrid.SubscriptionValidationEvent",
    "eventTime": "2018-01-25T22:12:19.4556811Z",
    "metadataVersion": "1",
    "dataVersion": "1"
  }
]
```

Voor validation response moet je een `validationResponse` property retourneren:

```json
{
  "validationResponse": "512d38b6-c7b8-40c8-89fe-f46f9e9622b6"
}
```

#### Asynchronous handshake

Wanneer je de `validationCode` niet kunt retourneren in een response, doe dan een GET request naar de URL in de `validationUrl` property. De subscription wordt alleen approved als de GET request een `200 OK` status retourneert.

### Publishing Events naar Event Grid

Voor publishing van events moeten requests naar Event Grid toegang hebben:

- **SAS Token**: Creëer een Shared Access Signature (SAS) token voor de resource. Event Grid ondersteunt twee types SAS tokens: _topic level_ en _event subscription level_.
- **Key authentication**: Gebruik access keys die je kunt ophalen van de topic of domain.
- **Microsoft Entra JWT Token**: Meer security en fijnmazige toegangscontrole.

## Belangrijke Exam Tips

1. ✅ **Event size limiet**: Maximaal 1 MB per event, array ook maximaal 1 MB
2. ✅ **Event delivery**: At-least-once delivery, retries voor 24 uur of 30 pogingen
3. ✅ **Topics**: System (Azure services), Custom (eigen apps), Partner (third-party)
4. ✅ **Schema types**: Event Grid schema en CloudEvents v1.0 schema
5. ✅ **Retry errors**: 400, 401, 404, 408, 413, 429, 500, 502, 503, 504, SSL errors, timeouts
6. ✅ **Filtering**: Event type, Subject (begin/end), Advanced (operators op data fields)
7. ✅ **Event handlers**: Webhooks, Functions, Event Hubs, Service Bus, Storage Queues
8. ✅ **Subscription validation**: Synchronous (validationCode in response) of Asynchronous (GET naar validationUrl)
9. ✅ **Batching**: Max events per batch (1-5000), Preferred batch size in KB
10. ✅ **Security**: SAS token, Key authentication, Microsoft Entra JWT token
11. ✅ **Expiration**: Non-Azure subscriptions expireren na 60 dagen inactiviteit
12. ✅ **System topics**: Automatisch aangemaakt bij Azure resource creation
13. ✅ **Domain topics**: Voor multi-tenant scenarios, groupeer topics onder één resource
14. ✅ **Event subscription**: Vereist admin/contributor access op de event source resource
15. ✅ **CloudEvents**: Open specification voor interoperabiliteit, uniform event schema
