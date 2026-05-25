# DECISIONS — architektonická a strategická rozhodnutí

> Tento dokument zachycuje rozhodnutí, která jsme udělali, a důvody. Cíl: chránit projekt před driftem v delším časovém horizontu a předáním na CIS.

## Formát zápisu

Každé rozhodnutí má:
- **ID** (D1, D2, ...)
- **Datum**
- **Otázka / kontext** — co jsme řešili
- **Rozhodnutí** — co jsme zvolili
- **Důvody** — proč právě tohle, krátce
- **Alternativy a proč ne** — co jsme zvažovali a zamítli
- **Cena rozhodnutí** — co tím obětujeme, co bude bolet
- **Reverzibilita** — jak těžké je rozhodnutí změnit později

Nejnovější rozhodnutí nahoru, číslování pokračuje vzestupně.

---

## D3 — Vývojové prostředí: WSL2

**Datum:** 2026-05-25
**Otázka:** Vyvíjet nativně ve Windows, nebo v WSL2?
**Rozhodnutí:** Veškerá práce s Dockerem, k3d, Tiltem a Pythonem v WSL2 (Ubuntu). Pracovní adresář `/mnt/d/devel/studijni/studijni-chatbot`.
**Důvody:**
- Docker Desktop pro Windows funguje, ale Docker Engine v WSL2 je rychlejší a stabilnější pro stack s vícero kontejnery
- Tilt + k3d je primárně testovaný na Linux/macOS, na Windows nativně má známé problémy
- Shell skripty v upstream šabloně předpokládají bash/zsh, ne PowerShell
- Konzistence s produkčním prostředím (K8s běží na Linuxu)

**Alternativy a proč ne:**
- **Nativní Windows + Docker Desktop** — funguje, ale pomalejší file I/O, problémy s file watchingem v Tiltu, nutnost konvertovat skripty pro PowerShell
- **Linux VM (VirtualBox/VMware)** — overhead správy VM, slabší integrace s VS Code

**Cena rozhodnutí:**
- Soubory leží na Windows filesystému (`/mnt/d/...`), což má pomalejší I/O než nativní WSL2 filesystem (`~/`). Pokud narazíme na pomalé Tilt buildy, fallback je přesun do `~/projects/studijni-chatbot` uvnitř WSL2 (vyžaduje pak konzistentní práci z WSL2, ne přímou editaci z Windows GUI).
- Editace souborů z Windows GUI (VS Code) přes `\\wsl$\Ubuntu\...` funguje, ale přes `/mnt/d/...` je přímější

**Reverzibilita:** Vysoká. Repo je obyčejný git fork, lze přesunout kamkoli.

---

## D2 — Hlavní šablona projektu

**Datum:** 2026-05-25
**Otázka:** Jakou základní šablonu / framework použít pro RAG chatbot?
**Rozhodnutí:** `stackitcloud/rag-template` (Apache 2.0, FastAPI + Vue + Qdrant + Langfuse + Helm).
**Důvody:**
- Apache 2.0 bez paywallu (na rozdíl od Onyx CE/EE rozdělení)
- Qdrant místo Vespy → řádově nižší memory footprint pro malý korpus
- Velikost umožňuje porozumění a customizaci (~200 commitů, jedna firma)
- Dependency-injector pattern poskytuje explicitní hooky pro náhradu komponent
- Helm chart pro K8s je oficiální, K8s-native
- Langfuse pokrývá observability/feedback části, které jsou jinde za paywallem
- Politicky čisté: "integrovali jsme open-source šablonu" nezní jako "Krupička sám něco upletl" ani "koupili jsme Onyx"

**Alternativy a proč ne:**
- **Onyx (dříve Danswer)** — admin UI s analytics, query history a custom code za paywallem (EE). Vespa žere paměť (8+ GB minimum), pro malý korpus přemrštěné. Vendor lobby může argumentovat "jen jste zabalili jiný produkt".
- **Kotaemon (Cinnamon)** — slabší admin features, žádný dashboard, žádný feedback loop. Gradio UI méně přizpůsobitelné.
- **R2R (SciPhi)** — výborný retrieval framework, ale primárně API + admin dashboard; chat UI pro koncové uživatele by se musel doplnit. Self-host je second-class oproti SciPhi Cloud.
- **vstorm-co/full-stack-ai-agent-template** — generator scaffoldu, dostali bychom čistý kód, ale žádné dashboardy hotové. Pro 6týdenní MVP moc holé.
- **Vlastní řešení od nuly (Qdrant + FastAPI + Streamlit + Langfuse)** — plná kontrola, ale 3–4 týdny vývoje místo 1 týdne konfigurace.

**Cena rozhodnutí:**
- Vue frontend (ne Next.js/React) — pokud později budeme chtít hluboké UI customizace, je to drobná překážka
- Závislost na STACKIT-specifických hookech pro LLM (nutná úprava, ne fork)
- Menší komunita než Onyx → pokud projekt odepíše, jsme sami
- Hybrid search (BM25+vector) v Qdrantu může vyžadovat dopsat — ověřit v kroku 1

**Reverzibilita:** Střední. Customizace kódu se zachová, vector store data lze migrovat, ale frontend a admin UI by se přepisovaly.

---

## D1 — Architektonický směr: koupit / forknout / postavit

**Datum:** 2026-05-25
**Otázka:** Postavit RAG od nuly, vzít OSS šablonu a customizovat, nebo nasadit hotový produkt (Onyx)?
**Rozhodnutí:** Forknout open-source šablonu a customizovat (Varianta B z přehledové diskuse).
**Důvody:**
- Náš scope je doménový (4-vrstvá autorita, layer-1 grounding pro kritická fakta, 6-vrstvý security model). To není běžný enterprise search — hotový produkt nesedí 1:1.
- Onyx Community Edition nemá analytics dashboard, query history s auditem, custom PII filter — to vše je v paywall verzi
- Onyx Vespa žere 8+ GB RAM jen za to že běží (pro náš korpus ~200 dokumentů přemrštěné)
- Vlastní řešení od nuly je nákladnější a politicky horší (Krupička sám "něco upletl")
- Forknutí šablony je kompromis: hotový základ + flexibilita customizace

**Alternativy a proč ne:**
- **Onyx CE jako produkt** — paywall na klíčové admin features, Vespa memory hog, vendor lobby riziko
- **Vlastní řešení (Qdrant + FastAPI + Streamlit)** — 3–4 týdny extra vývoje, slabší politický artefakt
- **SaaS (Glean, Cohere North)** — out of scope pro veřejné akademické VŠCHT prostředí, GDPR komplikace

**Cena rozhodnutí:**
- Závislost na upstream šabloně (upgrady, breaking changes)
- Customizace kódu znamená divergence od upstream → časem těžší pull nových verzí
- Část funkčnosti, kterou potřebujeme (4-vrstvá logika), si stejně musíme dopsat

**Reverzibilita:** Vysoká v rané fázi (před customizací), klesající v čase.

---

## Předpokládaná budoucí rozhodnutí (rezervovaná místa)

Tato rozhodnutí přijdou v dalších krocích. Místa rezervovaná, aby šlo snadno doplnit.

### D4 — LLM endpoint pro produkční nasazení
**Status:** Nerozhodnuto. Předpoklad: self-hosted vLLM s českým-friendly modelem (Qwen2.5-32B nebo větší). Konkrétní model a infrastruktura — řeší se s CIS.

### D5 — Embedding model
**Status:** Předběžně Qwen3-Embedding-0.6B (důvod: kvalita pro češtinu, rozumná velikost pro self-host). Potvrdíme po smoke testu kvality v kroku 4.

### D6 — Cílová K8s platforma
**Status:** Nerozhodnuto. Možnosti: Kubernetes@CERIT-SC (Brno), Kubernetes@CESNET, vlastní VŠCHT K8s. Závisí na dohodě s Dibuszovou (CIS).

### D7 — Strategie pro hybrid search
**Status:** Potvrzeno — šablona má hybrid search (dense + sparse/FastEmbed BM25) přes Qdrant `RetrievalMode.HYBRID`. Žádná akce potřena.

### D8 — Strategie pro 4-vrstvou autoritativní logiku
**Status:** Závisí na architektonické skice z kroku 7. Rozhodneme: implementovat v MVP, nebo odsunout do verze 2.

### D9 — GDPR / DPIA proces
**Status:** Otevřená otázka A2 z OPEN_QUESTIONS.md. Eskalace na Dibuszovou. Mimo rozsah Claude Code.

### D10 — Veřejnost MVP
**Status:** Předběžně potvrzeno — anonymní veřejný přístup bez autentizace, bez personalizace na fakultu+program. Personalizace je verze 2.

---

## D4 — Hybrid search v šabloně

**Datum:** 2026-05-25
**Otázka:** Má šablona `stackitcloud/rag-template` built-in hybrid search (BM25 + dense), nebo musíme dopsat?
**Rozhodnutí:** Šablona má hybrid search připravený — `RetrievalMode.HYBRID` je výchozí v `vector_db_settings.py`, používá `FastEmbedSparse` pro BM25-like sparse embeddings + dense od STACKIT/Ollama.
**Důvody:**
- Qdrant client v `dependency_container.py:108-123` injektuje both dense embedder a sparse_embedder do `QdrantVectorStore`
- `retrieval_mode=RetrievalMode.HYBRID` je explicitně nastaveno ve `vector_db_settings.py:32`
- Žádná customizace potřena pro základní hybrid search

**Alternativy a proč ne:**
- **Pouze dense embeddings** — horší retrieval pro přesné termíny, ztrácíme BM25 benefit
- **Custom BM25 implementace** — zbytečné, šablona už to má

**Cena rozhodnutí:**
- FastEmbed sparse model je přednastavený, pokud chceme jiný BM25 variantu, musíme to přenastavit
- Hybrid search může být pomalejší než čistě dense (ale pro náš malý korpus irrelevantní)

**Reverzibilita:** Vysoká — stačí změnit config v Helm values.

---

## D5 — Reranker ve retrieval pipeline

**Datum:** 2026-05-25
**Otázka:** Jaký reranker použít a kde je v pipeline?
**Rozhodnutí:** FlashRank (`ms-marco-MultiBERT-L-12`) běží lokálně jako post-retrieval reranking v `CompositeRetriever._arerank_pruning()`.
**Důvody:**
- Běžící lokálně, není to API call — nižší latency
- Model se stáhne při prvním použití
- Konfigurovatelné přes `RERANKER_ENABLED`, `RERANKER_K_DOCUMENTS`, `RERANKER_MIN_RELEVANCE_SCORE` v Helm values

**Alternativy a proč ne:**
- **Cross-encoder reranker z HuggingFace** — podobná kvalita, ale větší model
- **ColBERT** — lepší kvalita, ale výrazně vyšší compute cost
- **Žádný reranker** — horší precision v answer generation

**Cena rozhodnutí:**
- FlashRank model ~500MB, stáhne se automaticky
- Extra latency ~50-100ms pro reranking fázi

**Reverzibilita:** Vysoká — lze přepnout configem nebo vypnout.

---

## D6 — Lokální development setup (k3d + Tilt)

**Datum:** 2026-05-25
**Otázka:** Jak nastavit lokální development prostředí pro vývoj na Windows?
**Rozhodnutí:** k3d cluster s vlastním OCI registry + Tilt pro K8s development. Vše v WSL2.
**Důvody:**
- k3d je lehký Kubernetes pro lokální vývoj, kompatibilní s Helm charts
- Tilt poskytuje rychlý feedback loop s automatickým rebuildem při změnách kódu
- Vlastní registry (k3d-myregistry:5500) nutný kvůli ghcr.io auth problémům
- WSL2 poskytuje Linux-like prostředí potřebné pro shell skripty v upstream šabloně

**Proces nasazení:**
1. `k3d cluster create rag-dev --api-port 6550 --agents 1 --servers 1 --registry-create k3d-myregistry:5500 --registry-use k3d-myregistry:5500`
2. Upravit `Tiltfile`: změnit registry z `ghcr.io/stackitcloud/rag-template` na `k3d-myregistry:5500/rag-template`
3. Upravit `infrastructure/rag/values.yaml`: nastavit LLM endpoint, API klíče, image repository
4. Vytvořit `.env` soubor s lokálními proměnnými
5. `tilt up` pro spuštění všech služeb

**Problémy a řešení:**
- **CRLF line endings v env.sh**: Git na Windows převádí LF na CRLF, řešeno `sed -i 's/\r$//' services/frontend/env.sh`. Pro trvalé řešení nastavit `git config core.autocrlf false`
- **Chybějící secrets**: Langfuse vyžaduje `langfuse-init-secrets` secret před startem webu. Vytvořit manuálně: `kubectl create secret generic langfuse-init-secrets -n rag --from-literal=...`
- **Dirty migration state**: Prisma migrations mohou zůstat v nekonzistentním stavu. Řešení: smazat namespace a začít znovu (`kubectl delete namespace rag`), nebo opravit přímo v DB
- **Ingress DNS nefunguje**: k3d nemá ve výchozím nastavení nginx ingress controller. Pro lokální test použít port-forward: `kubectl port-forward -n rag svc/frontend 3000:8080`

**Pro přístup ke službám (port-forward):**
```bash
wsl kubectl port-forward -n rag svc/frontend 3000:8080       # Frontend
wsl kubectl port-forward -n rag svc/admin-frontend 3001:8080  # Admin UI
wsl kubectl port-forward -n rag svc/backend 8000:8080         # Backend API
wsl kubectl port-forward -n rag svc/rag-langfuse-web 3030:3000 # Langfuse
```

**Alternativy a proč ne:**
- **Nativní Windows + Docker Desktop**: Pomalejší file I/O, problémy s file watchingem v Tiltu
- **Linux VM**: Vyšší overhead, horší integrace s VS Code
- **Minikube místo k3d**: k3d je lehčí a lépe integrovaný s Docker Engine

**Cena rozhodnutí:**
- Soubory na `/mnt/d/...` mají pomalejší I/O než nativní WSL filesystem
- Ingress DNS nefunguje bez dodatečné konfigurace
- Nutnost udržovat více komponent (k3d, registry, Tilt)

**Reverzibilita:** Vysoká — vše je lokální, lze kdykoli smazat a začít znovu. Pro produkci se používá jiný cluster.

---

## D7 — Self-hosted LLM na e-INFRA CZ

**Datum:** 2026-05-25
**Otázka:** Který LLM provider použít pro testovací i produkční fázi?
**Rozhodnutí:** Self-hosted vLLM na e-INFRA CZ (`https://llm.ai.e-infra.cz/v1/`) s modelem `meta-llama/Llama-3.1-8B-Instruct`.
**Důvody:**
- Politicky neutrální (vŠCHT vlastní infrastruktura)
- Žádné externí API poplatky
- GDPR compliant (data zůstávají v ČR/EU)
- Konfigurovatelné přes `STACKIT_VLLM_BASE_URL` a `STACKIT_VLLM_MODEL` environment proměnné

**Konfigurace:**
- `STACKIT_VLLM_API_KEY` nastaveno v `.env` a Helm values
- `STACKIT_VLLM_BASE_URL=https://llm.ai.e-infra.cz/v1/`
- `STACKIT_VLLM_MODEL=meta-llama/Llama-3.1-8B-Instruct`

**Alternativy a proč ne:**
- **Anthropic API**: Externí poskytovatel, vyšší náklady v delším horizontu
- **Ollama lokálně**: Nedostatečná kapacita pro produkční provoz
- **OpenAI API**: Data mimo EU, GDPR komplikace

**Cena rozhodnutí:**
- Závislost na dostupnosti e-INFRA CZ infrastruktruy
- Potřeba spravovat API klíče

**Reverzibilita:** Vysoká — stačí změnit environment proměnné v Helm values.

---

## D8 — Port-forward vs Ingress pro lokální access

**Datum:** 2026-05-25
**Otázka:** Jak přistupovat ke službám lokálně?
**Rozhodnutí:** Pro lokální development použít port-forward. Pro produkční nasazení použít Ingress s nginx controllerem.
**Důvody:**
- Port-forward funguje okamžitě bez dodatečné konfigurace
- Ingress vyžaduje aktivaci nginx ingress controlleru v k3d
- Pro produkci je Ingress správný způsob (DNS, TLS, load balancing)

**Produkční konfigurace:**
- Aktivovat nginx ingress controller v K8s clusteru
- Nastavit DNS záznamy pro `rag.localhost` (nebo doménu organizace)
- Nastavit TLS certifikáty (Let's Encrypt nebo vlastní CA)

**Alternativy a proč ne:**
- **LoadBalancer service**: Každá služba dostane vlastní external IP, méně elegantní než single ingress
- **NodePort exposure**: Expozuje porty na všech nodech, bezpečnostní riziko

**Cena rozhodnutí:**
- Port-forward blokuje terminál (řešení: spustit v pozadí s `&`)
- Musíme udržovat více port-forwardů pro různé služby

**Reverzibilita:** Vysoká — pouze lokální přístupový způsob, nic ovlivňujícího kód.
