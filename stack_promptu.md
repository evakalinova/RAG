# Stack promptů — AI RAG Chatbot

---

## PROMPT 1 — Projektový dokument

**[1]** ahoj, potřebuju pomoct sestavit prompt pro generování projektového dokumentu. jde o AI chatbota na firemní směrnice, běží to lokálně na Ollama, Qdrant a FastAPI. dokument bude pro vedení a IT management, potřebujeme schválení projektu. tady máš schéma systému co jsem navrhla, koukni do něj, samozřejmě musí zahrnovat shrnutí, cíle, přínosy a další. postupně budeme skládat výsledný prompt, navrhni kostru a doplň ji

**[2]** jo to vypadá dobře, ale přidej tam že musí být executive summary na začátku, krátké, ať je to o přínosu pro firmu a ne jen technický popis

**[3]** tohle executive summary pořád není ono, je to moc technické. chci aby to vypadalo nějak takhle: projekt zavádí AI chatbota který zaměstnancům umožní najít informaci z interních směrnic do 60 sekund místo dosavadních 5-15 minut. systém běží 100% lokálně, žádná firemní data neopustí infrastrukturu... atd. takhle nějak to chci, uprav tu instrukci

**[4]** ok a musí tam být taky sekce s měřitelnými cíli a přínosy, konkrétní čísla, kolik času se ušetří, že každá odpověď musí mít citaci zdroje a podobně

**[5]** pipeline má víc kroků než tam máš, zasílám schéma znovu, projdi ingest pipeline krok po kroku a doplň to celé. chat pipeline taky není kompletní, chybí tam citation builder a audit log, doplň to

**[6]** jo tohle už sedí se schématem, přidej ještě tabulku klíčových vlastností a přínosů, vlastnost, popis, přínos pro firmu. chci tam lokální provoz, citace zdrojů, audit log a rychlost odpovědi

**[7]** čtenáři jsou IT manažeři s technickým backgroundem takže odborné termíny jsou v pohodě, přidej to do promptu ať to tam zbytečně nevysvětluje

**[8]** důležitá věc kterou jsem zapomněla říct, směrnice nejsou na SharePointu ale v systému Pinya. do SharePointu se to musí ručně přesunout, žádné propojení těch dvou systémů není. oprav to v promptu

**[9]** tohle jsi neopravil dobře, pořád tam píše že zaměstnanci procházejí dokumenty na SharePointu, to není pravda. v současném stavu jsou dokumenty v Pinyi, oprav to

**[10]** ok a ještě do předpokladů přidej že před spuštěním je nutné ručně přesunout dokumenty z Pinya na SharePoint, bez toho to nemůže fungovat

**[11]** do otevřených otázek dej otázku kdo tohle bude řešit a jestli jednorázově nebo průběžně

**[12]** přidej sekci rozsah projektu, první iterace MVP a druhá iterace rozšíření, a co vůbec není v rozsahu

**[13]** přidej sekci cílová skupina, kdo to používá a kdo to spravuje

**[14]** uprav prompt aby to generovalo stručněji

**[15]** přidej omezení 600 až 1200 slov

**[16]** vypadá to ok, zkontroluj jestli je to celé v souladu se schématem, hlavně iterace a technický stack

**[17]** ok pošli mi celý prompt jako jeden čistý celek v md ke stažení

---

### Výsledný prompt 1

```
Jsi zkušený technický projektový manažer, business analytik a solution architect.
Tvým úkolem je vytvořit profesionální projektový dokument ve formátu Markdown v češtině.

Dokument je určen pro vedení firmy, IT management a technické vedoucí pracovníky.
Čtenáři jsou technicky zdatní — IT manažeři a vedoucí pracovníci s technickým backgroundem.
Používej technickou terminologii bez vysvětlování základních pojmů (pipeline, embedding, endpoint, REST API, vektorová databáze apod.).

Tvůj výstup musí být strukturovaný, přehledný, profesionální a musí přesně vycházet z architektury uvedené níže.
Nezjednodušuj technologii a nic si nevymýšlej mimo uvedený kontext.

---

# Kontext projektu

Jedná se o interní firemní projekt:

**AI RAG Chatbot nad interními směrnicemi společnosti**

Cílem projektu je umožnit zaměstnancům rychlé vyhledávání informací z interních dokumentů pomocí AI chatbota.

Dokumenty jsou primárně:
- interní směrnice
- procesní dokumenty
- případně další dokumenty (faktury, smlouvy)

Dokumenty jsou uloženy v systému Pinya. Před spuštěním ingest pipeline je nutné je ručně exportovat na SharePoint — žádné automatické propojení mezi Pinyou a SharePointem neexistuje. Ingest pipeline následně stahuje dokumenty ze SharePointu přes MS Graph API.

V současném stavu zaměstnanci:
- hledají informace ručně
- otevírají jednotlivé PDF dokumenty
- procházejí obsah manuálně

To je pomalé a neefektivní.

---

# Navrhované řešení

Navrhované řešení je AI chatbot využívající architekturu RAG (Retrieval-Augmented Generation).

Princip fungování:
1. systém vyhledá relevantní pasáže dokumentů
2. na jejich základě vytvoří odpověď
3. odpověď vždy obsahuje citaci zdrojového dokumentu (název souboru, stránka, odkaz na SharePoint)

Uživatel položí otázku → dostane odpověď → vidí zdroj.

---

# Architektura systému

Systém je rozdělen na dvě hlavní pipeline:

## Ingest Pipeline (zpracování dokumentů)

1. SharePoint Connector — synchronizace PDF, SHA-256 hash porovnání (stahují se pouze nové nebo změněné soubory)
2. PDF Parser — lokální extrakce textu a struktur z PDF pomocí pymupdf4llm (zachovává tabulky a nadpisy)
3. Text Cleaner — RegEx čištění záhlaví, zápatí
4. Metadata Agent — Ollama/Mistral 7B extrahuje: název, oddělení, platnost, verze, pro koho platí; Pydantic validace
5. Summary Agent — Ollama/Mistral 7B generuje 1–2 větné shrnutí dokumentu
6. Text Chunker — rozdělení na bloky ~1000 znaků s přesahem 150 znaků
7. Document Builder — LangChain Document objekt: chunk + metadata + shrnutí
8. Embedder — OllamaEmbeddings/nomic-embed-text převede text na vektor
9. Indexer — uložení do Qdrant + záznam do SQLite

## Chat Pipeline (odpovídání na dotazy)

1. Query Embedder — stejný model jako při ingestu (nomic-embed-text) — kritické pravidlo
2. Retrieval — vektorové vyhledávání v Qdrantu, top 10 výsledků
3. Answer Agent — ChatOllama/Mistral 7B, LangChain prompt se striktním pokynem: odpovídej POUZE z dokumentů
4. Citation Builder — metadata z Qdrantu, sestavení odkazů na SharePoint
5. Audit log — zápis dotazu, odpovědi a zdrojů do SQLite

---

# Technologie

Architektura je navržena jako 100% lokální řešení — žádná data neopouštějí firemní infrastrukturu.

- LLM + embeddingy: Ollama — Mistral 7B (mistral:latest), embedding model nomic-embed-text
- Vektorová databáze: Qdrant (Docker, port 6333)
- Backend: Python 3.11+ + FastAPI + uvicorn (port 8000)
- Asynchronní zpracování: Celery + Redis (Docker, port 6379)
- PDF parsing: pymupdf4llm (lokální, zachovává tabulky)
- Orchestrace pipeline: LangChain + langchain-ollama + langchain-qdrant
- Audit a logování: SQLite (audit_log.db, 3 tabulky: dotazy, dokumenty, chyby)
- SharePoint konektor: office365-rest-python-client + MS Graph API

---

# Bezpečnost

- Žádná firemní data neopouštějí infrastrukturu — model, embeddingy i databáze běží lokálně
- Audit log zachycuje každý dotaz, odpověď, zdroje, hodnocení uživatele a dobu odpovědi
- 2. iterace: autentizace uživatelů (JWT), filtrování dokumentů podle oddělení

---

# Iterace vývoje

## 1. iterace — MVP
- Ingest pipeline (SharePoint → Qdrant)
- Chat pipeline (dotaz → odpověď + citace)
- Audit log (SQLite)
- .env validace při startu
- Fallback při výpadku modelu

## 2. iterace — Rozšíření
- Hybrid search (vektorový + BM25 fulltext)
- BGE-Reranker-v2-m3 (reranking výsledků)
- Konverzační paměť (ConversationSummaryMemory)
- Autentizace uživatelů (JWT, filtrování podle oddělení)
- Health check endpoint GET /health
- Rate limiting (slowapi + Redis)

---

# Struktura dokumentu

## 1. Hlavička dokumentu
- Název projektu, verze (v1.0), datum, stav dokumentu

## 2. Executive Summary (max 150 slov)
- Co projekt řeší, jaký má přínos pro firmu
- Proč lokální AI systém (ochrana dat, GDPR, žádné API poplatky)
- Dvě iterace vývoje

## 3. Problém a cíl projektu
- Současný stav a jeho problémy
- Cílový stav po implementaci
- Měřitelné cíle (čas hledání, přesnost, citace zdrojů)

## 4. Cílová skupina
- Kdo systém používá (zaměstnanci, oddělení)
- Kdo systém spravuje (IT admin)
- Způsob přístupu

## 5. Funkční popis systému
- Popis z pohledu uživatele
- Co uživatel vidí: dotaz → odpověď → zdroj s odkazem na SharePoint
- Stručný popis co se děje na pozadí (pro IT management)

## 6. Klíčové vlastnosti a přínosy
Tabulka: Vlastnost | Popis | Přínos pro firmu
Zahrnout: lokální provoz, citace zdrojů, SHA-256 synchronizace, audit log, rychlost, rozšiřitelnost

## 7. Bezpečnost a ochrana dat
- Lokální provoz — žádný únik dat
- Audit log — co se loguje a proč
- Plánovaná autentizace podle oddělení (2. iterace)

## 8. Rozsah projektu
- Co je v rozsahu 1. iterace
- Co je v rozsahu 2. iterace
- Co není v rozsahu

## 9. Předpoklady a závislosti
- Hardware: server s GPU 8GB VRAM (Mistral 7B), nebo bez GPU (odpovědi 30–60 s)
- Software: Python 3.11+, Docker Desktop, Ollama, VS Code, Git
- Přístup: SharePoint credentials (MS Graph API)
- Před spuštěním: ruční export dokumentů z Pinya na SharePoint

## 10. Otevřené otázky
3–5 konkrétních otázek které je třeba zodpovědět před spuštěním — včetně: kdo zajistí přesun dokumentů z Pinya na SharePoint, jestli jednorázově nebo průběžně, kdo bude správce systému, jak bude řešen monitoring

---

# Požadavky na výstup

- Jazyk: čeština, profesionální styl
- Formát: Markdown (# ## ###, tabulky, odrážky)
- Délka: 600–1200 slov (bez hlavičky a tabulek)
- Každá sekce musí být srozumitelná samostatně
- Orientace na přínos pro firmu
- Drž se přesně uvedeného technického stacku — nic nevymýšlej
```

---

## PROMPT 2 — Technicky-realizační plán

**[1]** potřebuju druhý prompt, na technicky-realizační plán pro vývojový tým. zasílám schéma. prompt musí vygenerovat dokument, kde vývojář najde úplně všechno, od instalace až po testování. začni lehce s návrhem a postupně budeme připojovat a upravovat další části, máš tady schéma, které jsem vytvořila, na základě toho to postav

**[2]** jo, to by pro začátek šlo, hele ve schématu vidím konkrétní názvy souborů a kroky pipeline ale v promptu to tak konkrétní není, můžeš to doplnit podle schématu? a přidej tam i seznam těch knihoven které jsou ve schématu, ty teď přidáš do promptu přesné pokyny a části, které má výsledně prompt vytvořit

**[3]** tady má prompt co jsem dělala v GPT, je tvůj lepší nebo to sloučíme dohromady?

**[4]** 1) jo, sloučit

**[5]** hele ve schématu u každého souboru vidím že tam jsou různé důležité věci, třeba u toho sharepoint.py je tam něco o porovnávání souborů a u metadata_agent.py něco o validaci. přidej že u každého souboru musí být popsané tyhle důležité věci, co může vývojáře překvapit

**[6]** přidej jak to nainstalovat krok po kroku, každý příkaz zvlášť ať to může vývojář zkopírovat. a přidej že pořadí je důležité, Docker musí být první

**[7]** hele tohle není dobře, některé věci jsou tam v první části ale ve schématu jsou označené jako rozšíření takže patří až do druhé části. zasílám schéma znovu, oprav to podle toho

**[8]** pořád to není úplně správně, projdi to ještě jednou a zkontroluj že všechno co je ve schématu jako rozšíření je skutečně až ve druhé části

**[9]** přidej testování, jak vývojář ověří že to celé funguje, nějaké konkrétní příkazy a pár ukázkových dotazů v češtině

**[10]** přidej co dělat když něco nefunguje, třeba Ollama neodpovídá nebo Qdrant vrací prázdné výsledky, a taky bezpečnostní checklist co musí vývojář zkontrolovat před nasazením

**[11]** vypadá to dobře, pošli mi celý prompt ke stažení

---

### Výsledný prompt 2

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
Dokumenty jsou uloženy v systému Pinya — před spuštěním ingest pipeline je nutné je ručně exportovat na SharePoint. Ingest pipeline pracuje výhradně se SharePointem přes MS Graph API.
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
| 09 | qdrant_store.py | Indexer | vektor + Document → uloženo v Qdrantu + SQLite | qdrant-client, langchain-qdrant, sqlite3 |

## Chat Pipeline — kroky, orchestruje rag_chain.py

| Krok | Soubor | Komponenta | Vstup → Výstup | Iterace |
|---|---|---|---|---|
| 01 | rag_chain.py | Query Embedder | text dotazu → vektor dotazu | 1. iterace |
| 02 | rag_chain.py | Retrieval | vektor → top 10 chunků | 1. iterace |
| 02 | rag_chain.py | Hybrid Retrieval | vektor + BM25 + metadata filtr → top 10 chunků | 2. iterace |
| 03 | reranker.py | Reranker | dotaz + 10 chunků → top 3–5 seřazených | 2. iterace |
| 04 | rag_chain.py | Answer Agent | dotaz + top chunky → text odpovědi | 1. iterace |
| 05 | rag_chain.py | Citation Builder | odpověď + metadata chunků → odpověď + citace | 1. iterace |
| 06 | rag_chain.py | Memory Manager | dotaz + shrnutí historie → kontextová odpověď | 2. iterace |
| 07 | sqlite3 | Audit log | finální odpověď → text + zdroje + zápis do SQLite | 1. iterace |

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
5.4 — Ollama: instalace, spuštění ollama serve, stažení modelů (ollama pull mistral, ollama pull nomic-embed-text), ověření na portu 11434
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
Tabulka: Pořadí | Soubor | Co implementovat | Závislosti | Odhadovaná náročnost (hodiny)
Pořadí musí být: utils/ soubory nejdříve → pak core/ orchestrátoři → nakonec main.py
⚠️ Vysvětlit proč nelze začít s orchestrátorem (ingest.py, rag_chain.py)

## 8. Implementační plán — 2. iterace
Stejná tabulková struktura jako bod 7, ale pouze pro komponenty 2. iterace.
Zahrnout: reranker.py, hybrid search v rag_chain.py, Memory Manager v rag_chain.py, autentizace v main.py, health check v main.py, rate limiting v main.py

## 9. Detailní popis každého modulu
Pro každý soubor v app/utils/, app/core/ a app/main.py samostatná subsekce:
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
Vysvětlení proč SQLite a ne PostgreSQL.
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
Každá položka označena ✓ (hotovo) nebo ✗ (nutno dořešit):
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
