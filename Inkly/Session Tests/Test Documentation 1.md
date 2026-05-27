# First Tests for my Gaussian Scraper

**Date:** 2026-05-27  
**File in repo:** `tests/test_gaussian_scraper.py`  
**Result:** 7/7 tests passed

---

## What These Tests Cover

| Test                                                | Module    | What It Checks                               |
| --------------------------------------------------- | --------- | -------------------------------------------- |
| `test_extract_returns_passages_containing_keywords` | extractor | Keyword matches are included                 |
| `test_extract_ignores_short_lines`                  | extractor | Lines below minimum length are skipped       |
| `test_extract_returns_empty_when_no_matches`        | extractor | No false positives when nothing matches      |
| `test_extract_caps_results_at_max_passages`         | extractor | Result count is capped correctly             |
| `test_fetch_returns_none_on_request_error`          | fetcher   | Network failure returns `None` cleanly       |
| `test_fetch_returns_none_on_non_200_status`         | fetcher   | HTTP errors (e.g. 404) return `None` cleanly |
| `test_fetch_extracts_text_from_html`                | fetcher   | Real HTML parsed correctly, noise stripped   |

All fetcher tests use `monkeypatch` to fake the HTTP call 

---

## The Tests

```python
from __future__ import annotations

from inkly.scraper import extractor, fetcher
from inkly.scraper.extractor import extract_relevant_passages
from inkly.scraper.fetcher import fetch_page_text


# --- extractor tests ---

def test_extract_returns_passages_containing_keywords():
    text = "\n".join([
        "Load the Gaussian module before submitting your job.",
        "This line has nothing relevant in it at all and is long enough.",
        "Use %mem and %nproc in your Gaussian input file to request resources.",
        "Another irrelevant line that contains no matching keywords whatsoever.",
    ])

    results = extract_relevant_passages(text, keywords=["gaussian", "%mem"])

    assert any("Gaussian" in p for p in results)
    assert any("%mem" in p for p in results)


def test_extract_ignores_short_lines():
    text = "\n".join([
        "Gaussian",              # too short
        "g16",                   # too short
        "Use Gaussian 16 with the appropriate memory settings in your Slurm script.",
    ])

    results = extract_relevant_passages(text, keywords=["gaussian", "g16"])

    assert len(results) == 1
    assert "Use Gaussian 16" in results[0]


def test_extract_returns_empty_when_no_matches():
    text = "\n".join([
        "This documentation page is about a completely unrelated topic.",
        "There are no chemistry keywords anywhere in this passage at all.",
    ])

    results = extract_relevant_passages(text, keywords=["gaussian", "g16"])

    assert results == []


def test_extract_caps_results_at_max_passages(monkeypatch):
    monkeypatch.setattr(extractor, "MAX_PASSAGES_PER_SOURCE", 3)

    # Build more lines than the cap allows.
    lines = [
        f"Use Gaussian with nproc and mem settings for job number {i}."
        for i in range(10)
    ]
    text = "\n".join(lines)

    results = extract_relevant_passages(text, keywords=["gaussian"])

    assert len(results) == 3


# --- fetcher tests ---

def test_fetch_returns_none_on_request_error(monkeypatch):
    import requests

    def fake_get(*args, **kwargs):
        raise requests.RequestException("connection refused")

    monkeypatch.setattr(requests, "get", fake_get)

    result = fetch_page_text("http://fake-url.example.com")

    assert result is None


def test_fetch_returns_none_on_non_200_status(monkeypatch):
    import requests
    from types import SimpleNamespace

    monkeypatch.setattr(
        requests,
        "get",
        lambda *a, **kw: SimpleNamespace(status_code=404, text=""),
    )

    result = fetch_page_text("http://fake-url.example.com")

    assert result is None


def test_fetch_extracts_text_from_html(monkeypatch):
    import requests
    from types import SimpleNamespace

    fake_html = """
    <html>
      <body>
        <nav>Skip this nav content</nav>
        <p>Load the Gaussian module before submitting your job.</p>
        <li>Request memory carefully in your Slurm script.</li>
        <footer>Skip this footer</footer>
      </body>
    </html>
    """

    monkeypatch.setattr(
        requests,
        "get",
        lambda *a, **kw: SimpleNamespace(status_code=200, text=fake_html),
    )

    result = fetch_page_text("http://fake-url.example.com")

    assert result is not None
    assert "Load the Gaussian module" in result
    assert "Request memory carefully" in result
    assert "Skip this nav content" not in result
    assert "Skip this footer" not in result
```

---

## Key Patterns Used

**`monkeypatch.setattr` on a module constant**
```python
monkeypatch.setattr(extractor, "MAX_PASSAGES_PER_SOURCE", 3)
```
Temporarily overrides the cap value so we can test the limiting behaviour without needing 10+ matching lines in real data.

**`monkeypatch.setattr` on a library function**
```python
monkeypatch.setattr(requests, "get", fake_get)
```
Replaces the real `requests.get` with a fake for the duration of the test. This means no real HTTP requests are made — the test runs offline and deterministically every time.

**`SimpleNamespace`**
```python
SimpleNamespace(status_code=404, text="")
```
A quick way to create a fake object with specific attributes. Used here to mimic a `requests.Response` object without importing or constructing the real thing.

---

## Related Notes

- [[Gaussian Scraper Final Thoughts]]  the code these tests are written for
- [[Testing Inkly]]  general testing concepts and patterns used here
- [[Work Session 1]] the session where these tests were written
