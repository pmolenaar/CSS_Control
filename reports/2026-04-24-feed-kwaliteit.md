# Onderzoeksrapport: CSS Feed-kwaliteit per 24 april 2026

**Datum:** 2026-04-24
**Onderzocht door:** Paul Molenaar / Claude (Reshift)
**Status:** Feed-kwaliteit is op technische hygiëne verbeterd, maar kernfouten uit BE-suspensierapport (22 apr) zijn **nog steeds aanwezig**
**Accounts:** Kieskeurig.nl CSS (Ads ID `3380189117`), Kieskeurig.be CSS (Ads ID `8823617537`)

Dit rapport vergelijkt de feed-staat van 22 april (zie [kieskeurig-be-suspensie.md](2026-04-22-kieskeurig-be-suspensie.md)) met 24 april, met focus op **kwaliteit van de indienig**, niet account-status. De BE-suspensie is bekend en niet opgeheven — dat is geen onderwerp van dit rapport.

---

## 1. Methode

Feed opgehaald via de Google Ads API (`shopping_product` resource) door `~/kk_feedcheck/fetch_css_feed.py`, uitgebreid met `custom_attribute0..4`:

```sql
SELECT
    shopping_product.item_id,
    shopping_product.title,
    shopping_product.brand,
    shopping_product.price_micros,
    shopping_product.language_code,
    shopping_product.status,
    shopping_product.custom_attribute0,  -- ProductCode
    shopping_product.custom_attribute1,  -- merchant naam (lowestPrice.Shop.Name)
    shopping_product.custom_attribute2,  -- PredictedCommission
    shopping_product.custom_attribute3,  -- aantal offers
    shopping_product.custom_attribute4   -- label (longtail / p001 / google_demand / e000)
FROM shopping_product
```

Snapshots:
- `~/kk_feedcheck/snapshots/NL_quality_20260424.json` — 82.865 producten
- `~/kk_feedcheck/snapshots/BE_quality_20260424.json` — 932 producten

Vergeleken met 22 april snapshots (zelfde locatie).

---

## 2. Volumeverschuiving 22 → 24 april

| Merk | 22 apr | 24 apr | Δ |
|------|--------|--------|---|
| NL | 350.087 | 82.865 | **-76,3%** |
| BE | 133.574 | 932 | **-99,3%** |

De dalingen komen **niet** voort uit een feed-kwaliteitsslag aan Reshift-zijde, maar uit automatische expiry/cleanup door Google (suspended items worden verwijderd, `GoogleCssExpiryJob` zorgt voor verlooptijd). Dit maakt analyse van de **resterende** producten relevant — dat wat Reshift momenteel actief indient.

---

## 3. Kritieke bevindingen

### 3.1 Bol.com-deduplicatie is niet gedeployed 🔴

De hoofdoorzaak van de BE-suspensie — bol.com (shop X) + bol.plaza (shop Y) worden als 2 aparte merchants geteld terwijl Google ze als 1 juridische entiteit (Ahold-Delhaize) behandelt — is **niet opgelost in de productie-feed**.

| Merchant-naam in `custom_attribute1` | BE count | NL count |
|--------------------------------------|----------|----------|
| `bol.` | 203 | 38 |
| `bol. plaza` | 213 | 69 |

Beide entiteiten komen tegelijk in veel producten voor, tellen als "2 aanbieders" in de Reshift-filter, maar Google zal ze als 1 aanbieder behandelen. De `BolShopIds`-lijst analoog aan `AmazonShopIds` in `ProductDataProcessor.cs` is nog steeds afwezig in `pc_api`.

### 3.2 BE-feed: 51,6% van producten schendt multi-merchant regel 🔴

CSS-policy vereist **minimum 2 onafhankelijke aanbieders** per product. Verdeling van `custom_attribute3` (offer count) in BE:

| Offers | Count | % |
|--------|-------|---|
| **0** | 59 | 6,3% — producten zonder één aanbieder ingediend |
| **1** | 422 | 45,3% — single-merchant |
| 2 | 159 | 17,1% |
| 3 | 269 | 28,9% |
| 4-6 | 22 | 2,4% |
| `-1` | 1 | 0,1% — negatieve count, bug |

**Totaal: 481 van 932 BE-producten (51,6%) voldoet niet aan de multi-merchant regel.**

Van de 481 single/zero-merchant producten zijn 404 longtail met offers=1 en 59 met offers=0 — dit zijn precies de categorieën die Google bij suspensie aanhaalde.

### 3.3 BE-feed: longtail-dominantie onveranderd 🔴

| Label | 22 apr | 24 apr |
|-------|--------|--------|
| `longtail` | 96,1% | **95,9%** |
| `google_demand` | 2,2% | 4,0% |
| `p001`..`p005` | 0,6% | — |
| `sales` | — | 0,1% |
| andere | ~1,1% | — |

De longtail-tak van `GetProductsAsync()` (Elasticsearch-query op `totalPrices >= 2 voor publicationId`) produceert onverminderd producten die Google als low-intent beoordeelt.

### 3.4 NL-feed: schema-anomalie — custom_attributes zijn niet wat de code belooft 🔴

Volgens `ProductDataProcessor.CreateExtractedProductData()` in `pc_api` moeten de custom_attributes specifieke waarden bevatten. De werkelijke NL-feed wijkt hier significant van af:

| Attribute | Documentatie/v1-code | Werkelijke NL-waarden |
|-----------|---------------------|------------------------|
| `custom_attribute1` | `lowestPrice.Shop.Name` (merchant naam) | **`goed`** (4.293 producten), **`slecht`** (1.370), `cashback` (120), vermengd met echte merchant-namen |
| `custom_attribute3` | `prices.Count.ToString()` (aantal offers) | **`leverbaar`** (8.681), **`op voorraad`** (3.807), `leverdatum` (1.965), `(leeg)` (68.056), vermengd met cijfers |
| `custom_attribute4` | label (`longtail`/`google_demand`/`p00N`/`e0N0`) | **82,1% leeg**, plus nieuwe waarden: `no-index`, `<10`, `>10`, `under-index`, `near-index`, `index` |

**Interpretatie:** Ofwel is er sinds maart 2026 een **tweede extraction-job** die NL parallel vult met een andere schema-definitie, ofwel is de code zonder doc-update gewijzigd. Het vullen van `custom_attribute1` met rating-termen als "goed"/"slecht" en van `custom_attribute3` met voorraad-strings is een afwijking die dringend in de pc_api-code gezocht moet worden.

Gevolg voor feed-beoordeling: de policy-logica van Google kan deze velden niet interpreteren zoals verwacht, wat bijdraagt aan het hoge `NOT_ELIGIBLE`-percentage (zie 3.5).

### 3.5 NL-feed: taal- en prijsproblemen

| Metric | Waarde |
|--------|--------|
| `language_code=en` | 68.054 (**82,1%**) |
| `language_code=nl` | 14.811 (17,9%) |
| Prijs buiten CSS-bereik €25-€3.000 | 22.469 (27,1%) |
| Ontbrekende `custom_attribute2` (commission) | 73.563 (88,8%) |

Een feed voor een Nederlandstalige consumentensite (kieskeurig.nl) met 82% Engelstalige metadata is een rode vlag. CSS-policy vereist een prijsbereik dat hier voor 27% van de producten niet klopt.

---

## 4. Verbeteringen sinds 22 april 🟢

Technische hygiëne is duidelijk beter. Onderstaande issues van Google's policy-crawler zijn sterk gedaald of weg:

| Issue | 22 apr | 24 apr |
|-------|--------|--------|
| `utf8_encoding_error` | 658 | 1 |
| `reserved_gtin` | 182 | 0 |
| `coupon_gtin` | 17 | 0 |
| `invalid_upc` | 9 | 0 |
| `guns_parts_policy_violation` | 33 | 0 |
| `dangerous_knives_policy_violation` | 13 | 0 |
| `subscriptions_policy_violation` | 6 | 0 |
| `tobacco_policy_violation` | 5 | 0 |
| `child_porn_policy_violation` | 1 | 0 |
| `image_too_big` | 24 | 3 |
| `illegal_drugs_policy_violation` | 74 | 42 |
| `healthcare_unapproved_supplements_policy_violation` | 20 | 8 |
| `image_too_small` | 1.815 | 588 |
| `image_decoding_error` | 26 | 2 |

Dit suggereert dat de expiry/cleanup mechanisch heeft opgeruimd wat Google als problematisch markeerde. Het betekent **niet** dat nieuw ingediende producten vrij zijn van dezelfde issues — het betekent dat het aanbod gekrompen is tot wat overbleef na Google's filter.

---

## 5. Overall compliance-verdict

| Aspect | 22 apr | 24 apr | Status |
|--------|--------|--------|--------|
| Technische hygiëne (encoding, GTIN, policy-keywords) | Veel issues | Sterk gedaald | 🟢 |
| Multi-merchant regel (BE) | Hoofdoorzaak suspensie | **51,6% schendt nog** | 🔴 |
| Bol.com-dedup | Ontbreekt | **Ontbreekt** | 🔴 |
| Longtail-aandeel BE | 96,1% | **95,9%** | 🔴 |
| NL taal | Onbekend | 82% `en` | 🔴 |
| NL prijsrange | Onbekend | 27% buiten €25-€3.000 | 🔴 |
| NL schema-integriteit | Onbekend | **Corrupt** — custom_attributes herbestemd | 🔴 |

**Conclusie:** De feed is vandaag **technisch netter** dan 22 april, maar op de **kern-CSS-policy-regels** (multi-merchant, longtail-kwaliteit, merchant-deduplicatie, content-taal) is **geen materiële vooruitgang** geboekt. De daling in volume is een bijproduct van Google's cleanup, niet een gevolg van Reshift's correcties.

Een heraanvraag tot review van het BE-account zou op dit moment waarschijnlijk niet slagen: de gerapporteerde problemen uit de suspensie-brief zijn in de feed-output nog zichtbaar (single-merchant longtail, bol.com-dubbeltelling). Eerst `BolShopIds` implementeren en de feed opnieuw laten doorlopen is een voorwaarde.

---

## 6. Aanbevolen acties (in volgorde van prioriteit)

### 6.1 Hoogste prioriteit — vóór re-review-aanvraag

1. **Implementeer `BolShopIds`-deduplicatie** in `pc_api/src/Reshift.ProductCatalog.Jobs/Application/Extraction/Css/Services/ProductDataProcessor.cs`, analoog aan `AmazonShopIds`. Zie het aparte structurele pad: de `shop_owner_groups`-tabel in `~/css/migrations/001_shop_owner_groups.sql`.
2. **Filter producten met `offers < 2`** uit de `CreateExtractedProductData()`-pad — de effective merchant count na Amazon/Bol-dedup moet ≥2 zijn.
3. **Voeg `account-health check`** toe aan het begin van `GoogleCssExtractionJob.ExecuteAsync()`: als het account gesuspendeerd is, sla over in plaats van door te pushen.

### 6.2 Hoge prioriteit — kwaliteit NL

4. **Onderzoek NL schema-anomalie** (`custom_attribute1=goed|slecht|cashback`, `custom_attribute3` met voorraad-strings). Controleer of een tweede extraction-job actief is of de v1-code gewijzigd. Relevante zoektocht:
   - `grep -r 'goed\|slecht\|cashback' pc_api/src` voor tekstuele bronnen
   - Kijk naar recente commits op `GoogleCssExtractionJob.cs` en `ProductDataProcessor.cs`
5. **Los de `language_code=en` op 82% NL-producten op.** Identificeer waar dit veld wordt toegekend.
6. **Filter producten buiten €25-€3.000** uit — de SQL-pre-filter doet dit al, maar 27% van NL-producten schendt het, dus gaat er een tweede route om de filter heen.

### 6.3 Middellange termijn — architectuur

7. De herbouw naar ClickHouse + diff-based submission zoals in [css-feed-architectuurreview.md](../architecture/css-feed-architectuurreview.md) beschreven voor Gina ter Elst. Bij implementatie van merchant-dedup via een `shop_owner_groups`-tabel wordt fout #1 structureel opgelost.

---

## 7. Bijlagen

- Ruwe snapshots: `~/kk_feedcheck/snapshots/NL_quality_20260424.json` (82.865 producten), `BE_quality_20260424.json` (932 producten)
- Extra query-script voor custom_attributes: `~/kk_feedcheck/fetch_css_feed.py` (uitgebreid versie in `/tmp/fetch_quality.py` sessie 24 apr)
- Eerdere suspensierapport: [2026-04-22-kieskeurig-be-suspensie.md](2026-04-22-kieskeurig-be-suspensie.md)
- Architectuurreview: [../architecture/css-feed-architectuurreview.md](../architecture/css-feed-architectuurreview.md)
- Code-flow schema: [../architecture/css-feed-flow.md](../architecture/css-feed-flow.md)
