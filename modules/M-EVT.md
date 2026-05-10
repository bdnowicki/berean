# M-EVT — Events Management

> **Faza**: 3 (Wydarzenia + Analityka) · **Zespół**: 1 osoba · **Estymacja**: 3-4 tyg.

## Cel

Zarządzanie 4-6 wydarzeniami centralnymi rocznie (Warszawa/Kraków, 50-200 osób): rejestracja uczestników, mailing przypominający, integracja z Discordem, post-event NPS. Kompletnie RODO-zgodne (retention 13 mies., samoobsługa).

## Stack

- **Backend**: FastAPI
- **Frontend**: Next.js (formularz publiczny + admin dashboard)
- **Email**: SMTP (np. Resend free tier 3k/mies. lub własny postfix)
- **Calendar**: `ics` library (Python) — generowanie `.ics` dla uczestników
- **QR codes**: `qrcode` library — generowanie QR dla check-in
- **Postgres** — wydarzenia, rejestracje, audit
- **Discord integration**: M-COMM bot

## Funkcjonalności

### F1. Tworzenie wydarzenia (admin)

Admin/pastor tworzy event:
```
Event
  id, slug (np. "konferencja-archeologia-2027"),
  title, subtitle, description (markdown),
  starts_at TIMESTAMP, ends_at TIMESTAMP,
  timezone DEFAULT "Europe/Warsaw",
  location_name, address, city, lat NULL, lng NULL,
  capacity INT, registered_count INT (computed),
  agenda JSONB,  -- [{time, title, speaker}]
  speakers JSONB,
  ticket_price_pln DECIMAL DEFAULT 0,
  registration_opens_at TIMESTAMP, registration_closes_at TIMESTAMP,
  status (draft | published | cancelled | completed),
  cover_image_url, created_at, created_by FK
```

Admin może:
- Stworzyć draft → preview
- Opublikować (widoczne na public site M-WEB)
- Zaktualizować (zmienione pola → email do zarejestrowanych)
- Anulować (z reason; zarejestrowani dostają email)

### F2. Publiczna rejestracja

Formularz na `bnd.pl/wydarzenia/{slug}` (M-WEB):
- Imię i nazwisko (required)
- Email (required, weryfikacja przez double opt-in link)
- Telefon (opcjonalnie, dla SMS przypomnienia)
- Miasto (opcjonalnie, dla statystyk)
- "Skąd dowiedziałeś się o wydarzeniu?" (FB / Discord / znajomy / inne)
- Checkbox RODO: "Zgadzam się na przetwarzanie danych w celu organizacji wydarzenia, retention 13 mies."
- Checkbox newsletter (opcjonalnie, opt-in): "Chcę otrzymywać info o przyszłych wydarzeniach"
- reCAPTCHA / hCaptcha (anti-bot)

### F3. Double opt-in email

```
Po POST /api/events/{slug}/register:
1. Tworzony Registration ze statusem "pending_confirm"
2. Email z linkiem `/api/events/confirm?token={JWT-15min}`
3. Po kliknięciu: status → "confirmed", wysyłany email "Potwierdzenie + szczegóły"
4. Jeśli token expires: ponowna rejestracja
```

Po confirm: lista uczestników w admin UI, miejsce policzone (kapacita).

### F4. Mailing automatyczny

| Event | Trigger | Treść |
|---|---|---|
| Welcome + .ics | Po confirm | Witamy + agenda + .ics file + link do FAQ |
| Reminder 7d | 7 dni przed | Przypomnienie + szczegóły dojazdu + Discord invite |
| Reminder 1d | 1 dzień przed | Final reminder + check-in instructions |
| Day-of | Rano w dzień wydarzenia | "Do zobaczenia za X godzin, oto mapa parkingu" |
| Post-event NPS | 24h po | "Jak Ci się podobało? Oceń 0-10 + komentarz" |
| Post-event materials | 7 dni po | "Slajdy / nagrania / podsumowanie + zaproszenie na Discord" |

Cron worker (Celery beat) co godzinę sprawdza co wysłać.

### F5. Discord integration (przez M-COMM)

- Po publikacji wydarzenia: auto-post w `#spotkania` z embed
- Auto-create Discord Event (Stage / In-person link)
- Po rejestracji w aplikacji: opcjonalny role assignment (jeśli user ma podpięty Discord) — `Uczestnik konferencji 2027`
- Po wydarzeniu: thread "Wrażenia" w `#spotkania`

### F6. Check-in na miejscu

- Każda rejestracja ma unikalny QR code (na potwierdzeniu i .ics)
- Wolontariusz na wydarzeniu skanuje QR mobilną aplikacją (PWA na phone)
- System loguje attendance (`attended_at`)
- Manualne dodanie ("walk-in") z formularza recepcji

### F7. Post-event NPS + statistics

- 24h po: email z linkiem do ankiety (NPS 0-10 + free text)
- Anonimowe (link z token-em jednorazowym)
- Statystyki:
  - Attendance rate (zarejestrowanych / przyszłych)
  - NPS score
  - Top kategorie "skąd dowiedziałeś się"
  - Top miasta uczestników

Eksport CSV/PDF dla pastora.

### F8. RODO compliance

| Element | Implementacja |
|---|---|
| Retention | 13 miesięcy od daty wydarzenia, potem auto-delete (cron) |
| Right to access | Self-service `/moje-dane` (M-AUTH) eksportuje wszystkie rejestracje usera |
| Right to delete | Self-service usuwa rejestracje + mark `deleted` |
| Right to rectify | User w UI edytuje swoje dane przed wydarzeniem |
| Consent | Explicit opt-in checkbox + double opt-in email |
| Data minimization | Tylko niezbędne (imię, email); telefon i miasto opcjonalne |
| Audit | Każda akcja w `event_audit` |

### F9. Płatności (jeśli `ticket_price_pln > 0`)

Faza 3 MVP: free events tylko. Faza 4+: integracja z Przelewy24 / Stripe / PayU. **Nie ma w v1.**

## Schemat danych

```sql
events
  id, slug UNIQUE, title, subtitle, description,
  starts_at, ends_at, timezone,
  location_name, address, city, lat, lng,
  capacity, agenda JSONB, speakers JSONB,
  ticket_price_pln, registration_opens_at, registration_closes_at,
  status, cover_image_url, created_by, created_at, updated_at

event_registrations
  id, event_id FK, user_id FK NULL,  -- NULL jeśli gość bez konta
  full_name, email, phone NULL, city NULL, source NULL,
  status (pending_confirm | confirmed | cancelled | attended | no_show),
  confirm_token VARCHAR NULL, confirmed_at NULL,
  qr_code_url, attended_at NULL,
  newsletter_opt_in BOOL,
  retention_until TIMESTAMP,  -- starts_at + 13 mies
  created_at, updated_at

event_emails
  id, registration_id FK, email_type ENUM, sent_at, opened_at NULL

event_nps
  id, registration_id FK, score INT 0-10, comment TEXT NULL, created_at

event_audit
  id, event_id FK, action, actor_id FK NULL, payload JSONB, timestamp
```

## API / Kontrakty

```
# Public
GET  /api/events                                              → Event[] (published, future)
GET  /api/events/{slug}                                       → Event z agenda
POST /api/events/{slug}/register                              → 202 (email sent)
GET  /api/events/confirm?token=...                            → 302 (page success/failure)
POST /api/events/registrations/{id}/cancel  (token-based)     → 200
POST /api/events/{event_id}/nps             (token)           → 201

# Admin
POST /api/admin/events                  (ADMIN/PASTOR)        → Event
PUT  /api/admin/events/{id}             (ADMIN/PASTOR)        → 200
POST /api/admin/events/{id}/publish                           → 200
POST /api/admin/events/{id}/cancel      {reason}              → 200
GET  /api/admin/events/{id}/registrations                     → Registration[]
POST /api/admin/events/{id}/checkin     {qr_token}            → 200
GET  /api/admin/events/{id}/stats                             → {attendance, nps, sources}
GET  /api/admin/events/{id}/export      ?format=csv           → CSV blob

# Eventy
- event.published { event_id }
- event.registered { event_id, registration_id }
- event.confirmed { event_id, registration_id }
- event.checkin { event_id, registration_id }
- event.nps_submitted { event_id, score }
- event.cancelled { event_id, reason }
```

## Definition of Done

- [ ] Admin może stworzyć event w UI, publish'ować
- [ ] Public site (M-WEB) pokazuje listę i szczegóły
- [ ] Rejestracja działa: form → email confirm → status confirmed
- [ ] Welcome email z .ics → pojawia się w Google/Outlook calendar
- [ ] Auto-mailing: 7d, 1d, day-of przypomnienia
- [ ] Discord integration: po publish event → post w #spotkania, auto-create Discord Event
- [ ] QR check-in działa (PWA scan)
- [ ] Post-event NPS email + zbieranie odpowiedzi
- [ ] Stats dashboard: attendance rate, NPS, sources
- [ ] RODO: retention cron usuwa rejestracje > 13 mies. od starts_at; export/delete działają
- [ ] CSV export dla pastora
- [ ] Capacity limit: po przekroczeniu rejestracja → "lista rezerwowa" (status `waitlist`)

## Edge cases i decyzje

- **Wydarzenie anulowane**: wszyscy zarejestrowani dostają email + zwrot (jeśli płatne, faza 4) + propozycja innej daty
- **Walk-ins**: admin w UI może dodać uczestnika ręcznie (audit log "added by admin Z")
- **Double registration tego samego email**: blokada na poziomie DB unique(`event_id`, `email`)
- **Email expired (TLS issue, spam folder)**: alternatywny self-service: user może wpisać email na `/moje-dane` i zobaczyć rejestracje
- **Capacity reached during pending_confirm rush**: pierwsza confirmed wygrywa; pending bez confirm po 60 min → release
- **Polskie znaki w QR / .ics**: UTF-8 encoding zwalidowane
- **NPS spamming (bot)**: token jednorazowy + rate limit per IP
- **GDPR delete podczas wydarzenia**: status `deleted`, ale `attended` zachowane jako agregat ("przyszło 142 osoby") bez PII
- **Mailing failure (SMTP down)**: retry 3× z backoff, alert do admina; user zawsze może poprosić o manual reminder

## Zależności

**Wchodzi**: M-INFRA (Postgres, SMTP), M-AUTH (user accounts), M-COMM (Discord cross-post + role)
**Wychodzi**: M-ANALYTICS (eventy + statystyki), M-WEB (public form)

## Risks

| Risk | Mit. |
|---|---|
| RODO niezgodność | 13-mies retention cron, double opt-in, samoobsługa export/delete, audit log każdej akcji |
| Email deliverability (SPF/DKIM) | DMARC + SPF skonfigurowane w M-INFRA, test mail-tester.com przed launchem |
| Bot spam rejestracjami | hCaptcha + rate limit per IP + email domain check (disposable email block) |
| Wycieknięte dane uczestników | Dane szyfrowane at-rest; minimalizacja; audyt dostępu (kto i kiedy odczytał listę) |
| Capacity overrun | Hard limit + waitlist; auto-release przy cancel |
| Brak frekwencji (zarejestrowanych przyjdzie 50%) | Dual-channel reminders (email + Discord), confirm-drop w 24h przed nieodpowiadających |
