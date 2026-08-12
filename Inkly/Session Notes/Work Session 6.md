# Work Session 6

**Date:** 2026-07-08
**Goal:** Run the scraper, fix dead sources, replace with working alternatives, and collect data into gaussian_docs.json.

---

## Plan

- [x] Run scraper and assess source health
- [x] Update sources.py with working replacements
- [x] Re-run scraper and verify output
- [x] Commit and push

---

## Scraper First Run Results

| Source | Status | Notes |
|---|---|---|
| Harvard RC | OK - 10 passages | Working fine |
| NERSC | HTTP 404 | Page moved or removed |
| Princeton RC | HTTP 403 | Actively blocking scraper |
| University of Michigan | Network error | Timeout or DNS failure |
| Ohio Supercomputer Center | OK - 10 passages | Working fine |
| Chemistry Stack Exchange | Skipped | gaussian tag no longer exists on Chemistry SE |

Only 2 of 5 HTML sources working. Stack Exchange tag was removed entirely.
 :(
## Source Replacements

| Dead source                      | Replacement                                                    | Reason                                                |
| -------------------------------- | -------------------------------------------------------------- | ----------------------------------------------------- |
| NERSC (404)                      | TACC - docs.tacc.utexas.edu/software/gaussian/                 | Solid Gaussian HPC docs, confirmed 200                |
| Princeton RC (403)               | NSC Sweden - nsc.liu.se/software/installed/tetralith/gaussian/ | Confirmed 200, has Gaussian/GaussView content         |
| University of Michigan (timeout) | N/A - replaced by SE source change below                       |                                                       |
| Chemistry SE (no gaussian tag)   | Matter Modeling SE - same tag gaussian, 215 questions          | gaussian tag exists and active on mattermodeling site |

---

## Second Scrape Results

All 5 sources working. 50 passages collected, up from 20 on the first run.

| Source | Status |
|---|---|
| Harvard RC | OK - 10 passages |
| TACC | OK - 10 passages |
| NSC Sweden | OK - 10 passages |
| Ohio Supercomputer Center | OK - 10 passages |
| Matter Modeling SE | OK - 10 passages |

Output saved to ~/.inkly/gaussian_docs.json.

---

## Growing the Dataset

### Raise MAX_PASSAGES_PER_SOURCE to 30

Changed `MAX_PASSAGES_PER_SOURCE` in `extractor.py` from 10 to 30. No test failures. Passages went from 50 to 126, words from 1,881 to 5,221.

### Add More Stack Exchange Tags

Added 4 new Matter Modeling SE tags to `sources.py`:
- `density-functional-theory` (1545 questions)
- `computational-chemistry` (388 questions)
- `quantum-chemistry` (322 questions)
- `high-performance-computing` (151 questions)

Passages went from 126 to 237, words to 25,422, file size to 161.7 KB.

### Add More HPC Doc Sites

Tested several candidate URLs. Two confirmed working with good content:
- Alliance Canada - skipped by scraper (likely JS-rendered, to revisit later)
- HPC Wiki - OK, 30 passages extracted

Final dataset after all changes:

| Metric | Value |
|---|---|
| Sources | 10 |
| Total passages | 267 |
| Total words | 26,651 |
| File size | 169.4 KB |

---

## Related Notes

- [[Work Session 5]] - previous session where scraper was optimized
