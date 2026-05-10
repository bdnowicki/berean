# M-RAG — Source Library & RAG

> **Faza**: 1 (Silnik treści) · **Zespół**: 1-2 osoby · **Estymacja**: 4-5 tyg.

## Cel

Repozytorium kuratorowanych źródeł wiedzy + retrieval API. AI generujące wpisy zawsze pobiera kontekst z tego repozytorium — nigdy nie pisze "z głowy". Pastor uploaduje nowe źródła, AI klasyfikuje, admin akceptuje, dokument wchodzi do vector store.

## Stack

- **FastAPI** (backend ingest + retrieval)
- **Qdrant** self-hosted (vector store)
- **Sentence-transformers** lub **multilingual-e5-base** (embeddings, lokalny CPU)
- **Unstructured** + **PyPDF2** + **EbookLib** + **python-docx** (parsing dokumentów)
- **Postgres** — metadata, status uploadów, audit
- **Celery + Redis** — kolejka długich zadań (parsing, embedding)

## Źródła v1

| Źródło | Format | Wolumen | Uwaga |
|---|---|---|---|
| Biblia (Warszawska, Tysiąclecia, Poznańska, BWP) | API (api.biblia.info.pl) lub statyczne JSON | ~31k wersetów × 4 tłum. | Każdy werset = oddzielny chunk z metadata `{book, chapter, verse, translation}` |
| Pisma Ellen G. White | API egwwritings.org | 70+ książek po polsku | Faza 1: indeksujemy, ale w whitelist 'sources_for_grounding_only' — AI nie cytuje EGW jawnie |
| SDA Bible Commentary | PDF batch (12 tomów EN) | ~10k stron | AI tłumaczy na polski przy generowaniu; chunki EN z metadata `{volume, page, biblical_ref}` |
| Materiały archeologiczne | PDF/HTML manual | 50-100 dok. na start | Israel Antiquities Authority, BAR Magazine, polskie publikacje. Kuratorowane ręcznie. |
| Upload custom | PDF/EPUB/MD/DOCX | dowolne | Pipeline: pastor upload → AI classify → admin approve |

## Funkcjonalności

### F1. Ingest pipeline

```
[Upload/Fetch] → [Parse] → [Chunk] → [Embed] → [Qdrant upsert]
                  ↓
              [AI classify]
                  ↓
            [Admin review]
                  ↓ (approve)
              [Activate]
```

### F2. Parsing

- **PDF**: PyPDF2 (text-based) lub `unstructured` (z OCR jeśli scan)
- **EPUB**: EbookLib → HTML → text
- **DOCX**: python-docx
- **HTML**: BeautifulSoup + readability
- **Biblia**: structured JSON (per-werset)
- **EGW API**: structured JSON z paragrafami

### F3. Chunking strategy

| Typ źródła | Strategia | Rozmiar chunka | Overlap |
|---|---|---|---|
| Biblia | per werset | 1 werset | 0 |
| EGW / SDA Commentary | semantic (paragrafy) | 200-500 tokens | 50 tokens |
| Archeologia / custom | fixed window | 400 tokens | 80 tokens |

Każdy chunk dostaje metadata:
```python
{
    "source_id": UUID,
    "source_type": "biblia|egw|sda_commentary|archeologia|custom",
    "category": "archeologia|prophecy|science|theology|...",
    "biblical_ref": "Dn 2,31-45" | null,
    "chunk_index": int,
    "doctrine_alignment_score": 0-10,  # AI classify
    "language": "pl|en",
    "uploaded_by": user_id,
    "approved_by": admin_id,
    "approved_at": timestamp,
    "active": bool,
}
```

### F4. Embeddings

- Model: `intfloat/multilingual-e5-base` (768 dim, dobry PL/EN, CPU-friendly)
- Alternative dla wyższej jakości: `BAAI/bge-m3` (1024 dim, lepszy multilingual ale wolniejszy)
- Generowane przez M-AI z rolą `embed`
- Batch przetwarzania: 32 chunki naraz

### F5. Upload workflow (pastor → AI → admin)

```
1. Pastor klika "Dodaj źródło" → upload PDF/EPUB
2. System: extract metadata (autor, tytuł z PDF info)
3. System: parse → chunki → embedding
4. AI classify każdy chunk:
   - kategoria (archeologia/proroctwa/nauka/teologia/...)
   - doctrine_alignment_score (0-10 zgodności z 28 FB ADW)
   - red flags wykryte? (lista cytatów problematycznych)
5. Status: PENDING_REVIEW
6. Admin/pastor widzi w queue:
   - Streszczenie AI: "Książka X autorstwa Y, klasyfikacja: archeologia. Zgodność z ADW: 8.2/10. Wykryte problematyczne fragmenty: 0."
   - Sample 5 chunków (pierwsze, ostatnie, najwyższy score, najniższy score, losowy)
7. Admin: ACCEPT / REJECT (z powodem) / ACCEPT_PARTIAL (oznacza chunki problematyczne jako 'inactive')
8. Po akceptacji: chunki aktywne w retrieval
```

### F6. Retrieval API

```python
class RetrieveRequest(BaseModel):
    query: str
    k: int = 5
    filters: dict = {}  # {"source_type": "biblia", "category": "prophecy", ...}
    rerank: bool = True

class RetrieveChunk(BaseModel):
    text: str
    metadata: dict
    score: float

# Implementacja:
# 1. Embed query (przez M-AI z rolą 'embed')
# 2. Qdrant search z filtrami
# 3. Opcjonalny rerank (cross-encoder na top 20 → top 5)
# 4. Return
```

Strategia per-zastosowanie:
- **Generowanie postu**: hybrid retrieval — `query=temat_posta` + 1-2 wersety biblijne (jeśli zadane explicit)
- **Q&A pipeline**: query = pytanie użytkownika, k=10, rerank=True
- **LLM-judge** (M-GUARD): query = wygenerowany post, retrieve doctrine cards (oddzielny vector store managed przez M-GUARD)

### F7. Source management UI

- Lista wszystkich źródeł z metadata (autor, status, score, liczba chunków, data)
- Filtry: typ, status, kategoria
- Akcje: aktywuj/dezaktywuj, ponowne embedowanie (jeśli model embeddingu się zmienia), download oryginału
- Statystyki: top-10 najczęściej cytowanych chunków w generowanych postach

### F8. Re-indexing

- Gdy admin zmienia model embeddingu w M-AI:
  - System ostrzega: "Zmiana wymaga re-embed wszystkich N chunków, koszt szacowany: X"
  - Po potwierdzeniu: Celery task background, progress tracking

## Schemat danych (Postgres)

```sql
sources
  id UUID PK, title, author, source_type, format,
  uploaded_by FK, uploaded_at,
  status (pending_classify | pending_review | active | rejected | inactive),
  rejection_reason TEXT NULL,
  approved_by FK NULL, approved_at NULL,
  doctrine_alignment_score FLOAT NULL,
  ai_summary TEXT NULL,
  file_path TEXT (lokalny lub S3-compatible)

source_chunks  (mirror w Qdrant)
  id UUID PK, source_id FK, chunk_index INT,
  text TEXT, embedding_model VARCHAR,
  category VARCHAR, biblical_ref VARCHAR NULL,
  doctrine_alignment_score FLOAT,
  red_flags JSONB,  -- [{type, text, severity}]
  active BOOL DEFAULT TRUE

source_audit
  id, source_id FK, action (uploaded|classified|approved|rejected|deactivated),
  actor_id FK, timestamp, details JSONB
```

## API / Kontrakty

```
POST /api/sources/upload          (PASTOR) multipart       → 202 (async)
GET  /api/sources                 ?status&type             → Source[]
GET  /api/sources/{id}                                     → Source z chunkami
POST /api/sources/{id}/approve    (ADMIN_TECH)             → 200
POST /api/sources/{id}/reject     (ADMIN_TECH) {reason}    → 200
POST /api/sources/{id}/deactivate (ADMIN_TECH)             → 200
GET  /api/sources/{id}/chunks     ?active                  → Chunk[]

POST /api/rag/retrieve            {query, k, filters}      → RetrieveChunk[]
POST /api/rag/embed               {texts}                  → number[][]   (proxy do M-AI)

# Eventy
- source.uploaded { source_id, uploaded_by }
- source.classified { source_id, ai_summary, alignment_score }
- source.approved { source_id, approved_by }
- source.rejected { source_id, reason }
```

## Definition of Done

- [ ] Pastor może uploadować PDF i po 5-10 min widzi w queue review z AI classification
- [ ] Admin może zaakceptować źródło → chunki dostępne w retrieval
- [ ] `POST /api/rag/retrieve` z `query="proroctwo o Tyrze"` zwraca relevantne fragmenty (Ez 26 z Biblii + ew. komentarz)
- [ ] Filtry działają: `source_type=biblia` zwraca tylko wersety
- [ ] Re-embedding: zmiana modelu w admin UI → background task migruje wszystko, retrieval nie ma downtime (blue-green collection w Qdrant)
- [ ] EGW indexed ale flag 'sources_for_grounding_only' działa (M-AI go używa, ale instrukcja "nie cytuj jawnie" w system prompcie)
- [ ] Polskie znaki w PDFach OCR: poprawnie zachowane (ą, ę, ć, ł itp.)
- [ ] Test E2E: upload książki → classify → approve → retrieve → fragment użyty w generowanym poście (M-EDIT)

## Edge cases i decyzje

- **PDF z scanami (OCR)**: `unstructured` z `tesseract-ocr-pol` w kontenerze. Wolne, ale jednorazowo.
- **Bardzo długi dokument**: parsing > 5 min → status `processing` z progress; Celery, nie HTTP timeout
- **Duplikaty**: hash treści (SHA-256) — odrzucamy upload duplikatu z komunikatem "źródło X już istnieje"
- **Niesprawdzone źródła z internetu**: explicit warning "to źródło może zawierać niespójności doktrynalne" — admin świadomie akceptuje
- **Prawa autorskie**: dokumenty trzymane na serwerze parafii, brak dystrybucji publicznej. SDA Commentary to fair-use cytatów; własne reformułowania w postach, nie kopia.
- **Język**: SDA Commentary EN → AI tłumaczy do PL przy generowaniu posta (instrukcja w system prompcie); chunki przechowywane w EN
- **Biblia tłumaczenia**: AI domyślnie cytuje Biblię Warszawską (najpopularniejsza w PL). Admin może zmienić default.
- **Vector store collision**: nazwy collection w Qdrant: `bnd_sources_v1`, `bnd_doctrine_v1` (M-GUARD). Wersjonowanie pozwala migrować.

## Zależności

**Wchodzi**: M-INFRA (Postgres, Qdrant, Redis), M-AI (embeddings + classification), M-AUTH (RBAC)
**Wychodzi**:
- M-EDIT używa do generowania postów
- M-GUARD ma własny vector store, ale współdzieli Qdrant instance
- M-COMM Q&A pipeline retrievuje przy generowaniu odpowiedzi

## Risks

| Risk | Mit. |
|---|---|
| Indeksacja źródeł sprzecznych z ADW | Pre-classification AI + manual review przez pastora; whitelist tematów (faza 1: bez EGW jawnie) |
| Halucynacja "cytatów" Biblii | Każdy wygenerowany cytat M-EDIT weryfikuje przez M-RAG retrieve i porównuje string-similarity > 95% |
| Qdrant pada → brak retrieval | Fallback: pełnotekstowe wyszukiwanie Postgres (fts) — gorsza jakość, ale działa |
| Wolny ingest (1000+ stron PDF) | Background task + estimowanie czasu + email do uploadującego "gotowe" |
| Embedding model deprecation | Re-embed via blue-green collection swap; old collection żyje do walidacji |
