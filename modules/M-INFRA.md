# M-INFRA — DevOps & Infrastructure

> **Faza**: 0 (Fundamenty) · **Zespół**: 1 osoba (DevOps lead) · **Estymacja**: 3-4 tyg.

## Cel

Działająca, monitorowana, zbackupowana infrastruktura na serwerze parafialnym, gotowa pod wszystkie pozostałe moduły. Wszystkie usługi w Docker Compose pod jednym hostem, z auto-TLS, kopiami zapasowymi i alertami.

## Kontekst i ograniczenia

- Serwer parafialny: 16-32 GB RAM, brak GPU
- Self-hosted (RODO + zaufanie pastorów)
- Budżet zewnętrzny niski → wszystko open-source
- Domena: do ustalenia (sugestia: `app.biblianaukadowody.pl` lub `bnd.kosciol.pl`)

## Stack

- **Konteneryzacja**: Docker + Docker Compose (jeden host, multi-service)
- **Reverse proxy + TLS**: Caddy (auto-cert Let's Encrypt)
- **Bazy**:
  - Postgres 16 (transakcje aplikacyjne)
  - Qdrant (vector store dla M-RAG i M-GUARD)
  - Redis 7 (kolejka zadań Celery/RQ)
  - ClickHouse (DWH dla M-ANALYTICS — od fali 3)
- **Monitoring**: Uptime Kuma (status), Prometheus + Grafana (metryki), Loki (logi)
- **Backupy**: `pgbackrest` lub `restic` → szyfrowane → S3-compatible (Backblaze B2 lub OVH Object Storage, ~10 PLN/mies)
- **CI/CD**: GitHub Actions → SSH deploy
- **Sekrety**: Docker secrets + `.env` plik out-of-tree

## Funkcjonalności

### F1. Bootstrap serwera
- Skrypt `bootstrap.sh` instalujący: Docker, Docker Compose, ufw, fail2ban, automatic updates
- User dedykowany `bnd` z sudo, klucze SSH zamiast hasła
- ufw allow: 22 (rate-limited), 80, 443; reszta deny

### F2. Docker Compose stack
- `compose.yml` z usługami: caddy, postgres, qdrant, redis, backend (placeholder), frontend (placeholder), uptime-kuma, prometheus, grafana, loki
- Healthchecks dla wszystkich
- Wolumeny named (nie bind mounts) dla portability backupów
- Networki: `internal` (DB) i `public` (proxy)

### F3. TLS i reverse proxy
- `Caddyfile` z auto-cert dla:
  - `app.<domena>` → frontend
  - `api.<domena>` → backend
  - `metabase.<domena>` → metabase (od fali 3)
  - `grafana.<domena>` → grafana (basic auth)
- Brotli/gzip compression, HSTS, security headers

### F4. Backupy
- Daily: Postgres dump → szyfrowanie GPG → upload do B2
- Daily: Qdrant snapshot → upload do B2
- Daily: ClickHouse backup → upload do B2 (od fali 3)
- Daily: uploads/ folder (źródła) → upload do B2
- Retention: 30 dni daily, 12 miesięcy monthly
- Restore runbook + test restore raz na kwartał

### F5. Monitoring i alerty
- Uptime Kuma sprawdza co 60s: api healthcheck, frontend, postgres, qdrant
- Prometheus: container CPU/RAM, dysk, http_requests
- Grafana dashboardy: System Overview, Application, AI Costs (po podłączeniu M-AI)
- Loki: agregacja logów wszystkich kontenerów
- Alerty (Slack lub email):
  - Container down > 5 min
  - Disk > 85%
  - Postgres slow query > 5s
  - Backup nie powiodł się
- Heartbeat external: cron-monitor.io free tier (100/mies) sprawdza, czy "pulse" przychodzi co godz.

### F6. CI/CD
- `.github/workflows/deploy.yml`:
  - Run testów na PR
  - Po merge do `main` → SSH deploy
  - Blue-green nie potrzebny (zero-downtime niekrytyczny dla MVP)
- Branch protection: main wymaga 1 review + zielonych testów

### F7. Secrets management
- `.env` plik na serwerze (chmod 600)
- `.env.example` w repo z wszystkimi kluczami (puste)
- Secrets dla CI: GitHub Actions secrets
- Rotacja kluczy: dokumentacja procedury

## API / Kontrakty

Brak — moduł infrastrukturalny, ekspoonuje "platformę" dla pozostałych.

**Wymagania platformowe dla innych modułów**:
- Każdy moduł dostarcza `Dockerfile` + sekcję w `compose.yml`
- Każdy moduł musi mieć `/healthz` endpoint zwracający 200 + `{"status": "ok"}`
- Każdy moduł loguje do `stdout` (Loki zbiera)
- Każdy moduł czyta config z env vars (12-factor)

## Zmienne środowiskowe (template)

```env
# Domeny
APP_DOMAIN=app.biblianaukadowody.pl
API_DOMAIN=api.biblianaukadowody.pl

# Postgres
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_DB=bnd
POSTGRES_USER=bnd
POSTGRES_PASSWORD=<generate>

# Qdrant
QDRANT_HOST=qdrant
QDRANT_PORT=6333
QDRANT_API_KEY=<generate>

# Redis
REDIS_URL=redis://redis:6379/0

# Backupy
B2_ACCOUNT_ID=<set>
B2_APPLICATION_KEY=<set>
B2_BUCKET=bnd-backups
BACKUP_GPG_RECIPIENT=admin@biblianaukadowody.pl

# Alerty
SLACK_WEBHOOK_URL=<set>
ALERT_EMAIL=admin@biblianaukadowody.pl

# Heartbeat
HEARTBEAT_URL=https://hc-ping.com/<uuid>
```

## Definition of Done

- [ ] Serwer dostępny pod HTTPS na 3 subdomenach (app, api, grafana)
- [ ] `docker compose up -d` startuje cały stack
- [ ] Wszystkie kontenery `healthy` po 60s
- [ ] Postgres + Qdrant + Redis dostępne dla pozostałych modułów
- [ ] Daily backup wykonał się co najmniej raz, restore przetestowany
- [ ] Uptime Kuma + Grafana dostępne, pierwszy alert testowy działa (np. zatrzymaj kontener → alert w 5 min)
- [ ] CI/CD: PR do dummy modułu deployuje się automatycznie
- [ ] Runbook (`RUNBOOK.md`) zawiera: restart procedure, restore procedure, scale-up procedure, dostęp SSH, rotacja kluczy
- [ ] Dokumentacja "Onboarding nowego deva" — jak dostać dostęp, co zainstalować lokalnie

## Edge cases i decyzje

- **Brak GPU**: nie hostujemy LLM lokalnie. Ollama może być dostępne dla embeddings (CPU) i prostych zadań background, ale ostre LLM = API zewn.
- **Disk space**: Qdrant + Postgres + ClickHouse + uploads + backupy local + logi. Min 200 GB SSD. Monitor.
- **Power outage**: serwer parafialny → UPS na 30+ min, auto-shutdown przy <20% baterii.
- **Pojedynczy host = SPOF**: zaakceptowane w MVP. Disaster recovery = restore z B2 na nowy host (≤ 4h RTO).
- **Aktualizacje OS**: unattended-upgrades dla security patches; major upgrade kwartalnie z manualnym oknem.

## Zależności

**Wchodzi**: nic.
**Wychodzi**: wszystkie inne moduły zależą od dostępności tej platformy.

## Risks

| Risk | Mit. |
|---|---|
| Awaria sprzętu | Backup + dokumentacja restore na nowym hoście |
| Zhakowanie | fail2ban, ufw, auto-updates, brak panelu admina dostępnego publicznie, MFA dla SSH |
| Wyciek backupu | GPG encryption przed uploadem, klucz prywatny tylko u admina |
| Licznik B2 wyższy niż założony | Monitor, bucket lifecycle policy (auto-delete > 12 mies.) |
