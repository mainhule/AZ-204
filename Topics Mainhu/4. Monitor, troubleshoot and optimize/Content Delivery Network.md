# [Azure Content Delivery Network (CDN)](https://learn.microsoft.com/en-us/azure/cdn/cdn-overview)

Azure Content Delivery Network (CDN) is een gedistribueerd netwerk van servers dat content efficiënt kan leveren aan gebruikers. CDN's slaan cached content op in edge locations (Point of Presence - PoP) die dicht bij eindgebruikers staan om latency te minimaliseren.

## Belangrijkste Voordelen

- **Betere prestaties**: Verminderde latency door content dichter bij gebruikers te plaatsen
- **Schaalbaarheid**: Handelt grote loads af en vermindert load op origin servers
- **Kostenreductie**: Minder bandwidth gebruik op origin servers
- **Wereldwijde distributie**: 100+ PoP locaties wereldwijd
- **DDoS bescherming**: Ingebouwde bescherming tegen DDoS attacks
- **HTTPS ondersteuning**: Secure content delivery met SSL/TLS

## Use Cases

- **Static website hosting**: Leveren van HTML, CSS, JavaScript, afbeeldingen
- **Video streaming**: On-demand en live video streaming
- **Software distributie**: Downloads van grote bestanden (updates, installers)
- **Gaming**: Game assets en updates
- **API acceleration**: Cache API responses voor betere performance
- **E-commerce**: Product afbeeldingen en catalogi

## Azure CDN Products

Azure biedt vier verschillende CDN producten:

| Product                              | Provider       | Key Features                                     | Pricing        |
| ------------------------------------ | -------------- | ------------------------------------------------ | -------------- |
| **Azure CDN Standard from Microsoft**| Microsoft      | Integratie met Azure services, basis features    | 💰 Laag        |
| **Azure CDN Standard from Edgio**    | Edgio (Verizon)| Advanced analytics, real-time stats              | 💰💰 Gemiddeld |
| **Azure CDN Premium from Edgio**     | Edgio (Verizon)| Rules engine, advanced caching, token auth       | 💰💰💰 Hoog    |
| **Azure Front Door**                 | Microsoft      | Modern CDN + WAF + load balancing + routing      | 💰💰💰 Premium  |

### Azure CDN Standard from Microsoft (Recommended voor AZ-204)

- Geïntegreerd met Azure services
- Automatische certificaat provisioning
- Basis caching rules
- Query string caching
- Compression

### Azure CDN Premium from Edgio

- Geavanceerde rules engine met 50+ regels
- Real-time statistics en detailed analytics
- Token authentication voor content security
- Custom SSL certificates
- Advanced HTTP/HTTPS settings

### Azure Front Door

Modern alternatief met extra features:

- Application load balancing
- Web Application Firewall (WAF)
- URL-based routing
- Session affinity
- SSL offloading
- Custom domains met wildcard support

## Core Concepten

### CDN Profile

Een container voor CDN endpoints. Één profile per pricing tier.

```sh
az cdn profile create \
    --name MyCDNProfile \
    --resource-group MyResourceGroup \
    --sku Standard_Microsoft
```

### CDN Endpoint

Een specifieke configuratie voor content delivery van een origin.

- Unieke hostname: `<endpoint-name>.azureedge.net`
- Kan meerdere origins hebben (voor failover)
- Caching rules
- Custom domain mapping

```sh
az cdn endpoint create \
    --name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup \
    --origin www.contoso.com \
    --origin-host-header www.contoso.com
```

### Origin

De bron server waar CDN content vandaan haalt:

- **Azure Storage**: Blob storage voor static content
- **Web App**: Azure App Service
- **Cloud Service**: Azure Cloud Services
- **Custom origin**: Elke publiek toegankelijke webserver

## Caching Behavior

### Caching Rules

CDN bepaalt of content gecached wordt op basis van:

1. **Global caching rules**: Van toepassing op alle requests
2. **Custom caching rules**: Specifieke paths of file extensions
3. **Query string caching**: Hoe om te gaan met query parameters

### Query String Caching Modes

```sh
az cdn endpoint update \
    --name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup \
    --query-string-caching-behavior BypassCaching
```

| Mode                  | Gedrag                                                                  | Use Case                          |
| --------------------- | ----------------------------------------------------------------------- | --------------------------------- |
| **IgnoreQueryString** | Query strings worden genegeerd; alle requests delen dezelfde cache     | Static content zonder variaties   |
| **BypassCaching**     | Requests met query strings worden niet gecached                         | Dynamic content                   |
| **UseQueryString**    | Elke unieke URL (met query string) wordt apart gecached                 | Personalized content              |

### Cache Expiration (TTL)

Time-to-live bepaalt hoe lang content in de cache blijft:

```http
Cache-Control: max-age=3600
Cache-Control: public, max-age=86400
Expires: Wed, 21 Oct 2025 07:28:00 GMT
```

**Priority** (hoogste naar laagste):

1. Custom caching rules in CDN
2. `Cache-Control` header van origin server
3. Default CDN TTL (7 dagen voor Standard Microsoft)

```sh
# Stel standaard TTL in via caching rule
az cdn endpoint rule add \
    --name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup \
    --order 1 \
    --rule-name "SetCacheExpiration" \
    --action-name "CacheExpiration" \
    --cache-behavior "SetIfMissing" \
    --cache-duration "1.00:00:00"
```

### File Compression

CDN kan automatisch bestanden comprimeren:

```sh
az cdn endpoint update \
    --name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup \
    --enable-compression true \
    --content-types-to-compress "text/plain" "text/html" "text/css" "application/javascript"
```

**Vereisten**:

- Bestand groter dan 1 KB en kleiner dan 8 MB
- MIME type in de compression lijst
- Request bevat `Accept-Encoding: gzip` header

**Ondersteunde types**:

- text/plain, text/html, text/css
- application/javascript, application/json, application/xml
- image/svg+xml

## Cache Purging

Verwijder content uit de cache voordat TTL verloopt:

```sh
# Purge specifieke bestanden
az cdn endpoint purge \
    --name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup \
    --content-paths "/images/logo.png" "/css/styles.css"

# Purge alle content
az cdn endpoint purge \
    --name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup \
    --content-paths "/*"

# Purge directory
az cdn endpoint purge \
    --name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup \
    --content-paths "/images/*"
```

### Pre-loading Content

Pre-load (warm-up) populaire content in de cache:

```sh
az cdn endpoint load \
    --name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup \
    --content-paths "/index.html" "/images/hero.jpg"
```

⭐ **Use case**: Voor grote events of product launches om ervoor te zorgen dat content al gecached is.

## Custom Domains

Gebruik je eigen domain naam in plaats van `azureedge.net`:

### Stap 1: Voeg CNAME record toe

```
cdn.contoso.com  CNAME  myendpoint.azureedge.net
```

### Stap 2: Map custom domain in Azure

```sh
az cdn custom-domain create \
    --name cdn-contoso-com \
    --endpoint-name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup \
    --hostname cdn.contoso.com
```

### Stap 3: Enable HTTPS

```sh
az cdn custom-domain enable-https \
    --name cdn-contoso-com \
    --endpoint-name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup
```

Azure CDN biedt **gratis** managed SSL certificates (via DigiCert):

- Automatische provisioning
- Automatische renewal
- Geen extra kosten

**Certificaat validatie methoden**:

1. **Domain validation**: Bevestig dat je de domain owner bent
2. **CNAME mapping**: Azure valideert via CNAME record

## Geo-Filtering

Beperk toegang tot content op basis van land/regio:

```sh
az cdn endpoint rule add \
    --name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup \
    --order 1 \
    --rule-name "BlockCountries" \
    --match-variable "RemoteAddress" \
    --operator "GeoMatch" \
    --match-values "CN" "RU" \
    --action-name "Deny"
```

**Use cases**:

- Content licenties die regiospecifiek zijn
- Compliance requirements (GDPR)
- Security (blokkeer malicious regions)

## Origin Types

### Azure Storage Blob

Ideaal voor static content hosting.

```sh
# Maak storage account en container
az storage account create --name mystorageacct --resource-group MyResourceGroup
az storage container create --name mycontainer --account-name mystorageacct --public-access blob

# Maak CDN endpoint met Storage als origin
az cdn endpoint create \
    --name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup \
    --origin mystorageacct.blob.core.windows.net \
    --origin-path "/mycontainer" \
    --origin-host-header mystorageacct.blob.core.windows.net
```

### Azure App Service

Gebruik CDN voor dynamic websites met static assets:

```sh
az cdn endpoint create \
    --name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup \
    --origin mywebapp.azurewebsites.net \
    --origin-host-header mywebapp.azurewebsites.net
```

**Best practice**: Cache alleen `/images/*`, `/css/*`, `/js/*` paths.

### Custom Origin

Elke publiek toegankelijke webserver:

```sh
az cdn endpoint create \
    --name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup \
    --origin www.example.com \
    --origin-host-header www.example.com \
    --http-port 80 \
    --https-port 443
```

## Monitoring en Diagnostics

### Metrics

Key metrics in Azure Portal:

- **Request count**: Totaal aantal requests naar CDN
- **Data transfer**: Uitgaande bandwidth
- **Cache hit ratio**: Percentage requests served vanuit cache
  - > 90% = goed
  - < 70% = mogelijk misconfiguratie
- **Origin latency**: Response time van origin server
- **4xx/5xx error rates**: Client en server errors

### Diagnostic Logs

Enable logging naar Storage Account, Event Hub of Log Analytics:

```sh
az monitor diagnostic-settings create \
    --name CDN-Diagnostics \
    --resource /subscriptions/{sub-id}/resourceGroups/MyResourceGroup/providers/Microsoft.Cdn/profiles/MyCDNProfile/endpoints/MyEndpoint \
    --logs '[{"category": "CoreAnalytics", "enabled": true}]' \
    --storage-account mystorageaccount
```

**Log categories**:

- **CoreAnalytics**: Request counts, bandwidth, latency, cache hits
- **WebApplicationFirewall**: WAF logs (alleen Azure Front Door)

## Best Practices

### 1. Optimize Cache Hit Ratio

- Stel passende TTL waarden in
- Gebruik `Cache-Control` headers op origin
- Vermijd query strings voor static content
- Implementeer versioning in filenames (`app.v2.js`)

### 2. Minimize Origin Load

- Pre-load populaire content
- Gebruik lange cache durations voor static assets
- Implementeer cache warming voor grote events

### 3. Security

- Enable HTTPS voor alle endpoints
- Gebruik custom domains met managed certificates
- Implementeer geo-filtering indien nodig
- Gebruik SAS tokens voor private blob content

### 4. Cost Optimization

- Gebruik compression om bandwidth te verminderen
- Kies het juiste pricing tier
- Monitor en purge unused content
- Implementeer lifecycle policies voor origin storage

### 5. Performance

- Gebruik closest origin server
- Minimize redirect chains
- Optimize image sizes
- Implement lazy loading voor images

## Common Exam Scenarios

### Scenario 1: Static Website with Global Audience

**Vraag**: Een static website moet snel laden voor gebruikers wereldwijd.

**Oplossing**:
1. Host content in Azure Blob Storage
2. Maak Azure CDN profile (Standard Microsoft)
3. Configure endpoint met Storage als origin
4. Enable compression
5. Set appropriate TTL (1 week voor static assets)

### Scenario 2: Secure Content Delivery

**Vraag**: Beveiligde levering van premium content met custom domain.

**Oplossing**:
1. Configure custom domain (cdn.contoso.com)
2. Enable managed HTTPS certificate
3. Implement token authentication (Premium Edgio)
4. Set geo-filtering rules

### Scenario 3: Video Streaming Platform

**Vraag**: Efficiënte delivery van video content met adaptive bitrate.

**Oplossing**:
1. Store videos in Azure Blob Storage
2. Use Azure CDN with Standard Microsoft
3. Enable range requests
4. Set long TTL (30 days)
5. Pre-load popular videos

### Scenario 4: Cache Invalidation

**Vraag**: Direct nieuwe content zichtbaar maken na deployment.

**Oplossing**:
```sh
# Purge old content
az cdn endpoint purge --content-paths "/*"

# Pre-load nieuwe content
az cdn endpoint load --content-paths "/index.html" "/app.js"
```

### Scenario 5: Cost Reduction

**Vraag**: Verlaag bandwidth kosten voor een high-traffic website.

**Oplossing**:
1. Enable file compression
2. Optimize image formats (WebP)
3. Set appropriate cache durations
4. Use CDN instead van direct origin access

## CLI Commands Reference

```sh
# Maak CDN profile
az cdn profile create \
    --name MyCDNProfile \
    --resource-group MyResourceGroup \
    --sku Standard_Microsoft

# Maak CDN endpoint
az cdn endpoint create \
    --name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup \
    --origin www.contoso.com

# Update caching behavior
az cdn endpoint update \
    --name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup \
    --query-string-caching-behavior IgnoreQueryString

# Purge content
az cdn endpoint purge \
    --name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup \
    --content-paths "/*"

# Pre-load content
az cdn endpoint load \
    --name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup \
    --content-paths "/index.html"

# Voeg custom domain toe
az cdn custom-domain create \
    --name myCustomDomain \
    --endpoint-name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup \
    --hostname cdn.contoso.com

# Enable HTTPS
az cdn custom-domain enable-https \
    --name myCustomDomain \
    --endpoint-name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup

# Lijst alle endpoints
az cdn endpoint list \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup

# Verwijder endpoint
az cdn endpoint delete \
    --name MyEndpoint \
    --profile-name MyCDNProfile \
    --resource-group MyResourceGroup
```

## Belangrijke Exam Tips

1. **CDN Profile** is container voor endpoints; kies pricing tier op profile level
2. **Query string caching**: `IgnoreQueryString` is default voor Standard Microsoft
3. **Custom domains** vereisen CNAME record vóór mapping in Azure
4. **HTTPS certificates** zijn gratis met managed certificates (auto-renewal)
5. **Purge vs Load**: Purge verwijdert cache, Load pre-populates cache
6. **Cache hit ratio** > 90% is goed; monitor dit metric
7. **Compression** moet expliciet enabled worden (niet default)
8. **TTL priority**: CDN rules > Cache-Control header > Default TTL
9. **Geo-filtering** alleen beschikbaar in Premium Edgio
10. **Origin types**: Blob Storage, App Service, Cloud Service, Custom
11. **Azure Front Door** is modern alternatief met extra features (WAF, routing)
12. Voor **static websites**: gebruik Blob Storage + CDN combinatie
