# dynamic-tariffs-nl

Open dataset met de actuele tarieven van Nederlandse leveranciers met
**dynamische stroomcontracten**. Bijgehouden door [SlimHuys.nl](https://slimhuys.nl).

> **Status**: read-only mirror. De canonical source-of-truth is de
> `suppliers`-tabel binnen [SlimHuys/slimhuys.nl](https://github.com/SlimHuys/slimhuys.nl).
> Deze repo wordt periodiek bijgewerkt met `php artisan tariffs:export`.

## Inhoud

`tariffs.json` — één JSON-bestand met alle 13+ actieve leveranciers + per
leverancier een **declaratieve pricing-spec** die de exacte berekening
beschrijft. Inclusief auto-gerendered formule-tekst voor menselijke
verificatie.

```json
{
  "$schema": "https://slimhuys.nl/schemas/dynamic-tariffs-nl/v2.json",
  "generated_at": "2026-05-02T09:00:00+02:00",
  "source_url": "https://slimhuys.nl/v1/suppliers",
  "license": "CC-BY-4.0",
  "maintainer": "SlimHuys (https://slimhuys.nl)",
  "suppliers": [
    {
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
      "pricing_spec": {
        "version": 1,
        "consumed": {
          "epex": "quarter",
          "pipeline": [
            { "op": "add", "value": 0.0207, "label": "Tibber-markup" },
            { "op": "add", "ref": "tax.energy_tax_electricity", "label": "Energiebelasting" },
            { "op": "multiply", "ref": "tax.vat_rate", "factor": "1+x", "label": "21% btw" }
          ]
        },
        "delivered": {
          "strategy": "epex_passthrough",
          "params": {}
        }
      },
      "formula": "**Afnemen (per kWh):**\n  EPEX-kwartierprijs\n  + €0,0207 — Tibber-markup\n  + Energiebelasting (tax_rates) — Energiebelasting\n  × (1 + 21% btw (tax_rates)) — 21% btw\n\n**Teruglevering (per kWh):**\n  max(0, EPEX-kwartier)"
    }
  ]
}
```

## Pipeline — hoe lees ik dit?

De `pipeline`-array is een **lineaire stack van bewerkingen** die in
volgorde op de EPEX-prijs worden toegepast. Elke stap heeft:

- `op: "add"` — telt iets op (markup, energiebelasting)
- `op: "multiply", factor: "1+x"` — vermenigvuldigt met (1 + waarde) (btw)

Refs als `tax.energy_tax_electricity` worden bij berekening vervangen door
het actuele tarief uit de `tax_rates`-tabel — zo staat de spec niet vast
op een belastingjaar.

**Voorbeeld — Tibber, EPEX = €0,10/kWh, energiebelasting €0,0916, btw 21%:**

| Stap | Operatie | Waarde |
|---|---|---|
| Start | EPEX-kwartierprijs | €0,1000 |
| 1 | + 0,0207 (markup) | €0,1207 |
| 2 | + 0,0916 (energiebelasting) | €0,2123 |
| 3 | × 1,21 (btw) | €0,2569 |

**Voorbeeld — leverancier met handling-fee NA btw:**

| Stap | Operatie | Waarde |
|---|---|---|
| Start | EPEX-uurprijs | €0,1738 |
| 1 | + 0,0916 (energiebelasting) | €0,2654 |
| 2 | × 1,21 (btw) | €0,3211 |
| 3 | + 0,02 (handling-fee post-btw) | €0,3411 |

## delivered.strategy

| Strategy | Beschrijving |
|---|---|
| `epex_passthrough` | `max(0, EPEX-kwartier) + optionele markup`. De meeste leveranciers. |
| `zonneplan_smoothed_bonus` | Gewogen-gemiddelde EPEX over zonuren met EPEX>0, daarna `× (1 + bonus_pct) + markup`. Cap op jaar-export. |

Nieuwe strategies (bv. terugleverkosten-staffel) worden toegevoegd zodra
een leverancier ze invoert.

## Schema-uitleg

| Veld | Beschrijving |
|---|---|
| `slug` | Stabiele identifier — gebruik dit voor lookups. |
| `verified_at` | Laatste datum waarop een mens de spec heeft bevestigd tegen leverancier-bron. |
| `sources` | Array bron-URL's (officiële tarieven-pagina, HA-template-discussion). |
| `params.billing_resolution_minutes` | `15` (kwartier) of `60` (uur). |
| `params.markup_eur_per_kwh` | Inkoopvergoeding ex-btw — info; canonical waarde staat in `pricing_spec`. |
| `pricing_spec.consumed.pipeline` | Bewerkings-stack voor afnemen-tarief. |
| `pricing_spec.delivered.strategy` | Welke teruglever-formule wordt toegepast. |
| `formula` | Auto-gerendered markdown — leesbare versie van `pricing_spec`. |

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
