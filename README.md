# SentimentLive — Analyse de sentiment temps réel par NLP

> Détectez les crises de réputation avant qu'elles explosent. Sentiment en temps réel, alertes instantanées.

[![FastAPI](https://img.shields.io/badge/FastAPI-0.111-009688?style=flat&logo=fastapi)](https://fastapi.tiangolo.com)
[![Next.js](https://img.shields.io/badge/Next.js-14-black?style=flat&logo=next.js)](https://nextjs.org)
[![Transformers](https://img.shields.io/badge/Transformers-4.41-yellow?style=flat)](https://huggingface.co/transformers)
[![WebSocket](https://img.shields.io/badge/WebSocket-Native-6366f1?style=flat)](https://developer.mozilla.org/WebSockets)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791?style=flat&logo=postgresql)](https://postgresql.org)

---

## Table des matières
1. [Vue d'ensemble](#vue-densemble)
2. [Stack technique](#stack-technique)
3. [Architecture mono-repo](#architecture-mono-repo)
4. [Diagrammes UML](#diagrammes-uml)
5. [PRD](#prd)
6. [User Stories](#user-stories)
7. [Règles métier](#règles-métier)
8. [Spécification API](#spécification-api)
9. [Simulation UI](#simulation-ui)
10. [Dataset](#dataset)
11. [Déploiement](#déploiement)
12. [CI/CD](#cicd)
13. [Roadmap](#roadmap)

---

## Vue d'ensemble

SentimentLive analyse en temps réel des flux de texte (réseaux sociaux, avis clients, tickets support) et affiche l'évolution du sentiment sur un dashboard live. Le pipeline NLP utilise DistilBERT (CPU-friendly, <50ms par texte) pour la classification 3-classes (POSITIVE / NEGATIVE / NEUTRAL), avec extraction d'entités nommées, détection de spikes et alertes configurables.

**Domaine :** Marketing / Brand Monitoring / Support Client  
**Dataset :** [Twitter US Airline Sentiment (Kaggle)](https://www.kaggle.com/datasets/crowdflower/twitter-airline-sentiment) — 14 640 tweets, 3 classes  
**Port VM :** 3036 | **Sous-domaine :** sentimentlive.wikolabs.com

---

## Stack technique

| Couche | Technologie | Rôle |
|--------|------------|------|
| Frontend | Next.js 14, TypeScript, Tailwind CSS, Recharts | Dashboard live, graphes temporels, word cloud |
| Backend | FastAPI (Python 3.11), WebSocket natif, asyncio | API REST + WebSocket push temps réel |
| NLP Classification | Transformers (distilbert-base-uncased-finetuned-sst-2-english) | Sentiment 3-classes, <50ms CPU |
| NER | spaCy (en_core_web_sm) | Extraction entités nommées (brand, produit, personne) |
| Baseline | VADER (vaderSentiment) | Fallback rapide / comparaison |
| Base de données | PostgreSQL 16 (analyses historiques, alertes) | Persistance |
| Stream Buffer | Redis 7 (Pub/Sub + list) | Buffer flux entrant + distribution WebSocket |
| Infra | Docker Compose, Nginx | VM mono-repo (port 3036) |

### backend/requirements.txt
```
fastapi==0.111.0
uvicorn[standard]==0.29.0
transformers==4.41.2
torch==2.3.0+cpu
spacy==3.7.4
vaderSentiment==3.3.2
redis==5.0.4
pandas==2.2.2
asyncpg==0.29.0
sqlalchemy[asyncio]==2.0.30
pydantic==2.7.1
python-multipart==0.0.9
```

---

## Architecture mono-repo

```
sentimentlive/
├── frontend/
│   ├── src/app/
│   │   ├── page.tsx             # Dashboard principal live
│   │   ├── batch/               # Analyse CSV en lot
│   │   └── config/              # Configuration alertes
│   └── src/components/
│       ├── LiveSentimentFeed.tsx   # Feed textes + badges sentiment
│       ├── SentimentTimeline.tsx   # Recharts AreaChart rolling 5min
│       ├── AlertBanner.tsx         # Bannière crise détectée
│       ├── EntityWordCloud.tsx     # Entités liées au sentiment
│       └── BatchUpload.tsx         # Upload CSV → analyse bulk
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── routers/
│   │   │   ├── analysis.py         # POST /analyze, GET /history
│   │   │   ├── stream.py           # WebSocket /ws/stream
│   │   │   └── alerts.py           # GET /alerts, POST /config
│   │   ├── services/
│   │   │   ├── classifier.py       # DistilBERT pipeline
│   │   │   ├── ner.py              # spaCy NER
│   │   │   ├── aggregator.py       # Fenêtres glissantes 5min
│   │   │   ├── alert_engine.py     # Règles alertes
│   │   │   └── stream_manager.py   # Redis Pub/Sub + WebSocket
│   │   └── models/
│   │       ├── analysis.py
│   │       └── alert.py
│   ├── requirements.txt
│   └── Dockerfile
├── docker-compose.yml
└── .github/workflows/deploy.yml
```

---

## Diagrammes UML

### Architecture temps réel

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#6366f1', 'primaryTextColor': '#fff', 'primaryBorderColor': '#4338ca', 'lineColor': '#374151', 'secondaryColor': '#f0f0ff'}}}%%
flowchart TB
    subgraph SOURCES["📥 Sources"]
        TWITTER["Twitter/X API\n(optionnel)"]
        CSV["Upload CSV\n(batch mode)"]
        MOCK["Générateur mock\n(demo)"]
        API_IN["POST /analyze\n(webhook)"]
    end

    subgraph VM["🖥️ VM — Docker Compose (port 3036)"]
        API["FastAPI\nWebSocket + REST"]
        BERT["DistilBERT\nClassifier"]
        NER["spaCy\nNER"]
        AGG["Aggregator\nFenêtre 5min"]
        ALERT["Alert Engine\nSeuils configurables"]
        PUB["Redis Pub/Sub\nStream buffer"]
        PG[("PostgreSQL\nHistorique")]
        NEXT["Next.js 14\nDashboard live"]
    end

    subgraph OUTPUT["📤 Sorties"]
        SLACK["Slack webhook\n(alerte)"]
        EMAIL["Email\n(rapport)"]
    end

    SOURCES --> API
    API --> BERT --> NER
    NER --> AGG --> ALERT
    AGG --> PUB
    PUB --> NEXT
    ALERT --> SLACK
    ALERT --> PG
    BERT --> PG
    NEXT --> API
```

### Séquence — WebSocket temps réel

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#6366f1', 'primaryTextColor': '#fff', 'lineColor': '#374151'}}}%%
sequenceDiagram
    participant BROWSER as Browser (Next.js)
    participant WS as WebSocket (/ws/stream)
    participant BERT as DistilBERT
    participant NER as spaCy NER
    participant AGG as Aggregator
    participant ALERT as Alert Engine
    participant REDIS as Redis Pub/Sub
    participant PG as PostgreSQL

    BROWSER->>WS: Connect WebSocket
    WS-->>BROWSER: {type: "connected", status: "live"}

    loop Chaque nouveau texte entrant
        Note over WS: Source: API POST, Twitter, ou mock generator
        WS->>BERT: classify(text)
        BERT-->>WS: {label: "NEGATIVE", score: 0.94}

        WS->>NER: extract_entities(text)
        NER-->>WS: [{entity: "Vol AF001", type: "ORG"}]

        WS->>AGG: update_window(label, score, entities)
        AGG-->>WS: {positive_5min: 34%, negative_5min: 58%}

        WS->>ALERT: check_rules(window_stats)
        Note over ALERT: Rule: negative > 60% en 5min → CRISIS

        alt Alerte déclenchée
            ALERT->>PG: INSERT alert_event
            WS-->>BROWSER: {type:"alert", level:"CRISIS", message:"..."}
        end

        WS->>REDIS: PUBLISH stream_channel {text, sentiment, entities}
        WS-->>BROWSER: {type:"sentiment", text, label, score, entities, window_stats}
    end
```

### Modèle de données (ER)

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#6366f1', 'primaryTextColor': '#fff', 'lineColor': '#374151'}}}%%
erDiagram
    ANALYSIS {
        uuid id PK
        text content
        string source
        string lang
        string label
        float positive_score
        float negative_score
        float neutral_score
        jsonb entities
        string aspect
        timestamp analyzed_at
    }
    WINDOW_AGGREGATE {
        uuid id PK
        timestamp window_start
        timestamp window_end
        int total_count
        float positive_pct
        float negative_pct
        float neutral_pct
        boolean spike_detected
    }
    ALERT {
        uuid id PK
        string level
        string message
        jsonb trigger_data
        boolean acknowledged
        timestamp triggered_at
        timestamp ack_at
    }
    ALERT_CONFIG {
        uuid id PK
        string rule_name
        float threshold_pct
        int window_minutes
        string level
        boolean active
    }

    ANALYSIS }o--|| WINDOW_AGGREGATE : "agrégé dans"
    ALERT }o--|| ALERT_CONFIG : "défini par"
```

### Machine à états — Niveaux d'alerte

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#6366f1', 'primaryTextColor': '#fff', 'lineColor': '#374151'}}}%%
stateDiagram-v2
    [*] --> NORMAL : Sentiment > 40% positif
    NORMAL --> WARNING : Négatif 40-60% en 5min
    WARNING --> CRISIS : Négatif > 60% en 5min
    CRISIS --> WARNING : Négatif redescend 40-60%
    WARNING --> NORMAL : Négatif < 40%
    CRISIS --> NORMAL : Négatif < 40% (résolution rapide)
    WARNING --> [*]
    CRISIS --> [*]
```

---

## PRD

### Problème
Une crise de réputation non détectée peut coûter des millions en quelques heures (ex. United Airlines -4% en 1 journée après incident viral). Les équipes marketing surveillent manuellement les mentions — trop lent, trop tardif. Les dashboards analytiques traditionnels fonctionnent en J-1, pas en temps réel.

### Solution
Pipeline NLP temps réel : classify + aggregate + alert en moins de 2 secondes par texte. Dashboard WebSocket mis à jour à chaque nouvel élément. Alertes Slack dès qu'un seuil de sentiment négatif est franchi.

### Utilisateurs cibles
| Persona | Besoin |
|---------|--------|
| Responsable marketing | Monitoring brand en temps réel, détection crises |
| Responsable support | Détecter les vagues de mécontentement clients |
| Community manager | Réagir aux tendances avant qu'elles deviennent virales |

### OKRs
- Détection d'une crise de sentiment < 2 min après le début du spike
- Latence de classification par texte < 50 ms
- Disponibilité WebSocket > 99.5%
- Précision modèle > 87% sur benchmark Twitter Airline

---

## User Stories

```
US-01 [Marketing] En tant que responsable marketing,
      je veux un dashboard live du sentiment de ma marque
      afin de réagir en temps réel aux crises de réputation.

US-02 [Système] En tant que moteur d'alerte,
      je veux envoyer une notification Slack
      dès que le sentiment négatif dépasse 60% en 5 minutes
      afin d'alerter l'équipe avant que la crise s'emballe.

US-03 [Analyste] En tant qu'analyste,
      je veux uploader un CSV de 10 000 avis et voir l'analyse en lot
      afin d'auditer le sentiment sur une période passée.

US-04 [Marketing] En tant que community manager,
      je veux voir quelles entités (marques, produits, personnes)
      sont associées aux tweets négatifs
      afin d'identifier l'objet précis du mécontentement.

US-05 [Dev] En tant que développeur,
      je veux une API WebSocket documentée
      afin d'intégrer le flux sentiment dans notre dashboard interne.

US-06 [Support] En tant que responsable support,
      je veux comparer le sentiment de cette semaine vs la semaine dernière
      afin de mesurer l'impact de nos actions correctives.
```

---

## Règles métier

Simulables dans l'UI avec un générateur de tweets mock.

| # | Règle | Valeur par défaut | Simulable UI |
|---|-------|------------------|-------------|
| R1 | Seuil alerte WARNING | Négatif > 40% en 5 min | ✅ Slider seuil |
| R2 | Seuil alerte CRISIS | Négatif > 60% en 5 min | ✅ |
| R3 | Spike détection | Volume > 3× moyenne 5min → flag "viral" | ✅ |
| R4 | Confidence minimum | score < 0.6 → label "Mixed" | ✅ Visible dans feed |
| R5 | Fenêtre d'agrégation | Rolling 5 min, update toutes les 10s | ✅ Configurable |
| R6 | Aspect-based | Extraction (livraison / qualité / prix) + sentiment associé | ✅ Filter aspect |
| R7 | Langue auto | Detect FR/EN/ES → modèle approprié | ✅ Tag langue |
| R8 | Batch mode | Upload CSV → analyse asynchrone → export PDF | ✅ Batch UI |
| R9 | Comparaison historique | Période sélectionnable : 7j / 30j / custom | ✅ Date picker |
| R10 | Rate limiting | 1 000 req/min (protection CPU DistilBERT) | ✅ Counter visible |

---

## Spécification API

**Base URL :** `http://sentimentlive.wikolabs.com/api/v1`  
**WebSocket :** `ws://sentimentlive.wikolabs.com/ws/stream`

### POST /analyze
```json
// Request
{"text": "Le vol AF001 était catastrophique, jamais je ne reprendrai cette compagnie !", "source": "twitter"}

// Response (< 50ms)
{
  "id": "analysis_uuid",
  "label": "NEGATIVE",
  "scores": {"positive": 0.02, "negative": 0.94, "neutral": 0.04},
  "entities": [{"text": "AF001", "label": "ORG"}, {"text": "compagnie", "label": "ORG"}],
  "aspect": "service",
  "lang": "fr",
  "latency_ms": 38
}
```

### GET /history
```
GET /history?source=twitter&label=NEGATIVE&date_from=2025-05-20&limit=100
```

### GET /windows
Retourne les agrégats par fenêtres de 5 minutes pour la dernière heure.

### WebSocket /ws/stream
```json
// Message envoyé par le serveur à chaque nouvelle analyse
{
  "type": "sentiment",
  "data": {
    "text": "...",
    "label": "NEGATIVE",
    "score": 0.94,
    "entities": [...],
    "window_stats": {"positive_pct": 32, "negative_pct": 61, "volume": 47},
    "alert": {"level": "CRISIS", "message": "Sentiment négatif > 60% depuis 3 min"}
  }
}
```

### GET /alerts/config
### PUT /alerts/config
Configure les seuils d'alerte par niveau (WARNING, CRISIS) et les canaux (Slack webhook).

---

## Simulation UI

Mode démo **sans Twitter API** — générateur de tweets mock embarqué.

| Composant | Description |
|-----------|-------------|
| **Live Tweet Feed** | Génère un tweet mock toutes les 1–3s (setTimeout random) avec badge POSITIVE/NEGATIVE/NEUTRAL |
| **Sentiment Timeline** | Recharts AreaChart stacked, mise à jour en temps réel via WebSocket mock local |
| **Alert Banner** | S'affiche automatiquement quand le NEGATIVE dépasse le seuil configuré |
| **Entity WordCloud** | Entités les plus mentionnées + couleur selon sentiment associé |
| **Crisis Simulator** | Bouton "Déclencher une crise" → injecte 20 tweets négatifs en 10s |
| **Batch Upload** | Upload CSV de tweets → analyse synchrone → stats + export |
| **Aspect Filter** | Filtrer par livraison / qualité / prix / service |

```typescript
// frontend/src/lib/mock-generator.ts
const NEGATIVE_TWEETS = [
  "Encore un retard d'1h sans explication. Service déplorable.",
  "Colis perdu pour la 3ème fois ! Je change de prestataire.",
]
export function generateMockTweet(): Tweet {
  const isNeg = Math.random() < 0.3 // 30% négatif par défaut
  return { text: isNeg ? sample(NEGATIVE_TWEETS) : sample(POSITIVE_TWEETS), source: "mock" }
}
```

---

## Dataset

**Kaggle :** [Twitter US Airline Sentiment](https://www.kaggle.com/datasets/crowdflower/twitter-airline-sentiment)

```bash
kaggle datasets download -d crowdflower/twitter-airline-sentiment -p backend/app/ml/data/
```

**Contenu :** 14 640 tweets sur 6 compagnies aériennes US, labellisés manuellement : positive (2 363), negative (9 178), neutral (3 099). Utilisé pour fine-tuner DistilBERT sur le domaine social media.

---

## Déploiement

```yaml
version: "3.9"
services:
  postgres:
    image: postgres:16-alpine
    environment: {POSTGRES_DB: sentimentlive, POSTGRES_USER: sl_user, POSTGRES_PASSWORD: "${POSTGRES_PASSWORD}"}
    volumes: [pg_data:/var/lib/postgresql/data]

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 128mb

  backend:
    build: ./backend
    environment:
      DATABASE_URL: postgresql+asyncpg://sl_user:${POSTGRES_PASSWORD}@postgres/sentimentlive
      REDIS_URL: redis://redis:6379
      MODEL_NAME: distilbert-base-uncased-finetuned-sst-2-english
      SPACY_MODEL: en_core_web_sm
    depends_on: [postgres, redis]
    expose: ["8000"]

  frontend:
    build: ./frontend
    expose: ["3000"]

  nginx:
    image: nginx:alpine
    ports: ["3036:80"]
    volumes: ["./nginx.conf:/etc/nginx/nginx.conf:ro"]

volumes:
  pg_data:
```

---

## CI/CD

```yaml
name: Deploy SentimentLive
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VM_HOST }}
          username: ${{ secrets.VM_USER }}
          key: ${{ secrets.VM_SSH_KEY }}
          script: |
            cd /opt/sentimentlive && git pull origin main
            docker compose up -d --build
```

---

## Roadmap

### Phase 1 — MVP (Semaines 1–4)
- [ ] Pipeline DistilBERT + spaCy + VADER
- [ ] WebSocket FastAPI avec Redis Pub/Sub
- [ ] Dashboard Next.js temps réel avec Recharts
- [ ] Générateur mock + mode démo

### Phase 2 — Alertes (Semaines 5–8)
- [ ] Moteur d'alertes configurable (seuils, canaux)
- [ ] Batch upload CSV + export rapport
- [ ] Aspect-based sentiment (spaCy dependency parsing)
- [ ] Comparaison historique et tendances

### Phase 3 — Avancé (Semaines 9–12)
- [ ] Fine-tuning DistilBERT sur données domain-specific
- [ ] Support Twitter API v2 (optionnel)
- [ ] Topic modeling (LDA/BERTopic) sur clusters négatifs
- [ ] Rapport hebdomadaire auto-généré (PDF)

---

*Un produit [Wikolabs](https://wikolabs.com) — Intelligence artificielle appliquée aux métiers*
