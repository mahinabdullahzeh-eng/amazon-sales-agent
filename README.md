# Amazon Sales Intelligence Agent

Platforma zasilana przez AI do analizy produktów Amazon i optymalizacji ofert sprzedaży. System łączy web scraping, wieloagentowe potoki AI oraz ustrukturyzowane przetwarzanie danych w celu badania produktów, analizy słów kluczowych, oceny szans sprzedażowych i generowania zoptymalizowanych pod kątem SEO treści ofert.

---

## 📥 Jak otworzyć i pobrać tę dokumentację

Dokumentacja stworzona przez GitHub Copilot jest dostępna w tym repozytorium jako plik `README.md`. Możesz ją otworzyć i pobrać na kilka sposobów:

### 🌐 Podgląd online (GitHub)
Otwórz w przeglądarce — GitHub renderuje plik Markdown automatycznie:

```
https://github.com/mahinabdullahzeh-eng/amazon-sales-agent/blob/copilot/analyze-amazon-agent-functions/README.md
```

### ⬇️ Pobranie samego pliku README.md (plik tekstowy)
Kliknij prawym przyciskiem myszy na poniższy link i wybierz „Zapisz link jako…":

```
https://raw.githubusercontent.com/mahinabdullahzeh-eng/amazon-sales-agent/copilot/analyze-amazon-agent-functions/README.md
```

Lub użyj `curl` / `wget` w terminalu:

```bash
curl -O https://raw.githubusercontent.com/mahinabdullahzeh-eng/amazon-sales-agent/copilot/analyze-amazon-agent-functions/README.md
# albo:
wget https://raw.githubusercontent.com/mahinabdullahzeh-eng/amazon-sales-agent/copilot/analyze-amazon-agent-functions/README.md
```

### 📦 Pobranie całego repozytorium (ZIP)
Pobierz wszystkie pliki projektu jako archiwum ZIP:

```
https://github.com/mahinabdullahzeh-eng/amazon-sales-agent/archive/refs/heads/copilot/analyze-amazon-agent-functions.zip
```

### 🖥️ Klonowanie repozytorium (Git)
```bash
git clone https://github.com/mahinabdullahzeh-eng/amazon-sales-agent.git
cd amazon-sales-agent
git checkout copilot/analyze-amazon-agent-functions
# Plik README.md znajdziesz w katalogu głównym projektu
```

> **Wskazówka:** Po scaleniu gałęzi do `main` dokumentacja będzie dostępna bezpośrednio pod adresem `https://github.com/mahinabdullahzeh-eng/amazon-sales-agent`.

---

## Spis treści

1. [Przegląd architektury](#przegląd-architektury)
2. [Poczwórny potok agentów AI](#poczwórny-potok-agentów-ai)
   - [Agent 1 – Agent badawczy (Research Agent)](#agent-1--agent-badawczy-research-agent)
   - [Agent 2 – Agent kategoryzacji słów kluczowych (Keyword Agent)](#agent-2--agent-kategoryzacji-słów-kluczowych-keyword-agent)
   - [Agent 3 – Agent scoringu (z subagentami)](#agent-3--agent-scoringu-z-subagentami)
   - [Agent 4 – Agent optymalizacji SEO](#agent-4--agent-optymalizacji-seo)
3. [System scrapowania Amazon](#system-scrapowania-amazon)
4. [System anty-blokujący](#system-anty-blokujący)
5. [Narzędzia do przetwarzania słów kluczowych](#narzędzia-do-przetwarzania-słów-kluczowych)
6. [System zarządzania zadaniami (Job Manager)](#system-zarządzania-zadaniami-job-manager)
7. [Endpointy API](#endpointy-api)
8. [Aplikacja frontendowa](#aplikacja-frontendowa)
9. [Zewnętrzne API i usługi](#zewnętrzne-api-i-usługi)
10. [Konfiguracja i zmienne środowiskowe](#konfiguracja-i-zmienne-środowiskowe)
11. [Stos technologiczny](#stos-technologiczny)
12. [Wdrożenie](#wdrożenie)

---

## Przegląd architektury

```
Pliki CSV (eksport z Helium 10 Cerebro)
        │
        ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                         Backend FastAPI                                     │
│                                                                              │
│  POST /api/v1/amazon-sales-intelligence  (synchroniczny)                    │
│  POST /api/v1/start-analysis             (asynchroniczne zadanie w tle)     │
│  GET  /api/v1/job-status/{job_id}                                           │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  POTOK 4 AGENTÓW AI                                  │   │
│  │                                                                       │   │
│  │  [1] ResearchAgent   → scraping + pozycja rynkowa + oceny trafności │   │
│  │          ↓                                                            │   │
│  │  [2] KeywordAgent    → kategoryzacja słów kluczowych (6 kategorii)   │   │
│  │          ↓                                                            │   │
│  │  [3] ScoringRunner   → intencja + metryki + wolumen + warianty       │   │
│  │     ├─ IntentScoringSubagent                                         │   │
│  │     ├─ MetricsSubagent (deterministyczny)                            │   │
│  │     ├─ BroadVolumeAgent                                              │   │
│  │     ├─ KeywordVariantAgent                                           │   │
│  │     ├─ RootRelevanceAgent                                            │   │
│  │     └─ OpportunitySubagent                                           │   │
│  │          ↓                                                            │   │
│  │  [4] SEOOptimizationAgent → tytuł + punktory + słowa kluczowe back  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Magazyn danych: Redis (Upstash) lub lokalny system plików                  │
└────────────────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│   Frontend Next.js      │
│   (Panel KeywordAI)     │
└─────────────────────────┘
```

---

## Poczwórny potok agentów AI

Wszystkie agenty są zbudowane na **OpenAI Agents SDK** (`openai-agents>=0.2.9`) i używają modelu `gpt-5-mini-2025-08-07` z minimalnym wysiłkiem rozumowania dla oszczędności kosztów.

### Agent 1 – Agent badawczy (Research Agent)

**Plik:** `backend/app/local_agents/research/`

**Cel:** Scrapuje ofertę produktu na Amazon i produkuje ustrukturyzowane dane rynkowe.

**Co robi:**
1. Scrapuje stronę docelowego produktu za pomocą wielu scraperów (Scrapy, Playwright, fallback SERP API).
2. Scrapuje produkty konkurentów wymienione w przesłanych plikach CSV (konkurenci przychodowi i wzorniczo-projektowi).
3. Oblicza **bazowy wynik trafności (0–10)** dla każdego słowa kluczowego z plików CSV:
   - Wzór: zlicza, ile ASIN-ów konkurentów zajmuje top 10 dla danego słowa kluczowego.
   - Wynik = `(top_10_count / total_asins) × 20`, ograniczony do 10.
4. Stosuje korektę **trafności dosłownej**: zwiększa wynik dla słów kluczowych, których tokeny są zawarte w tytule produktu.
5. Stosuje korektę **trafności tytułów konkurentów**: zwiększa wynik, gdy wielu konkurentów używa danego słowa kluczowego w swoich tytułach.
6. Stosuje **dynamiczny próg trafności** (3–9 w zależności od rozmiaru zbioru danych) do filtrowania słabych słów kluczowych przed przetwarzaniem przez AI.
7. Wysyła zescrapowane dane + uproszczony kontekst CSV do modelu AI `ResearchAgent` w celu ustrukturyzowanej analizy.

**Schemat wyjściowy (`ResearchOutput`):**
- `content_sources` – jakość ekstrakcji tytułu, zdjęć, treści A+, recenzji, sekcji Q&A
- `market_position` – poziom cenowy (budżetowy/premium) wraz z ceną i danymi jednostkowymi
- `main_keyword` – wybrane główne słowo kluczowe z kandydatami i uzasadnieniem
- `current_listing` – istniejący tytuł, punktory, słowa kluczowe backendu

**Zwraca również (dla kolejnych agentów):**
- `scraped_product` – pełne ustrukturyzowane dane zescrapowane
- `base_relevancy_scores` – słowo kluczowe → wynik (0–10), filtrowany dynamicznym progiem
- `full_base_relevancy_scores` – pełne wyniki bez filtrowania
- `adjusted_relevancy_scores` – wyniki po korektach dosłownych i konkurencyjnych
- `keyword_root_analysis` / `priority_roots` – wyniki ekstrakcji rdzeni słów kluczowych
- `ai_keyword_root_analysis` – analiza rdzeni wspierana przez AI (subagent `RootExtractionAgent`)
- `competitor_scrapes` – zescrapowane dane konkurentów (cena, ocena, liczba recenzji)

**Runner:** `backend/app/local_agents/research/runner.py` → `ResearchRunner.run_research()`

---

### Agent 2 – Agent kategoryzacji słów kluczowych (Keyword Agent)

**Plik:** `backend/app/local_agents/keyword/`

**Cel:** Klasyfikuje każde słowo kluczowe do dokładnie jednej z sześciu kategorii.

**Sześć kategorii słów kluczowych:**

| Kategoria | Opis |
|---|---|
| `Relevant` | Podstawowe słowa kluczowe bezpośrednio opisujące produkt w jego bazowej formie |
| `Design-Specific` | Słowa kluczowe opisujące atrybuty/warianty (materiał, rozmiar, kolor, opakowanie) tej samej formy produktu |
| `Irrelevant` | Słowa kluczowe opisujące inną formę produktu lub niemające związku z produktem |
| `Branded` | Słowa kluczowe zawierające jakąkolwiek nazwę marki (własnej lub konkurencji) |
| `Spanish` | Słowa kluczowe w języku hiszpańskim lub innym nieangielskim |
| `Outlier` | Skrajnie ogólne terminy o dużym wolumenie wyszukiwań, obejmujące szeroki wachlarz produktów |

**Jak działa:**
1. Wyodrębnia kontekst produktu: tytuł, markę, bazową formę produktu (plastry/proszek/całe/płyn/kapsułki itp.) ze zescrapowanego produktu.
2. Filtruje słowa kluczowe o zerowej trafności.
3. Dzieli słowa kluczowe na **partie po 75** w celu zapobiegania obcięciu JSON.
4. Dla każdej partii buduje prompt z pełnym kontekstem produktu, regułami wykrywania marek i bazowymi wynikami trafności.
5. Wywołuje `KeywordAgent` (model AI), który stosuje sekwencyjny algorytm kategoryzacji: Marka → Język → Forma produktu → Powiązanie kontekstualne → Outlier → Atrybuty → Wynik końcowy.
6. Łączy wyniki partii i normalizuje nazwy pól.

**Wynik:** `KeywordAnalysisResult` zawierający:
- `items[]` – każde słowo kluczowe z polami `phrase`, `category`, `relevancy_score`
- `stats` – liczby i przykłady dla każdej kategorii
- `product_context` – wyodrębnione metadane produktu

**Runner:** `backend/app/local_agents/keyword/runner.py` → `KeywordRunner.run_keyword_categorization()`

---

### Agent 3 – Agent scoringu (z subagentami)

**Plik:** `backend/app/local_agents/scoring/`

**Cel:** Wzbogaca skategoryzowane słowa kluczowe o metryki, oceny intencji, dane o szerokim wolumenie, analizę wariantów i flagi okazji.

**Subagenty i komponenty:**

#### IntentScoringSubagent
**Plik:** `backend/app/local_agents/scoring/subagents/intent_agent.py`

Ocenia intencję zakupową dla każdego słowa kluczowego w **skali 0–3**:
- `0` – Nieistotne
- `1` – Jeden istotny aspekt
- `2` – Dwa istotne aspekty
- `3` – Trzy istotne aspekty (silna intencja transakcyjna)

Używa modelu `gpt-5-mini-2025-08-07` z zescrapowanym produktem i bazowymi wynikami trafności jako kontekstem.

#### MetricsSubagent (deterministyczny)
**Plik:** `backend/app/local_agents/scoring/subagents/metrics_agent.py`

Wyodrębnia metryki Helium 10 **bezpośrednio z przesłanych plików CSV** bez AI — deterministyczne wyszukiwanie po frazie kluczowej:
- `search_volume` – z kolumny „Search Volume" (wolumen wyszukiwań)
- `title_density` – z kolumny „Title Density" (gęstość w tytułach)
- `cpr` – z kolumny „CPR"
- `competition` – Competing Products, Ranking Competitors, Competitor Rank, Competitor Performance Score

#### BroadVolumeAgent
**Plik:** `backend/app/local_agents/scoring/subagents/broad_volume_agent.py`

Oblicza **zagregowany wolumen wyszukiwań dla każdego rdzenia słowa kluczowego**:
- Identyfikuje token rdzeniowy dla każdego słowa kluczowego (główny rzeczownik, z pominięciem stopwords i marek).
- Sumuje wolumen wyszukiwań dla wszystkich słów kluczowych o tym samym rdzeniu.
- Uwzględnia tylko słowa kluczowe z kategorii `Relevant` i `Design-Specific`.
- Zwraca mapę `broad_search_volume_by_root`.

#### KeywordVariantAgent
**Plik:** `backend/app/local_agents/scoring/subagents/keyword_variant_agent.py`

Obsługuje **warianty liczby pojedynczej/mnogiej i rodzajniki** (np. „strawberry freeze dried" vs „strawberries freeze dried" vs „the strawberry freeze dried"):
- Wykrywa grupy wariantów za pomocą AI.
- Wybiera optymalny wariant na podstawie wolumenu wyszukiwań i struktury gramatycznej.
- Zwraca `variant_groups` i `optimized_keywords` z deduplikacją.

#### RootRelevanceAgent
**Plik:** `backend/app/local_agents/scoring/subagents/root_relevance_agent.py`

Filtruje, które **rdzenie słów kluczowych** powinny być uwzględnione w obliczeniach szerokiego wolumenu, oceniając ich trafność za pomocą AI. Zastępuje programistyczne filtrowanie inteligentną oceną trafności rdzeni.

#### OpportunitySubagent
**Plik:** `backend/app/local_agents/scoring/subagents/opportunity_agent.py`

Stosuje **reguły szans przy zerowej gęstości tytułów**:
- Jeśli `title_density == 0` i słowo kluczowe jest trafne: oznacza jako szansę.
- Jeśli `title_density` jest niskie, a wolumen przyzwoity: oznacza jako szansę.
- Zwraca `opportunity_decision` i `opportunity_reason` dla każdego słowa kluczowego.

**Runner:** `backend/app/local_agents/scoring/runner.py` → `ScoringRunner.run_scoring()`

---

### Agent 4 – Agent optymalizacji SEO

**Plik:** `backend/app/local_agents/seo/`

**Cel:** Analizuje bieżący stan SEO oferty i generuje w pełni zoptymalizowany tytuł, punktory i słowa kluczowe backendu.

**Co robi:**

1. **Analiza bieżącego SEO** (deterministyczna):
   - Oblicza procent pokrycia słowami kluczowymi (ile trafnych słów kluczowych pojawia się w tytule, punktorach, backendzie).
   - Mierzy efektywność znaków (jak dobrze jest wykorzystywana dostępna przestrzeń znaków).
   - Identyfikuje luki w pokryciu rdzeni słów kluczowych.
   - Identyfikuje brakujące słowa kluczowe o wysokiej intencji i dużym wolumenie.

2. **Walidator słów kluczowych** (`SEOKeywordValidator`):
   - Zapobiega halucynacjom AI — weryfikuje, że każde słowo kluczowe w wyjściu AI rzeczywiście istnieje w zbiorze badawczym.
   - Przeprowadza walidację i korektę po wygenerowaniu treści.

3. **Optymalizacja AI** (przez `SEOOptimizationAgent`):
   - Generuje zoptymalizowany **tytuł** (≤200 znaków): najpierw nazwa marki, następnie słowa kluczowe w kolejności malejącego wolumenu, zawiera 2–3 słowa kluczowe Design-Specific z rdzenia o najwyższym wolumenie.
   - Generuje **5 punktorów**: pierwsze 2 używają 5 słów kluczowych o najwyższym wolumenie, punktory 3–5 używają terminów o średnio-wysokim wolumenie.
   - Generuje **słowa kluczowe backendu**: terminy nieobecne w tytule/punktorach, synonimy, alternatywne pisownie i literówki.

4. **Filtr słów kluczowych SEO** (`seo_keyword_filter.py`): Post-przetwarza wyjście AI w celu walidacji i korekty uwzględnionych słów kluczowych.

5. **Agent zgodności z Amazon** (`subagents/amazon_compliance_agent.py`): Sprawdza treści SEO pod kątem wytycznych Amazon dotyczących ofert.

6. **Agent analizy tytułów konkurentów** (`subagents/competitor_title_analysis_agent.py`): Analizuje wzorce w tytułach konkurentów w celu wsparcia strategii optymalizacji.

**Schemat wyjściowy (`SEOAnalysisResult`):**
- `current_seo` – analiza tytułu/punktorów/backendu z metrykami pokrycia
- `optimized_seo` – ulepszony tytuł, punktory, słowa kluczowe backendu z listami słów kluczowych i liczbą znaków
- `comparison` – metryki przed/po: % pokrycia, wynik intencji, przechwycony wolumen
- `improvement_rationale` – wyjaśnienie decyzji optymalizacyjnych

**Runner:** `backend/app/local_agents/seo/runner.py` → `SEORunner.run_seo_analysis()`

---

## System scrapowania Amazon

**Katalog:** `backend/app/services/amazon/`

System używa wielu strategii scrapowania z automatycznym przełączaniem awaryjnym:

### Scraper Scrapy
**Plik:** `scraper.py`, `search_scraper.py`

Główny scraper oparty na frameworku **Scrapy**. Wyodrębnia konkretne identyfikatory elementów HTML ze stron produktów Amazon:

| ID elementu | Zawartość |
|---|---|
| `#productTitle` | Tytuł produktu |
| `#productOverview_feature_div` | Marka, rozmiar i cechy w formie klucz-wartość |
| `#feature-bullets` | Cechy w formie punktorów |
| `#productDescription` | Opis produktu |
| `#prodDetails` | Specyfikacje techniczne |
| `#detailBullets_feature_div` | Szczegółowe punktory |
| `#aplus` | Moduły A+ (Enhanced Brand Content) |

Wyodrębnia również: zdjęcia produktu (przez atrybut JSON `data-a-dynamic-image` i `srcset`), recenzje klientów, pary pytań i odpowiedzi (Q&A).

### Scraper Playwright
**Plik:** `playwright_scraper.py`

Scraper z bezgłową przeglądarką oparty na **Playwright** dla treści renderowanych przez JavaScript. Używany jako fallback, gdy Scrapy jest blokowany lub gdy treść wymaga wykonania JavaScript.

### Scraper SERP API
**Plik:** `serp_api_scraper.py`

Integracja z **SerpAPI** (`serpapi.com`) przy użyciu silnika `amazon_product`. Zapewnia niezawodne, szybkie scrapowanie bez ryzyka blokady IP. Wymaga klucza SerpAPI (zmienna środowiskowa `SERPAPI_API_KEY`). Zwraca znormalizowane dane produktu zgodne z wewnętrznym formatem potoku.

### Lekkie scrapery standalone
**Pliki:** `standalone_scraper.py`, `standalone_search_scraper.py`

Lekkie scrapery używające **requests + BeautifulSoup** oraz **CloudScraper** do szybkiego, samodzielnego scrapowania bez pełnego frameworku Scrapy.

### Wsparcie dla wielu rynków
**Plik:** `country_handler.py`

Obsługuje wiele rynków Amazon poprzez mapowanie kodów rynku na bazowe adresy URL:

| Kod | Rynek |
|---|---|
| `US` | amazon.com |
| `UK` / `GB` | amazon.co.uk |
| `DE` | amazon.de |
| `FR` | amazon.fr |
| `IT` | amazon.it |
| `ES` | amazon.es |
| `CA` | amazon.ca |
| `JP` | amazon.co.jp |
| `AU` | amazon.com.au |
| `IN` | amazon.in |
| `MX` | amazon.com.mx |
| `BR` | amazon.com.br |

---

## System anty-blokujący

**Katalog:** `backend/app/services/amazon/anti_blocking/`

Kompleksowy system anty-detekcyjny zapobiegający blokowaniu przez Amazon.

### Rotacja User-Agent
**Plik:** `user_agents.py`

Utrzymuje pulę prawdziwych ciągów user-agent przeglądarek (Chrome, Firefox, Safari na Windows, Mac, Linux) i losowo wybiera jeden na każde żądanie.

### Randomizacja nagłówków
**Plik:** `headers.py`

Generuje realistyczne nagłówki HTTP żądania dla każdego zapytania:
- Rotacja `Accept-Language` (warianty języka angielskiego USA)
- Kombinacje `Accept-Encoding`
- Warianty `Cache-Control`
- Nagłówki specyficzne dla sesji Amazon (`x-requested-with`, `x-amzn-RequestId`)
- Nagłówki `Referer` i `Origin`

### Rotacja proxy
**Plik:** `proxy_manager.py`

Obsługuje wielu dostawców proxy konfigurowanych przez zmienne środowiskowe:
- Bezpośrednia lista proxy (`SCRAPER_PROXY`, `SCRAPER_PROXY_LIST`)
- **Bright Data** (Luminati): `BRIGHT_DATA_HOST`, `BRIGHT_DATA_USER`, `BRIGHT_DATA_PASS`
- **Smartproxy**: `SMARTPROXY_HOST`, `SMARTPROXY_USER`, `SMARTPROXY_PASS`
- **Oxylabs**: `OXYLABS_HOST`, `OXYLABS_USER`, `OXYLABS_PASS`

Domyślnie rotuje proxy co 30 sekund.

### Middleware Scrapy
**Plik:** `middlewares.py`

Niestandardowe middleware pobierania Scrapy:
- `RotateUserAgentMiddleware` – rotuje user-agent przy każdym żądaniu
- `RotateHeadersMiddleware` – generuje świeże, realistyczne nagłówki dla każdego żądania
- `ProxyRotationMiddleware` – wstrzykuje rotującą konfigurację proxy
- `CustomRetryMiddleware` – ponawia próby przy kodach 503/429 z wykładniczym cofaniem, oznacza proxy jako nieprawidłowe

### Integracja CloudScraper
**Plik:** `anti_blocking.py`

Używa biblioteki **CloudScraper** do omijania stron Cloudflare i wyzwań JavaScript. Przełącza się na standardowe requests, gdy CloudScraper jest niedostępny.

---

## Narzędzia do przetwarzania słów kluczowych

**Katalog:** `backend/app/services/keyword_processing/`

### Ekstrakcja rdzeni (Root Extraction)
**Plik:** `root_extraction.py`

Deterministyczna ekstrakcja rdzeni słów kluczowych:
- Identyfikuje priorytetowe terminy rdzeniowe (najbardziej znaczące rzeczowniki) dla listy słów kluczowych.
- Używana do grupowania słów kluczowych i obliczania szerokiego wolumenu wyszukiwań.
- `get_priority_roots_for_search()` zwraca czołowe rdzenie uszeregowane według zagregowanego wolumenu wyszukiwań.

### Procesor wsadowy (Batch Processor)
**Plik:** `batch_processor.py`

`optimize_keyword_processing_for_agents()` łączy ekstrakcję rdzeni z obu list słów kluczowych (CSV przychodowy i wzorniczy), deduplikuje je i zwraca zoptymalizowaną analizę do spożycia przez agenty AI.

### Sortowanie według intencji
**Plik:** `sort.py`, `intent.py`

Sortuje słowa kluczowe według oceny intencji i wyniku trafności dla priorytetowej analizy.

### Subagent ekstrakcji rdzeni AI
**Plik:** `backend/app/local_agents/keyword/subagents/root_extraction_agent.py`

Ekstrakcja terminów rdzeniowych wspierana przez AI przy użyciu OpenAI Agents SDK. Zapewnia bogatszą analizę rdzeni niż podejście deterministyczne dzięki rozumieniu kontekstu produktu.

### Subagent klasyfikacji intencji AI
**Plik:** `backend/app/local_agents/keyword/subagents/intent_classification_agent.py`

Subagent AI do klasyfikowania intencji słów kluczowych przy użyciu kontekstu produktu i bazowych wyników trafności.

### Deduplikacja słów kluczowych (w ResearchRunner)

Zanim agenty AI przetworzą słowa kluczowe, potok:
1. Scala słowa kluczowe z obu plików CSV (przychodowego i wzorniczego).
2. Usuwa dokładne duplikaty (bez rozróżnienia wielkości liter).
3. Gdy słowo kluczowe pojawia się w obu plikach CSV, zachowuje **najwyższy wynik trafności** ze wszystkich źródeł.
4. Stosuje dynamiczny filtr progu trafności przed wysłaniem do agentów AI.

---

## System zarządzania zadaniami (Job Manager)

**Plik:** `backend/app/services/job_manager.py`

Dla długotrwałych analiz (przetwarzanie tysięcy słów kluczowych może trwać minuty) system udostępnia **asynchroniczny system zadań w tle**.

### Backendy przechowywania danych (wybór automatyczny):
1. **Redis (Upstash)** – podstawowe przechowywanie dla wdrożeń produkcyjnych
   - Konfiguracja przez `UPSTASH_REDIS_URL` i `UPSTASH_REDIS_TOKEN`
   - Zadania wygasają po `JOB_TTL_HOURS` (domyślnie: 24 godziny)
2. **System plików** – fallback dla lokalnego środowiska deweloperskiego (przechowywane w katalogu `jobs/`)

### Cykl życia zadania:
1. `JobManager.create_job()` → zwraca unikalny identyfikator UUID zadania
2. Status: `pending` (oczekujące) → `processing` (przetwarzanie, postęp 0–100%) → `complete` (zakończone) / `failed` (nieudane)
3. `JobManager.update_status()` – aktualizuje postęp i komunikat podczas przetwarzania
4. `JobManager.save_results()` – zapisuje pełne wyjście potoku
5. `JobManager.get_results()` – pobiera zapisane wyniki według identyfikatora zadania

### Ogranicznik szybkości OpenAI
**Plik:** `backend/app/services/openai_rate_limiter.py`

Ogranicznik szybkości metodą token bucket dla wywołań OpenAI API:
- `OPENAI_REQUESTS_PER_MINUTE` (domyślnie: 15)
- `OPENAI_REQUESTS_PER_SECOND` (domyślnie: 2)
- Wykładnicze cofanie z `OPENAI_MAX_RETRIES` (domyślnie: 3)

### Monitor użycia OpenAI
**Plik:** `backend/app/services/openai_monitor.py`

Śledzi użycie tokenów, liczby żądań i szacowane koszty na uruchomienie potoku. Rejestruje szczegółowe statystyki, gdy `ENABLE_OPENAI_MONITORING=true` i `LOG_DETAILED_STATS=true`.

---

## Endpointy API

**Bazowy URL:** `/api/v1`

### `POST /amazon-sales-intelligence` (synchroniczny)
Uruchamia kompletny potok 4 agentów. Przyjmuje `multipart/form-data`:

| Pole | Typ | Wymagane | Opis |
|---|---|---|---|
| `asin_or_url` | string | ✅ | ASIN Amazon lub pełny URL produktu |
| `marketplace` | string | domyślnie: `US` | Kod rynku |
| `main_keyword` | string | opcjonalne | Nadpisanie automatycznie wykrytego głównego słowa kluczowego |
| `revenue_csv` | plik | opcjonalne | Eksport Helium 10 Cerebro (czołowi konkurenci przychodowi) |
| `design_csv` | plik | opcjonalne | Eksport Helium 10 Cerebro (czołowi konkurenci wzorniczo-projektowi) |

### `POST /start-analysis` (asynchroniczny – zalecany dla produkcji)
Te same parametry co powyżej. Zwraca natychmiast `job_id`. Użyj odpytywania statusu do śledzenia postępu.

### `GET /job-status/{job_id}`
Odpytywanie postępu zadania. Zwraca:
```json
{
  "status": "processing",
  "progress": 45,
  "message": "Running keyword categorization...",
  "result": null
}
```

Gdy `status == "complete"`, pole `result` zawiera pełne wyjście potoku.

### `POST /upload/csv`
Samodzielny endpoint przesyłania i parsowania pliku CSV. Zwraca sparsowane dane wierszy jako JSON.

### `GET /`
Sprawdzenie stanu serwisu — zwraca `{"message": "Welcome to the Amazon Sales Agent API"}`.

---

## Aplikacja frontendowa

**Katalog:** `frontend/`

Aplikacja panelowa **Next.js** (React) zapewniająca interfejs webowy dla platformy.

**Framework:** Next.js z App Router, TypeScript, Tailwind CSS, komponenty shadcn/ui

**Strony panelu:**
- `/dashboard` – Przegląd z kluczowymi metrykami (łączna liczba badań, znalezione słowa kluczowe, średni wynik optymalizacji, przeanalizowani konkurenci)
- `/dashboard/research` – Formularz nowego badania (wejście ASIN/URL, wybór rynku, słowo kluczowe, przesyłanie CSV)
- `/dashboard/upload` – Przesyłanie plików CSV
- `/dashboard/results` – Wyświetlanie wyników badań
- `/dashboard/history` – Historia poprzednich analiz
- `/dashboard/reports` – Wygenerowane raporty

**Komunikacja frontend–backend:**
- `lib/api.ts` – Synchroniczny klient API używający `fetch`
- `lib/api-client-background.ts` – Asynchroniczny klient API do odpytywania zadań w tle
- `lib/config.ts` – Konfiguracja URL backendu (przez zmienną środowiskową `NEXT_PUBLIC_API_URL`)

**Wdrożony pod adresem:** `https://amazon-sales-agent.vercel.app` (Vercel)

---

## Zewnętrzne API i usługi

| Usługa | Cel | Konfiguracja |
|---|---|---|
| **OpenAI API** | Zasila wszystkie 4 agenty AI i subagenty | `OPENAI_API_KEY` |
| **SerpAPI** | Niezawodne scrapowanie produktów Amazon | `SERPAPI_API_KEY` |
| **Upstash Redis** | Przechowywanie zadań w tle w produkcji | `UPSTASH_REDIS_URL`, `UPSTASH_REDIS_TOKEN` |
| **Bright Data** | Dostawca proxy rezydencjalnych do scrapowania | `BRIGHT_DATA_HOST/USER/PASS` |
| **Smartproxy** | Alternatywny dostawca proxy | `SMARTPROXY_HOST/USER/PASS` |
| **Oxylabs** | Alternatywny dostawca proxy | `OXYLABS_HOST/USER/PASS` |

**Model OpenAI:** `gpt-5-mini-2025-08-07` (używany przez wszystkich agentów)
**Framework AI:** `openai-agents>=0.2.9` (OpenAI Agents SDK)

---

## Konfiguracja i zmienne środowiskowe

Wszystkie ustawienia są ładowane z pliku `.env` lub środowiska. Patrz `backend/app/core/config.py`.

### Wymagane
| Zmienna | Opis |
|---|---|
| `OPENAI_API_KEY` | Klucz API OpenAI dla wszystkich agentów AI |

### Zachowanie agentów
| Zmienna | Domyślna | Opis |
|---|---|---|
| `OPENAI_MODEL` | `gpt-4` | Nazwa modelu OpenAI (każdy agent nadpisuje tę wartość kodem używającym `gpt-5-mini-2025-08-07`; ta zmienna środowiskowa nie jest używana przez agentów) |
| `USE_AI_AGENTS` | `true` | Włącz/wyłącz agenty AI |
| `FALLBACK_TO_DIRECT` | `true` | Przełącz na przetwarzanie bezpośrednie, jeśli agenty zawiodą |

### Ograniczanie szybkości
| Zmienna | Domyślna | Opis |
|---|---|---|
| `OPENAI_REQUESTS_PER_MINUTE` | `15` | Limit żądań OpenAI na minutę |
| `OPENAI_REQUESTS_PER_SECOND` | `2` | Limit chwilowy żądań OpenAI na sekundę |
| `OPENAI_MAX_RETRIES` | `3` | Maksymalna liczba ponownych prób przy błędach limitu |
| `OPENAI_BASE_RETRY_DELAY` | `1.0` | Bazowe opóźnienie (sekundy) dla wykładniczego cofania |

### Przetwarzanie wsadowe
| Zmienna | Domyślna | Opis |
|---|---|---|
| `BATCH_SIZE` | `25` | Słowa kluczowe na partię przetwarzania |
| `MAX_CONCURRENT_BATCHES` | `3` | Limit równoczesnych partii |
| `BATCH_TIMEOUT` | `120` | Sekundy na partię |
| `KEYWORD_BATCH_SIZE` | `500` | Słowa kluczowe na partię ekstrakcji rdzeni |

### Przechowywanie danych
| Zmienna | Domyślna | Opis |
|---|---|---|
| `UPSTASH_REDIS_URL` | — | URL REST Upstash Redis |
| `UPSTASH_REDIS_TOKEN` | — | Token uwierzytelniający Upstash Redis |
| `USE_REDIS_FOR_JOBS` | `true` | Użyj Redis do przechowywania zadań |
| `JOB_TTL_HOURS` | `24` | Godziny do wygaśnięcia danych zadania |

### Scrapowanie
| Zmienna | Domyślna | Opis |
|---|---|---|
| `SCRAPER_PROXY` | — | Pojedynczy URL proxy |
| `SCRAPER_PROXY_LIST` | — | Lista proxy oddzielona przecinkami |
| `BRIGHT_DATA_HOST/USER/PASS` | — | Dane uwierzytelniające proxy Bright Data |
| `SMARTPROXY_HOST/USER/PASS` | — | Dane uwierzytelniające Smartproxy |
| `OXYLABS_HOST/USER/PASS` | — | Dane uwierzytelniające Oxylabs |
| `SERPAPI_API_KEY` | — | Klucz SerpAPI dla scrapera SERP API |

### CORS
| Zmienna | Domyślna | Opis |
|---|---|---|
| `CORS_ORIGINS` | `http://localhost:3000,...` | Dozwolone źródła oddzielone przecinkami |

---

## Stos technologiczny

### Backend
| Technologia | Wersja | Rola |
|---|---|---|
| Python | ≥3.13 | Środowisko uruchomieniowe |
| FastAPI | ≥0.116.1 | Framework webowy + REST API |
| OpenAI Agents SDK | ≥0.2.9 | Framework agentów AI |
| Scrapy | ≥2.13.3 | Scrapowanie Amazon |
| Playwright | ≥1.40.0 | Scrapowanie z bezgłową przeglądarką |
| BeautifulSoup4 | ≥4.12.0 | Parsowanie HTML |
| CloudScraper | ≥1.2.71 | Omijanie Cloudflare |
| Upstash Redis | ≥0.15.0 | Przechowywanie zadań w tle |
| Pydantic | (przez FastAPI) | Walidacja danych i schematy |
| uvicorn | (przez FastAPI) | Serwer ASGI |
| uv | — | Menedżer pakietów |
| pytest | ≥8.4.1 | Testowanie |

### Frontend
| Technologia | Rola |
|---|---|
| Next.js (App Router) | Framework React |
| TypeScript | JavaScript z typowaniem statycznym |
| Tailwind CSS | Framework CSS oparty na klasach użytkowych |
| shadcn/ui | Biblioteka komponentów UI |
| Lucide React | Ikony |

---

## Wdrożenie

### Backend – Render
**Plik:** `backend/render.yaml`

Backend jest skonfigurowany do wdrożenia na platformie [Render](https://render.com):
- Polecenie startowe: `uvicorn app.main:app --host 0.0.0.0 --port 8000 --timeout-keep-alive 18000`
- Polecenie deweloperskie: to samo z flagą `--reload`
- Długi timeout podtrzymania połączenia (18 000 sekund) do obsługi długotrwałych uruchomień potoku.

Skrypty:
- `backend/scripts/deploy.sh` – pomocnik wdrożeniowy
- `backend/scripts/health_check.sh` – sprawdzenie stanu endpointu

### Frontend – Vercel
Frontend Next.js jest wdrożony na platformie [Vercel](https://vercel.com). Ustaw `NEXT_PUBLIC_API_URL` na URL backendu Render.

### Lokalny rozwój

```bash
# Backend (wymaga Python 3.13+)
cd backend
uv sync
cp .env.example .env   # dodaj OPENAI_API_KEY
uv run dev

# Frontend
cd frontend
npm install
cp .env.local.example .env.local   # ustaw NEXT_PUBLIC_API_URL
npm run dev
```

---

## Format danych wejściowych (CSV Helium 10 Cerebro)

Potok przyjmuje eksporty badań słów kluczowych z **Helium 10 Cerebro**. Oczekiwane kolumny CSV:

| Kolumna | Opis |
|---|---|
| `Keyword Phrase` | Fraza wyszukiwania |
| `Search Volume` | Miesięczny wolumen wyszukiwań na Amazon |
| `Relevancy` | Wynik trafności Helium 10 |
| `Title Density` | Ile czołowych wyników zawiera słowo kluczowe w tytule |
| `CPR` | Cerebro Product Rank (szacowana liczba rozdań potrzebna do osiągnięcia rankingu) |
| `Cerebro IQ Score` | Wynik szans Helium 10 |
| `Competing Products` | Liczba produktów konkurujących o słowo kluczowe |
| `B0XXXXXXXXX` (kolumny) | Organiczny ranking każdego ASIN dla słowa kluczowego |

Przyjmowane są dwa pliki:
- **Revenue CSV** – słowa kluczowe od konkurentów generujących najwyższe przychody
- **Design CSV** – słowa kluczowe od konkurentów z najlepszymi wariantami wzorniczo-projektowymi
