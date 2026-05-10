# M-GUARD — Theological Guardrails

> **Faza**: 1 (Silnik treści) · **Zespół**: 1 osoba (dev) + pastor jako konsultant doktrynalny · **Estymacja**: 3-4 tyg.

## Cel

Czterowarstwowa obrona teologiczna ZANIM treść trafi do pastora-zatwierdzającego. Egzekwuje zgodność z 28 Fundamental Beliefs ADW, blokuje "red flag" doktryny, ogranicza zakres tematyczny do whitelist'y (faza 1: bez 1844, sanktuarium, EGW jako jawnego źródła).

## Cztery warstwy obrony

```
[Generowany post]
       ↓
┌─────────────────────────────────────────────────┐
│ Warstwa 1: System prompt z 28 FB ADW             │
│ → wpinany do każdego wywołania M-AI generation  │
└──────────────────┬──────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────┐
│ Warstwa 2: Doctrine RAG                          │
│ → AI pobiera relevantne fragmenty doktryny       │
│   przy generowaniu (kontekst inline)             │
└──────────────────┬──────────────────────────────┘
                   ↓
              [POST WYGENEROWANY]
                   ↓
┌─────────────────────────────────────────────────┐
│ Warstwa 3: Whitelist/blacklist filter            │
│ → regex + LLM czy temat dopuszczony              │
│ → wykryte 'red flags' (np. "papież nieomylny")  │
└──────────────────┬──────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────┐
│ Warstwa 4: LLM-judge                             │
│ → drugi (tani) model ocenia 0-10 zgodność z ADW │
│ → wraca do regeneracji jeśli < 8                │
│ → zapisuje feedback dla pastora                  │
└──────────────────┬──────────────────────────────┘
                   ↓
              [DO M-EDIT (kolejka redaktora)]
```

## Stack

- **FastAPI** (backend)
- **Qdrant** — oddzielna kolekcja `bnd_doctrine_v1` (28 FB + red flags + tematy whitelist)
- **Postgres** — versioned doctrine cards, blacklist regex
- **Redis** — cache wyników judge dla deterministycznych promptów

## Funkcjonalności

### F1. Doctrine cards repository (28 Fundamental Beliefs ADW)

Każda doktryna jako "card" w Postgres + chunki w vector store:

```
DoctrineCard
  id, number (1-28), title, summary,
  full_text TEXT (oficjalny tekst FB),
  key_phrases TEXT[] (ważne sformułowania, idiomy),
  red_flags TEXT[] (sformułowania sprzeczne z tą doktryną),
  examples_aligned TEXT[] (3-5 przykładów dobrego użycia),
  examples_misaligned TEXT[] (3-5 przykładów niedopuszczalnych),
  version INT, edited_by FK, edited_at
```

Lista 28 (strukturalnie):
1. Holy Scriptures, 2. Trinity, 3. Father, 4. Son, 5. Holy Spirit, 6. Creation, 7. Nature of Humanity, 8. Great Controversy, 9. Life Death and Resurrection of Christ, 10. Experience of Salvation, 11. Growing in Christ, 12. The Church, 13. The Remnant and Its Mission, 14. Unity in the Body of Christ, 15. Baptism, 16. Lord's Supper, 17. Spiritual Gifts and Ministries, 18. The Gift of Prophecy, 19. The Law of God, 20. **The Sabbath**, 21. Stewardship, 22. Christian Behavior, 23. Marriage and the Family, 24. **Christ's Ministry in the Heavenly Sanctuary**, 25. **The Second Coming of Christ**, 26. **Death and Resurrection** (state of the dead), 27. The Millennium and the End of Sin, 28. The New Earth.

### F2. Red flags database (versioned)

Tabela `red_flags`:

| flag_type | pattern (regex/keyword) | doctrine_violated | severity (1-3) | example |
|---|---|---|---|---|
| immortality_of_soul | `nieśmiertelnoś(ć\|ci) duszy` | 26 | 3 | "Po śmierci dusza idzie do nieba/piekła" |
| sunday_sabbath | `niedziel(a\|ę)\b.*(dzień Pański\|sabat)` | 20 | 3 | "Niedziela jako biblijny dzień odpoczynku" |
| pope_infallibility | `nieomylnoś(ć\|ci) papieża` | 13, 18 | 3 | "Magisterium kościelne ma autorytet równy Pismu" |
| purgatory | `czyść(ec\|cu)` | 26 | 3 | "Dusze cierpią w czyśćcu" |
| transubstantiation | `transsubstancj` | 16 | 2 | "W eucharystii dochodzi do realnej zamiany" |
| mary_mediator | `Maryja.*pośredni(czka\|k)` | 4, 9 | 3 | "Maria jako pośredniczka łask" |
| evolution_theistic | `(ewolucji.*teistycznej\|miliard(y\|ów) lat)` | 6 | 2 | "Bóg posłużył się ewolucją" |
| premillennial_mistake | `pochwycen.*kościoła` | 25, 27 | 2 | "Tajemne pochwycenie kościoła przed uciskiem" |

`severity 3` = blokada twarda (post nie idzie dalej, regeneracja).
`severity 2` = warning, post idzie do pastora z flagą.
`severity 1` = info w audit log.

### F3. Whitelist tematów (per faza)

```python
PHASE_1_TOPICS_ALLOWED = {
    "archeologia_biblijna",
    "spelnione_proroctwa",  # tylko Daniel 2, 5, Tyr — bez 1844
    "fine_tuning",
    "stworzenie_vs_ewolucja",  # ostrożnie
    "wierzacy_naukowcy",
    "manuskrypty",
    "wiarygodnosc_historyczna",
}

PHASE_1_TOPICS_BLOCKED = {
    "1844_investigative_judgment",  # do fazy 2
    "egw_jako_zrodlo_jawne",        # do fazy 2
    "sanktuarium_doktryna",         # do fazy 2
    "polityka",
    "denominacja_porownania",       # nie atakujemy katolików/protestantów
    "konkretni_polit_or_celebryci",
    "wojna_izrael_palestyna",
}
```

Konfigurowalne przez admin UI (M-OPS) — admin może odblokować temat per kampania.

### F4. System prompt builder

Dynamicznie składa system prompt dla M-AI generation:

```python
def build_system_prompt(role: str, topic: str | None) -> str:
    base = TEOLOGICZNE_FUNDAMENTY_28_FB  # zawsze
    safety = TEMATY_DOPUSZCZONE_FAZA_1   # zawsze
    voice = TON_RACJONALNY_BEZ_EMOCJI    # zawsze

    if topic in PHASE_1_TOPICS_BLOCKED:
        raise ForbiddenTopic(topic)

    if topic == "spelnione_proroctwa":
        base += PROROCTWA_GUIDANCE  # extra: tylko Daniel 2, 5, Tyr; nie wchodź w 1844

    return base + safety + voice
```

### F5. LLM-judge service

Drugi model (tani — Haiku/Gemini Flash) ocenia wygenerowany post.

**Prompt (skrót)**:
```
Jesteś teologiem adwentystycznym oceniającym treści apologetyczne.
Ocenisz post pod kątem zgodności z 28 Fundamental Beliefs ADW i naszymi zasadami treści.

POST DO OCENY:
{post_content}

Kryteria (oceń każde 0-10):
1. Brak sprzeczności doktrynalnych
2. Brak red flags z listy: {red_flags_list}
3. Trzymanie się whitelist'y tematów: {whitelist}
4. Ton racjonalny, bez emocjonalnego nacisku
5. Cytaty biblijne dokładne (sprawdź przeciwko {biblical_chunks_retrieved})
6. Brak wysuwanych claims bez podstawy w {sources_used}

Wynik (JSON):
{
  "scores": {<dim>: 0-10},
  "overall": 0-10,
  "issues": [{"severity": 1-3, "type": "...", "quote": "...", "explanation": "..."}],
  "verdict": "PASS" | "REVIEW" | "REGENERATE"
}
```

Decyzja:
- `overall >= 8` AND brak issues sev=3 → `PASS`
- `overall >= 6` AND brak issues sev=3 → `REVIEW` (idzie do pastora z flagami)
- inaczej → `REGENERATE` (wraca do M-AI z feedbackiem)

Limit: max 2 regeneracje, potem `MANUAL_REVIEW` (eskalacja do pastora z notą).

### F6. Real-time check w generowaniu

```python
async def safe_generate(req, topic):
    # Warstwa 1+2: wbudowane w M-AI przez system_prompt_builder
    system = guard.build_system_prompt(req.role, topic)
    req.messages = [Message(role="system", content=system), *req.messages]

    # Augmentacja z doctrine RAG (warstwa 2)
    doctrine_context = await guard.retrieve_doctrine_context(req.messages[-1].content)
    req.messages[-1].content += f"\n\nKONTEKST DOKTRYNALNY:\n{doctrine_context}"

    # Generation (M-AI)
    candidates = await ai.generate_variants(req, n=3)

    # Warstwa 3: filter
    candidates = [c for c in candidates if not guard.has_blacklist_violation(c.content)]

    # Warstwa 4: judge
    judged = await asyncio.gather(*[guard.judge(c) for c in candidates])
    passing = [c for c, j in zip(candidates, judged) if j.verdict in ("PASS", "REVIEW")]

    if not passing:
        # regeneracja z feedbackiem
        feedback = consolidate_feedback(judged)
        return await safe_generate(req.with_feedback(feedback), topic)

    return [(c, j) for c, j in zip(candidates, judged) if j.verdict in ("PASS", "REVIEW")]
```

### F7. Audit i metryki

- Każde wywołanie judge: zapis w `guard_audit` (post_id, scores, verdict, issues)
- Statystyki dostępne w M-ANALYTICS:
  - Acceptance rate (PASS / total)
  - Top red flags wykryte
  - Średni score per kategoria tematu
  - Czas judge avg/p99
  - Regeneration rate (ile generuje się 2× zanim PASS)

## API / Kontrakty

```
POST /api/guard/check              {text, topic?}            → JudgeResult
POST /api/guard/check-batch        {texts[]}                 → JudgeResult[]
GET  /api/guard/doctrine-cards     ?version                  → DoctrineCard[]
PUT  /api/guard/doctrine-cards/{n} (PASTOR/ADMIN)            → 200 (nowa wersja)
GET  /api/guard/red-flags                                    → RedFlag[]
PUT  /api/guard/red-flags          (PASTOR/ADMIN)            → 200
GET  /api/guard/whitelist/topics   ?phase                    → string[]
PUT  /api/guard/whitelist/topics   (ADMIN)                   → 200
POST /api/guard/build-system-prompt {role, topic}            → {system_prompt}

# Eventy
- guard.judged { post_id, verdict, overall_score }
- guard.regeneration_triggered { post_id, attempt, feedback }
- guard.escalated_manual { post_id, reason }
- guard.doctrine_card_updated { number, version, by }
```

## Definition of Done

- [ ] 28 doctrine cards wprowadzonych w Postgres + zembedowanych w Qdrant `bnd_doctrine_v1`
- [ ] Lista red flags pokrywa min. 8 typów z severity 3
- [ ] System prompt dynamicznie się buduje per topic (i blokuje topiki blacklist)
- [ ] LLM-judge zwraca strukturalny JSON z verdict
- [ ] Test E2E: post zawierający "papież nieomylny" → blokada (severity 3 + judge REGENERATE)
- [ ] Test E2E: post o Tyrze (proroctwo) → PASS, score >= 8
- [ ] Test E2E: post sugerujący ewolucję teistyczną → REVIEW (warning, idzie do pastora)
- [ ] Pastor może edytować doctrine card w UI (versioned, z historia zmian)
- [ ] Maks. 2 regeneracje, potem MANUAL_REVIEW
- [ ] Audit log: 100% wywołań judge zapisanych

## Edge cases i decyzje

- **False positive na red flag**: pastor może w UI dodać "exception phrase" (np. "papież" w kontekście historycznym Reformacji jest OK)
- **Post o EGW w fazie 2 (po odblokowaniu)**: dodatkowy red flag "EGW jako autorytet większy niż Pismo" (zawsze)
- **Cytaty biblijne**: judge sprawdza czy `bible_quote` w poście pojawia się w retrieval z M-RAG (Qdrant cosine > 0.85). Jeśli nie — issue sev 2 "potencjalna halucynacja cytatu"
- **Język**: judge działa po polsku, ale doctrine cards mogą zawierać EN oryginał + PL tłumaczenie
- **Versioning doctrine**: zmiana cardu nie usuwa starej wersji — nowa version, audit_log; embedding regenerowany asynchronicznie
- **Cache judgement**: dla deterministycznych wejść (`temperature=0`) cache 24h — same post nie powinien być oceniany 100×
- **Faza 2 odblokowanie tematów**: admin w UI przesuwa topic z BLOCKED do ALLOWED; system prompt się zmienia natychmiast (nowe wywołania)

## Zależności

**Wchodzi**: M-INFRA (Postgres, Qdrant, Redis), M-AI (judge model + embeddings), M-AUTH (RBAC), M-RAG (cytaty biblijne do weryfikacji)
**Wychodzi**: M-EDIT (każdy generowany post przechodzi przez M-GUARD przed wejściem do kolejki redaktora)

## Risks

| Risk | Mit. |
|---|---|
| LLM-judge halucynuje (akceptuje zły post) | Drugi pastor robi spot-check 5% PASS posts; metryki w analytics |
| Doctrine card wprowadzony błędnie przez admina | Versioning; pastor zatwierdza zmiany; każda edycja w audit |
| Whitelist faza 1 blokuje wartościowe tematy | Pastor może override jednym klikiem (z nutą "świadoma decyzja, faza 1") |
| Regeneracja w pętli (3 × REGENERATE) | Hard limit 2, potem MANUAL_REVIEW; alert do pastora |
| Sztywne regex'y red flags wpadają w false positive | Pastor edytuje listę przez UI; każda zmiana versioned |
| Doctrine zmienia się w oficjalnym ADW | Pastor odpowiada za update kart; release notes per wersja |
