# Generic Landing Page — Beleid en Uitleg

**Bronnen:**  
- https://support.google.com/merchants/answer/2948694  
- https://support.google.com/css-center/answer/7524491 (sectie CSS-productpagina's)  
**Geraadpleegd:** 2026-04-22

---

## Wat is een "Generic Landing Page"?

Een "Generic Landing Page" (generieke bestemmingspagina) is een landingspagina die **niet voldoet aan de Google CSS-vereiste** dat er minimaal 2 onafhankelijke merchants zichtbaar zijn voor het betreffende product.

Google beschouwt een pagina als "generiek" als:

- Er slechts **1 merchant** wordt getoond (het product heeft geen concurrerende aanbiedingen).
- De **twee getoonde merchants feitelijk dezelfde entiteit** zijn (bijv. bol.com + bol.com.be / bol.plaza = beiden Ahold-Delhaize).
- De pagina een **HTTP-fout** geeft (404, 5xx) op het moment van crawlen.
- De pagina een **placeholder** bevat of doorverwijst naar een generieke categoriepagina.
- De content **niet overeenkomt** met het product in de feed.

---

## Gevolgen

### Op productniveau
- Disapproval: product verschijnt niet meer in Google Shopping.
- Beperking: product verschijnt slechts in beperkte gevallen.

### Op accountniveau
- **Waarschuwing**: Google stuurt een email met voorbeelden en een termijn.
- **Suspensie**: Als het probleem niet wordt opgelost, wordt het volledige account gebanned.
- **Directe suspensie** (geen waarschuwing) bij wijdverbreide of ernstige overtredingen.

Na suspensie duurt een review maximaal 7 werkdagen.

---

## Voorkomen

1. **Filter producten** met minder dan 2 onafhankelijke merchants vóór indiening bij Google CSS.
2. **Identificeer schijnbare dubbeltelling**: bol.com + bol.com.be + bol.plaza zijn dezelfde juridische entiteit (Ahold-Delhaize) en tellen als 1 merchant.
3. **Monitor regelmatig** via Google Ads API (`shopping_product.issues`) of Merchant Center diagnostiek.
4. **Verwijder verlopen producten** tijdig (gebruik `GoogleCssExpiryJob`).

---

## CSS-specifieke interpretatie

Voor CSS-operators gelden striktere eisen dan voor gewone merchants. De vraag is niet alleen "is de pagina bereikbaar?" maar expliciet:

> "Toont de pagina minstens 2 **verschillende merchant-domeinen** met prijs en koopknop?"

Een pagina met 10 aanbiedingen van alleen bol.com geldt als generiek, net als een pagina met slechts 1 aanbieder.
