# M-AUTH — Auth & Identity

> **Faza**: 0 (Fundamenty) · **Zespół**: 1 osoba · **Estymacja**: 2-3 tyg.

## Cel

Bezpieczny, wielowarstwowy system uwierzytelniania i autoryzacji dla wszystkich grup użytkowników: pastorów (z MFA), redaktorów, moderatorów i członków społeczności. Dodatkowo: samoobsługa RODO (export, delete) i RBAC dla pozostałych modułów.

## Stack

- **Backend**: FastAPI + `fastapi-users` lub własna implementacja (Argon2id dla haseł)
- **MFA**: TOTP (`pyotp`) + opcjonalnie WebAuthn/Yubikey (`fido2`)
- **Discord OAuth**: `authlib`
- **Tokens**: JWT short-lived (15 min access) + refresh token (30 dni, rotujący)
- **Storage**: Postgres (tabele users, sessions, refresh_tokens, audit_log)

## Funkcjonalności

### F1. Modele i role

```
User
  id (UUID), email, password_hash (Argon2id, NULL dla Discord-only),
  display_name, avatar_url, created_at, last_login_at,
  totp_secret (encrypted), totp_enabled, webauthn_credentials,
  discord_id (NULL jeśli email-based),
  status (active|suspended|deleted),
  retention_until (NULL lub data — dla soft-delete RODO)

Role (enum)
  PASTOR, REDAKTOR, MODERATOR, CZLONEK, ADMIN_TECH

UserRole
  user_id, role, granted_by, granted_at, revoked_at (NULL=aktywne)
  -- Pastor=teolog=admin: jeden user może mieć [PASTOR, ADMIN_TECH] jednocześnie
```

### F2. Flow logowania per rola

| Rola | Flow |
|---|---|
| Pastor | email + hasło + TOTP (obowiązkowe) → JWT |
| Redaktor / Moderator | email + hasło → JWT (TOTP opcjonalne) |
| Członek | Discord OAuth → JWT |
| Admin tech | jak Pastor + audit log każdego logowania |

### F3. Discord OAuth dla członków
- Redirect na `discord.com/oauth2/authorize?...&scope=identify`
- Callback `/auth/discord/callback`
- Stworzenie usera z `discord_id`, `display_name`, `avatar_url`
- Auto-assign `CZLONEK` role
- Rola `WOLONTARIUSZ_MODERATOR` przypisywana ręcznie przez admina

### F4. RBAC middleware dla pozostałych modułów

```python
# FastAPI dependency
def require_role(*allowed: Role):
    async def dep(user: User = Depends(get_current_user)):
        if not any(r in user.active_roles for r in allowed):
            raise HTTPException(403)
        return user
    return dep

# Użycie w innych modułach
@router.post("/posts/approve/{id}")
async def approve_post(id: UUID, user = Depends(require_role(Role.PASTOR))):
    ...
```

### F5. RODO self-service (`/moje-dane`)

- **Export**: button "Pobierz moje dane" → generuje JSON ze wszystkim, co system o użytkowniku wie (z M-EVT, M-COMM, Q&A, audit log → przez wewnętrzne API innych modułów). Wysyłane mailem jako załącznik (link 24h).
- **Delete**: button "Usuń moje konto" z modalem potwierdzenia + 14-dniowym soft-delete window
  - User status → `deleted`, `retention_until = now + 14d`
  - Po 14 dniach: hard delete (cron job)
  - W ciągu 14 dni user może zalogować się i kliknąć "anuluj usunięcie"
  - Kontent wygenerowany przez usera (komentarze, pytania Q&A) zostaje, ale autorstwo zmieniane na "Użytkownik usunięty"

### F6. Audit log

Każda akcja krytyczna zapisywana w `audit_log`:
- Login (sukces/porażka, IP, user-agent)
- Zmiana hasła
- Włączenie/wyłączenie MFA
- Przyznanie/odebranie roli
- Self-service export/delete
- Logout (wszystkie sesje)

Retention: 12 miesięcy. Dostępne dla `ADMIN_TECH`.

### F7. Session management
- Refresh token rotation (każde użycie generuje nowy)
- "Zaloguj się ze wszystkich urządzeń" / "Wyloguj wszędzie"
- Lista aktywnych sesji w `/moje-konto/sesje`
- Detekcja anomalii (login z nowego kraju → email do usera)

## API / Kontrakty

```
POST   /api/auth/register          { email, password, display_name } → 201
POST   /api/auth/login             { email, password, totp? }       → { access, refresh }
POST   /api/auth/login/discord                                       → 302 (redirect)
GET    /api/auth/discord/callback  ?code=...                         → { access, refresh }
POST   /api/auth/refresh           { refresh }                       → { access, refresh }
POST   /api/auth/logout            { all_devices?: bool }            → 204
POST   /api/auth/totp/setup                                          → { qr_code, secret }
POST   /api/auth/totp/confirm      { code }                          → { backup_codes[] }
POST   /api/auth/totp/disable      { password, code }                → 204
GET    /api/me                                                       → User
POST   /api/me/export                                                → 202 (mail wysłany)
POST   /api/me/delete              { password }                      → 202 (14d window)
POST   /api/me/cancel-delete                                         → 200
GET    /api/me/sessions                                              → Session[]
DELETE /api/me/sessions/{id}                                         → 204
GET    /api/admin/users            (ADMIN_TECH)                      → User[]
POST   /api/admin/users/{id}/roles (ADMIN_TECH)                      → 200
```

## Definition of Done

- [ ] Pastor może zarejestrować konto → włączyć TOTP → zalogować się tylko z TOTP
- [ ] Redaktor może zalogować się email+hasło, opcjonalnie TOTP
- [ ] Discord OAuth login działa, tworzy usera z rolą CZLONEK
- [ ] `require_role(Role.PASTOR)` blokuje redaktora z 403
- [ ] User może wyeksportować swoje dane (JSON mailem)
- [ ] Soft-delete działa: konto znika z UI, po 14d hard-delete; w trakcie window można anulować
- [ ] Audit log pokazuje wszystkie krytyczne akcje
- [ ] Brute-force protection: 5 nieudanych prób → 15-min lockout per email + per IP
- [ ] Hasła: min. 12 znaków, sprawdzane przeciwko HIBP API (k-anonymity, free)
- [ ] Refresh token rotation działa, kradzież starego refresh = wylogowanie

## Edge cases i decyzje

- **Pastor zgubił TOTP**: backup codes (10 jednorazowych przy setupie); jeśli i te zgubione → manual reset przez admina tech (z log audytowym)
- **Discord OAuth + email**: użytkownik może później dodać hasło i email do konta Discord (merge tożsamości)
- **Email change**: wymaga potwierdzenia obu adresów (link na stary + na nowy)
- **Konto suspended**: można zalogować się, ale wszystkie endpointy poza `/me` zwracają 403
- **Migracja ról**: gdy pastor odchodzi/dochodzi nowy, admin tech ręcznie revoke/grant; M-AUTH NIE robi auto-promotions
- **Discord-only user dodaje hasło**: `discord_id` zostaje, dopisujemy `password_hash`, login możliwy obiema drogami

## Zależności

**Wchodzi**: M-INFRA (Postgres dostępny)
**Wychodzi**: każdy moduł UI/API używa `require_role(...)` dependency

## Risks

| Risk | Mit. |
|---|---|
| Wyciek bazy haseł | Argon2id (memory-hard), salt, no plaintext anywhere |
| Phishing pastora | TOTP obowiązkowy, alert po loginie z nowego IP/kraju |
| Discord ban / OAuth provider down | Fallback: email link reset hasła pozwala dostać się klasyczną drogą (jeśli user dodał hasło) |
| Forgotten TOTP po zmianie urządzenia | Backup codes + manual admin reset z audytem |
| RODO niezgodność | Soft-delete window (14d) + hard-delete cron + audit log każdego delete |
