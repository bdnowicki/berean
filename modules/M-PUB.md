# M-PUB — Publishing & Scheduler

> **Faza**: 1 (Silnik treści) · **Zespół**: 1-2 osoby · **Estymacja**: 4-5 tyg.

## Cel

Niezawodna, bezpieczna publikacja zatwierdzonych postów na FB Page + Instagram Business w sztywnych slotach z konfiguracji admina. Wszystkie warstwy bezpieczeństwa: kill-switch, ban-list, soft-publish 30-min, audit log + alerty.

## Stack

- **Backend**: FastAPI
- **Scheduler**: APScheduler (in-process) lub Celery beat
- **Worker**: Celery (Redis broker)
- **Meta API**: `facebook-business` SDK + bezpośrednie HTTP (Graph API v19+)
- **Postgres** — schedule, attempts, audit
- **Notyfikacje**: Slack webhook, SMTP

## Funkcjonalności

### F1. Slot configuration

Admin definiuje sztywne sloty (M-OPS UI):

```
Slot
  id, day_of_week (0-6), time TIME (np. 09:00),
  channel (FB | IG | BOTH), category_filter VARCHAR[] (np. ["archeologia", "proroctwa"]),
  active BOOL, priority INT,
  description TEXT (np. "Czwartek 9:00 — najmocniejszy slot tygodnia")
```

Default config (z STRATEGIA-FANPAGE.md sekcja 2):
| Slot | Dzień | Godz. | Kanał | Charakter |
|---|---|---|---|---|
| Manifest/refleksja | Niedz | 18:00 | BOTH | refleksja, dyskusje |
| Mocny temat | Czw | 09:00 | BOTH | szok-fakt, archeologia |
| Lunch break | Wt | 13:00 | BOTH | nauka, fakty |
| Dowód spektakularny | Sob | 10:00 | BOTH | proroctwa |

### F2. Auto-assignment do slotu

Po `APPROVED` w M-EDIT:
```python
def assign_slot(post: Post) -> ScheduleEntry:
    # Najbliższy slot pasujący do kategorii posta i jeszcze nie zajęty
    candidate_slots = query_slots(category=post.brief.topic_category)
    next_available = find_first_unoccupied(candidate_slots, after=now() + timedelta(hours=1))

    return ScheduleEntry(
        post_id=post.id,
        slot_id=next_available.id,
        scheduled_at=next_available.next_occurrence(),
        status="SCHEDULED"
    )
```

Pastor/admin może override: ręcznie wybrać inny slot, przesunąć w czasie, "publikuj natychmiast" (admin only).

### F3. Soft-publish 30-min window

```
[APPROVED + slot assigned]
        ↓
[scheduled_at - 30 min]  ← 30-min countdown begins, status = "PRE_PUBLISH"
        ↓
[Powiadomienie do pastora + Slack: "Post opublikuje się za 30 min, anuluj jeśli trzeba"]
        ↓
[scheduled_at]
        ↓
[Worker → Meta Graph API → publish]
        ↓
[status = "PUBLISHED" lub "FAILED"]
```

W oknie `PRE_PUBLISH`:
- Pastor/admin może kliknąć "Anuluj" → status `CANCELLED`
- Wymagane uprawnienie: PASTOR lub ADMIN_TECH
- Audit log każdego anulowania z reason

### F4. Kill-switch

- Globalna flaga `publishing_enabled` w Postgres (singleton config)
- Endpoint `/api/publish/kill-switch` (ADMIN_TECH) → wywala wszystkie pending publikacje na hold
- UI: czerwony button "ZATRZYMAJ WSZYSTKO" w admin panelu
- Po włączeniu kill-switch:
  - Worker przestaje wykonywać scheduled tasks
  - Wszystkie `PRE_PUBLISH` → `ON_HOLD`
  - Slack alert do całego admin team
- Wyłączenie: tylko ADMIN_TECH + audit log z reason

### F5. Ban-list filter

Tabela `ban_list`:
| pattern_type | pattern | reason | severity | added_by |
|---|---|---|---|---|
| regex | `\b(politycy|partyjny)\b` | "Bez polityki" | hard | admin |
| keyword | "papież Franciszek" | "Konkretni hierarchowie kościelni" | hard | admin |
| llm_topic | "wojna w Ukrainie" | "Aktualne konflikty" | hard | admin |

Sprawdzane:
1. Przy zatwierdzaniu w M-EDIT (pre-flight, ostrzeżenie)
2. Przy soft-publish (twardy block — `BANNED` status, alert)

LLM check (drogi): tylko gdy regex + keyword nie złapały i admin chce extra warstwę.

### F6. Meta Graph API integration

Konfiguracja:
- FB Page Access Token (long-lived, ~60 dni → auto-refresh przez `Page Access Token`)
- IG Business Account ID + IG Access Token
- Tokens w Postgres encrypted (Fernet)

Publikacja FB:
```python
async def publish_to_fb(post: Post):
    if post.has_image:
        # Upload zdjęcia + post jako attachment
        photo_id = await fb.upload_photo(image_bytes)
        return await fb.create_post(
            page_id=settings.FB_PAGE_ID,
            message=post.fb_content,
            attached_media=[{"media_fbid": photo_id}]
        )
    else:
        return await fb.create_post(message=post.fb_content)
```

Publikacja IG:
```python
async def publish_to_ig(post: Post):
    # IG wymaga obrazka — bez obrazu nie publikujemy
    if not post.has_image:
        raise NoImageForIG()

    container_id = await ig.create_media_container(
        image_url=post.image_url,  # public URL (host na S3-compatible)
        caption=post.ig_content
    )
    return await ig.publish_container(container_id)
```

### F7. Retry i error handling

| Error | Strategia |
|---|---|
| Token expired (190) | Auto-refresh; jeśli fail → alert do admina |
| Rate limit (4/17/32) | Exponential backoff, max 3 próby |
| Image too large | Resize (PIL) → retry |
| Invalid OG metadata | Strip metadata → retry |
| Permanent (validation) | `FAILED`, alert, ręczna interwencja |
| Network/5xx | Retry 3× z backoff (60s, 5min, 15min) |

### F8. Audit log + alerty

Każda akcja:
```
publish_audit
  id, post_id FK, action (scheduled|cancelled|published|failed|banned),
  channel (FB|IG), actor_id NULL (jeśli system),
  meta_response JSONB, error TEXT NULL,
  fb_post_id NULL, ig_post_id NULL,
  timestamp
```

Slack alerts:
- Każda publikacja: `#bnd-publishing` channel
- Każdy fail: `#bnd-alerts` z @here mention
- Kill-switch on/off: `#bnd-alerts`
- Ban-list hit: `#bnd-alerts`

Email digest (admin tygodniowy): "5 publikacji, 0 fails, 1 cancelled, top engagement: ..."

### F9. Multi-channel coordination

Każdy approved post w M-EDIT ma `fb_content` i `ig_content`. M-PUB:
- Tworzy 2 ScheduleEntry (jeden FB, jeden IG, ten sam scheduled_at lub IG +5 min offset)
- Publikuje równolegle
- Status posta `PUBLISHED` tylko jeśli oba kanały sukces; jeśli jeden fail → `PARTIAL_PUBLISHED`, alert

### F10. Manual override

Admin endpointy:
- `POST /api/publish/{post_id}/now` — natychmiast (omija slot)
- `POST /api/publish/{post_id}/reschedule` {new_time} — zmiana slotu
- `POST /api/publish/{post_id}/cancel` — z każdego stanu pre-publish
- `POST /api/publish/{post_id}/republish` (PASTOR) — ponowna publikacja po edycji (rzadko, np. literówka — tworzy nowy post w Meta)

## Schemat danych

```sql
slots
  id, day_of_week INT, time TIME, channel ENUM, category_filter TEXT[],
  priority INT, active BOOL, description, created_at

schedule_entries
  id, post_id FK, slot_id FK NULL,  -- null jeśli manual override
  scheduled_at TIMESTAMP, channel ENUM,
  status (SCHEDULED | PRE_PUBLISH | PUBLISHING | PUBLISHED | FAILED | CANCELLED | ON_HOLD | BANNED),
  attempts INT, last_attempt TIMESTAMP NULL,
  fb_post_id NULL, ig_post_id NULL,
  meta_response JSONB NULL, error TEXT NULL,
  created_at, updated_at

publish_audit  (patrz F8)

ban_list
  id, pattern_type ENUM (regex|keyword|llm_topic), pattern TEXT,
  reason TEXT, severity ENUM (hard|soft), added_by FK, added_at, active BOOL

publishing_config  (singleton)
  publishing_enabled BOOL DEFAULT TRUE,
  killed_by FK NULL, killed_at NULL, kill_reason TEXT
```

## API / Kontrakty

```
POST /api/publish/schedule         {post_id}                  → ScheduleEntry[]   (auto-assign)
POST /api/publish/schedule/manual  {post_id, when, channel}   → ScheduleEntry
POST /api/publish/{id}/cancel      (PASTOR/ADMIN)             → 200
POST /api/publish/{id}/now         (ADMIN)                    → 200
POST /api/publish/{id}/reschedule  (ADMIN) {when}             → 200
POST /api/publish/{id}/republish   (PASTOR)                   → 200
GET  /api/publish/upcoming         ?days=7                    → ScheduleEntry[]
GET  /api/publish/slots            (ADMIN)                    → Slot[]
PUT  /api/publish/slots            (ADMIN)                    → 200
GET  /api/publish/ban-list                                    → BanRule[]
POST /api/publish/ban-list         (ADMIN)                    → 201
DELETE /api/publish/ban-list/{id}  (ADMIN)                    → 204
POST /api/publish/kill-switch      (ADMIN) {enabled, reason}  → 200
GET  /api/publish/audit            ?from&to                   → Audit[]

# Eventy
- publish.scheduled { post_id, slot_id, scheduled_at }
- publish.pre_publish_started { post_id, scheduled_at }   # 30 min ahead
- publish.cancelled { post_id, by, reason }
- publish.committed { post_id, channel, fb_post_id?, ig_post_id? }
- publish.failed { post_id, channel, error }
- publish.kill_switch_toggled { enabled, by, reason }
- publish.banned { post_id, pattern, reason }
```

## Definition of Done

- [ ] Admin definiuje slot Czw 09:00 BOTH; auto-publish działa
- [ ] APPROVED post w M-EDIT trafia do schedule w najbliższym slocie
- [ ] Soft-publish: 30 min przed publikacją pastor dostaje notyfikację, może anulować
- [ ] Kill-switch: kliknięcie zatrzymuje wszystkie pending publikacje, alert na Slacku
- [ ] Ban-list: regex "papież Franciszek" blokuje publikację, alert
- [ ] FB + IG: oba kanały publikują z 2 różnymi tekstami z M-EDIT
- [ ] Token refresh: gdy długi token wygasa za 7 dni, auto-refresh (cron)
- [ ] Retry: gdy Meta zwraca 503, retry 3× z backoffem
- [ ] Failed publication: alert + entry w audit log + status `FAILED` na M-EDIT post
- [ ] Manual override: admin może opublikować "teraz" omijając slot
- [ ] Republish: po publikacji literówka, pastor może triggerować republish (poprzedni FB post zostaje)
- [ ] Audit log: 100% publikacji w `publish_audit`

## Edge cases i decyzje

- **Slot zajęty (już ktoś zaplanował)**: następny dostępny + warning "slot X zajęty, przesunięto na Y"
- **Brak slotu pasującego do kategorii**: post zostaje w `APPROVED` bez schedule, alert do admina "skonfiguruj slot dla kategorii X"
- **IG bez obrazka**: blokujemy schedule IG → alert "post wymaga obrazu dla IG, dodaj lub odznacz IG channel"
- **FB ban Page (poważne ryzyko)**: po 3 fails z kodem dotyczącym policy → automatyczny kill-switch + alert
- **Token rotation**: pastor/admin podpisuje OAuth raz na 60 dni; UI przypomina 7 dni przed wygaśnięciem
- **Timezone**: cały system w `Europe/Warsaw`; sloty są w czasie polskim, scheduled_at w UTC w DB
- **Holiday handling**: admin może oznaczyć datę jako "no-publish" (np. Wielki Piątek refleksyjny, ale szok-fakt nieodpowiedni)
- **Publikacja w przeszłości**: niedozwolone — endpoint waliduje
- **Edycja po publikacji**: Meta nie pozwala edytować postów IG; FB pozwala. Decyzja: po publikacji NIE edytujemy automatycznie; pastor może w UI Meta ręcznie poprawić, audit log to zaznacza

## Zależności

**Wchodzi**: M-INFRA (Postgres, Redis, Celery), M-AUTH, M-EDIT (APPROVED posts)
**Wychodzi**: M-COMM (cross-post FB→Discord po publikacji), M-ANALYTICS (eventy do DWH)

## Risks

| Risk | Mit. |
|---|---|
| FB Page suspended z powodu spamu | Konserwatywne tempo (4-5 postów/tydz.), unikanie clickbaitu, soft-publish 30-min na ostatnią weryfikację |
| Token wycieka | Encrypted-at-rest (Fernet), tylko worker container ma dostęp, rotacja co 60 dni |
| Publikacja błędnego posta | 3 niezależne barriery: M-GUARD (auto), pastor (manual), soft-publish (30 min bezpieczeństwa) |
| Meta API change | Wersja v19+, monitoring deprekacji, alerty z `Graph API Changelog` |
| Zegar systemu zły → publikacja w złym czasie | NTP sync (M-INFRA), monitoring drift |
| Worker pada w trakcie publish | Idempotency: `idempotency_key` per ScheduleEntry, retry safe |
