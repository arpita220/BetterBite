# BetterBite

Check a packaged food before you buy it. BetterBite flags the ingredients worth avoiding, explains them in plain language, and suggests healthier brands in the same category with prices side by side.

![Python](https://img.shields.io/badge/Python-3.11+-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white)
![Next.js](https://img.shields.io/badge/Next.js-14-000000?logo=nextdotjs&logoColor=white)
![SerpApi](https://img.shields.io/badge/Powered%20by-SerpApi-4E7CE0)

Built for the **SerpApi India Hackathon 2026**, Knowledge & Public Interest track.

**Demo video:** [ADD YOUTUBE LINK]
**Live app:** [ADD LINK, or delete this line if it's local only]

![BetterBite screenshot](docs/screenshot-results.png)

## Why I built this

Most people can't read an ingredient list and tell what matters. Additive codes mean nothing in a shop aisle, and nobody knows if the "healthier" option on the next shelf costs 20 rupees more or 200. BetterBite answers both questions in a few seconds.

## What it does

- Takes a photo of a pack, or a typed product or category like "Maggi" or "chips".
- Lists the ingredients in that product that are worth avoiding, each with a one-line explanation. Technical detail is one tap away.
- Suggests several healthier brands in the same category. Every card says why it's better (for example, millet flour instead of refined wheat) and shows its price next to the scanned product.

Categories covered: instant noodles, potato chips, malt drinks, ketchup. [EDIT TO MATCH YOURS]

## How SerpApi is used

| Engine | Used for |
|---|---|
| Google Shopping (`google_shopping`, `gl=in`) | The scanned product's price, then one lookup per suggested alternative. The price comparison on each card comes from these results. |
| Google Lens (`google_lens`) | Identifying a product from an uploaded photo. [DELETE THIS ROW IF PHOTO SCANNING ISN'T IN YOUR SUBMITTED VERSION] |

Lookups are independent, so one failed request doesn't blank the whole result. If a price isn't found, the card says so.

## Project structure

```
backend/
  main.py                    FastAPI app and endpoints
  models.py                  Pydantic schemas
  services/
    serpapi_service.py       SerpApi calls
    additive_db.py           Ingredient reference table
    alternatives_db.py       Alternative brands by category
    analyzer.py              Ties the lookups together
    demo_cache.py            Saved results for the preset buttons
frontend/
  app/                       Next.js pages
  components/                UI components
  lib/api.ts                 Typed client for the backend
```

[CHECK THIS AGAINST YOUR REAL FOLDERS]

## Getting started

You need Python 3.11+, Node 18+, and a free SerpApi key from https://serpapi.com/manage-api-key.

### Backend

```bash
cd backend
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
```

Put your key in `.env`:

```
SERPAPI_API_KEY=your_key_here
```

Then start the server:

```bash
uvicorn main:app --reload --port 8000
```

### Frontend

```bash
cd frontend
npm install
cp .env.local.example .env.local
npm run dev
```

Open http://localhost:3000.

### No API key?

The preset buttons (Maggi, Lay's, Bournvita, Kissan) load saved results, so you can see the full screen without any API calls. Typing a product or uploading a photo runs live searches and needs the key.

## API

| Method | Path | What it does |
|---|---|---|
| GET | `/api/health` | Confirms the server is up and whether a key is set |
| GET | `/api/demo/presets` | Lists the saved demo products |
| GET | `/api/demo/{key}` | Returns a saved result |
| POST | `/api/analyze` | Analyzes a product by name |
| POST | `/api/analyze/upload` | Analyzes an uploaded photo |

[CHECK THESE AGAINST YOUR REAL ROUTES]

## Adding a category

Open `backend/services/alternatives_db.py`, add a block with the match keywords and three or four alternative brands, and restart the backend.

## Limitations

- The ingredient notes and alternative brands come from small tables maintained by hand. They aren't a complete or official database and haven't all been fact-checked.
- Prices are whatever Google Shopping returns at the time of the search. Pack sizes differ, so compare per 100 g yourself unless the card shows it. [EDIT IF YOU BUILT PER-100 G PRICING]
- Photo scanning only works when the backend is reachable from the internet, because SerpApi fetches the image from a URL.
- This isn't medical or regulatory advice. Check the pack and current FSSAI rules before relying on anything shown here.

## Roadmap

- More food categories
- Price per 100 g on every card
- Hindi labels
- Shareable result image for WhatsApp

[KEEP ONLY WHAT YOU REALLY PLAN TO DO]

## AI tools

Built with Claude (Anthropic), which generated the first version of the backend and frontend and helped draft the ingredient notes and interface text. I reviewed and tested the code. [ADD ANY OTHER TOOLS]

## License

[CHOOSE ONE, e.g. MIT, and add a LICENSE file. If you don't add one, others have no right to reuse the code.]

## Acknowledgements

Search data from [SerpApi](https://serpapi.com).
