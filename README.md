# dynamic-tariffs-nl

Open dataset met de actuele tarieven van Nederlandse leveranciers met
**dynamische stroomcontracten**. Bijgehouden door [SlimHuys.nl](https://slimhuys.nl).

> **Status**: read-only mirror. De canonical source-of-truth is de
> `suppliers`-tabel binnen [SlimHuys/slimhuys.nl](https://github.com/SlimHuys/slimhuys.nl).
> Deze repo wordt periodiek bijgewerkt met `php artisan tariffs:export`.

## Inhoud

`tariffs.json` — één JSON-bestand met alle 13+ actieve leveranciers, hun
inkoopvergoeding, vaste maandkosten, terugleververgoeding-structuur en
metadata.

```json
{
  "$schema": "https://slimhuys.nl/schemas/dynamic-tariffs-nl/v1.json",
  "generated_at": "2026-05-01T12:00:00+02:00",
  "source_url": "https://slimhuys.nl/v1/suppliers",
  "license": "CC-BY-4.0",
  "maintainer": "SlimHuys (https://slimhuys.nl)",
  "suppliers": [
    {
      "slug": "tibber",
      "name": "Tibber",
      "active": true,
      "official_api": true,
      "billing_resolution_minutes": 15,
      "markup_eur_per_kwh": 0.0207,
      "monthly_fee_eur": 4.95,
      "feedin": {
        "markup_eur_per_kwh": 0.0,
        "bonus_pct": 0.0,
        "bonus_daytime_only": false,
        "bonus_annual_cap_kwh": null
      },
      "signup_url": "https://tibber.com/nl",
      "notes": "Verified 2026-04-28 — support.tibber.com (HIGH); officiële GraphQL-API beschikbaar."
    }
  ]
}
```

## Schema-uitleg

| Veld | Beschrijving |
|---|---|
| `slug` | Stabiele identifier — gebruik dit voor lookups. |
| `billing_resolution_minutes` | `15` (kwartier-tarief) of `60` (uur-tarief). Bepaalt hoe vaak de prijs verandert. |
| `markup_eur_per_kwh` | Inkoopvergoeding boven de EPEX-prijs voor afgenomen stroom. |
| `monthly_fee_eur` | Vaste maandkosten (excl. netbeheer/belasting). |
| `feedin.markup_eur_per_kwh` | Vaste opslag op de terugleververgoeding (bv. Zonneplan +€0,02). |
| `feedin.bonus_pct` | Procentuele bonus boven de feedin (bv. Zonneplan 0,10 = 10%). |
| `feedin.bonus_daytime_only` | Zonnebonus alleen tussen op-/ondergang. |
| `feedin.bonus_annual_cap_kwh` | Maximaal aantal kWh per kalenderjaar dat de bonus krijgt (`null` = onbeperkt). |
| `signup_url` | Publieke aanmeldpagina van de leverancier. |
| `notes` | Verificatie-bron + datum + confidence (HIGH/MEDIUM). |

## Hergebruik

CC-BY-4.0 — vrij hergebruik mits SlimHuys.nl als bron vermeld. Geschikt
voor:

- Andere vergelijker-sites
- Home Assistant-scripts (zie ook [slimhuys-homeassistant](https://github.com/SlimHuys/slimhuys-homeassistant))
- Onderzoekers / blogs over de Nederlandse energiemarkt

Voor live tarieven inclusief actuele EPEX + energiebelasting: bel
`https://slimhuys.nl/v1/suppliers` rechtstreeks aan (gratis, ongeauth).

## Issues / correcties

Tarief verkeerd? Open een issue met:

- Welke leverancier (slug)
- Welk veld
- Wat het zou moeten zijn
- **Bron-link** (officiële leverancier-pagina, screenshot van eigen jaarafrekening)

PR's met correcties op `tariffs.json` worden NIET direct gemerged — we
verifiëren bron-link en propageren via de SlimHuys-canonical-DB. Anders
loopt deze mirror uit de pas.

## Build/sync

Deze repo wordt periodiek bijgewerkt door:

```bash
# In de slimhuys.nl-repo:
php artisan tariffs:export --out=/path/to/dynamic-tariffs-nl/tariffs.json
git -C /path/to/dynamic-tariffs-nl commit -am "tariffs: sync $(date +%Y-%m-%d)"
git -C /path/to/dynamic-tariffs-nl push
```

Frequentie: na elke significante seeder-update (typisch maandelijks).

## License

[CC-BY-4.0](LICENSE) — Creative Commons Attribution 4.0 International
