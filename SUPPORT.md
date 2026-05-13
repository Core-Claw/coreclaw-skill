# Support

This repository is documentation and workflow guidance for CoreClaw agents.

## What this repository supports

- discovering scrapers
- reading live scraper schemas
- running jobs
- polling status
- reading results
- exporting files
- inspecting logs
- rerunning jobs
- aborting running tasks

## What this repository does not support

- CoreClaw account recovery
- API key issuance or rotation
- custom scraper development
- platform outage response
- guaranteed scraper-specific data quality

## When reporting a problem

Include:

- the scraper slug
- the run slug, if one exists
- the exact request payload you used
- the endpoint you called
- the raw error response or log snippet

## Where to look first

- `README.md` or `README.zh-CN.md` for usage
- `SKILL.md` for agent workflow rules
- `openapi.json` for API contract details
- `REGRESSION.md` for release verification checks