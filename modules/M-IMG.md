# M-IMG — Image Generation Pipeline

> **Faza**: 2 (Media + Społeczność) · **Zespół**: 1 osoba (dev) + grafik konsultant · **Estymacja**: 4-5 tyg.

## Cel

Generowanie brandowanych grafik dla każdego posta z **HITL na poziomie promptu** — pastor zatwierdza brief graficzny PRZED generowaniem obrazu (oszczędność kosztów + bezpieczeństwo). W warunkach budżetu < 5k PLN/rok i braku GPU: priorytetem są **szablony + AI tekst**, image gen tylko jako fallback.

## Stack

- **Backend**: FastAPI
- **Image processing**: Pillow (PIL), `cairo-svg` lub `playwright` do renderowania HTML→PNG
- **Image gen** (opcjonalne, fallback):
  - **Pollinations.ai API** (free tier, niestabilne ale 0 PLN)
  - **Together.ai Flux Schnell API** (~$0.001/img, jakość OK)
  - **Replicate Flux Schnell** (~$0.003/img)
  - Brak self-hosted Flux (wymaga GPU 12GB+ VRAM)
- **Storage**: lokalne filesystem + serve przez Caddy z auto-cache; opcjonalnie Backblaze B2 dla publicznych URL (IG wymaga publicznego)
- **Postgres** — metadata, prompts versioned
- **Redis** — cache wygenerowanych obrazów (hash promptu)

## Dwa tryby (skonfigurowane per-post lub default)

### Tryb A: Templates + AI text (PRIORYTET, default)

Grafik (1 osoba zewn.) projektuje 10-15 szablonów w Figma:
- T1: Hook+grafika tła (proroctwo)
- T2: Cytat biblijny + tło ilustracyjne
- T3: Galeria postaci (do wpisu o naukowcach)
- T4: Mapa + zaznaczenie (archeologia)
- T5: Infografika 3-step
- T6: Carousel cover (IG)
- T7: Carousel slide (IG)
- T8: Quote card (IG)
- T9: Statystyka / liczba (np. "2700 lat")
- T10: Dwupanelowa porównawcza
- T11-15: warianty stylistyczne

Każdy szablon to:
- SVG/HTML z slotami `{title}`, `{subtitle}`, `{caption}`, `{image_url}`
- Reguły layoutu (max znaków per slot)
- Brand identity z STRATEGIA-FANPAGE.md sekcja 6 (granat/grafit + złoto/biel, serif headings)

AI generuje "image brief" (M-AI z rolą `image-brief`):
```json
{
  "template": "T1",
  "title": "PIECZĘĆ KRÓLA EZECHIASZA",
  "subtitle": "2700 lat · Jerozolima · 2015",
  "caption": "2 Krl 18,1",
  "background_image_query": "ancient hebrew seal artifact close-up dark background",
  "background_image_source": "unsplash | wikipedia | manual"
}
```

Pastor zatwierdza brief. M-IMG renderuje przez `playwright` HTML→PNG (1080x1080 / 1080x1350 / 1080x1920).

### Tryb B: AI image generation end-to-end

Wybierany świadomie przez pastora gdy szablon nie pasuje (np. unikalna scena).

```
[AI generuje opis sceny]
   ↓
[Pastor zatwierdza prompt]
   ↓
[Image gen API call]
   ↓
[Pastor zatwierdza wygenerowany obraz]
   ↓
[Post-processing: resize, watermark, brand bar w stopce]
```

**Hard rules** dla AI image gen:
- Nigdy nie generujemy postaci Chrystusa, Boga, aniołów (ryzyko herezji wizualnej + ADW preferencja braku obrazów Boskości)
- Nigdy konkretni żyjący politycy/duchowni
- Krajobrazy, artefakty, mapy, abstrakcje — OK

## Funkcjonalności

### F1. Template registry

```
ImageTemplate
  id, code (T1..T15), name, description,
  channel ENUM (FB | IG | BOTH | STORY),
  format VARCHAR (1080x1080 | 1080x1350 | 1080x1920),
  html_template TEXT (Jinja2),
  required_slots JSONB,  -- [{name: "title", max_chars: 50}, ...]
  preview_image_url, version INT, active BOOL
```

Rendering:
```python
def render_template(template: ImageTemplate, slots: dict) -> bytes:
    html = jinja2.render(template.html_template, **slots)
    page = await playwright.new_page(viewport={"width": ..., "height": ...})
    await page.set_content(html)
    return await page.screenshot(type="png", omit_background=False)
```

### F2. Image brief generator (M-AI integration)

```python
async def generate_image_brief(post: Post) -> ImageBrief:
    request = AIRequest(
        role="image-brief",
        messages=[
            Message(role="system", content=IMAGE_BRIEF_SYSTEM_PROMPT),
            Message(role="user", content=f"""
                Post: {post.fb_content}
                Kategoria: {post.brief.topic_category}

                Wygeneruj brief graficzny.
                Zwróć JSON: {{template, title, subtitle, caption, background_image_query, ...}}
            """)
        ]
    )
    response = await ai.generate(request)
    return ImageBrief.parse_raw(response.content)
```

System prompt zawiera:
- Listę dostępnych szablonów z opisami
- Brand guidelines (paleta, typografia)
- Reguły: max znaków per slot

### F3. Background image sourcing

3 strategie:
- **Unsplash API** (free, wymaga atrybucji): query → fetch → check quality
- **Wikipedia / Commons**: dla artefaktów archeologicznych z otwartą licencją
- **Manual upload** (admin): jeśli AI nie znajdzie — admin uploaduje (np. zdjęcie z papierowego artykułu)

Cache: każdy fetch zapisany w `image_cache` z hashem query.

### F4. AI image gen (tryb B)

```python
async def generate_ai_image(prompt: str) -> bytes:
    # Try providers in order: Pollinations → Together → Replicate
    for provider in [pollinations, together, replicate]:
        try:
            return await provider.generate(prompt, model="flux-schnell")
        except (RateLimitError, ProviderDownError):
            continue
    raise NoImageProviderAvailable()
```

Hard guards (przed wywołaniem):
- Prompt nie zawiera `Jezus|Chrystus|Bóg|anioł|polityk` (regex blacklist)
- Prompt < 500 znaków
- Pastor zatwierdził prompt explicit

### F5. HITL workflow (Human in the Loop)

```
[Post APPROVED w M-EDIT]
       ↓
[M-IMG: generate brief (AI)]
       ↓
[Pastor widzi brief + sample tłem]
       ↓
   [Akceptuj] / [Edytuj] / [Wybierz inny szablon] / [Wygeneruj inaczej]
       ↓
[Render (template) lub generate (AI)]
       ↓
[Pastor widzi finalny obraz]
       ↓
   [Akceptuj] / [Regeneruj] / [Edytuj brief]
       ↓
[Image attached do posta → M-PUB]
```

Maksymalnie 3 iteracje briefu, potem MANUAL_REVIEW.

### F6. Multi-format generation

Z jednego briefu generuje 3 warianty:
- 1080x1080 (FB feed square)
- 1080x1350 (FB feed 4:5, więcej miejsca)
- 1080x1920 (IG Stories / Reels cover)

Templates są designed jako responsive — render z różnymi `viewport`.

### F7. Brand bar / watermark

Każdy obraz ma w stopce dyskretny pasek:
- Logo (lewy róg)
- "BibliaNaukaDowody.pl" (prawy róg)
- Podpis cytatu / źródła (jeśli aplikable)

Definiowane w template (Jinja2 component).

### F8. Image cache i CDN

- Wygenerowane obrazy: `/var/lib/bnd/images/{post_id}/{format}.png`
- Cache hash: `template_id + slots_hash + bg_image_hash`
- Serwowane przez Caddy: `https://cdn.bnd.pl/images/{post_id}/{format}.png`
- IG wymaga publicznego URL → tu Caddy; alternatywnie B2 public bucket
- Retention: 24 mies. po publikacji posta (potem soft-delete)

## API / Kontrakty

```
POST /api/images/brief             {post_id}                 → ImageBrief
POST /api/images/brief/{id}/edit   (PASTOR)                  → 200
POST /api/images/render            {brief_id, format}        → ImageBytes (or URL)
POST /api/images/regenerate        {brief_id}                → ImageBytes (alternative)
POST /api/images/{id}/approve      (PASTOR)                  → 200, attached do posta
POST /api/images/{id}/reject       (PASTOR) {feedback}       → triggeruje regen
GET  /api/images/templates                                    → ImageTemplate[]
POST /api/images/templates         (ADMIN)                   → 201
GET  /api/images/{post_id}/{format}.png                       → image (serwowany przez CDN)

# Eventy
- image.brief_generated { post_id, brief }
- image.brief_approved { post_id, by }
- image.rendered { post_id, format, mode (template|ai) }
- image.approved { post_id, image_url }
- image.regen_requested { post_id, attempt }
```

## Definition of Done

- [ ] 5 szablonów (T1-T5) gotowych w Figma + skonwertowanych do Jinja2 HTML
- [ ] Pipeline: post APPROVED → AI generuje brief → pastor zatwierdza → render → preview
- [ ] 3 formaty (1080x1080, 1080x1350, 1080x1920) generują się z jednego briefu
- [ ] Brand bar w każdym wygenerowanym obrazie
- [ ] Hard guards: prompt z "Jezus" → blokada
- [ ] Pollinations API integration działa (fallback gdy template nie pasuje)
- [ ] Storage + CDN: obraz dostępny pod publicznym URL po renderze
- [ ] M-PUB konsumuje image URL przy publikacji FB+IG
- [ ] Pastor może odrzucić obraz i triggerować regen z feedbackiem
- [ ] Cache: regen tego samego briefu w 1h zwraca z cache (cost saving)

## Edge cases i decyzje

- **Brief nie pasuje do żadnego szablonu**: AI proponuje "custom" → tryb B (image gen)
- **AI image gen down (wszyscy providers)**: fallback na "blank template z tytułem" (degradacja, ale post idzie)
- **Tekst nie mieści się w slot**: AI dostaje feedback "skróć title do 50 znaków" → regen briefu
- **Polskie znaki w renderingu**: font Inter + Crimson Text z subset PL — ą, ę, ć, ł, ń, ó, ś, ź, ż przetestowane
- **Wysokie DPI**: rendering w 2x rozdzielczości (2160x2160), downscale do target — ostrość
- **Concurrent generation**: rate limit 5 równoległych renderów (playwright RAM-hungry)
- **Background image rights**: każde Unsplash → atrybucja w metadata (nie zawsze widoczna w grafice, ale w post_audit), licencja CC0 priorytetem
- **AI image gen halucynuje**: pastor zawsze widzi przed publikacją; auto-rejection jeśli generowanie wraca z confidence < 0.5 (jeśli provider wspiera)

## Zależności

**Wchodzi**: M-INFRA (storage, Caddy CDN), M-AUTH (PASTOR for HITL), M-AI (image-brief generation, image gen via providers), M-EDIT (post.fb_content jako wejście)
**Wychodzi**: M-PUB (image_url do publikacji)

## Risks

| Risk | Mit. |
|---|---|
| Wygenerowany obraz herezyjny / niewłaściwy | HITL pastor zatwierdza, hard guards na prompcie, blacklist tematów |
| Polskie znaki źle renderowane | Subset font + test golden file; fallback monospace |
| Storage zapełnia dysk | Retention policy 24 mies., monthly archive starych do B2 |
| Provider down (Pollinations często) | 3-tier fallback: Pollinations → Together → Replicate → template degradation |
| Prawa autorskie do background images | Tylko CC0/Unsplash; uploaded manually wymaga "potwierdzam licencję" checkbox |
| Brand inconsistency (różne fonty/kolory) | Wszystkie templates wspólnie z grafikiem; wersjonowane; PR-review na zmiany |
