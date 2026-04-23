# CSS Feed Flow — Code Architecture

Schematisch overzicht van het proces dat de Google CSS feed genereert en indient bij Google Merchant Center.

---

## Overzicht

```
┌─────────────────────────────────────────────────────────────────────┐
│  Kubernetes (Scaleway) — namespace: production                       │
│                                                                     │
│  CronJob: job-google-css-extraction                                 │
│  Schedule: 0 * * * * (elk uur)                                      │
│  Image: ghcr.io/reshift/job-runner:latest                           │
│  Config: compare-infra/.../job-runner/configmap.yaml                │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  GoogleCssExtractionJob.ExecuteAsync()                              │
│  pc_api/.../Extraction/Css/GoogleCssExtractionJob.cs                │
│                                                                     │
│  Itereert over geconfigureerde accounts:                            │
│    • Kieskeurig.nl  (AccountId: 140803054, PublicationId: 1)        │
│    • Kieskeurig.be  (AccountId: 5352009739, PublicationId: 2)       │
│    • Review.nl      (AccountId: 5499041869, PublicationId: 13)      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │  GetProductsAsync(feedLabel,   │
              │  publicationId)               │
              └───────────────────────────────┘
                    │               │
          ┌─────────┘               └──────────┐
          ▼                                    ▼
┌──────────────────────┐          ┌────────────────────────────┐
│  BigQuery            │          │  Elasticsearch             │
│                      │          │  index: search-product-    │
│  Table:              │          │  catalog                   │
│  reshift_insights.   │          │                            │
│  daily_demand_labels │          │  Filter:                   │
│                      │          │  lowestPublicationPrice.   │
│  Levert: variant_    │          │  publicationId = X         │
│  gtin + label        │          │  totalPrices >= 2          │
│  (google_demand,     │          │                            │
│   p001, p002, e000)  │          │  Levert: EANs van longtail │
│                      │          │  producten (label=longtail)│
└──────────────────────┘          └────────────────────────────┘
          │                                    │
          └──────────────┬─────────────────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │  products dict:     │
              │  { EAN → label }    │
              │  ~133k producten BE │
              └─────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ProductDataProcessor.ProcessProductsAsync()                        │
│  pc_api/.../Extraction/Css/Services/ProductDataProcessor.cs         │
│                                                                     │
│  Verwerkt in batches van 1.000 producten                            │
└─────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STAP 1 — SQL Pre-filter (PostgreSQL)                               │
│                                                                     │
│  SELECT p.ean                                                       │
│  FROM prices p                                                      │
│  JOIN shop_publication sp ON sp.shop_id = p.shop                    │
│    AND sp.site_id = @publicationId                                  │
│  WHERE p.ean = ANY(@eans)                                           │
│  GROUP BY p.ean                                                     │
│  HAVING COUNT(DISTINCT p.shop) >= 2              ← min. 2 shops    │
│     AND COUNT(DISTINCT CASE WHEN p.shop                             │
│         NOT IN (79,765,7811) THEN p.shop END) >= 1  ← non-Amazon  │
│     AND MIN(p.amount) >= 25                      ← min. €25        │
│     AND MIN(p.amount) < 3000                     ← max. €3.000     │
│                                                                     │
│  ⚠️  Bol.com + bol.plaza tellen hier als 2 aparte shops            │
└─────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STAP 2 — Per-product validatie (C#)                                │
│                                                                     │
│  • Product bestaat in DB?              → anders: skip               │
│  • Heeft merk?                         → anders: skip               │
│  • Heeft thumbnail?                    → anders: skip               │
│  • Heeft EAN-aliases?                  → anders: skip               │
│  • Geblokkeerd productcode (erotica)?  → skip                       │
│  • Prijzen ophalen via IPriceProvider                               │
│  • Eerste prijs ShopType.Miscellaneous → verwijder uit lijst        │
│  • Alle prijzen Amazon-only?           → skip                       │
│                                                                     │
│  Amazon-deduplicatie: ✅                                            │
│  AmazonShopIds = [79, 765, 7810, 7811, 7996, 7997]                 │
│  effectivePriceCount = nonAmazon + (hasAmazon ? 1 : 0)             │
│  effectivePriceCount < 2 → skip                                     │
│                                                                     │
│  Bol.com-deduplicatie: ❌ ONTBREEKT                                 │
│  Bol.com (shop X) + bol.plaza (shop Y) = effectivePriceCount 2     │
│  Google telt dit als 1 merchant → policy-overtreding               │
└─────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STAP 3 — Product bouwen: CreateExtractedProductData()              │
│                                                                     │
│  custom_attribute0 = ProductCode                                    │
│  custom_attribute1 = lowestPrice.Shop.Name  (merchant naam)        │
│  custom_attribute2 = PredictedCommission                            │
│  custom_attribute3 = prices.Count.ToString()  ← NumberOfOffers     │
│  custom_attribute4 = label  (longtail / google_demand / p001 ...)  │
│                                                                     │
│  URL = {domain}/prp/{id}?utm_campaign=css_pla_beta                 │
│  ID  = EAN (virtueel) of KieskeurigId (echt product)               │
└─────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│  GoogleCssExporter.ExportAsync()                                    │
│  pc_api/.../Extraction/Css/Exporters/GoogleCssExporter.cs          │
│                                                                     │
│  Batches van 200 producten                                          │
│  Max. 50 gelijktijdige API-calls                                    │
└─────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│  GoogleCssClient.InsertProductAsync()                               │
│  pc_api/.../Extraction/Css/GoogleCssClient.cs                       │
│                                                                     │
│  Google Content API for Shopping                                    │
│  Service account: css-beta-script@orca-205614.iam.gserviceaccount   │
│  Endpoint: accounts/{merchantId}/products                           │
└─────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Google Merchant Center                                             │
│                                                                     │
│  Kieskeurig.nl  MC: 140803054  │  Kieskeurig.be  MC: 5352009739    │
│  Google Ads:    3380189117     │  Google Ads:    8823617537         │
│  Status: actief                │  Status: GESUSPENDEERD             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Aanvullende CronJob: GoogleCssExpiryJob

```
CronJob: job-google-css-expiry
Schedule: 0 12 * * * (dagelijks om 12:00)
Image: ghcr.io/reshift/job-runner:latest

Verwijdert verlopen producten uit Merchant Center.
Draait slechts 1× per dag — verwijderde producten blijven
tot 24 uur zichtbaar voor Google-crawlers.
```

---

## Betrokken repositories en bestanden

| Component | Locatie |
|-----------|---------|
| CronJob definitie | `compare-infra/applications/jobs/job-runner/extraction/google-css-extraction.yaml` |
| Account configuratie | `compare-infra/applications/jobs/job-runner/configmap.yaml` → `GoogleCss.Accounts` |
| Hoofd extractie job | `pc_api/src/Reshift.ProductCatalog.Jobs/Application/Extraction/Css/GoogleCssExtractionJob.cs` |
| Filter + verwerking | `pc_api/src/Reshift.ProductCatalog.Jobs/Application/Extraction/Css/Services/ProductDataProcessor.cs` |
| Google API client | `pc_api/src/Reshift.ProductCatalog.Jobs/Application/Extraction/Css/GoogleCssClient.cs` |
| Exporter | `pc_api/src/Reshift.ProductCatalog.Jobs/Application/Extraction/Css/Exporters/GoogleCssExporter.cs` |
| Product model | `pc_api/src/Reshift.ProductCatalog.Jobs/Application/Extraction/Css/GoogleCssProductAttribute.cs` |
