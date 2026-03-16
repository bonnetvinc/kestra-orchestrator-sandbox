# kestra-orchestrator-sandbox

POC Kestra — 3 use cases avancés illustrant les patterns de production.

## Prérequis

- [Docker](https://docs.docker.com/get-docker/) + Docker Compose
- [Taskfile](https://taskfile.dev/docs/installation)

## Structure

```
.
├── core/                               # Infrastructure
│   ├── docker-compose.yaml             # Kestra distribué (scheduler/worker/executor/webserver)
│   └── .env.example                    # Variables d'environnement
│
├── flows/                              # Flows Kestra
│   ├── uc1-etl-subflows/               # UC1 — ETL Modulaire
│   │   ├── shared/
│   │   │   ├── http-extractor.yaml     # Subflow générique HTTP
│   │   │   └── data-quality.yaml       # Subflow DQ réutilisable
│   │   └── exchange-rates-pipeline.yaml
│   ├── uc2-event-driven/               # UC2 — Event-Driven + Approval Gate
│   │   └── api-ingestion-pipeline.yaml
│   └── uc3-self-healing-batch/         # UC3 — Batch Auto-Résilient
│       ├── nightly-batch.yaml
│       ├── process-batch-item.yaml     # Subflow batch (ForEachItem)
│       └── batch-monitor.yaml          # Monitor découplé (ExecutionStatus trigger)
│
├── scripts/                            # Scripts standalone (dev/test local)
│   ├── uc1/transform_rates.py
│   ├── uc2/validate_payload.js
│   └── uc3/process_chunk.py
│
├── tests/                              # Flows de test d'intégration
│   ├── uc1/test-exchange-rates.yaml
│   ├── uc2/test-file-processor.yaml
│   └── uc3/test-nightly-batch.yaml
│
└── Taskfile.yml
```

---

## Démarrage rapide

```sh
# 1. Config locale
cp core/.env.example .env.local
cp .env.secrets.example .env.secrets
# → éditer .env.secrets avec vos clés

# 2. Démarrer Kestra
task up

# 3. Vérifier que l'API répond
task health

# 4. Déployer tous les flows
task flows:deploy

# 5. Lancer les tests
task flows:test
```

Interface : [http://localhost:8080](http://localhost:8080)

---

## Use Cases

### UC1 — ETL Modulaire avec Subflows
**Namespace :** `demo.finance` / `demo.finance.shared`

Pipeline de taux de change quotidiens avec état incrémental.

| Pattern | Implémentation |
|---|---|
| Subflows réutilisables | `http-extractor` + `data-quality` dans `demo.finance.shared` |
| KV Store incrémental | Checkpoint `exchange_rates_last_run_USD` (TTL 30j) |
| Namespace Secrets | `{{ secret('EXCHANGE_API_KEY') }}` |
| Plugin defaults | timeout + retry HTTP au niveau flow |
| Schedule + Backfill | Lun-ven 9h, backfill depuis 2025-01-01 |
| Error handler | `errors:` block avec log (→ Slack en prod) |

```sh
task flows:deploy:uc1
task flow:run NAMESPACE=demo.finance FLOW_ID=exchange-rates-pipeline
```

---

### UC2 — Event-Driven avec Human Approval Gate
**Namespace :** `demo.platform`

Ingestion multi-sources avec approbation humaine avant chargement.

| Pattern | Implémentation |
|---|---|
| Multi-triggers | Schedule + Webhook (avec secret) + Manuel |
| Parallel tasks | `EachParallel` sur les sources (Node.js) |
| Human Approval Gate | `Pause` + `onResume` avec décision |
| Conditional routing | `runIf` sur load/skip-dry-run/skip-rejected |
| Dry run | Input `dry_run=true` bypasse le chargement |

```sh
task flows:deploy:uc2

# Trigger via webhook :
curl -X POST "http://localhost:8080/api/v1/executions/webhook/demo.platform/api-ingestion-pipeline/WEBHOOK_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"source_ids": "posts,users", "dry_run": true}'
```

---

### UC3 — Batch Auto-Résilient avec Monitoring
**Namespace :** `demo.batch`

Batch nuit avec traitement distribué et observateur découplé.

| Pattern | Implémentation |
|---|---|
| ForEachItem distribué | Subflow `process-batch-item` par batch de 10 |
| Concurrency limit | `concurrencyLimit: 3` + flow-level `QUEUE` |
| KV Checkpointing | Reprise depuis le dernier offset (self-healing) |
| errors block | Sauvegarde d'urgence du checkpoint avant crash |
| ExecutionStatus trigger | `batch-monitor` découplé surveille `nightly-batch` |

```sh
task flows:deploy:uc3
task flow:run NAMESPACE=demo.batch FLOW_ID=nightly-batch
```

---

## Secrets

Les secrets sont stockés dans le Secret Store Kestra (chiffrés, jamais loggés).

```sh
cp .env.secrets.example .env.secrets
# Remplir .env.secrets avec les vraies valeurs

task secrets:set NAMESPACE=demo.finance
task secrets:set NAMESPACE=demo.platform
task secrets:set NAMESPACE=demo.batch
# Ou tout d'un coup :
task secrets:set:all
```

Utilisation dans un flow :
```yaml
headers:
  Authorization: "Bearer {{ secret('EXCHANGE_API_KEY') }}"
```

---

## Tests

```sh
task flows:test          # tous les UC
task flows:test:uc1      # UC1 uniquement
task flows:test:uc2      # UC2 uniquement
task flows:test:uc3      # UC3 uniquement
```

Les flows de test sont déployés dans des namespaces `*.tests` isolés.

---

## Déploiement Remote (SSH)

```sh
# Dans .env.local :
# SSH_HOST=kestra.example.com
# SSH_USER=ubuntu
# SSH_KEY_PATH=~/.ssh/id_rsa
# KESTRA_REMOTE_URL=https://kestra.example.com
# KESTRA_REMOTE_USER=admin@kestra.io
# KESTRA_REMOTE_PASSWORD=xxx

# Option A : deploy direct (port 8080 exposé)
task deploy:remote

# Option B : tunnel SSH (port non exposé)
task deploy:tunnel        # terminal 1 — garde le tunnel ouvert
task deploy:tunnel:deploy # terminal 2 — deploy via localhost:18080
```

---

## Référence Taskfile

| Commande | Description |
|---|---|
| `task up` | Démarrer Kestra |
| `task down` | Stopper Kestra |
| `task down:prune` | Stopper + supprimer les volumes |
| `task health` | Vérifier que l'API répond |
| `task flows:validate` | Valider tous les flows (dry-run) |
| `task flows:deploy` | Déployer tous les flows |
| `task flows:deploy:uc1/uc2/uc3` | Déployer un UC spécifique |
| `task flows:test` | Lancer tous les tests |
| `task secrets:set NAMESPACE=...` | Push secrets → Kestra |
| `task secrets:list NAMESPACE=...` | Lister les secrets d'un namespace |
| `task deploy:remote` | Deploy remote via SSH |
| `task deploy:tunnel` | Ouvrir un tunnel SSH |
| `task flow:run NAMESPACE=... FLOW_ID=...` | Déclencher un flow |
| `task flow:status NAMESPACE=... FLOW_ID=...` | Status des dernières exécutions |
| `task logs SERVICE=kestra-worker` | Logs d'un service |

---

## Liens utiles

- [Documentation Kestra](https://kestra.io/docs/)
- [Plugin Reference](https://kestra.io/plugins/)
- [GitHub Kestra](https://github.com/kestra-io/kestra)
