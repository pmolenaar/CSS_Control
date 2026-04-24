# Onderzoeksrapport: Kieskeurig.nl CSS Feed — plotselinge volumedaling

**Datum:** 2026-04-24
**Onderzocht door:** Paul Molenaar / Claude (Reshift)
**Status:** Oorzaak geïdentificeerd — geen crisis, wel een aandachtspunt
**Account:** Kieskeurig.nl CSS — Merchant Center ID `140803054`, Google Ads ID `3380189117`
**Vervangt:** Delen van [2026-04-24-feed-kwaliteit.md](2026-04-24-feed-kwaliteit.md) die op verkeerde interpretatie waren gebaseerd — zie §6 "Intrekkingen" onderaan.

---

## Management summary

Tussen 22 en 24 april daalde het aantal producten in de kieskeurig.nl CSS feed (volgens Google Ads API) van **347.096** naar **79.808** — een daling van **77%**. Vrijwel alle verdwenen producten waren **echte producten** (260k van de 260k UUID-geïdentificeerde producten verdwenen; virtuele producten met EAN-IDs bleven relatief stabiel).

**De oorzaak is niet een "feed-crisis" of een bug, maar de combinatie van drie factoren:**

1. **`GoogleCssExpiryJob` hanteert een 12-uurs refresh-drempel.** Elke dag om 12:00 UTC verwijdert deze job actief uit Merchant Center alles wat in de afgelopen 12 uur niet opnieuw door pc_api is gepusht.
2. **Pc_api's deploy-pijplijn was in de aanloop naar 22 april gebroken.** Tim Baijens pushte op 22 april drie deploy-fixes (GitHub Actions, Docker tag, price cast) om de pijplijn weer werkend te krijgen.
3. **Tim's bol-dedup commit (23 april, voor BE-suspensie-opheffing) filtert strenger.** Producten die effectief single-merchant zijn (bol + bol.plaza als 2 tellen werd geblokkeerd) worden niet meer gepusht.

Het gecombineerde effect: nadat pc_api weer draaide en strenger ging filteren, haalde de expiry-job de oude voorraad actief weg. Resultaat: een veel **kleinere maar CSS-compliantere feed**.

Dit is **geen operationele crisis** voor kieskeurig.nl CSS specifiek — de feed is consistent met de multi-merchant CSS-policy. De openliggende vraag is of de feed-grootte ten opzichte van de business-ambitie acceptabel is, en of de 12-uurs expiry-drempel nog passend is gegeven de strengere filtering.

---

## 1. Wat we zagen in de data

Snapshots via Google Ads API (`shopping_product` resource) op klant `3380189117`:

| Metric | 22 apr 13:55 UTC | 24 apr 14:00 UTC | Δ |
|--------|------------------|-------------------|---|
| Totaal producten | 347.096 | 79.808 | **-77,1%** |
| UUID-formatted (echte producten) | 260.204 | 337 | -99,87% |
| Numeric (virtuele producten) | 60.594 | 52.921 | -12,7% |
| Hyphen-coded (`-xxx-nl` — legacy) | 14.383 | 14.357 | -0,2% |
| Overig | 11.915 | 12.193 | +2,3% |

**Status-distributie 24 apr:**
- ELIGIBLE: 126
- ELIGIBLE_LIMITED: 131
- NOT_ELIGIBLE: 82.608 — waarvan **70.879 alleen met `not_eligible_in_any_campaign`** (feed-technisch OK, alleen geen actieve Ads-campagne) en **11.730 met echte feed/quality-issues**

---

## 2. Hoe de verklaring is gevonden

### Stap 1 — Waar komen UUIDs vandaan?

Uit broncode-analyse van `pc_api`:

```csharp
// src/Reshift.ProductCatalog.Domain/Entity.cs
public abstract class Entity {
    public Guid Id { get; set; }     // Product.Id is een Guid
}

// src/Reshift.ProductCatalog.Jobs/Application/Extraction/Css/Services/ProductDataProcessor.cs
var id = virtualProduct ? product.Eans.First() : product.Id.ToString();
// → virtueel: numeric EAN  |  echt: Guid.ToString() = UUID
```

Bevestigd: de 260k UUIDs zijn gewoon de echte producten uit pc_api. Er is **geen aparte "UUID-feed-source"** zoals ik in eerdere analyse vermoedde.

### Stap 2 — Wat veranderde er in pc_api?

Git log op `reshift/pc_api` sinds 20 april:

| Datum | Commit | Wat |
|-------|--------|-----|
| 20 apr | `ae0f363e` | ClickHousePriceHistoryProvider (niet CSS) |
| 22 apr 10:00 | `0cb6265c` | **Fixed deployment of pc_api in GitHub Actions** |
| 22 apr 14:51 | `718898b9` | **Fix docker image tag format to use full SHA for GHCR** |
| 22 apr 16:30 | `802fab42` | **Fix price cast in price_snapshots** |
| 23 apr 09:58 | `b28e2967` | **Refactor ProductDataProcessor to better filter multiple marketplaces** (bol-dedup voor BE-suspensie-opheffing) |

De drie fixes op 22 april suggereren sterk dat de deploy-pijplijn in de dagen ervoor gebroken was. Pas vanaf ~22 april 16:30 draaide pc_api weer werkelijk met een verse image.

### Stap 3 — De verwijderings-motor

Broncode `GoogleCssExpiryJob.cs`:

```csharp
await liveProducts
    .Where(cssProduct => cssProduct.CssProductStatus.LastUpdateDate < DateTime.UtcNow.AddHours(-12))
    .ParallelForEachAsync(async product => {
        await client.DeleteProductAsync(product.Name.Split('/').Last(), cancellationToken);
    });
```

Deze job:
- Draait dagelijks om **12:00 UTC** (CronJob schedule `0 12 * * *`)
- Haalt uit Merchant Center de huidige product-lijst op
- Verwijdert actief alles waar `LastUpdateDate` ouder is dan **12 uur**
- Doet dit parallel, per DELETE-call aan de Google Content API

**De 12-uurs drempel is strikt.** Als pc_api zelfs enkele uren niet draait, valt een grote set producten buiten het venster en wordt ze bij de volgende 12:00-run permanent verwijderd.

### Stap 4 — De tijdlijn rekonstrueren

| Moment | Gebeurtenis | Resultaat |
|--------|-------------|-----------|
| ~18 apr (schatting) | pc_api deploy-pijplijn breekt | Product-refreshes stoppen geleidelijk |
| 21 apr 11:48-11:52 | Ezomerhuis pauzeert 4 brede CSS-campagnes (separate actie, Ads-kant) | Beïnvloedt eligibility, niet MC-aantallen |
| 22 apr 10:00-16:30 | Tim's drie deploy-fixes | Pijplijn hersteld |
| 22 apr 12:00 UTC | Expiry-job draait | Verwijdert eerste batch oude producten |
| **22 apr 13:55 UTC (snapshot)** | NL feed = **347k producten** | Voor Tim's bol-dedup |
| 23 apr 09:58 | Tim's bol-dedup commit | Stricter filter live na volgende deploy |
| 23 apr 12:00 UTC | Expiry-job draait | Verwijdert producten die nog geen strict-filter-refresh hadden gekregen |
| 24 apr 12:00 UTC | Expiry-job draait | Verwijdert meer producten |
| **24 apr 14:00 UTC (snapshot)** | NL feed = **79,8k producten** | 260k echte producten weg |

De 2-daagse vertraging tussen "pijplijn terug online" en "oude content op MC" komt overeen met twee runs van de expiry-job à 12:00 UTC.

---

## 3. Hoe 70k "orphan" producten te interpreteren

Van de 82.608 NOT_ELIGIBLE producten op 24 april heeft **70.879 alleen `not_eligible_in_any_campaign`** als issue. Deze producten zijn:
- Feed-technisch goedgekeurd
- Niet gekoppeld aan een actieve Google Ads Shopping-campagne
- Dit is **geen feed-probleem** — het is een Ads-campagne-scope-probleem

Achtergrond: NL customer `3380189117` heeft **6 ENABLED vs 502 PAUSED** campagnes:

| ENABLED campagnes (6) |
|---|
| Target ROAS experiment - CSS / NL / MOBILE / TELEVISIE |
| Target ROAS experiment - CSS / NL / MOBILE / WASDROGER [Manual CPC] |
| CSS / NL / RETAILPROMO / B45-200 |
| CSS / NL / BOL / CPS _Target_ROAS_Experiment |
| CSS / NL / ERPC |
| CSS / NL / PRPC |

Vier van de op 21 april gepauzeerde campagnes (`CSS/NL/AMAZON`, `AMAZON MARKETPLACE`, `LONGTAIL`, `BOL/CPS`) waren de brede campagnes die grote delen van het assortiment dekten. Ezomerhuis pauzeerde deze — context en reden zijn niet bekend uit dit onderzoek. Die pauze is **separaat** van het volumeverlies: het beïnvloedt welke producten als ELIGIBLE verschijnen, maar is niet verantwoordelijk voor het feit dat 260k producten fysiek uit MC zijn verwijderd.

---

## 4. Is dit een probleem?

### Voor CSS-policy compliance: **nee**

De feed is nu CSS-policy-compliantiger dan voorheen:
- Multi-merchant regel wordt nu strikt gehandhaafd (bol + bol.plaza tellen als 1)
- Technische hygiëne is sterk verbeterd (UTF-8-fouten, GTIN-issues, policy-violations bijna allemaal weg)

### Voor kieskeurig.nl business: **afhankelijk van je perspectief**

- Als je weinig actieve Ads-campagnes hebt en CSS voornamelijk als free-organic-comparison-traffic ziet: **geen issue**
- Als je de feed-grootte als werkend assortiment beschouwt: **relevante daling** om over met marketing/ads-team te praten

### Is de `GoogleCssExpiryJob` 12-uurs drempel nog passend?

Waarschijnlijk te strikt. Met de nieuwe filter levert pc_api minder producten per run, waardoor producten die in de "rand" van de filter zitten snel expireren en bij elke herevaluatie opnieuw ge-delete worden. Een drempel van bijvoorbeeld 36 of 48 uur zou het systeem toleranter maken voor tijdelijke pijplijn-hikups zonder directe CSS-compliance in te leveren.

---

## 5. Wat bijgesteld moet worden in de praktijk

1. **Check Merchant Center UI direct**. De Ads API view die wij gebruiken kan producten tonen die in MC zitten maar niet aan campagnes gekoppeld zijn. Voor de échte feed-state: `https://merchants.google.com/mc/products` op account `140803054`.
2. **Overweeg de expiry-drempel** te verhogen van 12 uur naar 24-48 uur. Eén regel in `GoogleCssExpiryJob.cs`: `AddHours(-12)` → `AddHours(-36)`. Verlies: 24 extra uren dat verouderde producten in MC staan. Winst: robuustheid tegen kortstondige pijplijn-uitval.
3. **Monitor de pc_api cron-runs** actief. Als `job-google-css-extraction` een dag niet draait, verliest de feed een grote chunk binnen 24 uur. Een simpele Slack/email-alert bij missed cron-runs zou dat vangen.
4. **Evalueer de Ads-campagne-strategie** los van dit onderzoek. 502 paused vs 6 enabled is een opmerkelijke verhouding. Dit lijkt een bewuste business-keuze te kunnen zijn, maar verdient aandacht van marketing.

---

## 6. Intrekkingen — wat ik in eerdere analyse verkeerd zei

In het vorige rapport (`2026-04-24-feed-kwaliteit.md`) stonden een aantal claims die op incomplete info gebaseerd waren. Voor de duidelijkheid:

| Eerdere claim | Correctie |
|---|---|
| "BE: 51,6% van producten schendt multi-merchant regel" | Verkeerd geïnterpreteerd veld. `custom_attribute3` is **DeliveryCode** (leverbaar/op voorraad), niet offer-count. De echte multi-merchant compliance is via de `effectivePriceCount` check in de code — daar is bol-dedup nu actief. |
| "NL custom_attributes bevatten onverwachte waarden — duidt op code-corruptie" | Geen corruptie. `custom_attribute1 = Shop.Name` dus shops als `goed`, `slecht`, `cashback` zijn echte shop-namen in de pc_api database. |
| "Er is een tweede/derde feed-source (UUID-generator)" | Nee — de UUIDs komen uit pc_api zelf (`Product.Id = Guid`). Er is wel een **verlaten Cloud Function** `sea_google_css_generator` (crashing sinds dataset `sea_top_200_for_devs` verwijderd is op 2026-03-16 door `marvin.ronk@oncore.ai`), maar die contribueert niet meer aan de actieve feed. |
| "Bol-dedup niet gedeployed" | In code bevestigd gedeployed (commit `b28e2967`, 23 apr). Productie-verificatie via actieve pod-image-hash is nog niet gedaan (vereist Scaleway K8s-toegang). |
| "NL feed is in crisis-staat" | Overdreven. Feed is kleiner en technisch schoner. Het "crisis"-label kwam door verkeerde interpretatie van de Ads API view. |
| "Een tweede feed-source stopt massaal met pushen" | De werkelijke oorzaak is `GoogleCssExpiryJob` dat met 12-uurs drempel verouderde producten verwijdert. |

Het originele 2026-04-22 rapport over de BE-suspensie blijft ongewijzigd — dat onderzoek en de bol-dedup-aanbeveling daaruit zijn correct en door Tim opgevolgd.

---

## 7. Verificatie: hebben onze efficiency-slagen pc_api geraakt?

Gedreven door de vraag of de feed-krimp toch door onze BigQuery-optimalisaties van 23 april kan zijn veroorzaakt. Per actie nagegaan:

| Onze actie | Target | Leest pc_api dit? |
|---|---|---|
| `kieskeurignl_erpc_calculation` gepauzeerd | BQ scheduled query → `reshift_insights.kieskeurignl_erpc_calculation` | **Nee** — pc_api leest uit `reshift_insights.erpc_table_7d` (andere tabel) |
| Bemmel-query 5min → 1×/dag | scheduled query `68d84eee` (jspijkstra) | **Nee** — Bemmel-rankingsysteem heeft geen raakpunt met CSS-extraction |
| Amazon-queries gepauzeerd | `conversions_staging.amazon_earnings_kieskeurig_id_ean` | **Nee** — post-processing van click-attributie, niet feed-input |

**Conclusie:** onze optimalisaties raakten de feed niet.

### Wel: een verwante vondst

Pc_api's `PriceErpcSyncJob.cs`:
```csharp
"SELECT ean, shop_id, erpc 
 FROM `kieskeurig-data-analyse.reshift_insights.erpc_table_7d` 
 WHERE erpc IS NOT NULL AND erpc > 0"
```

Deze tabel heeft **0 rijen** en is laatst gewijzigd op **2025-09-30** — al 7 maanden dood. Tim heeft op 4 maart `applyErpcOverride` toegevoegd aan `GetOrderedPricesAsync` (`571de796`), maar er is geen data om te overriden. Niet feed-brekend (fallback werkt), wel onderhoud-debt.

Er zijn **vier actieve ERPC-achtige scheduled queries** in BigQuery die naar andere tabellen schrijven:
- `erpc_product_daily` (elke 4u)
- `erpc_product_daily_history` (dagelijks 09:30)
- `erpc_ean_buckets_daily` (dagelijks 09:15)
- `kieskeurignl_erpc_calculation` (disabled door ons 23 apr — harmloos)

Niemand vult `erpc_table_7d`. Actie: of pc_api migreren naar `erpc_product_daily`, of de sync-code verwijderen. Zie AANDACHTSLIJST item 11.

---

## 8. Onmiddellijk herstel — concreet uitvoerbare stappen

**Doel:** de feed terug naar zijn werkelijke kwaliteits-maximum brengen, zonder Tim's bol-dedup-fix te ondermijnen.

### Strategie

Tim's filter is correct. De 260k-drop komt doordat `GoogleCssExpiryJob` producten verwijderde sneller dan pc_api ze kon rehydrateren met de nieuwe filter-logica. Herstel = voorkomen dat er meer verdwijnen + pc_api laten rehydrateren.

### Stap 1 — Stop de bloeding (5 min)

Schort de expiry-job op zodat hij niet verder verwijdert terwijl we pc_api laten werken.

```bash
kubectl patch cronjob job-google-css-expiry -n production \
  -p '{"spec":{"suspend":true}}'

kubectl get cronjob job-google-css-expiry -n production -o jsonpath='{.spec.suspend}'
# Verwacht: true
```

Risico: nul. De job doet alleen DELETE. Tijdens de pause blijven er ook niet-gekwalificeerde producten staan, maar die worden toch al door Google's 30-dagen-interne-expiry opgeruimd zolang pc_api ze niet refresht.

### Stap 2 — Forceer volledige push (30-60 min uitvoeringstijd)

Trigger `job-google-css-extraction` direct, los van het uurlijkse cron-schema.

```bash
kubectl create job --from=cronjob/job-google-css-extraction \
  manual-push-$(date +%Y%m%d-%H%M) -n production

# Volg uitvoering:
kubectl logs -f job/manual-push-<TIMESTAMP> -n production
```

Belangrijke log-regels om op te letten:
- `Account: Kieskeurig.nl, Publication ID: 1, Total Products: N` (aantal in te voeren kandidaten)
- `Performance: X products/sec | Skip Summary - NotFound: ..., NotEnoughPrices: ...`
- `Batch processing completed: ... products yielded`

Het `Yielded`-aantal is hoeveel er daadwerkelijk naar Merchant Center gepusht zijn.

### Stap 3 — Verifieer (5 min)

Zodra de job klaar is:

```bash
cd ~/kk_feedcheck && python3 fetch_css_feed.py
```

Vergelijk de nieuwe `NL_*.json` met `NL_20260424T140032Z.json`:
- **Product-count > 79.808**: pc_api heeft producten teruggepusht die door de expiry-job waren weggehaald — succes
- **Product-count ≈ 79.808 of minder**: Tim's filter filtert een groot deel van de 260k correct uit — geen bug, wel bewust kleinere feed

Beide uitkomsten zijn legitiem. De **ware feed-grootte** is wat je na stap 3 ziet.

### Stap 4 — Heractiveer expiry-job (1 min)

```bash
kubectl patch cronjob job-google-css-expiry -n production \
  -p '{"spec":{"suspend":false}}'
```

### Stap 5 (permanent) — Verhoog expiry-drempel naar 48u

Pull-request op `reshift/pc_api`, bestand `src/Reshift.ProductCatalog.Jobs/Application/Extraction/Css/GoogleCssExpiryJob.cs`:

```diff
- .Where(cssProduct => cssProduct.CssProductStatus.LastUpdateDate < DateTime.UtcNow.AddHours(-12))
+ .Where(cssProduct => cssProduct.CssProductStatus.LastUpdateDate < DateTime.UtcNow.AddHours(-48))
```

Eén regel, zeer laag risico. Bij een toekomstige pijplijn-hikup heeft het systeem 48u speling in plaats van 12u. Minimale CSS-compliance-impact (Google houdt producten sowieso 30 dagen).

### Tijdschema

| Stap | Duur | Wachttijd daarna |
|---|---|---|
| 1. Suspend expiry | 1 min | — |
| 2. Trigger extraction | 30-60 min | wachten op completion |
| 3. Verifieer snapshot | 5 min | — |
| 4. Resume expiry | 1 min | — |
| 5. PR expiry-drempel | 15 min PR, + review-cycle | deploy afwachten |

Totaal onmiddellijk herstel: **~1-1,5 uur** vanaf moment dat je kubectl-toegang hebt.

### Toegangsvereisten

- `kubectl` context voor Scaleway cluster, namespace `production` — beschikbaar via de `compare-infra` setup (pas nog bevestigen dat jouw user admin-rechten heeft op die namespace)
- Optioneel: PR-review-rechten op `reshift/pc_api` voor stap 5

### Risico-beoordeling

| Stap | Risico | Mitigatie |
|---|---|---|
| Suspend expiry-job | Laag — oude non-compliante producten blijven tijdelijk staan | Houd stap kort (<2u) en Resume direct na verificatie |
| Manual extraction trigger | Laag — doet hetzelfde als uurlijks, alleen eerder | Monitor logs; bij failure: gewone cron pakt het later op |
| Verifieer snapshot | Nul risico | — |
| Resume expiry | Nul risico | — |
| 12u → 48u code-change | Laag — uitstel van cleanup van stale products | Review door Tim; eventueel eerst op tst-account |

### Wat NIET doet deze hersteloperatie

- Producten die Tim's filter terecht afwijst (single-merchant disguised as multi) komen NIET terug. Dat is de bedoeling.
- De 4 gepauzeerde Ads-campagnes blijven PAUSED. Dit is een Ads-management-kwestie, geen feed-kwestie.

---

## 9. Bijlagen

- Ads API snapshots: `~/kk_feedcheck/snapshots/NL_20260422T135545Z.json`, `NL_20260424T140032Z.json`
- Code-referenties:
  - `~/pc_api/src/Reshift.ProductCatalog.Jobs/Application/Extraction/Css/Services/ProductDataProcessor.cs`
  - `~/pc_api/src/Reshift.ProductCatalog.Jobs/Application/Extraction/Css/GoogleCssExpiryJob.cs`
  - `~/compare-infra/applications/jobs/job-runner/extraction/google-css-expiry.yaml`
- Eerder: [2026-04-22-kieskeurig-be-suspensie.md](2026-04-22-kieskeurig-be-suspensie.md), [2026-04-24-feed-kwaliteit.md](2026-04-24-feed-kwaliteit.md) (ingetrokken delen — zie §6)
- Architectuurreview: [../architecture/css-feed-architectuurreview.md](../architecture/css-feed-architectuurreview.md)
