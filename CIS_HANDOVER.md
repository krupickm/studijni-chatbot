# Předání projektu CIS — Technická specifikace

> Tento dokument slouží jako technický předávací materiál pro CIS (Centrum informačních služeb VŠCHT).

---

## 1. Architektura Projektu

### 1.1 Součásti Systému

```
┌─────────────────────────────────────────────────────────────┐
│                    VŠCHT RAG Chatbot                        │
├─────────────────────────────────────────────────────────────┤
│  Frontend (Vue.js) ────┐                                    │
│  Admin UI (Vue.js) ────┼──→ Backend (FastAPI) ──→ LLM      │
│                        │         │                         │
│  Langfuse ─────────────┘         ↓                         │
│                        │    Vector Store                   │
│  S3 Storage ←──────────┘       (Qdrant)                    │
│                                                                │
│  Databases: PostgreSQL, ClickHouse, KeyDB (Redis)            │
│  Message Queue: Zookeeper                                     │
│  Document Extractor: PDF/HTML parsing                        │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Technologický Stack

| Komponenta | Technologie | Verze | Poznámka |
|------------|-------------|-------|----------|
| Backend | FastAPI + Python | 3.13 | RAG pipeline API |
| Frontend | Vue.js | 3.x | Chat UI + Admin UI |
| Vector DB | Qdrant | Latest | Hybrid search (dense + sparse) |
| Relational DB | PostgreSQL | 17 | Uživatelská data, konfigury |
| Cache | KeyDB (Redis fork) | Latest | Session cache, rate limiting |
| Analytics | ClickHouse | Latest | Query logs, analytics |
| Message Queue | Zookeeper | Latest | Coordination |
| Object Storage | MinIO (dev) / S3 (prod) | Latest | Dokumenty, chunky |
| Observability | Langfuse | 3.152.0 | Tracing, feedback |
| Orchestration | Kubernetes | Latest | Produkční nasazení |
| LLM | vLLM (self-hosted) | Latest | e-INFRA CZ |

---

## 2. Environment Proměnné

### 2.1 Povinné Proměnné

```bash
# S3 Object Storage
S3_ACCESS_KEY_ID=<access-key>
S3_SECRET_ACCESS_KEY=<secret-key>

# Basic Authentication (pro dev; produkce doporučuje OIDC)
BASIC_AUTH_USER=<username>
BASIC_AUTH_PASSWORD=<password>

# Langfuse
LANGFUSE_PUBLIC_KEY=<public-key>
LANGFUSE_SECRET_KEY=<secret-key>

# LLM Provider (e-INFRA CZ)
STACKIT_VLLM_API_KEY=<api-key>
STACKIT_VLLM_BASE_URL=https://llm.ai.e-infra.cz/v1/
STACKIT_VLLM_MODEL=meta-llama/Llama-3.1-8B-Instruct

# Embedder Provider
STACKIT_EMBEDDER_API_KEY=<api-key>
STACKIT_EMBEDDER_MODEL=Qwen3-Embedding-0.6B
```

### 2.2 Volitelné Proměnné

```bash
# Langfuse initialization (pro automatické nastavení org/project)
LANGFUSE_INIT_ORG_ID=vscht-org
LANGFUSE_INIT_PROJECT_ID=rag-chatbot
LANGFUSE_INIT_USER_EMAIL=admin@vscht.cz
LANGFUSE_INIT_USER_PASSWORD=<admin-password>

# RAGAS evaluation (pro hodnocení kvality)
RAGAS_OPENAI_API_KEY=<openai-key>
```

---

## 3. Helm Chart Konfigurace

### 3.1 Hlavní Konfigurační Soubory

| Soubor | Popis |
|--------|-------|
| `infrastructure/rag/values.yaml` | Hlavní konfigurační soubor pro Helm |
| `.env` | Lokální development environment proměnné |
| `services/*/env.sh` | Service-specifické environment skripty |

### 3.2 Klíčové Sekce v values.yaml

```yaml
# LLM Configuration
backend:
  secrets:
    stackitVllm:
      apiKey:
        value: "<API-KEY>"
      baseUrl:
        value: "https://llm.ai.e-infra.cz/v1/"
      model:
        value: "meta-llama/Llama-3.1-8B-Instruct"

# Embedder Configuration
embedding:
  provider: "stackit"
  secrets:
    stackitEmbedder:
      apiKey:
        value: "<API-KEY>"
      model:
        value: "Qwen3-Embedding-0.6B"

# Ingress Configuration
backend:
  ingress:
    host:
      name: "rag.vscht.cz"  # Pro produkci

# Features
features:
  frontend:
    enabled: true
  minio:
    enabled: false  # Pro produkci použít externí S3
  langfuse:
    enabled: true
  qdrant:
    enabled: true
  keydb:
    enabled: true
  mcp:
    enabled: true
```

---

## 4. Retrieval Pipeline

### 4.1 Proces Retrievalu

```
User Query
    ↓
Language Detection (angličtina/čeština)
    ↓
Query Rephrasing (LLM)
    ↓
Vector Search (Qdrant - Hybrid Search)
    ├── Dense embeddings (STACKIT/Ollama)
    └── Sparse embeddings (FastEmbed BM25)
    ↓
Reranking (FlashRank - ms-marco-MultiBERT-L-12)
    ↓
Context Assembly
    ↓
Answer Generation (LLM)
    ↓
Citation Extraction
    ↓
Response
```

### 4.2 Konfigurace Retrievalu

```python
# vector_db_settings.py
retrieval_mode = RetrievalMode.HYBRID  # Dense + Sparse
top_k = 10  # Počet dokumentů k retrieval
rerank_enabled = True
rerank_model = "ms-marco-MultiBERT-L-12"
rerank_top_k = 5  # Počet dokumentů po reranking
```

---

## 5. Dependency Injection

Projekt používá `dependency_injector` library pro DI.

### 5.1 Hlavní Container

```python
# dependency_container.py
class DIContainer(containers.DeclarativeContainer):
    config = providers.Configuration()
    
    # LLM Provider
    llm_provider = providers.Factory[LLMProvider](
        StackitLLMProvider if config.llm.type == "stackit" else OllamaLLMProvider,
        api_key=config.llm.api_key,
        base_url=config.llm.base_url,
        model=config.llm.model
    )
    
    # Embedder Provider
    embedder_provider = providers.Factory[EmbedderProvider](
        StackitEmbedderProvider if config.embedder.type == "stackit" else OllamaEmbedderProvider,
        api_key=config.embedder.api_key,
        model=config.embedder.model
    )
```

### 5.2 Customizace LLM/Embedder

Pro změnu LLM provideru:
1. Upravit `values.yaml` → `backend.secrets.stackitVllm.*`
2. Případně doplnit nový provider v `libs/rag-core-lib/src/rag_core_lib/providers/`
3. Přidat selector v `dependency_container.py`

---

## 6. Services Detail

### 6.1 rag-backend (Chat API)

**Port:** 8080  
**Endpointy:**
- `POST /chat` — Odeslat zprávu a získat odpověď
- `GET /health` — Health check

**Odpovědnost:** Hlavní chat API, retrieval pipeline orchestration

### 6.2 admin-backend (Admin API)

**Port:** 8080  
**Endpointy:**
- `POST /documents/upload` — Nahrát dokument
- `DELETE /documents/{id}` — Smazat dokument
- `GET /documents` — Seznam dokumentů
- `POST /chunks/reindex` — Reindexovat chunky

**Odpovědnost:** Správa dokumentů, upload, chunking, reindexování

### 6.3 document-extractor

**Port:** 8080  
**Endpointy:**
- `POST /extract` — Extrahovat text z dokumentu

**Odpovědnost:** Parsing PDF, HTML, DOCX dokumentů (Docling, MarkItDown, Tesseract)

### 6.4 frontend (Chat UI)

**Port:** 8080  
**Technologie:** Vue.js 3

**Odpovědnost:** Uživatelské chat rozhraní

### 6.5 admin-frontend (Admin UI)

**Port:** 8080  
**Technologie:** Vue.js 3

**Odpovědnost:** Admin rozhraní pro správu dokumentů, monitorování

### 6.6 mcp-server (IDE Integrace)

**Port:** 8000  
**Odpovědnost:** Model Context Protocol server pro IDE pluginy

---

## 7. Infrastructure Components

### 7.1 Qdrant (Vector Database)

**Verze:** Latest  
**Storage:** Persistent volume  
**Replikace:** Single instance (pro dev); replikace pro produkci

**Konfigurace:**
- Hybrid search enabled
- Sparse encoder: FastEmbed (BM25-like)
- Dense encoder: STACKIT/Ollama

### 7.2 PostgreSQL

**Verze:** 17  
**Storage:** Persistent volume  
**Replikace:** Single instance (pro dev); replica pro produkci

**Databáze:**
- `langfuse` — Langfuse data
- `rag` — RAG aplikace data (pokud použito)

### 7.3 ClickHouse

**Verze:** Latest  
**Replication:** 3 shards (pro dev); upravit dle potřeby

**Použití:** Analytics, query logs, aggregation

### 7.4 KeyDB (Redis Fork)

**Použití:** Cache, session storage, rate limiting

### 7.5 MinIO (Local Dev Only)

**Použití:** S3-compatible storage pro lokální vývoj

**Produkce:** Nahradit CESNET S3 nebo jiným S3 providerem

### 7.6 Zookeeper

**Použití:** Coordination pro ClickHouse distributed tables

---

## 8. Monitoring a Observability

### 8.1 Langfuse

**Funkce:**
- Request tracing
- Token usage tracking
- Feedback collection (thumbs up/down)
- Query history
- Cost monitoring

**Dashboard přístup:**
- Dev: http://localhost:3030 (port-forward)
- Prod: https://langfuse.rag.vscht.cz (nebo zvolená doména)

**Admin credentials:**
- Email: `admin@vscht.cz`
- Password: Nastavit v Helm values

### 8.2 K8s Monitoring

**Doporučené nástroje:**
- Prometheus + Grafana (metriky)
- Loki (log aggregation)
- Jaeger (distributed tracing)

---

## 9. Bezpečnost

### 9.1 Autentizace

**Dev:** Basic Auth (jednoduché uživatelské jméno/heslo)  
**Produkce:** Doporučeno OIDC/SAML přes Keycloak nebo Azure AD

### 9.2 TLS/SSL

**Dev:** HTTP (lokální)  
**Produkce:** HTTPS s Let's Encrypt nebo vlastní CA certifikátem

### 9.3 Network Policies

Helm chart obsahuje základní network policies pro izolaci služeb.

### 9.4 Secrets Management

**Dev:** Helm values + `.env` soubor  
**Produkce:** Doporučeno External Secrets Operator nebo HashiCorp Vault

---

## 10. CI/CD Pipeline

### 10.1 Doporučený Setup

```
GitHub/GitLab Repository
    ↓
CI Pipeline (build + test)
    ↓
Build Docker images
    ↓
Push to container registry
    ↓
CD Pipeline (Helm deploy)
    ↓
Kubernetes cluster
```

### 10.2 GitOps Approach (Doporučeno)

- ArgoCD nebo Flux pro synchronizaci K8s state
- Separate repository pro K8s manifests
- Automatické deploy na merge do main branch

---

## 11. Known Issues a Workarounds

### 11.1 Windows Development

**Problém:** Git autocrlf převádí LF na CRLF, což láme shell skripty.

**Řešení:**
```bash
git config core.autocrlf false
sed -i 's/\r$//' services/frontend/env.sh
```

### 11.2 Langfuse Dirty Migration

**Problém:** Prisma migrations mohou zůstat v nekonzistentním stavu.

**Řešení:** Smazat namespace a restartovat (`kubectl delete namespace rag`)

### 11.3 Ingress DNS na Windows

**Problém:** `rag.localhost` se neresolve bez nginx ingress controlleru.

**Řešení:** Použít port-forward pro lokální vývoj, aktivovat ingress controller pro produkci.

---

## 12. Kontakty

| Role | Jméno | Kontakt |
|------|-------|---------|
| Vývojář/koordinátor | Martin Krupička | martin.krupicka@vscht.cz |
| Obsahový vlastník | Fialová, Hladíková, Pátková | pedagogicke@vscht.cz |
| Technický vlastník | Dibuszová (CIS) | cis@vscht.cz |
| Infrastruktura | e-INFRA CZ support | support@e-infra.cz |

---

## 13. Verze Dokumentu

| Verze | Datum | Autor | Změny |
|-------|-------|-------|-------|
| 1.0 | 2026-05-25 | Martin Krupička | Initial version |
