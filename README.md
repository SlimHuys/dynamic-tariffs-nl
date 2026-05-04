# dynamic-tariffs-nl

Open dataset met de actuele tarieven van Nederlandse leveranciers met
**dynamische stroomcontracten**. Onderhouden door [SlimHuys.nl](https://slimhuys.nl).

> **Status**: alleen-lezen mirror. De canonieke bron is de
> `suppliers`-tabel in [SlimHuys/slimhuys.nl](https://github.com/SlimHuys/slimhuys.nl).
> Deze repo wordt periodiek bijgewerkt met `php artisan tariffs:sync`.

## Inhoud

Per leverancier één JSON-bestand in [`suppliers/`](suppliers/). Zie de
[index in `suppliers/README.md`](suppliers/) voor een overzicht met
markup, maandfee en gastarief per leverancier.

Elk bestand bevat:

- Ruwe parameters (markup, maandfee, factuurresolutie, gastarief)
- Een **declaratieve pricing-spec** die de exacte berekening beschrijft
- Auto-gegenereerde formuletekst voor menselijke verificatie
- Bronvermeldingen + datum van laatste verificatie

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

## Pipeline — hoe lees ik dit?

De `pipeline`-array is een **lineaire reeks bewerkingen** die in volgorde
op de EPEX-prijs worden toegepast. Elke stap heeft:

- `op: "add"` — telt iets op (opslag, energiebelasting)
- `op: "multiply", factor: "1+x"` — vermenigvuldigt met (1 + waarde) (btw)

Verwijzingen als `tax.energy_tax_electricity` worden bij berekening
vervangen door het actuele tarief uit de `tax_rates`-tabel — zo blijft
de spec geldig over belastingjaren heen.

**Voorbeeld — Tibber, EPEX €0,10/kWh, energiebelasting €0,0916, btw 21%:**

| Stap | Bewerking | Waarde |
|---|---|---|
| Start | EPEX-kwartierprijs | €0,1000 |
| 1 | + 0,0916 (energiebelasting) | €0,1916 |
| 2 | × 1,21 (btw) | €0,2318 |
| 3 | + 0,0207 (Tibber-opslag) | €0,2525 |

**Voorbeeld — leverancier met opslag vóór btw:**

| Stap | Bewerking | Waarde |
|---|---|---|
| Start | EPEX-uurprijs | €0,1000 |
| 1 | + 0,0149 (opslag) | €0,1149 |
| 2 | + 0,0916 (energiebelasting) | €0,2065 |
| 3 | × 1,21 (btw) | €0,2499 |

## delivered.strategy

| Strategy | Beschrijving |
|---|---|
| `epex_passthrough` | `max(0, EPEX-kwartier) + optionele opslag`. De meeste leveranciers. |
| `zonneplan_smoothed_bonus` | Gewogen gemiddelde van EPEX over zonuren met EPEX > 0, vervolgens `× (1 + bonus_pct) + opslag`. Met jaarcap op export. |

Nieuwe strategieën (bijvoorbeeld voor terugleverkosten-staffels) worden
toegevoegd zodra een leverancier ze introduceert.

## Schema-uitleg

| Veld | Beschrijving |
|---|---|
| `slug` | Stabiele identifier — gebruik dit voor lookups. |
| `verified_at` | Laatste datum waarop een mens de spec heeft bevestigd tegen de leveranciersbron. |
| `sources` | Lijst met bron-URL's (officiële tarievenpagina, HA-template-discussion). |
| `params.billing_resolution_minutes` | `15` (kwartier) of `60` (uur). |
| `params.markup_eur_per_kwh` | Leveranciersopslag exclusief btw — informatief; de canonieke waarde staat in `pricing_spec`. |
| `params.monthly_fee_eur` | Vaste maandkosten van het contract. |
| `default_gas_rate_eur_m3` | All-in gastarief in €/m³ (incl. opslag, energiebelasting en btw). `null` voor leveranciers zonder gascontract. |
| `pricing_spec.consumed.pipeline` | Bewerkingsreeks voor het afnametarief. |
| `pricing_spec.delivered.strategy` | Welke teruglever-formule wordt toegepast. |
| `formula` | Auto-gegenereerde markdown — leesbare versie van `pricing_spec`. |

## Hergebruik

CC-BY-4.0 — vrij hergebruik mits [SlimHuys.nl](https://slimhuys.nl) als
bron wordt vermeld. Geschikt voor:

- Andere vergelijkingssites
- Home Assistant-scripts (zie ook [slimhuys-homeassistant](https://github.com/SlimHuys/slimhuys-homeassistant))
- Onderzoekers en blogs over de Nederlandse energiemarkt

Voor live tarieven inclusief actuele EPEX en energiebelasting: doe een
GET-request naar `https://slimhuys.nl/v1/suppliers` (gratis, zonder
authenticatie).

## Fouten melden

Klopt een tarief niet? Open een issue met:

- Welke leverancier (slug)
- Welk veld
- Wat het zou moeten zijn
- **Bron-URL** (officiële leverancierspagina of screenshot van je eigen
  jaarafrekening)

Pull-requests op `suppliers/<slug>.json` worden niet direct gemerged: we
verifiëren de bron en passen de canonieke seeder aan in de SlimHuys-repo,
zodat deze mirror niet uit de pas gaat lopen met de werkelijkheid.

## Bouwen en synchroniseren

Deze repo wordt periodiek bijgewerkt vanuit de slimhuys.nl-repo:

```bash
php artisan tariffs:sync             # clone + export + commit + push
php artisan tariffs:sync --dry-run   # toont diff zonder iets te pushen
```

Frequentie: na elke significante seeder-update (typisch maandelijks).

## Licentie

[CC-BY-4.0](LICENSE) — Creative Commons Attribution 4.0 International.
