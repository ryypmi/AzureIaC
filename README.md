# Azure-infrastruktuurin rakentaminen Bicepillä

Tässä tehtävässä saat valmiin dockerisoidun C#/.NET Web API -sovelluksen (TodoApi), joka käyttää PostgreSQL-tietokantaa. Tehtäväsi on rakentaa **Bicep-skriptit**, jotka luovat tarvittavan Azure-infrastruktuurin, sekä **deployment-skripti**, joka automatisoi sovelluksen julkaisun Azureen.

Sovellus julkaistaan Azureen **natiivina .NET-sovelluksena** (ei Docker-konttina).

## Tavoitteet

Tämän tehtävän jälkeen osaat:

- ✅ Ymmärtää Infrastructure as Code (IaC) -periaatteet
- ✅ Kirjoittaa Bicep-templateja Azure-resurssien luomiseen
- ✅ Käyttää moduuleja Bicep-koodin organisointiin
- ✅ Luoda App Service ja PostgreSQL Flexible Server Bicepillä
- ✅ Rakentaa interaktiivisen deployment-skriptin (PowerShell)
- ✅ Julkaista .NET-sovelluksen Azureen natiivina (zip deploy)
- ✅ Konfiguroida ympäristömuuttujat (connection string) App Servicessä
- ✅ Käyttää `what-if`-esikatselua turvalliseen käyttöönottoon

## Teoria-aineisto

Lue seuraavat materiaalit ennen tehtävän aloittamista:

- [Infrastructure as Code (IaC)](https://github.com/xamk-mire/Xamk-wiki/blob/main/Cloud%20technologies/Azure/Infrastructure-as-Code.md) - Mikä on IaC, miksi sitä käytetään, Bicep-perusteet ja esimerkit
- [Bicep - Azuren IaC-kieli](https://github.com/xamk-mire/Xamk-wiki/blob/main/Cloud%20technologies/Azure/Bicep.md) - Bicepin syntaksi, moduulit, funktiot ja edistyneet ominaisuudet

## Esivaatimukset

- .NET 8 SDK
- Docker Desktop (asennettu ja käynnissä -- tarvitaan lokaaliin testaukseen)
- **Azure CLI** (asennettu ja kirjautuneena)
- **Bicep** (tulee Azure CLI:n mukana)
- VS Code + **Bicep-laajennus** (`ms-azuretools.vscode-bicep`)
- Azure-tilaus (subscription) -- esim. Azure for Students

---

## Vaihe 1: Sovelluksen ymmärtäminen (~15 min)

### 1.1 Projektin kuvaus

Saat valmiin **TodoApi**-sovelluksen -- yksinkertaisen tehtävähallinnan Web API:n, joka on rakennettu ASP.NET Core Minimal API -tyylillä ja käyttää PostgreSQL-tietokantaa.

### 1.2 Projektirakenne

```
StarterCode/
├── TodoApi/
│   ├── TodoApi.csproj          ← Projekti ja riippuvuudet
│   ├── Program.cs              ← Minimal API + endpointit
│   ├── Models/
│   │   └── Todo.cs             ← Todo-entiteetti
│   ├── Data/
│   │   └── AppDbContext.cs     ← EF Core DbContext
│   └── appsettings.json        ← Konfiguraatio
├── Dockerfile                  ← Multi-stage Docker build (lokaali kehitys)
├── docker-compose.yml          ← API + PostgreSQL (lokaali kehitys)
├── .env                        ← Salasanat (ei Gitiin!)
└── .dockerignore               ← Docker build -poissulkemiset
```

### 1.3 API-endpointit

| Metodi | Polku | Kuvaus |
|---|---|---|
| `GET` | `/api/todos` | Hae kaikki todot |
| `GET` | `/api/todos/{id}` | Hae yksittäinen todo |
| `POST` | `/api/todos` | Luo uusi todo |
| `PUT` | `/api/todos/{id}` | Päivitä todo |
| `DELETE` | `/api/todos/{id}` | Poista todo |
| `GET` | `/health` | Health check |

### 1.4 Miten sovellus lukee tietokantayhteyden?

Sovellus lukee connection stringin **ympäristömuuttujasta** `ConnectionStrings__DefaultConnection` (tai `appsettings.json`-tiedostosta):

```csharp
// Program.cs
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));
```

ASP.NET Core lukee asetukset tässä järjestyksessä (myöhempi voittaa):
1. `appsettings.json`
2. `appsettings.{Environment}.json`
3. **Ympäristömuuttujat** ← Azure App Service käyttää tätä
4. Komentoriviparametrit

Kun asetamme App Serviceen ympäristömuuttujan `ConnectionStrings__DefaultConnection`, sovellus käyttää sitä automaattisesti -- koodia ei tarvitse muuttaa!

> **Kaksi alaviivaa (`__`)** on ASP.NET Coren tapa ilmaista JSON-hierarkiaa ympäristömuuttujissa. `ConnectionStrings__DefaultConnection` vastaa JSON:ia: `{ "ConnectionStrings": { "DefaultConnection": "..." } }`

### 1.5 Käynnistä ja testaa lokaalisti

```bash
cd StarterCode

# Käynnistä Docker Composella
docker compose up -d --build

# Testaa API
curl http://localhost:8080/api/todos
curl http://localhost:8080/health

# Avaa Swagger UI selaimessa
# http://localhost:8080/swagger
```

Testaa Swaggerissa:
1. **POST /api/todos** -- Luo uusi todo:
   ```json
   {
     "title": "Opettele Bicep",
     "description": "Azure IaC -tehtävä"
   }
   ```
2. **GET /api/todos** -- Varmista, että todo tallentui
3. **PUT /api/todos/{id}** -- Merkitse todo valmiiksi (`isCompleted: true`)

```bash
# Pysäytä palvelut
docker compose down
```

### 1.6 Mitä tarvitaan Azuressa?

Jotta sovellus voidaan julkaista Azureen, tarvitaan seuraavat resurssit:

```
┌─────────────────────────────────────────────────────────┐
│                    Azure                                 │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │          Resource Group (rg-todoapp-dev)         │    │
│  │                                                  │    │
│  │  ┌────────────────────────────────────────┐     │    │
│  │  │  App Service Plan (asp-todoapp-dev)    │     │    │
│  │  │                                        │     │    │
│  │  │  ┌──────────────────────────────────┐  │     │    │
│  │  │  │   Web App (.NET 8 natiivi)       │  │     │    │
│  │  │  │                                  │  │     │    │
│  │  │  │   Ympäristömuuttujat:            │  │     │    │
│  │  │  │   ConnectionStrings__             │  │     │    │
│  │  │  │     DefaultConnection = ...      │──┼─┐   │    │
│  │  │  └──────────────────────────────────┘  │ │   │    │
│  │  └────────────────────────────────────────┘ │   │    │
│  │                                              │   │    │
│  │  ┌───────────────────────────────────────┐  │   │    │
│  │  │  PostgreSQL Flexible Server           │◄─┘   │    │
│  │  │  (psql-todoapp-dev)                   │      │    │
│  │  │                                       │      │    │
│  │  │  tododb-tietokanta                    │      │    │
│  │  └───────────────────────────────────────┘      │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

| Azure-resurssi | Miksi tarvitaan? | Docker Compose -vastaavuus |
|---|---|---|
| **Resource Group** | Kaikkien resurssien "kansio" | - |
| **App Service Plan** | Laskentakapasiteetti (CPU, muisti) | Docker host |
| **Web App** | .NET-sovelluksen ajo + ympäristömuuttujat | `api`-palvelu + `environment`-osio |
| **PostgreSQL Flexible Server** | Hallittu tietokanta | `postgres`-palvelu |

> **Huomaa yhteys:** Docker Composessa connection string asetetaan `environment`-osiossa. Azuressa **sama asia** tehdään App Servicen ympäristömuuttujilla -- sovelluksen koodi on identtinen!

### ✅ Tarkistuslista ennen jatkamista

- [ ] Docker Compose käynnistää sovelluksen lokaalisti
- [ ] API vastaa osoitteessa http://localhost:8080/swagger
- [ ] Ymmärrät, miten sovellus lukee connection stringin ympäristömuuttujasta
- [ ] Ymmärrät, mitä Azure-resursseja tarvitaan

---

## Vaihe 2: Azure-ympäristön valmistelu (~15 min)

### 2.1 Asenna työkalut

```powershell
# Tarkista Azure CLI
az --version

# Jos ei asennettu:
winget install Microsoft.AzureCLI

# Tarkista Bicep
az bicep version

# Päivitä Bicep uusimpaan
az bicep upgrade
```

### 2.2 Kirjaudu Azureen

```powershell
# Kirjaudu sisään (avaa selaimen)
az login

# Tarkista aktiivinen subscription
az account show --output table

# Listaa kaikki subscriptionit
az account list --output table
```

### 2.3 Asenna VS Code Bicep -laajennus

Asenna **Bicep**-laajennus VS Codeen (Extension ID: `ms-azuretools.vscode-bicep`). Tämä antaa IntelliSense-tuen, validoinnin ja virheilmoitukset Bicep-tiedostoille.

### 2.4 Luo infra-kansiorakenne

Luo seuraava kansiorakenne `StarterCode`-kansioon:

```
StarterCode/
├── (olemassa olevat tiedostot...)
└── infra/
    ├── main.bicep              ← Päätemplate (luot tämän)
    ├── main.bicepparam         ← Parametrit (luot tämän)
    └── modules/
        ├── appservice.bicep    ← App Service (luot tämän)
        └── postgresql.bicep    ← PostgreSQL (luot tämän)
```

```powershell
# Luo kansiot
mkdir infra/modules
```

### ✅ Tarkistuslista ennen jatkamista

- [ ] Azure CLI asennettu ja kirjautuneena
- [ ] Bicep asennettu ja päivitetty
- [ ] VS Code Bicep -laajennus asennettu
- [ ] `infra/`-kansiorakenne luotu

---

## Vaihe 3: Ensimmäinen Bicep -- Resource Group (~20 min)

Aloitetaan yksinkertaisesta: luodaan pelkkä Resource Group.

### 3.1 Mikä on Resource Group?

Resource Group on Azuren "kansio", johon kaikki toisiinsa liittyvät resurssit sijoitetaan. Se helpottaa resurssien hallintaa, kustannusten seurantaa ja poistamista.

### 3.2 Luo main.bicep

Luo tiedosto `infra/main.bicep`:

```bicep
// infra/main.bicep
// Kuvaus: TodoApp Azure-infrastruktuurin päätemplate
targetScope = 'subscription'

// ─── PARAMETRIT ───

@description('Sovelluksen nimi, käytetään resurssien nimeämisessä')
param appName string

@allowed(['dev', 'prod'])
@description('Ympäristö')
param environment string = 'dev'

@description('Azure-sijainti')
param location string = 'northeurope'

// ─── MUUTTUJAT ───

var resourceGroupName = 'rg-${appName}-${environment}'

var tags = {
  Application: appName
  Environment: environment
  ManagedBy: 'Bicep'
}

// ─── RESURSSIT ───

// Resource Group
resource rg 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name: resourceGroupName
  location: location
  tags: tags
}

// ─── TULOSTEET ───

output resourceGroupName string = rg.name
output resourceGroupId string = rg.id
```

**Rivi riviltä selitettynä:**

| Rivi | Selitys |
|---|---|
| `targetScope = 'subscription'` | Deployment kohdistuu subscription-tasolle (ei yksittäiseen resource groupiin), koska luomme itse resource groupin |
| `param appName string` | Pakollinen parametri -- sovelluksen nimi |
| `@allowed(['dev', 'prod'])` | **Dekoraattori** -- rajoittaa parametrin sallitut arvot |
| `var resourceGroupName = ...` | **Muuttuja** -- lasketaan parametreista, noudattaa Azure-nimeämiskäytäntöä (`rg-<app>-<env>`) |
| `var tags = { ... }` | **Tagit** -- metatietoja resurssien organisointiin ja kustannusseurantaan |
| `resource rg ...` | Luo Resource Groupin Azureen |
| `output resourceGroupName ...` | **Tuloste** -- palauttaa luodun resurssin nimen, jota voidaan käyttää skripteissä |

### 3.3 Luo parametritiedosto

Luo tiedosto `infra/main.bicepparam`:

```bicep
using 'main.bicep'

param appName = 'todoapp'
param environment = 'dev'
param location = 'northeurope'
```

> **Miksi parametritiedosto?** Se erottaa arvot templatesta. Tuotantoympäristölle voi luoda oman tiedoston `main.prod.bicepparam` eri arvoilla.

### 3.4 Testaa: What-if

`what-if` näyttää mitä muutoksia deployment tekisi **ilman, että mitään muutetaan**:

```powershell
az deployment sub what-if `
  --location northeurope `
  --template-file infra/main.bicep `
  --parameters infra/main.bicepparam
```

Tulosteessa näet:

```
Resource changes: 1 to create

  + Microsoft.Resources/resourceGroups/rg-todoapp-dev [2023-07-01]

      location: "northeurope"
      tags.Application: "todoapp"
      tags.Environment: "dev"
      tags.ManagedBy:   "Bicep"
```

### 3.5 Deploy!

```powershell
az deployment sub create `
  --location northeurope `
  --template-file infra/main.bicep `
  --parameters infra/main.bicepparam `
  --name "deploy-rg-$(Get-Date -Format 'yyyyMMdd-HHmmss')"
```

### 3.6 Tarkista Azure Portalissa

1. Avaa [Azure Portal](https://portal.azure.com)
2. Etsi "Resource groups"
3. Varmista, että `rg-todoapp-dev` on luotu oikeilla tageilla

```powershell
# Tai tarkista CLI:llä
az group show --name rg-todoapp-dev --output table
```

> **Kokeile idempotenssia!** Suorita sama deployment uudelleen. Huomaat, että mitään ei muutu -- Bicep tunnistaa, että resurssit ovat jo olemassa.

### ✅ Tarkistuslista ennen jatkamista

- [ ] `main.bicep` luotu ja syntaksi on oikein
- [ ] `what-if` näyttää 1 resurssin luontiin
- [ ] Resource Group `rg-todoapp-dev` luotu Azureen
- [ ] Tagit näkyvät Azure Portalissa

---

## Vaihe 4: PostgreSQL-moduuli (~25 min)

Nyt luomme ensimmäisen moduulin. **Azure Database for PostgreSQL Flexible Server** on hallittu PostgreSQL-palvelu -- Azure hoitaa päivitykset, varmuuskopiot ja korkean saatavuuden puolestasi.

### 4.1 Miksi moduuli?

Moduulit mahdollistavat koodin uudelleenkäytön ja organisoinnin. Sen sijaan, että kaikki resurssit olisivat yhdessä valtavassa tiedostossa, kukin resurssikokonaisuus on omassa tiedostossaan.

```
main.bicep              ← "Kapellimestari" -- kutsuu moduuleja
  ├── postgresql.bicep  ← Tietokantamoduuli
  └── appservice.bicep  ← App Service -moduuli
```

### 4.2 Miksi hallittu tietokanta?

| | Docker-kontti (lokaali) | Azure Flexible Server |
|---|---|---|
| **Varmuuskopiot** | Sinun vastuullasi | Automaattiset (7-35 pv) |
| **Päivitykset** | Manuaaliset | Automaattiset |
| **Skaalaus** | Manuaalinen | Yhdellä klikkauksella |
| **Korkea saatavuus** | Ei | Zone-redundant |
| **Monitorointi** | Itse konfiguroitava | Sisäänrakennettu |
| **Hinta** | Ilmainen | Burstable B1ms ~12$/kk |

### 4.3 Luo PostgreSQL-moduuli

Luo tiedosto `infra/modules/postgresql.bicep`:

```bicep
// infra/modules/postgresql.bicep
// Kuvaus: Azure Database for PostgreSQL Flexible Server

@description('Azure-sijainti')
param location string

@description('Ympäristö')
param environment string

@description('Sovelluksen nimi')
param appName string

@secure()
@description('PostgreSQL-ylläpitäjän salasana')
param administratorPassword string

@description('PostgreSQL-ylläpitäjän käyttäjätunnus')
param administratorLogin string = 'pgadmin'

// Palvelimen nimi: globaalisti uniikki
var serverName = 'psql-${appName}-${environment}-${uniqueString(resourceGroup().id)}'

// PostgreSQL Flexible Server
resource postgresServer 'Microsoft.DBforPostgreSQL/flexibleServers@2022-12-01' = {
  name: serverName
  location: location
  sku: {
    name: 'Standard_B1ms'    // Burstable -- halvin vaihtoehto (~12$/kk)
    tier: 'Burstable'
  }
  properties: {
    version: '16'
    administratorLogin: administratorLogin
    administratorLoginPassword: administratorPassword
    storage: {
      storageSizeGB: 32
    }
    backup: {
      backupRetentionDays: 7
      geoRedundantBackup: 'Disabled'    // Kehityksessä ei tarvita
    }
    highAvailability: {
      mode: 'Disabled'                  // Kehityksessä ei tarvita
    }
  }
  tags: {
    Application: appName
    Environment: environment
    ManagedBy: 'Bicep'
  }
}

// Firewall: Salli pääsy Azure-palveluista (App Service)
resource firewallRuleAzure 'Microsoft.DBforPostgreSQL/flexibleServers/firewallRules@2022-12-01' = {
  parent: postgresServer
  name: 'AllowAzureServices'
  properties: {
    startIpAddress: '0.0.0.0'
    endIpAddress: '0.0.0.0'
  }
}

// Tietokanta
resource database 'Microsoft.DBforPostgreSQL/flexibleServers/databases@2022-12-01' = {
  parent: postgresServer
  name: 'tododb'
  properties: {
    charset: 'UTF8'
    collation: 'en_US.utf8'
  }
}

// Tulosteet
output serverName string = postgresServer.name
output serverFqdn string = postgresServer.properties.fullyQualifiedDomainName
output databaseName string = database.name

// Connection string App Servicelle (ympäristömuuttuja)
output connectionString string = 'Host=${postgresServer.properties.fullyQualifiedDomainName};Port=5432;Database=tododb;Username=${administratorLogin};Password=${administratorPassword}'
```

**Avainkohtia:**

| Kohta | Selitys |
|---|---|
| `@secure() param administratorPassword` | `@secure()` -dekoraattori varmistaa, ettei salasana näy lokeissa tai deployment-historiassa |
| `sku: 'Standard_B1ms'` | **Burstable** on halvin taso -- soveltuu kehitykseen ja pieniin työkuormiin |
| `firewallRuleAzure: '0.0.0.0' - '0.0.0.0'` | Erityinen sääntö, joka sallii kaikkien Azure-palveluiden (kuten App Servicen) pääsyn tietokantaan |
| `output connectionString` | Koottu connection string, joka asetetaan App Servicen ympäristömuuttujaksi |

> **Huomaa connection stringin rakenne:** Se on täsmälleen sama formaatti kuin `docker-compose.yml`:n `ConnectionStrings__DefaultConnection`, mutta `Host` osoittaa Azure PostgreSQL -palvelimeen Docker-kontin sijaan.

### 4.4 Päivitä main.bicep

Lisää PostgreSQL-moduuli ja uusi parametri `main.bicep`-tiedostoon:

```bicep
// infra/main.bicep
targetScope = 'subscription'

// ─── PARAMETRIT ───

@description('Sovelluksen nimi, käytetään resurssien nimeämisessä')
param appName string

@allowed(['dev', 'prod'])
@description('Ympäristö')
param environment string = 'dev'

@description('Azure-sijainti')
param location string = 'northeurope'

@secure()
@description('PostgreSQL-ylläpitäjän salasana')
param dbPassword string

// ─── MUUTTUJAT ───

var resourceGroupName = 'rg-${appName}-${environment}'

var tags = {
  Application: appName
  Environment: environment
  ManagedBy: 'Bicep'
}

// ─── RESURSSIT ───

// Resource Group
resource rg 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name: resourceGroupName
  location: location
  tags: tags
}

// PostgreSQL Flexible Server
module postgresql 'modules/postgresql.bicep' = {
  name: 'postgresqlDeployment'
  scope: rg
  params: {
    location: location
    environment: environment
    appName: appName
    administratorPassword: dbPassword
  }
}

// ─── TULOSTEET ───

output resourceGroupName string = rg.name
output postgresServerName string = postgresql.outputs.serverName
output postgresServerFqdn string = postgresql.outputs.serverFqdn
output postgresConnectionString string = postgresql.outputs.connectionString
```

### 4.5 Päivitä parametritiedosto

Päivitä `infra/main.bicepparam`:

```bicep
using 'main.bicep'

param appName = 'todoapp'
param environment = 'dev'
param location = 'northeurope'
param dbPassword = readEnvironmentVariable('DB_PASSWORD', '')
```

> **Huomaa:** `readEnvironmentVariable` lukee salasanan ympäristömuuttujasta -- salasana ei ole koodissa!

### 4.6 Testaa ja deploy

```powershell
# Aseta salasana ympäristömuuttujaksi
$env:DB_PASSWORD = "TurvAllinenSalasana123!"

# Esikatselu
az deployment sub what-if `
  --location northeurope `
  --template-file infra/main.bicep `
  --parameters infra/main.bicepparam

# Deploy
az deployment sub create `
  --location northeurope `
  --template-file infra/main.bicep `
  --parameters infra/main.bicepparam `
  --name "deploy-db-$(Get-Date -Format 'yyyyMMdd-HHmmss')"
```

> **Huom:** PostgreSQL Flexible Serverin luominen kestää 5-10 minuuttia. Tämä on normaalia!

### ✅ Tarkistuslista ennen jatkamista

- [ ] `postgresql.bicep` moduuli luotu
- [ ] `main.bicep` päivitetty kutsumaan PostgreSQL-moduulia
- [ ] `@secure()` parametri käytössä salasanalle
- [ ] PostgreSQL-palvelin näkyy Azure Portalissa
- [ ] Firewall-sääntö sallii Azure-palveluiden pääsyn

---

## Vaihe 5: App Service -moduuli (~25 min)

Nyt luomme **App Servicen**, joka ajaa .NET-sovelluksen natiivisti Azuressa. App Service on PaaS-palvelu (Platform as a Service) -- sinun ei tarvitse hallita palvelimia, vaan Azure hoitaa infrastruktuurin.

### 5.1 App Service Plan vs. Web App

```
App Service Plan (laskentaresurssi):
┌──────────────────────────────────────┐
│  CPU: 1 core                         │
│  RAM: 1.75 GB                        │
│  Taso: B1 (Basic)                    │
│                                      │
│  ┌──────────────┐                    │
│  │   Web App    │ ← .NET 8 runtime   │
│  │  (todoapi)   │   pyörii tässä     │
│  └──────────────┘                    │
└──────────────────────────────────────┘
```

- **App Service Plan** = "palvelin" (CPU, muisti, hinta)
- **Web App** = sovellus, joka pyörii Planin päällä

### 5.2 Miten ympäristömuuttujat toimivat?

App Servicessa sovelluksen asetukset (App Settings) näkyvät sovellukselle **ympäristömuuttujina**. Tämä on täsmälleen sama tapa kuin Docker Composessa:

```
Docker Compose (lokaali):
─────────────────────────
environment:
  - ConnectionStrings__DefaultConnection=Host=postgres;Port=5432;...

Azure App Service (pilvi):
──────────────────────────
App Settings:
  ConnectionStrings__DefaultConnection = Host=psql-todoapp-dev.postgres.database.azure.com;Port=5432;...

→ Sovelluksen koodi on IDENTTINEN molemmissa! ASP.NET Core lukee
  ympäristömuuttujan automaattisesti Configuration-rajapinnan kautta.
```

### 5.3 Luo App Service -moduuli

Luo tiedosto `infra/modules/appservice.bicep`:

```bicep
// infra/modules/appservice.bicep
// Kuvaus: App Service Plan + Web App (.NET 8 natiivi)

@description('Azure-sijainti')
param location string

@description('Ympäristö')
param environment string

@description('Sovelluksen nimi')
param appName string

@description('PostgreSQL connection string')
@secure()
param databaseConnectionString string

// ─── MUUTTUJAT ───

var appServicePlanName = 'asp-${appName}-${environment}'
var webAppName = 'app-${appName}-${environment}-${uniqueString(resourceGroup().id)}'

// ─── APP SERVICE PLAN ───

resource appServicePlan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: appServicePlanName
  location: location
  kind: 'linux'
  sku: {
    name: 'B1'              // Basic -- halvin taso joka tukee AlwaysOn:ia
  }
  properties: {
    reserved: true           // true = Linux
  }
  tags: {
    Application: appName
    Environment: environment
    ManagedBy: 'Bicep'
  }
}

// ─── WEB APP ───

resource webApp 'Microsoft.Web/sites@2023-12-01' = {
  name: webAppName
  location: location
  kind: 'app,linux'
  properties: {
    serverFarmId: appServicePlan.id
    httpsOnly: true
    siteConfig: {
      linuxFxVersion: 'DOTNETCORE|8.0'    // Natiivi .NET 8 runtime
      alwaysOn: true
      minTlsVersion: '1.2'
      ftpsState: 'Disabled'
      healthCheckPath: '/health'           // Käyttää sovelluksen health-endpointia
      appSettings: [
        {
          // ─── TÄMÄ ON AVAINASETUS ───
          // Asettaa tietokantayhteyden ympäristömuuttujaksi.
          // ASP.NET Core lukee tämän automaattisesti:
          //   builder.Configuration.GetConnectionString("DefaultConnection")
          name: 'ConnectionStrings__DefaultConnection'
          value: databaseConnectionString
        }
        {
          name: 'ASPNETCORE_ENVIRONMENT'
          value: environment == 'prod' ? 'Production' : 'Development'
        }
      ]
    }
  }
  tags: {
    Application: appName
    Environment: environment
    ManagedBy: 'Bicep'
  }
}

// ─── TULOSTEET ───

output webAppName string = webApp.name
output webAppUrl string = 'https://${webApp.properties.defaultHostName}'
```

**Avainkohtia:**

| Kohta | Selitys |
|---|---|
| `kind: 'app,linux'` | Kertoo Azurelle, että tämä on Linux-pohjainen .NET-sovellus (ei kontti) |
| `linuxFxVersion: 'DOTNETCORE\|8.0'` | Asettaa .NET 8 runtimen -- Azure tarjoaa runtimen, sinun ei tarvitse pakata sitä mukaan |
| `healthCheckPath: '/health'` | Azure kutsuu tätä endpointia säännöllisesti tarkistaakseen, onko sovellus kunnossa. Jos endpoint ei vastaa, Azure käynnistää sovelluksen uudelleen. |
| `ConnectionStrings__DefaultConnection` | **Tämä on ympäristömuuttuja**, jonka ASP.NET Core lukee automaattisesti. Se ylikirjoittaa `appsettings.json`:n arvon. Sovelluksen koodi ei muutu! |
| `alwaysOn: true` | Pitää sovelluksen käynnissä, ei sammuta sitä käyttämättömyyden takia |
| `@secure() param databaseConnectionString` | Connection string sisältää salasanan, joten se merkitään turvalliseksi |

### 5.4 Päivitä main.bicep -- lopullinen versio

Päivitä `infra/main.bicep` lisäämällä App Service -moduuli:

```bicep
// infra/main.bicep
// Kuvaus: TodoApp Azure-infrastruktuurin päätemplate
targetScope = 'subscription'

// ─── PARAMETRIT ───

@description('Sovelluksen nimi, käytetään resurssien nimeämisessä')
param appName string

@allowed(['dev', 'prod'])
@description('Ympäristö')
param environment string = 'dev'

@description('Azure-sijainti')
param location string = 'northeurope'

@secure()
@description('PostgreSQL-ylläpitäjän salasana')
param dbPassword string

// ─── MUUTTUJAT ───

var resourceGroupName = 'rg-${appName}-${environment}'

var tags = {
  Application: appName
  Environment: environment
  ManagedBy: 'Bicep'
}

// ─── RESURSSIT ───

// 1. Resource Group
resource rg 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name: resourceGroupName
  location: location
  tags: tags
}

// 2. PostgreSQL Flexible Server
module postgresql 'modules/postgresql.bicep' = {
  name: 'postgresqlDeployment'
  scope: rg
  params: {
    location: location
    environment: environment
    appName: appName
    administratorPassword: dbPassword
  }
}

// 3. App Service (Web App, .NET 8 natiivi)
module appService 'modules/appservice.bicep' = {
  name: 'appServiceDeployment'
  scope: rg
  params: {
    location: location
    environment: environment
    appName: appName
    databaseConnectionString: postgresql.outputs.connectionString
  }
}

// ─── TULOSTEET ───

output resourceGroupName string = rg.name
output postgresServerFqdn string = postgresql.outputs.serverFqdn
output webAppName string = appService.outputs.webAppName
output webAppUrl string = appService.outputs.webAppUrl
```

**Huomaa moduulien ketjutus:**

```
postgresql.bicep ─── outputs.connectionString ───► appservice.bicep
                                                      │
    Connection string asetetaan                       │
    Web Appin ympäristömuuttujaksi:                   ▼
    ConnectionStrings__DefaultConnection = Host=psql-...;Port=5432;Database=tododb;...
```

Bicep ymmärtää automaattisesti, että App Service riippuu PostgreSQL:stä (koska se käyttää sen tulostetta), ja luo ne oikeassa järjestyksessä.

### 5.5 Deploy koko infrastruktuuri

```powershell
# Aseta salasana
$env:DB_PASSWORD = "TurvAllinenSalasana123!"

# Esikatselu -- tarkista AINA ensin!
az deployment sub what-if `
  --location northeurope `
  --template-file infra/main.bicep `
  --parameters infra/main.bicepparam

# Deploy
az deployment sub create `
  --location northeurope `
  --template-file infra/main.bicep `
  --parameters infra/main.bicepparam `
  --name "deploy-full-$(Get-Date -Format 'yyyyMMdd-HHmmss')"
```

> **Huom:** Koko infran deployment kestää ~10-15 minuuttia (PostgreSQL on hitain).

### 5.6 Tarkista tulosteet

```powershell
# Näytä viimeisen deploymentin tulosteet
az deployment sub show `
  --name "deploy-full-<aikaleima>" `
  --query properties.outputs `
  --output table
```

### ✅ Tarkistuslista ennen jatkamista

- [ ] `appservice.bicep` moduuli luotu
- [ ] Connection string asetetaan App Servicen ympäristömuuttujaksi Bicepissä
- [ ] `main.bicep` sisältää kaikki 3 resurssia (RG, PostgreSQL, App Service)
- [ ] Moduulit ketjutettu oikein (PostgreSQL → App Service connection string)
- [ ] Koko infra deploydattu Azureen
- [ ] Kaikki resurssit näkyvät Azure Portalissa `rg-todoapp-dev`:ssä

---

## Vaihe 6: Deployment-skripti (~35 min)

Nyt rakennamme **interaktiivisen PowerShell-skriptin**, joka automatisoi koko julkaisuprosessin: subscription-valinta, infrastruktuurin luonti, sovelluksen julkaisu ja ympäristömuuttujien konfigurointi.

### 6.1 Deployment-skriptin tarkoitus

```
deploy.ps1 suorittaa:

1. 🔐 Tarkista Azure-kirjautuminen
2. 📋 Listaa subscriptionit → käyttäjä valitsee
3. 👁️ What-if esikatselu → käyttäjä hyväksyy
4. 🏗️ Deploy Bicep-infrastruktuuri
5. 📋 Näytä luodut resurssit
6. 📦 Julkaise .NET-sovellus (dotnet publish + zip deploy)
7. ✅ Näytä tulos ja URL
```

### 6.2 Miten natiivi .NET deploy toimii?

Toisin kuin Docker-konttijulkaisussa, natiivissa julkaisussa:

```
Lokaali kehitys (Docker Compose):
┌─────────────┐    docker build     ┌──────────────┐    docker run     ┌─────────┐
│ Lähdekoodi  │ ─────────────────→ │ Docker image │ ─────────────→  │ Kontti  │
└─────────────┘                     └──────────────┘                 └─────────┘

Azure-julkaisu (natiivi .NET):
┌─────────────┐   dotnet publish    ┌──────────────┐   az webapp deploy  ┌───────────┐
│ Lähdekoodi  │ ─────────────────→ │  ZIP-paketti │ ─────────────────→ │ App       │
└─────────────┘   (käännetty DLL)   └──────────────┘   (zip deploy)     │ Service   │
                                                                         │ (.NET 8)  │
                                                                         └───────────┘
```

1. **`dotnet publish`** -- kääntää sovelluksen ja luo DLL-tiedostot
2. **ZIP-paketti** -- paketoidaan DLL:t zip-tiedostoon
3. **`az webapp deploy`** -- lähetetään zip App Serviceen, joka purkaa sen ja käynnistää

### 6.3 Luo deploy.ps1

Luo tiedosto `StarterCode/deploy.ps1`:

```powershell
#!/usr/bin/env pwsh
# deploy.ps1 -- TodoApp Azure Deployment Script
# Käyttö: ./deploy.ps1

param(
    [string]$Environment = "dev"
)

$ErrorActionPreference = "Stop"

Write-Host ""
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "  TodoApp Azure Deployment" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host ""

# ─── 1. TARKISTA ESIVAATIMUKSET ───

Write-Host "[1/7] Tarkistetaan esivaatimukset..." -ForegroundColor Yellow

# Tarkista Azure CLI
if (-not (Get-Command az -ErrorAction SilentlyContinue)) {
    Write-Host "VIRHE: Azure CLI ei ole asennettu!" -ForegroundColor Red
    Write-Host "Asenna: winget install Microsoft.AzureCLI" -ForegroundColor Gray
    exit 1
}

# Tarkista .NET SDK
if (-not (Get-Command dotnet -ErrorAction SilentlyContinue)) {
    Write-Host "VIRHE: .NET SDK ei ole asennettu!" -ForegroundColor Red
    exit 1
}

# Tarkista kirjautuminen
$account = az account show 2>$null | ConvertFrom-Json
if (-not $account) {
    Write-Host "Ei kirjautuneena Azureen. Kirjaudutaan..." -ForegroundColor Yellow
    az login
    $account = az account show | ConvertFrom-Json
}

Write-Host "  Kirjautuneena: $($account.user.name)" -ForegroundColor Green
Write-Host ""

# ─── 2. VALITSE SUBSCRIPTION ───

Write-Host "[2/7] Valitse Azure-subscription:" -ForegroundColor Yellow
Write-Host ""

$subscriptions = az account list --query "[].{Name:name, Id:id, IsDefault:isDefault}" | ConvertFrom-Json

for ($i = 0; $i -lt $subscriptions.Count; $i++) {
    $sub = $subscriptions[$i]
    $marker = if ($sub.IsDefault) { " (nykyinen)" } else { "" }
    Write-Host "  [$i] $($sub.Name)$marker" -ForegroundColor White
}

Write-Host ""
$selection = Read-Host "Valitse numero (Enter = nykyinen)"

if ($selection -ne "") {
    $selectedSub = $subscriptions[[int]$selection]
    az account set --subscription $selectedSub.Id
    Write-Host "  Subscription asetettu: $($selectedSub.Name)" -ForegroundColor Green
} else {
    $selectedSub = $subscriptions | Where-Object { $_.IsDefault -eq $true }
    Write-Host "  Käytetään nykyistä: $($selectedSub.Name)" -ForegroundColor Green
}
Write-Host ""

# ─── 3. KYSY SALASANA ───

Write-Host "[3/7] PostgreSQL-salasana:" -ForegroundColor Yellow

if (-not $env:DB_PASSWORD) {
    $securePassword = Read-Host "  Anna PostgreSQL admin -salasana" -AsSecureString
    $env:DB_PASSWORD = [Runtime.InteropServices.Marshal]::PtrToStringAuto(
        [Runtime.InteropServices.Marshal]::SecureStringToBSTR($securePassword)
    )
}
Write-Host "  Salasana asetettu" -ForegroundColor Green
Write-Host ""

# ─── 4. WHAT-IF ESIKATSELU ───

Write-Host "[4/7] Infrastruktuurin esikatselu (what-if)..." -ForegroundColor Yellow
Write-Host ""

az deployment sub what-if `
    --location northeurope `
    --template-file infra/main.bicep `
    --parameters infra/main.bicepparam

Write-Host ""
$confirm = Read-Host "Haluatko jatkaa deploymenttia? (y/n)"
if ($confirm -ne "y") {
    Write-Host "Deployment peruttu." -ForegroundColor Yellow
    exit 0
}
Write-Host ""

# ─── 5. DEPLOY INFRASTRUKTUURI ───

Write-Host "[5/7] Deploydaan infrastruktuuri Bicepilla..." -ForegroundColor Yellow

$deploymentName = "deploy-$(Get-Date -Format 'yyyyMMdd-HHmmss')"

$ErrorActionPreference = "SilentlyContinue"
$rawOutput = az deployment sub create `
    --location northeurope `
    --template-file infra/main.bicep `
    --parameters infra/main.bicepparam `
    --name $deploymentName `
    --query properties.outputs `
    --output json 2>&1
$azExitCode = $LASTEXITCODE
$ErrorActionPreference = "Stop"

if ($azExitCode -ne 0) {
    Write-Host "ERROR: Deployment failed (exit code $azExitCode)." -ForegroundColor Red
    $rawOutput | ForEach-Object { Write-Host $_ }
    exit 1
}

# Filter out non-JSON lines (e.g. "Bicep CLI is already installed...", warnings)
$jsonString = ($rawOutput | Where-Object { $_ -isnot [System.Management.Automation.ErrorRecord] }) -join "`n"
$firstBrace = $jsonString.IndexOf('{')
if ($firstBrace -ge 0) {
    $deployment = $jsonString.Substring($firstBrace) | ConvertFrom-Json
} else {
    Write-Host "ERROR: Deployment returned no output." -ForegroundColor Red
    $rawOutput | ForEach-Object { Write-Host $_ }
    exit 1
}

$rgName = $deployment.resourceGroupName.value
$webAppName = $deployment.webAppName.value
$webAppUrl = $deployment.webAppUrl.value

Write-Host ""
Write-Host "  Resource Group:  $rgName" -ForegroundColor Green
Write-Host "  PostgreSQL:      $($deployment.postgresServerFqdn.value)" -ForegroundColor Green
Write-Host "  Web App:         $webAppName" -ForegroundColor Green
Write-Host "  URL:             $webAppUrl" -ForegroundColor Green
Write-Host ""

# ─── 6. NÄYTÄ RESURSSIT JA JULKAISE SOVELLUS ───

Write-Host "[6/7] Luodut Azure-resurssit:" -ForegroundColor Yellow
Write-Host ""

az resource list --resource-group $rgName --output table

Write-Host ""
Write-Host "Julkaistaan .NET-sovellus App Serviceen..." -ForegroundColor Yellow
Write-Host ""

# Käännä sovellus
Write-Host "  Käännetään sovellus (dotnet publish)..." -ForegroundColor Gray
dotnet publish TodoApi/TodoApi.csproj -c Release -o ./publish --nologo --verbosity quiet

# Paketoi ZIP-tiedostoksi (manually create ZIP with forward slashes for Linux compatibility)
Write-Host "  Paketoidaan ZIP-tiedostoksi..." -ForegroundColor Gray
if (Test-Path ./publish.zip) { Remove-Item ./publish.zip }
Add-Type -AssemblyName System.IO.Compression
$publishPath = (Resolve-Path ./publish).Path
$zipPath = Join-Path (Get-Location) "publish.zip"
$zip = [System.IO.Compression.ZipFile]::Open($zipPath, [System.IO.Compression.ZipArchiveMode]::Create)
try {
    $files = Get-ChildItem -Path $publishPath -Recurse -File
    foreach ($file in $files) {
        $relativePath = $file.FullName.Substring($publishPath.Length + 1)
        $entryName = $relativePath.Replace('\', '/')
        [System.IO.Compression.ZipFileExtensions]::CreateEntryFromFile(
            $zip, $file.FullName, $entryName,
            [System.IO.Compression.CompressionLevel]::Optimal
        ) | Out-Null
    }
} finally {
    $zip.Dispose()
}

# Lähetä App Serviceen (zip deploy)
Write-Host "  Lähetetään App Serviceen (zip deploy)..." -ForegroundColor Gray
az webapp deploy `
    --resource-group $rgName `
    --name $webAppName `
    --src-path ./publish.zip `
    --type zip

# Siivoa väliaikaiset tiedostot
Remove-Item -Recurse -Force ./publish
Remove-Item ./publish.zip

Write-Host "  Sovellus julkaistu!" -ForegroundColor Green
Write-Host ""

# ─── 7. VALMIS ───

Write-Host "========================================" -ForegroundColor Green
Write-Host "  Deployment valmis!" -ForegroundColor Green
Write-Host "========================================" -ForegroundColor Green
Write-Host ""
Write-Host "  Sovelluksen URL:  $webAppUrl" -ForegroundColor Cyan
Write-Host "  Swagger UI:      $webAppUrl/swagger" -ForegroundColor Cyan
Write-Host "  Health check:    $webAppUrl/health" -ForegroundColor Cyan
Write-Host ""
Write-Host "  Huom: Ensimmäinen käynnistys voi kestää 1-2 minuuttia." -ForegroundColor Yellow
Write-Host ""
Write-Host "  Ympäristömuuttujat (App Settings) asetettu Bicepillä:" -ForegroundColor Yellow
Write-Host "    ConnectionStrings__DefaultConnection = Host=psql-...;..." -ForegroundColor Gray
Write-Host "    ASPNETCORE_ENVIRONMENT = Development" -ForegroundColor Gray
Write-Host ""
Write-Host "  Siivoa resurssit kun olet valmis:" -ForegroundColor Yellow
Write-Host "  ./cleanup.ps1" -ForegroundColor Gray
Write-Host ""
```

### 6.4 Luo cleanup.ps1

Luo tiedosto `StarterCode/cleanup.ps1` resurssien poistamiseen:

```powershell
#!/usr/bin/env pwsh
# cleanup.ps1 -- Poistaa kaikki TodoApp Azure-resurssit
# Käyttö: ./cleanup.ps1

$ErrorActionPreference = "Stop"

$rgName = "rg-todoapp-dev"

Write-Host ""
Write-Host "========================================" -ForegroundColor Red
Write-Host "  TodoApp Azure Cleanup" -ForegroundColor Red
Write-Host "========================================" -ForegroundColor Red
Write-Host ""

# Tarkista, onko resource group olemassa
$exists = az group exists --name $rgName
if ($exists -eq "false") {
    Write-Host "Resource Group '$rgName' ei ole olemassa." -ForegroundColor Yellow
    exit 0
}

# Näytä resurssit
Write-Host "Seuraavat resurssit poistetaan:" -ForegroundColor Yellow
Write-Host ""
az resource list --resource-group $rgName --output table
Write-Host ""

# Vahvistus
$confirm = Read-Host "Haluatko varmasti poistaa KAIKKI resurssit? (kirjoita 'delete' vahvistaaksesi)"
if ($confirm -ne "delete") {
    Write-Host "Peruttu." -ForegroundColor Yellow
    exit 0
}

# Poista
Write-Host ""
Write-Host "Poistetaan resource group '$rgName'..." -ForegroundColor Yellow
az group delete --name $rgName --yes --no-wait

Write-Host ""
Write-Host "Poistopyyntö lähetetty! Poistuminen kestää muutaman minuutin." -ForegroundColor Green
Write-Host "Seuraa tilannetta Azure Portalissa tai komennolla:" -ForegroundColor Gray
Write-Host "  az group show --name $rgName --query properties.provisioningState" -ForegroundColor Gray
Write-Host ""
```

### 6.5 Skriptin suorittaminen

```powershell
# Varmista, että olet StarterCode-kansiossa
cd StarterCode

# Suorita deployment-skripti
./deploy.ps1
```

Skripti ohjaa sinut vaihe vaiheelta:

```
========================================
  TodoApp Azure Deployment
========================================

[1/7] Tarkistetaan esivaatimukset...
  Kirjautuneena: opiskelija@xamk.fi

[2/7] Valitse Azure-subscription:
  [0] Azure for Students (nykyinen)
  [1] Visual Studio Enterprise

Valitse numero (Enter = nykyinen): ⏎

[3/7] PostgreSQL-salasana:
  Anna PostgreSQL admin -salasana: ********

[4/7] Infrastruktuurin esikatselu (what-if)...
  + Microsoft.Resources/resourceGroups/rg-todoapp-dev
  + Microsoft.DBforPostgreSQL/flexibleServers/psql-todoapp-dev-...
  + Microsoft.Web/serverfarms/asp-todoapp-dev
  + Microsoft.Web/sites/app-todoapp-dev-...

Haluatko jatkaa deploymenttia? (y/n): y

[5/7] Deploydaan infrastruktuuri Bicepilla...
  Resource Group:  rg-todoapp-dev
  PostgreSQL:      psql-todoapp-dev-xyz123.postgres.database.azure.com
  Web App:         app-todoapp-dev-xyz123
  URL:             https://app-todoapp-dev-xyz123.azurewebsites.net

[6/7] Luodut Azure-resurssit:
  Name                         Type                                      Location
  ---------------------------  ----------------------------------------  -----------
  asp-todoapp-dev              Microsoft.Web/serverfarms                  northeurope
  app-todoapp-dev-xyz123       Microsoft.Web/sites                       northeurope
  psql-todoapp-dev-xyz123      Microsoft.DBforPostgreSQL/flexibleServers  northeurope

Julkaistaan .NET-sovellus App Serviceen...
  Käännetään sovellus (dotnet publish)...
  Paketoidaan ZIP-tiedostoksi...
  Lähetetään App Serviceen (zip deploy)...
  Sovellus julkaistu!

========================================
  Deployment valmis!
========================================

  Sovelluksen URL:  https://app-todoapp-dev-xyz123.azurewebsites.net
  Swagger UI:      https://app-todoapp-dev-xyz123.azurewebsites.net/swagger
  Health check:    https://app-todoapp-dev-xyz123.azurewebsites.net/health

  Ympäristömuuttujat (App Settings) asetettu Bicepillä:
    ConnectionStrings__DefaultConnection = Host=psql-...;...
    ASPNETCORE_ENVIRONMENT = Development
```

### ✅ Tarkistuslista ennen jatkamista

- [ ] `deploy.ps1` luotu ja toimii
- [ ] `cleanup.ps1` luotu
- [ ] Skripti kysyy subscription-valinnan
- [ ] What-if -esikatselu näytetään ennen deploymenttia
- [ ] Sovellus julkaistaan natiivina (`dotnet publish` + zip deploy)
- [ ] Connection string asetetaan Bicepillä ympäristömuuttujaksi
- [ ] Sovelluksen URL näytetään lopussa

---

## Vaihe 7: Testaus ja siivous (~15 min)

### 7.1 Testaa sovellus Azuressa

Avaa sovelluksen URL selaimessa (näkyy deploy-skriptin tulosteessa):

1. **Health check:** `https://<app-nimi>.azurewebsites.net/health`
   - Pitäisi palauttaa: `{"status":"healthy","timestamp":"..."}`

2. **Swagger UI:** `https://<app-nimi>.azurewebsites.net/swagger`
   - Testaa CRUD-operaatiot

3. **POST** -- Luo uusi todo:
   ```json
   {
     "title": "Ensimmäinen Azure-todo!",
     "description": "Tämä on tallennettu Azure PostgreSQL:ään"
   }
   ```

4. **GET** -- Varmista, että todo tallentui tietokantaan

### 7.2 Tarkista ympäristömuuttujat Azure Portalissa

1. Avaa [Azure Portal](https://portal.azure.com)
2. Navigoi: **Resource Groups** → `rg-todoapp-dev` → Web App
3. Vasemmasta valikosta: **Configuration** → **Application settings**
4. Varmista, että näet:
   - `ConnectionStrings__DefaultConnection` = `Host=psql-...;Port=5432;Database=tododb;...`
   - `ASPNETCORE_ENVIRONMENT` = `Development`

> Nämä asetukset tulivat Bicep-templatesta! App Service näyttää ne ympäristömuuttujina sovellukselle.

### 7.3 Vianmääritys

Jos sovellus ei vastaa heti, odota 1-2 minuuttia (ensimmäinen käynnistys on hitaampi).

```powershell
# Tarkista App Servicen lokit
az webapp log tail --resource-group rg-todoapp-dev --name <webapp-nimi>

# Tarkista sovelluksen tila
az webapp show --resource-group rg-todoapp-dev --name <webapp-nimi> --query state

# Tarkista, onko ympäristömuuttujat asetettu oikein
az webapp config appsettings list --resource-group rg-todoapp-dev --name <webapp-nimi> --output table
```

### 7.4 Siivoa resurssit (tärkeää!)

> **Azure-resurssit maksavat!** Muista aina poistaa resurssit, kun et enää tarvitse niitä.

```powershell
# Käytä cleanup-skriptiä
./cleanup.ps1

# Tai manuaalisesti:
az group delete --name rg-todoapp-dev --yes
```

### ✅ Lopputarkistus

- [ ] Sovellus toimii Azuressa (health check OK)
- [ ] CRUD-operaatiot toimivat PostgreSQL:n kanssa
- [ ] Swagger UI avautuu
- [ ] Ympäristömuuttujat näkyvät Azure Portalissa
- [ ] Resurssit siivottu pois (jos et enää tarvitse)

---

## Lopputulos: Tiedostorakenne

Tehtävän jälkeen projektisi näyttää tältä:

```
StarterCode/
├── TodoApi/                        ← Valmis sovellus (annettu)
│   ├── TodoApi.csproj
│   ├── Program.cs
│   ├── Models/Todo.cs
│   ├── Data/AppDbContext.cs
│   └── appsettings.json
├── Dockerfile                      ← Lokaali Docker-kehitys (annettu)
├── docker-compose.yml              ← Lokaali kehitys (annettu)
├── .env                            ← Lokaali salasana (annettu)
├── .dockerignore                   ← (annettu)
│
├── infra/                          ← ★ SINUN TEKEMÄSI ★
│   ├── main.bicep                  ← Päätemplate
│   ├── main.bicepparam             ← Parametrit
│   └── modules/
│       ├── appservice.bicep        ← App Service + ympäristömuuttujat
│       └── postgresql.bicep        ← PostgreSQL
│
├── deploy.ps1                      ← ★ SINUN TEKEMÄSI ★
└── cleanup.ps1                     ← ★ SINUN TEKEMÄSI ★
```

---

## Reflektio: Mitä opimme?

### Lokaali kehitys vs. Azure-julkaisu

| Konsepti | Docker Compose (lokaali) | Azure (pilvi) |
|---|---|---|
| **Sovelluksen ajo** | Docker-kontti | App Service (.NET 8 natiivi) |
| **Tietokanta** | PostgreSQL-kontti | PostgreSQL Flexible Server |
| **Connection string** | `docker-compose.yml` environment | App Service App Settings (Bicep) |
| **Infrastruktuurin luonti** | `docker compose up` | `az deployment sub create` (Bicep) |
| **Infrastruktuurin poisto** | `docker compose down -v` | `az group delete` |

> **Avainopetus:** Sovelluksen koodi on **täsmälleen sama** molemmissa. Ainoa ero on se, **miten** ympäristömuuttujat (erityisesti connection string) asetetaan.

### IaC:n hyödyt käytännössä

| Toiminto | Ilman IaC:tä | IaC:n kanssa |
|---|---|---|
| **Uusi ympäristö (staging)** | 30-60 min käsin klikkausta | `param environment = 'staging'` → deploy |
| **Ympäristön poisto** | Muista poistaa kaikki resurssit | `az group delete` |
| **Muutoshistoria** | "Kuka muutti mitä?" | `git log infra/` |
| **Katselmointi** | Ei mahdollista | Pull request Bicep-muutoksille |
| **Dokumentaatio** | Erillinen dokumentti (vanhenee) | Koodi **on** dokumentaatio |

### Opitut taidot

- **Bicep** -- Azuren IaC-kieli, resurssien deklaratiivinen kuvaaminen
- **Moduulit** -- Koodin uudelleenkäyttö ja organisointi
- **Parametrit ja muuttujat** -- Konfiguraation eriyttäminen koodista
- **Ympäristömuuttujat** -- Connection stringin siirto Docker Composesta App Serviceen
- **What-if** -- Turvallinen deployment-käytäntö
- **Deployment-skripti** -- Automatisoitu julkaisuprosessi
- **App Service** -- .NET-sovelluksen ajaminen natiivisti Azuressa
- **PostgreSQL Flexible Server** -- Hallittu tietokanta Azuressa
- **Zip Deploy** -- Sovelluksen julkaisu App Serviceen

---

## Arviointiperusteet

### Hyväksytty

- ✅ `main.bicep` ja vähintään 1 moduuli luotu (App Service tai PostgreSQL)
- ✅ Infrastruktuuri deploydaan onnistuneesti Azureen
- ✅ Sovellus käynnistyy Azuressa

### Hyvä

- ✅ Molemmat moduulit luotu (PostgreSQL + App Service)
- ✅ Connection string asetetaan App Servicen ympäristömuuttujaksi Bicepissä
- ✅ Parametritiedosto käytössä
- ✅ `@secure()` käytössä salasanalle
- ✅ Deployment-skripti (`deploy.ps1`) toteutettu subscription-valinnalla ja what-if-esikatselulla
- ✅ CRUD-operaatiot toimivat PostgreSQL:n kanssa Azuressa
- ✅ Cleanup-skripti (`cleanup.ps1`) toteutettu
- ✅ Teoriakysymykset vastattu

### Kiitettävä

- ✅ Tagit kaikissa resursseissa (`Application`, `Environment`, `ManagedBy`)
- ✅ Moduulien tulosteet ketjutettu oikein (PostgreSQL connection string → App Service)
- ✅ Health check konfiguroitu App Serviceen
- ✅ Deployment-skripti sisältää virheenkäsittelyn ja selkeän käyttökokemuksen
- ✅ Koodi on siistiä, kommentoitua ja hyvin organisoitua
- ✅ Kaikki vaiheet toteutettu huolellisesti

---

## Palauttaminen

Palauta seuraavat:

2. **Deployment-skriptit mennyt läpi** Kuvat skriptien ajosta (`deploy.ps1`, `cleanup.ps1`)
3. **Kuvakaappaus** -- Swagger UI toimii Azuressa (näytä URL-palkki)
4. **Kuvakaappaus** -- Azure Portal: App Servicen Configuration/Application settings -näkymä, jossa näkyy `ConnectionStrings__DefaultConnection`
5. **Teoriakysymykset** -- Vastaa kaikkiin kysymyksiin tiedostossa [questions.md](questions.md)

**Varmista että:**

- ✅ `deploy.ps1` toimii alusta loppuun (subscription-valinta → sovellus käynnissä)
- ✅ `cleanup.ps1` poistaa kaikki resurssit
- ✅ Bicep-koodi on syntaksiltaan oikein (`az bicep build --file infra/main.bicep`)
- ✅ Salasanat eivät ole kovakoodattuna Bicep-tiedostoissa
- ✅ Connection string asetetaan ympäristömuuttujana (ei kovakoodattuna sovellukseen)

> **Muista poistaa Azure-resurssit** kun olet palauttanut tehtävän, ettet kuluta Azure-krediittejä turhaan!

---

## Tuki ja kysymykset

Jos tarvitset apua:

1. **Tarkista IaC-materiaali:** [Infrastructure as Code](https://github.com/xamk-mire/Xamk-wiki/blob/main/Cloud%20technologies/Azure/Infrastructure-as-Code.md)
2. **Tarkista Bicep-materiaali:** [Bicep-syväsukellus](https://github.com/xamk-mire/Xamk-wiki/blob/main/Cloud%20technologies/Azure/Bicep.md)
3. **Bicep Playground:** [Kokeile selaimessa](https://aka.ms/bicepdemo)
4. **Azure-dokumentaatio:** [Bicep Docs](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
5. **Kysy kavereilta tai opettajalta**

### Hyödylliset komennot

```powershell
# Tarkista Bicep-syntaksi
az bicep build --file infra/main.bicep

# Esikatselu (what-if)
az deployment sub what-if --location northeurope --template-file infra/main.bicep --parameters infra/main.bicepparam

# Listaa resurssit
az resource list --resource-group rg-todoapp-dev --output table

# Tarkista App Servicen ympäristömuuttujat
az webapp config appsettings list --resource-group rg-todoapp-dev --name <webapp-nimi> --output table

# App Servicen lokit
az webapp log tail --resource-group rg-todoapp-dev --name <webapp-nimi>

# Poista kaikki
az group delete --name rg-todoapp-dev --yes
```
