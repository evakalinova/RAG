# RAG Chatbot — Prompty pro generování dokumentů


## PROMPT 2 — Technicky-realizační plán

```
Jsi senior softwarový architekt, backend engineer a DevOps engineer.
Tvým úkolem je vytvořit kompletní technicky-realizační plán projektu ve formátu Markdown v češtině.

Dokument je určen výhradně pro vývojový tým — musí být přesný, kompletní a přímo použitelný jako podklad pro implementaci bez nutnosti dohledávat cokoliv jinde.

Pravidla která musíš dodržet:
- Tvůj výstup musí přesně vycházet z níže uvedené architektury a referenčního přehledu
- Nezjednodušuj technologii, nic si nevymýšlej mimo uvedený kontext
- Nenahrazuj uvedené knihovny jinými — stack je závazný
- Struktura složek, názvy souborů a názvy komponent jsou závazné — neměň je, nepřidávej soubory navíc
- Každý příkaz musí být funkční a otestovatelný
- Pasti a kritická rozhodnutí označuj explicitně: ⚠️ nebo > **Pozor:**

---

# Kontext projektu

Projekt: AI RAG Chatbot nad interními směrnicemi společnosti
Účel: Zaměstnanci kladou dotazy v přirozené češtině, systém vyhledá relevantní pasáže z interních dokumentů a vygeneruje odpověď s citací zdroje (název souboru, stránka, odkaz na SharePoint).
Dokumenty jsou uloženy na SharePointu a v systému Pinya — ingest pipeline pracuje výhradně se SharePointem přes MS Graph API. Dokumenty ze systému Pinya jsou exportovány jako PDF a manuálně nahrávány na SharePoint.
Kritické omezení: 100% lokální provoz — žádná firemní data nesmí opustit infrastrukturu, žádné externí AI API.

---

# Referenční přehled struktury projektu

> Tento přehled je závazný. Názvy souborů, komponenty, knihovny a pořadí kroků musí odpovídat přesně tomuto dokumentu.

## Fáze 0 — Infrastruktura a software — spustit v tomto pořadí

| # | Komponenta | Příkaz | Port | Popis |
|---|---|---|---|---|
| ⚠️ 1. | Docker Desktop | stáhnout z docker.com | — | Spouští Qdrant a Redis v izolovaných kontejnerech — nainstalovat úplně první |
| 1. | Qdrant | docker compose up -d | 6333 | Vektorová databáze — ukládá embeddingy + metadata |
| 1. | Redis | docker compose up -d | 6379 | Fronta zpráv pro Celery |
| 2. | Ollama | winget install Ollama.Ollama / ollama serve | 11434 | Spouští AI modely lokálně |
| 3. | FastAPI server | uvicorn app.main:app --reload | 8000 | Vstupní brána pro obě pipeline |
| 4. | Celery Worker | celery -A app.core.celery_app worker | — | Zpracování PDF na pozadí — spustit v samostatném terminálu |

## Ollama — Lokální AI modely

| Model | Příkaz | Použití v systému | Hardware |
|---|---|---|---|
| mistral:latest | ollama pull mistral | Hlavní LLM: generuje odpovědi, extrahuje metadata, píše shrnutí | GPU 8 GB VRAM (doporučeno); CPU funguje, odpověď 30–60 s |
| nomic-embed-text | ollama pull nomic-embed-text | Embedding model: převádí text na vektory — ⚠️ musí být stejný při ingestu i dotazech | CPU stačí |
| BGE-Reranker-v2-m3 | přes FlagEmbedding | Reranker výsledků vyhledávání | CPU; 2. iterace |
| mixtral:latest | ollama pull mixtral | Upgrade pro lepší češtinu a složitější dotazy | GPU 24 GB VRAM; volitelný upgrade |

## Python knihovny — 1. iterace

| Knihovna | Účel |
|---|---|
| fastapi | REST API framework |
| uvicorn | ASGI server pro FastAPI |
| celery | Fronta asynchronních úloh |
| redis | Python klient pro Redis (broker pro Celery) |
| pymupdf4llm | Lokální extrakce textu z PDF, zachovává tabulky a nadpisy jako Markdown |
| langchain | Orchestrace pipeline, prompt management |
| langchain-ollama | LangChain integrace pro Ollama (ChatOllama, OllamaEmbeddings) |
| langchain-qdrant | LangChain integrace pro Qdrant |
| qdrant-client | Přímý Python klient pro Qdrant |
| office365-rest-python-client | SharePoint konektor přes MS Graph API |
| requests | HTTP klient |
| pydantic | Validace dat a Pydantic modely |
| pydantic-settings | BaseSettings pro .env validaci při startu |
| python-dotenv | Načítání .env souboru |
| hashlib | SHA-256 hash — stdlib, bez instalace |
| re | RegEx čištění textu — stdlib, bez instalace |
| sqlite3 | Audit log — stdlib, bez instalace |
| os, pathlib | Práce se soubory — stdlib, bez instalace |

## Python knihovny — 2. iterace (přidat k 1. iteraci)

| Knihovna | Účel |
|---|---|
| FlagEmbedding | BGE-Reranker-v2-m3 |
| python-jose | JWT tokeny |
| passlib | Hashování hesel |
| fastapi.security | OAuth2 schéma pro FastAPI |
| slowapi | Rate limiting middleware |
| httpx | Async HTTP klient pro health check |

## Struktura projektu — závazná

rag-assistant/
├── docker-compose.yml        ← spouští Qdrant + Redis
├── requirements.txt          ← všechny pip závislosti
├── .env                      ← přihlašovací údaje a URL (nikdy na GitHub)
├── .env.example              ← vzor bez hodnot (patří na GitHub)
├── .gitignore                ← ignoruje .env, data/raw/, audit_log.db
├── audit_log.db              ← vznikne automaticky při prvním spuštění
├── data/
│   └── raw/                  ← stažené PDF ze SharePointu (nikdy na GitHub)
└── app/
    ├── __init__.py
    ├── main.py               ← FastAPI app, endpointy, middleware, .env validace
    ├── core/
    │   ├── __init__.py
    │   ├── ingest.py         ← orchestrátor ingest pipeline, volá utils v pořadí 01–09
    │   ├── rag_chain.py      ← orchestrátor chat pipeline, volá utils v pořadí 01–07
    │   └── celery_app.py     ← Celery konfigurace, připojení na Redis
    └── utils/
        ├── __init__.py
        ├── sharepoint.py     ← SharePoint konektor, SHA-256 hash porovnání
        ├── pdf_parser.py     ← pymupdf4llm extrakce textu
        ├── cleaner.py        ← RegEx čištění textu
        ├── metadata_agent.py ← Ollama/mistral extrakce metadat, Pydantic validace
        ├── summary_agent.py  ← Ollama/mistral shrnutí dokumentu
        ├── chunker.py        ← RecursiveCharacterTextSplitter + Document Builder
        ├── embedder.py       ← OllamaEmbeddings/nomic-embed-text
        ├── reranker.py       ← BGE-Reranker-v2-m3 (2. iterace)
        └── qdrant_store.py   ← ukládání do Qdrantu + zápis do SQLite

## Ingest Pipeline — 9 kroků, orchestruje ingest.py

Spouští se asynchronně přes Celery worker po zavolání POST /ingest/sync.
Pořadí kroků je závazné — každý krok přijímá výstup předchozího.

| Krok | Soubor | Komponenta | Vstup → Výstup | Knihovny |
|---|---|---|---|---|
| 01 | sharepoint.py | SharePoint Connector | — → PDF soubory v data/raw/ | office365-rest-python-client, hashlib |
| 02 | pdf_parser.py | PDF Parser | PDF → Markdown text | pymupdf4llm |
| 03 | cleaner.py | Text Cleaner | Markdown + šum → čistý text | re (stdlib) |
| 04 | metadata_agent.py | Metadata Agent | čistý text → JSON s metadaty | ChatOllama, pydantic |
| 05 | summary_agent.py | Summary Agent | čistý text → shrnutí 1–2 věty | ChatOllama |
| 06 | chunker.py | Text Chunker | čistý text → seznam chunků | RecursiveCharacterTextSplitter |
| 07 | chunker.py | Document Builder | chunk + metadata + shrnutí → Document objekt | langchain |
| 08 | embedder.py | Embedder | text chunku → vektor čísel | OllamaEmbeddings, nomic-embed-text |
| 09 | qdrant_store.py | Indexer → Qdrant | vektor + Document → uloženo v Qdrantu + SQLite | qdrant-client, langchain-qdrant, sqlite3 |

Poznámky:
- 01 SharePoint Connector: porovnává SHA-256 hash; stahuje pouze nové nebo změněné soubory
- 02 PDF Parser: pymupdf4llm.to_markdown() zachovává tabulky a nadpisy, běží 100% offline
- 03 Text Cleaner: RegEx místo AI — deterministické, rychlé, předvídatelné
- 04 Metadata Agent: extrahuje název, oddělení, platnost_od, platnost_do, verze, pro_koho; Pydantic validuje výstup
- 06 Text Chunker: bloky ~1 000 znaků, přesah 150 znaků pro zachování kontextu na hranicích
- 08 Embedder: ⚠️ kritické pravidlo — stejný model při ingestu i při dotazech (nomic-embed-text)

## Chat Pipeline — 7 kroků, orchestruje rag_chain.py

Spouští se synchronně při každém POST /chat — bez Celery, odpověď se vrátí přímo.

| Krok | Soubor | Komponenta | Vstup → Výstup | Knihovny |
|---|---|---|---|---|
| 01 | rag_chain.py | Query Embedder | text dotazu → vektor dotazu | OllamaEmbeddings, nomic-embed-text |
| 02 | rag_chain.py | Retrieval (1. iter.) | vektor → top 10 chunků | qdrant-client, langchain-qdrant |
| 02 | rag_chain.py | Hybrid Retrieval (2. iter.) | vektor + BM25 + metadata filtr → top 10 chunků | BM25 + vector (nativní v Qdrantu) |
| 03 | reranker.py | Reranker (2. iter.) | dotaz + 10 chunků → top 3–5 seřazených | BGE-Reranker-v2-m3, FlagEmbedding |
| 04 | rag_chain.py | Answer Agent | dotaz + top chunky → text odpovědi | ChatOllama, mistral:latest |
| 05 | rag_chain.py | Citation Builder | odpověď + metadata chunků → odpověď + citace | vlastní Python logika |
| 06 | rag_chain.py | Memory Manager (2. iter.) | dotaz + shrnutí historie → kontextová odpověď | ConversationSummaryMemory |
| 07 | sqlite3 | Odpověď + Audit log | finální odpověď → text + zdroje + zápis do SQLite | sqlite3 (stdlib) |

Poznámky:
- 01 Query Embedder: ⚠️ kritické pravidlo — stejný model jako při ingestu (nomic-embed-text); jiný model = nefunkční vyhledávání
- 04 Answer Agent: striktní pokyn v promptu: odpovídej POUZE z dokumentů; pokud odpověď není v dokumentech — přiznej to; upgrade na Mixtral = změna jednoho řádku
- 05 Citation Builder: citace jsou přesné z metadat Qdrantu — model je nevymýšlí

## FastAPI endpointy

| Metoda | Endpoint | Kdo volá | Popis |
|---|---|---|---|
| POST | /ingest/sync | IT admin | Zapíše úlohu do Redisu → Celery spustí pipeline asynchronně → okamžitě vrátí {"status": "spuštěno"} |
| POST | /chat | Uživatel | Synchronně spustí chat pipeline → vrátí odpověď + citace |
| GET | /health | Monitoring (2. iter.) | Zkontroluje dostupnost Qdrantu, Redisu, Ollamy; vrátí stav každé závislosti |

## SQLite audit_log.db — 3 tabulky

tabulka dotazy:
id INTEGER PRIMARY KEY, timestamp TEXT, text_dotazu TEXT, text_odpovedi TEXT, citovane_zdroje TEXT (JSON), hodnoceni INTEGER (1=palec nahoru / 0=palec dolu), oddeleni_uzivatele TEXT, doba_odpovedi_ms INTEGER

tabulka dokumenty:
id INTEGER PRIMARY KEY, timestamp TEXT, nazev_souboru TEXT, sharepoint_url TEXT, sha256_hash TEXT, datum_modifikace TEXT, pocet_chunku INTEGER, stav TEXT (OK / chyba)

tabulka chyby:
id INTEGER PRIMARY KEY, timestamp TEXT, typ_chyby TEXT, modul TEXT, detail_chyby TEXT, soubor_nebo_dotaz TEXT, vyresen INTEGER (0/1)

## Průřezové komponenty

| Komponenta | Soubor | Iterace | Popis | Knihovny |
|---|---|---|---|---|
| .env Validace | spouští se při startu main.py | ⚠️ 1. iterace | Ověří vyplnění všech hodnot v .env při startu; pokud chybí klíč → app odmítne nastartovat | pydantic-settings BaseSettings, python-dotenv |
| Fallback při výpadku | try/except v rag_chain.py | ⚠️ 1. iterace | Při výpadku Ollamy nebo Qdrantu vrátí uživateli hezkou zprávu + zápis do SQLite tabulky chyby | try/except, sqlite3 |
| Health Check | GET /health v main.py | 2. iterace | Vrátí stav všech závislostí — Qdrant, Redis, Ollama | fastapi, httpx |
| Autentizace | middleware v main.py | 2. iterace | JWT — filtrování směrnic podle oddělení z metadat Qdrantu | python-jose, passlib, fastapi.security |
| Rate Limiting | middleware v main.py | 2. iterace | Omezení dotazů na uživatele za minutu; počítadlo v Redis | slowapi, redis |

## Přehled iterací

1. iterace — MVP:
- Ingest pipeline (SharePoint → Qdrant)
- Chat pipeline (dotaz → odpověď + citace)
- Audit log (SQLite)
- .env validace při startu
- Fallback při výpadku modelu

2. iterace — Rozšíření:
- Hybrid search (vektorový + BM25 fulltext)
- BGE-Reranker-v2-m3 (reranking výsledků)
- ConversationSummaryMemory (konverzační paměť)
- JWT autentizace (filtrování podle oddělení)
- GET /health endpoint
- Rate limiting (slowapi + Redis)
- Upgrade na Mixtral 8x7B (volitelně)

---

# Struktura výstupního dokumentu — co musí výstup obsahovat

## 1. Hlavička
Název projektu, verze dokumentu (v1.0), datum, autor, stav (Draft / Schváleno)

## 2. Přehled architektury
- Princip RAG v max 100 slovech — pro vývojáře kteří s RAG dosud nepracovali
- ASCII diagram znázorňující tok dat: SharePoint → Ingest Pipeline → Qdrant → Chat Pipeline → Uživatel
- Kritické pravidlo embeddinků: stejný model při ingestu i dotazu — co se stane při porušení
- Vysvětlení role každé infrastrukturní komponenty: Qdrant, Redis, Ollama, FastAPI, Celery

## 3. Sumarizace technologií a modelů
Tři oddělené tabulky:

Tabulka A — AI modely: Model | Typ | Účel v systému | HW požadavky | Iterace
Tabulka B — Frameworky a knihovny: Knihovna | Účel | Iterace
Tabulka C — Infrastruktura: Komponenta | Port | Způsob spuštění | Iterace

## 4. Hardwarové a systémové požadavky
Tabulka: Komponenta | Minimální požadavek | Doporučené | Poznámka
Zahrnout: OS (Windows 10+/Linux/macOS), RAM, GPU VRAM, disk, Python verze, Docker verze
Uvést obě varianty: s GPU (5–10 s odpověď) vs bez GPU (30–60 s odpověď)

## 5. Instalace prostředí — krok po kroku
Každý příkaz v samostatném bash code blocku. Pořadí je závazné.

5.1 — Docker Desktop: odkaz ke stažení, příkaz pro ověření instalace
5.2 — Python 3.11+: instalace, ověření verze, vytvoření virtualenv, aktivace (Windows i Linux/macOS)
5.3 — Git a VS Code: instalace, ověření
5.4 — Ollama: instalace (winget pro Windows / brew pro macOS / curl pro Linux), spuštění ollama serve, stažení modelů (ollama pull mistral, ollama pull nomic-embed-text), ověření na portu 11434
5.5 — Struktura projektu: kompletní mkdir a touch příkazy pro vytvoření celé složkové struktury
5.6 — pip závislosti: kompletní requirements.txt se všemi knihovnami 1. iterace, pip install -r requirements.txt
5.7 — Docker Compose: kompletní obsah docker-compose.yml, docker compose up -d, ověření portů 6333 a 6379
5.8 — Spuštění systému: pořadí spuštění všech komponent s příkazy, ověření že vše běží na správných portech

## 6. Konfigurace .env
Kompletní .env soubor se všemi proměnnými, ukázkovými hodnotami a komentáři ke každé proměnné.
Proměnné: SHAREPOINT_CLIENT_ID, SHAREPOINT_CLIENT_SECRET, SHAREPOINT_TENANT_ID, SHAREPOINT_SITE_URL, QDRANT_URL, REDIS_URL, OLLAMA_URL, OLLAMA_MODEL, OLLAMA_EMBED_MODEL, LOG_LEVEL
⚠️ Upozornění: .env nesmí být na GitHubu — přidat do .gitignore
Ukázka .env.example s prázdnými hodnotami

## 7. Implementační plán — 1. iterace
Tabulka: Pořadí | Soubor | Co implementovat | Závislosti (musí existovat před tímto krokem) | Odhadovaná náročnost (hodiny)
Pořadí musí být: utils/ soubory nejdříve → pak core/ orchestrátoři → nakonec main.py
⚠️ Vysvětlit proč nelze začít s orchestrátorem (ingest.py, rag_chain.py)

## 8. Implementační plán — 2. iterace
Stejná tabulková struktura jako bod 7, ale pouze pro komponenty 2. iterace.
Zahrnout: reranker.py, hybrid search v rag_chain.py, Memory Manager v rag_chain.py, autentizace v main.py, health check v main.py, rate limiting v main.py

## 9. Detailní popis každého modulu
Pro každý soubor v app/utils/, app/core/ a app/main.py samostatná subsekce s těmito položkami:
- Účel: 1 věta
- Vstup: co přijímá (typ, formát)
- Výstup: co vrací (typ, formát)
- Klíčové knihovny s ukázkou importů
- Kritická rozhodnutí a pasti (označit ⚠️)
- Poznámky k implementaci

## 10. FastAPI endpointy — detailní specifikace
Pro každý endpoint (POST /ingest/sync, POST /chat, GET /health):
- HTTP metoda + URL
- Popis účelu
- Request body (JSON schéma s příkladem)
- Response body (JSON schéma s příkladem)
- Kdo volá a kdy
- Příklad volání přes curl

## 11. SQLite schéma
Kompletní CREATE TABLE příkazy pro všechny 3 tabulky se správnými datovými typy a komentáři u každého sloupce.
Vysvětlení: proč SQLite a ne PostgreSQL (stdlib, bez instalace, pro audit log dostačující, jednoduché na backup).
Ukázka SELECT dotazů pro 3 nejčastější analýzy: nejčastější dotazy, průměrná doba odpovědi, přehled chybovosti.

## 12. Testování a ověření funkčnosti
12.1 — Ověření infrastruktury: curl/docker příkazy pro ověření každého portu (6333, 6379, 11434, 8000)
12.2 — Ověření Ollamy: ollama list, testovací dotaz přes curl na port 11434
12.3 — První ingest: jak spustit přes FastAPI /docs, co zkontrolovat v SQLite po ingestu, jak ověřit že chunky jsou v Qdrantu
12.4 — První chat dotaz: příklad curl na POST /chat, očekávaný formát JSON odpovědi s citacemi
12.5 — Ukázkové testovací dotazy v češtině (5 příkladů různé složitosti)

## 13. Troubleshooting
Tabulka: Symptom | Pravděpodobná příčina | Diagnostický příkaz | Řešení
Povinné záznamy:
- Ollama neodpovídá na portu 11434
- Model se nevejde do VRAM — fallback na CPU
- Qdrant vrací prázdné výsledky po ingestu
- Chat vrací prázdnou odpověď nebo halucinuje
- Celery nepřijímá úlohy z Redisu
- SHA-256 hash se nemění ale soubor byl aktualizován na SharePointu
- .env validace selže při startu aplikace

## 14. Bezpečnostní checklist
Každá položka označena ✓ (hotovo) nebo ✗ (nutno dořešit) s popisem co zkontrolovat:
- .env není na GitHubu
- data/raw/ není na GitHubu
- audit_log.db není na GitHubu
- .gitignore obsahuje všechny citlivé soubory
- Ollama není přístupná mimo localhost (port 11434 pouze interní)
- Qdrant není přístupný mimo localhost (port 6333 pouze interní)
- Fallback při výpadku implementován a otestován
- Rate limiting nastaven (2. iterace)
- JWT autentizace implementována (2. iterace)

---

# Požadavky na výstup

- Jazyk: čeština; technické názvy knihoven, příkazy, názvy souborů a proměnných ponech v angličtině
- Formát: Markdown — nadpisy (#, ##, ###), tabulky, code blocky (```python, ```bash, ```sql, ```yaml)
- Délka: bez omezení — dokument musí být kompletní, vývojář nesmí potřebovat žádný jiný zdroj
- Každý bash/python příkaz musí být v samostatném code blocku a musí být funkční
- Pasti a kritická rozhodnutí označovat ⚠️ nebo > **Pozor:**
- Tabulky musí být konzistentní — stejné sloupce v rámci každé sekce
- Drž se přesně uvedeného technického stacku — nenahrazuj knihovny jinými, nic si nevymýšlej
```
