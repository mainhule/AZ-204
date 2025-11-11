# [Azure App Configuration](https://learn.microsoft.com/en-us/azure/azure-app-configuration/)

Azure App Configuration centraliseert app settings en feature flags, waardoor het eenvoudiger wordt om hiërarchische configuraties te beheren over verschillende settings en locaties. Het maakt real-time updates mogelijk zonder dat de app opnieuw opgestart hoeft te worden. Hoewel het _alle data versleutelt_, mist het de geavanceerde security features van Azure Key Vault, zoals hardware-level encryption, granular access policies en management environments.

Je kunt configuratie importeren en exporteren tussen Azure App Configuration en aparte bestanden. Het ondersteunt ook aparte stores voor verschillende development stages zoals testing en production.

## Azure App Configuration Overzicht

Azure App Configuration beheert configuratiedata met behulp van key-value pairs.

- **Keys**: Unieke, _case-sensitive_ identifiers voor waarden. Ze kunnen elk unicode karakter bevatten behalve `*`, `,`, en `\` (gereserveerd kan ge-escaped worden met '\'). Gebruik delimiters zoals `/` of `:` voor hiërarchische organisatie. Azure behandelt _keys als geheel_ en dwingt geen structuur af. Voorbeeld: `AppName:Service1:ApiEndpoint`
- **Labels**: Groepeer keys op criteria, bijv. environments of versies (wat _niet native ondersteund wordt_). Default label is `null`. Voorbeeld: `Key = AppName:DbEndpoint & Label = Test`. Key prefixes zijn een alternatieve manier van groeperen (labelen). Om expliciet te verwijzen naar een key-value zonder label, gebruik `\0`. Verschillende labels creëren verschillende versies van dezelfde key, deze worden beschouwd als verschillende (unieke) entries.
- **Values**: Unicode strings optioneel gekoppeld aan een user-defined content type voor aanvullende metadata.

### Configuratie en Querying

```cs
// Laad configuratie
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(connectionString)
           // Selecteer een subset van de configuration keys (beginnen met `TestApp:` en hebben geen label) van Azure App Configuration
           // Om alle keys te selecteren: KeyFilter.Any
           .Select("TestApp:*", LabelFilter.Null)
           // Configureer om configuratie opnieuw te laden als de geregistreerde sentinel key gewijzigd is
           .ConfigureRefresh(refreshOptions => refreshOptions.Register("TestApp:Settings:Sentinel", refreshAll: true));
});
builder.Services.AddFeatureManagement();
builder.Services.AddFeatureFilter<TargetingFilter>(); // (bijvoorbeeld)

// Query keys
AsyncPageable<ConfigurationSetting> settings = client.GetConfigurationSettingsAsync(new SettingSelector { KeyFilter = "AppName:*" });
await foreach (ConfigurationSetting setting in settings)
    Console.WriteLine($"Key: {setting.Key}, Value: {setting.Value}");
```

#### Chaining

Bij het gebruik van meerdere `.Select()`, als een key met dezelfde naam bestaat in beide labels, wordt de waarde van de laatste `.Select()` gebruikt.

```cs
// In dit voorbeeld, als een key zoals 'TestApp:Key' bestaat met zowel het 'dev' label als zonder label,
// wordt de waarde gekoppeld aan het 'dev' label gebruikt. Dit is vanwege de volgorde van de .Select() aanroepen.
// Als je de volgorde van de .Select() aanroepen omdraait, zou de waarde zonder label voorrang krijgen.
.Select("TestApp:*", LabelFilter.Null)
.Select("TestApp:*", "dev");
```

## [Feature Management](https://learn.microsoft.com/en-us/azure/azure-app-configuration/howto-feature-filters)

- **Feature flag**: Een binaire variabele (on/off) die de uitvoering van een gekoppeld codeblok controleert.
- **Feature manager**: Een software package die de lifecycle van feature flags beheert, en aanvullende functies biedt zoals caching en het bijwerken van flag states.
- **Filter**: Een regel die de status van een feature flag bepaalt, gebaseerd op gebruikersgroepen, apparaattypes, geografische locatie of tijdvensters.

Een succesvol feature management systeem vereist:

- Een applicatie die feature flags gebruikt.
- Een aparte repository die feature flags en hun states opslaat.

### Feature Flags gebruiken in Code

Feature flags worden gebruikt als Boolean state variabelen in conditional statements:

```csharp
bool featureFlag = true; // statische waarde
bool featureFlag = isBetaUser(); // geëvalueerde waarde

if (featureFlag) { /* ... */ }
else { /* ... */ }
```

#### [Feature flags configureren](https://learn.microsoft.com/en-us/azure/azure-app-configuration/quickstart-feature-flag-aspnet-core)

```cs
// Laad configuratie van Azure App Configuration
builder.Configuration.AddAzureAppConfiguration(options =>
{
    // Standaard als er geen parameter doorgegeven wordt (options.UseFeatureFlags()), laadt het alle feature flags zonder label
    // De standaard refresh expiration van feature flags is 30 seconden
    options.UseFeatureFlags(featureFlagOptions =>
    {
        // Selecteer feature flags van "TestApp" namespace, met label "dev"
        featureFlagOptions.Select("TestApp:*", "dev");
        featureFlagOptions.CacheExpirationInterval = TimeSpan.FromMinutes(5);
    });
});

// Voeg feature management toe aan de container van services.
builder.Services.AddFeatureManagement();
// Een feature filter registreren
builder.Services.AddFeatureFilter<TargetingFilter>(); // (bijvoorbeeld)
```

### [Conditional feature flags](https://learn.microsoft.com/en-us/azure/azure-app-configuration/howto-feature-filters-aspnet-core)

Staat toe dat de feature flag dynamisch ingeschakeld of uitgeschakeld wordt.

- `PercentageFilter` schakelt de feature flag in op basis van een percentage.
- `TimeWindowFilter` schakelt de feature flag in gedurende een gespecificeerd tijdvenster.
- `TargetingFilter` schakelt de feature flag in voor gespecificeerde gebruikers en groepen.

#### [Staged rollout van features inschakelen voor targeted audiences met `TargetingFilter`](https://learn.microsoft.com/en-us/azure/azure-app-configuration/howto-targetingfilter-aspnet-core)

1. Implementeer `ITargetingContextAccessor`

   ```cs
   private const string TargetingContextLookup = "TestTargetingContextAccessor.TargetingContext";
   public ValueTask<TargetingContext> GetContextAsync()
   {
       if (httpContext.Items.TryGetValue(TargetingContextLookup, out object value))
           return new ValueTask<TargetingContext>((TargetingContext)value);

       // Voorbeeld: `test@contoso.com` - User: `test`, Group(s): `contoso.com`
       List<string> groups = new List<string>();
       if (httpContext.User.Identity.Name != null)
           groups.Add(httpContext.User.Identity.Name.Split("@", StringSplitOptions.None)[1]);

       var targetingContext = new TargetingContext
       {
           UserId = httpContext.User.Identity.Name,
           Groups = groups
       };
       httpContext.Items[TargetingContextLookup] = targetingContext;
       return new ValueTask<TargetingContext>(targetingContext);
    }
   ```

1. Voeg `TargetingFilter` toe: `services.AddFeatureManagement().AddFeatureFilter<TargetingFilter>();`

1. Update de `ConfigureServices` method om de `ITargetingContextAccessor` implementatie toe te voegen: `services.AddSingleton<ITargetingContextAccessor, TestTargetingContextAccessor>();`

1. `app > Feature Manager > (maak feature flag aan) > (schakel in) > Edit > Use feature filter > Targeting filter >  Override by Groups en Override by Users`

### Feature Flags declareren

Een feature flag bestaat uit een naam en één of meer filters die bepalen of de feature aan (`true`) staat. Wanneer meerdere filters gebruikt worden, worden ze in volgorde geëvalueerd totdat er één de feature inschakelt. Als geen enkele dat doet, staat de feature uit.

De feature manager ondersteunt _appsettings.json_ als configuratiebron voor feature flags:

```jsonc
{
  "FeatureManagement": {
    "FeatureA": true, // Feature aan
    "FeatureB": false, // Feature uit
    "FeatureC": {
      "EnabledFor": [
        {
          "Name": "Percentage",
          "Parameters": { "Value": 50 }
        }
      ]
    }
  }
}
```

### Feature Flag Repository

Azure App Configuration dient als een gecentraliseerde repository voor feature flags, waardoor het mogelijk is om alle feature flags die in een applicatie gebruikt worden te externaliseren. Dit maakt het mogelijk om feature states te wijzigen zonder de applicatie te hoeven aanpassen en opnieuw te deployen.

## Security

### [Customer-Managed Keys gebruiken voor Encryption](https://learn.microsoft.com/en-us/azure/azure-app-configuration/concept-customer-managed-keys)

Een managed identity authenticeert met Microsoft Entra ID en wraps de encryption key met behulp van Azure Key Vault. De wrapped key wordt opgeslagen en de unwrapped key wordt gecached voor een uur, en vervolgens vernieuwd.

Vereisten:

- _Een Standard tier_ Azure App Configuration
- Azure Key Vault met soft-delete en purge-protection
- Een niet-verlopen, ingeschakelde RSA of RSA-HSM key in de Key Vault met wrap en unwrap mogelijkheden

Na setup, wijs een managed identity toe aan de App Configuration en verleen het `GET`, `WRAP`, en `UNWRAP` (staat het decrypten van eerder wrapped keys toe) permissions in de access policy van de Key Vault:

```sh
az keyvault set-policy --key-permissions get wrapKey unwrapKey
```

## [Private endpoint](https://learn.microsoft.com/en-us/azure/azure-app-configuration/concept-private-endpoint)

- Maakt veilige toegang mogelijk tot Azure App Configuration via een private link met een IP adres uit de VNet address space.
- Verkeer blijft op het Microsoft backbone netwerk, waardoor blootstelling aan het publieke internet voorkomen wordt.
- Blokkeert standaard public network toegang; kan opnieuw ingeschakeld worden.
- Gebruikt dezelfde connection strings/auth; geen app wijzigingen nodig.

## Key Vault configureren

```cs
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(
        builder.Configuration["ConnectionStrings:AppConfig"])
            .ConfigureKeyVault(kv => kv.SetCredential(new DefaultAzureCredential()));
});
```

Na de initialisatie kun je de waarden van Key Vault references op dezelfde manier benaderen als de waarden van reguliere App Configuration keys.

## Configuratie importeren / exporteren

Importeer alle keys en feature flags van een bestand en pas test label toe: `az appconfig kv import -n MyAppConfiguration --label test -s file --path D:/abc.json --format json`

Exporteer alle keys en feature flags met label test naar een json bestand: `az appconfig kv export -n MyAppConfiguration --label test -d file --path D:/abc.json --format json`

## CLI

- [az appconfig kv import](https://learn.microsoft.com/en-us/cli/azure/appconfig/kv?view=azure-cli-latest#az-appconfig-kv-import)
- [az appconfig kv export](https://learn.microsoft.com/en-us/cli/azure/appconfig/kv?view=azure-cli-latest#az-appconfig-kv-export)
- [az appconfig identity](https://learn.microsoft.com/en-us/cli/azure/appconfig/identity?view=azure-cli-latest)
