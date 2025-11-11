# [Azure Managed Identities](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/)

Stel Azure App Services-gebaseerde apps in staat om toegang te krijgen tot andere services zonder credentials te beheren. Deze identiteiten zijn _Azure-exclusief_ en _kunnen niet gebruikt worden met andere cloud providers_ zoals AWS of GCP.

**Verplicht**: Navigeer in Azure Portal naar `Settings > Access policies > Add Access Policy` om je app toegang te geven. Selecteer permissions en identity name en type. Het verwijderen van een policy kan 24 uur duren vanwege caching.

| Eigenschap                     | System-assigned managed identity                                                                                                                                       | User-assigned managed identity                                                                                                                                                                                                                                                                                                                                    |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Aanmaken                       | Aangemaakt als onderdeel van een Azure resource (bijvoorbeeld Azure Virtual Machines of Azure App Service).                                                            | Aangemaakt als een op zichzelf staande Azure resource.                                                                                                                                                                                                                                                                                                            |
| Levenscyclus                   | Gedeelde levenscyclus met de Azure resource.<br>Verwijderd wanneer de parent resource verwijderd wordt.<br>Kan niet expliciet verwijderd worden.                      | Onafhankelijke levenscyclus.<br>Moet expliciet verwijderd worden.                                                                                                                                                                                                                                                                                                 |
| Delen tussen Azure resources   | Kan niet gedeeld worden.<br>Kan alleen gekoppeld worden aan een enkele Azure resource.                                                                                | Kan gedeeld worden.<br>Dezelfde user-assigned managed identity kan gekoppeld (gedeeld) worden met meerdere Azure resources.                                                                                                                                                                                                                                       |
| Veelvoorkomende use cases      | Workloads binnen een enkele Azure resource.<br>Workloads die onafhankelijke identiteiten nodig hebben.<br>Bijvoorbeeld een applicatie die draait op een enkele virtual machine. | Workloads die draaien op meerdere resources en een enkele identity kunnen delen.<br>Workloads die pre-authorization nodig hebben voor een beveiligde resource, als onderdeel van een provisioning flow.<br>Workloads waar resources regelmatig gerecycled worden, maar permissions consistent moeten blijven.<br>Bijvoorbeeld een workload waar meerdere virtual machines toegang moeten hebben tot dezelfde resource. |

**Als je een multi-tenant setup hebt, gebruik dan Application Service Principal!**

## [Role-based access control (Azure RBAC)](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-portal#assign-a-user-as-an-administrator-of-a-subscription)

Beheer toegang tot Azure resources. Om Azure roles toe te wijzen, moet je `Microsoft.Authorization/roleAssignments/write` permissions hebben, zoals `User Access Administrator` of `Owner`.

Read access tot alle resources: `*/read`.

De inheritance volgorde voor scope is Management group, Subscription, Resource group, Resource. Bij het toewijzen van toegang, volg de regel van least privilege. Let op: Controleer goed of je permissions geeft voor resource of resource group!

## Managed Identity gebruiken met een Virtual Machine

1. **Initieer Managed Identity**: Vraag om het inschakelen (system assigned) of aanmaken (user assigned) van managed identity via ARM.
1. **Maak Service Principal**: ARM maakt een service principal aan in de vertrouwde Entra ID tenant voor de managed identity.
1. **Configureer Identity**: ARM werkt [IMDS](https://learn.microsoft.com/en-us/azure/virtual-machines/instance-metadata-service) (VM Specifiek) bij met de service principal client ID en certificate.
1. **Wijs Roles & Access toe**: Gebruik service principal informatie om toegang te geven tot Azure resources via RBAC.
1. **Vraag Token aan**: Code op Azure resource vraagt een token aan bij IMDS: `http://169.254.169.254/metadata/identity/oauth2/token`
1. **Verkrijg Token**: Door gebruik te maken van de geconfigureerde client ID en certificate, retourneert Entra ID een JWT access token op aanvraag.
1. **Gebruik Token**: Code gebruikt het token om te authenticeren bij Entra ID-ondersteunde services.

## Identities Beheren

1. **System-assigned Identity**

   ```sh
   # Een resource aanmaken (zoals een VM of een andere service die het ondersteunt) met een system-assigned identity
   az <service> create --resource-group $resourceGroup --name myResource --assign-identity '[system]'

   # Een system-assigned identity toewijzen aan een bestaande resource
   az <service> identity assign --resource-group $resourceGroup --name myResource --identities '[system]'
   ```

1. **User-assigned Identity**

   ```sh
   # Maak eerst de identity aan
   az identity create --resource-group $resourceGroup --name identityName

   # Een resource aanmaken (zoals een VM of een andere service die het ondersteunt) met een user-assigned identity
   az <service> create --assign-identity $identityName --resource-group $resourceGroup --name $resourceName
   #az <service> create --assign-identity '/subscriptions/<SubId>/resourcegroups/$resourceGroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/myIdentity' --resource-group $resourceGroup --name $resourceName

   # Een user-assigned identity toewijzen aan een bestaande resource
   az <service> identity assign --identities $identityName --resource-group $resourceGroup --name $resourceName
   # az <service> identity assign --identities '/subscriptions/<SubId>/resourcegroups/$resourceGroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/myIdentity' --resource-group $resourceGroup --name $resourceName
   ```

Zowel system-assigned als user-assigned managed identities kunnen specifieke Azure roles krijgen, waardoor ze bepaalde acties kunnen uitvoeren op specifieke Azure resources. Deze roles zijn onderdeel van Azure's Role-Based Access Control ([RBAC](https://docs.microsoft.com/en-us/azure/role-based-access-control/overview)) systeem, dat fine-grained access management biedt voor Azure resources.

```sh
az role assignment create --assignee <PrincipalId> --role <RoleName> --scope <Scope>
```

## Azure Access Control

- [**Roles**](https://docs.microsoft.com/en-us/azure/role-based-access-control/role-definitions): Definiëren welke acties je kunt uitvoeren.
  - **Owner**: Volledige toegang, inclusief role assignment.
  - **Contributor**: Volledige toegang, geen role assignment.
  - **Reader**: Alleen lezen.
  - **User Access Administrator**: Beheert gebruikerstoegang tot resources.
- [**Scopes**](https://docs.microsoft.com/en-us/azure/role-based-access-control/scope-overview): Definiëren waar acties van toepassing zijn.
  - **Management Group**: Alle subscriptions en resources.
  - **Subscription**: Alle resources in subscription.
  - **Resource Group**: Alle resources in group.
  - **Resource**: Alleen specifieke resource.

[Deny assignments](https://docs.microsoft.com/en-us/azure/role-based-access-control/deny-assignments) **overschrijven role assignments** om specifieke acties te blokkeren.

### Hiërarchie voor het beheren van een resource (van laagste naar hoogste permission levels)

- Geen role: Gebruikers of managed identities krijgen alleen de permissions die ze nodig hebben, zoals read of write, zonder een voorgedefinieerde role te gebruiken.
- Resource `Reader`
- Resource `Contributor`
- Resource `Owner`
- Resource `Administrator`
- Global Administrator

## Een Access Token verkrijgen met Azure Managed Identities

**DefaultAzureCredential**: Deze class probeert meerdere authenticatiemethoden op basis van de beschikbare environment of sign-in details, en stopt zodra het succesvol is. Het controleert de volgende bronnen in volgorde:

1. Environment variables ([`EnvironmentCredential`](https://learn.microsoft.com/en-us/dotnet/api/azure.identity.environmentcredential?view=azure-dotnet)) - `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, plus:
   - Service principle met secret ([`ClientSecretCredential`](https://learn.microsoft.com/en-us/dotnet/api/azure.identity.clientsecretcredential?view=azure-dotnet)): `AZURE_CLIENT_SECRET`
   - Service principal met certificate ([`ClientCertificateCredential`](https://learn.microsoft.com/en-us/dotnet/api/azure.identity.clientcertificatecredential?view=azure-dotnet)): `AZURE_CLIENT_SECRET`, `AZURE_CLIENT_CERTIFICATE_PATH`, `AZURE_CLIENT_CERTIFICATE_PASSWORD`, `AZURE_CLIENT_SEND_CERTIFICATE_CHAIN`
   - Username en password ([`UsernamePasswordCredential`](https://learn.microsoft.com/en-us/dotnet/api/azure.identity.usernamepasswordcredential?view=azure-dotnet)): `AZURE_USERNAME`, `AZURE_PASSWORD`
1. Managed Identity als de applicatie gedeployed is op een Azure host waar deze feature is ingeschakeld.

   ```cs
   new ManagedIdentityCredential(); // system-assigned
   new ManagedIdentityCredential(clientId: userAssignedClientId); // user-assigned
   new DefaultAzureCredential(new DefaultAzureCredentialOptions { ManagedIdentityClientId = userAssignedClientId }); // user-assigned
   ```

1. Visual Studio als de developer erdoorheen geauthenticeerd is.
1. Azure CLI (`AzureCliCredential`) als de developer geauthenticeerd is via het `az login` commando.
1. Azure PowerShell als de developer geauthenticeerd is via het `Connect-AzAccount` commando.
1. Interactive browser, hoewel deze optie standaard uitgeschakeld is.

   ```cs
   new InteractiveBrowserCredential();
   new DefaultAzureCredential(includeInteractiveCredentials: true);
   ```

**ChainedTokenCredential**: Stelt gebruikers in staat om meerdere credential instances te combineren om een aangepaste chain van credentials te definiëren.

```csharp
// authenticeer met managed identity, en val terug op authenticatie via Azure CLI als managed identity niet beschikbaar is in de huidige environment
var credential = new ChainedTokenCredential(new ManagedIdentityCredential(), new AzureCliCredential());
```

## [Logging](https://github.com/Azure/azure-sdk-for-net/blob/Azure.Identity_1.9.0/sdk/core/Azure.Core/samples/Diagnostics.md#logging)

```cs
// Zorg dat AzureEventSourceListener in scope en actief is tijdens het gebruik van de client library voor log collection.
// Maak het aan als een top-level member van de class die de Event Hubs client gebruikt.
using AzureEventSourceListener listener = AzureEventSourceListener.CreateConsoleLogger();

DefaultAzureCredentialOptions options = new DefaultAzureCredentialOptions
{
    Diagnostics =
    {
        LoggedHeaderNames = { "x-ms-request-id" },
        LoggedQueryParameters = { "api-version" },
        IsAccountIdentifierLoggingEnabled = true, // schakel logging van gevoelige informatie in
        IsLoggingContentEnabled = true // log details over het account dat gebruikt werd voor authenticatie en authorization
    }
};
```

Exceptions: Service client methods gooien `AuthenticationFailedException` voor token problemen.

## [Token caching](https://github.com/Azure/azure-sdk-for-net/blob/Azure.Identity_1.9.0/sdk/identity/Azure.Identity/samples/TokenCache.md)

Tokens kunnen opgeslagen worden in _memory_ (standaard) of op _disk_ (opt-in). Gebruik `TokenCachePersistenceOptions()` voor default cache, specificeer een `Name` voor geïsoleerde cache, en `UnsafeAllowUnencryptedStorage` voor onversleutelde opslag. Verschillende credentials ondersteunen verschillende caching types - CLI: None, Default en Managed - alleen cache, rest: beide.
