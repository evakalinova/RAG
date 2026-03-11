# RAG Chatbot — Prompt pro generování dokumentů

---

## PROMPT 1 — Projektový popis a sumarizace

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
- hledají informace ručně v systému Pinya
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
- Současný stav a jeho problémy — zaměstnanci hledají informace ručně v systému Pinya
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
- Před spuštěním: dokumenty musí být ručně přesunuty z Pinya na SharePoint — ingest pipeline předpokládá že soubory jsou na SharePointu k dispozici

## 10. Otevřené otázky
3–5 konkrétních otázek které je třeba zodpovědět před spuštěním, povinně:
- Kdo zajistí přesun dokumentů z Pinya na SharePoint a jak — jednorázově před spuštěním nebo průběžně při přidávání nových směrnic?
- Frekvence synchronizace SharePointu (návrh: nočně přes cron job)
- Kdo bude zodpovědný IT admin po předání systému?
- Správa přístupových práv pro MS Graph API (Azure AD app registration)
- Monitoring a alerting — postačuje GET /health endpoint nebo je potřeba integrace s existujícím nástrojem?

---

# Požadavky na výstup

- Jazyk: čeština, profesionální styl
- Formát: Markdown (# ## ###, tabulky, odrážky)
- Délka: 600–1200 slov (bez hlavičky a tabulek)
- Každá sekce musí být srozumitelná samostatně
- Orientace na přínos pro firmu
- Drž se přesně uvedeného technického stacku — nic nevymýšlej
```
