# [Azure Queue Storage](https://docs.microsoft.com/en-us/azure/storage/queues/storage-queues-introduction)

Azure Queue Storage is een service voor het opslaan van grote aantallen messages. Je hebt toegang tot messages van overal ter wereld via geauthenticeerde calls met HTTP of HTTPS. Een queue message kan tot **64 KB** groot zijn. Een queue kan **miljoenen messages** bevatten, tot de totale capaciteitslimiet van een storage account.

Queues worden vaak gebruikt om een **backlog** van werk te creëren dat asynchroon wordt verwerkt.

## [Key Concepten](https://learn.microsoft.com/en-us/azure/storage/queues/storage-queues-introduction#queue-storage-concepts)

- **URL formaat**: Queues zijn bereikbaar via het volgende URL formaat: `https://<storage account>.queue.core.windows.net/<queue>`

  Bijvoorbeeld: `https://myaccount.queue.core.windows.net/images-to-download`

- **Storage account**: Alle toegang tot Azure Storage gebeurt via een storage account.

- **Queue**: Een queue bevat een set messages. De queue naam **moet volledig lowercase zijn**.

- **Message**: Een message, in elk formaat, van maximaal **64 KB**. De maximale tijd dat een message in de queue kan blijven is **7 dagen**. Voor versie 2017-07-29 of later, kan de maximale time-to-live elk positief getal zijn, of -1 wat aangeeft dat het message niet verloopt. Als deze parameter wordt weggelaten, is de default time-to-live **zeven dagen**.

## Queue Storage vs Service Bus Queues

### Wanneer Queue Storage te gebruiken

- Je applicatie moet **meer dan 80 GB aan messages** opslaan in een queue
- Je applicatie wil **progress tracking** voor het verwerken van een message binnen de queue. Dit is nuttig als de worker die een message verwerkt crasht. Een andere worker kan die informatie gebruiken om door te gaan vanaf waar de vorige worker is gebleven.
- Je hebt **server side logs** nodig van alle transacties uitgevoerd tegen je queues

### Wanneer Service Bus Queues te gebruiken

- Je oplossing moet messages kunnen ontvangen **zonder de queue te pollen**. Met Service Bus kan dit worden bereikt met een long-polling receive operatie met de TCP-gebaseerde protocollen die Service Bus ondersteunt.
- Je oplossing vereist de queue om een **gegarandeerde first-in-first-out (FIFO)** ordered delivery te bieden
- Je oplossing moet in staat zijn om **automatische duplicate detectie** te ondersteunen
- Je wilt dat je applicatie messages verwerkt als **parallelle long-running streams** (messages worden geassocieerd met een stream met de **session ID** property op het message). In dit model concurreert elke node in de consumerende applicatie voor streams, in plaats van messages. Wanneer een stream wordt gegeven aan een consumerende node, kan de node de state van de applicatie stream state onderzoeken met transacties.
- Je oplossing vereist **transactioneel gedrag en atomicity** bij het verzenden of ontvangen van meerdere messages van een queue
- Je applicatie handelt messages af die **groter zijn dan 64 KB** maar waarschijnlijk niet de 256-KB limiet benaderen (Standard tier) of 100 MB (Premium tier)

## QueueClient

De `QueueClient` class helpt je om **messages op te slaan in Azure Queue Storage**.

### Instantie creëren

Om een `QueueClient` te instantiëren, roep je de constructor aan met ofwel een connection string of een storage URI. Gebruik `CreateIfNotExistsAsync` om de queue te maken als deze nog niet bestaat:

```cs
// Instantieer een QueueClient met een connection string
QueueClient queueClient = new QueueClient(connectionString, queueName);

// Maak de queue aan als deze nog niet bestaat
await queueClient.CreateIfNotExistsAsync();
```

### Messages naar queue sturen

Om een message te verzenden naar een queue, roep je de `SendMessageAsync` method aan. Een message kan ofwel een **string** (in UTF-8 formaat) of een **byte array** zijn:

```cs
// Stuur een message naar de queue
await queueClient.SendMessageAsync("Hello, Queue Storage!");

// Stuur een message met een custom visibility timeout en TTL
await queueClient.SendMessageAsync(
    messageText: "Delayed message",
    visibilityTimeout: TimeSpan.FromSeconds(30), // Onzichtbaar voor 30 seconden
    timeToLive: TimeSpan.FromMinutes(5) // Verloopt na 5 minuten
);
```

### Peek messages

**Peek** messages zonder ze uit de queue te verwijderen door `PeekMessagesAsync` aan te roepen. Als je geen waarde voor de `maxMessages` parameter doorgeeft, is de default om één message te peeken:

```cs
// Peek een enkel message
PeekedMessage[] peekedMessages = await queueClient.PeekMessagesAsync();
Console.WriteLine($"Peeked message: {peekedMessages[0].Body}");

// Peek meerdere messages (max 32)
PeekedMessage[] multiplePeeked = await queueClient.PeekMessagesAsync(maxMessages: 10);
foreach (var message in multiplePeeked)
{
    Console.WriteLine($"Message: {message.Body}");
}
```

### Message properties updaten

Om de **contents** en **visibility timeout** setting van een message te updaten, roep je `UpdateMessageAsync` aan. Het visibility timeout kan worden ingesteld op 0 tot 7 dagen, met -1 wat aangeeft dat het message niet verloopt:

```cs
// Ontvang een message
QueueMessage[] messages = await queueClient.ReceiveMessagesAsync(maxMessages: 1);
QueueMessage message = messages[0];

// Update het message met nieuwe content en reset visibility timeout
await queueClient.UpdateMessageAsync(
    message.MessageId,
    message.PopReceipt,
    messageText: "Updated message content",
    visibilityTimeout: TimeSpan.FromSeconds(60)
);
```

### Messages ontvangen van de queue

Ontvang messages van de front van de queue door de `ReceiveMessagesAsync` method aan te roepen. Standaard is een message **onzichtbaar voor 30 seconden** om andere code de mogelijkheid te geven om hetzelfde message te verwerken. Om de default waarde te wijzigen, stel je de parameter `visibilityTimeout` in:

```cs
// Ontvang tot 10 messages van de queue met default visibility timeout (30 seconden)
QueueMessage[] messages = await queueClient.ReceiveMessagesAsync(maxMessages: 10);

// Ontvang messages met custom visibility timeout (60 seconden)
QueueMessage[] customMessages = await queueClient.ReceiveMessagesAsync(
    maxMessages: 5,
    visibilityTimeout: TimeSpan.FromSeconds(60)
);

foreach (var message in messages)
{
    Console.WriteLine($"Message: {message.Body}");
    Console.WriteLine($"Message ID: {message.MessageId}");
    Console.WriteLine($"Pop Receipt: {message.PopReceipt}");
    Console.WriteLine($"Dequeue Count: {message.DequeueCount}");
}
```

### Message queue lengte bepalen

Je kunt een schatting krijgen van het aantal messages in een queue. De `GetProperties` method retourneert queue properties inclusief het message count. De `ApproximateMessagesCount` property bevat het geschatte aantal messages in de queue. Dit aantal is **niet lager** dan het werkelijke aantal messages in de queue, maar kan hoger zijn:

```cs
QueueProperties properties = await queueClient.GetPropertiesAsync();
int approximateMessageCount = properties.ApproximateMessagesCount;
Console.WriteLine($"Approximate message count: {approximateMessageCount}");
```

### Messages van queue verwijderen

Je kunt een message uit de queue verwijderen in twee stappen:

1. Roep `ReceiveMessagesAsync` aan om het volgende message in de queue te krijgen. Een message geretourneerd van `ReceiveMessagesAsync` wordt onzichtbaar voor elke andere code die messages uit deze queue leest. Standaard blijft dit message onzichtbaar voor 30 seconden.
2. Om het message uit de queue te verwijderen, roep je `DeleteMessageAsync` aan.

Dit twee-stappen proces zorgt ervoor dat als je code faalt om een message te verwerken door hardware of software failure, een ander instance van je code hetzelfde message kan krijgen en het opnieuw kan proberen:

```cs
// Ontvang het message
QueueMessage[] messages = await queueClient.ReceiveMessagesAsync(maxMessages: 1);
QueueMessage message = messages[0];

// Verwerk het message
Console.WriteLine($"Processing: {message.Body}");

// Verwijder het message van de queue
await queueClient.DeleteMessageAsync(message.MessageId, message.PopReceipt);
```

### Queue verwijderen

Om een queue en alle messages erin te verwijderen, roep je de `DeleteAsync` method aan op het queue object:

```cs
await queueClient.DeleteAsync();
```

## Belangrijke Properties

### QueueMessage Properties

- **MessageId**: Unieke identifier voor het message
- **PopReceipt**: Vereist voor het updaten of verwijderen van het message
- **Body**: De message content als BinaryData
- **DequeueCount**: Aantal keren dat het message is ontvangen maar niet verwijderd
- **InsertedOn**: Wanneer het message is toegevoegd aan de queue
- **ExpiresOn**: Wanneer het message verloopt
- **NextVisibleOn**: Wanneer het message weer zichtbaar wordt na being received

### Visibility Timeout

Wanneer je een message ontvangt met `ReceiveMessagesAsync`, wordt het tijdelijk onzichtbaar gemaakt voor andere consumers. Dit voorkomt dat meerdere consumers hetzelfde message tegelijk verwerken:

- **Default**: 30 seconden
- **Range**: 0 seconden tot 7 dagen
- **Special value**: -1 betekent dat het message niet verloopt

Als je het message niet verwijdert binnen de visibility timeout, wordt het automatisch weer zichtbaar en kan het opnieuw worden ontvangen.

## Advanced Scenarios

### Poison Messages

Een "poison message" is een message dat herhaaldelijk faalt om te verwerken. Gebruik de `DequeueCount` property om te detecteren hoeveel keer een message is ontvangen:

```cs
QueueMessage[] messages = await queueClient.ReceiveMessagesAsync();
foreach (var message in messages)
{
    if (message.DequeueCount > 5)
    {
        // Dit is waarschijnlijk een poison message
        // Verplaats naar een andere queue of log voor onderzoek
        Console.WriteLine($"Poison message detected: {message.MessageId}");
        
        // Verwijder het poison message
        await queueClient.DeleteMessageAsync(message.MessageId, message.PopReceipt);
    }
}
```

### Batch Processing

Voor efficiency kun je meerdere messages tegelijk ontvangen en verwerken:

```cs
// Ontvang tot 32 messages (maximum)
QueueMessage[] messages = await queueClient.ReceiveMessagesAsync(maxMessages: 32);

foreach (var message in messages)
{
    try
    {
        // Verwerk het message
        Console.WriteLine($"Processing: {message.Body}");
        
        // Verwijder na succesvolle processing
        await queueClient.DeleteMessageAsync(message.MessageId, message.PopReceipt);
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error processing message {message.MessageId}: {ex.Message}");
        // Het message wordt automatisch weer zichtbaar na visibility timeout
    }
}
```

## Vergelijking: Queue Storage vs Service Bus

| Feature | Queue Storage | Service Bus Queues |
|---------|---------------|-------------------|
| **Message size** | Tot 64 KB | Tot 256 KB (Standard), 100 MB (Premium) |
| **Queue size** | Miljoenen messages | Beperkt door quota |
| **TTL** | Max 7 dagen | Onbeperkt (default geen expiration) |
| **Ordering** | Geen garantie | FIFO met sessions |
| **Duplicate detection** | Nee | Ja |
| **Transactions** | Nee | Ja |
| **Polling** | Vereist | Long-polling ondersteund |
| **Price** | Zeer goedkoop | Duurder |
| **Best voor** | Simpele queuing, grote volumes | Enterprise messaging, complexe routing |

## Belangrijke Exam Tips

1. ✅ **Message size**: Maximaal 64 KB per message
2. ✅ **TTL**: Default 7 dagen, maximum 7 dagen, -1 voor geen expiration
3. ✅ **Visibility timeout**: Default 30 seconden, range 0-7 dagen
4. ✅ **PopReceipt**: Vereist voor UpdateMessageAsync en DeleteMessageAsync
5. ✅ **ReceiveMessagesAsync**: Maakt message tijdelijk onzichtbaar (default 30s)
6. ✅ **PeekMessagesAsync**: Bekijk messages zonder ze uit queue te verwijderen (max 32)
7. ✅ **DequeueCount**: Detecteer poison messages door hoge dequeue count
8. ✅ **CreateIfNotExistsAsync**: Maak queue als deze niet bestaat
9. ✅ **Queue naam**: Moet volledig lowercase zijn
10. ✅ **Two-step delete**: ReceiveMessagesAsync → verwerk → DeleteMessageAsync
11. ✅ **ApproximateMessagesCount**: Geschat aantal messages, kan hoger zijn dan werkelijk
12. ✅ **URL formaat**: https://{account}.queue.core.windows.net/{queue}
13. ✅ **vs Service Bus**: Queue Storage voor simpliciteit/volume, Service Bus voor enterprise features
14. ✅ **Batch processing**: Max 32 messages per ReceiveMessagesAsync call
15. ✅ **Storage account capacity**: Miljoenen messages mogelijk tot account limiet
