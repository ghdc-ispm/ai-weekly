# AI Weekly

A weekly AI news page for executives, workers worried about their jobs, and developers. It filters out hype, doom and PR, and keeps concrete news, how-tos and analysis.

Free-tier stack, Python standard library only (no third-party packages).

## How it works

```
GitHub Actions (cron, Mondays 07:00 UTC)
  -> Google News RSS (6 queries, last 7 days)
  -> Gemini: score + classify headlines
  -> Gemini: group duplicate stories
  -> Frontier labs block (OpenAI, Anthropic, Google, Meta)
  -> docs/data.json + docs/archive/YYYY-Www.json
  -> GitHub Pages renders docs/index.html
```

### Pipeline (`scripts/build.py`)
1. Fetch Google News RSS for several queries and dedupe by title.
2. Gemini pass 1 scores each headline (0-10), assigns a type (`news`, `howto`, `analysis`, `hype`) and a category.
3. Optional pass 2 (`SECOND_PASS=1`) re-scores borderline items using article text. Experimental: Google News links redirect, so it may not work.
4. Keep score >= 7 and not `hype`.
5. Gemini groups the same story reported by several outlets; extra outlets appear as "Also: ...".
6. Group by category (`analysis` items go to "Food for thought"), top 5 per category.
7. Frontier labs: official feeds where available, otherwise Google News `site:` queries; Gemini keeps only the lab's own releases and features.
8. Write `docs/data.json`, a weekly archive file and `docs/archive/index.json`.

### Front end (`docs/index.html`)
Static page. Masthead, top-story hero, fixed 3x2 category grid with a "Read more" button per card, a full-width Frontier labs strip, past-issue links, dark mode and a single-column mobile layout. It uses `textContent` only, allows only `https` links and sets a strict Content-Security-Policy.

## Repo layout

```
.github/workflows/weekly.yml   schedule + steps
scripts/build.py               fetch, classify, write JSON
docs/index.html                static page served by Pages
docs/data.json                 latest issue (generated)
docs/archive/                  one JSON per week + index.json (generated)
```

## Setup

1. Create a **public** repo and add the files above.
2. Get a Gemini API key from Google AI Studio.
3. In the repo: **Settings -> Secrets and variables -> Actions**
   - Secret: `GEMINI_API_KEY`
   - Variable: `GEMINI_MODEL` (a current Gemini Flash model name; check Google's docs)
4. **Settings -> Pages**: deploy from branch `main`, folder `/docs`.
5. **Settings -> Actions -> General -> Workflow permissions**: read and write.
6. **Actions** tab -> `weekly-news` -> **Run workflow** to test without waiting for Monday.

## Configuration

| Where | What |
|---|---|
| `QUERIES` in `build.py` | Google News search terms |
| `RUBRIC` in `build.py` | What counts as signal vs noise; tune this first |
| `LABS` in `build.py` | Frontier lab sources |
| `SECOND_PASS` (workflow env) | `"1"` enables article-text re-scoring |
| `--ch`, `--lh` in `index.html` | Card heights for the fixed grid |

## Tuning the filter

Headline-only classification has limits, so expect some misses. Review the first few runs, note what slipped through or was dropped, and adjust `RUBRIC`. Temporarily printing the model's `why` field helps.

## Security notes

- The repo is public: code, prompts, commit history and (to logged-in users) Actions logs are visible. Secret values are masked, but never print the key or request headers.
- Headlines and feed text are untrusted input. Gemini calls have no tools and return JSON only, and the prompts tell the model to ignore instructions inside the data.
- Pin actions to full commit SHAs before going live (currently `@v5` tags). Do not use `pull_request_target`.
- Enable 2FA, secret scanning and push protection, and require approval for fork pull requests.
- If the API key ever leaks, revoke it in AI Studio and create a new one.

## Known limitations

- Google News may return `503` from GitHub's shared IPs; retries are recommended in `get()`.
- Google and Meta lab sources return few items; feed URLs are unverified.
- With 7+ categories, the smallest is hidden in the grid.
- Cross-week duplicates are not removed.
- GitHub may disable scheduled workflows after about 60 days without repo activity; check after a few weeks.
- Model names, free-tier limits and GitHub UI labels change; verify against current docs.

## Roadmap

- RSS `feed.xml` (enables free RSS-to-email newsletter tools later)
- Fixed category order
- Stronger model for the Frontier labs classification
