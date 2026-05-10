# M-ANALYTICS — DWH + Dashboards

> **Faza**: 3 (Wydarzenia + Analityka) · **Zespół**: 1 osoba · **Estymacja**: 3-4 tyg.

## Cel

Hurtownia danych (ClickHouse) + dashboardy (Metabase) do podejmowania decyzji redakcyjnych: które tematy działają, które sloty dają engagement, ile kosztuje AI, jak pastor performuje. Wszystko self-hosted, open-source, koszt zerowy.

## Stack

- **DWH**: ClickHouse (open-source, columnar, idealne do agregacji)
- **ETL**: Python (Celery beat) + httpx (FB/IG API), psycopg (czytanie z app Postgres)
- **Visualization**: Metabase (open-source, łatwe dashboardy)
- **Embedded reports**: API Metabase + tokens dla embed w admin UI
- **Eksport**: CSV, Google Sheets (przez Metabase API)

## Źródła danych

| Źródło | Typ | Częstotliwość | Klucz |
|---|---|---|---|
| FB Insights API | external | hourly | post_id, page_id |
| IG Insights API | external | hourly | media_id |
| Discord stats (M-COMM) | internal | daily | thread_id, message_id |
| App Postgres (posts, briefs, ai_usage, events) | internal | hourly | post_id |
| Audit logs (M-EDIT, M-PUB, M-GUARD) | internal | hourly | event_id |

## ClickHouse schema (faktografia)

```sql
-- Fakty
CREATE TABLE fact_post_engagement (
    date Date,
    timestamp DateTime,
    post_id UUID,
    channel Enum8('FB'=1, 'IG'=2),
    impressions UInt32,
    reach UInt32,
    likes UInt32,
    comments UInt32,
    shares UInt32,
    saves UInt32,
    clicks UInt32,
    engagement_rate Float32  -- (likes+comments+shares) / reach
) ENGINE = MergeTree() ORDER BY (date, post_id, channel);

CREATE TABLE fact_ai_usage (
    date Date,
    timestamp DateTime,
    request_id UUID,
    post_id UUID NULL,
    role LowCardinality(String),
    provider LowCardinality(String),
    model LowCardinality(String),
    tokens_in UInt32,
    tokens_out UInt32,
    cost_usd Decimal(10, 6),
    latency_ms UInt32,
    cached UInt8
) ENGINE = MergeTree() ORDER BY (date, role);

CREATE TABLE fact_editorial_events (
    date Date,
    timestamp DateTime,
    post_id UUID,
    event_type LowCardinality(String),  -- created, generated, editor_chose, sent_to_pastor, approved, rejected, etc.
    actor_id UUID NULL,
    actor_role LowCardinality(String),
    payload String  -- JSON
) ENGINE = MergeTree() ORDER BY (date, event_type);

CREATE TABLE fact_discord_messages (
    date Date,
    timestamp DateTime,
    thread_id UUID,
    message_discord_id String,
    sentiment LowCardinality(String),
    sentiment_confidence Float32,
    is_reply UInt8
) ENGINE = MergeTree() ORDER BY (date, thread_id);

CREATE TABLE fact_event_registrations (
    date Date,
    timestamp DateTime,
    event_id UUID,
    registration_id UUID,
    action Enum8('registered'=1, 'confirmed'=2, 'attended'=3, 'cancelled'=4),
    source LowCardinality(String),
    city LowCardinality(String) NULL
) ENGINE = MergeTree() ORDER BY (date, event_id);

-- Wymiary
CREATE TABLE dim_posts (
    post_id UUID,
    topic_category String,
    has_image UInt8,
    fb_word_count UInt32,
    ig_word_count UInt32,
    judge_score_first_variant Float32,
    selected_variant_provider String,
    selected_variant_model String,
    pastor_id UUID,
    sources_used Array(UUID),
    published_at DateTime NULL,
    slot_day_of_week UInt8 NULL,
    slot_time String NULL
) ENGINE = ReplacingMergeTree(updated_at) ORDER BY post_id;
```

## ETL

### ETL.1: FB/IG Insights (hourly)

```python
@celery.task
async def etl_fb_insights():
    posts_to_fetch = await db.query("""
        SELECT post_id, fb_post_id FROM fact_post_engagement
        WHERE published_at > now() - interval '30 days'
          AND fb_post_id IS NOT NULL
    """)
    async with httpx.AsyncClient() as client:
        for batch in chunked(posts_to_fetch, 50):
            data = await fb_api.batch_insights(batch, metrics=[
                "post_impressions", "post_engaged_users", "post_reactions_by_type_total",
                "post_clicks_by_type", "post_shares"
            ])
            await ch.insert("fact_post_engagement", data)
```

### ETL.2: Internal events (hourly)

Pull z Postgres `post_events`, `ai_usage`, `discord_messages`, `event_registrations` → push do ClickHouse.

### ETL.3: Discord ETL (daily)

Bot codziennie (M-COMM) eksportuje thread/message statystyki → ClickHouse.

### ETL.4: Backfill

Komenda CLI: `bnd-etl backfill --from 2027-01-01 --source fb` (na wypadek zmian schema).

## Dashboardy w Metabase

### D1: Engagement Overview (główny)

- Daily impressions / reach / engagement rate (timeline, 90 dni)
- Top 10 postów wg engagement_rate
- Heatmap engagement per slot (day of week × hour)
- Comparison FB vs IG (wskaźniki side-by-side)

### D2: Content Performance

- Engagement per `topic_category` (proroctwa vs archeologia vs nauka...)
- Engagement vs has_image (bool)
- Engagement vs `fb_word_count` (scatter — czy długie posty działają lepiej?)
- Engagement vs hour-of-publication
- Top wykorzystywanych źródeł (po `sources_used`)

### D3: AI Cost Dashboard

- Total cost USD per day / per month
- Cost per provider (Anthropic / OpenAI / DeepSeek...)
- Cost per role (draft / final / judge)
- Cost per post (avg, p50, p99)
- Cache hit rate (oszczędność $)
- Alert: jeśli daily > budget

### D4: Editorial Workflow

- Czas redaktor → pastor (avg, p50, p95)
- Czas pastor → decyzja (avg, p50, p95)
- % approved / rejected / regenerated / terminal
- Top przyczyny rejection (group by judge_issue_type)
- Pastor performance: liczba decyzji, czas avg, accept rate
- Brief → publication czas total

### D5: Community Health

- Daily messages count per kanał Discord
- Sentiment distribution (positive / neg-constructive / neg-toxic / spam)
- Top contributors (najbardziej aktywni członkowie)
- Q&A pipeline: liczba pytań, top 5 weekly, time-to-answer
- Moderation actions per dzień

### D6: Events

- Registrations per event
- Confirmation rate (registered → confirmed)
- Attendance rate (confirmed → attended)
- NPS distribution per event
- Sources per event (skąd uczestnicy się dowiedzieli)
- City heatmap

### D7: Cohort + Funnel

- User funnel: FB → strona → Discord → Q&A → wydarzenie
- Cohort retention (jak długo członkowie są aktywni)
- Cross-channel: % członków Discord którzy też przyszli na wydarzenie

## API / Kontrakty

```
GET  /api/analytics/dashboard/{id}        → embedded Metabase URL z tokenem (TTL 1h)
GET  /api/analytics/export/posts          ?from&to → CSV
GET  /api/analytics/export/ai-usage       ?from&to → CSV
GET  /api/analytics/etl/status            (ADMIN_TECH) → {last_run, errors[], lag_minutes}
POST /api/analytics/etl/trigger           (ADMIN_TECH) {source} → 202 (manual run)

# Eventy
- analytics.etl_completed { source, rows_inserted, duration_ms }
- analytics.etl_failed { source, error }
- analytics.budget_alert { date, threshold, current_cost }
```

## Definition of Done

- [ ] ClickHouse w Docker Compose (M-INFRA), tabele utworzone
- [ ] ETL.1 (FB/IG) działa hourly, zbiera metryki ostatnich 30 dni
- [ ] ETL.2-4 działają, dane w ClickHouse rośnie
- [ ] Metabase deployed pod `metabase.bnd.pl`, połączony z ClickHouse
- [ ] 7 dashboardów (D1-D7) gotowych
- [ ] Embedded report w admin panelu (M-OPS) — pastor widzi D1 i D4 bez logowania do Metabase
- [ ] CSV export dla każdego dashboardu
- [ ] Budget alert: jeśli daily AI cost > X PLN → Slack alert (ETL emituje event)
- [ ] Backfill CLI dostępny i przetestowany
- [ ] Doc strony "jak czytać dashboardy" — dla pastora

## Edge cases i decyzje

- **FB API rate limit**: spread requests across hour z cron'em + retry; fail nie krytyczny (kolejna iteracja podejdzie)
- **Token expired**: fallback do `M-PUB` token refresh; alert jeśli rotation fail
- **Schema migration ClickHouse**: `ALTER TABLE` lub blue-green table swap
- **Backfill kosztowny (drogi FB API)**: rate limit + estymowanie ile tokenów
- **GDPR + analytics**: ClickHouse trzyma `user_id` (UUID), ale brak PII; w razie delete usera — mark `user_id = NULL`
- **Metabase login**: pastor + admin mają dostęp; redaktor opcjonalnie (tylko D1 i D2)
- **Stale ETL**: alert jeśli last_run > 90 min (M-INFRA monitoring)
- **Bardzo duża tabela `fact_post_engagement`**: TTL 24 mies. (po tym agregat-only); ClickHouse natywnie obsługuje TTL
- **Manual data entry** (np. wydarzenie offline): admin może wprowadzić attendance manually w UI; flag `source=manual`

## Zależności

**Wchodzi**: M-INFRA (ClickHouse, Metabase), wszystkie inne moduły (źródła danych)
**Wychodzi**: insighty dla pastora/redaktora — nie ma "consumera" technicznego, ale wpływa na decyzje (np. zmiana slotów)

## Risks

| Risk | Mit. |
|---|---|
| ETL pada → brak danych → "lecimy na ślepo" | Multi-source redundancy (jeśli FB padnie, internal events nadal lecą); alert |
| Niespójność (Postgres mówi 100 likes, FB 95) | Single source of truth: ClickHouse + dokumentacja "FB source canonical for engagement" |
| ClickHouse zapełnia dysk | TTL polityka + monthly archive starych do B2 |
| Metabase wycieka (publiczny URL z tokenem) | Krótki TTL tokenów, IP allowlist, audit każdego embed |
| FB API deprecation metryk | Monitoring `Graph API Changelog`, abstract layer w ETL.1 |
