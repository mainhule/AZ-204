# [Shared Access Signatures (SAS)](https://learn.microsoft.com/en-us/azure/storage/common/storage-sas-overview)

Een Shared Access Signature (SAS) is een URI die beperkte toegangsrechten verleent tot Azure Storage resources. Het is een signed URI die wijst naar één of meer storage resources en een token bevat met een speciale set query parameters.

Gebruik een SAS voor veilige, tijdelijke toegang tot je storage account, vooral wanneer gebruikers hun eigen data moeten lezen/schrijven of voor het kopiëren van data binnen Azure Storage.

Opmerking: Bij voorkeur zou je Entra ID moeten gebruiken

## Types SAS

1. [**User Delegation SAS**](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-user-delegation-sas-create-dotnet): Deze methode gebruikt Microsoft Entra ID credentials om een SAS aan te maken. Het is een veilige manier om beperkte toegang te verlenen tot je Azure Storage resources zonder je account key te delen. Het wordt aanbevolen wanneer je fine-grained access control wilt bieden aan clients die geauthenticeerd zijn met Entra ID. Het account _moet_ `generateUserDelegationKey` permission hebben, of `Contributor` role.

1. [**Service SAS**](https://learn.microsoft.com/en-us/azure/storage/blobs/sas-service-create-dotnet): Deze methode gebruikt je storage account key om een SAS aan te maken. Het is een eenvoudige manier om beperkte toegang te verlenen tot je Azure Storage resources. Het is echter minder veilig dan de User Delegation SAS omdat het je account key deelt. Het wordt typisch gebruikt wanneer je toegang wilt bieden aan clients die niet geauthenticeerd zijn met Entra ID.

1. [**Account SAS**](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-sas-create-dotnet): Deze methode gebruikt je storage account key om een SAS aan te maken. Het wordt aangemaakt op storage account niveau, waardoor toegang tot meerdere services binnen het account mogelijk is. Het wordt typisch gebruikt wanneer je toegang moet bieden tot verschillende services in je storage account. Het deelt echter je account key, vergelijkbaar met de Service SAS.

   - Ad hoc SAS: Definieert start, expiry en permissions in de SAS URI. Elke SAS kan een ad hoc SAS zijn.
   - Service SAS: Gebruikt een stored policy op resources om start, expiry en permissions over te nemen.

## Hoe SAS werkt

Een SAS vereist twee componenten: een URI naar de resource waartoe je toegang wilt en een SAS token dat je hebt aangemaakt om toegang tot die resource te autoriseren.

- **URI**: `https://<account>.blob.core.windows.net/<container>/<blob>?`
- **SAS token**: `sp=r&st=2020-01-20T11:42:32Z&se=2020-01-20T19:42:32Z&spr=https&sv=2019-02-02&sr=b&sig=SrW1HZ5Nb6MbRzTbXCaPm%2BJiSEn15tC91Y4umMPwVZs%3D`

[Referentie](https://learn.microsoft.com/en-us/rest/api/storageservices/create-service-sas):

| Component | Vriendelijke Naam                           | Beschrijving                                                                                                                                                                                       |
| --------- | --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sp        | `S`hared `P`ermissions                  | Controleert de toegangsrechten. Mogelijke waarden zijn: `a`dd, `c`reate, `d`elete, `l`ist, `r`ead, `w`rite. Bijv: `sp=acdlrw` verleent alle beschikbare rechten.                                         |
| st        | `S`hared Access Signature Start `T`ime  | De datum en tijd waarop toegang start. Bijv: `st=2023-07-28T11:42:32Z` betekent dat de toegang start om 11:42:32 UTC op 28 juli 2023.                                                                     |
| se        | `S`hared Access Signature `E`xpiry Time | De datum en tijd waarop toegang eindigt. Bijv: `se=2023-07-28T19:42:32Z` betekent dat de toegang eindigt om 19:42:32 UTC op 28 juli 2023.                                                         |
| sv        | `S`torage API `V`ersion                 | De versie van de storage API die gebruikt wordt. Bijv: `sv=2020-02-10` betekent dat storage API versie 2020-02-10 gebruikt wordt.                                                                      |
| sr        | `S`torage `R`esource                    | Het type storage waartoe toegang wordt verkregen. Mogelijke waarden zijn: `b`lob, `f`ile, `q`ueue, `t`able, `c`ontainer, `d`irectory. Bijv: `sr=b` betekent dat een blob benaderd wordt.                               |
| sig       | `Sig`nature                             | De cryptografische handtekening. Bijv: `sig=SrW1HZ5Nb6MbRzTbXCaPm%2BJiSEn15tC91Y4umMPwVZs%3D` is een cryptografische handtekening.                                                                             |
| sip       | `S`ource `IP` Range                     | (Optioneel) Toegestane IP adressen of IP range. Bijv: `sip=168.1.5.60-168.1.5.70` betekent dat alleen de IP adressen van 168.1.5.60 tot 168.1.5.70 toegestaan zijn.                                               |
| spr       | `S`upported `Pr`otocols                 | (Optioneel) Toegestane protocols. Mogelijke waarden zijn: 'https', 'http,https'. Bijv: `spr=https` betekent dat alleen HTTPS toegestaan is.                                                                        |
| si        | `S`tored Access **Policy** `I`dentifier | (Optioneel) De naam van de stored access policy. Bijv: `si=MyAccessPolicy` betekent dat de stored access policy genaamd "MyAccessPolicy" gebruikt wordt.                                                           |
| rscc      | `R`e`s`ponse `C`ache `C`ontrol          | (Optioneel) De response header override voor cache control. Bijv: `rscc=public` betekent dat de "Cache-Control" header ingesteld wordt op "public".                                                                 |
| rscd      | `R`e`s`ponse `C`ontent `D`isposition    | (Optioneel) De response header override voor content disposition. Bijv: `rscd=attachment; filename=example.txt` betekent dat de "Content-Disposition" header ingesteld wordt op "attachment; filename=example.txt". |
| rsce      | `R`e`s`ponse `C`ontent `E`ncoding       | (Optioneel) De response header override voor content encoding. Bijv: `rsce=gzip` betekent dat de "Content-Encoding" header ingesteld wordt op "gzip".                                                               |
| rscl      | `R`e`s`ponse `C`ontent `L`anguage       | (Optioneel) De response header override voor content language. Bijv: `rscl=en-US` betekent dat de "Content-Language" header ingesteld wordt op "en-US".                                                             |
| rsct      | `R`e`s`ponse `C`ontent `T`ype           | (Optioneel) De response header override voor content type. Bijv: `rsct=text/plain` betekent dat de "Content-Type" header ingesteld wordt op "text/plain".                                                           |

Opmerking: Alle 2-letter parameters zijn vereist, behalve `si` (Access Policy)

## Best Practices

- Gebruik altijd HTTPS.
- Gebruik user delegation SAS waar mogelijk.
- Stel je expiration time in op de kleinste bruikbare waarde.
- Verleen alleen de toegang die vereist is.
- Maak een middle-tier service om gebruikers en hun toegang tot storage te beheren wanneer er een onaanvaardbaar risico is bij het gebruik van een SAS.

## [Stored Access Policies](https://learn.microsoft.com/en-us/rest/api/storageservices/define-stored-access-policy)

Stellen je in staat om SAS te groeperen en aanvullende constraints in te stellen zoals start time, expiry time en permissions. Werken op **container** niveau.

Gebruik `SetAccessPolicy` op `BlobContainer` om een array toe te passen die een enkele `BlobSignedIdentifier` bevat met een geconfigureerde `BlobAccessPolicy` voor de `AccessPolicy` property.

```cs
BlobSignedIdentifier identifier = new BlobSignedIdentifier
{
    Id = "stored access policy identifier",
    AccessPolicy = new BlobAccessPolicy
    {
        ExpiresOn = DateTimeOffset.UtcNow.AddHours(1),
        Permissions = "rw"
    }
};

blobContainer.SetAccessPolicy(permissions: new BlobSignedIdentifier[] { identifier });
```

```sh
az storage container policy create \
    --name <stored access policy identifier> \
    --container-name <container name> \
    --start <start time UTC datetime> \
    --expiry <expiry time UTC datetime> \
    --permissions <(a)dd, (c)reate, (d)elete, (l)ist, (r)ead, of (w)rite> \
    --account-key <storage account key> \
    --account-name <storage account name> \
```

Om een policy te annuleren (intrekken), kun je het verwijderen, hernoemen of de expiration time instellen op een datum in het verleden.

Om alle access policies van de resource te verwijderen, roep de `Set ACL` operatie aan met een lege request body.

## Werken met SAS

Samenvatting:

- Service SAS en Account SAS gebruiken `StorageSharedKeyCredential`; User delegation SAS gebruikt `DefaultAzureCredential` of vergelijkbare Entra ID
- Service SAS en User delegation SAS gebruiken `BlobSasBuilder`; Account SAS gebruikt `AccountSasBuilder`
- Permissions instellen: `BlobSasPermissions` voor user en service; `AccountSasPermissions` voor account
- URI verkrijgen:
  - User delegation SAS: Gebruik key gegenereerd van `BlobServiceClient.GetUserDelegationKeyAsync` als eerste param van `BlobSasBuilder.ToSasQueryParameters(key, accountName)`; geef het door aan `BlobUriBuilder(BlobClient.Uri).Sas`
  - Service SAS: `BlobClient.GenerateSasUri(BlobSasBuilder)`
  - Account SAS: `BlobSasBuilder.ToSasQueryParameters(sharedKeyCredential)` en construeer Uri ervan op root niveau (`https://{accountName}.blob.core.windows.net?{sasToken}`)

```cs
// StorageSharedKeyCredential gebruiken met account name en key direct voor authenticatie.
// Deze key heeft volledige permissions voor alle operaties op alle resources in je storage account.
// Werkt voor alle SAS types, maar minder veilig.
var credential = new StorageSharedKeyCredential("<account-name>", "<account-key>");

// DefaultAzureCredential gebruiken met Entra ID. Veiliger, maar werkt niet voor Service SAS.
// TokenCredential credential = new DefaultAzureCredential();

var serviceClient = new BlobServiceClient(new Uri("<account-url>"), credential);
var blobClient = serviceClient.GetBlobContainerClient("<container-name>").GetBlobClient("<blob-name>");

// Maak een SAS token voor de blob resource dat ook 1 dag geldig is
BlobSasBuilder sasBuilder = new BlobSasBuilder()
{
    BlobContainerName = blobClient.BlobContainerName,
    BlobName = blobClient.Name,
    Resource = "b", // HINT: in geval van ontbrekende BlobName property, dan Resource = "c"
    StartsOn = DateTimeOffset.UtcNow,
    ExpiresOn = DateTimeOffset.UtcNow.AddDays(1)
};
sasBuilder.SetPermissions(BlobSasPermissions.Read | BlobSasPermissions.Write);

////////////////////////////////////////////////////
// User Delegation SAS
////////////////////////////////////////////////////

// Vraag de user delegation key aan
// UserDelegationKey wordt gebruikt om het SAS token te signen en heeft zijn eigen geldigheid (kan gebruikt worden voor meerdere SAS)
UserDelegationKey userDelegationKey = await serviceClient.GetUserDelegationKeyAsync(
    DateTimeOffset.UtcNow,
    DateTimeOffset.UtcNow.AddDays(1));
var userSas = sasBuilder.ToSasQueryParameters(userDelegationKey, serviceClient.AccountName);
// Voeg het SAS token toe aan de blob URI
BlobUriBuilder uriBuilder = new BlobUriBuilder(blobClient.Uri) { Sas = userSas };
var blobClientSASUserDelegation = new BlobClient(uriBuilder.ToUri());

////////////////////////////////////////////////////
// Service SAS
////////////////////////////////////////////////////

Uri blobSASURIService = blobClient.GenerateSasUri(sasBuilder);
var blobClientSASService = new BlobClient(blobSASURIService);
```

Account SAS:

```cs
var sharedKeyCredential = new StorageSharedKeyCredential("<account-name>", "<account-key>");

// Maak een SAS token dat 1 dag geldig is
var sasBuilder = new AccountSasBuilder()
{
    Services = AccountSasServices.Blobs | AccountSasServices.Queues,
    ResourceTypes = AccountSasResourceTypes.Service,
    ExpiresOn = DateTimeOffset.UtcNow.AddDays(1),
    Protocol = SasProtocol.Https
};
sasBuilder.SetPermissions(AccountSasPermissions.Read | AccountSasPermissions.Write);

// Gebruik de key om het SAS token te krijgen
// OPMERKING: Je kunt sharedKeyCredential doorgeven aan ToSasQueryParameters (ook geldig voor Service SAS)
var sasToken = sasBuilder.ToSasQueryParameters(sharedKeyCredential).ToString();

// Maak een BlobServiceClient object met de account SAS toegevoegd
var blobServiceURI = $"https://{accountName}.blob.core.windows.net";
var blobServiceClientAccountSAS = new BlobServiceClient(
    new Uri($"{blobServiceURI}?{sasToken}"));
```

```sh
# Wijs de benodigde permissions toe aan de gebruiker
az role assignment create \
 --role "Storage Blob Data Contributor" \
 --assignee <email> \
 --scope "/subscriptions/<subscription>/resourceGroups/<resource-group>/providers/Microsoft.Storage/storageAccounts/<storage-account>"

# Account Level SAS: --account-name en --account-key
# Service Level SAS: --resource-types + hetzelfde als hierboven
# User Level SAS: --auth-mode login
# (ook van toepassing op Stored Access Policies)

# Genereer een user delegation SAS voor een container
az storage container generate-sas \
 --account-name <storage-account> \
 --name <container> \
 --permissions acdlrw \
 --expiry <date-time> \
 --auth-mode login \
 --as-user

# Genereer een user delegation SAS voor een blob
az storage blob generate-sas \
 --account-name <storage-account> \
 --container-name <container> \
 --name <blob> \
 --permissions acdrw \
 --expiry <date-time> \
 --auth-mode login \
 --as-user \
 --full-uri

# Intrek alle user delegation keys voor het storage account
az storage account revoke-delegation-keys \
 --name <storage-account> \
 --resource-group $resourceGroup
```
