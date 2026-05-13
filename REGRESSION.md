# Regression checklist

Use this file as a maintainer smoke-check before publishing changes.

## Core checks

- read the target scraper through `GET /api/scraper?slug=...`
- confirm `data.version` matches the run request
- confirm `data.parameters.custom` is used as the live schema descriptor
- confirm the README example matches the live scraper shape
- confirm `SKILL.md` hard rules still point to the live schema first

## Representative contract checks

- one keyword-based scraper
- one array-based scraper
- one nested-object or repeated-item scraper

## Runtime checks

- sync run without `callback_url`
- async run without `callback_url`
- async run with `callback_url` when webhook orchestration is needed
- result read via `result/list`
- file read via `result/export`
- log inspection via `run/last/log`
- rerun with `callback_url`
- abort request and response contract

## Publish checks

- README.md and README.zh-CN.md describe the same capabilities
- README files are user-facing only
- maintenance notes stay in maintainer docs
- no fixed `custom` field names are presented as universal
- license and support files exist