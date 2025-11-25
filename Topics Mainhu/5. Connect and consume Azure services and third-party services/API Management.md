# [Azure API Management](https://docs.microsoft.com/en-us/azure/api-management/)

API Management is een service waarmee je APIs kunt beheren, beveiligen, publiceren en monitoren.

## [Tiers](https://learn.microsoft.com/en-us/azure/api-management/api-management-features)

- **Consumption Tier**: Geen Entra ID integratie, private endpoint support voor inbound connections, developer portal, built-in cache, built-in analytics, backup en restore, management over Git, directe management API, Azure Monitor en Log Analytics request logs, Static IP.
- **Developer Tier**: Zoals Premium, zonder multi-region deployment en availability zones.
- **Basic Tier**: Geen Entra ID integratie en workspaces.
- **Standard Tier**: Heeft Entra ID Integratie en workspaces.
- **Premium Tier**: Heeft VNET, meerdere custom domain names, self-hosted gateway

## Componenten

- **API Gateway**: Endpoint dat API calls routeert, credentials verifieert, quotas en limieten afdwingt, requests/responses transformeert zoals gespecificeerd in policy statements, responses cachet en logs/metrics produceert voor monitoring.
- **Management Plane**: Administratieve interface voor service instellingen, definiëren en importeren van API schema, verpakken van APIs in producten, policy setup, analytics en gebruikersbeheer.
- **Developer Portal**: Automatisch gegenereerde website voor API documentatie die developers in staat stelt om API details te bekijken, interactieve consoles te gebruiken, een account aan te maken en te subscriben voor API keys, gebruik te analyseren, API definities te downloaden en API keys te beheren.

## Products

Products bundelen APIs voor developers. Ze hebben een titel, beschrijving en gebruiksvoorwaarden. Ze kunnen **Open** zijn (bruikbaar zonder subscription) of **Protected** (vereist subscription). Subscription goedkeuring is ofwel auto-goedgekeurd of vereist admin goedkeuring.

## Groups

Groups bepalen product zichtbaarheid voor developers. API Management heeft drie onveranderlijke systeem groups:

- **Administrators**: Beheren service instances, APIs, operations en products. Azure subscription admins zitten in deze group.
- **Developers**: Geauthenticeerde portal gebruikers kunnen apps bouwen met je APIs. Ze hebben toegang tot het Developer portal om API operations te gebruiken en kunnen worden aangemaakt door admins, uitgenodigd of zelf-geregistreerd. Ze behoren tot meerdere groups en kunnen subscriben op group-zichtbare products.
- **Guests**: Niet-geauthenticeerde portal gebruikers met potentiële read-only toegang, zoals het bekijken van APIs maar niet het aanroepen ervan.

Naast systeem groups kunnen admins custom groups vormen of externe groups gebruiken van gerelateerde Microsoft Entra ID tenants.

## [API Gateways](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/gateway)

API gateways, zoals Azure's API Management gateway, beheren communicatie tussen clients en meerdere front- en back-end services, wat de noodzaak elimineert voor clients om specifieke endpoints te kennen (zie gateways als _reverse proxy_). Deze gateways stroomlijnen service integratie, vooral wanneer services worden bijgewerkt of geïntroduceerd. Ze zorgen ook voor taken zoals SSL termination, authentication en rate limiting. Azure's API Management gateway proxyt specifiek API requests, handhaaft policies en verzamelt telemetry data.

- **Managed** gateways zijn standaard componenten gedeployd in Azure voor elke API Management instance. Ze handelen alle API traffic af, ongeacht waar de APIs worden gehost.
- **Self-hosted** gateways zijn optionele, containerized versies van managed gateways. Ze zijn geschikt voor hybride en multicloud omgevingen, waardoor on-premises APIs en APIs over meerdere clouds beheerd kunnen worden vanuit een enkele Azure API Management service.

## [Policies overview](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-policies)

Een set statements die sequentieel worden uitgevoerd op een API's request of response. Ze kunnen worden toegepast op verschillende scopes:

- **Global Scope**: Pas organisatiebrede policies toe die elke API en operation beïnvloeden. Ideaal voor het afdwingen van beveiligingsmaatregelen zoals IP filtering of logging over alle APIs.
- **Workspace Scope**: Target APIs binnen een specifieke workspace. Best voor interne organisatie, het scheiden van APIs op basis van teams of afdelingen.
- **Product Scope**: Groepeer meerdere APIs onder een enkel product om het subscription proces te vereenvoudigen. Best voor externe organisatie, groeperen van APIs op basis van functionaliteit of business logica.
- **API Scope**: Pas policies toe die alle operations binnen een specifieke API beïnvloeden. Nuttig voor API-specifieke transformaties of validaties.
- **Operation Scope**: Fijnmazige controle over individuele API operations. Ideaal voor operation-specifieke validaties of transformaties.

### Types van policies

- **Inbound**: Toegepast voordat naar de backend wordt gerouteerd. Voorbeelden: validatie, authentication, rate limiting, request transformation.
- **Backend**: Toegepast voordat de request de backend bereikt. Voorbeelden: URL rewriting, headers instellen.
- **Outbound**: Toegepast op de response voordat deze naar de client wordt gestuurd. Voorbeelden: response transformation, caching, headers toevoegen.

Opmerking: Als client een response verwacht in een bepaald formaat (voorbeeld: XML), controleer de vraag om te zien welk type endpoint wordt gebruikt (voorbeeld: JSON). Als ze verschillend zijn, moet transform policy worden toegepast (voorbeeld: `json-to-xml-policy`)

### Policy Configuratie

`<base />`: voer de standaard policies uit die zijn gedefinieerd op andere scopes (bijv. Product of Global scope). Biedt de mogelijkheid om policy evaluatie volgorde af te dwingen.

```xml
<policies>
  <inbound>
    <!-- statements die op de request worden toegepast komen hier -->
  </inbound>
  <backend>
    <!-- statements die worden toegepast voordat de request wordt doorgestuurd naar
         de backend service komen hier -->
  </backend>
  <outbound>
    <!-- statements die op de response worden toegepast komen hier -->
  </outbound>
  <on-error>
    <!-- statements die worden toegepast als er een error conditie is komen hier -->
  </on-error>
</policies>
```

**Alle tijden zijn in seconden!**  
**Alle groottes zijn in KB!**

### [Named values](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-properties?tabs=azure-cli)

Voeg een named value toe: `Dashboard > API Management Services > service > Named values`

**Types:**

- Plain: Literal string of policy expression
- Secret: Literal string of policy expression die is encrypted door API Management
- [Key vault](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-properties): Identifier van een secret opgeslagen in een Azure key vault. Na update in de key vault wordt een named value in API Management binnen 4 uur bijgewerkt. Vereist managed identity. Configureer ofwel een _key vault access policy_ of _Azure RBAC access_ voor een API Management managed identity. Key Vault Firewall vereist _system-assigned managed identity_.

Voeg een secret toe:

```sh
az apim nv create --resource-group $resourceGroup \
    --display-name "named_value_01" --named-value-id named_value_01 \
    --secret true --service-name apim-hello-world --value test
```

Om een named value in een policy te gebruiken, plaats je de display name binnen een dubbel paar accolades zoals `{{ContosoHeader}}`. Als de waarde een expression is, wordt deze geëvalueerd. Als de waarde de naam is van een andere named value - niet.

## Policy Expressions

Kunnen worden gebruikt als attribute waarden of text waarden in elke API Management policy. Ze kunnen een enkele C# statement zijn ingesloten in `@(expression)` of een multi-statement C# code block, ingesloten in `@{expression}`, dat een waarde retourneert.

Voorbeeld `set-body`:

```xml
<set-body>
  @{
    var response = context.Response.Body.As<JObject>();
    foreach (var key in new [] {"minutely", "hourly", "daily", "flags"}) {
    response.Property (key).Remove ();
    }
  return response.ToString();
  }
</set-body>
```

### [Throttling](https://learn.microsoft.com/en-us/azure/api-management/api-management-sample-flexible-throttling)

Gebruik `rate-limit-by-key` (limiteer aantal requests) of `quota-by-key` (limiteer bandwidth en/of aantal requests). Renewal period is in _seconden_, bandwidth size is in _KB_. Gebruik `counter-key` om identity of IP te specificeren.

Wanneer quota wordt overschreden, wordt een `403 Forbidden` status geretourneerd.

- Throttle per IP: `counter-key="@(context.Request.IpAddress)"`
- Throttle per Identity: `counter-key="@(context.Request.Headers.GetValueOrDefault("Authorization","").AsJwt()?.Subject)"`

### [Caching](https://learn.microsoft.com/en-us/azure/api-management/api-management-sample-cache-by-key)

Azure APIM heeft ingebouwde ondersteuning voor HTTP response caching met de resource URL als key. Je kunt de key wijzigen met request headers met de vary-by properties.

- Get from cache (`cache-lookup`): inbound
- Store to cache (`cache-store`): outbound
- Anderen zijn mixed

#### Voorbeelden

##### Cache per header

```http
https://myapi.azure-api.net/me
Authorization: Bearer <access_token>
```

Endpoint is niet uniek, maar de authorization header is dat wel voor elke gebruiker.

```xml
<cache-lookup>
    <vary-by-header>Authorization</vary-by-header>
</cache-lookup>
```

##### Cache per query parameter

```http
https://myapi.azure-api.net/samples?topic=apim&section=caching
```

Endpoint heeft twee query parameters: `topic` en `section`. Gebruik puntkomma om te scheiden.

```xml
<cache-lookup>
    <vary-by-query-parameter>topic;section</vary-by-query-parameter>
</cache-lookup>
```

##### Cache per url

```http
https://myapi.azure-api.net/items/123456
```

Endpoint heeft geen parameters, maar de url is uniek. Deze keer gebruik je een lege string.

```xml
<cache-lookup>
    <vary-by-query-parameter></vary-by-query-parameter>
</cache-lookup>
```

##### Fragment caching

Wanneer je informatie van een extern systeem wilt toevoegen aan de huidige response, zonder het elke keer op te halen. Bijvoorbeeld `/me/tasks` retourneert gebruikerstaken en profiel, maar het profiel is opgeslagen op `/userprofile/{userid}`. Om te voorkomen dat het profiel elke keer wordt opgehaald, moeten de volgende regels worden geïmplementeerd:

```xml
<!-- Extract userId from JWT -->
<set-variable
  name="enduserid"
  value="@(context.Request.Headers.GetValueOrDefault("Authorization","").Split(' ')[1].AsJwt()?.Subject)" />

<!-- Data wordt verondersteld opgeslagen te zijn in userprofile (bijvoorbeeld) -->
<!-- Als userprofile nog niet gecached is, stuur een request en sla de response op in cache -->
<choose>
    <when condition="@(!context.Variables.ContainsKey("userprofile"))">
        <!-- Maak een HTTP request naar /userprofile/{userid} om het op te halen  -->
        <send-request params>options</send-request>

        <!-- Sla op in cache -->
        <cache-store-value
          key="@("userprofile-" + context.Variables["enduserid"])"
          value="@(((IResponse)context.Variables["userprofileresponse"]).Body.As<string>())" duration="100000" />
    </when>
</choose>

<!-- Gebruik userprofile uit cache -->
<cache-lookup-value
  key="@("userprofile-" + context.Variables["enduserid"])"
  variable-name="userprofile" />
```

## API Security

### [Via Entra ID](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-protect-backend-with-aad)

1. Registreer een applicatie in Entra ID om de API te vertegenwoordigen
2. Configureer een JWT validation policy om requests vooraf te autoriseren (`validate-jwt`)

### [Managed Identities](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-use-managed-service-identity)

Gebruik de `authentication-managed-identity` policy om te authenticeren met een service via managed identity. Het haalt een access token op van Microsoft Entra ID en zet deze in de `Authorization` header met het Bearer scheme. Het token wordt gecached tot het verloopt. Als geen client-id wordt gegeven, wordt de system-assigned identity gebruikt.

Voorbeeld: `<authentication-managed-identity resource="resource" client-id="clientid of user-assigned identity" output-token-variable-name="token-variable" ignore-error="true|false"/>`, waar resource kan zijn `https://graph.microsoft.com`, `https://management.azure.com/`, etc.

Use cases:

- Verkrijg een custom TLS/SSL certificaat voor de API Management instance van Azure Key Vault
- Sla named values op en beheer ze vanuit Azure Key Vault
- Authenticeer bij een backend met een user-assigned identity
- Log events naar een event hub

### Via Subscriptions

API Management laat je APIs beveiligen met subscription keys. Developers moeten deze keys opnemen in HTTP requests bij het benaderen van APIs. Als dat niet gebeurt, wijst API Management gateway de requests af. De subscription keys komen van subscriptions, die developers kunnen verkrijgen zonder toestemming nodig te hebben van API publishers. Naast dit zijn OAuth2.0, Client certificates en IP allow listing andere beveiligingsmethoden.

#### [Subscription Keys](https://learn.microsoft.com/en-us/azure/api-management/api-management-subscriptions)

Een subscription key is een unieke, automatisch gegenereerde key opgenomen in client request headers of als query string parameter. Het is gekoppeld aan een subscription die verschillende scopes kan hebben die gevarieerde toegangsniveaus bieden. Subscriptions laten je permissions en policies minutieus controleren.

Drie belangrijkste subscription scopes:

- **All APIs**: Geeft toegang tot alle APIs geconfigureerd in de service.
- **Single API**: Toegangscontrole beperkt tot een specifieke API en zijn endpoints.
- **Product**: Van toepassing op een bepaald product (een collectie van APIs) in API Management, elk met verschillende toegangsregels, usage quotas en gebruiksvoorwaarden.

Keys moeten worden opgenomen in elke request naar een protected API. Ze kunnen worden geregenereerd indien nodig, zoals als een key is gelekt. Subscriptions hebben een primary en secondary key om te helpen bij regeneratie zonder downtime. In products met enabled subscriptions moeten clients een key opgeven bij het aanroepen van APIs. Ze kunnen een key verkrijgen via een subscription request.

##### Aanroepen van API met Subscription Key

API calls hebben een geldige key nodig in HTTP requests. Deze kan worden doorgegeven in de request header of als query string. De standaard header naam is **Ocp-Apim-Subscription-Key**, en de standaard query string is **subscription-key**. APIs kunnen worden getest met de developer portal of command-line tools zoals curl. Hier zijn voorbeeld curl commando's met een header en een URL query string:

```sh
curl --header "Ocp-Apim-Subscription-Key: <key string>" https://<apim gateway>.azure-api.net/api/path
```

```sh
curl https://<apim gateway>.azure-api.net/api/path?subscription-key=<key string>
```

Het niet doorgeven van de key resulteert in een **401 Access Denied** response.

### [API Security met Certificates](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-mutual-certificates-for-clients)

- `authentication-certificate`: Gebruikt door APIM om zichzelf te authenticeren bij de backend service.
- `validate-client-certificate` (_inbound_): Gebruikt door APIM om het client certificaat te valideren dat verbinding maakt met APIM.

Configureer toegang tot key vault:

- Access policy: `Access Policies > + Create > Secret permissions > Permissions tab > Get and List ; Principal tab > principal > managed identity > Next > Review + create > Create`
- RBAC access: `Access control (IAM) > Add role assignment > Role tab > Key Vault Secrets User ; Members tab > Managed identity > + Select members > identity`

#### TLS Client Authentication

API gateways inspecteren client certificaten op specifieke attributen, zoals Certificate Authority (CA), Thumbprint, Subject en Expiration Date. Deze kunnen worden gecombineerd voor custom policies.

Certificaten zijn ondertekend om tampering te voorkomen. Valideer ontvangen certificaten om te zorgen dat ze authentiek zijn. Trusted CAs of fysiek afgeleverde self-signed certificaten zijn manieren om authenticiteit te bevestigen.

In de _Consumption_ tier moeten client certificaten _handmatig enabled_ zijn op de **Custom domains** pagina.

#### Controleer de thumbprint tegen certificaten geüpload naar API Management

Gebruik `<when condition="$(...)">` policy met `<return-response>` en `<set-status code="403" reason="Invalid client certificate" />`:

```cs
// Client certificaat
context.Request.Certificate == null || context.Request.Certificate.Thumbprint != "desired-thumbprint"

// Certificaten geüpload naar API Management
context.Request.Certificate == null || !context.Request.Certificate.Verify() || !context.Deployment.Certificates.Any(c => c.Value.Thumbprint == context.Request.Certificate.Thumbprint)

// Controleer de issuer en subject van een client certificaat
context.Request.Certificate == null || context.Request.Certificate.Issuer != "trusted-issuer" || context.Request.Certificate.SubjectName.Name != "expected-subject-name"
```

## [Error handling](https://learn.microsoft.com/en-us/azure/api-management/api-management-error-handling-policies)

- Azure API Management gebruikt een `ProxyError` object, toegankelijk via `context.LastError`, voor het afhandelen van errors tijdens request processing.
- Policies zijn verdeeld in inbound, backend, outbound en on-error secties. Processing springt naar de on-error sectie als er een error optreedt.
- Als er geen on-error sectie is, ontvangen callers `400` of `500` HTTP response messages tijdens een error.
- Voorgedefinieerde errors bestaan voor ingebouwde stappen en policies, elk met een source, condition, reason en message.
- Custom gedrag, zoals het loggen van errors of het maken van nieuwe responses, kan worden geconfigureerd in de on-error sectie.

## Versions en Revisions

- Gebruik **Revisions** voor non-breaking changes, waardoor testing en updates mogelijk zijn zonder bestaande gebruikers te beïnvloeden. Gebruikers kunnen verschillende revisions benaderen door een andere query string te gebruiken op hetzelfde endpoint.
- Gebruik **Versions** voor breaking changes, wat publicatie vereist en mogelijk vereist dat gebruikers hun applicaties updaten.

[Versioning schemes](https://learn.microsoft.com/en-us/azure/api-management/api-management-versions):

- Path-based versioning: `https://apis.contoso.com/products/v1` en `https://apis.contoso.com/products/v2`
- Header-based versioning: Bijvoorbeeld custom header genaamd `Api-Version,` en clients specificeren `v1` of `v2`
- Query string-based versioning: `https://apis.contoso.com/products?api-version=v1` en `https://apis.contoso.com/products?api-version=v2`

_Header-based versioning_ als de _URL hetzelfde moet blijven_. Revisions en andere types van versioning schemas vereisen gewijzigde URL.

Het maken van aparte gateways of web APIs zou gebruikers dwingen om een ander endpoint te benaderen. Een aparte gateway biedt complete isolatie.

```sh
az apim api release create --resource-group $resourceGroup \
    --api-id demo-conference-api --api-revision 2 --service-name apim-hello-world \
    --notes 'Testing revisions. Added new "test" operation.'
```

## Belangrijke Exam Tips

1. ✅ **Policies scope**: Global > Workspace > Product > API > Operation
2. ✅ **Policy secties**: Inbound → Backend → Outbound → On-Error
3. ✅ **Subscription keys**: Ocp-Apim-Subscription-Key header of subscription-key query param
4. ✅ **Named values**: Gebruik `{{naam}}` in policies, opslaan in Key Vault voor secrets
5. ✅ **Caching**: `cache-lookup` (inbound) en `cache-store` (outbound)
6. ✅ **Rate limiting**: `rate-limit-by-key` (aantal requests), `quota-by-key` (bandwidth + requests)
7. ✅ **Versioning schemes**: Path, Header of Query string based
8. ✅ **Revisions**: Non-breaking changes, Versions: Breaking changes
9. ✅ **Self-hosted gateway**: Voor hybrid/multi-cloud scenarios
10. ✅ **Managed Identity**: Gebruik voor authenticatie bij backend services (Key Vault, etc.)
11. ✅ **validate-jwt**: Voor Entra ID token validatie
12. ✅ **Premium tier**: Vereist voor VNET, multi-region, self-hosted gateway
13. ✅ **Developer Portal**: Automatisch gegenereerde documentatie voor API consumers
14. ✅ **Products**: Bundel van APIs met eigen subscription en toegangsregels
15. ✅ **Consumption tier**: Geen Entra ID, client certificates moeten handmatig enabled worden
