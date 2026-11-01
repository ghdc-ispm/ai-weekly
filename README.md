# AI Weekly

A weekly AI news page for executives, workers worried about their jobs, and developers. It filters out hype, doom and PR, and keeps concrete news, how-tos and analysis.

Free-tier stack, Python standard library only (no third-party packages).

Live access: https://ghdc-ispm.github.io/ai-weekly

## How it works

```
GitHub Actions (cron, Mondays 07:00 UTC)
  -> Google News RSS (13 queries, last 7 days: general, Swiss, preferred sources)
  -> Gemini: score + classify headlines
  -> Gemini: group duplicate stories
  -> Frontier labs block (OpenAI, Anthropic, Google, Meta)
  -> docs/data.json + docs/archive/YYYY-Www.json
  -> GitHub Pages renders docs/index.html
```

### Pipeline (`scripts/build.py`)
1. Fetch Google News RSS and dedupe by title. Three query groups:
   - **General** (`QUERIES`): ChatGPT, AI assistants, AI at work, privacy, tools, software business.
   - **Swiss** (`SWISS`, English edition for Switzerland): AI and the job market. Not job ads.
   - **Preferred sources** (`PREF`): `site:` queries for XDA, swissinfo.ch and Hacker News. They run first so the item cap cannot cut them.
   Each query keeps at most `PER_QUERY` items, and the total is capped at `MAX_ITEMS`. A failing query is skipped, and `get()` retries on temporary errors (429, 5xx).
2. Gemini pass 1 scores each headline (0-10), assigns a type (`news`, `howto`, `analysis`, `hype`), a category and a `swiss` flag.
3. Optional pass 2 (`SECOND_PASS=1`) re-scores borderline items using article text. Experimental: Google News links redirect, so it may not work.
4. Keep score >= 7 and not `hype`. Items from preferred sources need only `PREF_MIN` (6). **Work & Jobs keeps only Swiss stories.**
5. Gemini groups the same story reported by several outlets; extra outlets appear as "Also: ...".
6. Group by category, top 5 per category. `analysis` items go to "Food for thought", except Work & Jobs items, which stay in Work & Jobs.
7. Frontier labs: official feeds where available, otherwise Google News `site:` queries. Gemini keeps only the lab's own releases and features (customer stories and partnerships are dropped).
8. Write `docs/data.json`, a weekly archive file and `docs/archive/index.json`.

### Front end (`docs/index.html`)
Static page, one file plus fonts.
- Masthead and a top-story box (best-scored item).
- **Six fixed card slots** in two rows: Business & Strategy, Work & Jobs, Tools & Dev / Frontier labs, Privacy & Safety, Food for thought. An empty slot shows a placeholder.
- A full-width **Other** card collects every other category (Other, Rules & Society, any new name). It is hidden when empty.
- Each card has a fixed height on desktop. Items that do not fit are hidden, and a "Read more" button opens the card. On phones the layout is one column with no hidden items.
- Pastel colour bar per category, past-issue links, weekly archive (`?w=YYYY-Www`).
- Safety: `textContent` only (no `innerHTML`), only `https` links, and a strict Content-Security-Policy.

## Repo layout

```
.github/workflows/weekly.yml   schedule (Mondays 07:00 UTC) + manual run; runs build.py, commits docs/
scripts/build.py               fetch, classify, dedupe, write JSON
docs/index.html                static page served by Pages (layout, styles, script)
docs/fonts/                    self-hosted fonts: Noto Serif, Montserrat, Noto Sans (.woff2)
docs/data.json                 latest issue (generated)
docs/archive/                  one JSON per week + index.json (generated)
```

## Setup

1. Create a **public** repo and add the files above, including the three font files in `docs/fonts/`.
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
| `QUERIES` in `build.py` | General Google News search terms |
| `SWISS` in `build.py` | Swiss queries (AI and the job market) |
| `PREF`, `PREF_MIN` in `build.py` | Preferred sources and their lower score threshold |
| `PER_QUERY`, `MAX_ITEMS` in `build.py` | Item caps per query and in total |
| `RUBRIC` in `build.py` | What counts as signal vs noise; tune this first |
| `LABS` in `build.py` | Frontier lab sources |
| `SECOND_PASS` (workflow env) | `"1"` enables article-text re-scoring |
| `SLOTS`, `COL` in `index.html` | Card order and colours |
| `--ch`, `--lh` in `index.html` | Card heights for the fixed grid (desktop) |

## Tuning the filter

Headline-only classification has limits, so expect some misses. Review the first few runs, note what slipped through or was dropped, and adjust `RUBRIC`. Temporarily printing the model's `why` field helps.

## Fonts

Fonts are self-hosted in `docs/fonts/` (latin subset, variable weight), so visitors' browsers make no request to Google. They come from the Fontsource packages and are licensed under the SIL Open Font License 1.1.

## Security notes

- The repo is public: code, prompts, commit history and (to logged-in users) Actions logs are visible. Secret values are masked, but never print the key or request headers.
- Headlines and feed text are untrusted input. Gemini calls have no tools and return JSON only, and the prompts tell the model to ignore instructions inside the data.
- Actions are pinned to full commit SHAs. Do not use `pull_request_target`. Update the SHAs from time to time, or enable Dependabot for GitHub Actions.
- The Content-Security-Policy allows scripts and styles inline, data and fonts from the same site only, and nothing else.
- Enable 2FA, secret scanning and push protection, and require approval for fork pull requests.
- If the API key ever leaks, revoke it in AI Studio and create a new one.

## Known limitations

- Google News may return `503` from GitHub's shared IPs; `get()` retries, and a failing query is skipped.
- Google and Meta lab sources return few items; feed URLs are unverified.
- Hacker News via Google News probably returns few items. The Hacker News search API (Algolia) would be more complete.
- Work & Jobs is Swiss-only, so the card is often empty.
- Long headlines mean only 2 to 3 items fit per card before "Read more".
- Cross-week duplicates are not removed.
- GitHub may disable scheduled workflows after about 60 days without repo activity; check after a few weeks.
- Model names, free-tier limits and GitHub UI labels change; verify against current docs.

## Roadmap

- RSS `feed.xml` (enables free RSS-to-email newsletter tools later)
- Hacker News via its search API
- Stronger model for the Frontier labs classification
- Student and part-time job board (separate repo, recon step first)
