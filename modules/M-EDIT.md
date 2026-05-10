# M-EDIT — Editorial Workflow

> **Faza**: 1 (Silnik treści) · **Zespół**: 1-2 osoby (full-stack lead + dev) · **Estymacja**: 5-6 tyg.

## Cel

Pełen pipeline od wygenerowania posta przez AI, przez wybór wariantu i edycję przez redaktora, do zatwierdzenia przez pastora — z 72h SLA, eskalacją, regeneracją po feedbacku i nauką z odrzuceń. Centrum decyzyjne aplikacji.

## Stack

- **Backend**: FastAPI + SQLAlchemy
- **Frontend**: Next.js (App Router) + TanStack Query + Zustand
- **Edytor**: Tiptap (rich text, ale ograniczony do tekstu z hashtagami; brak HTML)
- **Real-time**: Server-Sent Events (SSE) dla powiadomień (kolejka pastora)
- **Notyfikacje**: email (SMTP) + Slack webhook + in-app

## Lifecycle posta

```
                ┌──────────────┐
                │   IDEA       │  (z M-COMM Q&A top 5, lub manualny brief)
                └──────┬───────┘
                       │
                  generate (M-AI + M-GUARD)
                       ↓
                ┌──────────────┐
                │   DRAFT      │  (3 warianty + judge results)
                └──────┬───────┘
                       │
                  redaktor wybiera wariant + edytuje
                       ↓
                ┌──────────────────┐
                │  IN_EDITORIAL    │
                └──────┬───────────┘
                       │
                  redaktor klika "do pastora"
                       ↓
                ┌──────────────────┐
                │  PENDING_PASTOR  │  ← 72h SLA timer
                └──┬─────┬──────┬──┘
                   │     │      │
       approve     │     │      │  reject z feedback
                   ↓     │      ↓
            ┌───────────┐│ ┌──────────────────┐
            │ APPROVED  │└→│  REGENERATING    │ → wraca do DRAFT
            └─────┬─────┘  └──────────────────┘
                  │
                  ↓
              [M-PUB scheduler]
```

Stany: `IDEA`, `DRAFT`, `IN_EDITORIAL`, `PENDING_PASTOR`, `REGENERATING`, `APPROVED`, `SCHEDULED`, `PUBLISHED`, `REJECTED_TERMINAL`, `ARCHIVED`.

## Funkcjonalności

### F1. Brief composer (input do generacji)

Redaktor (lub admin/pastor) tworzy brief:
```
Brief
  - topic_category: enum (archeologia / proroctwa / nauka / ...)
  - hook_seed: string (np. "Pieczęć Ezechiasza znaleziona w 2015 r.")
  - main_thesis: string (1-2 zdania)
  - sources_to_use: source_id[] (z M-RAG, opcjonalne)
  - bible_refs: list[string] (np. ["2 Krl 18,1-2", "Iz 36-39"])
  - target_slot: timestamp (opcjonalne sugerowane)
  - constraints: string (np. "BEZ wzmianki o EGW")
```

Brief może też być wygenerowany przez AI z Q&A (M-COMM).

### F2. Generowanie 3 wariantów

```python
async def generate_post_drafts(brief: Brief) -> Post:
    # M-RAG retrieval based on brief
    sources = await rag.retrieve(brief.hook_seed + " " + brief.main_thesis, k=8,
                                  filters={"source_type": brief.allowed_sources})

    # M-AI generation z M-GUARD pre-prompt
    candidates_with_judge = await guard.safe_generate(
        AIRequest(role="final", messages=[..., sources_context]),
        topic=brief.topic_category
    )

    post = Post(
        brief_id=brief.id,
        status="DRAFT",
        variants=[
            Variant(
                content=c.content,
                judge_score=j.overall,
                judge_issues=j.issues,
                provider=c.provider,
                model=c.model,
                cost_usd=c.cost_usd,
            ) for c, j in candidates_with_judge
        ]
    )
    return post
```

### F3. Editor UI (faza dla redaktora)

Komponenty:
- **3 panele** obok siebie z wariantami (header: provider, model, judge_score)
- Każdy panel: pełny tekst posta + lista issues z M-GUARD (jeśli REVIEW)
- Akcje per wariant: **WYBIERZ**, **REGENERUJ Z FEEDBACKIEM**, **POŁĄCZ Z INNYM** (cherry-pick fragmentów)
- Po wyborze: przejście do edytora pełnoekranowego
- Tiptap edytor z:
  - Sprawdzanie polskich znaków (lint: braknący diakrytyk → warning)
  - Liczba znaków hooka (target 40-80)
  - Liczba zdań w głównej części (target 3-7)
  - Liczba zdań w rozwinięciu (target 20-30)
  - Hashtag suggester (oparte na metadata briefu)
- Side panel: użyte źródła z M-RAG (kliknięcie → cytat)
- Akcje na końcu: **POPROŚ O INNĄ WERSJĘ** | **ODRZUĆ I ZAMKNIJ** | **DO PASTORA**

### F4. Pastor UI (faza zatwierdzania)

Po `PENDING_PASTOR`:
- Pastor widzi w `/pastor/queue` listę postów z timerem ("za 18h SLA")
- Karta posta: pełny tekst (FB + IG wersje), źródła użyte, judge_score, edycje redaktora
- 4 akcje:
  - **AKCEPTUJ** (1 click → `APPROVED` → schedule via M-PUB)
  - **AKCEPTUJ Z DROBNĄ POPRAWKĄ** (otwiera edytor inline → po zapisaniu `APPROVED`)
  - **ODRZUĆ I REGENERUJ** (formularz feedback → `REGENERATING`)
  - **ODRZUĆ TERMINALNIE** (formularz powodu → `REJECTED_TERMINAL` → trafia do biblioteki "do nauki")
- Kanał Slack/email: powiadomienie po pojawieniu się w kolejce + przed deadline'em (24h, 6h)

### F5. Regeneration loop

```python
async def regenerate(post: Post, feedback: str):
    post.regen_attempts += 1
    if post.regen_attempts > 3:
        post.status = "MANUAL_REVIEW"
        notify_admin("Post X przekroczył 3 regeneracje, wymaga interwencji")
        return

    # Augmentuj brief o feedback i archiwum negative examples
    brief = post.brief.augment(
        rejection_feedback=feedback,
        previous_attempts=post.variants,
        negative_examples=await get_negative_examples_similar_to(post.brief)
    )

    new_drafts = await generate_post_drafts(brief)
    post.status = "DRAFT"
    post.variants = new_drafts.variants
    notify_redaktor(post)
```

### F6. SLA tracking

- Cron co 5 min: szuka postów `PENDING_PASTOR > 72h - X`
- Eskalacja co 24h:
  - 24h przed deadline: powiadomienie pastora
  - 6h przed: powiadomienie pastora + admina
  - Po deadline: eskalacja do drugiego pastora (backup)
  - Post nadal `PENDING_PASTOR` — nie wygasa, tylko alert
- Konfiguracja per pastor: dyżury (kalendarz "kto w danym tygodniu odbiera kolejkę")

### F7. Archive "do nauki"

Wszystkie `REJECTED_TERMINAL` trafiają do biblioteki:
- Zapisany feedback pastora
- Tagged automatycznie przez AI (kategoria błędu: ton, doktryna, fakty, stylistyka)
- Używane jako few-shot **negative examples** w przyszłych generacjach (niedopuszczalne wzorce)

```python
def get_negative_examples_similar_to(brief) -> list[str]:
    # Znajdź 3 najnowsze REJECTED_TERMINAL z tej samej kategorii
    # Dodaj do system prompta "Unikaj wzorca XYZ — patrz przykłady poniżej"
```

### F8. Statystyki redakcyjne (do M-ANALYTICS)

Eventy emitowane:
- `post.created`, `post.variants_generated`, `post.editor_chose_variant`
- `post.editor_regen_requested`, `post.sent_to_pastor`
- `post.pastor_approved`, `post.pastor_rejected`, `post.pastor_edited_inline`
- `post.regen_attempts_exceeded`, `post.archived_for_learning`

M-ANALYTICS agreguje: czas redaktor→pastor, czas pastor→decyzja, % regeneracji, % terminal reject, top przyczyny odrzuceń.

### F9. Multi-channel content (FB + IG)

Po wyborze wariantu, AI generuje 2 wersje:
- **FB**: hook 40-80 znaków + 3-7 zdań główne + CTA + rozwinięcie 20-30 zdań + hashtagi
- **IG**: hook 40-80 znaków + 3-5 zdań kondensat + CTA + hashtagi (limit 30) + carousel-friendly (max 5 plansz)

Redaktor edytuje obie. Pastor widzi obie. Akceptacja jednoczesna.

## Schemat danych

```sql
briefs
  id, topic_category, hook_seed, main_thesis,
  sources_to_use UUID[], bible_refs TEXT[],
  target_slot TIMESTAMP NULL, constraints TEXT,
  created_by FK, created_at

posts
  id, brief_id FK, status (enum), regen_attempts INT DEFAULT 0,
  selected_variant_id NULL, fb_content TEXT, ig_content TEXT,
  scheduled_at TIMESTAMP NULL, published_at NULL,
  pastor_assignee FK NULL, sla_deadline TIMESTAMP NULL,
  created_at, updated_at

post_variants
  id, post_id FK, content TEXT, provider, model,
  judge_score FLOAT, judge_issues JSONB,
  cost_usd DECIMAL, created_at

post_events  (audit log)
  id, post_id FK, event_type, actor_id FK,
  payload JSONB, created_at

post_archive_negative  (do nauki)
  id, brief_id FK, content TEXT, rejection_reason,
  category_tags TEXT[], created_at

pastor_duty_schedule
  id, pastor_id FK, week_start DATE, week_end DATE
```

## API / Kontrakty

```
POST /api/briefs                   {topic_category, ...}     → Brief
POST /api/posts/generate           {brief_id}                → Post (DRAFT)
GET  /api/posts                    ?status&assignee          → Post[]
GET  /api/posts/{id}                                         → Post + variants + events
POST /api/posts/{id}/select-variant {variant_id}             → 200
PATCH /api/posts/{id}/content      {fb?, ig?}                → 200
POST /api/posts/{id}/request-regen {feedback}                → 202
POST /api/posts/{id}/send-to-pastor                          → 200
POST /api/posts/{id}/approve       (PASTOR)                  → 200 → schedule
POST /api/posts/{id}/reject        (PASTOR) {feedback}       → 200 (REGENERATING)
POST /api/posts/{id}/reject-terminal (PASTOR) {reason}       → 200 (ARCHIVED)
GET  /api/pastor/queue             (PASTOR)                  → Post[]
GET  /api/pastor/duty-schedule                               → Schedule
PUT  /api/pastor/duty-schedule     (ADMIN)                   → 200

# Eventy (Redis pub/sub) — patrz F8
```

## Definition of Done

- [ ] Redaktor może stworzyć brief, kliknąć "Generuj" i dostać 3 warianty w ≤ 60s
- [ ] Editor UI: redaktor wybiera wariant, edytuje (FB+IG), wysyła do pastora
- [ ] Pastor widzi kolejkę z SLA timerem, klika "Akceptuj" → post leci do M-PUB
- [ ] Pastor pisze feedback → post wraca do redaktora z nowymi wariantami z uwzględnionym feedbackiem
- [ ] SLA: po 24h przed deadline pastor dostaje email; po 72h eskalacja do drugiego pastora
- [ ] Maks 3 regen → MANUAL_REVIEW + alert
- [ ] REJECTED_TERMINAL → archive → następny generation tego topiku zawiera "unikaj X" w prompcie
- [ ] FB i IG wersje obie generowane, edytowalne osobno, akceptowane razem
- [ ] Pastor może edytować inline i akceptować jednym klikiem
- [ ] Wszystkie eventy w `post_events` (audit log)

## Edge cases i decyzje

- **Pastor zaakceptował, ale potem chce cofnąć**: window 30 min (przed soft-publish, M-PUB) — endpoint `unapprove`. Po 30 min: tylko admin może cofnąć z M-PUB
- **Redaktor zniknął w trakcie edycji**: auto-save co 30s (Tiptap supports); inny redaktor widzi "edytuje X od 2h" → może przejąć
- **Dwóch redaktorów na tym samym poście**: optimistic locking — drugi dostaje konflikt z propozycją merge
- **Brief bez sources_to_use**: AI sam wybiera retrieval z M-RAG na podstawie hook_seed
- **Brak pastora w dyżurze (urlop)**: kolejka idzie do drugiego pastora; jeśli obaj nieaktywni — admin dostaje alert
- **AI nie generuje sensownego wariantu (3× MANUAL_REVIEW)**: brief zostaje w `IDEA`, admin/pastor edytuje brief ręcznie
- **Bardzo długi rozwinięcie 20-30 zdań**: limit zaszyty w prompt `final` w M-AI; redaktor może rozwinąć bardziej, ale pastor widzi warning "post > 35 zdań"

## Zależności

**Wchodzi**: M-INFRA, M-AUTH (RBAC: redaktor, pastor), M-AI (generation, regeneration), M-GUARD (judge w pipeline), M-RAG (retrieval źródeł)
**Wychodzi**: M-PUB (przyjmuje APPROVED posts do schedule), M-COMM (Q&A może tworzyć briefy), M-ANALYTICS (eventy)

## Risks

| Risk | Mit. |
|---|---|
| Pastor zatwierdza wszystko bez czytania (rubber stamp) | Metryki czasu na review — alert jeśli avg < 30s; pre-flight: post musi być otwarty min. 60s zanim "Akceptuj" odblokuje |
| Redaktor publikuje przez pomyłkę bez pastora | Twardy guard w API: tylko pastor ma `require_role(PASTOR)` na approve |
| Nieskończona pętla regen | Hard limit 3 + MANUAL_REVIEW |
| Konflikt między FB a IG content (różne fakty) | AI generuje obie z tego samego briefu — gwarancja spójności; pastor widzi diff jeśli redaktor zmienił tylko jedną |
| Pastor nie dostaje notyfikacji (email spam folder) | Multi-channel: email + Slack + in-app SSE; co najmniej jeden zadziała |
