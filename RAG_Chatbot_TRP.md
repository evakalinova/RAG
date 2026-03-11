# RAG Chatbot — Technicky-realizační plán

**Projekt:** AI RAG Chatbot nad interními směrnicemi společnosti
**Verze dokumentu:** v1.1
**Datum:** Březen 2026
**Autor:** Senior Software Architect / Backend Engineer / DevOps Engineer
**Stav:** Draft

---

## 1. Přehled architektury

### Princip RAG

RAG (Retrieval-Augmented Generation) je architektura, která kombinuje vektorové vyhledávání s generativním jazykovým modelem. Systém nepředává modelu celé dokumenty — místo toho při každém dotazu vyhledá v Qdrantu nejrelevantnější textové bloky (chunky) a pouze ty předá modelu jako kontext. Model odpovídá **výhradně** z tohoto kontextu, nikoli z vlastních trénovacích dat. Výsledkem je přesná, citovatelná odpověď navázaná na konkrétní firemní dokument.

### Tok dat

```
┌─────────────────────────────────────────────────────┐
│             SharePoint (PDF soubory)                │
│  Dokumenty ze systému Pinya jsou manuálně nahrány   │
│  na SharePoint jako PDF před prvním ingestem        │
└───────────────────────┬─────────────────────────────┘
                        │  MS Graph API
                        ▼
┌─────────────────────────────────────────────────────┐
│                  INGEST PIPELINE                    │
│  (asynchronně — Celery worker)                      │
│                                                     │
│  01 sharepoint.py   → stažení PDF (SHA-256 diff)   │
│  02 pdf_parser.py   → Markdown extrakce            │
│  03 cleaner.py      → RegEx čištění                │
│  04 metadata_agent  → extrakce metadat (Mistral)   │
│  05 summary_agent   → shrnutí dokumentu (Mistral)  │
│  06 chunker.py      → rozdělení na bloky           │
│  07 chunker.py      → Document Builder             │
│  08 embedder.py     → převod na vektory            │
│  09 qdrant_store.py → uložení + SQLite záznam      │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│            Qdrant (vektory + metadata)              │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│                  CHAT PIPELINE                      │
│  (synchronně — FastAPI endpoint /chat)              │
│                                                     │
│  01 Query Embedder    → převod dotazu na vektor    │
│  02 Retrieval         → top 10 chunků (vector)     │
│     [2. it.] Hybrid   → BM25 + vector (rozšíření) │
│  03 [2. it.] Reranker → seřazení top 3–5          │
│  04 Answer Agent      → generování odpovědi        │
│  05 Citation Builder  → sestavení citací           │
│  06 [2. it.] Memory   → konverzační kontext        │
│  07 Audit log         → zápis do SQLite            │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│        Uživatel (odpověď + citace zdrojů)           │
└─────────────────────────────────────────────────────┘
```

### ⚠️ Kritické pravidlo embeddingů

Embedding model **musí být identický** při ingestu dokumentů i při zpracování dotazů uživatele. Oba kroky musí používat `nomic-embed-text` načtený z proměnné `OLLAMA_EMBED_MODEL` v `.env` — nikdy nezadrátovat název modelu natvrdo v kódu.

**Co se stane při porušení:** Různé embedding modely produkují vektory v odlišných vektorových prostorech různých dimenzí. Dotaz embeddovaný jiným modelem než dokumenty vrátí z Qdrantu zcela irelevantní nebo prázdné výsledky. Systém přitom nevyhodí žádnou chybu — bude fungovat, ale odpovědi budou nesmyslné. Tato chyba je extrémně těžko diagnostikovatelná.

### Role infrastrukturních komponent

| Komponenta | Role v systému |
|---|---|
| **Qdrant** | Vektorová databáze. Ukládá embedding vektory chunků spolu s metadaty (název souboru, číslo stránky, SharePoint URL, oddělení). Při dotazu vrátí top-N nejpodobnějších chunků pomocí kosinové podobnosti v milisekundách. |
| **Redis** | Message broker pro Celery. FastAPI zapíše ingest úlohu do Redisu a okamžitě vrátí odpověď. Celery worker si úlohu vyzvedne a zpracuje PDF na pozadí — FastAPI neblokuje a zůstává dostupné pro chat dotazy. |
| **Ollama** | Lokální runtime pro AI modely. Spouští Mistral 7B (generování odpovědí, extrakce metadat, shrnutí) a nomic-embed-text (embeddingy) výhradně na firemním serveru. Žádná data neopouštějí infrastrukturu, žádné API poplatky, žádná závislost na internetu. |
| **FastAPI** | Vstupní HTTP brána. Přijímá požadavky na `POST /ingest/sync` (IT admin) a `POST /chat` (uživatel). Validuje vstupy přes Pydantic, deleguje ingest do Celery, spouští chat pipeline synchronně. |
| **Celery** | Asynchronní worker. Spouštěný v samostatném procesu. Zpracovává celou ingest pipeline (kroky 01–09) na pozadí — stažení PDF ze SharePointu, parsování, chunking, embedding, indexace do Qdrantu. |

---

## 2. Sumarizace technologií a modelů

### Tabulka A — AI modely

| Model | Typ | Účel v systému | HW požadavky | Iterace |
|---|---|---|---|---|
| `mistral:latest` | LLM | Generuje odpovědi uživatelům; extrahuje metadata z dokumentů (metadata_agent); generuje shrnutí (summary_agent) | GPU 8 GB VRAM (doporučeno); CPU: 30–60 s/odpověď | 1. |
| `nomic-embed-text` | Embedding model | Převádí text chunků i dotazů na vektory — ⚠️ musí být stejný při ingestu i dotazech | CPU stačí, nenáročný na paměť | 1. |
| `BGE-Reranker-v2-m3` | Reranker | Přeřadí top 10 chunků z Qdrantu podle skutečné relevance k dotazu před předáním do Answer Agenta | CPU; přes knihovnu `FlagEmbedding` | 2. |
| `mixtral:latest` | LLM (upgrade) | Silnější model pro lepší zpracování češtiny a složitějších dotazů; výměna za Mistral = změna jednoho řádku v `.env` | GPU 24 GB VRAM | 2. (volitelný) |

### Tabulka B — Frameworky a knihovny

| Knihovna | Účel | Iterace |
|---|---|---|
| `fastapi` | REST API framework | 1. |
| `uvicorn[standard]` | ASGI server pro FastAPI | 1. |
| `celery` | Fronta asynchronních úloh — ingest pipeline | 1. |
| `redis` | Python klient pro Redis (Celery broker) | 1. |
| `pymupdf4llm` | Lokální extrakce textu z PDF; zachovává tabulky a nadpisy jako Markdown | 1. |
| `langchain` | Orchestrace pipeline, prompt management, `Document` objekt, `RecursiveCharacterTextSplitter` | 1. |
| `langchain-ollama` | LangChain integrace pro Ollama — `ChatOllama`, `OllamaEmbeddings` | 1. |
| `langchain-qdrant` | LangChain integrace pro Qdrant — `QdrantVectorStore` | 1. |
| `qdrant-client` | Přímý Python klient pro Qdrant REST API | 1. |
| `office365-rest-python-client` | SharePoint konektor přes MS Graph API | 1. |
| `requests` | HTTP klient | 1. |
| `pydantic` | Validace dat, Pydantic modely pro metadata a API schémata | 1. |
| `pydantic-settings` | `BaseSettings` pro načtení a validaci `.env` při startu aplikace | 1. |
| `python-dotenv` | Načítání `.env` souboru do prostředí | 1. |
| `hashlib` | SHA-256 hash pro detekci změn souborů — stdlib, bez instalace | 1. |
| `re` | RegEx čištění textu — stdlib, bez instalace | 1. |
| `sqlite3` | Audit log — stdlib, bez instalace | 1. |
| `os`, `pathlib` | Práce se soubory a cestami — stdlib, bez instalace | 1. |
| `FlagEmbedding` | BGE-Reranker-v2-m3 reranking | 2. |
| `python-jose` | JWT tokeny pro autentizaci | 2. |
| `passlib` | Hashování hesel | 2. |
| `fastapi.security` | OAuth2 schéma pro FastAPI | 2. |
| `slowapi` | Rate limiting middleware | 2. |
| `httpx` | Async HTTP klient pro health check endpointy závislostí | 2. |

### Tabulka C — Infrastruktura

| Komponenta | Port | Způsob spuštění | Iterace |
|---|---|---|---|
| Docker Desktop | — | Instalace z docker.com; `docker --version` pro ověření | 1. |
| Qdrant | 6333 | `docker compose up -d` (service `qdrant` v `docker-compose.yml`) | 1. |
| Redis | 6379 | `docker compose up -d` (service `redis` v `docker-compose.yml`) | 1. |
| Ollama | 11434 | `ollama serve` — na Windows/macOS běží automaticky po instalaci jako služba | 1. |
| FastAPI / uvicorn | 8000 | `uvicorn app.main:app --reload` — terminál 3 | 1. |
| Celery Worker | — | `celery -A app.core.celery_app worker --loglevel=info` — terminál 4 (samostatný) | 1. |

---

## 3. Hardwarové a systémové požadavky

| Komponenta | Minimální požadavek | Doporučené | Poznámka |
|---|---|---|---|
| Operační systém | Windows 10+, Ubuntu 20.04+, macOS 12+ | Ubuntu 22.04 LTS nebo Windows 11 | Docker Desktop vyžaduje WSL2 na Windows |
| RAM | 16 GB | 32 GB | Mistral 7B na CPU vyžaduje ~8 GB RAM jen pro model |
| GPU VRAM | — (CPU postačuje) | NVIDIA 8 GB VRAM (RTX 3070 nebo lepší) | S GPU: odpověď 5–10 s; bez GPU: 30–60 s |
| Disk | 30 GB volného místa | 50 GB+ | Ollama modely ~8 GB, Qdrant data, stažené PDF soubory |
| Python | 3.11+ | 3.11.x stable | `pydantic-settings` v2 vyžaduje Python 3.11+ |
| Docker | 24.0+ | nejnovější stable | Kontejnery pro Qdrant a Redis |
| CUDA (volitelné) | CUDA 11.8+ | CUDA 12.x | Pouze pro NVIDIA GPU — Ollama detekuje automaticky |

**Varianta bez GPU (CPU only):**
- Mistral 7B inference: 30–60 sekund na odpověď
- nomic-embed-text embedding: 1–3 sekundy na chunk
- Ingest 100 dokumentů: odhadem 2–4 hodiny

**Varianta s GPU (NVIDIA 8 GB VRAM+):**
- Mistral 7B inference: 5–10 sekund na odpověď
- nomic-embed-text embedding: < 1 sekunda na chunk
- Ingest 100 dokumentů: odhadem 20–40 minut

---

## 4. Instalace prostředí — krok po kroku

> ⚠️ **Pořadí instalace je závazné.** Docker Desktop musí být nainstalován jako první — Qdrant a Redis na něm závisejí.

### 4.1 Docker Desktop

Stáhnout z: [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)

> **Pozor:** Na Windows je nutné po instalaci povolit WSL2 integraci: Docker Desktop → Settings → Resources → WSL Integration → zapnout pro vaši distribuci.

```bash
docker --version
```

```bash
docker compose version
```

Očekávaný výstup: verze Docker Engine 24+ a Docker Compose 2+.

---

### 4.2 Python 3.11+

**Windows:**
```bash
winget install Python.Python.3.11
```

**macOS:**
```bash
brew install python@3.11
```

**Linux (Ubuntu):**
```bash
sudo apt update && sudo apt install python3.11 python3.11-venv python3-pip -y
```

Ověření verze:
```bash
python --version
```

Vytvoření virtualenv:
```bash
python -m venv venv
```

Aktivace — Windows:
```bash
venv\Scripts\activate
```

Aktivace — macOS / Linux:
```bash
source venv/bin/activate
```

> **Pozor:** Virtualenv musí být aktivní při každé práci s projektem. Po aktivaci se zobrazí `(venv)` před příkazovým řádkem.

---

### 4.3 Git a VS Code

**Git — Windows:**
```bash
winget install Git.Git
```

**Git — macOS:**
```bash
brew install git
```

**Git — Linux:**
```bash
sudo apt install git -y
```

Ověření:
```bash
git --version
```

**VS Code — Windows:**
```bash
winget install Microsoft.VisualStudioCode
```

**VS Code — macOS:**
```bash
brew install --cask visual-studio-code
```

Doporučená rozšíření VS Code: `Python`, `Pylance`, `REST Client`, `SQLite Viewer`, `YAML`.

---

### 4.4 Ollama

**Windows:**
```bash
winget install Ollama.Ollama
```

**macOS:**
```bash
brew install ollama
```

**Linux:**
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Spuštění Ollama serveru (Linux/macOS — na Windows běží automaticky jako služba po instalaci):
```bash
ollama serve
```

Stažení hlavního LLM modelu:
```bash
ollama pull mistral
```

Stažení embedding modelu:
```bash
ollama pull nomic-embed-text
```

Ověření dostupnosti na portu 11434:
```bash
curl http://localhost:11434
```

Očekávaný výstup: `Ollama is running`

Výpis nainstalovaných modelů:
```bash
ollama list
```

---

### 4.5 Struktura projektu

Vytvoření celé složkové struktury:

```bash
mkdir -p rag-assistant/app/core
mkdir -p rag-assistant/app/utils
mkdir -p rag-assistant/data/raw
```

```bash
cd rag-assistant
```

Vytvoření všech souborů:
```bash
touch docker-compose.yml requirements.txt .env .env.example .gitignore
```

```bash
touch app/__init__.py app/main.py
```

```bash
touch app/core/__init__.py app/core/ingest.py app/core/rag_chain.py app/core/celery_app.py
```

```bash
touch app/utils/__init__.py \
      app/utils/sharepoint.py \
      app/utils/pdf_parser.py \
      app/utils/cleaner.py \
      app/utils/metadata_agent.py \
      app/utils/summary_agent.py \
      app/utils/chunker.py \
      app/utils/embedder.py \
      app/utils/reranker.py \
      app/utils/qdrant_store.py
```

Ověření výsledné struktury:
```bash
find . -type f | sort
```

Očekávaný výstup:
```
./.env
./.env.example
./.gitignore
./app/__init__.py
./app/core/__init__.py
./app/core/celery_app.py
./app/core/ingest.py
./app/core/rag_chain.py
./app/main.py
./app/utils/__init__.py
./app/utils/chunker.py
./app/utils/cleaner.py
./app/utils/embedder.py
./app/utils/metadata_agent.py
./app/utils/pdf_parser.py
./app/utils/qdrant_store.py
./app/utils/reranker.py
./app/utils/sharepoint.py
./app/utils/summary_agent.py
./docker-compose.yml
./requirements.txt
```

---

### 4.6 pip závislosti

Kompletní `requirements.txt` pro 1. iteraci:

```text
# API framework
fastapi==0.115.0
uvicorn[standard]==0.30.6

# Asynchronní fronta úloh
celery==5.4.0
redis==5.0.8

# PDF parsing — lokální, zachovává tabulky
pymupdf4llm==0.0.17

# LangChain orchestrace
langchain==0.3.0
langchain-community==0.3.0
langchain-ollama==0.2.0
langchain-qdrant==0.1.4

# Vektorová databáze
qdrant-client==1.11.0

# SharePoint konektor
office365-rest-python-client==2.5.12
requests==2.32.3

# Validace a konfigurace
pydantic==2.9.0
pydantic-settings==2.5.2
python-dotenv==1.0.1
```

Instalace (virtualenv musí být aktivní):
```bash
pip install -r requirements.txt
```

Ověření instalace:
```bash
pip list | grep -E "fastapi|langchain|qdrant|celery"
```

---

### 4.7 Docker Compose

Kompletní `docker-compose.yml`:

```yaml
version: "3.9"

services:
  qdrant:
    image: qdrant/qdrant:latest
    container_name: qdrant
    ports:
      - "127.0.0.1:6333:6333"   # Pouze localhost — nesmí být dostupné z internetu
    volumes:
      - qdrant_data:/qdrant/storage
    restart: unless-stopped

  redis:
    image: redis:7
    container_name: redis
    ports:
      - "127.0.0.1:6379:6379"   # Pouze localhost
    restart: unless-stopped

volumes:
  qdrant_data:
```

Spuštění kontejnerů:
```bash
docker compose up -d
```

Ověření Qdrant (port 6333):
```bash
curl http://localhost:6333/healthz
```

Očekávaný výstup: JSON s `"title": "qdrant - vector search engine"`

Ověření Redis (port 6379):
```bash
docker exec redis redis-cli ping
```

Očekávaný výstup: `PONG`

---

### 4.8 Spuštění systému — závazné pořadí

> ⚠️ **Celery worker nesmí nastartovat dříve než Redis.** FastAPI musí nastartovat až po načtení `.env` s platnými hodnotami.

**Terminál 1 — Docker (Qdrant + Redis):**
```bash
docker compose up -d
```

**Terminál 2 — Ollama:**
```bash
ollama serve
```

**Terminál 3 — FastAPI:**
```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

**Terminál 4 — Celery Worker:**
```bash
celery -A app.core.celery_app worker --loglevel=info
```

Ověření všech komponent najednou:
```bash
curl http://localhost:6333/healthz && \
curl http://localhost:11434 && \
curl -o /dev/null -s -w "%{http_code}\n" http://localhost:8000/docs
```

Očekávaný výstup posledního příkazu: `200`

---

## 5. Konfigurace .env

> ⚠️ **`.env` nesmí být nikdy commitnut na GitHub.** Obsahuje přihlašovací údaje k SharePointu (client secret) — únik znamená přístup k firemním dokumentům.

Kompletní `.env` se všemi proměnnými, ukázkovými hodnotami a komentáři:

```dotenv
# ── SharePoint / MS Graph API ──────────────────────────────────────────────────
# Azure AD → App Registration → Application (client) ID
SHAREPOINT_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# Azure AD → App Registration → Certificates & secrets → Client secret value
SHAREPOINT_CLIENT_SECRET=your~ClientSecret~Value~Here

# Azure AD → Tenant ID (Directory ID) — najdeš v Azure Portal → Azure Active Directory
SHAREPOINT_TENANT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# URL SharePoint site s interními dokumenty
# Formát: https://vasefirma.sharepoint.com/sites/nazev-site
SHAREPOINT_SITE_URL=https://vasefirma.sharepoint.com/sites/interni-dokumenty

# ── Qdrant ─────────────────────────────────────────────────────────────────────
# URL Qdrant instance (Docker na localhostu)
QDRANT_URL=http://localhost:6333

# ── Redis ──────────────────────────────────────────────────────────────────────
# URL Redis pro Celery broker — /0 = databáze č. 0
REDIS_URL=redis://localhost:6379/0

# ── Ollama ─────────────────────────────────────────────────────────────────────
# URL Ollama serveru
OLLAMA_URL=http://localhost:11434

# Hlavní LLM model — Answer Agent, Metadata Agent, Summary Agent
OLLAMA_MODEL=mistral:latest

# Embedding model — ⚠️ NESMÍ SE MĚNIT po prvním ingestu bez smazání Qdrant kolekce
# Změna modelu = nutný reingest všech dokumentů
OLLAMA_EMBED_MODEL=nomic-embed-text

# ── Logování ───────────────────────────────────────────────────────────────────
# Úroveň logování: DEBUG / INFO / WARNING / ERROR
LOG_LEVEL=INFO
```

`.env.example` — vzor bez citlivých hodnot, **patří na GitHub**:

```dotenv
SHAREPOINT_CLIENT_ID=
SHAREPOINT_CLIENT_SECRET=
SHAREPOINT_TENANT_ID=
SHAREPOINT_SITE_URL=
QDRANT_URL=http://localhost:6333
REDIS_URL=redis://localhost:6379/0
OLLAMA_URL=http://localhost:11434
OLLAMA_MODEL=mistral:latest
OLLAMA_EMBED_MODEL=nomic-embed-text
LOG_LEVEL=INFO
```

`.gitignore` — povinný minimální obsah:

```gitignore
# Přihlašovací údaje — nikdy na GitHub
.env

# Stažené PDF soubory — mohou obsahovat důvěrné dokumenty
data/raw/

# Audit log — obsahuje dotazy a odpovědi uživatelů
audit_log.db

# Python
venv/
__pycache__/
*.pyc
*.pyo
.pytest_cache/

# OS
.DS_Store
Thumbs.db
```

---

## 6. Implementační plán — 1. iterace

> ⚠️ **Proč nelze začít s orchestrátory `ingest.py` a `rag_chain.py`:**
> Orchestrátoři importují utility moduly (`from app.utils import sharepoint, pdf_parser, ...`). Pokud tyto soubory nejsou implementovány, Python selže při importu s `ModuleNotFoundError` nebo `ImportError` ještě před spuštěním jakékoli logiky. Správný postup je implementovat každý utility modul samostatně, ověřit jeho funkčnost izolovaně, a teprve poté sestavit orchestrátor který je volá v pořadí.

| Pořadí | Soubor | Co implementovat | Závislosti (musí existovat před tímto krokem) | Odhadovaná náročnost (hod.) |
|---|---|---|---|---|
| 1 | `app/core/celery_app.py` | Celery instance, připojení na Redis broker přes `REDIS_URL` | Redis běží na portu 6379; `REDIS_URL` v `.env` | 1 |
| 2 | `app/utils/sharepoint.py` | Připojení MS Graph API, procházení složek, SHA-256 hash porovnání, stažení PDF do `data/raw/` | `SHAREPOINT_*` proměnné v `.env`; `office365-rest-python-client` nainstalován | 4 |
| 3 | `app/utils/pdf_parser.py` | `pymupdf4llm.to_markdown()` wrapper — vstup: `Path`, výstup: `str` Markdown | `pymupdf4llm` nainstalován | 1 |
| 4 | `app/utils/cleaner.py` | RegEx vzory pro záhlaví, zápatí, čísla stran — vstup/výstup: `str` | — (stdlib `re`) | 2 |
| 5 | `app/utils/metadata_agent.py` | `ChatOllama` prompt pro extrakci metadat; Pydantic model `DocumentMetadata`; validace výstupu | Ollama běží; `mistral:latest` stažen; `langchain-ollama`, `pydantic` | 3 |
| 6 | `app/utils/summary_agent.py` | `ChatOllama` prompt pro 1–2 větné shrnutí | Ollama běží; `mistral:latest` stažen | 2 |
| 7 | `app/utils/chunker.py` | `RecursiveCharacterTextSplitter` (chunk 1000, overlap 150) + Document Builder (chunk + metadata + shrnutí → `Document`) | `langchain` nainstalován | 2 |
| 8 | `app/utils/embedder.py` | `OllamaEmbeddings` wrapper — vstup: `list[str]`, výstup: `list[list[float]]` | Ollama běží; `nomic-embed-text` stažen | 1 |
| 9 | `app/utils/qdrant_store.py` | Uložení `Document` objektů do Qdrantu; inicializace kolekce (dim. 768); inicializace SQLite tabulek; zápis záznamu do tabulky `dokumenty` | Qdrant běží na 6333; kroky 7 a 8 hotovy | 3 |
| 10 | `app/core/ingest.py` | Celery task orchestrátor — volá kroky 01–09 v pořadí; `try/except` per soubor; zápis chyb do SQLite | Kroky 2–9 hotovy a otestovány izolovaně | 3 |
| 11 | `app/core/rag_chain.py` | Chat pipeline — Query Embedder → Retrieval (top 10) → Answer Agent → Citation Builder → Audit log; fallback `try/except` | Kroky 8–9 hotovy; Qdrant obsahuje data z testu ingestu | 4 |
| 12 | `app/main.py` | FastAPI app; `BaseSettings` `.env` validace při startu; `POST /ingest/sync`; `POST /chat`; fallback error handlery | Kroky 1, 10, 11 hotovy | 3 |

**Celková odhadovaná náročnost 1. iterace: ~29 hodin**

---

## 7. Implementační plán — 2. iterace

| Pořadí | Soubor | Co implementovat | Závislosti | Odhadovaná náročnost (hod.) |
|---|---|---|---|---|
| 1 | `app/utils/reranker.py` | `BGE-Reranker-v2-m3` přes `FlagEmbedding`; vstup: dotaz + 10 chunků; výstup: top 3–5 seřazených `Document` | `FlagEmbedding` nainstalován; 1. iterace plně funkční | 3 |
| 2 | `app/core/rag_chain.py` | Přidat Hybrid Retrieval (BM25 + vector nativně v Qdrantu); integrace `reranker.py` jako krok 03; přidat `ConversationSummaryMemory` jako krok 06 | `reranker.py` hotov; Qdrant s BM25 indexem | 5 |
| 3 | `app/main.py` | JWT middleware — `python-jose` pro vydávání a validaci tokenů; `passlib` pro hashování hesel; filtrování Qdrant dotazů podle hodnoty `oddeleni` z JWT payload | `python-jose`, `passlib`, `fastapi.security` nainstalovány | 4 |
| 4 | `app/main.py` | `GET /health` endpoint — async kontrola dostupnosti Qdrantu, Redisu a Ollamy přes `httpx`; vrátí stav každé závislosti | `httpx`, `qdrant-client`, `redis` nainstalovány | 2 |
| 5 | `app/main.py` | Rate limiting middleware — `slowapi` + Redis jako počítadlo; konfigurovat limit per uživatel/minutu | `slowapi` nainstalován; Redis běží | 2 |

**Celková odhadovaná náročnost 2. iterace: ~16 hodin**

---

## 8. Detailní popis každého modulu

---

### `app/utils/sharepoint.py`

**Účel:** Připojení na SharePoint přes MS Graph API, detekce změněných souborů pomocí SHA-256 hashe a stažení nových nebo změněných PDF do `data/raw/`.

**Vstup:** Konfigurace z `.env` — `SHAREPOINT_CLIENT_ID`, `SHAREPOINT_CLIENT_SECRET`, `SHAREPOINT_TENANT_ID`, `SHAREPOINT_SITE_URL`

**Výstup:** `list[Path]` — cesty ke staženým PDF souborům v `data/raw/`

**Klíčové knihovny:**
```python
from office365.runtime.auth.client_credential import ClientCredential
from office365.sharepoint.client_context import ClientContext
import hashlib
import requests
import os
from pathlib import Path
```

> ⚠️ **Past — synchronní API:** `office365-rest-python-client` používá synchronní volání. Nevolat přímo z async FastAPI endpointu — vždy přes Celery worker, jinak zablokuje event loop.

> ⚠️ **Past — SHA-256 vs. datum modifikace:** SHA-256 hash se počítá z binárního obsahu souboru, ne z názvu nebo data modifikace. Pokud SharePoint změní metadata souboru bez změny obsahu, datum modifikace se změní, ale hash zůstane stejný — soubor se správně nepřestahuje. Opačný případ: obsah se změní ale datum ne — hash tuto situaci zachytí.

**Poznámky k implementaci:**
- Před stažením načíst SHA-256 hash naposledy stažené verze ze SQLite tabulky `dokumenty`
- Porovnat s hashem aktuální verze na SharePointu — stahovat pouze při neshodě
- Ukládat stažené PDF do `data/raw/{nazev_souboru}.pdf`
- Logovat každý stažený soubor do SQLite jako nový záznam se stavem `OK`

---

### `app/utils/pdf_parser.py`

**Účel:** Lokální extrakce textu z PDF souboru do Markdown formátu se zachováním tabulek, nadpisů a struktury dokumentu.

**Vstup:** `Path` — cesta k PDF souboru v `data/raw/`

**Výstup:** `str` — Markdown reprezentace obsahu dokumentu

**Klíčové knihovny:**
```python
import pymupdf4llm
from pathlib import Path
```

> ⚠️ **Past — skenované dokumenty:** `pymupdf4llm` extrahuje text z digitálních PDF (generovaných počítačem). Pro skenované dokumenty (PDF jako obrázek) vrátí prázdný nebo nesmyslný výstup bez chyby. Interní směrnice jsou zpravidla digitální — pokud by se přidaly skeny, nutno přidat OCR krok (např. `pytesseract`).

**Poznámky k implementaci:**
```python
def parse_pdf(pdf_path: Path) -> str:
    return pymupdf4llm.to_markdown(str(pdf_path))
```

---

### `app/utils/cleaner.py`

**Účel:** Deterministické odstranění šumu z Markdown textu — záhlaví, zápatí, čísla stran, opakující se právní doložky.

**Vstup:** `str` — Markdown text z `pdf_parser.py`

**Výstup:** `str` — čistý text bez šumu, normalizovaný whitespace

**Klíčové knihovny:**
```python
import re
```

> ⚠️ **Past — RegEx agresivita:** Příliš agresivní vzory mohou odstranit i části obsahu (např. nadpisy obsahující čísla stran). Vzory testovat na reprezentativních ukázkách dokumentů před nasazením.

**Poznámky k implementaci:**
```python
def clean_text(text: str) -> str:
    # Čísla stran: "- 12 -", "Strana 12", "Str. 3 / 45"
    text = re.sub(r'-\s*\d+\s*-', '', text)
    text = re.sub(r'(?i)[Ss]trana\s+\d+(\s*/\s*\d+)?', '', text)
    # Opakující se záhlaví/zápatí
    text = re.sub(r'(?m)^.{0,80}(s\.r\.o\.|a\.s\.|Interní|DŮVĚRNÉ).{0,80}$', '', text)
    # Normalizace nadměrných prázdných řádků
    text = re.sub(r'\n{3,}', '\n\n', text)
    return text.strip()
```

---

### `app/utils/metadata_agent.py`

**Účel:** Extrakce strukturovaných metadat z dokumentu pomocí Mistral 7B s validací výstupu přes Pydantic.

**Vstup:** `str` — čistý text dokumentu (doporučeno prvních ~2 000 znaků)

**Výstup:** `DocumentMetadata` — Pydantic model s validovanými metadaty

**Klíčové knihovny:**
```python
from langchain_ollama import ChatOllama
from langchain.schema import HumanMessage, SystemMessage
from pydantic import BaseModel
from typing import Optional
import json
import os
```

**Pydantic model:**
```python
class DocumentMetadata(BaseModel):
    nazev: str
    oddeleni: Optional[str] = None
    platnost_od: Optional[str] = None
    platnost_do: Optional[str] = None
    verze: Optional[str] = None
    pro_koho: Optional[str] = None
```

> ⚠️ **Past — nestabilní JSON výstup LLM:** Mistral může vrátit JSON obalený v Markdown backticks (` ```json ... ``` `) nebo s extra textem před/po. Vždy parsovat obranou logikou — extrahovat JSON přes `re.search(r'\{.*\}', response, re.DOTALL)` a obalit do `try/except`. Při selhání vrátit `DocumentMetadata(nazev=nazev_souboru)` — ingest nesmí selhat kvůli metadatům.

**Poznámky k implementaci:**
- Prompt musí explicitně instruovat model k vrácení **pouze** JSON bez Markdown formátování
- Předat modelu pouze prvních ~2 000 znaků textu — metadata jsou zpravidla na začátku dokumentu

---

### `app/utils/summary_agent.py`

**Účel:** Generování stručného 1–2 větného shrnutí dokumentu pomocí Mistral 7B pro kontext při zobrazení citací uživateli.

**Vstup:** `str` — čistý text dokumentu (doporučeno prvních ~3 000 znaků)

**Výstup:** `str` — shrnutí dokumentu, max. 200 znaků

**Klíčové knihovny:**
```python
from langchain_ollama import ChatOllama
from langchain.schema import HumanMessage, SystemMessage
import os
```

> ⚠️ **Past — délka shrnutí:** Shrnutí je přilepeno ke každému chunku a cestuje do kontextu modelu při chat pipeline. Příliš dlouhé shrnutí zbytečně spotřebuje kontext okno modelu. Vynucovat limit délky v promptu i při zpracování výstupu (`.strip()[:200]`).

---

### `app/utils/chunker.py`

**Účel:** Rozdělení čistého textu na překrývající se bloky a sestavení LangChain `Document` objektů s metadaty a shrnutím přilepeným ke každému chunku.

**Vstup:** `str` (text), `DocumentMetadata`, `str` (shrnutí), `str` (název souboru), `str` (SharePoint URL)

**Výstup:** `list[Document]` — LangChain Document objekty připravené k embeddování

**Klíčové knihovny:**
```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.schema import Document
```

**Konfigurace splitteru:**
```python
splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=150,
    separators=["\n\n", "\n", ".", " ", ""]
)
```

> ⚠️ **Past — metadata na každém chunku:** Metadata (`nazev_souboru`, `sharepoint_url`, `stranka`, `oddeleni`) musí být přilepena ke **každému** chunku — ne jen k prvnímu. Při retrívalu Qdrant vrátí jednotlivé chunky bez kontextu ostatních. Bez metadat na každém chunku nelze sestavit citaci.

**Poznámky k implementaci:**
```python
def build_documents(text, metadata, summary, filename, sp_url) -> list[Document]:
    chunks = splitter.split_text(text)
    return [
        Document(
            page_content=chunk,
            metadata={
                "nazev_souboru": filename,
                "sharepoint_url": sp_url,
                "oddeleni": metadata.oddeleni,
                "verze": metadata.verze,
                "shrnuti": summary,
                "chunk_index": i,
            }
        )
        for i, chunk in enumerate(chunks)
    ]
```

---

### `app/utils/embedder.py`

**Účel:** Převod textu chunků na vektorové reprezentace pomocí `nomic-embed-text` přes Ollama.

**Vstup:** `list[str]` — texty chunků

**Výstup:** `list[list[float]]` — embedding vektory (dimenze 768 pro `nomic-embed-text`)

**Klíčové knihovny:**
```python
from langchain_ollama import OllamaEmbeddings
import os
```

> ⚠️ **Kritické pravidlo:** Název modelu načítat **výhradně** z `os.getenv("OLLAMA_EMBED_MODEL")` — nikdy nezadrátovat jako string. Stejná proměnná musí být použita v `rag_chain.py` pro Query Embedder. Jakákoli odchylka způsobí nefunkční vyhledávání.

**Poznámky k implementaci:**
```python
def get_embeddings() -> OllamaEmbeddings:
    return OllamaEmbeddings(
        model=os.getenv("OLLAMA_EMBED_MODEL"),
        base_url=os.getenv("OLLAMA_URL")
    )
```

---

### `app/utils/qdrant_store.py`

**Účel:** Uložení Document objektů s vektory do Qdrant kolekce a zápis záznamu o zpracovaném dokumentu do SQLite tabulky `dokumenty`. Obsahuje také inicializaci SQLite schématu.

**Vstup:** `list[Document]` — chunky s embeddingy a metadaty; `str` — název kolekce; metadata dokumentu pro SQLite

**Výstup:** `None` (side effects: data v Qdrantu, záznam v SQLite)

**Klíčové knihovny:**
```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams
from langchain_qdrant import QdrantVectorStore
import sqlite3
import os
from datetime import datetime, timezone
```

> ⚠️ **Past — dimenze vektoru:** Qdrant kolekce musí být inicializována se správnou dimenzí **před prvním zápisem**. `nomic-embed-text` produkuje vektory dimenze **768**. Pokud se změní embedding model na jiný s jinou dimenzí, kolekci je nutno smazat a znovu vytvořit — reingest všech dokumentů.

**Inicializace kolekce:**
```python
def init_collection(client: QdrantClient, collection_name: str = "documents"):
    if collection_name not in [c.name for c in client.get_collections().collections]:
        client.create_collection(
            collection_name=collection_name,
            vectors_config=VectorParams(size=768, distance=Distance.COSINE)
        )
```

**Inicializace SQLite tabulek (volat při startu `main.py`):**
```python
def init_db(db_path: str = "audit_log.db"):
    conn = sqlite3.connect(db_path)
    conn.executescript("""
        CREATE TABLE IF NOT EXISTS dotazy (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp TEXT NOT NULL,
            text_dotazu TEXT NOT NULL,
            text_odpovedi TEXT,
            citovane_zdroje TEXT,
            hodnoceni INTEGER,
            oddeleni_uzivatele TEXT,
            doba_odpovedi_ms INTEGER
        );
        CREATE TABLE IF NOT EXISTS dokumenty (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp TEXT NOT NULL,
            nazev_souboru TEXT NOT NULL,
            sharepoint_url TEXT,
            sha256_hash TEXT,
            datum_modifikace TEXT,
            pocet_chunku INTEGER,
            stav TEXT DEFAULT 'OK'
        );
        CREATE TABLE IF NOT EXISTS chyby (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp TEXT NOT NULL,
            typ_chyby TEXT,
            modul TEXT,
            detail_chyby TEXT,
            soubor_nebo_dotaz TEXT,
            vyresen INTEGER DEFAULT 0
        );
    """)
    conn.commit()
    conn.close()
```

---

### `app/utils/reranker.py` *(2. iterace)*

**Účel:** Přeřazení top 10 chunků z Qdrantu podle skutečné relevance k dotazu pomocí `BGE-Reranker-v2-m3`.

**Vstup:** `str` (dotaz uživatele), `list[Document]` (top 10 chunků z Qdrantu)

**Výstup:** `list[Document]` — top 3–5 chunků seřazených podle relevance (sestupně)

**Klíčové knihovny:**
```python
from FlagEmbedding import FlagReranker
```

> ⚠️ **Past — inicializace rerankeru:** `FlagReranker` při prvním vytvoření instance stáhne model z HuggingFace (~550 MB). Inicializovat jako singleton při startu aplikace — ne při každém dotazu.

---

### `app/core/celery_app.py`

**Účel:** Konfigurace Celery instance a připojení na Redis broker — importováno v `ingest.py` a `main.py`.

**Vstup:** —

**Výstup:** `celery_app` instance

**Klíčové knihovny:**
```python
from celery import Celery
import os
from dotenv import load_dotenv
```

**Implementace:**
```python
load_dotenv()

celery_app = Celery(
    "rag_assistant",
    broker=os.getenv("REDIS_URL"),
    backend=os.getenv("REDIS_URL"),
    include=["app.core.ingest"]
)

celery_app.conf.update(
    task_serializer="json",
    result_serializer="json",
    accept_content=["json"],
    timezone="Europe/Prague",
)
```

---

### `app/core/ingest.py`

**Účel:** Orchestrátor ingest pipeline — Celery task, který volá utility kroky 01–09 v závazném pořadí a předává výstupy mezi kroky.

**Vstup:** — (spouští se jako Celery task bez argumentů; vše načítá z `.env`)

**Výstup:** `dict` — `{"zpracovano": int, "preskoceno": int, "chyby": int}`

**Klíčové knihovny:**
```python
from celery import shared_task
from app.utils.sharepoint import download_new_files
from app.utils.pdf_parser import parse_pdf
from app.utils.cleaner import clean_text
from app.utils.metadata_agent import extract_metadata
from app.utils.summary_agent import generate_summary
from app.utils.chunker import build_documents
from app.utils.embedder import get_embeddings
from app.utils.qdrant_store import store_documents, log_error
```

> ⚠️ **Past — chyba jednoho souboru nesmí zastavit ingest:** Každý soubor zpracovat v samostatném `try/except` bloku. Při chybě zapsat traceback do SQLite tabulky `chyby` a pokračovat dalším souborem.

**Struktura orchestrátoru:**
```python
@shared_task
def run_ingest():
    files = download_new_files()
    results = {"zpracovano": 0, "preskoceno": 0, "chyby": 0}
    for pdf_path in files:
        try:
            text = parse_pdf(pdf_path)           # krok 02
            text = clean_text(text)              # krok 03
            metadata = extract_metadata(text[:2000])  # krok 04
            summary = generate_summary(text[:3000])   # krok 05
            docs = build_documents(...)          # kroky 06 + 07
            store_documents(docs)                # kroky 08 + 09
            results["zpracovano"] += 1
        except Exception as e:
            log_error(type(e).__name__, "ingest.py", str(e), pdf_path.name)
            results["chyby"] += 1
    return results
```

---

### `app/core/rag_chain.py`

**Účel:** Orchestrátor chat pipeline — synchronně zpracuje dotaz uživatele a vrátí odpověď s citacemi.

**Vstup:** `str` — text dotazu uživatele

**Výstup:** `dict` — `{"odpoved": str, "zdroje": list[dict], "doba_odpovedi_ms": int}`

**Klíčové knihovny:**
```python
from langchain_ollama import ChatOllama, OllamaEmbeddings
from langchain_qdrant import QdrantVectorStore
from langchain.schema import HumanMessage, SystemMessage
from qdrant_client import QdrantClient
import sqlite3, time, os
from datetime import datetime, timezone
```

**Systémový prompt Answer Agenta:**
```python
SYSTEM_PROMPT = """Jsi asistent pro vyhledávání informací v interních firemních dokumentech.
Odpovídej VÝHRADNĚ na základě níže uvedených pasáží z dokumentů.
Pokud odpověď v dokumentech není obsažena, odpověz přesně:
'Tuto informaci jsem v dostupných dokumentech nenašel.'
Nikdy nevymýšlej informace ani neodpovídej z vlastních znalostí.
Odpovídej vždy v češtině."""
```

> ⚠️ **Past — fallback při výpadku:** Celou pipeline obalit do `try/except`. Při výpadku Ollamy nebo Qdrantu uživateli vrátit JSON s chybovou zprávou (HTTP 200 s polem `"chyba"`), zároveň zapsat chybu do SQLite tabulky `chyby`. Nikdy nevracet HTTP 500 bez kontextu.

> ⚠️ **Past — embedding model:** Query Embedder musí používat **stejný model** jako Embedder v ingest pipeline. Načítat z `os.getenv("OLLAMA_EMBED_MODEL")` — nikdy nezadrátovat.

---

### `app/main.py`

**Účel:** FastAPI aplikace — definice endpointů, validace `.env` při startu, delegování do orchestrátorů, fallback error handlery.

**Vstup:** HTTP požadavky na `/ingest/sync` a `/chat`

**Výstup:** JSON odpovědi

**Klíčové knihovny:**
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from pydantic_settings import BaseSettings
from pydantic import AnyUrl
from app.core.celery_app import celery_app
from app.core.rag_chain import process_query
from app.utils.qdrant_store import init_db
import os
```

**`.env` validace při startu — aplikace odmítne nastartovat při chybějící proměnné:**
```python
class Settings(BaseSettings):
    sharepoint_client_id: str
    sharepoint_client_secret: str
    sharepoint_tenant_id: str
    sharepoint_site_url: str
    qdrant_url: AnyUrl
    redis_url: str
    ollama_url: AnyUrl
    ollama_model: str
    ollama_embed_model: str
    log_level: str = "INFO"

    class Config:
        env_file = ".env"

settings = Settings()  # Selže při startu pokud chybí jakákoli povinná proměnná
```

> ⚠️ **Past — pořadí inicializace:** `init_db()` volat v `@app.on_event("startup")` handleru — ne na úrovni modulu, kde by se spustila i při importu v testech.

---

## 9. FastAPI endpointy — detailní specifikace

### `POST /ingest/sync`

**Účel:** Zapíše ingest úlohu do Redis fronty. Celery worker ji vyzvedne a spustí pipeline asynchronně na pozadí.

**Kdo volá a kdy:** IT admin manuálně nebo cron job po nahrání nových dokumentů na SharePoint.

**Request body:**
```json
{}
```
*(žádné tělo — pipeline stáhne vše ze SharePointu automaticky na základě `.env` konfigurace)*

**Response body (HTTP 200):**
```json
{
  "status": "spuštěno",
  "message": "Ingest pipeline byla zařazena do fronty. Sledujte audit_log.db pro výsledky.",
  "task_id": "d3b4e5f6-7890-abcd-ef01-234567890abc"
}
```

**Příklad volání přes curl:**
```bash
curl -X POST http://localhost:8000/ingest/sync \
  -H "Content-Type: application/json"
```

---

### `POST /chat`

**Účel:** Synchronně zpracuje dotaz uživatele — projde celou chat pipeline a vrátí odpověď s citacemi zdrojů.

**Kdo volá a kdy:** Uživatel přes chat interface při každém dotazu.

**Request body:**
```json
{
  "dotaz": "Jaká je lhůta pro podání žádosti o dovolenou?"
}
```

**Response body (HTTP 200 — úspěch):**
```json
{
  "odpoved": "Žádost o dovolenou je nutné podat minimálně 14 dní předem, pokud délka dovolené přesahuje 5 pracovních dní.",
  "zdroje": [
    {
      "soubor": "Smernice_HR_05_Dovolena.pdf",
      "stranka": 3,
      "sharepoint_url": "https://vasefirma.sharepoint.com/sites/interni-dokumenty/Smernice_HR_05_Dovolena.pdf",
      "oddeleni": "HR",
      "relevance_score": 0.92
    }
  ],
  "doba_odpovedi_ms": 7430
}
```

**Response body (HTTP 200 — informace nenalezena):**
```json
{
  "odpoved": "Tuto informaci jsem v dostupných dokumentech nenašel.",
  "zdroje": [],
  "doba_odpovedi_ms": 3210
}
```

**Response body (HTTP 200 — výpadek závislosti):**
```json
{
  "chyba": "Systém je dočasně nedostupný. Zkuste to prosím za chvíli.",
  "zdroje": [],
  "doba_odpovedi_ms": 120
}
```

**Příklad volání přes curl:**
```bash
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"dotaz": "Jaká je lhůta pro podání žádosti o dovolenou?"}'
```

---

### `GET /health` *(2. iterace)*

**Účel:** Zkontroluje dostupnost všech závislostí systému a vrátí stav každé z nich.

**Kdo volá a kdy:** Monitoring systém (Grafana, Zabbix) nebo IT admin.

**Request body:** žádné

**Response body (HTTP 200 — vše OK):**
```json
{
  "status": "ok",
  "zavislosti": {
    "qdrant": "ok",
    "redis": "ok",
    "ollama": "ok"
  }
}
```

**Response body (HTTP 503 — závislost nedostupná):**
```json
{
  "status": "degraded",
  "zavislosti": {
    "qdrant": "ok",
    "redis": "ok",
    "ollama": "chyba"
  }
}
```

**Příklad volání přes curl:**
```bash
curl http://localhost:8000/health
```

---

## 10. SQLite schéma

**Proč SQLite a ne PostgreSQL:** SQLite je součástí Python stdlib — nulová instalace, nulová konfigurace, nulová správa. Pro audit log s desítkami tisíc záznamů je výkon plně dostačující. Backup = zkopírovat jeden soubor `audit_log.db`. Soubor lze otevřít v DB Browser for SQLite bez databázového serveru.

```sql
-- ── Tabulka: dotazy ───────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS dotazy (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp           TEXT NOT NULL,      -- ISO 8601: "2025-03-15T14:32:01+01:00"
    text_dotazu         TEXT NOT NULL,      -- původní otázka uživatele
    text_odpovedi       TEXT,               -- vygenerovaná odpověď modelu
    citovane_zdroje     TEXT,               -- JSON: [{"soubor": "...", "stranka": 3, "url": "..."}]
    hodnoceni           INTEGER,            -- 1 = palec nahoru / 0 = palec dolů / NULL = bez hodnocení
    oddeleni_uzivatele  TEXT,               -- pro filtrování po zavedení JWT (2. iterace)
    doba_odpovedi_ms    INTEGER             -- celková doba zpracování v milisekundách
);

-- ── Tabulka: dokumenty ────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS dokumenty (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp           TEXT NOT NULL,      -- čas zpracování ingestem
    nazev_souboru       TEXT NOT NULL,      -- "Smernice_HR_05.pdf"
    sharepoint_url      TEXT,               -- přímý odkaz na soubor na SharePointu
    sha256_hash         TEXT,               -- SHA-256 otisk obsahu pro detekci změn
    datum_modifikace    TEXT,               -- datum poslední změny na SharePointu
    pocet_chunku        INTEGER,            -- počet indexovaných bloků v Qdrantu
    stav                TEXT DEFAULT 'OK'   -- "OK" nebo "chyba"
);

-- ── Tabulka: chyby ────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS chyby (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp           TEXT NOT NULL,      -- čas vzniku chyby
    typ_chyby           TEXT,               -- "OllamaUnavailable", "QdrantTimeout", "PDFParseError"
    modul               TEXT,               -- soubor kde chyba nastala
    detail_chyby        TEXT,               -- plný Python traceback
    soubor_nebo_dotaz   TEXT,               -- kontext: název souboru nebo text dotazu
    vyresen             INTEGER DEFAULT 0   -- 0 = nevyřešeno / 1 = vyřešeno
);
```

**Ukázkové analytické dotazy:**

Nejčastěji hledané výrazy (top 20):
```sql
SELECT text_dotazu, COUNT(*) AS pocet
FROM dotazy
GROUP BY text_dotazu
ORDER BY pocet DESC
LIMIT 20;
```

Průměrná doba odpovědi podle dne za posledních 7 dní:
```sql
SELECT
    DATE(timestamp)              AS den,
    ROUND(AVG(doba_odpovedi_ms)) AS prumer_ms,
    MIN(doba_odpovedi_ms)        AS minimum_ms,
    MAX(doba_odpovedi_ms)        AS maximum_ms,
    COUNT(*)                     AS pocet_dotazu
FROM dotazy
WHERE timestamp >= DATE('now', '-7 days')
GROUP BY den
ORDER BY den DESC;
```

Přehled nevyřešených chyb podle modulu:
```sql
SELECT
    modul,
    typ_chyby,
    COUNT(*)        AS pocet,
    MAX(timestamp)  AS posledni_vyskyt
FROM chyby
WHERE vyresen = 0
GROUP BY modul, typ_chyby
ORDER BY pocet DESC;
```

---

## 11. Testování a ověření funkčnosti

### 11.1 Ověření infrastruktury

Qdrant (port 6333):
```bash
curl http://localhost:6333/healthz
```
Očekávaný výstup: `{"title":"qdrant - vector search engine","version":"..."}`

Redis (port 6379):
```bash
docker exec redis redis-cli ping
```
Očekávaný výstup: `PONG`

Ollama (port 11434):
```bash
curl http://localhost:11434
```
Očekávaný výstup: `Ollama is running`

FastAPI (port 8000):
```bash
curl -o /dev/null -s -w "%{http_code}" http://localhost:8000/docs
```
Očekávaný výstup: `200`

---

### 11.2 Ověření Ollamy

Výpis nainstalovaných modelů:
```bash
ollama list
```

Testovací inference Mistral 7B:
```bash
curl http://localhost:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{"model": "mistral", "prompt": "Odpověz jednou větou česky: Co je to RAG?", "stream": false}'
```

Testovací embedding a ověření dimenze vektoru (musí být 768):
```bash
curl -s http://localhost:11434/api/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model": "nomic-embed-text", "prompt": "testovací text"}' \
  | python -c "import sys,json; d=json.load(sys.stdin); print(f'Dimenze: {len(d[\"embedding\"])}')"
```

Očekávaný výstup: `Dimenze: 768`

---

### 11.3 První ingest

Spuštění přes Swagger UI:
```
http://localhost:8000/docs → POST /ingest/sync → Execute
```

Ověření záznamu v SQLite po dokončení:
```bash
sqlite3 audit_log.db \
  "SELECT nazev_souboru, pocet_chunku, stav, timestamp FROM dokumenty ORDER BY timestamp DESC LIMIT 10;"
```

Ověření dat v Qdrantu — počet vektorů musí být > 0:
```bash
curl http://localhost:6333/collections/documents
```

---

### 11.4 První chat dotaz

```bash
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"dotaz": "Jaká je lhůta pro schválení cestovních náhrad?"}'
```

Ověření zápisu do audit logu:
```bash
sqlite3 audit_log.db \
  "SELECT timestamp, substr(text_dotazu,1,50), doba_odpovedi_ms FROM dotazy ORDER BY timestamp DESC LIMIT 5;"
```

---

### 11.5 Ukázkové testovací dotazy v češtině

| # | Dotaz | Složitost | Očekávaný typ odpovědi |
|---|---|---|---|
| 1 | `"Jaká je lhůta pro podání žádosti o dovolenou?"` | Jednoduchý | Konkrétní počet dní + citace HR směrnice |
| 2 | `"Kdo schvaluje pracovní cesty nad 5 000 Kč?"` | Jednoduchý | Název pozice nebo oddělení + citace |
| 3 | `"Jaký je postup při hlášení pracovního úrazu?"` | Střední | Číslovaný postup + citace BOZP dokumentu |
| 4 | `"Platí směrnice o home office i pro obchodní zástupce?"` | Střední | Odpověď s odkazem na konkrétní pasáž a podmínky |
| 5 | `"Co se stane když zaměstnanec opakovaně poruší GDPR pravidla?"` | Složitý | Sankční postup + citace compliance směrnice + případně více zdrojů |

---

## 12. Průřezové komponenty

> Platí pro celý systém — nejsou součástí jedné pipeline, chrání vše.

| Komponenta | Soubor | Iterace | Popis | Knihovny |
|---|---|---|---|---|
| **.env Validace** | spouští se při startu `main.py` | ⚠️ 1. iterace | Při startu ověří že všechny hodnoty v `.env` jsou vyplněné — SharePoint přihlašovací údaje, URL Ollamy, URL Qdrantu. Bez toho systém nastartuje, první dotaz selže a chyba bude záhadná. | `pydantic-settings · BaseSettings`, `python-dotenv` |
| **Fallback při výpadku** | `try/except` v `rag_chain.py` | ⚠️ 1. iterace | Když Ollama nebo Qdrant vypadnou, systém neselže s chybou — vrátí uživateli hezkou zprávu. Zapíše chybu do SQLite tabulky `chyby`. | `try/except` (stdlib), `sqlite3` |
| **Health Check** | `GET /health` v `main.py` | 2. iterace | Endpoint který vrátí stav všech závislostí — Qdrant, Redis, Ollama. Bez toho nevíš jestli systém běží správně dokud nepřijde chyba od uživatele. | `fastapi`, `qdrant-client`, `redis`, `httpx` |
| **Autentizace** | middleware v `main.py` | 2. iterace | Ověří kdo se ptá — filtrování směrnic podle oddělení (IT vidí jen IT směrnice) a zápis do SQLite kdo dotaz poslal. Bez toho je `/chat` otevřený pro kohokoliv. | `python-jose`, `passlib`, `fastapi.security` |
| **Rate Limiting** | middleware v `main.py` | 2. iterace | Omezí kolik dotazů může jeden uživatel poslat za minutu. Zabrání zahltení Ollamy a GPU serveru — jeden uživatel nemůže zablokovat ostatní. | `slowapi`, `redis` (počítadlo) |

---

## 13. Troubleshooting

| Symptom | Pravděpodobná příčina | Diagnostický příkaz | Řešení |
|---|---|---|---|
| Ollama neodpovídá na portu 11434 | Ollama služba není spuštěna nebo firewall blokuje port | `curl http://localhost:11434` | Spustit `ollama serve`; zkontrolovat `netstat -an \| grep 11434`; ověřit firewall |
| Model se nevejde do VRAM — inference velmi pomalá | GPU nemá dostatek VRAM; Ollama přepnula na CPU | `ollama ps` | Akceptovat pomalejší CPU inferenci nebo snížit `OLLAMA_NUM_CTX` |
| Qdrant vrací prázdné výsledky po ingestu | Ingest selhal tiše; kolekce je prázdná nebo neexistuje | `curl http://localhost:6333/collections/documents` | Zkontrolovat `chyby` v SQLite; zkontrolovat Celery logy; ověřit `vectors_count` > 0; spustit ingest znovu |
| Chat vrací odpověď bez citací nebo halucinuje | Systémový prompt ignorován; chunky nemají správná metadata | Zkontrolovat `citovane_zdroje` v SQLite tabulce `dotazy` | Ověřit `SYSTEM_PROMPT` v `rag_chain.py`; zkontrolovat metadata v `chunker.py` |
| Celery nepřijímá úlohy z Redisu | Redis není dostupný nebo špatná `REDIS_URL` | `docker exec redis redis-cli ping` | Ověřit `REDIS_URL` v `.env`; restartovat: `docker compose restart redis` |
| SHA-256 hash se nemění ale soubor byl aktualizován na SharePointu | Obsah souboru je identický (pouze jiná metadata na SharePointu) | Ručně porovnat: `sha256sum soubor.pdf` | Smazat záznam z tabulky `dokumenty` a spustit ingest znovu |
| `.env` validace selže při startu aplikace | Chybí povinná proměnná nebo překlep v názvu klíče | `python -c "from app.main import settings; print(settings.dict())"` | Porovnat `.env` s `.env.example`; zkontrolovat překlepy; ověřit aktivní virtualenv |

---

## 14. Bezpečnostní checklist

| Položka | Stav | Co zkontrolovat |
|---|---|---|
| `.env` není na GitHubu | ✗ nutno ověřit | `git status` a `git log --all --full-history -- .env` — nesmí být nikde v historii |
| `data/raw/` není na GitHubu | ✗ nutno ověřit | `git ls-files data/` — výstup musí být prázdný |
| `audit_log.db` není na GitHubu | ✗ nutno ověřit | `git ls-files *.db` — výstup musí být prázdný |
| `.gitignore` obsahuje všechny citlivé soubory | ✗ nutno ověřit | Zkontrolovat sekci 5 — povinný obsah `.gitignore`; `git check-ignore -v .env` |
| Ollama není přístupná mimo localhost | ✗ nutno ověřit | `netstat -an \| grep 11434` — musí zobrazit `127.0.0.1:11434`, nikoli `0.0.0.0:11434` |
| Qdrant není přístupný mimo localhost | ✗ nutno ověřit | V `docker-compose.yml` ověřit `"127.0.0.1:6333:6333"` — s explicitní vazbou na localhost |
| Fallback při výpadku implementován a otestován | ✗ nutno dořešit | Zastavit Ollamu a ověřit `POST /chat` vrátí JSON s `"chyba"` místo HTTP 500 |
| Rate limiting nastaven | ✗ 2. iterace | Implementovat `slowapi` middleware; otestovat opakovanými rychlými dotazy |
| JWT autentizace implementována | ✗ 2. iterace | Implementovat `python-jose` middleware; ověřit filtrování dokumentů podle `oddeleni` z tokenu |
