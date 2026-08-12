# Work Session 10

**Commit Date:** 2026-08-12
**Goal:** Actually test the Ollama summarization feature end to end with a real local model, instead of trusting that it works because the code looks right.

---

## Plan

- [x] Install Ollama and pull the model the scraper defaults to
- [x] Test summarization against real scraped passages
- [x] Diagnose why the first real test silently failed
- [x] Fix the root cause and make the fix configurable
- [x] Verify the fix against the real model again
- [x] Run a full scrape of both domains with real summarization enabled
- [x] Decide what to do about the sources that still miss the timeout

---

## Installing Ollama

Ollama was not installed on my laptop at all before this session, not just not running. 

1. Downloaded and ran the Windows installer from ollama.com. It installs as a background service that starts automatically.
2. Verified the install: the `ollama` CLI was not immediately on PATH in the current shell session (a stale PATH in an already-running shell, common right after a fresh install), but the server itself was already reachable at `http://localhost:11434` returning HTTP 200, and the executable was found directly at `%LOCALAPPDATA%\Programs\Ollama\ollama.exe`.
3. Pulled `llama3` (4.7 GB) in the background and confirmed it landed with `ollama list`.

---

## First Real Test: Silent Failure

Called `summarize_passages()` directly against real passages from an already-scraped `gaussian_docs.json` (not mocked, not synthetic). Result: `None`, with no error printed, because the function swallows the underlying exception and returns `None` on any failure so the caller can fall back to storing raw passages.

Diagnosed by bypassing the wrapper and hitting the raw Ollama API directly:
- A trivial "say hello" prompt took 26.7 seconds, almost entirely `load_duration` (loading the 4.7 GB model into memory on first use).
- The real summarization prompt (longer, five real passages) took 32.1 seconds on one run and 22.8 seconds on another, both close to or past the scraper's `REQUEST_TIMEOUT_SEC = 30`.
- One later run, using a different real source, took 91.6 seconds.

Conclusion: this was never a bug in the Ollama integration itself. It was a timeout tuned for a much faster inference setup than plain CPU-only local Ollama actually provides. A slow but working Ollama looked identical to a broken one, since the failure was silent by design (so a genuinely unreachable Ollama does not crash the scraper).

---

## The Fix

- Bumped `REQUEST_TIMEOUT_SEC` in `summarizer.py` from 30 to 120, with a comment recording the real observed range (22.8s to 91.6s) so the number is not arbitrary.
- Added `timeout` as a parameter to `summarize_passages()`.
- Added a `--summary-timeout` flag to `scrape.py`, following the exact same pattern as the `--model` flag from Work Session 9: default unchanged unless overridden, threaded through `run_scrape` and `summarize_results`.
- `summarize_results()` now also prints the active timeout alongside the model name, so it is visible in the run output.
- Two new tests confirming the timeout value actually reaches the underlying request, and that the default still applies when not overridden.

---

## Verification

Retested against the real model, not mocked, twice more:
- A direct `summarize_passages()` call on a different real source succeeded in 91.6 seconds, comfortably inside the new 120s default, and would have failed every time under the old 30s limit.
- The actual `summarize_results()` wrapper (the function `run_scrape` calls) was tested directly too, confirming it stores the summary into the result dictionary correctly, not just that the lower-level function returns text.
- Summary content was checked for accuracy against the source passages both times: no fabricated facts, consistent with the anti-hallucination prompt rules from Work Session 7.

118/118 tests passing.

---

## Full Scrape With Real Summarization

Ran both saved configs end to end with the fixed timeout, not a sample, the actual production configs: `py scrape.py --config configs/gaussian.toml` and the same for bioinformatics. Ran in the background since a full run with real per-source summarization takes a while.

Results:
- Gaussian: 25 sources scraped, 556 passages, 19 of 25 sources got a summary.
- Bioinformatics: 21 sources scraped, 468 passages, 15 of 21 sources got a summary.
- Scraping, dedup, and saving succeeded for every single source in both domains. The gaps are only in the optional summarization step, which is designed to fall back to raw passages rather than fail the run.

Diagnosed one of the misses directly the same way as the earlier timeout bug: a Matter Modeling Stack Exchange source had a longer prompt (3416 characters versus about 1182 for a typical HTML source) and took 204.9 seconds with no timeout applied, well past the 120s default.

This showed my first theory was too narrow. It is not simply "Stack Exchange sources are slower." Two plain HTML pages (Harvard RC, HPC Wiki) also missed the timeout, while several Stack Exchange tags (quantum-chemistry, basis-sets, td-dft, hpc) succeeded well within it. The real pattern is plain CPU-inference timing variance across a long sequential run of 46 total summarization calls, not something tied to source type. There is no single fixed timeout that guarantees zero fallbacks under that kind of variance, only a tradeoff between how long a slow-but-working request is allowed to run before giving up.

**Decision: leave `REQUEST_TIMEOUT_SEC` at 120s, no further code change.** `--summary-timeout` already exists as a manual override for anyone who wants to push for fuller coverage on a specific run. The current default already fixed the original silent-failure bug (30s was catching legitimate successful requests), and the remaining gap is a genuine tradeoff rather than a bug to chase further right now.

---

## Related Notes

- [[Work Session 9]] - the `--model` flag this session's `--summary-timeout` flag directly follows the pattern of
- [[Work Session 7]] - the anti-hallucination summarizer prompt this session's real summaries were checked against
