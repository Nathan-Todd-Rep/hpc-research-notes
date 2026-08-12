# Scraper Redesign

**Date Started:** 2026-07-03
**Goal:** Redesign the Gaussian docs scraper into a generalised, domain-agnostic scraping tool. A user should be able to define a domain, keywords, and sources, and the scraper handles the rest. Today it scrapes Gaussian. Tomorrow it could scrape bioinformatics, materials science, or any other HPC research domain.

---

## Motivation

The scraper currently has Gaussian hardcoded into every layer:
- `sources.py` contains hardcoded URLs and keywords for Gaussian only
- `extractor.py` references `GAUSSIAN_KEYWORDS` directly
- Output is always written to `~/.inkly/gaussian_docs.json`

The end vision is a tool where a user selects parameters, not a tool that only knows how to scrape one domain.

---

## Proposed Architecture

### 1. ScraperConfig

A dataclass that holds everything needed to run a scrape for any domain:

```python
@dataclass
class ScraperConfig:
    name: str                        # e.g. "gaussian", "bioinformatics"
    keywords: list[str]              # keyword filter list
    html_sources: list[dict]         # list of {label, url}
    se_sources: list[dict]           # list of {site, tag, label}
    output_path: Path                # defaults to ~/.inkly/{name}_docs.json
```

### 2. Config Input Methods

Three ways to define a scrape config:

| Method           | Example                                                                                | Use Case                               |
| ---------------- | -------------------------------------------------------------------------------------- | -------------------------------------- |
| TOML file        | `scrape.py --config gaussian.toml`                                                     | Repeatable, version-controlled configs |
| CLI flags        | `scrape.py --name gaussian --keywords "gaussian,g16" --se-tag mattermodeling:gaussian` | Quick one-off runs                     |
| Interactive mode | `scrape.py` with no args                                                               | New users, exploratory scraping        |

### 3. Module Restructure

- `sources.py` -- no longer hardcodes Gaussian data. Replaced by the config system.
- `configs/gaussian.toml` -- Gaussian config migrated here, ships as a built-in example
- `configs/bioinformatics.toml` -- second example config for bioinformatics domain
- `scrape.py` -- updated to load a `ScraperConfig` and run the pipeline against it

### 4. Output

Each domain writes to its own file:

```
~/.inkly/gaussian_docs.json
~/.inkly/bioinformatics_docs.json
```

Inkly loads whichever file matches the active domain at runtime.

---

## Decisions

- [x] **Primary interface: interactive mode.** Running `scrape.py` with no arguments launches a setup wizard. Simple questions, no config file required. Saves a TOML file at the end so the user can reuse or share their config.
- [x] **TOML as power-user option.** `scrape.py --config gaussian.toml` skips the wizard entirely for repeatable runs.
- [x] **sources.py kept as fallback.** Gaussian sources stay hardcoded as a safety net during the transition. Can be deprecated later once the config system is proven.
- [x] **SE tag auto-discovery.** During interactive mode, the scraper queries the SE API for the user's topic and presents matching tags to choose from. No SE knowledge required from the user.

---

## Bioinformatics Stack Exchange - Available Tags

Checked 2026-07-07. Top tags by question count:

| Tag | Count |
|---|---|
| rna-seq | 717 |
| ngs | 299 |
| scrnaseq | 293 |
| vcf | 261 |
| fasta | 249 |
| genome | 217 |
| genomics | 159 |
| samtools | 154 |
| assembly | 153 |
| fastq | 144 |

---

## Interactive Wizard Design

Preset-based flow to make the wizard accessible to non-technical users:

```
What would you like to scrape?
  1. Gaussian (computational chemistry)
  2. Bioinformatics
  3. Custom topic

Select an option [1-3]: >
```

Presets pre-populate keywords AND html_sources (not just keywords) for accuracy --
curated, validated sources reduce the risk of pulling in low-quality or off-topic
passages that could lead to inaccurate summaries downstream.

Topic input is auto-sanitized (lowercased, spaces -> underscores) so users never
need to know the "correct" naming style.

If a user ends up with zero sources selected (no tags, no URLs), the wizard loops
and asks again rather than producing an invalid config.

Only 2 presets for now (Gaussian, Bioinformatics) -- more can be added later.

---

## Bioinformatics Preset - Validated Sources

Validated 2026-07-05, following the same process used for Gaussian sources.
All URLs confirmed HTTP 200 with strong keyword density.

### HTML Sources

| Source | URL | Keyword hits |
|---|---|---|
| OSC - BWA | osc.edu/resources/available_software/software_list/bwa | 30 |
| OSC - Samtools | osc.edu/resources/available_software/software_list/samtools | 31 |
| OSC - GATK | osc.edu/resources/available_software/software_list/gatk | 43 |
| Alliance Canada - Bioinformatics | docs.alliancecan.ca/wiki/Bioinformatics | 28 |
| NIH HPC - Samtools | hpc.nih.gov/apps/samtools.html | 64 |
| NIH HPC - BWA | hpc.nih.gov/apps/bwa.html | 31 |

### Stack Exchange Tags (site: bioinformatics)

Skipped `r` and `python` tags -- programming languages, not bioinformatics topics,
would pull in generic coding Q&A unrelated to HPC work.

| Tag | Count |
|---|---|
| rna-seq | 717 |
| ngs | 299 |
| vcf | 261 |
| genome | 217 |
| samtools | 154 |

### Keywords

`rna-seq, fastq, samtools, bwa, gatk, genome, variant-calling, ngs, vcf, alignment, bam, blast`

### End-to-End Validation

Ran `py scrape.py --config configs/bioinformatics.toml` against the real APIs/pages
to confirm the full pipeline works, not just the preset data structure.

Results: 9/10 sources succeeded, 246 passages collected, saved correctly to
`~/.inkly/bioinformatics_docs.json` (domain-specific output path confirmed working).

**Alliance Canada failed to extract passages again** -- same as the Gaussian wiki
page earlier. Confirms this is a fetcher limitation (likely JS-rendered content
BeautifulSoup can't see) rather than a one-off. Not blocking; noted for later.

---

## scrape.py CLI

Rewritten to dispatch on flags instead of hardcoding Gaussian:

```
py scrape.py                        # launches the interactive wizard
py scrape.py --config gaussian.toml # runs a saved config, no prompts
py scrape.py --legacy               # uses sources.py directly (fallback safety net)
```

`scrape_html_sources()` and `scrape_se_sources()` now take `html_sources`/`se_sources`
and `keywords` as parameters instead of importing hardcoded module-level constants.
`save_results()` takes an explicit `output_path`. `--legacy` mode builds a
`ScraperConfig` from the original `sources.py` constants so old behavior is
preserved byte-for-byte as a fallback.

Verified `--legacy` reproduces the exact same 267-passage, 10-source result as
before the redesign.

---

## Progress

- [x] Architecture planned
- [x] Open questions resolved
- [x] ScraperConfig schema implemented
- [x] TOML config loader implemented
- [x] SE tag auto-discovery implemented
- [x] Bioinformatics preset sources validated
- [x] Interactive wizard implemented (presets.py + wizard.py)
- [x] TOML config save implemented (save_toml_config, wired into wizard end-of-flow)
- [x] scrape.py updated to use config system (--config, --legacy, wizard fallback)
- [x] bioinformatics.toml created and validated end-to-end
- [x] gaussian.toml created and validated end-to-end (10/10 sources, 267 passages, matches --legacy exactly)
- [ ] CLI interface polish (argparse currently minimal -- may want --help examples etc.)
- [ ] sources.py refactored or removed (currently kept as --legacy fallback per decision)
- [x] Tests updated (78/78 passing; scrape.py itself not unit tested, consistent with prior convention of testing pipeline modules directly)

---

## Related Notes

- [[Work Session 6]] - session where dataset growth was started and redesign was motivated
- [[Work Session 5]] - scraper optimisation session
