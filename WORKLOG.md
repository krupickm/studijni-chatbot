# WORKLOG — průběžný log práce na VŠCHT RAG projektu

> Tento dokument je paměť projektu mezi sessions. Claude Code zapisuje po každém kroku. Uživatel zapisuje, když dělá něco mimo Claude Code.

## Formát zápisu

Každý zápis má:
- **Datum a čas** (YYYY-MM-DD HH:MM)
- **Krok / fáze** (např. "Krok 2 — k3d+Tilt setup")
- **Kdo zapisuje** (Claude Code / Martin)
- **Co bylo uděláno** (stručně, ne kompletní log příkazů)
- **Co zůstalo otevřené** (co je nedořešené, na co se čeká)
- **Další krok** (jasné jedno-dvouvětné určení)

Nejnovější zápisy nahoře.

---

## Šablona pro nový zápis (zkopíruj a vyplň)

```
### YYYY-MM-DD HH:MM — Krok X — [krátký název]
**Kdo:** Claude Code / Martin
**Uděláno:**
- ...

**Otevřené:**
- ...

**Další krok:**
- ...
```

---

## Zápisy

### 2026-05-25 12:15 — Krok 2 — Lokální spuštění k3d + Tilt (v provozu)
**Kdo:** Claude Code
**Uděláno:**
- Vytvořen k3d cluster `rag-dev` s 1 server + 1 agent node
- Nainstalovány závislosti: k3d v5.7.4, Helm v3.21.0, Tilt v0.37.3
- Přepsán kubectl binary (původní z Docker Desktop byl nekompatibilní)
- Nastaveno Docker login pro stažení Helm charts (minio OCI chart)
- Upraveno `infrastructure/rag/values.yaml`:
  - LLM endpoint: `https://llm.ai.e-infra.cz/v1/`
  - LLM model: `meta-llama/Llama-3.1-8B-Instruct`
  - API klíč nastaven
- `.env` soubor vytvořen s hodnotami pro dev (LLM, Langfuse, S3, basic auth)
- `tilt up` spuštěn, probíhá initial build všech služeb
- Služby nasazované: langfuse, minio, keydb, zookeeper, clickhouse, postgresql, qdrant, backend, admin-backend, extractor, frontend, mcp-server, ollama

**Otevřené:**
- Čekání na dokončení Tilt buildu (sledovat na http://localhost:10350)
- Embedding model zatím na default STACKIT/Ollama (potřebujeme Qwen3-Embedding-0.6B nebo alternativu)
- Smoke test dokumentů ještě neproveden

**Další krok:**
- Po dokončení Tilt buildu ověřit zdravé pody (`kubectl get pods -n rag`)
- Přistoupit k admin UI a chat UI přes ingress (rag.localhost, admin.rag.localhost)
- Provést smoke test z Kroku 3

---

### 2026-05-25 14:30 — Krok 2 — Dokončení lokálního spuštění k3d + Tilt
**Kdo:** Claude Code
**Uděláno:**
- Po několika restartech (CRLF line endings v env.sh, chybějící langfuse-init-secrets, dirty migration state) jsou všechny služby běžící
- Všechny 17 pódů ve stavu Running:
  - **Infrastruktura:** qdrant, postgresql, minio, keydb, clickhouse (3x shard), zookeeper (3x), langfuse (web + worker)
  - **Aplikace:** backend (2/2 ready), admin-backend, frontend, admin-frontend, extractor
- Port-forward pro lokální přístup:
  - Frontend: `wsl kubectl port-forward -n rag svc/frontend 3000:8080` → http://localhost:3000
  - Admin: `wsl kubectl port-forward -n rag svc/admin-frontend 3001:8080` → http://localhost:3001
  - Backend: `wsl kubectl port-forward -n rag svc/backend 8000:8080` → http://localhost:8000
  - Langfuse: `wsl kubectl port-forward -n rag svc/rag-langfuse-web 3030:3000` → http://localhost:3030

**Otevřené:**
- Ingress DNS `rag.localhost` nefunguje na Windows bez nginx ingress controlleru v k3d
- Pro produkční nasazení bude potřeba aktivovat nginx ingress controller nebo použít LoadBalancer service

**Další krok:**
- Krok 3 z BOOTSTRAP.md: smoke test s nahráním dokumentů do admin UI

---

### 2026-05-25 11:30 — Krok 1 — Orientace v repu (dokončeno)
**Kdo:** Claude Code
**Uděláno:**
- Naklonován fork `stackitcloud/rag-template` do `studijni-chatbot/`
- Prozkoumána struktura repa a retrieval pipeline
- **Services:** rag-backend (chat API), admin-backend (upload/chunking), document-extractor (PDF/HTML parsing), mcp-server (IDE integrace), frontend (Vue chat + admin UI)
- **Retrieval pipeline:**
  - Chunking: `SemanticTextChunker` (LangChain) v `libs/admin-api-lib/src/admin_api_lib/impl/chunker/`
  - Embedding: STACKIT nebo Ollama (konfigurovatelné přes env)
  - Vector DB: Qdrant s hybrid search (dense + sparse/FastEmbed BM25)
  - Rerank: FlashRank (`ms-marco-MultiBERT-L-12`) v `libs/rag-core-api/src/rag_core_api/impl/reranking/flashrank_reranker.py`
- **Hybrid search:** ANO — `RetrievalMode.HYBRID` je výchozí, dense (STACKIT/Ollama) + sparse (FastEmbedSparse)
- **DI Container:** `dependency_injector` library, selector pattern pro LLM/embedder provider
- **Helm charts:** `infrastructure/rag/` s templates pro backend, admin, frontend, extractor, qdrant, langfuse, keydb, ollama
- **LLM config místa:**
  | Soubor | Řádek | Proměnná | Hodnota |
  |--------|-------|---------|---------|
  | `.env.template` | 39 | `STACKIT_VLLM_API_KEY` | placeholder |
  | `infrastructure/rag/values.yaml` | 210 | `RAG_CLASS_TYPE_LLM_TYPE` | "stackit" |
  | `infrastructure/rag/values.yaml` | 221-222 | `STACKIT_EMBEDDER_MODEL` | "intfloat/e5-mistral-7b-instruct" |
  | `infrastructure/rag/values.yaml` | 232-236 | Ollama settings | llama3.2:3b |
- **Embedder config místa:**
  | Soubor | Řádek | Proměnná | Hodnota |
  |--------|-------|---------|---------|
  | `.env.template` | 40 | `STACKIT_EMBEDDER_API_KEY` | placeholder |
  | `infrastructure/rag/values.yaml` | 219 | `EMBEDDER_CLASS_TYPE_EMBEDDER_TYPE` | "stackit" |
  | `infrastructure/rag/values.yaml` | 238-239 | `OLLAMA_EMBEDDER_MODEL` | "bge-m3" |
- **Rerank:** FlashRank v `composite_retriever.py:_arerank_pruning()`, model `ms-marco-MultiBERT-L-12`, enabled=true

**Otevřené:**
- Cílový K8s cluster (CERIT-SC vs. CESNET vs. VŠCHT vlastní)
- LLM endpoint pro testovací fázi (self-hosted vLLM URL vs. Anthropic API vs. Ollama)
- Konkrétní embedding model pro produkci (Qwen3-Embedding-0.6B navrženo v DECISIONS.md)

**Další krok:**
- Krok 2 z BOOTSTRAP.md: lokální spuštění přes k3d + Tilt (vyžaduje ověření závislostí a volbu LLM provideru pro dev)

---

### 2026-05-25 — Inicializace projektu
**Kdo:** Martin
**Uděláno:**
- Připraveny dokumenty pro Claude Code: CLAUDE.md, BOOTSTRAP.md, WORKLOG.md (tento), DECISIONS.md
- Rozhodnutí postavit RAG nad stackitcloud/rag-template (viz DECISIONS.md, D2)
- Fork repa vytvořen na GitHubu
- Lokální klon v `D:\devel\studijni\studijni-chatbot` (z Windows) = `/mnt/d/devel/studijni/studijni-chatbot` (z WSL2). Přejmenováno z původního `rag-template` na `studijni-chatbot`.
- Rozhodnutí pracovat v WSL2 (viz DECISIONS.md, D3)

**Otevřené:**
- Volba cílového K8s clusteru (Kubernetes@CERIT-SC vs. CESNET vs. jiné)
- Volba LLM endpointu pro testovací fázi (self-hosted vLLM vs. Anthropic API vs. Ollama)

**Další krok:**
- Krok 1 z BOOTSTRAP.md: orientace v repu (klon je hotový, fork existuje)
