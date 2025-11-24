# [Azure Application Insights](https://docs.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)

Application Insights, een extensie van Azure Monitor, is een uitgebreide Application Performance Monitoring (APM) tool. Het biedt unieke functies die helpen bij het monitoren van applicaties gedurende hun hele levenscyclus, van ontwikkeling en testen tot productie.

Opmerking: Application Insights legt automatisch de Session Id vast, dus het is niet nodig om dit handmatig te doen.

| Feature                                         | Beschrijving                                                                                                     |
| ----------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Live Metrics                                    | Real-time monitoring zonder de host te beïnvloeden. Voor directe inzichten tijdens kritieke deployments.         |
| Availability (Synthetic Transaction Monitoring) | Test endpoints op beschikbaarheid en responsiviteit. Om uptime en SLA's te waarborgen.                          |
| GitHub of Azure DevOps integratie               | Maak GitHub of Azure DevOps work items in de context van Application Insights data.                              |
| Usage                                           | Volgt populaire features en gebruikersinteracties.                                                               |
| Smart Detection                                 | Automatische detectie van fouten en anomalieën door proactieve telemetry analyse.                                |
| Application Map                                 | Top-down overzicht van app architectuur met health indicatoren.                                                  |
| Distributed Tracing                             | Zoek en visualiseer een end-to-end flow van een bepaalde uitvoering of transactie.                               |

Application Insights monitort verschillende aspecten van de prestaties en gezondheid van je applicatie. Het verzamelt Metrics en Telemetry data, waaronder:

- **Request rates, response times en failure rates**: Identificeer populaire pagina's, piekgebruikstijden en gebruikerslocaties. Monitor paginaprestaties en detecteer resource problemen tijdens hoge request loads.
- **Dependency rates, response times en failure rates**: Controleer of externe services vertragingen veroorzaken.
- **Exceptions**: Analyseer geaggregeerde statistieken of specifieke instanties, bekijk stack traces en gerelateerde requests. Rapporteert zowel server als browser exceptions.
- **Page views en load performance**: Informatie gerapporteerd door de browsers van gebruikers.
- **AJAX calls**: Monitor rates, response times en failure rates voor AJAX calls van webpagina's.
- **User en session counts**: Houd gebruikers- en sessie-aantallen bij.
- **Performance counters**: Monitor CPU, geheugen en netwerkgebruik op Windows of Linux server machines.
- **Host diagnostics**: Verzamel diagnostische informatie van Docker of Azure.
- **Diagnostic trace logs**: Correleer trace events met requests door logs van je app te verzamelen.
- **Custom events en metrics**: Maak je eigen events en metrics in de client of server code om specifieke business events te volgen, zoals verkochte items of gewonnen games.

Hier zijn verschillende methoden om te beginnen met het monitoren en analyseren van je app's prestaties:

- **Run time**: Gebruik Application Insights met je web app op de server, ideaal voor reeds gedeployde apps en vereist geen code updates.
- **Development time**: Voeg Application Insights toe aan je code. Dit maakt aangepaste telemetry collectie mogelijk en uitgebreidere data verzameling.
- **Web page instrumentation**: Volg page views, AJAX en andere client-side activiteiten.
- **Mobile app analysis**: Gebruik Visual Studio App Center om mobile app gebruik te bestuderen.
- **Availability tests**: Ping regelmatig je website vanaf onze servers om beschikbaarheid te testen.

## Metrics

**Log-based metrics**: Biedt grondige data analyse en diagnostiek. ⭐: je hebt een complete set van events nodig ❌: high-volume apps die sampling / filtering vereisen.

**Standard metrics** zijn tijdreeks data **vooraf geaggregeerd** door ofwel SDK (versie heeft geen invloed op nauwkeurigheid) of backend (betere nauwkeurigheid), geoptimaliseerd voor snelle queries. ⭐: dashboards en real-time alerts, use cases _die sampling of filtering vereisen_.

Je kunt schakelen tussen deze metrics types met behulp van de namespace selector van de metrics explorer.

[Ondersteunde Metrics](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-metrics/metrics-index)

### [Sampling](https://learn.microsoft.com/en-us/azure/azure-monitor/app/sampling)

Vermindert dataverkeer en kosten terwijl de analyse nauwkeurig blijft. Helpt data limieten te voorkomen en maakt diagnostiek gemakkelijker. Hoge sampling rates (> 60%) kunnen de nauwkeurigheid van log-based metrics beïnvloeden. Vooraf geaggregeerde metrics in SDKs lossen dit probleem op, maar te veel filtering kan alerts missen.

**Simpel uitgelegd**: Stel je hebt een drukke webshop. In plaats van elke klik en elke paginaweergave op te slaan, kies je ervoor om maar 10% van alle gebeurtenissen te bewaren. Zo houd je de kosten laag, maar zie je nog steeds trends en problemen.

#### Soorten Sampling

- **Adaptive sampling**: Automatische sampling waarbij Application Insights het aantal te verzamelen telemetrie dynamisch aanpast op basis van de hoeveelheid verkeer. Bij veel verkeer wordt meer data weggelaten, bij weinig verkeer wordt meer data opgeslagen. Dit voorkomt overbelasting en hoge kosten. Standaard ingeschakeld, gebruikt in Azure Functions.

- **Fixed-rate sampling**: Hierbij stel je zelf een vast percentage in van de telemetrie die wordt opgeslagen (bijvoorbeeld 10%). Ongeacht het verkeer wordt altijd dat percentage bewaard. ⭐: synchroniseren van client en server data voor onderzoeken van gerelateerde events.

- **Ingestion sampling**: Een vorm van sampling waarbij slechts een deel van de telemetriegegevens wordt opgeslagen op het moment dat ze binnenkomen in Application Insights. Bepaalt bij binnenkomst welke data wordt opgeslagen (kan adaptief of vast zijn). Dit betekent dat niet alle data wordt bewaard, maar bijvoorbeeld slechts 10% van alle inkomende events, traces of requests. Gebruik als je maandelijkse limieten bereikt, of te veel data krijgt, of een oudere SDK gebruikt.

- **Configure sampling overrides**: Hiermee kun je uitzonderingen instellen op de sampling-regels. Bijvoorbeeld: bepaalde typen telemetrie (zoals errors of requests van een specifieke gebruiker) altijd opslaan, ongeacht de ingestelde sampling.

Voor web apps, om custom events te groeperen, gebruik dezelfde `OperationId` waarde.

#### Configuratie van sampling

```cs
var builder = TelemetryConfiguration.Active.DefaultTelemetrySink.TelemetryProcessorChainBuilder;

// Schakel AdaptiveSampling in om het totale telemetry volume op 5 items per seconde te houden.
builder.UseAdaptiveSampling(maxTelemetryItemsPerSecond:5);

// Fixed rate sampling
builder.UseSampling(10.0); // percentage

// Als je andere telemetry processors hebt:
builder.Use((next) => new AnotherProcessor(next));
```

## Telemetry Pipeline Componenten

De telemetry pipeline bestaat uit verschillende componenten die je helpen om data aan te passen voordat het naar Application Insights wordt verstuurd.

### Telemetry Initializer

**Wat doet het?** Voegt extra informatie toe aan elk telemetry item voordat het wordt verstuurd.

**Simpel uitgelegd**: Je wilt bij elke foutmelding in je app ook de gebruikersnaam meesturen. Met een telemetry initializer voeg je die gebruikersnaam toe aan elk telemetry-item voordat het naar Application Insights gaat.

**Voorbeeld**:
```cs
public class CustomTelemetryInitializer : ITelemetryInitializer
{
    public void Initialize(ITelemetry telemetry)
    {
        // Voeg gebruikersnaam toe aan alle telemetry
        telemetry.Context.User.Id = "jan.jansen@bedrijf.nl";
        
        // Voeg custom property toe
        if (telemetry is ISupportProperties propTelemetry)
        {
            propTelemetry.Properties["Environment"] = "Production";
            propTelemetry.Properties["ApplicationVersion"] = "2.1.0";
        }
    }
}

// Registreer in Startup.cs
services.AddApplicationInsightsTelemetry();
services.AddSingleton<ITelemetryInitializer, CustomTelemetryInitializer>();
```

### Telemetry Processor

**Wat doet het?** Filtert of wijzigt telemetry data voordat het wordt verstuurd. Je kunt bepaalde data eruit halen of aanpassen.

**Simpel uitgelegd**: Je wilt geen telemetrie van testgebruikers opslaan. Met een telemetry processor filter je alle data van testgebruikers eruit, zodat die niet in Application Insights terechtkomt.

**Voorbeeld**:
```cs
public class FilterTestUsersTelemetryProcessor : ITelemetryProcessor
{
    private ITelemetryProcessor Next { get; set; }

    public FilterTestUsersTelemetryProcessor(ITelemetryProcessor next)
    {
        this.Next = next;
    }

    public void Process(ITelemetry item)
    {
        // Filter telemetry van testgebruikers
        if (item is RequestTelemetry request && 
            request.Context.User.Id.StartsWith("test_"))
        {
            // Blokkeer deze telemetry (stuur niet door)
            return;
        }

        // Filter health check requests
        if (item is RequestTelemetry req && 
            req.Url.AbsolutePath.Contains("/health"))
        {
            return;
        }

        // Stuur door naar de volgende processor in de chain
        this.Next.Process(item);
    }
}

// Registreer in Startup.cs
services.AddApplicationInsightsTelemetryProcessor<FilterTestUsersTelemetryProcessor>();
```

### Telemetry Channel

**Wat doet het?** Bepaalt hoe de telemetry data wordt verstuurd naar Application Insights.

**Simpel uitgelegd**: Je app verzamelt telemetrie en stuurt die via het telemetry channel naar Application Insights. Je kunt bijvoorbeeld instellen dat de data eerst lokaal wordt opgeslagen en pas later wordt verstuurd als er weer internet is.

**Twee hoofdtypen**:

1. **InMemoryChannel**: Stuurt direct, geen persistentie (data verloren bij crash)
2. **ServerTelemetryChannel**: Buffert lokaal, stuurt asynchroon, betrouwbaarder

**Voorbeeld**:
```cs
// Configureer channel in Program.cs
services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = "InstrumentationKey=...";
});

services.Configure<TelemetryConfiguration>(config =>
{
    // Gebruik ServerTelemetryChannel met lokale opslag
    var channel = new ServerTelemetryChannel
    {
        StorageFolder = @"C:\TelemetryCache",
        MaxTelemetryBufferCapacity = 1000,
        // Verstuur elke 30 seconden of bij 500 items
        FlushIntervalInMilliseconds = 30000
    };
    
    config.TelemetryChannel = channel;
});
```

### Kortom

- **Sampling** = minder data opslaan (kostenreductie)
- **Initializer** = extra info toevoegen (verrijken)
- **Processor** = data filteren of verwijderen (opschonen)
- **Channel** = data verzenden (transport)


## [Custom events en metrics](https://learn.microsoft.com/en-us/azure/azure-monitor/app/api-custom-events-metrics#getmetric)

Gebruik [`GetMetric()`](https://learn.microsoft.com/en-us/azure/azure-monitor/app/get-metric) in plaats van `TrackMetric()`. `GetMetric()` handelt **pre-aggregation** af, waardoor kosten en prestatieproblemen geassocieerd met ruwe telemetry worden verminderd. Het vermijdt sampling, wat zorgt voor betrouwbare alerts. Het volgen van metrics op een gedetailleerd niveau kan leiden tot verhoogde kosten, netwerkverkeer en throttling risico's. `GetMetric()` lost deze zorgen op door samengevatte data elke minuut te verzenden.

```cs
TelemetryConfiguration configuration = TelemetryConfiguration.CreateDefault();
configuration.InstrumentationKey = "your-instrumentation-key-here";
var telemetry = new TelemetryClient(configuration);

// Stel eigenschappen in zoals UserId en DeviceId om de machine te identificeren.
// Deze informatie wordt gekoppeld aan alle events die de instance verzendt.
telemetry.Context.User.Id = "...";
telemetry.Context.Device.Id = "...";

// Monitort gebruikspatronen en stuurt data naar Custom Events voor zoeken.
// Het benoemt events en bevat string properties en numerieke metrics.
telemetry.TrackEvent("WinGame");

// GetMetric: vastleggen van lokaal vooraf geaggregeerde metrics voor .NET en .NET Core applicaties

// TrackMetric: niet de voorkeursmethode voor het verzenden van metrics, maar kan worden gebruikt als je je eigen pre-aggregation logica implementeert
var sample = new MetricTelemetry();
sample.Name = "queueLength";
sample.Sum = 42.3;
telemetry.TrackMetric(sample);

// Volg page views op meer of andere tijden
telemetry.TrackPageView("GameReviewPage");

// Volg de response times en success rates van calls naar een extern stuk code
var success = false;
var startTime = DateTime.UtcNow;
var timer = System.Diagnostics.Stopwatch.StartNew();

try
{
    // Code die mogelijk een exception kan gooien
    success = true;
}
catch (Exception ex)
{
    // Stuur exceptions naar Application Insights
    telemetry.TrackException(ex);

    // Log exceptions naar een diagnostische trace listener (Trace.aspx).
    Trace.TraceError(ex.Message);
}
finally
{
    timer.Stop();
    // TrackDependency: Volgt de prestaties van externe dependencies die niet automatisch door de SDK worden verzameld.
    // Gebruik het om response times voor databases of externe services te meten.
    // Stuur data naar Dependency Tracking in Application Insights
    telemetry.TrackDependency("DependencyType", "myDependency", "myCall", startTime, timer.Elapsed, success);
}

// Diagnosticeer problemen door een "breadcrumb trail" naar Application Insights te sturen
// Laat je langere data zoals POST informatie versturen.
telemetry.TrackTrace("Some message", SeverityLevel.Warning);

// Event log: gebruik ILogger of een class die erft van EventSource.

// Stuur data onmiddellijk, in plaats van te wachten op het volgende fixed-interval versturen
telemetry.Flush();
```

Lees meer: [Dependency tracking in Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/asp-net-dependencies)

## [Usage analysis](https://learn.microsoft.com/en-us/azure/azure-monitor/app/usage-overview)

- [User, session en event analysis](https://learn.microsoft.com/en-us/azure/azure-monitor/app/usage-segmentation)

  - **Users tool**: Telt unieke app gebruikers per browser/machine.
  - **Sessions tool**: Volgt feature gebruik per sessie; reset na 30min inactiviteit of 24u gebruik.
  - **Events tool**: Meet page views en custom events zoals clicks.

- [Funnels](https://learn.microsoft.com/en-us/azure/azure-monitor/app/usage-funnels): Voor lineaire, stap-voor-stap processen. Volg hoe gebruikers door verschillende stadia van je web applicatie bewegen, bijvoorbeeld hoeveel gebruikers van de home page naar het aanmaken van een ticket gaan. Gebruik funnels om te identificeren waar gebruikers mogelijk stoppen of je app verlaten, wat helpt om effectieve gebieden en verbeterpunten te begrijpen.
- [User Flows](https://learn.microsoft.com/en-us/azure/azure-monitor/app/usage-flows): Voor het begrijpen van complex, vertakkend gebruikersgedrag. Helpt je analyseren hoe gebruikers navigeren tussen pagina's en features van je web app. Het kan vragen beantwoorden zoals waar gebruikers naartoe gaan na het bezoeken van een pagina, waar ze je site verlaten, of ze dezelfde actie veel keer herhalen.
- [Cohorts](https://learn.microsoft.com/en-us/azure/azure-monitor/app/usage-cohorts): Groepeer en analyseer sets van gebruikers, sessies, events of operaties die iets gemeenschappelijk hebben. Bijvoorbeeld, je kunt een cohort maken van gebruikers die allemaal een nieuwe feature hebben geprobeerd.
- [Impact](https://learn.microsoft.com/en-us/azure/azure-monitor/app/usage-impact): Helpt je begrijpen hoe verschillende factoren zoals laadtijden en gebruikerseigenschappen de conversie rates beïnvloeden in verschillende delen van je app.
- [Retention](https://learn.microsoft.com/en-us/azure/azure-monitor/app/usage-retention): Helpt je begrijpen hoeveel gebruikers terugkomen naar je app en hoe vaak ze betrokken zijn bij specifieke taken of doelen. Bijvoorbeeld, als je een game site hebt, kun je zien hoeveel gebruikers terugkomen na het winnen of verliezen van een game.

## [Monitor een app (Instrumentation)](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-overview?tabs=aspnetcore)

- **Auto instrumentation**: Telemetry collectie via configuratie zonder de applicatiecode te wijzigen of instrumentation te configureren.
- **Manual Instrumentation**: Coderen tegen de **Application Insights** of **OpenTelemetry** API. Ondersteunt **Entra ID** en **Complex Tracing** (verzamel data die niet beschikbaar is in Application Insights)

## Azure Monitor Componenten

### Azure Monitor Logs

**Wat doet het?** Verzamelt en analyseert loggegevens van je Azure-resources, zoals foutmeldingen en activiteiten.

Je kunt Kusto queries (KQL) schrijven om door je logs te zoeken en patronen te vinden. Bijvoorbeeld: zoek naar alle errors van de laatste 24 uur.

### Azure Monitor Metrics

**Wat doet het?** Meet en bewaakt numerieke waarden (zoals CPU-gebruik, geheugen, aantal requests) van je resources in real-time.

Metrics zijn getallen die automatisch worden verzameld, zoals:
- Hoeveel requests per minuut
- Gemiddelde response tijd
- Percentage CPU gebruik
- Hoeveel geheugen wordt gebruikt

### Application Insights Alerts

**Wat doet het?** Stuurt meldingen als er afwijkingen of problemen worden gedetecteerd in je applicatie, bijvoorbeeld bij fouten of trage responstijden.

Je kunt bijvoorbeeld instellen:
- "Stuur een email als de response tijd > 3 seconden"
- "Stuur een SMS als er > 10 errors per minuut zijn"
- "Trigger een Azure Function als de app niet beschikbaar is"

### Application Insights Web Tests

**Wat doet het?** Simuleert gebruikersacties op je website om te controleren of deze bereikbaar en snel is, en waarschuwt bij problemen.

Er zijn verschillende soorten tests (zie Availability test sectie hieronder voor details).

## Proactieve Detectie van Problemen

### Smart Detection ✅ (Aanbevolen voor automatische waarschuwing)

**Wat doet het?** Gebruikt machine learning om automatisch prestatieproblemen en afwijkingen in je web app te detecteren en je te waarschuwen.

**Waarom kiezen voor Smart Detection?**
- ✅ Detecteert automatisch afwijkingen zonder configuratie
- ✅ Leert normale patronen en waarschuwt bij afwijkingen
- ✅ Detecteert verschillende problemen: trage response times, failure rate spikes, memory leaks
- ✅ Stuurt proactieve waarschuwingen
- ✅ Geen extra setup vereist

**Voorbeelden van wat het detecteert**:
- Plotselinge toename van errors
- Ongewoon trage response times
- Degradatie in prestaties
- Memory leaks
- Abnormale afhankelijkheden

### Snapshot Debugger ❌ (Niet voor automatische detectie)

**Wat doet het?** Legt de status van je applicatie vast op het moment van een fout (snapshot), zodat je kunt debuggen.

**Waarom NIET voor automatische waarschuwing?**
- ❌ Wordt gebruikt voor debugging, niet voor detectie
- ❌ Je moet handmatig naar snapshots kijken
- ❌ Reageert alleen op exceptions die al zijn opgetreden
- ✅ Wel handig: zie exacte variabele waarden en call stack op moment van crash

### Profiler ❌ (Niet voor automatische detectie)

**Wat doet het?** Analyseert de prestaties van je web app door gedetailleerde performance metrics te verzamelen.

**Waarom NIET voor automatische waarschuwing?**
- ❌ Verzamelt alleen data, waarschuwt niet automatisch
- ❌ Je moet handmatig de profiling resultaten bekijken
- ❌ Focus op performance analyse, niet op proactieve detectie
- ✅ Wel handig: zie welke code het langzaamst is (welke methods nemen de meeste tijd)

### Multi-step Test ❌ (Niet voor automatische detectie)

**Wat doet het?** Test de beschikbaarheid en responsiviteit van je applicatie door een reeks stappen uit te voeren (synthetic transactions).

**Waarom NIET voor automatische detectie van performance problemen?**
- ❌ Test alleen beschikbaarheid, niet prestatieproblemen
- ❌ Voert vooraf gedefinieerde scripts uit
- ❌ Detecteert geen afwijkingen in normaal verkeer
- ✅ Wel handig: test complete user journeys (login → winkelwagen → checkout)

### Vergelijking

| Feature          | Automatische Detectie | Waarschuwt Proactief | Use Case                          |
| ---------------- | --------------------- | -------------------- | --------------------------------- |
| Smart Detection  | ✅ Ja                 | ✅ Ja                | Automatisch problemen ontdekken   |
| Snapshot Debugger| ❌ Nee                | ❌ Nee               | Debuggen van crashes              |
| Profiler         | ❌ Nee                | ❌ Nee               | Performance analyse               |
| Multi-step Test  | ❌ Nee                | ⚠️ Alleen bij tests  | Availability monitoring           |

## [Availability test](https://learn.microsoft.com/en-us/azure/azure-monitor/app/troubleshoot-availability)

Tot 100 tests per Application Insights resource.

- [URL ping test (classic - wordt afgebouwd in september 2026)](https://learn.microsoft.com/en-us/azure/azure-monitor/app/monitor-web-app-availability): Controleer endpoint response en meet prestaties. Pas succescriteria aan met geavanceerde functies zoals het parsen van afhankelijke requests en retries. Het is afhankelijk van publiek internet DNS; zorg ervoor dat publieke domain name servers alle test domain names oplossen. Gebruik anders custom **TrackAvailability** tests.
- [Standard test](https://learn.microsoft.com/en-us/azure/azure-monitor/app/availability-standard-tests): Vergelijkbaar met URL ping, deze single request test omvat SSL certificaat validiteit, proactieve lifetime check, HTTP request verb (`GET`, `HEAD`, of `POST`), custom headers en geassocieerde data.
- [Custom TrackAvailability test](https://learn.microsoft.com/en-us/azure/azure-monitor/app/availability-azure-functions): Gebruik [TrackAvailability()](https://learn.microsoft.com/en-us/dotnet/api/microsoft.applicationinsights.telemetryclient.trackavailability) methode om resultaten naar Application Insights te sturen. Ideaal voor `multi-request` of `authentication` test scenario's. (Opmerking: Multi-step tests zijn de legacy versie; Om multi-step tests te maken, gebruik Visual Studio)

Voorbeeld: Maak een alert die je via email notificeert als de web app niet reageert:

`Portal > Application Insights resource > Availability > Add Test optie > Rules (Alerts) > stel action group in voor availability alert > Configureer notifications (email, SMS)`

## [Troubleshoot app performance met Application Map](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-map)

Application Map visualiseert applicatie topologie en hoe componenten interacteren via HTTP dependencies. Het is gebouwd van telemetry verzonden door de Application Insights SDK of OpenTelemetry.

### Belangrijkste Vereisten

- Gebruik een **enkele Application Insights resource** (niet alleen hetzelfde subscription) om alle componenten op een enkele map te zien.
- Componenten worden gegroepeerd op hun `cloud_RoleName` waarde. Om **component namen aan te passen**, override de waarde met:
  - Classic SDK: `cloud_RoleName`
  - OpenTelemetry: `service.name` + `service.namespace`

### Gecontaineriseerde Services met OpenTelemetry

In microservices of gecontaineriseerde omgevingen, gebruik **OpenTelemetry Resource attributes** om component identiteit te definiëren.

| OTel Attribute        | Maps naar in App Insights  | Voorbeeld            |
| --------------------- | -------------------------- | -------------------- |
| `service.name`        | Logische service naam      | `orders-api`         |
| `service.namespace`   | Component groepering       | `ecommerce-platform` |
| `service.instance.id` | Unieke service instance ID | `instance-01`        |

Deze waarden worden intern vertaald door Azure Monitor:

- `cloud_RoleName` = `service.namespace`.`service.name`
- `cloud_RoleInstance` = `service.instance.id` (indien opgegeven)

### [Application Map Integratie met OpenTelemetry](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-configuration)

Om alle services correct in Application Map weer te geven:

- Alle componenten **moeten telemetry naar dezelfde Application Insights resource sturen**.
- Elke component **moet een onderscheidende `cloud_RoleName` instellen** (via `service.name` en `service.namespace`).

## Monitor een lokale web API met Application Insights

```cs
// Dit forceert HTTPS
builder.Services.AddApplicationInsightsTelemetry();
builder.Services.AddServiceProfiler();
```

appsettings.json:

```jsonc
"ApplicationInsights": {
  // Nodig om telemetry data naar Application Insights te sturen
  "InstrumentationKey": "instrumentation-key"
}
```

Vertrouw lokale certificaten: `dotnet dev-certs https --trust`

## Azure Monitor

- Azure Monitor: Infrastructuur en multi-resource monitoring, inclusief hybride en multi-cloud omgevingen.
- Application Insights: Applicatie-level monitoring (application performance management - APM), speciaal voor web apps en services.

Application Insights data kan ook bekeken worden in Azure Monitor voor een gecentraliseerde ervaring.

### [Activity Log](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/activity-log)

Registreert subscription-level events, zoals wijzigingen aan resources of het starten van een virtual machine.

**Diagnostic Settings**: Maakt het mogelijk om de activity log naar verschillende locaties te sturen:

- **Log Analytics workspace**: Maak gebruik van log queries voor diepe inzichten (_Kusto queries_) en complexe alerting. Standaard worden events 90 dagen bewaard, maar je kunt een diagnostic setting maken voor langere retentie.
- **Azure Storage account**: Voor audit, statische analyse of backup. Minder duur, en logs kunnen daar voor onbepaalde tijd worden bewaard.
- **Azure Event Hubs**: Stream data naar externe systemen zoals third-party SIEMs en andere Log Analytics oplossingen.

OPMERKING: `az monitor activity-log` _kan geen_ data van Application Insight telemetry tonen!

## Configuratie

### Connection string

Van env var `APPLICATIONINSIGHTS_CONNECTION_STRING`. Bepaalt waar telemetry naartoe wordt gestuurd.

## Metric Source Scaling Rule

### Service Bus Queue

- **Message Count**: Het aantal berichten dat momenteel in de queue staat.
- **Active Message Count**: Het aantal actieve berichten in de queue.
- **Dead-letter Message Count**: Het aantal berichten dat is verplaatst naar de dead-letter queue.
- **Scheduled Message Count**: Het aantal berichten dat gepland is om op een toekomstig tijdstip in de queue te verschijnen.
- **Transfer Message Count**: Het aantal berichten dat is overgedragen naar een andere queue of topic.
- **Transfer Dead-letter Message Count**: Het aantal berichten dat is overgedragen naar de dead-letter queue voor een andere queue of topic.

### Azure Blob Storage

- **Blob Count**: Aantal blobs in een container.
- **Blob Size**: Totale grootte van blobs in een container.
- **Egress**: Data egress rate.
- **Ingress**: Data ingress rate.

### Azure Event Hub

- **Incoming Messages**: Aantal ontvangen berichten.
- **Outgoing Messages**: Aantal verzonden berichten.
- **Capture Backlog**: Aantal berichten dat wacht om te worden captured.

## [OpenTelemetry](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable)

### Custom Metrics

| OTel Instrument            | Azure Aggregatie Type     |
| -------------------------- | ------------------------- |
| `Counter` / `AsyncCounter` | Sum                       |
| `UpDownCounter`            | Sum                       |
| `Histogram`                | Min, Max, Avg, Sum, Count |

Histogram - komt het dichtst bij `GetMetric()` van de classic SDK

#### Voorbeelden

```csharp
var meter = new Meter("MyApp.Metrics");

var histogram = meter.CreateHistogram<long>("response_time_ms");
// Registreer een paar willekeurige verkoopprijzen voor appels en citroenen, met verschillende kleuren.
histogram.Record(rand.Next(1, 1000), new("name", "apple"), new("color", "red"));

var counter = meter.CreateCounter<long>("MyFruitCounter");
// Registreer het aantal verkochte vruchten, gegroepeerd op naam en kleur.
counter.Add(1, new("name", "apple"), new("color", "red"));
```

### [Span Mapping naar Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-add-modify)

Azure Monitor gebruikt het type **OpenTelemetry span** om te bepalen hoe telemetry in Application Insights wordt geclassificeerd. Deze classificatie gebeurt **automatisch** wanneer OpenTelemetry telemetry naar Application Insights wordt geëxporteerd.

#### Span Kind → Application Insights Type

| OpenTelemetry Span Kind | Application Insights Equivalent     | Beschrijving                                   |
| ----------------------- | ----------------------------------- | ---------------------------------------------- |
| `Server`                | **Request**                         | Vertegenwoordigt een inkomend HTTP request     |
| `Client`                | **Dependency**                      | Vertegenwoordigt een uitgaande call naar een service/API |
| `Producer`              | **Dependency**                      | Gebruikt voor messaging of queuing operaties   |
| `Consumer`              | **Request**                         | Bij het ontvangen van een bericht              |
| `Internal`              | **Custom Event / Custom Operation** | Custom telemetry (non-HTTP of systeem-intern)  |

Praktische Voorbeelden:

- Je roept een third-party REST API aan → OpenTelemetry gebruikt een `Client Span` → Application Insights logt een **Dependency**
- Je API ontvangt een request → `Server Span` → gelogd als een **Request**
- Je app post naar Azure Service Bus → `Producer Span` → gelogd als een **Dependency**
- Je background worker consumeert van een queue → `Consumer Span` → gelogd als een **Request**

### [Filtering](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-filter)

- Verwijder ruis (bijv. health checks)
- Drop gevoelige data (PII, credentials)
- Verbeter prestaties door low-value traces uit te sluiten

```csharp
// Via Instrumentation
builder.Services.AddOpenTelemetry().UseAzureMonitor().WithTracing(builder => builder.AddSqlClientInstrumentation(options => {
  options.SetDbStatementForStoredProcedure = false;
}));

// Via custom span processor
public class ActivityFilteringProcessor : BaseProcessor<Activity>
{
    public override void OnStart(Activity activity)
    {
        // Drop alle internal spans
        if (activity.Kind == ActivityKind.Internal)
            activity.IsAllDataRequested = false;
    }
}
```
