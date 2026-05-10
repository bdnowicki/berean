# Roadmap — Fala 0 (Fundamenty)

> **Cel fali**: gotowa, monitorowana, zbackupowana platforma + uwierzytelnianie. Nic merytorycznego (treść, AI, publikacja) — tylko fundament dla Fal 1+.
>
> **Czas trwania**: 4-6 tygodni · **Zespół**: 1-2 osoby (1 DevOps lead — może być pasywnie pomagający — + 1 backend dev)

## Definicja "fala 0 gotowa"

System spełnia wszystkie warunki:

- [ ] Domena podpięta, HTTPS działa pod 3 subdomenami: `app.<domena>`, `api.<domena>`, `grafana.<domena>`
- [ ] Stack Docker Compose działa: Postgres, Qdrant, Redis, Caddy, backend (placeholder), frontend (placeholder), uptime-kuma, prometheus, grafana, loki
- [ ] Wszystkie kontenery `healthy` po `docker compose up -d`
- [ ] Backupy daily uruchamiają się, restore przetestowany na świeżym hoście
- [ ] Monitoring + alerty: zatrzymanie kontenera triggeruje alert w 5 min
- [ ] CI/CD: PR → testy → merge → auto-deploy
- [ ] Pastor może zarejestrować konto → włączyć TOTP → zalogować się tylko z TOTP
- [ ] Redaktor może zalogować się email+hasło
- [ ] Discord OAuth działa, tworzy usera z rolą `CZLONEK`
- [ ] User może wyeksportować dane i zainicjować delete (14d soft-delete)
- [ ] RBAC: `require_role(PASTOR)` blokuje redaktora z 403
- [ ] Audit log zapisuje: login, zmianę roli, MFA toggle, self-delete

## Tygodnie

### Tydzień 1 — Setup serwera i Docker stack

**Cel**: serwer dostępny, Docker działa, Caddy serwuje "Hello World".

**Taski (M-INFRA)**:
- [ ] Day 1: Bootstrap serwera (`bootstrap.sh`): user `bnd`, ufw, fail2ban, automatic-updates, klucze SSH
- [ ] Day 2: Instalacja Docker + Docker Compose, weryfikacja
- [ ] Day 2-3: Repo setup (`bnd-platform`): struktura katalogów, `compose.yml` szkielet, `.env.example`, README
- [ ] Day 3-4: Caddy config — HTTPS auto-cert dla 3 subdomen (app, api, grafana)
- [ ] Day 4-5: Postgres + Qdrant + Redis kontenery, healthchecks
- [ ] Day 5: Pierwsze deployment — `docker compose up -d`, weryfikacja `https://app.<domena>` zwraca placeholder

**Risk: domena DNS propagation** — kup domenę i ustaw DNS dzień przed startem; daje 24h margines.

**DoD**: można `curl https://api.<domena>/healthz` i dostać 200 (placeholder).

### Tydzień 2 — Monitoring + Backupy

**Cel**: widoczność systemu + bezpieczeństwo danych.

**Taski (M-INFRA)**:
- [ ] Day 1-2: Uptime Kuma — checks na 3 subdomeny + Postgres + Qdrant
- [ ] Day 2-3: Prometheus + Grafana — basic dashboards (System Overview)
- [ ] Day 3: Loki — agregacja logów wszystkich kontenerów
- [ ] Day 4: Backupy: skrypt `backup.sh` z `pgbackrest` + GPG encryption + upload do B2; cron daily
- [ ] Day 5: Restore test — przygotuj świeży VPS, ściągnij backup, odtwórz Postgres+Qdrant. Udokumentuj.
- [ ] Day 5: Slack/email alerts skonfigurowane, test alert (zatrzymaj kontener → alert w 5 min)

**DoD**: alert testowy w Slacku po `docker stop bnd-postgres`; restore test sukces (świeży host, ostatni backup, dane = expected).

### Tydzień 3 — CI/CD + Auth foundation

**Cel**: deploy automatyczny + struktura backendu z auth.

**Taski (M-INFRA)**:
- [ ] Day 1: GitHub Actions workflow `deploy.yml` (test → SSH deploy)
- [ ] Day 2: Branch protection (main wymaga 1 review + tests passing)

**Taski (M-AUTH)**:
- [ ] Day 2: FastAPI skeleton (`bnd-api`): main.py, routers/, services/, models/, alembic
- [ ] Day 3: Postgres models: User, Role, UserRole, Session, RefreshToken, AuditLog (SQLAlchemy + Alembic migration)
- [ ] Day 4: Auth core: Argon2id, JWT short-lived + refresh, login endpoint, register endpoint
- [ ] Day 5: RBAC dependency `require_role(*roles)` + przykładowy endpoint `/api/test/pastor-only`

**DoD**: PR z dummy zmianą → automatyczny deploy → endpoint `/api/auth/register` + `/api/auth/login` zwraca tokeny → endpoint `pastor-only` z user-bez-roli zwraca 403.

### Tydzień 4 — MFA + Discord OAuth + Audit

**Cel**: pełna ścieżka multi-tier auth.

**Taski (M-AUTH)**:
- [ ] Day 1-2: TOTP setup (`pyotp`): `/api/auth/totp/setup` zwraca QR, `/confirm` aktywuje. Backup codes.
- [ ] Day 2-3: TOTP w login flow: jeśli user ma TOTP, drugi step po haśle
- [ ] Day 3-4: Discord OAuth (`authlib`): `/api/auth/login/discord` redirect, `/api/auth/discord/callback` create-or-update user, assign `CZLONEK`
- [ ] Day 4-5: Audit log integration: każdy login/logout/MFA-toggle/role-change w `audit_log`
- [ ] Day 5: Brute-force protection (5 fails → 15 min lockout per email + per IP)
- [ ] Day 5: Hash check przeciwko HIBP API (k-anonymity, free)

**DoD**: pastor może zarejestrować się → włączyć TOTP → wylogować → zalogować się tylko z TOTP. Discord login tworzy nowego usera. Brute-force lockout sprawdzony.

### Tydzień 5 — Self-service RODO + Frontend MVP

**Cel**: użytkownik może obsłużyć siebie + jest widoczny frontend.

**Taski (M-AUTH)**:
- [ ] Day 1: Self-service export — `/api/me/export` generuje JSON i wysyła mailem
- [ ] Day 2: Self-service delete — `/api/me/delete` ustawia `retention_until = now + 14d`, status `pending_delete`
- [ ] Day 2: Cron job: hard-delete users with `retention_until < now`
- [ ] Day 3: Self-service cancel-delete + "wyloguj wszędzie" + listę aktywnych sesji

**Taski (M-WEB minimum)**:
- [ ] Day 3-4: Next.js app skeleton w `bnd-web`: layout, login page, register page
- [ ] Day 4-5: `/login` (email+pass z TOTP step + Discord button), `/register`, `/moje-dane` (basic)
- [ ] Day 5: Deploy → `app.<domena>` pokazuje login form

**DoD**: użytkownik wchodzi na `app.<domena>`, klika "Zaloguj przez Discord", wraca jako CZLONEK. Pastor (utworzony przez admina przez `INSERT` SQL) loguje się email+hasło+TOTP. `/moje-dane` pokazuje dane.

### Tydzień 6 — Polish + dokumentacja + handoff do Fali 1

**Cel**: stabilność + dokumentacja → rampa do Fali 1.

**Taski**:
- [ ] Day 1-2: Buffer na bugi z poprzednich tygodni
- [ ] Day 2-3: `RUNBOOK.md` w M-INFRA: restart, restore, scale-up, dostęp SSH, rotacja kluczy, troubleshooting top 5 problemów
- [ ] Day 3-4: `CONTRIBUTING.md`: jak setup'ować dev locally (Docker Compose dev), jak otworzyć PR, code style, testy
- [ ] Day 4: Zaproś pastora i redaktora — zrób manualny test E2E (nowy user → login → MFA setup → /moje-dane export). Bug fix.
- [ ] Day 5: Demo dla zespołu Fali 1 (M-AI, M-RAG, M-GUARD, M-EDIT, M-PUB) — co jest gotowe, jakie API mają używać, gdzie są secrets, gdzie deploy
- [ ] Day 5: Retro Fali 0: co poszło dobrze, co źle, co przekazujemy

**DoD**: cały team Fali 1 może lokalnie odpalić stack, ma dostęp do staging, wie jak deploy. Pastor potwierdza że może się zalogować i wyeksportować dane.

## Kluczowe decyzje fali 0

### Wybór VPS staging vs produkcja

- **Decyzja**: na potrzeby Fali 0 wystarczy **jedno środowisko** (= produkcja). Self-hosted serwer parafialny wystarczy.
- **Powód**: oszczędność, kontrola, brak ruchu w fazie 0
- **Implikacja**: ostrożny CI/CD (testy obowiązkowe, manual smoke test po deploy)
- **Faza 1+**: dodać staging (drugi VPS lub VM na tym samym hoście)

### Sekrety lokalnie vs vault

- **Decyzja**: `.env` plik na serwerze (chmod 600) + GitHub Secrets dla CI. **Bez Vault'a/Doppler**.
- **Powód**: prosta operacja, mała skala, niska liczba sekretów
- **Faza 4+**: rozważ Vault gdy >5 środowisk lub >20 sekretów

### Postgres major version

- **Decyzja**: Postgres 16 (LTS, support do 2028)
- **Implikacja**: pgbackrest 2.x (compatible)

### Co NIE jest częścią Fali 0

- ❌ M-AI gateway (Fala 1)
- ❌ M-RAG (Fala 1)
- ❌ M-EDIT, M-PUB (Fala 1)
- ❌ Image generation (Fala 2)
- ❌ Discord bot (Fala 2 — ale Discord OAuth login JEST w Fali 0)
- ❌ Wydarzenia (Fala 3)
- ❌ Analytics dashboardy (Fala 3) — ale ClickHouse w `compose.yml` możemy umieścić od razu (placeholder, jeszcze nie używany)
- ❌ Public site bogaty (Fala 4) — w Fali 0 tylko login/register/moje-dane

## Risks Fali 0

| Risk | Mit. |
|---|---|
| Serwer parafialny nie spełnia 16 GB RAM | Pomiar przed startem; jeśli za słaby — rozważyć VPS Hetzner CX32 (8€/mies) jako tymczasowy |
| Domena wykupywana ad-hoc → DNS opóźnienia | Kup domenę dzień 0, ustaw DNS, czekaj 24h przed deployem |
| Pastor zgubi backup codes po setupie TOTP | Wymóg przy onboardingu pastora: spisanie backup codes w sejfie + drugi pastor jako manualny reset |
| Discord API zmiana (deprecation OAuth scope) | discord.py i `authlib` aktywne projekty; monitoring changelog |
| Brakujące kompetencje DevOps | Buffer w Tygodniu 6 + Reach out do społeczności Polskich DevOps na Discordzie/Slacku Sysops |
| Kolega odpada (1-osobowy zespół) | Dokumentacja na bieżąco (Tydzień 6 punkt 2-3); pastor wie gdzie są secrets |

## Kryteria przejścia do Fali 1

Fala 1 może wystartować jednocześnie z końcem Fali 0 (overlap Tydzień 6) **wtedy i tylko wtedy gdy**:

- [ ] Postgres + Qdrant + Redis dostępne i z kontami serwisowymi dla M-AI, M-RAG, M-GUARD
- [ ] Backend API `bnd-api` ma działającą strukturę gdzie można dodać nowe routery
- [ ] M-AUTH ma `require_role()` dependency, którą inne moduły zaimportują
- [ ] CI/CD deployuje zmiany w nowych modułach z dummy testem
- [ ] Pastor i redaktor mają konta i loginy działają

## Zasoby zewnętrzne (do przygotowania w Fali 0)

- [ ] Domena (~50 PLN/rok)
- [ ] Backblaze B2 account (≤ 1 GB free, ~$5/TB beyond)
- [ ] Slack workspace dla zespołu (free tier)
- [ ] GitHub repo (organization or personal)
- [ ] Discord developer app (dla OAuth) — free
- [ ] HIBP API (free, no auth needed)
- [ ] Email transactional: SMTP z domeny lub Resend free tier (3k/mies.)

## Co przejmuje Fala 1

Po zakończeniu Fali 0, następna fala startuje z 2 zespołami równolegle (zgodnie z preferencją "1-2 zespoły" + `pełna wizja etapowo`):

**Zespół A — AI Engine**:
- M-AI (4-5 tyg.)
- M-RAG (4-5 tyg., overlap z M-AI od T2)
- M-GUARD (3-4 tyg., wymaga M-AI gotowego do T3)

**Zespół B — Editorial + Publishing**:
- M-EDIT (5-6 tyg., wymaga M-AI + M-GUARD od T3)
- M-PUB (4-5 tyg., wymaga M-EDIT od T3)

Total Fala 1: ~10-12 tygodni do MVP "AI generuje → pastor zatwierdza → bot publikuje".

Patrz `WYMAGANIA-APLIKACJI.md` § 5 Roadmapa fal i specyfikacje per moduł w `modules/M-*.md`.
