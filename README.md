# CSS Management

Kennisbank voor het beheer van de Google CSS (Comparison Shopping Service) accounts van Reshift.

## Accounts

| Account | Merchant Center ID | Google Ads ID | Status |
|---------|-------------------|---------------|--------|
| Kieskeurig.nl | 140803054 | 3380189117 | Actief |
| Kieskeurig.be | 5352009739 | 8823617537 | Gesuspendeerd (apr 2026) |
| Review.nl | 5499041869 | — | Actief |
| MCC (beheer) | — | 4308528210 | — |

## Structuur

```
policies/       Google CSS beleidsdocumenten
reports/        Onderzoeksrapportages per incident
```

## Infrastructuur

De feed naar Google CSS wordt verstuurd door de Kubernetes CronJob `job-google-css-extraction` in de `compare-infra` repository (namespace: `production`). Deze draait elk uur en leest labels uit de Postgres-tabellen `css_labels_kieskeurig_nl` en `css_labels_kieskeurig_be`.

Configuratie: `compare-infra/applications/jobs/job-runner/configmap.yaml`
