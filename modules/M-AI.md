# M-AI — AI Gateway & Generation

> **Faza**: 1 (Silnik treści) · **Zespół**: 1-2 osoby (AI engineer lead + dev) · **Estymacja**: 4-5 tyg.

## Cel

Jednolity, configurable interfejs do generowania treści przy użyciu wielu providerów LLM. Pozwala adminowi szybko przełączać modele per zastosowanie ('draft', 'final', 'judge', 'image-brief') by eksperymentować na osi czas/jakość/koszt — bez zmiany kodu.

## Inwariant

> **Żadne dane osobowe nie wchodzą do promptów.** Pre-procesor `redact_pii(text)` jest pierwszym krokiem każdego wywołania.

## Stack

- **FastAPI** (Python 3.12+)
- **LiteLLM** — biblioteka unifikująca SDK providerów (Anthropic, OpenAI, Gemini, DeepSeek, Ollama itp.)
- **Pydantic** — schematy request/response
- **Postgres** — historia wywołań, koszty, konfiguracja routing
- **Redis** — cache odpowiedzi (deterministyczne prompty), rate limiting per provider
- **Tenacity** — retry z exponential backoff

## Funkcjonalności

### F1. Provider abstraction

```python
class LLMRequest(BaseModel):
    role: Literal["draft", "final", "judge", "image-brief", "embed", "summary"]
    messages: list[Message]
    max_tokens: int = 2000
    temperature: float = 0.7
    metadata: dict  # post_id, user_id, attempt_no

class LLMResponse(BaseModel):
    content: str
    provider: str
    model: str
    tokens_in: int
    tokens_out: int
    cost_usd: Decimal
    latency_ms: int
    request_id: UUID
```

```python
async def generate(req: LLMRequest) -> LLMResponse:
    # 1. PII redaction (HARD INVARIANT)
    req.messages = redact_pii(req.messages)

    # 2. Cache lookup (jeśli temperature=0 i deterministic)
    if req.temperature == 0:
        if cached := await cache.get(hash(req)): return cached

    # 3. Resolve routing: rola → provider+model
    routing = await routing_config.get(req.role)  # np. {"provider": "anthropic", "model": "claude-haiku-4-5"}

    # 4. Call via LiteLLM z retry + circuit breaker
    resp = await litellm.acompletion(model=f"{routing.provider}/{routing.model}", ...)

    # 5. Track koszt + log + emit metrics
    await record_usage(req, resp)
    return resp
```

### F2. Routing config (admin-editable)

Tabela `ai_routing`:

| role | primary_provider | primary_model | fallback_provider | fallback_model | max_cost_usd | active |
|---|---|---|---|---|---|---|
| draft | anthropic | claude-haiku-4-5 | deepseek | deepseek-chat | 0.05 | true |
| final | anthropic | claude-sonnet-4-6 | anthropic | claude-haiku-4-5 | 0.20 | true |
| judge | google | gemini-2.5-flash | anthropic | claude-haiku-4-5 | 0.01 | true |
| image-brief | anthropic | claude-haiku-4-5 | openai | gpt-5-mini | 0.02 | true |
| embed | local | multilingual-e5-base | - | - | 0 | true |

Admin UI (M-OPS) pozwala edytować bez deploymentu.

### F3. 3-warianty na żądanie

```python
async def generate_variants(req: LLMRequest, n: int = 3) -> list[LLMResponse]:
    # Równolegle, z różnymi seed'ami / temperaturami
    tasks = [generate(req.with_temperature(0.6 + 0.15 * i)) for i in range(n)]
    return await asyncio.gather(*tasks)
```

### F4. Cost tracking

- Tabela `ai_usage`: każde wywołanie z timestampem, providerem, modelem, kosztem, role, post_id
- Materialized view `ai_costs_daily`: agregat per dzień/provider/role
- Endpoint `/api/ai/usage?from=...&to=...&group_by=role|provider`
- Alert (przez M-INFRA): koszt dzienny > X PLN → Slack

### F5. Cost-aware fallback

- Jeśli dzienny budżet roli przekroczony → automatycznie przerzuć na fallback (tańszy)
- Konfigurowalne progi w admin UI
- Hard limit: kill-switch dla całego AI gateway (admin button)

### F6. PII redaction

Implementacja:
- Email regex
- Numery telefonu PL (`\+?48[\s-]?\d{3}[\s-]?\d{3}[\s-]?\d{3}`)
- PESEL (11 cyfr w specyficznym formacie)
- IBAN
- IP addresses
- Imiona-nazwiska (heurystyka: 2 słowa zaczynające się wielką, na liście top-1000 PL imion)
- Whitelist: imiona biblijne (Daniel, Ezechiel, Ezechiasz, Jezus, Maria...) — pobierane z Biblia RAG

Test: golden file z przykładami "powinno zostawić" vs "powinno usunąć".

### F7. Prompt versioning

- System prompty (np. dla 'draft' z 28 FB ADW) versioned w Postgres
- Zmiana promptu = nowa wersja, nigdy nie nadpisuje
- W `ai_usage` zapisujemy `prompt_version` → łatwa korelacja "zmieniliśmy prompt → jakość spadła"

### F8. Local Ollama integration

- Ollama hostowany lokalnie (CPU only, bez GPU = tylko małe modele)
- Sensowne use-cases:
  - Embeddings (`multilingual-e5-base` quantized — działa OK na CPU)
  - PII detection (mały model do walidacji regex)
  - Classifier (kategoria posta) — szybki, tani
- LiteLLM ma natywne wsparcie Ollama (`ollama/llama3.2`)

## API / Kontrakty

```
POST /api/ai/generate              {role, messages, ...}     → LLMResponse
POST /api/ai/generate-variants     {role, messages, n}       → LLMResponse[]
POST /api/ai/embed                 {texts: string[]}         → number[][]
GET  /api/ai/usage                 ?from&to&group_by         → usage agg
GET  /api/ai/routing               (ADMIN_TECH)              → routing[]
PUT  /api/ai/routing/{role}        (ADMIN_TECH)              → 200
POST /api/ai/kill-switch           (ADMIN_TECH) {enabled}    → 200

# Eventy (Redis pub/sub)
- ai.usage.recorded { role, provider, model, cost_usd, post_id }
- ai.kill_switch.toggled { enabled }
- ai.fallback.triggered { role, from, to, reason }
```

## Definition of Done

- [ ] Można wywołać `generate()` z rolą i dostać sensowną polską odpowiedź od domyślnego providera
- [ ] PII redaction usuwa 100% przypadków z golden file
- [ ] Admin może w UI zmienić routing (draft: haiku → deepseek) bez restartu
- [ ] 3-warianty zwracają trzy różne wersje (różnica > 30% bigramów)
- [ ] Cost tracking: po 100 wywołaniach widać sumaryczny koszt w dashboardzie
- [ ] Kill-switch działa: po włączeniu wszystkie `generate()` zwracają 503
- [ ] Fallback: gdy primary provider down (mock) → automatyczne przejście na fallback
- [ ] Cache hit: 2× to samo żądanie z `temperature=0` → drugie wraca z cache (latency < 50ms)
- [ ] Ollama embed działa lokalnie (CPU)

## Edge cases i decyzje

- **Provider rate limit**: LiteLLM ma wbudowany retry + obracanie API keys; jeśli przekroczony — fallback
- **Provider down**: circuit breaker (5 fail-ów w 60s → otwarty na 5 min)
- **Bardzo długie prompty**: limit 100k tokenów wejścia; powyżej → automatyczne dzielenie i podsumowanie (mapa-redukuj wzór)
- **Streaming**: w MVP nie wspieramy (UI może opcjonalnie używać `/api/ai/generate-stream` w fazie 2)
- **Deterministyczne wyniki**: dla judge zawsze `temperature=0` + cache aktywny
- **Provider z polskim**: priorytet Anthropic (Haiku/Sonnet) > OpenAI > Gemini > DeepSeek (DeepSeek czasem słabnie po polsku)
- **Wybór modelu zwiększający koszt**: blokowany jeśli przekraczałby `max_cost_usd` per request

## Zależności

**Wchodzi**: M-INFRA (Postgres, Redis, Ollama)
**Wychodzi**:
- M-EDIT (generuje warianty, regeneruje po feedback)
- M-RAG (embeddings dla vector store)
- M-GUARD (judge model)
- M-IMG (generuje brief graficzny)
- M-COMM (sentiment analysis komentarzy)

## Risks

| Risk | Mit. |
|---|---|
| PII wycieka do prompta mimo redactora | Test golden file + LLM judge sprawdzający output (sentinel test) + audit log każdego prompta (z hash, nie plaintext) |
| Koszt eksploduje | Budget per role + kill-switch + alert > 80% budżetu |
| Provider deprecation modelu | LiteLLM informuje o deprekacjach; routing config łatwo zmienić |
| Halucynacja w generowanym poście | Nie odpowiedzialność M-AI — odp. M-GUARD (judge) i M-EDIT (pastor); M-AI gwarantuje tylko wywołanie |
| Latency > 30s blokujący UI | Async + UI pokazuje progress; długie wywołania w tle (Celery) |
