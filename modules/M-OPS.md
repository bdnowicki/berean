# M-OPS — Admin & Configuration

> **Faza**: 4 (Optymalizacja, ciągła) · **Zespół**: 1 osoba · **Estymacja**: 3-4 tyg. (initial) + ciągłe iteracje

## Cel

Centralny panel admina łączący wszystkie konfigurowalne aspekty pozostałych modułów: routing AI, system prompty, ban-list, doctrine cards, sloty publikacji, kill-switch, cost dashboard. Bez tego moduły są niezarządzalne — to "kokpit pilota".

## Stack

- **Frontend**: Next.js (admin app, oddzielny od pastor app i public site)
- **Backend**: FastAPI (proxy do API innych modułów + dedicated endpoints)
- **Auth**: M-AUTH (require ADMIN_TECH role)
- **State**: TanStack Query + zustand
- **Forms**: React Hook Form + Zod
- **Editor**: Monaco Editor (do system prompt — versioning + diff)

## Layout admin panelu

```
┌─────────────────────────────────────────────┐
│  Sidebar                                    │
│  🏠 Dashboard         (overview)            │
│  🤖 AI Routing        (M-AI config)         │
│  📜 System Prompts    (versioned editor)    │
│  📚 Doctrine Cards    (28 FB CRUD)          │
│  🚫 Red Flags         (M-GUARD blacklist)   │
│  ✅ Topic Whitelist   (per faza)            │
│  📅 Slots             (M-PUB schedule)      │
│  🚷 Ban List          (M-PUB filter)        │
│  💀 Kill Switch       (emergency)           │
│  💰 Cost Dashboard    (M-AI usage)          │
│  📊 Analytics         (embed M-ANALYTICS)   │
│  👥 Users & Roles     (M-AUTH RBAC)         │
│  📥 Sources           (M-RAG library)       │
│  📜 Audit Logs        (cross-module)        │
│  ⚙️ System Settings    (env vars view)      │
└─────────────────────────────────────────────┘
```

## Funkcjonalności

### F1. Dashboard overview

Single page z KPI:
- Posts published (today / week / month)
- AI cost (today / week / month) — alert jeśli > threshold
- Pending pastor approvals (z timer)
- Active alerts (z M-COMM, M-PUB)
- System health (z M-INFRA — uptime checks)
- Recent activity feed (top 20 events cross-module)

### F2. AI Routing config

Tabela edytowalna `ai_routing` (M-AI):

| role | provider | model | fallback_provider | fallback_model | max_cost_usd | active |
|---|---|---|---|---|---|---|

- Click row → modal z form
- Test button: "Wywołaj testowo z prompts X" — sprawdza czy działa
- History: kto kiedy zmienił (audit log)

### F3. System Prompts editor

Lista promptów (per role, per moduł):
- `M-AI: draft` (z 28 FB ADW)
- `M-AI: final` (rozwinięcie 20-30 zdań)
- `M-AI: judge` (LLM-judge w M-GUARD)
- `M-AI: image-brief`
- `M-COMM: sentiment`

Każdy prompt:
- Monaco editor z syntax highlighting (markdown)
- Wersjonowanie (każdy save = nowa wersja, nigdy nie nadpisuje)
- Diff viewer (porównanie wersji)
- "Test prompt" — wywołuje z sample input
- Rollback do poprzedniej wersji (1 click, audit)
- Variable insertion: `{topic}`, `{sources}`, `{red_flags}` — dropdown

### F4. Doctrine Cards CRUD

Lista 28 FB (M-GUARD):
- Card-based UI, każda z numerem (1-28), tytułem, summary
- Edit modal:
  - Pełny tekst doktryny
  - Key phrases (tagged input)
  - Red flags (sformułowania sprzeczne)
  - Examples aligned (3-5)
  - Examples misaligned (3-5)
- Save → version bump + re-embedowanie (background task) → alert "re-embedding 245/600 chunks"
- Pastor widzi pełną historię edycji

### F5. Red Flags editor

Tabela (M-GUARD `red_flags`):
- Pattern type (regex / keyword / llm_topic)
- Pattern (text input z preview match)
- Doctrine violated (multi-select 1-28)
- Severity (1/2/3)
- Test field: paste post → highlight matches
- Add new / disable (soft delete)

### F6. Topic Whitelist (per faza)

UI:
- Tabela tematów: nazwa, status (allowed / blocked), faza
- Drag-drop między bucket'ami (allowed ⇄ blocked)
- "Promote to phase 2": jednym klikiem przenosi tematy z fazy 2 do fazy 1
- Preview: "Po zmianie ten post nie przeszedłby walidacji" (test sample)

### F7. Slot configuration (M-PUB)

Tabela slotów:
- Kalendarzowy widok tygodnia (D×H grid)
- Click empty cell → dodaj slot (channel, category, priority)
- Click istniejący → edit
- Drag to move
- "Suggested" overlay z benchmark'ów (pokazuje "Czw 9:00 — najmocniejszy slot wg branży")
- Disable/enable slot (zachowuje config, tylko nie używa)

### F8. Ban List editor

Tabela (M-PUB `ban_list`):
- Pattern type / pattern / reason / severity
- Test: paste post → flag matches
- Soft (warning) vs Hard (block)
- Eksport / import (admin może podzielić się listą z innymi instancjami)

### F9. Kill Switch panel

Czerwony przycisk "ZATRZYMAJ PUBLIKACJE":
- Modal: "Powód:" (text required)
- Confirmation 2× klik
- Po włączeniu:
  - Status banner na całym admin panelu: "🛑 PUBLIKACJE WSTRZYMANE (od X minut, by Y, powód: Z)"
  - Wszystkie pending publications → ON_HOLD
  - Slack alert
- Wyłączenie: tylko ADMIN_TECH + reason (audit)
- Historia kill-switch: tabela z czasem ON/OFF, by, reason

Inne podobne kill-switche:
- AI generation kill-switch (zatrzymuje M-AI calls)
- Discord bot kill-switch (zatrzymuje M-COMM bot)

### F10. Cost Dashboard

Embedded z M-ANALYTICS D3 (AI Cost) + dodatkowo:
- Real-time licznik: dzisiejszy spend USD / PLN
- Per-provider breakdown (chart)
- Alert thresholds: edytowalne (np. "alert > 5 USD/dzień", "alert > 50 USD/mies.")
- Auto-fallback: gdy próg przekroczony → automatyczne przerzucenie na tańszego providera (config in M-AI)

### F11. Users & Roles management

(Z M-AUTH `/api/admin/users`)
- Lista użytkowników z filter po roli
- Click → modal: zmiana ról, suspend, reset MFA, force logout
- Audit log każdej zmiany
- Bulk: "Suspend wszystkich `inactive > 365 dni`"

### F12. Sources library admin

Embedded list z M-RAG:
- Pending review (z F5 M-RAG)
- Active sources
- Recently used (top 20 w generowaniu)
- Re-classify (jeśli zmieniony AI model)

### F13. Audit Logs cross-module

Aggregowany feed (z Postgres + ClickHouse):
- Filter: module / event_type / actor / date range
- Każdy event: timestamp, module, actor, type, payload (JSON expandable)
- Eksport CSV
- Search full-text

### F14. System Settings (read-only env)

Lista env vars (bez wartości — dla bezpieczeństwa pokazujemy `***` dla secrets):
- Provider tokens (FB, IG, Anthropic, OpenAI, ...) — `***` + "Edit" → modal z password reveal + warning
- Domains
- SMTP config
- Backup schedule
- Wszystkie z M-INFRA env

Edycja: write do `.env` (przez backend API), trigger restart kontenera (z confirmation modal).

### F15. Bulk operations

- Re-embed wszystkich źródeł (po zmianie modelu embeddingu)
- Re-judge wszystkich postów ostatnich 30 dni (po zmianie doctrine cards)
- Eksport wszystkich danych (M-ANALYTICS) jako .zip

## API / Kontrakty

M-OPS w większości to **proxy/UI nad API innych modułów** — nie wprowadza nowych endpointów backend, tylko:

```
GET  /api/admin/dashboard          → KPIs aggregate
GET  /api/admin/audit              ?module&from&to → AuditEntry[]
POST /api/admin/system-restart     (ADMIN_TECH) {service} → 202
GET  /api/admin/env                (ADMIN_TECH) → masked env
PUT  /api/admin/env                (ADMIN_TECH) {key, value} → 200
POST /api/admin/bulk/re-embed      (ADMIN_TECH) → 202 (Celery task)
POST /api/admin/bulk/re-judge      (ADMIN_TECH) → 202

# Eventy
- admin.config_changed { module, field, by, old_value_hash, new_value_hash }
- admin.kill_switch_toggled { service, enabled, by, reason }
- admin.bulk_op_started { type, by }
- admin.bulk_op_completed { type, duration, success_count, error_count }
```

## Definition of Done

- [ ] Admin panel uruchamia się pod `admin.bnd.pl`, wymaga ADMIN_TECH role
- [ ] Dashboard pokazuje 6 KPI w real-time (posts, cost, pending, alerts, health, activity)
- [ ] AI Routing: zmiana w UI propaguje do M-AI bez restartu
- [ ] System Prompts: edytor z versioning, diff, test, rollback
- [ ] 28 Doctrine Cards w UI z edit + version history
- [ ] Red Flags + Topic Whitelist edytowalne
- [ ] Slot configuration: drag-drop calendar
- [ ] Kill Switch: 2-click confirmation + Slack alert
- [ ] Cost Dashboard: real-time spend + alert thresholds
- [ ] Users & Roles: assignment + audit
- [ ] Audit Logs: full-text search across modules
- [ ] System Settings: env masked + edit z restart
- [ ] Bulk ops: re-embed, re-judge

## Edge cases i decyzje

- **Zmiana System Prompt natychmiastowa**: nie wymaga restartu (M-AI czyta z DB, nie env)
- **Zmiana env (token FB)**: wymaga restart kontenera M-PUB; UI to zaznacza
- **Konflikt edycji** (2 admins równocześnie): optimistic lock + warning "X edytuje to od 5 min"
- **Bulk re-embed kosztowny**: estimuje koszt PRZED uruchomieniem (np. "245 chunks × $0.0001 = $0.025"), wymaga confirm
- **Audit log retention**: 12 mies. w Postgres, potem archiwum do B2
- **Admin zalogowany ale niesumowany (idle 30 min)**: auto-logout (security)
- **Zmiana doctrine card ze sprzecznymi przykładami**: validation pre-save (LLM check że examples_aligned i examples_misaligned są w opozycji)
- **Roll-back system prompt na bardzo starą wersję**: warning "ta wersja jest 6 mies. stara, nowsze zawierają X poprawek"

## Zależności

**Wchodzi**: M-INFRA, M-AUTH (RBAC ADMIN_TECH), wszystkie inne moduły (proxy do ich API)
**Wychodzi**: nikt nie zależy od M-OPS (jest UX nad innymi)

## Risks

| Risk | Mit. |
|---|---|
| Admin zmienia system prompt → AI generuje śmieci | Versioning + rollback + judge fallback (post < 8 → regeneracja, więc safety net) |
| Admin omyłkowo banuje słowo "Bóg" | Test field przed save + warning jeśli pattern matchuje > 50% bazy |
| Wyciek tokenów z env edycji | Audit każdego dostępu + masked display + reveal wymaga ponownego MFA |
| Bulk re-embed paraliżuje system | Background task z rate limit + monitoring + cancelable |
| Nieautoryzowany admin (kompromitacja konta) | MFA obowiązkowe, IP allowlist (opcjonalnie), alert na każdy login admin (Slack) |
