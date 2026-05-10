# Wymagania aplikacji "Biblia Nauka Dowody"

> Dokument główny. Konsoliduje decyzje strategiczne, architekturę wysokopoziomową i podział na 13 modułów dla niezależnych zespołów.
>
> **Powiązane**: [STRATEGIA-FANPAGE.md](../STRATEGIA-FANPAGE.md) (warstwa contentu), [modules/](modules/) (specyfikacje per moduł), [ROADMAP-FALA-0.md](ROADMAP-FALA-0.md) (pierwsza fala wykonawcza).

---

## 1. Cel aplikacji

System wspierający fanpage "Biblia Nauka Dowody" — racjonalna apologetyka po polsku zgodna z teologią adwentystów dnia siódmego. Cztery filary:

1. **AI generuje wpisy** na podstawie kuratorowanych źródeł (Biblia, EGW, SDA Bible Commentary, materiały archeologiczne, własne dokumenty).
2. **Pastorzy zatwierdzają** w "poczekalni" z dwustopniowym pipeline'em (redaktor → pastor).
3. **Automat publikuje** w optymalnych slotach na Facebook + Instagram.
4. **Społeczność dyskusyjna** na Discordzie z crowdsourced Q&A + 4-6 wydarzeń stacjonarnych rocznie.

---

## 2. Decyzje strategiczne (zatwierdzone w wywiadzie 8 rund)

### 2.1 Ramy techniczne

| Wymiar | Decyzja |
|---|---|
| Stack | Python (FastAPI) + React (Next.js) |
| Hosting | Self-hosted, serwer parafialny **16-32 GB RAM, bez GPU** |
| Konteneryzacja | Docker Compose (jeden host) |
| Bazy danych | Postgres (transakcje), Qdrant (vector store), Redis (kolejka), ClickHouse (DWH analityczne) |
| Reverse proxy | Caddy (auto-TLS) |
| Kanały publikacji | Facebook page + Instagram business (od MVP) |
| Społeczność | Discord — własny dedykowany serwer od dnia 1 |
| Budżet zewn. | < 5 000 PLN/rok (tanie modele AI + open-source self-hosted) |

### 2.2 AI / Generowanie treści

| Element | Decyzja |
|---|---|
| Architektura | Multi-provider gateway (LiteLLM lub własna abstrakcja) |
| Providerzy | Anthropic, OpenAI, DeepSeek, z.ai, Ollama (lokalny) — admin podpina per zastosowanie |
| Domyślne modele (cost-aware) | Draft: Haiku 4.5 / DeepSeek V3; Final: Haiku 4.5 (Sonnet okazjonalnie); Judge: Haiku / Gemini Flash |
| Warianty | AI generuje 3 warianty per post → redaktor wybiera 1 |
| Źródła RAG (v1) | Biblia (Warszawska, Tysiąclecia, Poznańska, BWP) + EGW (egwwritings.org) + SDA Bible Commentary + materiały archeologiczne + dokumenty uploadowane przez pastora |
| Embeddings | Lokalne `multilingual-e5-base` lub `paraphrase-multilingual-mpnet-base-v2` (CPU OK) |
| Vector store | Qdrant self-hosted |
| DPA | **Zero danych osobowych w promptach** — twardy invariant. Q&A anonimizowane przed wysłaniem do LLM. Żadne DPA niepotrzebne. |

### 2.3 Kontrola teologiczna (4 warstwy obrony)

1. **System prompt** zawiera 28 Fundamental Beliefs ADW + listę "red flags" (nieśmiertelność duszy, niedziela jako sabat, nieomylność papieża, etc.).
2. **LLM-judge** — drugi (tani) model ocenia wygenerowany post 0-10 pod kątem zgodności z ADW. Posty < 8 wracają do regeneracji automatycznie.
3. **Doctrine RAG** — oddzielny vector store z 28 FB i red flags; AI pobiera relevantne fragmenty przy każdym wywołaniu.
4. **Whitelist/blacklist tematów** — admin definiuje "tematy bezpieczne" (faza 1: archeologia, nauka, proroctwa Daniela; bez 1844, sanktuarium, EGW jako jawnego źródła) i "nigdy" (kontrowersje denominacyjne, polityka).

### 2.4 Workflow editorial

| Element | Decyzja |
|---|---|
| Stage'y | 2-stage: Redaktor → Pastor |
| Konsensus | 1 pastor z dyżuru wystarczy |
| SLA | 72h, eskalacja po przekroczeniu (alert do innego pastora) |
| Po odrzuceniu | (a) Pastor pisze feedback → AI regeneruje, (b) Pastor edytuje bezpośrednio → publikacja, (c) Post idzie do archiwum "do nauki" jako few-shot negative, (d) Statystyki dla redakcji |

### 2.5 Publikacja

| Element | Decyzja |
|---|---|
| Sloty | Sztywne z konfiguracji admina (wzorzec: STRATEGIA-FANPAGE.md sekcja 2 — Czw 9:00, Sob 10:00, Wt 19:00, Niedz 18:00) |
| Multi-channel | AI generuje 2 wersje per post — FB długi (3-7 zdań + rozwinięcie 20-30) i IG krótki (carousel-friendly) |
| Crisis controls | Wszystkie 4: kill-switch, ban-list (regex + LLM), soft-publish 30-min, audit log + Slack/email alerty |

### 2.6 Społeczność i wydarzenia

| Element | Decyzja |
|---|---|
| Platforma | Discord — dedykowany serwer od dnia 1, z możliwością multi-guild w v2 |
| Moderacja | AI sumuje sentymenty, ludzie moderują top-alerty (nie auto-ban) |
| Q&A | Crowdsourced — wierni głosują, top 5/tydzień wchodzi do kolejki M-EDIT |
| Wydarzenia | 4-6 wydarzeń centralnych rocznie (Warszawa/Kraków), 50-200 osób |
| Rejestracja | Imię + email, retention 13 mies., samoobsługa cancel |

### 2.7 Role i RBAC

- **Pastor = teolog = admin** (jedna osoba na start), drugi pastor jako backup do dyżurów.
- **Redaktor** (1-3 osoby) — świeccy współpracownicy, edytują, podsyłają do pastora.
- **Wolontariusz-moderator** (Discord) — moderuje top-alerty.
- **Członek** (Discord OAuth) — uczestnik społeczności, autor pytań Q&A, uczestnik wydarzeń.

### 2.8 Auth

- Pastor: email + TOTP MFA (Yubikey opcjonalnie)
- Redaktor: email + hasło
- Moderator: email + hasło
- Członek: Discord OAuth
- Self-service `/moje-dane`: export + delete (RODO art. 15 i 17)

### 2.9 Analityka

- DWH: ClickHouse (open-source self-hosted)
- ETL daily: FB/IG Insights API + Discord stats + wewnętrzne metryki (acceptance rate, time-to-review, koszty AI per post/per provider)
- Frontend: Metabase (open-source) z eksportem CSV/Sheets
- Korelacje kluczowe: temat → engagement, slot → engagement, źródło → akceptacja, koszt-API → jakość

---

## 3. Architektura wysokopoziomowa

```
┌──────────────────────────────────────────────────────────────────┐
│                         FRONTEND (Next.js)                       │
│  Public site │ Editor app │ Pastor app │ Admin app │ User self  │
└────────┬─────────────────────────────────────────────────────────┘
         │ REST/JSON
┌────────┴─────────────────────────────────────────────────────────┐
│                      BACKEND CORE (FastAPI)                      │
│  Auth │ Editorial │ Scheduler │ Source mgmt │ Events │ Q&A      │
└──┬─────────┬──────────┬────────────┬─────────────┬───────────────┘
   │         │          │            │             │
   │   ┌─────┴──────┐ ┌─┴────┐  ┌────┴─────┐  ┌────┴──────┐
   │   │ AI Gateway │ │ RAG  │  │ Image    │  │ Discord   │
   │   │ (multi-    │ │ +    │  │ Gen      │  │ Bot       │
   │   │ provider)  │ │ Vec. │  │ Pipeline │  │           │
   │   └─────┬──────┘ │ Store│  └────┬─────┘  └────┬──────┘
   │         │        └─┬────┘       │             │
   │   ┌─────┴──────────┴────────────┴─────────────┴──────┐
   │   │    Theological Guardrails (4 warstwy)            │
   │   └──────────────────────────────────────────────────┘
   │
┌──┴────────────────────────────────────────────────────────────┐
│           DATA LAYER (Docker Compose, self-hosted)            │
│  Postgres │ Qdrant │ Redis (queue) │ ClickHouse (DWH)         │
└───────────────────────────────────────────────────────────────┘
                  │
       ┌──────────┴───────────┐
       │  Meta Graph API      │
       │  (FB Page + IG)      │
       └──────────────────────┘
```

---

## 4. Lista modułów (13)

| ID | Moduł | Faza | Zespół | Plik |
|---|---|---|---|---|
| M-INFRA | DevOps & Infrastructure | 0 | 1 osoba | [modules/M-INFRA.md](modules/M-INFRA.md) |
| M-AUTH | Auth & Identity | 0 | 1 osoba | [modules/M-AUTH.md](modules/M-AUTH.md) |
| M-AI | AI Gateway & Generation | 1 | 1-2 osoby | [modules/M-AI.md](modules/M-AI.md) |
| M-RAG | Source Library & RAG | 1 | 1-2 osoby | [modules/M-RAG.md](modules/M-RAG.md) |
| M-GUARD | Theological Guardrails | 1 | 1 osoba (+pastor) | [modules/M-GUARD.md](modules/M-GUARD.md) |
| M-EDIT | Editorial Workflow | 1 | 1-2 osoby | [modules/M-EDIT.md](modules/M-EDIT.md) |
| M-PUB | Publishing & Scheduler | 1 | 1-2 osoby | [modules/M-PUB.md](modules/M-PUB.md) |
| M-IMG | Image Generation Pipeline | 2 | 1 osoba (+grafik) | [modules/M-IMG.md](modules/M-IMG.md) |
| M-COMM | Community / Discord | 2 | 1 osoba | [modules/M-COMM.md](modules/M-COMM.md) |
| M-EVT | Events Management | 3 | 1 osoba | [modules/M-EVT.md](modules/M-EVT.md) |
| M-ANALYTICS | DWH + Dashboards | 3 | 1 osoba | [modules/M-ANALYTICS.md](modules/M-ANALYTICS.md) |
| M-OPS | Admin & Configuration | 4 | 1 osoba | [modules/M-OPS.md](modules/M-OPS.md) |
| M-WEB | Public Site | 4 | 1 osoba | [modules/M-WEB.md](modules/M-WEB.md) |

---

## 5. Roadmapa fal

```
Fala 0  (4-6 tyg.)  ▸ Fundamenty
                    ▸ M-INFRA, M-AUTH

Fala 1  (8-12 tyg.) ▸ Silnik treści (MVP zamykający 3 z 4 założeń: AI→pastor→publikacja)
                    ▸ Zespół A: M-AI, M-RAG, M-GUARD
                    ▸ Zespół B: M-EDIT, M-PUB

Fala 2  (6-8 tyg.)  ▸ Media + Społeczność
                    ▸ Zespół A: M-IMG
                    ▸ Zespół B: M-COMM

Fala 3  (4-6 tyg.)  ▸ Wydarzenia + Analityka (zamknięcie 4. założenia)
                    ▸ Zespół A: M-EVT
                    ▸ Zespół B: M-ANALYTICS

Fala 4  (ciągła)    ▸ Optymalizacja
                    ▸ M-OPS, M-WEB
```

**Critical path do MVP** (założenia 1-3 działają):
M-INFRA → M-AUTH → M-AI → M-RAG → M-GUARD → M-EDIT → M-PUB

Po Fali 1 system publikuje w pełni autonomicznie z weryfikacją pastora. Fala 2-3 dodaje grafikę, społeczność i wydarzenia.

---

## 6. Ograniczenia budżetowe (< 5k PLN/rok)

Implikuje wybór **tanich modeli + open-source self-hosted**:

| Komponent | Wybór |
|---|---|
| LLM draft | Haiku 4.5 / DeepSeek V3 / Gemini Flash |
| LLM final | Haiku 4.5 (Sonnet okazjonalnie dla rozwinięć 20-30 zdań) |
| LLM judge | Haiku / Gemini Flash (lekki, szybki) |
| Embeddings | `multilingual-e5-base` self-hosted (CPU OK) |
| Vector store | Qdrant self-hosted |
| Image gen | Templates + PIL (priorytet) / Pollinations API (free, fallback) |
| Hosting | Serwer parafialny (już opłacony) |
| Monitoring | Uptime Kuma + Grafana OSS |
| DWH | ClickHouse self-hosted |
| Analytics UI | Metabase OSS |
| Email transactional | SMTP własny lub free tier (Resend free 3k/mies., SendGrid free 100/dzień) |
| Discord | Free |

Realny szacunek: **150-300 PLN/mies** przy 4-5 postach/tydz. (głównie tanie API).

---

## 7. Inwariant projektowy (kluczowy dla bezpieczeństwa)

> **Żadne dane osobowe nie wchodzą do promptów modeli AI.**

Egzekwowane przez:
- M-AI: `redact_pii(prompt)` przed każdym wywołaniem providera
- M-COMM: pytania Q&A anonimizowane (usunięcie `@username`, emaili) przed wejściem do M-EDIT
- M-EVT: dane uczestników (imię+email) **nigdy** nie idą do AI; wyjątek: agregaty (np. "5 osób z Warszawy")
- Audit log: każde wywołanie providera + sprawdzenie czy redaction się odbył

---

## 8. Definicje gotowości

### Definicja "MVP gotowe"

- [ ] Pastor może zalogować się przez email + TOTP MFA
- [ ] AI generuje 3 warianty posta z RAG opartym na min. Biblii + 1 dodatkowym źródle
- [ ] LLM-judge automatycznie odrzuca posty < 8/10 zgodności ADW
- [ ] Redaktor wybiera wariant, edytuje, podsyła do pastora
- [ ] Pastor akceptuje (1 click) albo pisze feedback → regeneracja
- [ ] Po akceptacji post trafia do schedulera w sztywnym slocie
- [ ] Soft-publish 30-min daje okno na anulowanie
- [ ] Publikacja na FB + IG (2 wersje z 1 briefu)
- [ ] Audit log każdej akcji + alert do admina
- [ ] Kill-switch zatrzymuje wszystko jednym klikiem

### Definicja "Pełna wizja gotowa"

MVP + Discord (cross-post FB→Discord, Q&A pipeline) + grafika (HITL prompt → image) + 1 wydarzenie zarejestrowane przez aplikację + Metabase z 5 podstawowymi dashboardami.

---

## 9. Otwarte ryzyka

| Ryzyko | Mitigation |
|---|---|
| Halucynacja AI w cytowaniu Biblii / EGW | LLM-judge weryfikuje cytaty z RAG; pastor zawsze ostatni głos |
| FB ban algorytmiczny za ban-bait / kontrowersje | Whitelist tematów + manualna moderacja w fazie 1 + soft-publish 30-min |
| Pastor nie nadąża z review (72h SLA) | Drugi pastor backup + eskalacja + redaktor edytuje na pastora |
| Self-hosted serwer pada w środku publikacji | Backup daily + monitoring + manual fallback (publikacja przez Meta Business Suite) |
| Discord toxic users | Sentiment AI flag → moderator → ban; kanał #pytania zamknięty na anonim |
| Koszty AI przekraczają budżet | Cost dashboard real-time + auto-fallback na tańszego providera + kill-switch dla AI generation |
| Plaglarism / prawa autorskie EGW/SDA | EGW w domenie publicznej (zmarła 1915 + prace przekazane fundacji); SDA Bible Commentary — fair use cytatów krótkich; własne reformułowania |

---

## 10. Następne kroki

Patrz [ROADMAP-FALA-0.md](ROADMAP-FALA-0.md) — szczegółowa rozpiska Fali 0 (M-INFRA + M-AUTH) na 4-6 tygodni.
