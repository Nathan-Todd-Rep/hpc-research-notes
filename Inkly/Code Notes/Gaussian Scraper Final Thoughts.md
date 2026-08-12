# Gaussian Scraper

**Files covered:**
- `inkly/scraper/sources.py` — list of trusted sources and keywords
- `inkly/scraper/fetcher.py` — fetches a URL and returns clean text
- `inkly/scraper/extractor.py` — filters text down to relevant passages
- `scripts/scrape_gaussian_docs.py` — the main runner script
- `inkly/plugins/docs_gaussian.py` — updated to load scraped data

**Built in:** Work Session 1 (2026-05-27)

---

## The Problem This Solves

The original `docs_gaussian.py` plugin served only 5 hardcoded generic lines. That's not enough to give users real, cluster-specific guidance. This scraper pipeline replaces that with actual content pulled from real HPC documentation sites.

---

## How the Pipeline Works

```
scripts/scrape_gaussian_docs.py   (you run this manually to refresh data)
        │
        ├── for each source in sources.py:
        │       ├── fetcher.fetch_page_text(url)      → raw text
        │       └── extractor.extract_relevant_passages(text) → filtered passages
        │
        └── saves all results to ~/.inkly/gaussian_docs.json

inkly/plugins/docs_gaussian.py    (runs automatically when ink is used)
        ├── tries to load ~/.inkly/gaussian_docs.json
        └── falls back to static docs_data.py snippets if file not found
```

---

## `sources.py` — The Seed List

```python
GAUSSIAN_SOURCES = [
    {"label": "Harvard RC — Gaussian", "url": "https://docs.rc.fas.harvard.edu/kb/gaussian/"},
    {"label": "NERSC — Gaussian",       "url": "https://docs.nersc.gov/applications/chemistry/gaussian/"},
    ...
]
```

A curated list of trusted HPC documentation sites known to contain Gaussian content. Adding a new source is as simple as appending a new entry.

Also contains `GAUSSIAN_KEYWORDS` — a list of terms used to decide whether a paragraph is relevant (`"gaussian"`, `"g16"`, `"%mem"`, `"sbatch"`, etc.).

**Why a seed list instead of open web search?**
Open web search requires API keys, is rate-limited, and returns unpredictable results. A curated list is fully reliable, easy to test, and easy to extend.

---

## `fetcher.py` — Fetching and Cleaning a Page

```python
def fetch_page_text(url: str) -> str | None:
```

Steps:
1. Makes an HTTP GET request using `requests` with a browser-like `User-Agent` header — some sites block requests that look like bots.
2. Returns `None` if the request fails or gets a non-200 status code.
3. Parses the HTML with **BeautifulSoup**.
4. Strips noise tags (`<script>`, `<style>`, `<nav>`, `<footer>`, `<header>`).
5. Extracts text only from content tags (`<p>`, `<li>`, `<pre>`, `<code>`, headings).
6. Returns all extracted text joined as a single string, one line per element.

**BeautifulSoup** is a Python library that parses HTML. It lets you navigate and extract parts of a webpage programmatically, without having to parse raw HTML strings yourself.

---

## `extractor.py` — Filtering to Relevant Passages

```python
def extract_relevant_passages(text: str, keywords: list[str] | None = None) -> list[str]:
```

Takes the raw text from the fetcher and keeps only lines that:
- Are at least `MIN_PASSAGE_LENGTH` characters (filters out single-word headings)
- Contain at least one keyword from the list (case-insensitive)

Returns at most `MAX_PASSAGES_PER_SOURCE` passages so the plugin context stays focused and doesn't flood the prompt.

---

## `scripts/scrape_gaussian_docs.py` — The Runner

Run this manually to refresh the data:

```bash
py scripts/scrape_gaussian_docs.py
```

Output:
```
=== Gaussian Docs Scraper ===

Fetching: Harvard RC — Gaussian
  URL: https://docs.rc.fas.harvard.edu/kb/gaussian/
  OK — 8 passages extracted

Fetching: NERSC — Gaussian
  ...

Saved 3 sources to /home/nathan/.inkly/gaussian_docs.json
Total passages collected: 24
```

The JSON file structure looks like:
```json
[
  {
    "label": "Harvard RC — Gaussian",
    "url": "https://...",
    "passages": [
      "Load the Gaussian module with: module load gaussian/g16",
      "..."
    ]
  }
]
```

---

## Updated `docs_gaussian.py` — Load Order

The plugin now tries three things in order:

```
1. Load ~/.inkly/gaussian_docs.json   ← scraped, rich content
2. Load static snippets in docs_data.py  ← simple fallback
3. Return "unavailable" message       ← last resort
```

This means the plugin works immediately even before anyone has run the scraper — the static fallback ensures it never breaks.

The scraped data is prefixed with the source label so the AI knows where each piece of information came from:

```
[Harvard RC — Gaussian]
Load the Gaussian module before submitting your job.
Use %mem and %nproc in your input file to match your Slurm resource request.
...
```

---

## Tests

`tests/test_gaussian_scraper.py` covers:

| Test | What It Checks |
|---|---|
| `test_extract_returns_passages_containing_keywords` | Keywords trigger inclusion |
| `test_extract_ignores_short_lines` | Short lines are filtered out |
| `test_extract_returns_empty_when_no_matches` | No false positives |
| `test_extract_caps_results_at_max_passages` | Cap is enforced |
| `test_fetch_returns_none_on_request_error` | Network failure handled cleanly |
| `test_fetch_returns_none_on_non_200_status` | HTTP errors handled cleanly |
| `test_fetch_extracts_text_from_html` | Real HTML parsed correctly, noise stripped |

All tests use `monkeypatch` to fake HTTP calls — no real network access needed to run them.

---

## Standalone Repository (Work Session 2)

The scraper was moved out of the Inkly repo and into its own standalone GitHub repository: `gaussian-docs-scraper`.

**What changed:**
- Package renamed from `inkly.scraper` to `gaussian_scraper`
- All imports updated throughout
- The scraper module was removed from the `nathan-dev` branch in Inkly
- `docs_gaussian.py` was kept in Inkly unchanged. It reads from the JSON file, so it does not depend on the scraper module being present.

The two repos are now cleanly separated:
- `gaussian-docs-scraper` produces `~/.inkly/gaussian_docs.json`
- Inkly's `docs_gaussian.py` plugin reads from it

---

## What's Next

- Run the scraper on a machine with access to the source URLs to generate real data
- Add Ollama summarization pass: after fetching and filtering, ask Ollama to condense passages into cleaner snippets
- Consider adding a `--keyword` flag to the runner so it can be used for topics other than Gaussian

---

## Related Notes

- [[Plugin System]] — the plugin system this scraper feeds into
- [[Work Session 1]] — the session where this was built
- [[Test Documentation 1]] — the tests written for this scraper
- [[Inkly Project Ideas]] — where the original Gaussian project idea came from
