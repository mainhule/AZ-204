# [Azure Cache for Redis](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-overview)

Azure Cache for Redis is een in-memory data store gebaseerd op Redis software. Het biedt een veilige, toegewijde Redis cache die wordt beheerd door Microsoft, gehost op Azure, en toegankelijk voor applicaties binnen en buiten Azure.

## Belangrijkste Use Cases

- **Data caching**: Verminder database load door veelgebruikte data in het geheugen op te slaan.
- **Content caching**: Cache statische content zoals headers, footers, banners.
- **Session store**: Bewaar gebruikerssessie data voor web applicaties (shopping carts, user history).
- **Job en message queuing**: Gebruik Redis lists voor asynchrone operaties.
- **Distributed transactions**: Ondersteun batch operaties als atomic transactions.

## Service Tiers

| Tier                | Beschrijving                                                                                      | Use Case                                    |
| ------------------- | ------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| **Basic** (C0-C6)   | Enkele VM, geen SLA. ⭐: Development/testing                                                      | 🧪 Dev/test, niet-kritische workloads       |
| **Standard** (C0-C6)| Twee VMs (primary/replica), hoge beschikbaarheid, 99.9% SLA                                      | ⏺️ Productie workloads                      |
| **Premium** (P1-P5) | Betere prestaties, grotere workloads, disaster recovery, enhanced security. 99.95% SLA           | ⚡ High-performance, enterprise features     |
| **Enterprise** (E10+)| Redis Enterprise, ondersteuning voor RediSearch, RedisBloom, RedisTimeSeries. 99.99% SLA        | 💎 Mission-critical, advanced modules       |
| **Enterprise Flash**| Combineert DRAM en flash storage voor kostenefficiëntie bij grote datasets                       | 💰 Grote datasets met lagere kosten         |

### Premium Tier Features

- **Data persistence**: RDB (snapshot) of AOF (append-only file) naar Azure Storage.
- **Redis cluster**: Automatische data sharding over meerdere nodes (maximaal 10 shards).
- **Geo-replication**: Koppel twee Premium cache instances voor disaster recovery.
- **Virtual Network (VNet)**: Deploy cache binnen je VNet voor complete netwerkisolatie.
- **Import/Export**: Importeer of exporteer Redis database (RDB) bestanden naar/van Azure Storage.

## Data Persistence

Alleen beschikbaar in **Premium** tier.

### RDB (Redis Database)

- Neemt snapshots op geconfigureerde intervallen
- Gebruikt minder disk I/O
- Snellere restart times
- Mogelijk dataverlies bij crash tussen snapshots

```sh
# Configuratie opties
15 minuten én minimaal 1000 writes
30 minuten én minimaal 100 writes
60 minuten én minimaal 1 write
```

### AOF (Append Only File)

- Logt elke write operatie
- Minimaal dataverlies (max 1 seconde)
- Grotere bestanden, langzamere restart
- Meer disk I/O

## Redis Clustering

Beschikbaar in **Premium**, **Enterprise** en **Enterprise Flash** tiers.

- Automatische data partitionering over meerdere Redis nodes
- Ondersteunt maximaal **10 shards** (Premium) of **120 shards** (Enterprise)
- Elke shard heeft een primary met optionele replicas voor high availability
- Data wordt verdeeld met **hash slots** (16384 slots totaal)

```cs
// Connection string met clustering
var connection = ConnectionMultiplexer.Connect("cachename.redis.cache.windows.net:6380,password=...,ssl=True,abortConnect=False");
```

## Geo-Replication

Alleen **Premium** tier.

- Koppel twee cache instances in verschillende regio's
- **Primary cache**: Alle read/write operaties
- **Secondary cache**: Read-only replica
- Asynchroon gerepliceerd (mogelijk dataverlies bij failover)
- Handmatige unlink vereist om secondary te promoveren tot primary

### Active Geo-Replication

**Enterprise** tier ondersteunt active geo-replication:

- Multi-master replicatie
- Beide caches accepteren writes
- Conflict resolution policies beschikbaar

## Caching Patterns

### Cache-Aside Pattern (Lazy Loading)

Applicatie controleert eerst de cache; bij een miss wordt data uit de database geladen en in de cache gezet.

```cs
public async Task<Product> GetProductAsync(int productId)
{
    var cache = connection.GetDatabase();
    string key = $"product:{productId}";
    
    // Probeer uit cache te halen
    var cachedProduct = await cache.StringGetAsync(key);
    
    if (!cachedProduct.IsNull)
    {
        return JsonSerializer.Deserialize<Product>(cachedProduct);
    }
    
    // Cache miss: haal uit database
    var product = await _dbContext.Products.FindAsync(productId);
    
    // Sla op in cache met expiration
    await cache.StringSetAsync(key, JsonSerializer.Serialize(product), TimeSpan.FromMinutes(10));
    
    return product;
}
```

### Write-Through Pattern

Data wordt tegelijkertijd naar de database én cache geschreven.

```cs
public async Task UpdateProductAsync(Product product)
{
    // Update database
    _dbContext.Products.Update(product);
    await _dbContext.SaveChangesAsync();
    
    // Update cache
    var cache = connection.GetDatabase();
    string key = $"product:{product.Id}";
    await cache.StringSetAsync(key, JsonSerializer.Serialize(product), TimeSpan.FromMinutes(10));
}
```

### Read-Through Pattern

De cache layer handelt database reads automatisch af.

### Write-Behind Pattern (Write-Back)

Writes gaan eerst naar de cache; database updates gebeuren asynchroon (batched).

## Verbinden met Azure Cache for Redis

### Connection String

```
cachename.redis.cache.windows.net:6380,password=primary_key_here,ssl=True,abortConnect=False
```

- **ssl=True**: Verplicht voor Azure Cache for Redis (non-SSL port is standaard uitgeschakeld)
- **abortConnect=False**: Voorkomt exception bij initiële verbindingsproblemen

### StackExchange.Redis Client

```cs
using StackExchange.Redis;

// Singleton connection (best practice)
private static Lazy<ConnectionMultiplexer> lazyConnection = new Lazy<ConnectionMultiplexer>(() =>
{
    string connectionString = Environment.GetEnvironmentVariable("REDIS_CONNECTION_STRING");
    return ConnectionMultiplexer.Connect(connectionString);
});

public static ConnectionMultiplexer Connection => lazyConnection.Value;

// Gebruik
var cache = Connection.GetDatabase();

// String operaties
await cache.StringSetAsync("key", "value");
var value = await cache.StringGetAsync("key");

// Met expiration
await cache.StringSetAsync("key", "value", TimeSpan.FromMinutes(5));

// Hash operaties (voor objecten)
await cache.HashSetAsync("user:1001", new HashEntry[] {
    new HashEntry("name", "John"),
    new HashEntry("email", "john@example.com")
});

// List operaties
await cache.ListLeftPushAsync("mylist", "item1");
await cache.ListRightPushAsync("mylist", "item2");

// Set operaties
await cache.SetAddAsync("myset", "member1");
var exists = await cache.SetContainsAsync("myset", "member1");

// Sorted Set operaties
await cache.SortedSetAddAsync("leaderboard", "player1", 100);
var topPlayers = await cache.SortedSetRangeByRankAsync("leaderboard", 0, 9, Order.Descending);
```

## Access Keys en Security

### Access Keys

- **Primary key**: Hoofd toegangssleutel
- **Secondary key**: Voor key rotation zonder downtime

```sh
# Regenerate keys
az redis regenerate-keys --name MyRedisCache --resource-group MyResourceGroup --key-type Primary
```

### Key Rotation Process

1. Update applicatie configuratie om secondary key te gebruiken
2. Regenerate primary key
3. Update applicatie configuratie om nieuwe primary key te gebruiken
4. Regenerate secondary key

### Firewall Rules

Configureer IP whitelist voor toegangscontrole:

```sh
az redis firewall-rules create \
    --name MyFirewallRule \
    --resource-group MyResourceGroup \
    --cache-name MyRedisCache \
    --start-ip 192.168.1.1 \
    --end-ip 192.168.1.10
```

### VNet Integration (Premium)

- Deploy cache binnen een Azure Virtual Network
- Complete netwerkisolatie
- Toegang alleen via resources binnen dezelfde VNet of via VPN/ExpressRoute

## Best Practices

### Connection Management

- **Hergebruik connections**: Maak een singleton `ConnectionMultiplexer` instance
- **Vermijd frequente connects/disconnects**: Connections zijn duur
- **Gebruik `abortConnect=False`**: Voorkom crashes bij verbindingsproblemen

### Data Expiration

```cs
// Absolute expiration
await cache.StringSetAsync("key", "value", TimeSpan.FromMinutes(10));

// Sliding expiration (vernieuw bij elke access)
await cache.StringGetAsync("key", CommandFlags.None);
await cache.KeyExpireAsync("key", TimeSpan.FromMinutes(10));
```

### Error Handling

```cs
try
{
    var value = await cache.StringGetAsync("key");
    if (value.IsNull)
    {
        // Cache miss: haal data van origin
    }
}
catch (RedisConnectionException ex)
{
    // Fallback naar database bij cache failure
    _logger.LogError(ex, "Redis connection failed");
    // Haal direct uit database
}
```

### Monitoring

Key metrics in Azure Portal:

- **Cache hits/misses**: Hit ratio moet > 80% zijn
- **Connected clients**: Monitor voor connection leaks
- **Server load**: CPU usage (> 80% = overbelast)
- **Memory usage**: Voorkom eviction door memory pressure
- **Operations per second**: Throughput monitoring

### Eviction Policies

Wat gebeurt er wanneer de cache vol is?

```sh
# Configureer via Azure Portal of CLI
az redis update --name MyRedisCache --resource-group MyResourceGroup --set redisConfiguration.maxmemory-policy=allkeys-lru
```

Eviction policies:

- **noeviction**: Return errors bij vol geheugen (default)
- **allkeys-lru**: Remove least recently used keys
- **allkeys-lfu**: Remove least frequently used keys
- **volatile-lru**: Remove LRU keys met expiration set
- **volatile-lfu**: Remove LFU keys met expiration set
- **allkeys-random**: Remove random keys
- **volatile-random**: Remove random keys met expiration set
- **volatile-ttl**: Remove keys met kortste TTL

## Scaling

### Vertical Scaling (Scale Up/Down)

- Wijzig cache tier (bijv. van C1 naar C3)
- Downtime mogelijk bij sommige tier changes
- Niet mogelijk: Basic ↔ Premium, Premium → Standard/Basic

### Horizontal Scaling (Scale Out)

- Alleen **Premium** tier met clustering
- Voeg of verwijder shards
- Data wordt automatisch gerebalanceerd
- Geen downtime bij scale out/in

```sh
az redis update --name MyRedisCache --resource-group MyResourceGroup --shard-count 3
```

## Common Exam Scenarios

### Scenario 1: Session State voor Web App

**Vraag**: Een web app moet gebruikerssessies opslaan die toegankelijk zijn voor alle instances.

**Oplossing**: Gebruik Azure Cache for Redis als session state provider.

```cs
// In Program.cs / Startup.cs
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration["Redis:ConnectionString"];
    options.InstanceName = "SessionCache";
});

builder.Services.AddSession(options =>
{
    options.IdleTimeout = TimeSpan.FromMinutes(30);
    options.Cookie.HttpOnly = true;
    options.Cookie.IsEssential = true;
});
```

### Scenario 2: Database Query Caching

**Vraag**: Verlaag database load voor veelgevraagde product data.

**Oplossing**: Implementeer Cache-Aside pattern met expiration.

### Scenario 3: High Availability Required

**Vraag**: Cache moet 99.9% beschikbaarheid garanderen.

**Oplossing**: Gebruik minimaal **Standard** tier (niet Basic) met automatic failover.

### Scenario 4: Multi-Region Deployment

**Vraag**: Applicatie draait in meerdere Azure regio's met disaster recovery vereisten.

**Oplossing**: Gebruik **Premium** tier met geo-replication of **Enterprise** tier met active geo-replication.

### Scenario 5: Large Dataset (>50 GB)

**Vraag**: Cache moet 100GB data bevatten met kostenoptimalisatie.

**Oplossing**: Gebruik **Enterprise Flash** tier voor combinatie van DRAM + flash storage.

## CLI Commands

```sh
# Maak een Redis cache aan
az redis create \
    --name MyRedisCache \
    --resource-group MyResourceGroup \
    --location westeurope \
    --sku Standard \
    --vm-size C1

# Haal connection string op
az redis list-keys --name MyRedisCache --resource-group MyResourceGroup

# Update cache configuratie
az redis update \
    --name MyRedisCache \
    --resource-group MyResourceGroup \
    --set enableNonSslPort=false

# Configureer firewall
az redis firewall-rules create \
    --name AllowOfficeIP \
    --cache-name MyRedisCache \
    --resource-group MyResourceGroup \
    --start-ip 203.0.113.0 \
    --end-ip 203.0.113.255

# Verwijder cache
az redis delete --name MyRedisCache --resource-group MyResourceGroup
```

## Belangrijke Exam Tips

1. **SSL is verplicht** voor Azure Cache for Redis (non-SSL port is standaard disabled)
2. **Data persistence** is alleen beschikbaar in Premium tier
3. **Geo-replication** vereist twee Premium tier caches in verschillende regio's
4. **VNet integration** is alleen mogelijk met Premium tier
5. **Clustering** is beschikbaar in Premium en Enterprise tiers
6. **ConnectionMultiplexer** moet worden hergebruikt (singleton pattern)
7. **Standard tier** biedt 99.9% SLA, Basic heeft geen SLA
8. Key rotation: gebruik secondary key tijdens regeneratie van primary key
9. **Cache-Aside pattern** is het meest gebruikte caching pattern
10. Monitor **cache hit ratio** om effectiviteit te meten (> 80% is goed)
