# [Azure Event Hubs](https://docs.microsoft.com/en-us/azure/event-hubs/event-hubs-about)

Azure Event Hubs is een big data streaming platform en event ingestion service. Het kan miljoenen events per seconde ontvangen en verwerken. Data verzonden naar een event hub kan worden getransformeerd en opgeslagen met elke real-time analytics provider of batching/storage adapters.

## [Belangrijke Concepten](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-features)

- **Event Hubs client**: De primaire interface voor developers die interacteren met de Event Hubs client library. Er zijn verschillende Event Hubs clients, elk gewijd aan een specifieke use case zoals publishing of consuming events.

- **Event Hubs producer**: Een type client die dient als bron voor telemetry data, diagnostische informatie, usage logs of andere log data, als onderdeel van een embedded device solution, een mobiele device applicatie, een gaming titel draaiend op een console of ander apparaat, een client- of server-gebaseerde business solution, of een website.

- **Event Hubs consumer**: Een type client die telemetry data, diagnostische informatie, usage logs of andere log data leest van de Event Hub en ermee verwerkt. Vaak zijn dit robust, high-scale platform infrastructure delen met ingebouwde analytics capabilities, zoals Azure Stream Analytics, Apache Spark.

- **Partition**: Een ordered sequence van events die wordt vastgehouden in een Event Hub. Partities zijn een organisatiemechanisme dat gerelateerd is aan de downstream parallelism vereist door event consumers. Azure Event Hubs biedt message streaming via een partitioned consumer pattern waarin elke consumer alleen een specifieke subset, of partition, van de message stream leest. Als nieuwere events arriveren, worden ze toegevoegd aan het einde van deze sequence. Het aantal partitions wordt gespecificeerd bij het aanmaken en kan niet worden gewijzigd.

- **Consumer group**: Een view (state, positie of offset) van een volledige Event Hub. Consumer groups stellen applicaties in staat om elk een aparte view te hebben van de event stream en om de stream onafhankelijk te lezen in hun eigen tempo met hun eigen offsets. Er kan maximaal 5 concurrent lezende consumers op een partition per consumer group zijn; echter het wordt **aanbevolen dat er slechts één actieve consumer is voor een gegeven partition en consumer group pairing**. Elke lezer ontvangt alle events.

- **Event receivers**: Elke entity die event data van een Event Hub leest. Event receivers verbinden via het AMQP 1.0 sessie. Het Event Hubs service levert events via een sessie zodra data beschikbaar is. Alle Kafka consumers fungeren als event receivers.

- **Throughput units** of **processing units**: Vooraf gekochte units van capaciteit die de throughput capaciteit van Event Hubs bepalen.

## [Event Hubs Capture](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-capture-overview)

Azure Event Hubs stelt je in staat om streaming data in Event Hubs automatisch te capturen in een Azure Blob storage of Azure Data Lake Storage account, met toegevoegde flexibiliteit om een tijd- of grootteinterval te specificeren.

Captured data wordt geschreven in **Apache Avro** formaat (een compact, snel, binair formaat dat rich data structuren biedt met inline schema).

### Hoe Event Hubs Capture werkt

Event Hubs is een time-retention durable buffer voor telemetry ingress, vergelijkbaar met een distributed log. De key tot scaling in Event Hubs is het [partitioned consumer model](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-features#partitions). Elke partition is een onafhankelijke data segment en wordt onafhankelijk geconsumeerd. Deze data verouderen op basis van de configureerbare retention period. Als gevolg hiervan raakt een gegeven Event Hub nooit "te vol".

Met Event Hubs Capture kun je je eigen Azure Blob storage account en container of Azure Data Lake Store account specificeren, die worden gebruikt om de gecaptureerde data op te slaan. Deze accounts kunnen zich in dezelfde regio bevinden als je Event Hub of in een andere regio.

Captured data wordt geschreven in **Apache Avro** formaat. Avro is een wijd gebruikt formaat in de Hadoop ecosystem.

### Capture windowing

Event Hubs Capture stelt je in staat om een window in te stellen voor het beheersen van capturing. Dit window is een minimale size en tijdsconfiguratie met een "first wins policy," wat betekent dat de eerste trigger die wordt tegengekomen een capture operatie veroorzaakt. Het volgende heeft 15 minuten capture window van 100 MB die triggers wanneer ofwel **15 minuten** zijn verstreken of wanneer **100 MB** data is gecaptured (wat eerst gebeurt):

```sh
az eventhubs eventhub update --resource-group $resourceGroup --name "hubName" \
    --namespace-name "namespaceName" --enable-capture true \
    --capture-interval 900 --capture-size-limit 104857600 \
    --destination-name EventHubArchive.AzureBlockBlob \
    --storage-account "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.Storage/storageAccounts/{storageAccount}" \
    --blob-container "containerName" --archive-name-format "{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}"
```

Merk op dat `--capture-interval` in **seconden** is en `--capture-size-limit` in **bytes**.

### Captured files naming convention

- `--archive-name-format`: `"{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}"`
- Bijvoorbeeld: `mynamespace/myeventhub/0/2017/12/08/03/03/17.avro`

### Scaling naar Throughput Units

Event Hubs traffic wordt gecontroleerd door **throughput units** (standard tier) of **processing units** (premium tier). Een enkele throughput unit staat toe:

- **Ingress**: Maximaal 1 MB per seconde of 1000 events per seconde (wat eerst komt).
- **Egress**: Maximaal 2 MB per seconde of 4096 events per seconde.

Als je meer dan de limiet van aangekochte throughput units probeert, wordt ingress gecontroleerd en worden throttling errors geretourneerd. Deze units kunnen worden verhoogd in incrementen van 20 via een support ticket voor zoveel als nodig is.

## [Event Processor](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-event-processor-host)

### EventProcessorClient (nieuwere versie)

De EventProcessorClient is een moderne client voor het consumeren van events in een load-balanced manier over meerdere instances van je applicatie. Het beheert:

- **Checkpoint management**: Automatisch bijhouden van de laatste verwerkte positie voor elke partition
- **Load balancing**: Automatische verdeling van partitions over meerdere instances
- **Error handling**: Retry logic en error recovery

```cs
var storageClient = new BlobContainerClient("connection-string", "container-name");
var processor = new EventProcessorClient(storageClient, "consumer-group", "connection-string", "event-hub-name");

// Registreer handlers voor processing events en errors
processor.ProcessEventAsync += ProcessEventHandler;
processor.ProcessErrorAsync += ProcessErrorHandler;

await processor.StartProcessingAsync();
```

### EventProcessorHost (oudere versie)

De EventProcessorHost is een intelligent consumer agent die het lezen van events van een Event Hub vereenvoudigt door de persistent checkpoints en parallelle reads te beheren. Met EventProcessorHost kunnen events worden gedistribueerd over meerdere receivers, zelfs wanneer deze worden gehost in verschillende nodes. Dit voorbeeld toont hoe EventProcessorHost wordt gebruikt voor een enkele receiver:

```cs
var eventProcessorHost = new EventProcessorHost(
    eventHubName,
    consumerGroupName,
    eventHubConnectionString,
    storageConnectionString,
    storageContainerName);

await eventProcessorHost.RegisterEventProcessorAsync<SimpleEventProcessor>();
await eventProcessorHost.UnregisterEventProcessorAsync();
```

Voordelen:

- **Checkpointing**: Maintain state over meerdere instances
- **Lease management**: Automatische verdeling van partitions
- **Fault tolerance**: Herbalanceren bij failures
- **Scaling**: Eenvoudig toevoegen van meer consumers

## [Partitions](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-features#partitions)

Een partition is een ordered sequence van events waarbij de meest recente events worden toegevoegd aan het einde. Partitions zijn het mechanisme voor schaalbaarheid, parallellisme en geordende event delivery.

### Aantal Partitions

Het aantal partitions wordt gespecificeerd bij creation en moet **tussen 1 en 32** zijn in de **standard tier**. In de **premium en dedicated tiers** kan het partition aantal oplopen tot **maximaal 2000 partitions per Event Hub**.

### Partition Keys

Een **partition key** is een waarde die wordt gebruikt om incoming event data te mappen naar specifieke partitions voor data organization doeleinden. Het is een _sender-supplied_ waarde doorgegeven aan een Event Hub. Het wordt verwerkt via een statische hashing functie, die de partition assignment creëert.

Events met dezelfde partition key worden altijd naar **dezelfde partition** gestuurd, wat geordende delivery garandeert voor die set van events. Als geen partition key wordt gespecificeerd, worden events in een **round-robin** manier gedistribueerd over alle partitions.

```cs
var eventData = new EventData(Encoding.UTF8.GetBytes("message"));
eventData.PartitionKey = "user-123"; // Events met dezelfde key gaan naar dezelfde partition
await producerClient.SendAsync(new[] { eventData });
```

### Waarom Partitions belangrijk zijn

- **Parallellisme**: Elke partition kan onafhankelijk worden gelezen door verschillende consumers
- **Geordende delivery**: Events binnen een partition worden in volgorde geleverd
- **Load balancing**: Distribueer load over meerdere consumer instances
- **Throughput**: Meer partitions = meer parallelle verwerking = hogere throughput

## Gebeurtenis publiceren

### EventDataBatch

Gebruik een `EventDataBatch` object voor het batchen van events voordat ze worden gepubliceerd. Dit is efficiënter dan het verzenden van individuele events:

```cs
var producerClient = new EventHubProducerClient("connection-string", "event-hub-name");

// Maak een batch
EventDataBatch eventBatch = await producerClient.CreateBatchAsync();

// Probeer events toe te voegen aan de batch
for (int i = 1; i <= 3; i++)
{
    if (!eventBatch.TryAdd(new EventData(Encoding.UTF8.GetBytes($"Event {i}"))))
    {
        // Als het event niet past, verzend de huidige batch en maak een nieuwe
        await producerClient.SendAsync(eventBatch);
        eventBatch = await producerClient.CreateBatchAsync();
        
        // Voeg het event toe aan de nieuwe batch
        if (!eventBatch.TryAdd(new EventData(Encoding.UTF8.GetBytes($"Event {i}"))))
        {
            throw new Exception($"Event {i} is te groot voor de batch");
        }
    }
}

// Verzend de laatste batch
if (eventBatch.Count > 0)
{
    await producerClient.SendAsync(eventBatch);
}
```

## [AMQP vs HTTPS](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-amqp-overview)

### AMQP (Advanced Message Queuing Protocol)

- **Persistent connection**: Maakt een duurzame bi-directionele TCP verbinding
- **Beste performance**: Lagere latency, hogere throughput
- **Best voor**: Long-running applicaties, high-volume scenarios, real-time streaming
- **Overhead**: Hogere initiële setup kosten maar lagere kosten per message
- **Poorten**: Gebruikt poort 5671 (AMQP) of 5672 (AMQP zonder TLS)

### HTTPS

- **Request/Response**: Elke operatie is een nieuwe HTTP request
- **Firewall friendly**: Werkt achter firewalls die alleen HTTP/HTTPS toestaan
- **Best voor**: Short-lived applicaties, lage volume scenarios, eenvoudige test scenarios
- **Overhead**: Hogere overhead per message door TLS handshake voor elke request
- **Poorten**: Gebruikt poort 443

### Kiezen tussen AMQP en HTTPS

| Scenario | Protocol |
|----------|----------|
| High-volume streaming | AMQP |
| Real-time processing | AMQP |
| Long-running consumers | AMQP |
| Enterprise messaging | AMQP |
| Firewall restrictions | HTTPS |
| Quick testing | HTTPS |
| Low-volume scenarios | HTTPS |

## Belangrijke Exam Tips

1. ✅ **Throughput units**: 1 MB/s ingress (1000 events/s), 2 MB/s egress (4096 events/s)
2. ✅ **Partitions**: 1-32 (standard), tot 2000 (premium/dedicated), geordende delivery binnen partition
3. ✅ **Partition key**: Events met dezelfde key → zelfde partition → geordende delivery
4. ✅ **Consumer groups**: Max 5 concurrent readers per partition, aanbeveling: 1 actieve consumer
5. ✅ **Capture**: Avro formaat, naar Blob Storage of Data Lake, time/size window triggers
6. ✅ **Event Processor**: Checkpoint management, load balancing, automatic partition assignment
7. ✅ **AMQP vs HTTPS**: AMQP voor high-volume/real-time, HTTPS voor firewall/testing
8. ✅ **EventDataBatch**: Efficient batching, gebruik TryAdd() om size limieten te respecteren
9. ✅ **Capture interval**: In seconden, capture size: in bytes
10. ✅ **Avro formaat**: Compact binary formaat met inline schema
11. ✅ **Standard tier**: Tot 32 partitions, throughput units
12. ✅ **Premium tier**: Tot 2000 partitions, processing units, betere performance
13. ✅ **Round-robin**: Geen partition key = events verdeeld over alle partitions
14. ✅ **Checkpoint storage**: Vereist Blob Storage container voor EventProcessorClient
15. ✅ **Scaling**: Verhoog throughput units in incrementen van 20 via support ticket
