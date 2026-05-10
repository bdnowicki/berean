# M-WEB — Public Site

> **Faza**: 4 (Optymalizacja, ciągła) · **Zespół**: 1 osoba · **Estymacja**: 3 tyg.

## Cel

Publiczny landing + portal społeczności: misja, ostatnie wpisy, formularz Q&A, rejestracja na wydarzenia, samoobsługa konta (RODO). Mobile-first, SEO-friendly, lekki (statyczny tam gdzie się da).

## Stack

- **Framework**: Next.js 14+ (App Router, RSC)
- **Style**: Tailwind CSS + shadcn/ui components
- **Forms**: React Hook Form + Zod
- **Auth**: M-AUTH (NextAuth lub własny adapter z JWT z FastAPI)
- **CMS-light**: MDX dla statycznych stron (FAQ, regulamin, polityka prywatności)
- **Analytics**: opt-in Plausible self-hosted (M-INFRA) — bez Google Analytics (zgodne z RODO bez cookie banner)
- **Deploy**: Static export gdzie możliwe, ISR dla list ostatnich postów

## Struktura strony

```
/                              → Landing (misja + 3 ostatnie posty + CTA)
/o-nas                         → Misja, zespół, co robimy
/wpisy                         → Lista ostatnich postów (z FB embed)
/wpisy/[slug]                  → Szczegóły posta (z rozwinięciem 20-30 zdań)
/pytania                       → Formularz Q&A + lista odpowiedzi
/pytania/[id]                  → Szczegóły pytania + odpowiedź
/wydarzenia                    → Lista nadchodzących wydarzeń
/wydarzenia/[slug]             → Szczegóły + rejestracja
/discord                       → Zaproszenie + opis społeczności
/zrodla                        → Lista publicznych źródeł (z M-RAG, "active=true & public=true")
/faq                           → MDX
/regulamin                     → MDX
/polityka-prywatnosci          → MDX
/moje-dane                     → User self-service (M-AUTH /api/me/*)
/login                         → Discord OAuth + email login
/admin                         → 302 → admin.bnd.pl
```

## Funkcjonalności

### F1. Landing page

- Hero: hasło "Biblia · Nauka · Dowody" + 1-zdaniowy manifest + CTA "Dołącz do Discord"
- Sekcja "Ostatnie wpisy" — 3 najnowsze opublikowane posty (z M-EDIT, ISR co 1h)
- Sekcja "Co znajdziesz" — 4 ikony: Archeologia / Proroctwa / Nauka / Manuskrypty
- Sekcja "Zaplanuj się" — najbliższe wydarzenie (z M-EVT)
- Sekcja "Zadaj pytanie" — formularz inline lub link do `/pytania`
- Footer: linki do FB, IG, Discord, regulamin, polityka, kontakt

### F2. Lista wpisów (`/wpisy`)

- Pagination 12 per page
- Filter po kategorii (archeologia / proroctwa / nauka / ...)
- Search po hooku/treści
- Każda karta: hook (40-80 znaków), data, kategoria, link do FB, link do szczegółów
- ISR (revalidate co 1h)
- SEO: meta tags per kategoria

### F3. Szczegóły wpisu (`/wpisy/[slug]`)

- Hook (h1)
- Pełny tekst (3-7 zdań)
- Rozwinięcie 20-30 zdań (w `<article>` z proper structure)
- CTA / pytanie z posta
- Cytaty biblijne z linkami do `/biblia/[book]/[chapter]` (faza 5+)
- Hashtags
- Linki: FB original post, Discord thread (z M-COMM)
- Sekcja "Powiązane wpisy" (z analytics — top 5 z tej kategorii)
- Open Graph metadata (dla share na social media)

### F4. Q&A formularz (`/pytania`)

- Form: pytanie (min 20, max 500 znaków), opcjonalny email do powiadomienia
- reCAPTCHA / hCaptcha
- Po submit: redirect do `/pytania/{id}` z threadem Discord linkiem
- Lista wszystkich pytań (publiczne, z głosowaniem upvote — przez Discord OAuth lub sesja)
- Pytania odpowiedziane: pełna odpowiedź na stronie + link do FB
- Filter: open / answered / top voted

### F5. Wydarzenia (`/wydarzenia`)

- Lista nadchodzących z M-EVT (ISR co 30 min)
- Każde: cover, title, date, city, "Zarejestruj się" / "Zapisy zamknięte" / "Pełne — lista rezerwowa"
- Click → szczegóły z agenda, speakers, mapa lokalizacji (OpenStreetMap embed)
- Form rejestracji inline (proxied to M-EVT API)
- Dla zalogowanych: wstępnie wypełnione imię + email

### F6. Discord invite (`/discord`)

- Wyjaśnienie czym jest Discord (dla niezaznajomionych)
- Co znajdziesz na serwerze (struktura kanałów)
- Zasady społeczności (link do `/regulamin`)
- Button "Dołącz teraz" → Discord invite link (z monitoring expirayi)
- Stats: liczba członków, średnia aktywność (publiczne agregaty z M-COMM)

### F7. User self-service (`/moje-dane`)

(M-AUTH endpointy)

- Logged in user widzi:
  - Swoje dane (display_name, email, role, joined date)
  - Listę swoich pytań Q&A
  - Listę swoich rejestracji na wydarzenia
  - Aktywne sesje (z `terminate session` button)
- Akcje:
  - Edytuj profil
  - Zmień hasło (jeśli email-based)
  - Włącz/wyłącz TOTP
  - Eksportuj wszystkie dane (mail z linkiem)
  - Usuń konto (14d soft-delete)

### F8. Login (`/login`)

- 2 opcje:
  - "Zaloguj przez Discord" (główny CTA — 70% userów to community members)
  - "Email + hasło" (dla redaktorów / pastorów)
- Magic link (passwordless) — opcjonalnie w fazie 5
- TOTP prompt po haśle (jeśli włączony)
- "Zapomniałem hasła" → reset link

### F9. SEO i performance

- Static export gdzie możliwe (FAQ, regulamin, polityka — pełne SSG)
- ISR dla treści dynamicznej (wpisy, wydarzenia)
- Open Graph + Twitter Card per strona
- robots.txt + sitemap.xml
- Schema.org structured data (Article, Event)
- Image optimization (next/image)
- Core Web Vitals: LCP < 2.5s, CLS < 0.1, FID < 100ms (mobile)
- Polski jako default lang, opcjonalnie EN w fazie 5+

### F10. Accessibility

- WCAG AA compliance
- Keyboard navigation w formularzach
- Screen reader support (aria-labels, role attributes)
- Color contrast 4.5:1 minimum
- Focus indicators widoczne
- Alt text dla obrazów (jeśli dekoracyjne — `alt=""`)
- Forms: jasne error messages, label association

### F11. RODO compliance public

- Banner cookie/consent: minimalny, tylko jeśli Plausible (analytics) — opt-in, bo Plausible nie używa cookies (ale konserwatywnie pokazujemy)
- Polityka prywatności w MDX z punktami:
  - Jakie dane zbieramy (rejestracja: imię+email, wydarzenia: imię+email+miasto)
  - Po co (organizacja wydarzeń, Q&A, social media)
  - Komu udostępniamy (Discord, Meta — przez ich publish API; AI providerzy — bez PII)
  - Retention: 13 mies. wydarzenia, 12 mies. audit logs
  - Prawa użytkownika: access, delete, rectify, portability
  - Kontakt do administratora danych
- "Edytuj zgody" link w footerze

## API / Kontrakty

M-WEB konsumuje API innych modułów (BFF pattern, jeśli potrzebne):

```
GET  /api/web/landing                 → {recent_posts, next_event, q_count}
GET  /api/web/posts                   ?page&category&search → PostPublic[]
GET  /api/web/posts/{slug}            → PostPublic + related
GET  /api/web/questions               ?status → QuestionPublic[]
POST /api/web/questions               {content, email?} → 201
GET  /api/web/events                  ?upcoming → EventPublic[]
POST /api/web/events/{slug}/register  (proxy do M-EVT) → 202

# Brak własnych eventów; konsumuje z M-EDIT, M-COMM, M-EVT
```

## Definition of Done

- [ ] Strona dostępna pod `bnd.pl` (lub wybraną domeną)
- [ ] Lighthouse score: Performance > 90, A11y > 95, SEO > 95
- [ ] Mobile-responsive (test na iPhone SE i Galaxy S20)
- [ ] Landing pokazuje 3 ostatnie posty z M-EDIT
- [ ] Lista wpisów paginowana, filtry działają
- [ ] Szczegóły wpisu z rozwinięciem 20-30 zdań renderują się poprawnie
- [ ] Q&A formularz: po submit thread w Discord (M-COMM) + email confirm (jeśli podany)
- [ ] Wydarzenia: rejestracja end-to-end z double opt-in
- [ ] User self-service: login → /moje-dane → eksport / delete działają
- [ ] Discord login działa (OAuth callback)
- [ ] Polityka prywatności + regulamin opublikowane
- [ ] sitemap.xml generowany, indexowany przez Google (sprawdzone w Search Console)
- [ ] Analytics Plausible działa (opcjonalnie)
- [ ] Polskie znaki diakrytyczne wszędzie poprawne

## Edge cases i decyzje

- **Public vs Private posts**: tylko `PUBLISHED` z M-EDIT; w fazie 5+ dodajemy `web_visibility = public|members_only`
- **Image hosting**: dla landing/SEO używamy obrazów z M-IMG (CDN przez Caddy)
- **Slug collision**: slug generowany z hooka + post_id suffix
- **Polskie znaki w URL**: nie używamy diakrytyków w slugach (URL slugify do ASCII)
- **CDN**: Caddy z aggressive caching dla static; ISR cache invalidation gdy nowy post `PUBLISHED` (webhook)
- **Mailing transactional**: SMTP z M-INFRA; backupowy provider (Resend free tier) gdy SMTP padnie
- **Bot scraping**: rate limit /api/web/* + robots.txt allows; Cloudflare jeśli ataki
- **Cookies**: tylko session cookie po loginie (essential); brak third-party cookies
- **Polska jurysdykcja**: serwer w Polsce, dane w Polsce, RODO compliance po polsku

## Zależności

**Wchodzi**: M-INFRA, M-AUTH (sessions), M-EDIT (posts public), M-COMM (Q&A), M-EVT (wydarzenia), M-RAG (publiczne źródła)
**Wychodzi**: nikt nie zależy od M-WEB (frontend layer)

## Risks

| Risk | Mit. |
|---|---|
| SEO ranking słaby | Structured data + sitemap + ISR + content quality (rozwinięcia 20-30 zdań) |
| Wycieki PII przez SSR | Strict separation: SSR tylko publiczne dane, prywatne przez client-side po auth |
| Slow performance na mobile | SSG/ISR + image optimization + Lighthouse CI w GitHub Actions |
| RODO niezgodność (Polska) | Polityka po polsku + retention + samoobsługa + minimalizacja |
| DDOS na formularz Q&A | Rate limit + Cloudflare; hCaptcha; honeypot field |
| Lokalna dostępność (RPO/RTO) | M-INFRA backupy + Caddy local cache; planned outages w nocy |
