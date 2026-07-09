# dynamic-tariffs-nl

**Open dataset met de actuele tarieven van Nederlandse leveranciers met een
dynamisch stroomcontract** — Tibber, Frank Energie, Zonneplan, Coolblue
Energie en anderen. Machine-leesbaar (JSON), vrij te hergebruiken
(CC-BY-4.0), onderhouden door [SlimHuys.nl](https://slimhuys.nl).

Bij een dynamisch contract betaal je de beursprijs (EPEX day-ahead) plus
een leveranciersopslag, energiebelasting en btw. Welke opslag, in welke
volgorde de belasting en btw worden toegepast, en hoe teruglevering wordt
verrekend verschilt per leverancier — en precies dát legt deze dataset
per leverancier exact vast.

> **Status: alleen-lezen mirror.** De canonieke bron is de
> `suppliers`-tabel in [SlimHuys/slimhuys.nl](https://github.com/SlimHuys/slimhuys.nl);
> deze repo wordt automatisch dagelijks bijgewerkt. Wijzigingen gaan dus
> niet via pull-requests op deze repo maar via issues — zie
> [Meehelpen](#meehelpen).

## Wat zit erin?

Per leverancier één JSON-bestand in [`suppliers/`](suppliers/), met:

- **Ruwe parameters** — opslag per kWh, vaste maandkosten,
  factuurresolutie (kwartier of uur), gastarief
- **Een declaratieve pricing-spec** — de exacte berekeningsstappen van
  beursprijs naar all-in-tarief, machine-uitvoerbaar
- **Leesbare formuletekst** — automatisch gegenereerd uit de spec, zodat
  je zonder JSON-kennis kunt controleren of de berekening klopt
- **Bronvermeldingen** en de datum waarop een mens de tarieven voor het
  laatst tegen de leveranciersbron heeft geverifieerd

Voorbeeld ([`suppliers/tibber.json`](suppliers/tibber.json)):

```json
{
  "$schema": "https://slimhuys.nl/schemas/dynamic-tariffs-nl/v2.json",
  "generated_at": "2026-05-04T15:00:00+02:00",
  "source_url": "https://slimhuys.nl/v1/suppliers",
  "license": "CC-BY-4.0",
  "maintainer": "SlimHuys (https://slimhuys.nl)",
  "supplier": {
    "slug": "tibber",
    "name": "Tibber",
    "active": true,
    "official_api": true,
    "verified_at": "2026-04-28",
    "sources": [
      "https://github.com/JaccoR/hass-entso-e/discussions/239"
    ],
    "params": {
      "billing_resolution_minutes": 15,
      "markup_eur_per_kwh": 0.0207,
      "monthly_fee_eur": 4.95
    },
    "default_gas_rate_eur_m3": 1.050,
    "pricing_spec": {
      "version": 1,
      "consumed": {
        "epex": "quarter",
        "pipeline": [
          { "op": "add", "ref": "tax.energy_tax_electricity", "label": "Energiebelasting" },
          { "op": "multiply", "ref": "tax.vat_rate", "factor": "1+x", "label": "21% btw" },
          { "op": "add", "value": 0.0207, "label": "Tibber-opslag" }
        ]
      },
      "delivered": {
        "strategy": "epex_passthrough",
        "params": {}
      }
    },
    "formula": "..."
  }
}
```

## Hoe lees ik een pricing-spec?

De `pipeline` is een reeks bewerkingen die **in volgorde** op de
EPEX-beursprijs worden toegepast:

- `op: "add"` — telt een bedrag op (opslag of energiebelasting)
- `op: "multiply"` met `factor: "1+x"` — vermenigvuldigt met 1 + waarde
  (zo werkt btw: waarde 0,21 → × 1,21)

Een verwijzing zoals `"ref": "tax.energy_tax_electricity"` wordt pas bij
de berekening ingevuld met het dan geldende belastingtarief. Daardoor
blijft een spec geldig over belastingjaren heen: als de energiebelasting
per 1 januari verandert, hoeft er niets aan de leveranciersbestanden te
gebeuren.

De volgorde van de stappen doet ertoe. Bij Tibber gaat de btw over de
beursprijs plus energiebelasting, en komt de opslag daar (al inclusief
btw) bovenop. Andere leveranciers rekenen hun opslag vóór de btw — dan
staat de add-stap eerder in de pipeline:

**Tibber — EPEX €0,10/kWh, energiebelasting €0,0916, btw 21%:**

| Stap | Bewerking | Waarde |
|---|---|---|
| Start | EPEX-kwartierprijs | €0,1000 |
| 1 | + 0,0916 energiebelasting | €0,1916 |
| 2 | × 1,21 btw | €0,2318 |
| 3 | + 0,0207 Tibber-opslag | €0,2525 |

**Leverancier met opslag vóór btw — zelfde uitgangspunten:**

| Stap | Bewerking | Waarde |
|---|---|---|
| Start | EPEX-uurprijs | €0,1000 |
| 1 | + 0,0149 opslag | €0,1149 |
| 2 | + 0,0916 energiebelasting | €0,2065 |
| 3 | × 1,21 btw | €0,2499 |

Twee leveranciers met een "vergelijkbare" opslag komen zo toch op een
ander all-in-tarief uit — dat verschil is precies waarom deze dataset
de volledige pipeline vastlegt in plaats van alleen een opslag-getal.

## Teruglevering (`delivered.strategy`)

| Strategy | Beschrijving |
|---|---|
| `epex_passthrough` | `max(0, EPEX-kwartierprijs) + optionele opslag`. De meeste leveranciers. |
| `zonneplan_smoothed_bonus` | Gewogen gemiddelde van EPEX over zonuren met een prijs boven nul, daarna `× (1 + bonus_pct) + opslag`, met een jaarcap op de vergoede export. |

Nieuwe strategieën (bijvoorbeeld terugleverkosten-staffels) komen erbij
zodra een leverancier ze introduceert.

## Veldenoverzicht

| Veld | Beschrijving |
|---|---|
| `slug` | Stabiele identifier — gebruik deze voor lookups, niet de naam. |
| `active` | `false` = leverancier neemt geen nieuwe klanten aan of is gestopt. |
| `official_api` | `true` als de leverancier een officiële consumenten-API heeft. |
| `verified_at` | Laatste datum waarop een mens de tarieven tegen de leveranciersbron heeft gecontroleerd. |
| `sources` | Bron-URL's: officiële tarievenpagina, of community-verificatie zoals een Home Assistant-discussion. |
| `params.billing_resolution_minutes` | Factuurresolutie: `15` (kwartierprijzen) of `60` (uurprijzen). |
| `params.markup_eur_per_kwh` | Leveranciersopslag exclusief btw. Informatief — de canonieke waarde staat in de `pricing_spec`. |
| `params.monthly_fee_eur` | Vaste maandkosten van het contract. |
| `default_gas_rate_eur_m3` | All-in gastarief in €/m³ (inclusief opslag, energiebelasting en btw). `null` als de leverancier geen gas levert. |
| `pricing_spec.consumed.pipeline` | Berekeningsstappen voor het afnametarief — zie hierboven. |
| `pricing_spec.delivered.strategy` | Welke terugleverformule geldt. |
| `formula` | Automatisch gegenereerde, leesbare weergave van de `pricing_spec`. |

## Meehelpen

Deze dataset is zo goed als z'n laatste verificatie. Leveranciers passen
hun opslag en voorwaarden geregeld aan, en niet elke wijziging halen we
zelf meteen op. Hulp uit de community is daarom zeer welkom — en een
kleine moeite:

### Een fout of verouderd tarief melden

[Open een issue](../../issues/new) met:

1. **Welke leverancier** (de `slug` uit het JSON-bestand)
2. **Welk veld** er niet klopt (bijvoorbeeld `markup_eur_per_kwh` of
   `monthly_fee_eur`)
3. **Wat de juiste waarde is**
4. **Een controleerbare bron** — een link naar de officiële
   tarievenpagina van de leverancier, of een screenshot van je eigen
   (jaar)afrekening waarop het tarief zichtbaar is

Die bron is het belangrijkste onderdeel: zonder bron kunnen we de
wijziging niet verifiëren en blijft het issue liggen.

### Een leverancier aandragen

Mis je een Nederlandse leverancier met een dynamisch contract? Open een
issue met de naam, de link naar de tarievenpagina en — als je die weet —
de opslag per kWh en de vaste maandkosten. Vermeld ook of de opslag
inclusief of exclusief btw op de site staat; dat bepaalt waar de
add-stap in de pipeline terechtkomt.

### Een tarief bevestigen

Ook "klopt nog steeds" is waardevol. Is de `verified_at` van jouw
leverancier ouder dan een paar maanden en kloppen de waarden nog met je
afrekening? Laat het weten in een issue — dan werken we de
verificatiedatum bij en weet iedereen dat de data vers is.

### Waarom geen pull-requests?

Deze repo is een gegenereerde mirror: de bestanden in `suppliers/`
worden automatisch overschreven vanuit de canonieke database van
SlimHuys. Een direct gemergde PR zou bij de eerstvolgende sync weer
verdwijnen. Meld wijzigingen daarom als issue; wij verwerken ze in de
bron, waarna ze bij de volgende dagelijkse sync hier verschijnen — mét
bijgewerkte `verified_at`. Je wordt in het issue op de hoogte gehouden.

## Hergebruik

[CC-BY-4.0](LICENSE): vrij te gebruiken, ook commercieel, zolang je
[SlimHuys.nl](https://slimhuys.nl) als bron vermeldt. Bijvoorbeeld voor:

- Vergelijkingssites en energie-apps
- Home Assistant-scripts en -dashboards (zie ook
  [slimhuys-homeassistant](https://github.com/SlimHuys/slimhuys-homeassistant))
- Onderzoek en artikelen over de Nederlandse energiemarkt

Liever live data dan een Git-checkout? `GET https://slimhuys.nl/v1/suppliers`
geeft dezelfde tarieven inclusief actuele EPEX-prijzen en
belastingtarieven — gratis, zonder authenticatie.

## Zie ook op SlimHuys

Liever de tarieven in een grafiek dan als JSON? Deze pagina's tonen
dezelfde data, doorgerekend met energiebelasting en btw:

- [Dynamische stroomprijs vandaag](https://slimhuys.nl/stroomprijzen-vandaag) — live EPEX-prijs per kwartier
- [Stroomprijs morgen](https://slimhuys.nl/stroomprijzen-morgen) — de day-ahead-tarieven zodra ENTSO-E ze rond 13:00 publiceert
- [Gemiddelde stroomprijs per maand en jaar](https://slimhuys.nl/gemiddelde-stroomprijs) — historische EPEX-gemiddelden
- [Negatieve stroomprijzen in Nederland](https://slimhuys.nl/negatieve-stroomprijzen) — alle uren met een prijs onder nul
- [Dynamische energieleveranciers vergelijken](https://slimhuys.nl/vergelijken) — jaarkosten van Tibber, Frank Energie, Zonneplan en Coolblue Energie naast elkaar
- [Wanneer is stroom het goedkoopst om de auto te laden?](https://slimhuys.nl/wanneer-laden)

Per leverancier is er een actuele tarief- en historiepagina, bijvoorbeeld
[Tibber](https://slimhuys.nl/leverancier/tibber),
[Frank Energie](https://slimhuys.nl/leverancier/frank-energie) en
[Zonneplan](https://slimhuys.nl/leverancier/zonneplan).

## Hoe deze repo wordt bijgewerkt

De bestanden in `suppliers/` worden gegenereerd vanuit de
slimhuys.nl-repo en dagelijks om 04:00 gesynchroniseerd:

```bash
php artisan tariffs:sync             # clone + export + commit + push
php artisan tariffs:sync --dry-run   # toont de diff zonder te pushen
```

Na een tariefwijziging in de bron staat de update hier dus uiterlijk de
volgende ochtend.

## Licentie

[CC-BY-4.0](LICENSE) — Creative Commons Attribution 4.0 International.
