# [Microsoft Graph](https://learn.microsoft.com/en-us/graph/)

Biedt een unified programmability model dat je kunt gebruiken om toegang te krijgen tot data in Microsoft 365, Windows 10 en Enterprise Mobility + Security.

- Endpoint: `https://graph.microsoft.com`. Kan user en device identity, toegang, compliance, security beheren en helpt organisaties te beschermen tegen data leakage of verlies.

- [Microsoft Graph connectors](https://learn.microsoft.com/en-us/microsoftsearch/connectors-overview): leveren **data extern aan de Microsoft cloud naar Microsoft Graph services en applicaties** (Box, Google Drive, Jira en Salesforce).
- [Microsoft Graph Data Connect](https://learn.microsoft.com/en-us/graph/data-connect-concept-overview): leveren **Microsoft Graph data naar populaire Azure data stores**.

## Resources

Resource specificeren de entity of complex type waarmee je interacteert, zoals `me`, `user`, `group`, `drive` of `site`. Top-level resources kunnen relaties hebben, waardoor toegang tot andere resources mogelijk is, zoals `me/messages` of `me/drive`. Interacties met resources worden gedaan via methods, bijv. `me/sendMail` voor het verzenden van een email. Permissions die nodig zijn voor elke resource kunnen variëren, waarbij hogere permissions vaak vereist zijn voor creation of updates vergeleken met lezen. [Meer over permissions](https://learn.microsoft.com/en-us/graph/permissions-reference).

## [Headers](https://learn.microsoft.com/en-us/graph/use-the-api#headers)

Bevatten standaard en custom HTTP types. Bepaalde APIs kunnen extra headers nodig hebben in requests. Verplichte headers zoals de `request-id` worden altijd geretourneerd door Microsoft Graph, en bepaalde headers, zoals `Retry-After` tijdens throttling of `Location` voor long-running operaties, zijn specifiek voor bepaalde APIs of features.

**Evolvable enumerations**: Standaard retourneert een GET operatie alleen bekende (bestaande) members voor properties. Het toevoegen van members aan bestaande enumerations kan applicaties breken die deze enums al gebruiken. Je kunt opt-in om alle members te ontvangen door een HTTP `Prefer` request header te gebruiken.

## Query Microsoft Graph met REST

[Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer)

### [Metadata](https://learn.microsoft.com/en-us/graph/traverse-the-graph?tabs=http#microsoft-graph-api-metadata)

`https://graph.microsoft.com/v1.0/$metadata`

Metadata in Microsoft Graph biedt inzicht in het data model, inclusief entity types, complex types en enumerations aanwezig in request en response data. Het definieert types, methods en enumerations in OData namespaces, waarbij het meeste van de API in de namespace `microsoft.graph` staat en sommige in subnamespaces zoals `microsoft.graph.callRecords`. Het helpt relaties tussen entities te begrijpen en maakt URL navigatie tussen hen mogelijk.

### REST gebruiken

```http
{HTTP method} https://graph.microsoft.com/{version}/{resource}?{query-parameters}
```

- HTTP _Authorization_ request header, als een _Bearer_ token
- [Pagination](https://learn.microsoft.com/en-us/graph/paging) wordt afgehandeld via `@odata.nextLink`.

| Method | Beschrijving                                  |
| ------ | -------------------------------------------- |
| GET    | Lees data van een resource.                   |
| POST   | Maak een nieuwe resource aan, of voer een actie uit. |
| PATCH  | Update een resource met nieuwe waarden.           |
| PUT    | Vervang een resource met een nieuwe.           |
| DELETE | Verwijder een resource.                           |

- Voor de CRUD methods `GET` en `DELETE` is geen request body vereist.
- De `POST`, `PATCH` en `PUT` methods vereisen een request body gespecificeerd in JSON formaat.

#### Voorbeelden

| Operatie                                | URL                                                                                                                      |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| GET mijn profiel                           | `https://graph.microsoft.com/v1.0/me`                                                                                    |
| GET mijn bestanden                             | `https://graph.microsoft.com/v1.0/me/drive/root/children`                                                                |
| GET mijn foto                             | `https://graph.microsoft.com/v1.0/me/photo/$value`                                                                       |
| GET mijn foto metadata                    | `https://graph.microsoft.com/v1.0/me/photo/`                                                                             |
| GET mijn mail                              | `https://graph.microsoft.com/v1.0/me/messages`                                                                           |
| GET mijn high importance email             | `https://graph.microsoft.com/v1.0/me/messages?$filter=importance eq 'high'`                                              |
| GET mijn calendar events                   | `https://graph.microsoft.com/v1.0/me/events`                                                                             |
| GET mijn manager                           | `https://graph.microsoft.com/v1.0/me/manager`                                                                            |
| GET laatste gebruiker die bestand foo.txt wijzigde     | `https://graph.microsoft.com/v1.0/me/drive/root/children/foo.txt/lastModifiedByUser`                                     |
| GET Microsoft 365 groups waarvan ik lid ben | `https://graph.microsoft.com/v1.0/me/memberOf/$/microsoft.graph.group?$filter=groupTypes/any(a:a eq 'unified')`          |
| GET gebruikers in mijn organisatie             | `https://graph.microsoft.com/v1.0/users`                                                                                 |
| GET groups in mijn organisatie            | `https://graph.microsoft.com/v1.0/groups`                                                                                |
| GET mensen gerelateerd aan mij                 | `https://graph.microsoft.com/v1.0/me/people`                                                                             |
| GET items trending om mij heen             | `https://graph.microsoft.com/beta/me/insights/trending`                                                                  |
| GET mijn recente activiteiten                 | `https://graph.microsoft.com/v1.0//me/activities/recent`                                                                 |
| PATCH (update) een recente activiteit van mij | `https://graph.microsoft.com/v1.0//me/activities/{activityId}`                                                           |
| GET mijn notities                             | `https://graph.microsoft.com/v1.0/me/onenote/notebooks`                                                                  |
| Selecteer specifieke velden                   | `https://graph.microsoft.com/v1.0/groups?$filter=adatumisv_courses/id eq '123'&$select=id,displayName,adatumisv_courses` |
| Alerts, filter op Category, top 5        | `https://graph.microsoft.com/v1.0/security/alerts?$filter=Category eq 'ransomware'&$top=5`                               |

#### MSAL gebruiken

```cs
var authority = "https://login.microsoftonline.com/" + tenantId;
var scopes = new []{ "https://graph.microsoft.com/.default" };

var app = ConfidentialClientApplicationBuilder.Create(clientId)
    .WithAuthority(authority)
    .WithClientSecret(clientSecret)
    .Build();

var result = await app.AcquireTokenForClient(scopes).ExecuteAsync();

var httpClient = new HttpClient();
var request = new HttpRequestMessage(HttpMethod.Get, "https://graph.microsoft.com/v1.0/me");
request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", result.AccessToken);

var response = await httpClient.SendAsync(request);
var content = await response.Content.ReadAsStringAsync();
```

### SDK gebruiken

```csharp
var scopes = new[] { "User.Read" };

// Multi-tenant apps kunnen "common" gebruiken,
// single-tenant apps moeten de tenant ID van de Azure portal gebruiken
var tenantId = "common";

// Waarde van app registratie
var clientId = "YOUR_CLIENT_ID";

// using Azure.Identity;
var options = new TokenCredentialOptions
{
    AuthorityHost = AzureAuthorityHosts.AzurePublicCloud
};

// Device code gebruiken: https://learn.microsoft.com/dotnet/api/azure.identity.devicecodecredential
var deviceOptions = new DeviceCodeCredentialOptions
{
    AuthorityHost = AzureAuthorityHosts.AzurePublicCloud,
    ClientId = clientId,
    TenantId = tenantId,
    // Callback functie die de user prompt ontvangt
    // Prompt bevat de gegenereerde device code die gebruiker moet
    // invoeren tijdens het auth proces in de browser
    DeviceCodeCallback = (code, cancellation) =>
    {
        Console.WriteLine(code.Message);
        return Task.FromResult(0);
    },
};
var credential = new DeviceCodeCredential(deviceOptions);
// var credential = new DeviceCodeCredential(callback, tenantId, clientId, options);

// Client certificate gebruiken: https://learn.microsoft.com/dotnet/api/azure.identity.clientcertificatecredential
// var clientCertificate = new X509Certificate2("MyCertificate.pfx");
// var credential = new ClientCertificateCredential(tenantId, clientId, clientCertificate, options);

// Client secret gebruiken: https://learn.microsoft.com/dotnet/api/azure.identity.clientsecretcredential
// var credential = new ClientSecretCredential(tenantId, clientId, clientSecret, options);

// On-behalf-of provider
// var oboToken = "JWT_TOKEN_TO_EXCHANGE";
// var onBehalfOfCredential = new OnBehalfOfCredential(tenantId, clientId, clientSecret, oboToken, options);

var graphClient = new GraphServiceClient(credential, scopes);

var user = await graphClient.Me.GetAsync();

var messages = await graphClient.Me.Messages
.GetAsync(requestConfig =>
{
    requestConfig.QueryParameters.Select =
        new string[] { "subject", "sender" };
    requestConfig.QueryParameters.Filter =
        "subject eq 'Hello world'";

    requestConfig.Headers.Add(
        "Prefer", @"outlook.timezone=""Pacific Standard Time""");
});

var message = await graphClient.Me.Messages[messageId].GetAsync();

var newCalendar = await graphClient.Me.Calendars
    .PostAsync(new Calendar { Name = "Volunteer" }); // nieuw

await graphClient.Teams["teamId"]
    .PatchAsync(new Team { }); // update

await graphClient.Me.Messages[messageId]
    .DeleteAsync();
```

## Token acquisition flow

- **Verkrijg een Authorization Code**: `GET https://login.microsoftonline.com/{tenant}/oauth2/authorize`
- **Verkrijg een Access Token**: `POST https://login.microsoftonline.com/customer.com/oauth2/token`
- **Roep Microsoft Graph aan**:

  ```http
  GET https://graph.microsoft.com/beta/users
  Authorization: Bearer <token>
  ```

## [Permissions](https://learn.microsoft.com/en-us/graph/permissions-reference)

`<Resource>.<Permission>` of `<Resource>.<Permission>.<Optional-Constrain>`

Voorbeeld voor `Users`:

- Huidige gebruiker: `User.Read`, `User.ReadWrite`
- Alle gebruikers (vereist admin consent): `User.Read.All`, `User.ReadWrite.All`, `User.ReadBasic.All` (geen admin consent)

De optionele `All` constraint verleent toegang tot alle gebruikers.

Voorbeeld voor `Calendars`:

- Calendars van huidige gebruiker: `Calendars.Read`, `Calendars.ReadWrite`
- Calendars gedeeld met huidige gebruiker: `Calendars.Read.Shared`, `Calendars.ReadWrite.Shared`

De optionele `Shared` constraint verleent toegang tot calendars waar de gebruiker toegang tot heeft, er is geen `All` constraint.
