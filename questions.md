# Teoriakysymykset -- Azure IaC ja Bicep

Vastaa seuraaviin kysymyksiin omin sanoin. Vastausten ei tarvitse olla pitkiä -- muutama lause riittää, kunhan osoitat ymmärryksesi.

---

## Osa 1: Infrastructure as Code

### Kysymys 1: IaC:n perusidea

Selitä omin sanoin, mitä Infrastructure as Code tarkoittaa ja miksi sitä käytetään. Anna esimerkki tilanteesta, jossa IaC on hyödyllisempi kuin resurssien luominen käsin Azure Portalissa.

```
Infrastructure as Code tarkoittaa, että infrastruktuuri määritellään ja luodaan koodin avulla manuaalisen tekemisen sijaan. Sitä käytetään, koska ympäristöt ovat tällöin toistettavia, nopeita luoda ja helpommin hallittavia.
Esimerkiksi jos pitää luoda sama ympäristö useaan kertaan (dev ja prod), IaC on parempi kuin Azure Portal, koska sama koodi voidaan ajaa uudelleen ja saadaan identtinen lopputulos.
```

### Kysymys 2: Deklaratiivinen vs. imperatiivinen

Selitä ero deklaratiivisen ja imperatiivisen IaC-lähestymistavan välillä. Kumpaan kategoriaan Bicep kuuluu?

```
Deklaratiivisessa mallissa kerrotaan, millainen lopputila halutaan, kun taas imperatiivisessa kerrotaan vaihe vaiheelta mitä tehdään.
Bicep kuuluu deklaratiiviseen malliin, koska siinä kuvataan haluttu infrastruktuuri eikä yksittäisiä komentoja.
```

### Kysymys 3: Idempotenssi

Mitä tarkoittaa, kun sanotaan, että IaC on **idempotenttia**? Miksi tämä ominaisuus on hyödyllinen?

```
Idempotenssi tarkoittaa, että sama koodi voidaan ajaa useita kertoja ja lopputulos pysyy samana.
Tämä on hyödyllistä, koska ympäristö voidaan turvallisesti päivittää tai luoda uudelleen ilman että syntyy turhia muutoksia.
```

### Kysymys 4: Konfiguraation ajautuminen (drift)

Selitä, mitä "configuration drift" tarkoittaa. Miten IaC auttaa estämään sitä?

```
Configuration drift tarkoittaa, että ympäristö muuttuu ajan myötä eri tilaan kuin mitä oli alun perin määritelty.
IaC estää tätä, koska koodi toimii “totuutena” ja ympäristö voidaan aina palauttaa siihen ajamalla määrittely uudelleen.
```

---

## Osa 2: Bicep

### Kysymys 5: Bicep vs. ARM

Miksi Bicep kehitettiin ARM JSON -templatejen tilalle? Mainitse vähintään 2 etua.

```
Bicep kehitettiin, koska ARM JSON on vaikealukuista. Bicep on selkeämpi ja lyhyempi kirjoittaa, ja siinä on parempi IntelliSense ja moduulituki
```

### Kysymys 6: Parametrit ja `@secure()`

Miksi tietokantasalasana merkitään `@secure()`-dekoraattorilla Bicepissä? Mitä tapahtuisi ilman sitä?

```
@secure()-dekoraattori piilottaa salasanan niin, ettei se näy lokeissa tai deployment-historiassa.
Ilman sitä salasana voisi näkyä selväkielisenä, mikä on tietoturvariski.
```

### Kysymys 7: Moduulit

Miksi infrastruktuurikoodi jaettiin tässä tehtävässä kolmeen erilliseen moduuliin (`acr.bicep`, `postgresql.bicep`, `appservice.bicep`) yhden ison tiedoston sijaan? Mainitse vähintään 2 syytä.

```
Koodi jaetaan moduuleihin, jotta se on helpompi lukea ja ylläpitää, sekä jotta samoja osia voi käyttää uudelleen.
Lisäksi virheiden paikantaminen on helpompaa kuin yhdessä isossa tiedostossa.
```

### Kysymys 8: `uniqueString()`

Miksi ACR:n ja PostgreSQL-palvelimen nimissä käytetään `uniqueString(resourceGroup().id)` -funktiota? Mitä tapahtuisi ilman sitä?

```
uniqueString() tekee resurssinimestä uniikin, jotta se ei mene päällekkäin muiden kanssa.
Ilman sitä deployment voisi epäonnistua, jos sama nimi on jo käytössä.
```

### Kysymys 9: `targetScope`

Mitä tarkoittaa `targetScope = 'subscription'` main.bicep-tiedostossa? Miksi emme käytä oletusarvoa `resourceGroup`?


```
targetScope = 'subscription' tarkoittaa, että deployment tehdään subscription-tasolla eikä yksittäiseen resource groupiin.
Tätä tarvitaan esimerkiksi silloin, kun luodaan resource group itse, eikä sitä ole vielä valmiina.
```

---

## Osa 3: Azure-resurssit

### Kysymys 10: Resource Group

Mikä on Azure Resource Groupin tarkoitus? Miksi kaikki sovelluksen resurssit kannattaa sijoittaa samaan resource groupiin?


```
Resource Group on looginen “kansio”, johon saman sovelluksen resurssit kootaan.
Kun kaikki ovat samassa ryhmässä, niitä on helpompi hallita, päivittää ja tarvittaessa poistaa yhdellä kertaa.
```

### Kysymys 11: Ympäristömuuttujat ja Connection String

Selitä, miten sovelluksen tietokantayhteys konfiguroidaan eri ympäristöissä:
- Miten connection string asetetaan **Docker Composessa** (lokaalissa kehityksessä)?
- Miten **sama** connection string asetetaan **Azure App Servicessä**?
- Miksi sovelluksen koodi ei muutu, vaikka ympäristö vaihtuu?

```
- Docker Composessa connection string asetetaan yleensä environment-muuttujana .env-tiedostossa tai docker-compose.yml:ssä.
- Azure App Servicessä se asetetaan Application Settings -kohtaan ympäristömuuttujaksi.
- Sovelluksen koodi ei muutu, koska se lukee arvon ympäristömuuttujasta, ei kovakoodatusta arvosta.
```


### Kysymys 13: PostgreSQL Flexible Server -- Firewall

Miksi PostgreSQL-palvelimeen luodaan firewall-sääntö `AllowAzureServices` (IP-alue `0.0.0.0 - 0.0.0.0`)? Mitä tapahtuisi ilman sitä?


```
AllowAzureServices-sääntö sallii muiden Azure-palveluiden yhteyden tietokantaan.
Ilman sitä sovellus ei pystyisi yhdistämään tietokantaan, koska yhteydet estettäisiin palomuurissa.
```

---

## Osa 4: Deployment ja turvallisuus

### Kysymys 15: What-if

Miksi `what-if` on tärkeä vaihe ennen deploymenttia? Anna esimerkki tilanteesta, jossa what-if estäisi ongelman.


```
What-if näyttää etukäteen mitä muutoksia deployment tekee ilman että mitään oikeasti tapahtuu.
Esimerkiksi se voi estää tilanteen, jossa tärkeä resurssi poistettaisiin vahingossa.
```

### Kysymys 16: Tagit

Miksi kaikkiin Azure-resursseihin lisättiin tagit (`Application`, `Environment`, `ManagedBy`)? Miten ne hyödyttävät käytännössä?


```
Tagit auttavat tunnistamaan ja hallitsemaan resursseja (esim. mihin sovellukseen ja ympäristöön ne kuuluvat).
Niitä voidaan käyttää esim. kustannusten seurantaan ja resurssien suodattamiseen.
```

### Kysymys 17: Siivous ja kustannukset

Miksi on tärkeää poistaa kehitysresurssit Azuresta kun niitä ei enää tarvita? Mikä on helpoin tapa poistaa kaikki tämän tehtävän resurssit kerralla?


```
Resurssit maksavat rahaa niin kauan kuin ne ovat olemassa, joten turhat kannattaa poistaa.
Helpoin tapa on poistaa koko resource group, jolloin kaikki siihen kuuluvat resurssit poistuvat kerralla.
```

---

