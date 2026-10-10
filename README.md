# AdulTrace (LabelTruth India)

**Checks packaged food for adulteration — and keeps the dual-standard comparison in its own view.**

Built for the **SerpApi India Hackathon 2026** — Track 5: *Knowledge & Public
Interest*.

Point AdulTrace at a pack of food and it answers the direct question: what is
in this, and what has been reported about it — additives matched from the
label, the adulterants labs watch for in that product category, and news and
regulatory coverage naming the brand or product. That is the **Adulteration &
product report**, and it is the default view.

A separate tab, **India vs. abroad**, holds the secondary question: how the
same product's listings, prices and additive treatment differ between India and
another market. Multinational brands do sell reformulated versions across
markets — a different oil, preservative or colouring — and the comparison tab
exists to show that gap without pretending it is the same thing as an
adulteration finding.

---

## How it uses SerpApi (3 engines)

| Step | Engine | Feeds |
|---|---|---|
| 1 | `engine=google_lens` | Identity — names the product from the packaging photo (report tab) |
| 2 | `engine=google_shopping` (×2: `gl=in` and `gl=uk/de/us`) | Listings and prices for the **India vs. abroad** tab |
| 3 | `engine=google_news` + `engine=google` | Coverage and food-safety mentions naming the brand/product (report tab) |

All three calls are orchestrated server-side in `backend/services/analyzer.py`
via `backend/services/serpapi_service.py`, which wraps every request in
consistent `{ok, data, error}` handling so one failed engine never blanks
the whole page — the UI just shows a warning banner and renders whatever
did come back.

## Architecture

```
cleanbite/
├── backend/                     FastAPI service
│   ├── main.py                  App entrypoint + REST endpoints
│   ├── models.py                Pydantic request/response schemas
│   ├── services/
│   │   ├── serpapi_service.py   Lens / Shopping / News / Search query builders
│   │   ├── additive_db.py       INS/E-number → risk tier reference table
│   │   ├── analyzer.py          Orchestrates the full pipeline
│   │   ├── price_utils.py       Listing prices → one same-currency comparison
│   │   └── demo_cache.py        Fixtures for offline demos (off by default)
│   ├── requirements.txt
│   └── .env.example
│
└── frontend/                    Next.js 14 (App Router) + TypeScript + Tailwind
    ├── app/
    │   ├── layout.tsx           Fonts (Fraunces / Inter / IBM Plex Mono)
    │   ├── page.tsx
    │   └── globals.css
    ├── components/
    │   ├── Dashboard.tsx        State + the two result tabs
    │   ├── Scanner.tsx          Photo upload / manual entry / demo picker
    │   ├── AdditiveLedger.tsx   What is on the label, per additive (report)
    │   ├── AdulterationHistoryCard.tsx  Unverified news & search mentions (report)
    │   ├── NewsFeed.tsx         News matching the search (report)
    │   ├── ComparisonView.tsx   India vs. overseas listings (comparison)
    │   ├── ScoreGauge.tsx       "Inspection Gap Score" meter (comparison)
    │   └── PriceInsight.tsx     Price difference card (comparison)
    ├── lib/api.ts                Typed fetch client for the backend
    └── package.json
```

### REST API

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/health` | Confirms the server is up, whether `SERPAPI_API_KEY` is configured, and whether demo fixtures are enabled |
| `GET` | `/api/demo/presets` | Lists the cached demo products, or `{"presets": [], "enabled": false}` when switched off |
| `GET` | `/api/demo/{key}` | Returns a cached full analysis (`bournvita`, `lays`, `maggi`, `ketchup`) — 409 unless `DEMO_PRESETS_ENABLED=true` |
| `POST` | `/api/analyze` | JSON body — analyze by product name + pasted ingredients text |
| `POST` | `/api/analyze/upload` | multipart/form-data — analyze from an uploaded packaging photo |

### Price comparison from the listings found

Every analysis also converts the cheapest India listing and the
cheapest overseas listing (from the same two Google Shopping calls above)
into a common currency and states, in one plain sentence, whether that
listing is cheaper or pricier in India — e.g. *"The cheapest listing found
costs about 33% less in India than in UK."* This lives in
`backend/services/price_utils.py` and renders in the frontend as the
**"Price check from these listings"** card. The conversion rate is a small,
clearly-labelled static table (not a live FX feed) so the demo never breaks
on stage — see the note at the top of that file if you want to wire in
SerpApi's `google_finance` engine for live rates instead.

### Not showing false information about a brand

The tool names real companies, so the pipeline is built to report only what
it actually observed:

- **No risk verdicts from keywords.** The old `adulteration_history` payload
  graded each Google hit `Critical`/`Warning`, filed it under an enforcement
  category and printed a red *"High Risk"* badge for the brand. It now returns
  `mentions` — title, snippet, publisher, date, URL and which keywords matched,
  each marked `verified: false`. `risk_level`, `severity` and `category` no
  longer exist anywhere in the payload or the UI.
- **No attribution to the wrong company.** A result is dropped unless it
  actually names the brand or product queried. Brand is also no longer guessed
  from a photo: an inferred brand led to pages about some other company being
  rendered under this one, so `brand` is `null` unless the user typed it.
- **A photo match is labelled a guess.** When a scan comes from an uploaded
  pack, the UI states which product Google Lens visually matched and lists the
  other candidates it saw, because the entire report below is about that match
  and it can be wrong — with a pointer to "Type it in" to correct it.
- **No invented metadata.** Missing dates and publishers stay `null` and render
  as "Date not reported" / "Publisher not stated" instead of "Recent" and
  "Google News", and there are no `#` placeholder links.
- **No unscored product scored.** With no ingredient list to read,
  `inspection_gap_score` is `null` and the gauge says "Not scored" — the old
  code defaulted to `100`, a clean bill of health for a label it had never seen.
- **Absence of coverage is never presented as innocence.** An empty mention
  search says only that the search found nothing, and a failed search says the
  search could not run — the green "no alerts detected" line is gone.
- **Category watchlists are labelled as checklists.** `category_watchlist`
  lists what labs commonly test for in a product *category*, with an explicit
  note that nothing was analysed in a lab here.
- **Provenance is on screen.** Live payloads carry
  `provenance: "live_search_unverified"`; the demo presets are stamped
  `synthetic_demo_fixture` at the API boundary, and the UI shows a banner
  saying which is which. Market labels are derived from the country actually
  queried instead of a hard-coded "Europe (EFSA)".

### Plain-language mode

Every flagged additive now carries two levels of explanation: a one-sentence
`simple_explanation` in everyday language (shown by default), and the fuller
regulatory detail (EFSA/FDA/Prop 65 references) tucked behind a "Show
technical detail" toggle. Risk tiers are labelled **High risk / Caution /
Low risk** instead of jargon. This keeps the tool readable for a general
shopper while still giving judges and journalists the sourced detail on
request.

---

## Setup

### 1. Backend

```bash
cd backend
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env
# then edit .env and paste your key from https://serpapi.com/manage-api-key
#   SERPAPI_API_KEY=your_key_here

uvicorn main:app --reload --port 8000
```

The API is now live at `http://localhost:8000`. Check `http://localhost:8000/api/health`.

> **Note on live photo scanning:** SerpApi's Google Lens engine fetches the
> image from a public URL rather than accepting raw bytes. The backend
> serves uploads at `/uploads/<file>`, which works once the API is deployed
> behind a public URL (Render, Railway, Fly.io, an ngrok tunnel, etc.). Set
> `PUBLIC_BASE_URL` in `.env` to that public origin. Running purely on
> `localhost`, use the **Quick Demo Presets** or the **"Type it in"** manual
> mode instead — both work without a public URL.

### 2. Frontend

```bash
cd frontend
npm install

cp .env.local.example .env.local
# defaults to http://localhost:8000, change if your backend is deployed elsewhere

npm run dev
```

Open `http://localhost:3000`.

### 3. Optional offline demo path (switched off by default)

`backend/services/demo_cache.py` holds four hand-written fixtures (Bournvita,
Lay's, Maggi, Kissan Ketchup) with made-up prices, listing snippets and
article claims about those real companies. Because the app must not present
unverified information about a brand, they are **off by default**:

- `GET /api/demo/presets` returns `{"presets": [], "enabled": false}`, so the
  frontend renders no fixture buttons at all (it used to render a hard-coded
  set even with no backend running).
- `GET /api/demo/{key}` answers `409 demo_presets_disabled`.
- `/api/health` reports `demo_presets_enabled` so you can see the current state.

To bring the offline path back for a presentation with unreliable venue wifi,
set `DEMO_PRESETS_ENABLED=true` in `backend/.env` and restart. Every fixture
response is then stamped `provenance: "synthetic_demo_fixture"` and the UI
paints an amber **"This is not live data"** band over the whole case file — say
that out loud when you demo it, because the numbers were written for the
interface, not fetched about those companies. The strictest option is to delete
`demo_cache.py` and the two `/api/demo/*` routes entirely.

Otherwise everything on screen comes from a live scan: upload a packaging photo
(needs `PUBLIC_BASE_URL`, see the note above) or use **"Type it in"**.

---

## Extending the additive database

`backend/services/additive_db.py` holds the INS/E-number → risk-tier table
and the alias list used to match free-text OCR/pasted ingredients. Add a new
entry to `ADDITIVE_DB` and, if it has common alternate names, an entry to
`_ALIASES` so `parse_ingredients_text()` picks it up automatically — no
other code changes needed.

## Disclaimers

This is a hackathon prototype. The additive risk classifications and demo
fixtures are illustrative starting points assembled from publicly reported
FSSAI/EFSA/FDA actions — **not** a certified regulatory or medical database.
Before publishing any specific claim about a named brand, verify it against
the primary source (FSSAI standards, EFSA opinions, or the product's current
label) — the whole point of the tool is to prompt that scrutiny, not to
replace it.
