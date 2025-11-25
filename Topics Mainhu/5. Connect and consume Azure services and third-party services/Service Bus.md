# [Azure Service Bus](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview)

Azure Service Bus is een volledig beheerde enterprise messaging service met message queues en publish-subscribe topics. Het wordt gebruikt voor het ontkoppelen van applicaties en services van elkaar, biedt een betrouwbare en veilige platform voor asynchrone data en state transfer.

## [Premium vs Standard](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-premium-messaging)

- **Premium Tier**: Resource isolation op CPU en memory niveau zorgt ervoor dat elke workload in isolatie draait. Deze resource container heet een **messaging unit**. Aan elk premium namespace is minstens één messaging unit toegewezen. Biedt voorspelbare performance, hogere throughput, en ondersteunt grotere message sizes (tot 100 MB).

- **Standard Tier**: Shared resources, geen gegarandeerde performance, message size tot 256 KB. Best voor ontwikkeling en testen, of low-throughput scenarios.

## [Service Bus Queues](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-queues-topics-subscriptions#queues)

Queues bieden **First In, First Out (FIFO)** message delivery aan een of meer concurrerende consumers. Messages worden doorgaans ontvangen en verwerkt door receivers in de volgorde waarin ze aan de queue zijn toegevoegd. Slechts één message consumer ontvangt en verwerkt elk message.

### Receive modes

Service Bus queues ondersteunen twee modes voor het ontvangen van messages:

#### Receive and delete

De broker markeert het message als consumed zodra de request van de consumer wordt ontvangen en retourneert het naar de consumer applicatie. Dit is het eenvoudigste model maar vereist dat de applicatie het verlies van messages kan tolereren als er een failure optreedt.

#### Peek lock (default)

De receive operatie wordt two-stage, wat het mogelijk maakt om applicaties te ondersteunen die missing messages niet kunnen tolereren:

1. Vindt het volgende message, **locks** het om te voorkomen dat andere consumers het ontvangen, en retourneert het message naar de applicatie.
2. Nadat de applicatie klaar is met het verwerken van het message, vraagt het de Service Bus service om de tweede stage van het receive proces te voltooien. Dan **markeert** de service het message als consumed.

Als de applicatie crasht voordat de completion request wordt uitgegeven, levert Service Bus het message opnieuw af zodra de applicatie herstart. Dit wordt ook wel **At-Least-Once** processing genoemd.

Als de applicatie het message niet verwerkt binnen de **lock timeout** (standaard 60 seconden), stelt Service Bus het message automatisch weer beschikbaar.

### Dead-letter queue

Elke Service Bus queue of topic subscription heeft een geassocieerde **dead-letter queue (DLQ)**. Een DLQ houdt messages vast die niet aan een receiver kunnen worden geleverd of niet kunnen worden verwerkt.

Messages worden verplaatst naar de DLQ in de volgende scenarios:

- Message **Time-to-Live (TTL)** verloopt
- Het maximale **delivery count** is overschreden
- De applicatie expliciet **dead-letters** het message

Messages in de DLQ kunnen worden onderzocht en opnieuw verwerkt.

```cs
// Ontvang messages van de dead-letter queue
var client = new ServiceBusClient(connectionString);
var receiver = client.CreateReceiver(queueName, new ServiceBusReceiverOptions
{
    SubQueue = SubQueue.DeadLetter
});

await foreach (ServiceBusReceivedMessage message in receiver.ReceiveMessagesAsync())
{
    // Verwerk het dead-lettered message
    Console.WriteLine($"Dead Letter Reason: {message.DeadLetterReason}");
    Console.WriteLine($"Dead Letter Error: {message.DeadLetterErrorDescription}");
}
```

## [Service Bus Topics en Subscriptions](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-queues-topics-subscriptions#topics-and-subscriptions)

Topics en subscriptions bieden een **one-to-many** communicatievorm in een **publish/subscribe** pattern. Dit is nuttig voor scaling naar grote aantallen ontvangers. Elk gepubliceerd message wordt beschikbaar gemaakt aan elke subscription geregistreerd bij het topic. Publishers sturen messages naar een topic en subscribers ontvangen messages van hun subscriptions.

### Subscriptions

Een topic subscription lijkt op een **virtual queue** die kopieën ontvangt van messages die naar het topic zijn gestuurd. Subscribers ontvangen messages van een subscription identiek aan hoe ze messages van een queue ontvangen.

### Subscription filters

Je kunt **filters** definiëren op subscriptions om te bepalen welke messages de subscription ontvangt:

- **SQL filters**: Voorwaardes gebaseerd op message properties en applicatie properties
- **Boolean filters**: 
  - `TrueFilter`: Selecteert alle messages (default)
  - `FalseFilter`: Selecteert geen messages
- **Correlation filters**: Match tegen een set van voorwaardes voor een of meer message properties

```cs
// Maak een subscription met een SQL filter
await adminClient.CreateSubscriptionAsync(
    new CreateSubscriptionOptions(topicName, "HighPriority")
    {
        DefaultMessageTimeToLive = TimeSpan.FromDays(7)
    },
    new CreateRuleOptions("HighPriorityRule", new SqlRuleFilter("Priority = 'High'"))
);

// Maak een subscription met een correlation filter
await adminClient.CreateSubscriptionAsync(
    new CreateSubscriptionOptions(topicName, "USOrders"),
    new CreateRuleOptions("USOrdersRule", new CorrelationRuleFilter { Subject = "orders", ApplicationProperties = { ["country"] = "US" } })
);
```

## [Message Sessions](https://learn.microsoft.com/en-us/azure/service-bus-messaging/message-sessions)

Sessions garanderen **FIFO (First-In-First-Out)** message ordering en bieden **stateful processing** van message streams. Om sessions te gebruiken, moet je `RequiresSession` instellen op `true` bij het aanmaken van een queue of subscription.

Elke message moet een **SessionId** hebben. Messages met dezelfde SessionId vormen een session en worden in volgorde verwerkt.

```cs
// Stuur messages met een session ID
var sender = client.CreateSender(queueName);
for (int i = 0; i < 5; i++)
{
    var message = new ServiceBusMessage($"Message {i}")
    {
        SessionId = "session-123"
    };
    await sender.SendMessageAsync(message);
}

// Ontvang messages van een specifieke session
var sessionReceiver = await client.AcceptSessionAsync(queueName, "session-123");
await foreach (var message in sessionReceiver.ReceiveMessagesAsync())
{
    Console.WriteLine($"Received: {message.Body}");
    await sessionReceiver.CompleteMessageAsync(message);
}
```

Use cases voor sessions:

- **Geordende verwerking**: Wanneer message volgorde binnen een groep belangrijk is
- **Stateful workflows**: Bijhouden van state over meerdere messages
- **Request-response patterns**: Correleren van requests met responses

## [Message Properties](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messages-payloads)

### System Properties (read-only)

- **MessageId**: Applicatie-gedefinieerde identifier voor het message
- **SequenceNumber**: Service Bus toegewezen uniek nummer
- **SessionId**: Voor session-enabled entities
- **TimeToLive**: Hoe lang het message geldig is
- **ScheduledEnqueueTime**: Wanneer het message beschikbaar moet worden

### Application Properties

Custom key-value pairs die je kunt gebruiken voor filtering en routing:

```cs
var message = new ServiceBusMessage("message body");
message.ApplicationProperties["Priority"] = "High";
message.ApplicationProperties["Region"] = "EU";
message.ApplicationProperties["CustomerId"] = 12345;
```

## Advanced Features

### Scheduled Messages

Je kunt messages plannen om op een later tijdstip beschikbaar te worden:

```cs
var message = new ServiceBusMessage("Delayed message");
var scheduleTime = DateTimeOffset.UtcNow.AddHours(2);
await sender.SendMessageAsync(message, scheduleTime);
```

### Auto-forwarding

Auto-forward laat je een queue of subscription chainen aan een andere queue of topic binnen dezelfde namespace:

```cs
var queueOptions = new CreateQueueOptions("sourceQueue")
{
    ForwardTo = "destinationQueue"
};
await adminClient.CreateQueueAsync(queueOptions);
```

### Transactions

Service Bus ondersteunt het groeperen van operaties tegen een enkele messaging entity binnen de scope van een transactie:

```cs
using var ts = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);
await sender.SendMessageAsync(message1);
await sender.SendMessageAsync(message2);
ts.Complete();
```

## Code Voorbeelden

### Stuur een message naar een queue

```cs
var client = new ServiceBusClient(connectionString);
var sender = client.CreateSender(queueName);

var message = new ServiceBusMessage("Hello, Service Bus!");
await sender.SendMessageAsync(message);
```

### Ontvang messages van een queue

```cs
var client = new ServiceBusClient(connectionString);
var receiver = client.CreateReceiver(queueName);

// Ontvang een batch van messages
IReadOnlyList<ServiceBusReceivedMessage> messages = await receiver.ReceiveMessagesAsync(maxMessages: 10);

foreach (var message in messages)
{
    Console.WriteLine($"Received: {message.Body}");
    // Complete het message om het van de queue te verwijderen
    await receiver.CompleteMessageAsync(message);
}
```

### Stuur messages naar een topic

```cs
var client = new ServiceBusClient(connectionString);
var sender = client.CreateSender(topicName);

var message = new ServiceBusMessage("Topic message");
message.ApplicationProperties["Category"] = "News";
await sender.SendMessageAsync(message);
```

### Ontvang messages van een subscription

```cs
var client = new ServiceBusClient(connectionString);
var receiver = client.CreateReceiver(topicName, subscriptionName);

await foreach (var message in receiver.ReceiveMessagesAsync())
{
    Console.WriteLine($"Received from subscription: {message.Body}");
    await receiver.CompleteMessageAsync(message);
}
```

## Belangrijke Exam Tips

1. ✅ **Queues vs Topics**: Queue = point-to-point (1 consumer), Topic = pub/sub (meerdere subscribers)
2. ✅ **Receive modes**: Peek-lock (default, at-least-once) vs Receive-and-delete (at-most-once)
3. ✅ **Dead-letter queue**: Voor messages die niet kunnen worden verwerkt, TTL verlopen, of max delivery count bereikt
4. ✅ **Message size**: Standard tier tot 256 KB, Premium tier tot 100 MB
5. ✅ **Sessions**: FIFO ordering, RequiresSession=true, gebruik SessionId voor geordende verwerking
6. ✅ **Lock timeout**: Default 60 seconden, message beschikbaar als processing niet compleet is
7. ✅ **Subscription filters**: SQL filters, Boolean filters (True/False), Correlation filters
8. ✅ **Premium tier**: Resource isolation, messaging units, voorspelbare performance
9. ✅ **Scheduled messages**: ScheduledEnqueueTime voor delayed delivery
10. ✅ **Auto-forwarding**: Chain queues/subscriptions binnen dezelfde namespace
11. ✅ **Transactions**: Groepeer operaties binnen een TransactionScope
12. ✅ **TTL (Time-to-Live)**: Message expiration, daarna naar DLQ
13. ✅ **Complete message**: Verwijdert message van queue/subscription na processing
14. ✅ **Application properties**: Custom key-value pairs voor filtering en routing
15. ✅ **System properties**: MessageId, SequenceNumber, SessionId (read-only)
