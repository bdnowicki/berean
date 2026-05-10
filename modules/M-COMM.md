# M-COMM — Community / Discord Integration

> **Faza**: 2 (Media + Społeczność) · **Zespół**: 1 osoba · **Estymacja**: 4 tyg.

## Cel

Dedykowany serwer Discord jako "dom" społeczności: cross-post z FB, sentiment analysis komentarzy, top-alert dashboard dla moderatorów, crowdsourced Q&A z głosowaniem (top 5/tydz. → kolejka M-EDIT). Zaprojektowane multi-guild ready (na fazę 2+ moglibyśmy dorzucić instalację bota na innych serwerach).

## Stack

- **Discord bot**: `discord.py` (async)
- **Backend**: FastAPI (interakcja z M-EDIT, M-AI dla sentiment)
- **Postgres** — lustro: użytkownicy, wątki, pytania Q&A, sentymenty
- **Redis** — cache Discord API rate limits, queue dla heavy ops
- **Sentiment**: M-AI z rolą `judge` (lekki model, np. Haiku)

## Struktura serwera Discord (od dnia 1)

```
🏛 BIBLIA · NAUKA · DOWODY
├── 📋 Informacje
│   ├── #zasady
│   ├── #ogłoszenia (auto-post nowych wpisów FB)
│   └── #spotkania (wydarzenia stacjonarne, zarządzane M-EVT)
├── 📚 Dyskusja tematyczna
│   ├── #archeologia
│   ├── #proroctwa
│   ├── #nauka-i-wiara
│   ├── #manuskrypty
│   └── #ogólne
├── ❓ Q&A
│   ├── #zadaj-pytanie (forum-style channel; każde pytanie = thread; głosowanie)
│   └── #odpowiedzi (auto-post zatwierdzonych odpowiedzi z M-EDIT)
├── 🛠 Moderacja (private — tylko Moderator+)
│   ├── #alerty-toksyczne
│   ├── #alerty-merytoryczne
│   └── #spam-bin
└── 🎤 Głosowe
    ├── Studio (live debaty / spotkania online)
    └── Lounge (luźne rozmowy)
```

Role:
- **Pastor** — pełne uprawnienia, kicks, role management, dostęp do moderacji
- **Moderator** (wolontariusz) — dostęp do moderacji, ban max 7 dni, brak role management
- **Członek** — wszystkie publiczne kanały
- **Czytelnik** (default po Discord OAuth via M-AUTH) — read-only po 24h, potem upgrade do Członek (anti-spam)
- **Bot** — bnd-bot z permissions: send/read/manage messages, manage threads

## Funkcjonalności

### F1. Cross-post FB → Discord

Po `publish.committed` event z M-PUB:
```python
@event_handler("publish.committed")
async def on_published(event):
    if event.channel == "FB":
        post = await edit_service.get_post(event.post_id)
        thread = await bot.create_thread(
            channel_id=settings.DISCORD_OGLOSZENIA_CHANNEL,
            name=f"📌 {post.brief.topic_category}: {extract_hook(post.fb_content)}",
            content=f"{post.fb_content}\n\n🔗 Zobacz na FB: {event.fb_url}\n\n💬 Dyskutuj poniżej."
        )
        await db.save_thread(post_id=post.id, discord_thread_id=thread.id)
```

Każdy thread:
- Header: krótki hook + link do FB
- Pierwszy comment bota: pełny tekst + rozwinięcie 20-30 zdań (z M-EDIT)
- Auto-archive po 30 dniach nieaktywności

### F2. Sentiment analysis komentarzy

Każdy nowy komentarz w wątku tematycznym:
```python
@bot.event
async def on_message(message):
    if message.author.bot: return
    sentiment = await ai.classify_sentiment(
        message.content,
        categories=["positive", "negative_constructive", "negative_toxic", "spam", "off_topic"],
        context_thread=message.thread.starter_message.content if message.thread else None
    )
    await db.save_sentiment(message_id=message.id, **sentiment)

    if sentiment.label == "negative_toxic" and sentiment.confidence > 0.85:
        await alert_moderators(message, reason="toksyczny", urgency="high")
    elif sentiment.label == "spam":
        await auto_hide(message)
```

Klasyfikacja używa M-AI (rola `judge`, prompt z 5 kategoriami).

### F3. Top-alert dashboard (dla moderatorów)

Web UI w aplikacji (nie tylko Discord):
- Lista wątków posortowana po:
  - Liczba komentarzy `negative_toxic` w 24h
  - Liczba `spam` flag
  - Ratio negatywnych do pozytywnych
- Każdy alert: kontekst (poprzednie 5 komentarzy), proponowane akcje:
  - Ukryj komentarz
  - Ostrzegaj autora (DM)
  - Timeout 1h / 24h / 7d
  - Ban
- "Szybka akcja" — moderator klika → bot wykonuje na Discord

### F4. Q&A pipeline z głosowaniem

Forum-style channel `#zadaj-pytanie`:
- Każde nowe pytanie = thread
- Bot wysyła pierwsze auto-reply: "Twoje pytanie jest w kolejce. Reaguj 👍 jeśli też chcesz znać odpowiedź."
- Cron poniedziałek 09:00:
  - Top 5 pytań poprzedniego tygodnia (najwięcej 👍)
  - Każde pytanie → tworzony brief w M-EDIT z `topic_category=qa`, `hook_seed=pytanie_treść`
  - AI generuje 3 warianty odpowiedzi
  - Po pastor approval → publikacja na FB + post w `#odpowiedzi` + odpowiedź w oryginalnym thread

```python
@cron("0 9 * * MON")
async def weekly_qa_top5():
    questions = await db.query_top_questions(window_days=7, limit=5)
    for q in questions:
        brief = Brief(
            topic_category="qa",
            hook_seed=q.summary,
            main_thesis=q.content,
            sources_to_use=[],  # AI sam retrievuje z M-RAG
            constraints=f"Odpowiedz na pytanie zadane przez społeczność."
        )
        await edit_service.create_brief_and_generate(brief)
        await bot.reply_in_thread(q.discord_thread_id,
            "Twoje pytanie zostało wybrane — odpowiedź pojawi się w ciągu 7 dni.")
```

### F5. Anti-spam i automoderacja

- Czytelnik (24h grace): tylko czyta, nie pisze
- Po 24h: auto-promotion do Członek
- Rate limit: max 5 wiadomości/min per user (Discord native + custom)
- Auto-block: link do clickbaitu/scama (URL blacklist + LLM check)
- Cooldown na flame wars: jeśli >10 wiadomości od 1 osoby w 5 min na 1 wątek → cooldown 15 min

### F6. Cross-platform user identity

User loguje się przez Discord OAuth (M-AUTH) → na serwerze ma rolę Członek (M-COMM bot przyznaje rolę).
Vice versa: gdy bot widzi nowego usera na serwerze, sprawdza w M-AUTH czy istnieje konto, jeśli nie — DM "zaloguj się na bnd.pl/login aby brać udział w wydarzeniach".

### F7. Multi-guild ready

Bot zaprojektowany jako multi-guild:
- Tabela `guild_config` (guild_id → settings: kanały announcement/qa/moderation)
- W fazie 2+: bot można zainstalować na innym serwerze (np. parafialnym), z własnym configiem
- W MVP: tylko nasz dedykowany serwer

### F8. Discord notifications dla pastorów

Kanał `#admin-bnd` (private):
- Każdy `negative_toxic` z confidence > 0.9
- Każdy ban
- Każde top-5 Q&A wybrane
- Każdy fail w pipeline (np. M-EDIT regeneration limit)

### F9. Live event support (przyszłość — fazia 3)

Discord Stage Events dla spotkań online:
- Bot tworzy event 7 dni przed datą
- RSVP przez Discord
- Po wydarzeniu: archiwizacja transkryptu (jeśli speakers OK z nagrywaniem)
- Linkowanie z M-EVT

## Schemat danych

```sql
discord_threads
  id, post_id FK NULL, discord_thread_id, discord_channel_id,
  topic_category, created_at, archived_at NULL

discord_messages
  id, discord_message_id, thread_id FK,
  author_discord_id, content, created_at, deleted_at NULL

discord_sentiments
  id, message_id FK, label, confidence, scores JSONB,
  judged_by_model VARCHAR, judged_at

discord_qa_questions
  id, discord_thread_id, asker_id (FK users, nullable), content, summary,
  upvotes INT, status (open|selected|answered|archived),
  brief_id FK NULL (po wyborze do M-EDIT), answered_at NULL

guild_config
  guild_id PK, name, channels JSONB (announcement, qa, moderation, ...),
  active BOOL

discord_moderation_actions
  id, target_user_id, action (hide|warn|timeout|ban), duration NULL,
  reason, by_moderator FK, performed_at, related_message_id FK NULL
```

## API / Kontrakty

```
GET  /api/community/threads        ?status&category         → Thread[]
GET  /api/community/qa             ?status&from             → Question[]
POST /api/community/qa/{id}/select (PASTOR/ADMIN)           → 200 (manual override poza cron)
GET  /api/community/sentiment      ?thread_id&from          → Sentiment[]
GET  /api/community/alerts         (MODERATOR+)             → Alert[]
POST /api/community/alerts/{id}/action (MODERATOR+) {action} → 200
GET  /api/community/stats          ?from&to                 → {messages, alerts, top_threads}

# Bot komendy (Discord)
/pytanie [treść]                                            → tworzy thread w #zadaj-pytanie
/zglos [reason]                                             → flaguje wiadomość do moderatorów
/info                                                       → pomoc bota
/sources [topic]                                            → bot retrievuje z M-RAG i odpowiada (faza 3+)

# Eventy
- community.thread_created { post_id, discord_thread_id }
- community.toxic_alert { message_id, confidence }
- community.qa_top5_selected { question_ids[] }
- community.user_banned { user_id, by, duration, reason }
```

## Definition of Done

- [ ] Discord bot działa, wykonuje commands, reaguje na messages
- [ ] Po publikacji FB → automatic thread w #ogłoszenia
- [ ] Sentiment analysis działa — toksyczne komentarze → alert w #alerty-toksyczne
- [ ] Q&A: użytkownik tworzy thread w #zadaj-pytanie, inni głosują 👍
- [ ] Cron poniedziałek 09:00: top 5 Q&A → briefy w M-EDIT
- [ ] Po pastor approval Q&A → odpowiedź w #odpowiedzi i w oryginalnym threadzie
- [ ] Anti-spam: nowy użytkownik nie może pisać przez 24h, auto-promotion
- [ ] Top-alert dashboard w aplikacji web (poza Discord)
- [ ] Discord OAuth zsynchronizowany z M-AUTH
- [ ] Multi-guild config schema (nawet jeśli używamy 1 guild)

## Edge cases i decyzje

- **Discord rate limit (Cloudflare 1024 req/10s)**: redis-based throttle, batch operations
- **Bot offline > 5 min**: alert do admin tech (M-INFRA monitoring)
- **Toxic user banned 7d wraca i powtarza**: pastor decyduje o permanent ban (audit log)
- **Pytanie Q&A duplikuje istniejące**: AI sprawdza similarity > 0.85 → bot DM "podobne pytanie istnieje, dołącz tutaj: link"
- **Pytanie Q&A z PII (autor wpisał email/telefon)**: bot redactuje (M-AI redact_pii) przed zapisaniem do `discord_qa_questions.content`; oryginał zostaje w Discord (bot nie ma uprawnień do edytowania cudzych wiadomości — tylko ostrzeżenie DM)
- **Polskie znaki w bot replies**: encoding utf-8 jawne, testowane
- **Timezone głosowań Q&A**: cutoff w niedzielę 23:59 czasu polskiego
- **Zmiana właściciela serwera Discord**: bot ma role z `Manage Server` permission, audit log każdego role change
- **Discord deprecation API**: discord.py community-maintained, śledzimy releases

## Zależności

**Wchodzi**: M-INFRA, M-AUTH (Discord OAuth), M-AI (sentiment + Q&A generation), M-RAG (Q&A retrieval), M-EDIT (briefy z Q&A), M-PUB (event publish.committed)
**Wychodzi**: M-EDIT (briefy Q&A), M-EVT (cross-post wydarzeń stacjonarnych), M-ANALYTICS (eventy do DWH)

## Risks

| Risk | Mit. |
|---|---|
| Toxic users zalewają community | Multi-tier moderation (24h grace + sentiment AI + moderatorzy + ban) |
| Discord ban naszego bota za rate limit | Redis throttling, batch ops, monitoring use-rate |
| Pytanie Q&A z PII użytkownika | Redaction przed indexowaniem, edukacja w #zasady |
| Sentiment AI false positive na religijny język ("piekło" w Mt 5 = OK) | Context-aware prompt do M-AI; whitelist phrasing biblijny |
| Nieadekwatne odpowiedzi Q&A (AI halucynuje) | Standardowy pipeline M-EDIT + pastor approval; nie publikujemy bez pastora |
| Brak engagement (nikt nie zadaje pytań) | Promocja w postach FB, kontent seed (pierwszych 20 pytań przygotowanych) |
