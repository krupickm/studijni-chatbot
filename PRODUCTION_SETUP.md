# Produkční nasazení — checklist a konfigurace

> Tento dokument obsahuje všechny kroky potřebné pro nasazení do produkčního Kubernetes clusteru na e-INFRA CZ.

---

## Předpoklady

- Přístup k cílovému K8s clusteru (CERIT-SC, CESNET nebo VŠCHT vlastní)
- Namespace připravený v clusteru
- S3 objektové úložiště přístupné (CESNET S3 nebo podobné)
- DNS záznam nastavený (např. `rag.vscht.cz` nebo `chatbot.vscht.cz`)
- TLS certifikát pro doménu (Let's Encrypt nebo vlastní CA)

---

## 1. Kubernetes Access

```bash
# Získat kubeconfig od CIS týmu
# Poté ověřit připojení:
kubectl cluster-info
kubectl get nodes
```

---

## 2. Namespace a RBAC

```bash
# Vytvořit namespace (pokud ještě není)
kubectl create namespace rag

# Ověřit přístup
kubectl get pods -n rag
```

---

## 3. S3 Object Storage

Pro produkci použít CESNET S3 nebo organizovaný S3-compatible storage.

**Helm values konfigurace:**
```yaml
shared:
  secrets:
    s3:
      accessKey:
        value: "<CESNET_S3_ACCESS_KEY>"
      secretKey:
        value: "<CESNET_S3_SECRET_KEY>"
      bucketName:
        value: "vscht-rag-documents"
      endpoint:
        value: "https://s3.cesnet.cz"
      region:
        value: "eu-west-1"
```

---

## 4. LLM Provider Konfigurace

**Produkční LLM endpoint (e-INFRA CZ):**
```yaml
backend:
  secrets:
    stackitVllm:
      apiKey:
        value: "<PRODUCTION_LLM_API_KEY>"
      baseUrl:
        value: "https://llm.ai.e-infra.cz/v1/"
      model:
        value: "meta-llama/Llama-3.1-8B-Instruct"  # Nebo větší model (Qwen2.5-32B)
```

**Embedder konfigurace:**
```yaml
embedding:
  provider: "stackit"  # Nebo "ollama" pro lokální embedder
  secrets:
    stackitEmbedder:
      apiKey:
        value: "<EMBEDDER_API_KEY>"
      model:
        value: "Qwen3-Embedding-0.6B"  # Doporučeno pro češtinu
```

---

## 5. Langfuse Konfigurace

Pro produkci doporučujeme oddělenou Langfuse instanci nebo SaaS verzi.

**Helm values:**
```yaml
langfuse:
  langfuse:
    web:
      image:
        tag: "3.152.0"  # Nebo nejnovější stabilní
    
    additionalEnv:
      - name: LANGFUSE_INIT_ORG_ID
        value: "vscht-production"
      - name: LANGFUSE_INIT_PROJECT_ID
        value: "rag-chatbot-prod"
      - name: LANGFUSE_INIT_USER_EMAIL
        value: "admin@vscht.cz"
      - name: LANGFUSE_INIT_USER_NAME
        value: "VŠCHT Admin"
      - name: LANGFUSE_INIT_USER_PASSWORD
        valueFrom:
          secretKeyRef:
            name: "langfuse-admin-credentials"
            key: "password"
```

---

## 6. Basic Authentication

Pro produkci doporučujeme silnější autentizaci (OIDC/SAML přes Keycloak).

**Minimální konfigurace:**
```yaml
shared:
  secrets:
    basicAuth:
      user:
        value: "<PRODUKČNÍ_UŽIVATEL>"
      password:
        value: "<SILNÉ_HESLO_MIN_20_ZNAKŮ>"
```

---

## 7. Ingress Konfigurace

**Aktivovat nginx ingress controller:**
```bash
# Pokud není nainstalován
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

**Ingress host konfigurace:**
```yaml
backend:
  ingress:
    host:
      name: "rag.vscht.cz"  # Nebo zvolená doména

adminBackend:
  ingress:
    enabled: true
    host:
      name: "admin.rag.vscht.cz"
```

**TLS konfigurace:**
```yaml
shared:
  config:
    tls:
      enabled: true
    certManager:
      enabled: true
      clusterIssuer: "letsencrypt-production"
```

---

## 8. Helm Deploy

```bash
# 1. Přepnout se do adresáře projektu
cd /path/to/studijni-chatbot

# 2. Ověřit Helm chart
helm lint infrastructure/rag --values infrastructure/rag/values.yaml

# 3. Dry-run test
helm template rag infrastructure/rag \
  --namespace rag \
  --include-crds \
  --values infrastructure/rag/values.yaml \
  > /tmp/rag-manifests.yaml

# 4. Deploy (nebo přes CI/CD pipeline)
helm upgrade --install rag infrastructure/rag \
  --namespace rag \
  --create-namespace \
  --values infrastructure/rag/values.yaml \
  --wait \
  --timeout 10m
```

---

## 9. Ověření Nasazení

```bash
# Zkontrolovat všechny pody
kubectl get pods -n rag

# Očekávaný stav: Všechny pody Running
# - admin-backend-*
# - admin-frontend-*
# - backend-*
# - extractor-*
# - frontend-*
# - rag-clickhouse-shard0-* (3x)
# - rag-keydb-0
# - rag-langfuse-web-*
# - rag-langfuse-worker-*
# - rag-minio-*
# - rag-postgresql-0
# - rag-qdrant-0
# - rag-zookeeper-* (3x)

# Ověřit ingress
kubectl get ingress -n rag

# Testovat dostupnost služeb
curl -I https://rag.vscht.cz/api/health
curl -I https://admin.rag.vscht.cz
```

---

## 10. Monitoring a Logging

**Langfuse dashboard:**
- Přistoupit přes https://langfuse.rag.vscht.cz
- Login pomocí nastavených admin credentials
- Sledovat metriky: request count, latency, token usage

**K8s monitoring:**
```bash
# Pod logs
kubectl logs -n rag -l app=backend -f

# Resource usage
kubectl top pods -n rag
```

---

## 11. Backup Strategie

**Důležité komponenty k zálohování:**
1. **PostgreSQL** — uživatelská data, konfigury
2. **Qdrant** — vector embeddings (lze znovu generovat z dokumentů)
3. **MinIO/S3** — nahrané dokumenty (kritické!)
4. **ClickHouse** — analytická data, query history

**Doporučený plán:**
- Denní backup PostgreSQL (pg_dump + WAL archiving)
- Denní snapshot MinIO bucketu
- Týdenní backup Qdrant volumu
- ClickHouse může být replikován nebo znovu naplněn z source dat

---

## 12. Disaster Recovery

**Postup při selhání:**
1. Obnovit namespace z Helm release (`helm rollback`)
2. Obnovit PostgreSQL z backupu
3. Obnovit MinIO/S3 z backupu
4. Rebuild Qdrant indexu z nahraných dokumentů
5. Ověřit integritu dat přes Langfuse dashboard

---

## Kontakty a Eskalace

- **Technický vlastník:** Dibuszová (CIS)
- **Obsahový vlastník:** Fialová, Hladíková, Pátková (pedagogické oddělení)
- **Vývojář:** Martin Krupička (ÚOK)
- **Infrastruktura e-INFRA CZ:** support@e-infra.cz

---

## Verze Dokumentu

| Verze | Datum | Autor | Změny |
|-------|-------|-------|-------|
| 1.0 | 2026-05-25 | Martin Krupička | Initial version |
