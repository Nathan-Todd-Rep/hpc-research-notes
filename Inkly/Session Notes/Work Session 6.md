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

## Related Notes

- [[Work Session 5]] - previous session where scraper was optimized
