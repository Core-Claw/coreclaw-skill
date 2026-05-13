# CoreClaw Agent Skill

[English](README.md) | [简体中文](README.zh-CN.md)

A CoreClaw workflow repository for AI agents. It is not an SDK. Its job is to help an agent use CoreClaw reliably across the full workflow: finding a scraper, reading the live schema, starting a run, tracking status, fetching results, exporting files, reading logs, rerunning jobs, and aborting runs.

## What this repository does

In one sentence:

**It helps an agent run the full CoreClaw workflow end to end: discover a scraper, run it correctly, retrieve results, and handle failures.**

Typical tasks it supports:

- finding the right scraper for a job
- reading a scraper's live input schema
- starting runs in sync or async mode
- tracking execution with `run_slug`
- reading paginated results or exporting CSV / JSON
- diagnosing failed runs with detail and logs
- rerunning, aborting, and reviewing runs and account information

## Repository contents

```text
.
├── CHANGELOG.md
├── LICENSE
├── README.md
├── README.zh-CN.md
├── REGRESSION.md
├── SKILL.md
├── SUPPORT.md
└── openapi.json
```

What these files are for:

- `README.md` / `README.zh-CN.md`: human-facing documentation and examples
- `SKILL.md`: workflow instructions for the agent
- `openapi.json`: API contract reference for the agent
- `REGRESSION.md`: maintainer smoke-check and release verification checklist
- `SUPPORT.md`: support boundaries and issue-routing guidance
- `CHANGELOG.md`: release history for documentation and contract changes
- `LICENSE`: repository license

## What kinds of jobs it is good for

This skill is useful for structured scraping workflows such as:

- ecommerce product lists, product detail pages, brands, prices, and reviews
- news, blog, and public webpage data collection
- maps, business listings, courses, and directory-style scraping
- large batch runs that need file export
- automatic failure diagnosis using run detail and logs

## How to use it

### 1. Put it into a skill-enabled agent environment

If your host supports a skills directory, place the whole repository there.

Example for Codex-style setups:

```bash
~/.codex/skills/coreclaw
```

Primary entry file:

```bash
SKILL.md
```

If your host does not support automatic skill loading, manually provide these two files to the agent:

- `SKILL.md`
- `openapi.json`

### 2. Prepare runtime requirements

You need:

- `curl`
- `jq`
- `CORECLAW_API_KEY`
- network access to `https://openapi.coreclaw.com`

Set the API key:

```bash
export CORECLAW_API_KEY="your_api_key"
```

### 3. Give the agent a task

These are good example prompts:

- "Find me a CoreClaw scraper for Amazon product listings."
- "Read this scraper's live schema and run it for me."
- "Start this scraper in async mode, save the run_slug, and keep polling until it finishes."
- "Show me the first 20 records, and export CSV if the result set is large."
- "Inspect why this run failed and extract the error logs."
- "Rerun this historical run and show me the new results."

## Recommended workflow

A typical CoreClaw agent workflow looks like this:

1. discover a suitable scraper
2. read the scraper live detail
3. build input from the live schema
4. start a sync or async run
5. track the run with `run_slug`
6. read results or export a file
7. inspect detail and logs if something fails

## How the live `custom` schema works

For every scraper, there are two different layers you need to keep separate:

- `data.parameters.custom` from `GET /api/scraper?slug=...` is the live schema descriptor
- `input.parameters.custom` in `POST /api/v1/scraper/run` is the actual payload you build from that descriptor

This distinction is critical:

- do not copy `data.parameters.custom` directly into `scraper/run`
- do not hardcode fields like `startURLs`, `keyword`, or `url`
- always fetch the current scraper detail first, then build the payload from the live schema

You can inspect this structure through the public `GET /api/scraper` endpoint. No API key or console login is required just to discover the live `custom` schema.

A safe mental model is:

1. discover a scraper with `/api/store`
2. fetch its live contract with `/api/scraper`
3. inspect `data.parameters.custom`
4. create `input.parameters.custom` that matches the live fields
5. send the run request

### Representative live schema patterns

The following public scraper contracts show why `custom` must stay dynamic.

#### Pattern A: flat keyword form

Representative scraper:

- `01KPAFF816MDMRD5MSCH2SBT68` — `Google Search By KeyWord`

Live schema characteristics:

- a flat object payload
- required field: `keyword`
- optional fields include `start`, `domain`, `gl`, `hl`, `location`, `safe`, and others

Minimal payload derived from the live schema:

```json
{
  "keyword": "openai"
}
```

#### Pattern B: array of request items with multiple fields

Representative scraper:

- `01KPD6M5YQADCQKGVKPDZVYC63` — `Google Map Details By Keyword`

Live schema characteristics:

- `custom.url` is an array
- each array item is a structured object
- required item fields include `base_location` and `keyword`
- optional item fields include `lang`, `max_results`, `fetch_reviews`, and `max_reviews_per_place`

Minimal payload derived from the live schema:

```json
{
  "url": [
    {
      "base_location": "Singapore",
      "keyword": "coffee shop"
    }
  ]
}
```

#### Pattern C: array of detail URLs

Representative scraper:

- `01KPD6M5YVHWCNQCRK32BD02TP` — `Google Map Detail By Detail URL`

Live schema characteristics:

- `custom.url` is also an array
- each item represents one place detail request
- required item field: `detail_url`
- optional item fields include `lang` and `max_reviews_per_place`

Minimal payload derived from the live schema:

```json
{
  "url": [
    {
      "detail_url": "https://www.google.com/maps/place/..."
    }
  ]
}
```

The important point is not the field names above. The important point is that these names came from live `/api/scraper` responses, and a different scraper can expose a different shape.

## Real usage examples

These are practical examples of how this skill is meant to be used.

### Example 1: Amazon product list scraping

Goal: search Amazon by keyword and get a structured product list.

Prompt:

```text
Find a CoreClaw scraper for Amazon product listings, read its live schema, run it with the keyword "wireless mouse", and show me the first 20 results.
```

Typical outputs:

- product title
- price
- brand
- rating
- review count
- product URL
- image URL

### Example 2: Amazon product detail scraping

Goal: input a product URL and get structured product detail data.

Prompt:

```text
Find a CoreClaw scraper for Amazon product detail pages by URL, read the live schema, run it with this product URL, and export the result as CSV.
```

Typical outputs:

- title
- seller
- brand
- description
- price
- category
- ASIN
- images
- rating
- detailed product fields

### Example 3: Async batch scraping with status tracking

Goal: launch a scraper asynchronously and keep tracking it until completion.

```text
Start this scraper in async mode, save the run_slug, poll every 10 seconds, and after completion tell me the result count before deciding whether to display or export.
```

Good for:

- larger datasets
- background execution
- automated downstream workflows

### Example 4: Export when the dataset is large

Goal: export a file when the result set is too large for inline display.

```text
If this run produces more than 50 records, do not paste everything inline. Export it as CSV and return the download link.
```

Good for:

- product catalogs
- listing directories
- article collections
- bulk structured data

### Example 5: Automatic failure diagnosis

Goal: do not stop at "the run failed"; collect useful debugging evidence automatically.

```text
If this run fails, check run/detail first, then run/last/log, and summarize the real reason.
```

Good for:

- bad input payloads
- worker-side execution errors
- page scraping issues
- timeouts or runtime environment issues

### Example 6: Rerun a historical task

Goal: replay the same run parameters from a previous run.

```text
Rerun this run_slug, track the new run, and after completion show me the first 10 results.
```

Good for:

- repeating the same collection job
- retrying after an issue is fixed
- scheduled or periodic re-collection

## Usage examples

### Example 7: Scrape a Google Maps business list

Goal: search businesses by keyword and location.

Prompt:

```text
Find a CoreClaw scraper for Google Maps business search, read its live schema, and collect business name, rating, address, phone number, and website for the keyword "coffee shop" in "Singapore".
```

Typical outputs:

- business name
- rating
- review count
- address
- phone number
- website
- map URL

### Example 8: Scrape a Google Maps detail page

Goal: input a place detail URL and get the structured place profile.

Prompt:

```text
Find a CoreClaw scraper for Google Maps detail URLs, read the live schema, run it on one place URL, and return the place name, address, categories, review rating, and review count.
```

Typical outputs:

- place title
- address
- categories
- review rating
- review count
- website
- photos

### Example 9: Collect news pages for monitoring

Goal: scrape a news listing and capture recent article metadata.

Prompt:

```text
Find a CoreClaw scraper for news listing pages, read the live schema, scrape the latest page from this section, and return title, publish time, category, and article URL.
```

Good for:

- media monitoring
- industry tracking
- content archiving
- periodic news summaries

### Example 10: Collect company directory data

Goal: gather company names, industries, and contact information from a directory site.

Prompt:

```text
Find a CoreClaw scraper for company directories and collect company name, industry, location, website, and contact information from this listing page.
```

Good for:

- sales lead collection
- industry directory building
- B2B data archiving

### Example 11: Preview first, then decide whether to export

Goal: inspect a small sample before choosing the delivery format.

Prompt:

```text
Read the first 20 records from this run. If the dataset is large, export a file; otherwise show the results inline.
```

Good for:

- quick quality checks
- field validation
- keeping conversation output compact

### Example 12: Reshape results for business use

Goal: reorganize scraped output into business-friendly fields.

Prompt:

```text
Reformat this result into columns for title, price, rating, and link. If a field is missing, keep the original value and do not drop the record.
```

Good for:

- procurement lists
- competitor tables
- business intelligence archives
- content operations handoff

## Quick start example

If you want the shortest possible path to trying the skill, give the agent these three steps:

### Step 1: find a scraper

```text
Find me a CoreClaw scraper for Amazon product listings.
```

### Step 2: read live parameters and run it

```text
Read the scraper's live schema and run it with the minimum required input.
```

### Step 3: fetch the result

```text
After completion, show me the first 20 records. If the dataset is large, export CSV.
```

## What kind of agent works best with this repository

This repository is especially useful when the agent can:

- read local files
- understand API structure from `openapi.json`
- execute HTTP requests
- store and reuse `run_slug`
- decide between inline reading and export
- inspect detail and logs when a run fails

## Practical usage tips

- read the live schema before building input
- use `result/list` for smaller inline work
- use `result/export` for larger deliveries
- keep the `run_slug` for all follow-up operations
- when something fails, inspect both detail and logs
- reuse the same scraper workflow for repeated jobs of the same type

## Who this repository is for

Good fit for:

- builders of AI agent products
- developers integrating CoreClaw into automation workflows
- AI applications that need structured scraping capability
- teams that want an agent, not a human operator, to manage scraper runs

## Support and release notes

For public usage and team maintenance, also see:

- `SUPPORT.md` for issue routing and support boundaries
- `REGRESSION.md` for maintainer smoke checks before release
- `CHANGELOG.md` for documentation and contract changes
- `LICENSE` for repository licensing terms

## References

- CoreClaw Store: https://coreclaw.com/store
- Skill entry point: `SKILL.md`
- OpenAPI contract: `openapi.json`