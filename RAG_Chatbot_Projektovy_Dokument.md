# AI RAG Chatbot nad interními směrnicemi společnosti

**Typ dokumentu:** Projektový dokument
**Verze:** v1.0
**Datum:** Březen 2026
**Stav:** Draft — k internímu schválení
**Klasifikace:** Interní — Důvěrné

---

## 1. Executive Summary

Projekt řeší chronicky neefektivní vyhledávání informací v interních dokumentech. Zaměstnanci v současném stavu ručně procházejí směrnice uložené v systému Pinya, což je časově náročné a chybové. Navrhovaný systém implementuje architekturu RAG (Retrieval-Augmented Generation) — AI chatbot, který na přirozenou otázku vyhledá relevantní pasáže dokumentů a vygeneruje odpověď s přímou citací zdrojového souboru.

Klíčová vlastnost systému je 100% lokální provoz: LLM model (Mistral 7B via Ollama), embeddingy (nomic-embed-text) i vektorová databáze (Qdrant) běží výhradně na firemní infrastruktuře. Žádná firemní data neopouštějí síť — systém je navržen v souladu s GDPR bez závislosti na externích API službách a bez průběžných licenčních poplatků za AI modely.

Vývoj je rozplánován do dvou iterací. První iterace (MVP) pokrývá kompletní ingest pipeline ze SharePointu do vektorové databáze a chat pipeline s auditním logem. Druhá iterace rozšiřuje systém o hybrid search, reranking výsledků, konverzační paměť a JWT autentizaci s filtrováním dokumentů podle oddělení.

---

## 2. Problém a cíl projektu

### 2.1 Současný stav

Interní směrnice a procesní dokumenty jsou uloženy v systému Pinya. Zaměstnanec, který potřebuje odpověď na konkrétní otázku, musí:

- Ručně vyhledat nebo odhadnout správný dokument v systému Pinya.
- Dokument otevřít v prohlížeči nebo stáhnout.
- Manuálně procházet obsah a hledat relevantní pasáž.
- Opakovat proces pro více dokumentů, pokud první nenalezne odpověď.

Tento postup je pomalý (typicky 5–15 minut na dotaz), frustrující a vede k tomu, že zaměstnanci pracují se zastaralými informacemi nebo obchází procesy z neznalosti.

### 2.2 Cílový stav

Po implementaci zaměstnanec položí dotaz v přirozené češtině (např. *„Jaká je lhůta pro podání dovolené?"*) a do několika sekund (GPU server) nebo do 60 sekund (CPU server) obdrží odpověď přímo citující relevantní pasáže dokumentů. Každá odpověď obsahuje název souboru, číslo stránky a přímý odkaz na SharePoint.

**Měřitelné cíle projektu:**

- Zkrácení průměrného času na nalezení informace z 5–15 minut na méně než 60 sekund.
- 100 % odpovědí doplněných citací zdrojového dokumentu.
- Auditní log zachycuje každý dotaz — měřitelná míra využití a přesnosti.
- Nulový únik firemních dat díky lokálnímu provozu.

---

## 3. Cílová skupina

### 3.1 Uživatelé systému

Primární uživatelé jsou zaměstnanci všech oddělení, kteří pracují s interními směrnicemi a procesními dokumenty — HR, finance, compliance, provoz. Přistupují k systému přes webový chat interface, který nevyžaduje žádné technické znalosti.

V druhé iteraci bude přístup řízen JWT autentizací; dokumenty budou filtrovány podle oddělení tak, aby zaměstnanec viděl pouze dokumenty relevantní pro jeho roli.

### 3.2 Správa systému

Systém spravuje IT admin. Jeho odpovědnosti zahrnují:

- Správu SharePoint konektoru a plánování synchronizace.
- Monitoring Ollama, Qdrant, Redis a FastAPI kontejnerů (Docker).
- Přidávání nových typů dokumentů do ingest pipeline.
- Revizi auditního logu a sledování kvality odpovědí.

---

## 4. Funkční popis systému

### 4.1 Pohled uživatele

Uživatel zadá dotaz v přirozené češtině. Systém odpoví strukturovanou odpovědí, která obsahuje:

- Přímou odpověď na dotaz generovanou z obsahu dokumentů.
- Citaci zdroje: název souboru, číslo stránky, odkaz na SharePoint.
- Upozornění, pokud relevantní informace nebyla v dostupných dokumentech nalezena.

Systém neodpovídá z obecných znalostí AI modelu — Answer Agent má striktní instrukci odpovídat výhradně na základě načtených dokumentů.

### 4.2 Ingest Pipeline (zpracování dokumentů)

Pipeline probíhá asynchronně přes Celery + Redis a zahrnuje tyto kroky:

1. **SharePoint Connector** — stáhne nové nebo změněné PDF soubory (SHA-256 hash porovnání).
2. **PDF Parser** (pymupdf4llm) — extrahuje text lokálně se zachováním tabulek a nadpisů.
3. **Text Cleaner** — odstraní záhlaví, zápatí pomocí RegEx.
4. **Metadata Agent** (Mistral 7B / Ollama) — extrahuje název, oddělení, platnost, verze, cílová skupina; výstup validován přes Pydantic.
5. **Summary Agent** (Mistral 7B / Ollama) — generuje 1–2 větné shrnutí dokumentu.
6. **Text Chunker** — rozdělí text na bloky ~1 000 znaků s přesahem 150 znaků.
7. **Embedder** (nomic-embed-text / Ollama) — převede každý chunk na vektorovou reprezentaci.
8. **Indexer** — uloží vektor + metadata + shrnutí do Qdrant a zaznamená záznam do SQLite.

### 4.3 Chat Pipeline (odpovídání na dotazy)

1. **Query Embedder** — převede dotaz uživatele na vektor stejným modelem jako při ingestu (nomic-embed-text) — kritická konzistence.
2. **Retrieval** — vyhledá top 10 nejrelevantnějších chunků z Qdrantu pomocí kosinové podobnosti.
3. **Answer Agent** (Mistral 7B / Ollama, LangChain prompt) — vygeneruje odpověď výhradně z načtených chunků.
4. **Citation Builder** — sestaví citace z metadat Qdrantu: název souboru, stránka, odkaz na SharePoint.
5. **Audit log** — zapíše dotaz, odpověď, zdroje a dobu odpovědi do SQLite.

---

## 5. Klíčové vlastnosti a přínosy

| Vlastnost | Popis | Přínos pro firmu |
|---|---|---|
| 100% lokální provoz | LLM, embeddingy i vektorová DB běží na firemním serveru — žádná data neopouštějí infrastrukturu | Soulad s GDPR, nulové riziko úniku dat, žádné API poplatky |
| Citace zdrojů | Každá odpověď obsahuje název souboru, číslo stránky a přímý odkaz na SharePoint | Zaměstnanec vidí zdroj, může ověřit informaci a přejít na originální dokument |
| SHA-256 synchronizace | SharePoint connector porovnává hash souborů — stahují se pouze nové nebo změněné dokumenty | Efektivní ingest bez zbytečného přepočítávání embeddingů; škálovatelné pro stovky souborů |
| Audit log (SQLite) | Každý dotaz, odpověď, zdroje, hodnocení uživatele a doba odpovědi jsou logované | Transparentnost, možnost analýzy využití, dohledatelnost výsledků |
| Rychlost odpovědi | Vektorové vyhledávání (top 10) + Mistral 7B inference lokálně; odpověď typicky do 10 s (GPU) | Výrazné zkrácení doby hledání oproti manuálnímu procházení PDF |
| Rozšiřitelnost | 2. iterace přidá hybrid search (BM25 + vektorový), reranking (BGE-Reranker-v2-m3) a JWT autentizaci | Připraveno na rozšíření o nové typy dokumentů, oddělení a pokročilé vyhledávání |

---

## 6. Technický stack

| Vrstva | Technologie | Poznámka |
|---|---|---|
| LLM inference | Ollama — Mistral 7B (`mistral:latest`) | Lokální, CPU/GPU |
| Embedding model | OllamaEmbeddings / `nomic-embed-text` | Musí být shodný při ingestu i dotazu |
| Vektorová databáze | Qdrant (Docker, port 6333) | Persistentní uložení vektorů + metadat |
| Backend API | Python 3.11+ / FastAPI / uvicorn (port 8000) | REST API, async |
| Task queue | Celery + Redis (Docker, port 6379) | Asynchronní ingest pipeline |
| PDF parsing | pymupdf4llm | Lokální, zachovává tabulky a nadpisy |
| Orchestrace | LangChain + langchain-ollama + langchain-qdrant | Prompt management, pipeline flow |
| Audit a logování | SQLite (`audit_log.db`, 3 tabulky) | dotazy, dokumenty, chyby |
| SharePoint konektor | office365-rest-python-client + MS Graph API | OAuth 2.0 přístup k SharePointu |

Celá architektura je navržena jako 100% lokální řešení s nulovými průběžnými náklady na AI služby. Všechny komponenty jsou provozovány v Dockeru nebo nativně na firemním serveru.

---

## 7. Bezpečnost a ochrana dat

### 7.1 Lokální provoz — nulový únik dat

Veškeré zpracování probíhá výhradně na firemní infrastruktuře. Ollama hostuje Mistral 7B a nomic-embed-text lokálně, Qdrant ukládá vektory lokálně, FastAPI backend běží na interním serveru. Systém nevyžaduje připojení k externím AI API (OpenAI, Anthropic, Google apod.) ani přenos dat mimo firemní síť.

### 7.2 Auditní log

SQLite databáze (`audit_log.db`) zaznamenává v reálném čase:

- Každý dotaz uživatele s časovým razítkem.
- Vygenerovanou odpověď a seznam citovaných zdrojů.
- Hodnocení odpovědi uživatelem.
- Dobu zpracování dotazu.
- Chybové události v pipeline.

Audit log umožňuje retrospektivní analýzu kvality odpovědí, identifikaci chybějících dokumentů a sledování vzorců využití.

### 7.3 Plánovaná autentizace (2. iterace)

Druhá iterace implementuje JWT autentizaci s přihlašováním uživatelů. Dokumenty budou tagované podle oddělení v metadatech; Qdrant Retrieval bude filtrovat výsledky tak, aby zaměstnanec viděl pouze dokumenty své organizační jednotky. Rate limiting (slowapi + Redis) ochrání backend před zneužitím.

---

## 8. Rozsah projektu

### 8.1 1. iterace — MVP

| Komponenta | Popis |
|---|---|
| SharePoint Connector | SHA-256 synchronizace, stahování PDF |
| Ingest Pipeline | PDF Parser → Text Cleaner → Metadata Agent → Summary Agent → Chunker → Embedder → Qdrant |
| Chat Pipeline | Query Embedder → Qdrant Retrieval (top 10) → Answer Agent → Citation Builder |
| LLM + Embeddingy | Ollama: Mistral 7B (`mistral:latest`) + `nomic-embed-text` |
| Backend API | FastAPI + uvicorn (port 8000), REST endpoint `/chat` |
| Asynchronní zpracování | Celery + Redis (Docker, port 6379) |
| Audit log | SQLite (`audit_log.db`) — dotazy, dokumenty, chyby |
| Validace konfigurace | `.env` validace při startu, fallback při výpadku modelu |

### 8.2 2. iterace — Rozšíření

| Komponenta | Popis |
|---|---|
| Hybrid Search | Vektorové vyhledávání + BM25 fulltext (kombinované skóre) |
| Reranking | BGE-Reranker-v2-m3 — přeřazení top výsledků před generováním odpovědi |
| Konverzační paměť | ConversationSummaryMemory — udržuje kontext více dotazů |
| Autentizace | JWT — přihlášení uživatelů, filtrování dokumentů podle oddělení |
| Health check | `GET /health` endpoint — monitoring dostupnosti |
| Rate limiting | slowapi + Redis — omezení počtu dotazů na uživatele |

### 8.3 Mimo rozsah

- Podpora jiných formátů než PDF (Word, Excel) — možné rozšíření v budoucích iteracích.
- Integrace s externími AI API (cloud LLM) — záměrně vyloučeno z bezpečnostních důvodů.
- Mobilní nativní aplikace — přístup přes webový prohlížeč.
- Automatická tvorba nebo úprava interních dokumentů.
- Vícejazyčná podpora — v aktuální iteraci pouze čeština.

---

## 9. Předpoklady a závislosti

### 9.1 Hardware

- **GPU server (doporučeno):** NVIDIA GPU s min. 8 GB VRAM pro Mistral 7B — typická doba odpovědi 5–10 s.
- **CPU server (minimum):** 16 GB RAM, moderní vícejádrový procesor — typická doba odpovědi 30–60 s.
- **Úložiště:** min. 50 GB pro Ollama modely, Qdrant data a PDF soubory.

### 9.2 Software

- Python 3.11+ s pip závislostmi (FastAPI, LangChain, Celery, Pydantic, pymupdf4llm, office365-rest-python-client).
- Docker Desktop — kontejnery pro Qdrant (port 6333) a Redis (port 6379).
- Ollama — lokální runner pro Mistral 7B a nomic-embed-text.
- Git pro správu verzí kódu a konfigurací.

### 9.3 Přístup a oprávnění

- SharePoint credentials s oprávněním pro MS Graph API (OAuth 2.0 app registration v Azure AD).
- Přístup k cílovým SharePoint knihovnám s interními dokumenty.
- Síťová konektivita mezi ingest serverem a SharePointem.
- SharePoint musí být před spuštěním pipeline ručně doplněn PDF dokumenty — ingest pipeline předpokládá, že soubory jsou na SharePointu k dispozici.

---

## 10. Otevřené otázky

1. **Frekvence synchronizace SharePointu:** Jak často má ingest pipeline kontrolovat nové nebo změněné dokumenty? Návrh: nočně (cron job). Je tato frekvence dostačující, nebo je potřeba real-time webhook notifikace ze SharePointu?

2. **Správa přístupových práv pro MS Graph API:** Kdo je zodpovědný za vytvoření a správu Azure AD app registration? Jaká minimální oprávnění jsou potřebná (`Sites.Read.All`, `Files.Read.All`)?

3. **Odpovědný IT admin:** Kdo bude mít na starost provoz, monitoring a aktualizace systému po předání? Je potřeba dokumentace pro provoz (runbook)?

4. **Monitoring a alerting:** Jaký nástroj bude použit pro monitoring dostupnosti (Qdrant, Ollama, FastAPI)? Postačuje `GET /health` endpoint + e-mailový alert, nebo je potřeba integrace s existujícím monitoring nástrojem (Grafana, Zabbix)?

5. **Scope dokumentů pro 1. iteraci:** Které konkrétní SharePoint knihovny a složky budou součástí prvního ingestu? Je potřeba indexovat všechny dokumenty, nebo pouze vybrané kategorie (např. pouze směrnice HR a Finance)?

---

*Interní dokument — Důvěrné | AI RAG Chatbot v1.0*
