# Onderzoeksrapport: Kieskeurig.be CSS Account Suspensie

**Datum:** 2026-04-22  
**Onderzocht door:** Paul Molenaar / Claude (Reshift)  
**Status:** Oorzaak geïdentificeerd — fix nog niet geïmplementeerd  
**Account:** Kieskeurig.be CSS — Merchant Center ID `5352009739`, Google Ads ID `8823617537`

---

## 1. Aanleiding

Google stuurde een waarschuwingsemail naar CSS Center ID `140803054` (Kieskeurig.nl MCC) met melding van "Generic CSS landing page"-overtredingen. De bijgevoegde voorbeeld-URLs betroffen Kieskeurig.be-pagina's. Vervolgens is het volledige BE-account gesuspendeerd.

**Voorbeeld-URLs uit de Google-waarschuwing:**
- `https://www.kieskeurig.be/prp/41862575` → HTTP 404 (verwijderd product)
- `https://www.kieskeurig.be/prp/8719243560463` → HTTP 404
- `https://www.kieskeurig.be/prp/51003170` → HTTP 404
- `https://www.kieskeurig.be/prp/8721128503625` → HTTP 404

Alle voorbeeld-URLs geven 404 — de producten bestaan niet meer op de site.

---

## 2. Eerste hypothesen

| Hypothese | Onderzocht | Conclusie |
|-----------|------------|-----------|
| Cloudflare blokkeert AdsBot-Google | ✅ | **Ontkracht** — AdsBot krijgt HTTP 200, zelfde als normale browser |
| Prijzen niet zichtbaar voor crawler (JS-only) | ✅ | **Ontkracht** — prijzen zijn server-side gerenderd (SSR via Astro) |
| Channable-feed bevat slechte producten | ✅ | **Ontkracht** — Channable-feed heeft 4.418 producten, allemaal ≥2 aanbieders, geen longtail |
| Feed bevat te veel single-merchant producten | ✅ | **Bevestigd — dit is de root cause** |

---

## 3. Feed-analyse via Google Ads API

### Methode

Queries uitgevoerd via de `shopping_product` resource van de Google Ads API (MCC `4308528210`):

```
customer_id BE: 8823617537
customer_id NL: 3380189117
```

### BE — Kieskeurig.be (8823617537)

**Totaal producten in CSS feed:** 128.416

| Label (custom_attribute4) | Aantal | % |
|---------------------------|--------|---|
| longtail | 123.440 | **96,1%** |
| google_demand | 2.808 | 2,2% |
| p001 | 793 | 0,6% |
| p002 | 539 | 0,4% |
| e000 | 836 | 0,7% |

**Aanbod-analyse longtail producten (custom_attribute3 = aantal concurrenten):**

| attr3 | Merchants | Aantal producten | % van longtail |
|-------|-----------|-----------------|----------------|
| 0 | 1 (alleen 1 merchant) | 9.406 | **7,6%** |
| 1 | 2 merchants | ~50.747 | **41,1%** |
| 2+ | 3+ merchants | ~63.287 | 51,3% |

Van de ~50.747 producten met attr3=1 (2 merchants): **een significant deel heeft bol.com + bol.com.be of bol.plaza** — zelfde juridische entiteit, telt als 1 merchant bij Google.

**Conclusie BE:** minstens 7,6% van de feed heeft aantoonbaar slechts 1 merchant. Inclusief bol.com-dubbeltelling is het werkelijke probleem waarschijnlijk 30-50% van het longtail-segment.

### NL — Kieskeurig.nl (3380189117)

**Totaal producten in CSS feed:** ~26.000 (precieze analyse niet volledig afgerond)

- NL heeft een veel kleinere longtail-feed.
- NL heeft doorgaans meer merchants per product door de grotere NL-markt.
- NL-account is **niet** gesuspendeerd.

**Opvallend verschil:** BE heeft 128k producten vs NL ~26k. De BE-feed is 5× groter, terwijl de Belgische markt kleiner is — dit suggereert dat de selectiedrempel voor BE-producten te laag is ingesteld.

---

## 4. Feedbron geïdentificeerd

### De foute feed komt NIET van Channable

Channable genereert `prices_BE.csv` met:
- 4.418 producten
- Alle producten hebben `prices_amount >= 2` (minimaal 2 aanbieders)
- Labels zijn uppercase (`LONGTAIL`, `GOOGLE_DEMAND`)
- Item IDs zijn gewone product-IDs

### De foute feed komt van `job-google-css-extraction`

**Locatie in infra:** `compare-infra/applications/jobs/job-runner/extraction/google-css-extraction.yaml`

**Kenmerken van deze feed:**
- ~128k producten voor BE, ~26k voor NL
- Labels zijn lowercase (`longtail`, `google_demand`)
- Item IDs zijn UUID-formaat
- Schedule: **elk uur** (`0 * * * *`)
- Gebruikt Postgres-tabel `css_labels_kieskeurig_be` als bron voor label-toewijzing

**Relevante configuratie** (uit `job-runner/configmap.yaml`):

```json
"GoogleCss": {
  "Accounts": [
    {
      "AccountId": "5352009739",
      "Name": "Kieskeurig.be",
      "Domain": "https://www.kieskeurig.be",
      "PublicationId": "2",
      "FeedLabel": "BE",
      "ContentLanguage": "NL",
      "LabelsTable": "css_labels_kieskeurig_be"
    }
  ]
}
```

**Service Account:** `css-beta-script@orca-205614.iam.gserviceaccount.com`

---

## 5. Root cause

De Postgres-tabel `css_labels_kieskeurig_be` bevat label-toewijzingen voor producten die **onvoldoende onafhankelijke merchants** hebben. De `GoogleCssExtractionJob` stuurt deze zonder filter door naar Google Merchant Center via de Content API.

Specifieke problemen:
1. **Single-merchant producten** (9.406 stuks, attr3=0) — absolute overtreding CSS-beleid.
2. **Schijnbare 2-merchant producten** waarbij beide aanbieders bol.com zijn (bol.com + bol.com.be of bol.plaza) — tellen bij Google als 1 merchant.
3. **Verlopen producten** — URLs die 404 geven worden kennelijk niet snel genoeg uit de feed verwijderd (de `GoogleCssExpiryJob` draait slechts 1× per dag om 12:00).

---

## 6. Aanbevolen fixes

### Fix 1 (Kritisch): Filter op minimaal 2 onafhankelijke merchants
In de logica die `css_labels_kieskeurig_be` vult: geen producten opnemen met `custom_attribute3 = '0'`.

### Fix 2 (Kritisch): Bol.com-deduplicatie
Merchant-domeinen `bol.com`, `bol.com.be` en `bol.plaza` zijn allemaal eigendom van Ahold-Delhaize en tellen als 1 merchant. Een product met alleen bol-varianten is feitelijk single-merchant en moet worden gefilterd.

### Fix 3 (Belangrijk): Snellere expiry van 404-producten
De `GoogleCssExpiryJob` draait 1× per dag. Verhoog naar meerdere keren per dag, of voeg een real-time trigger toe als producten van de site worden verwijderd.

### Fix 4 (Aanbevolen): Minimumdrempel voor BE verhogen
BE heeft 5× meer producten dan NL terwijl de markt kleiner is. Overweeg een hogere minimumdrempel voor opname in de BE CSS-feed (bijv. minimaal 3 onafhankelijke merchants voor longtail-categorieën).

---

## 7. Vervolgstappen

- [ ] Herstelverzoek indienen bij Google CSS Center na implementatie van fixes
- [ ] `css_labels_kieskeurig_be`-populatielogica aanpassen (Fix 1 + 2)
- [ ] Bol.com-detectie implementeren in feedlogica
- [ ] `GoogleCssExpiryJob` frequentie verhogen
- [ ] NL-feed controleren op dezelfde patronen (preventief)
- [ ] Monitoring instellen: wekelijkse check via Google Ads API op `custom_attribute3` distributie

---

## 8. Bijlagen

### Google Ads API-query gebruikt voor analyse

```sql
SELECT
    shopping_product.item_id,
    shopping_product.custom_attribute1,
    shopping_product.custom_attribute3,
    shopping_product.custom_attribute4,
    shopping_product.issues
FROM shopping_product
WHERE shopping_product.custom_attribute4 = 'longtail'
```

### Infrastructuur betrokken bestanden

| Bestand | Beschrijving |
|---------|-------------|
| `compare-infra/applications/jobs/job-runner/extraction/google-css-extraction.yaml` | CronJob definitie (elk uur) |
| `compare-infra/applications/jobs/job-runner/extraction/google-css-expiry.yaml` | CronJob expiry (dagelijks 12:00) |
| `compare-infra/applications/jobs/job-runner/configmap.yaml` | CSS account config + credentials |
| `kk_feedcheck/fetch_css_feed.py` | Script voor handmatige feed-analyse via Google Ads API |
