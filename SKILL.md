---
name: coreclaw
description: Reusable CoreClaw workflow for AI agents. Use it to discover a scraper, read the live input schema, run jobs in sync or async mode, poll status, fetch results, export data, inspect logs, rerun jobs, or debug failed runs.
homepage: https://coreclaw.com
metadata:
  openclaw:
    primaryEnv: CORECLAW_API_KEY
    requires:
      bins:
        - curl
        - jq
      env:
        - CORECLAW_API_KEY
---

# CoreClaw

Run scrapers from [CoreClaw Store](https://coreclaw.com/store) and retrieve structured results through the CoreClaw REST API.

Canonical API reference: [openapi.json](openapi.json)
Chinese repository guide: [README.zh-CN.md](README.zh-CN.md)

## When this skill should trigger

Use this skill when the user wants to:

- search CoreClaw Store for a suitable scraper
- inspect a scraper's live parameters or README
- start a scraper run in sync or async mode
- monitor an existing `run_slug`
- fetch paginated results for analysis
- export results as CSV or JSON
- inspect logs, rerun a job, abort a run, or review history
- debug why a CoreClaw run failed

## Authentication and base URL

Base URL: `https://openapi.coreclaw.com`

Most endpoints require the `CORECLAW_API_KEY` env var as the `api-key` header:

```bash
-H "api-key: $CORECLAW_API_KEY"
```

Public endpoints that do not require an API key:

- `GET /api/store`
- `GET /api/scraper`

## Runtime precedence

If runtime behavior conflicts with older examples or repository text, trust current runtime behavior first.

Priority order:

1. live runtime behavior
2. current platform docs
3. this repository's `openapi.json`
4. examples in `SKILL.md` and `README`

## Verified behavior snapshot

Verified during the 2026-05 regression pass:

- `scraper/run` accepts `is_async=false` without `callback_url`
- `scraper/run` also accepted `is_async=true` without `callback_url`
- if webhook orchestration is needed, `callback_url` should still be passed explicitly
- `rerun` rejected missing `callback_url` with `4000 Invalid request parameters`
- `abort` has a documented request/response contract, but successful abort conditions still need more runtime evidence
- different scrapers expose different `custom` schemas and must be treated dynamically

## Hard rules

1. Always read `GET /api/scraper?slug=...` before constructing a `scraper/run` request.
2. Never guess `version`; always use `data.version` from `/api/scraper`.
3. Never hardcode the shape of `input.parameters.custom`; always derive it from `data.parameters.custom` returned by `/api/scraper`.
4. Treat `input.parameters.system` as the stable platform layer, and `input.parameters.custom` as the scraper-specific layer.
5. Prefer `result/list` for small inline analysis and `result/export` for large datasets or file delivery.
6. When runtime and older examples disagree, explain the difference and follow runtime.

## Schema handling rules

1. Treat `data.parameters.custom` from `GET /api/scraper?slug=...` as the live schema descriptor.
2. Treat `input.parameters.custom` in `scraper/run` as the actual request payload derived from that descriptor.
3. Never hardcode scraper-specific field names such as `startURLs`, `keyword`, `url`, or nested item keys unless the live descriptor returned them for the current scraper.
4. When `custom` contains arrays, nested objects, or repeated item schemas, map the live shape first and only then fill the minimal required values.
5. If the live schema is unclear, stop and re-read `/api/scraper` instead of guessing.

## Maintenance note

Before publishing or revising any example in `README.md`, `README.zh-CN.md`, or `SKILL.md`, verify the example against a live scraper detail response and keep the example tied to the observed schema shape. If a new scraper family is added, record the observed pattern in `REGRESSION.md` so future updates stay reproducible.

## Quick decision guide

- Need a scraper candidate -> use `/api/store`
- Need the real request shape -> use `/api/scraper`
- Need small inline analysis -> use `result/list`
- Need a file or large dataset -> use `result/export`
- Need failure reason -> use `run/detail` first, then `run/last/log`
- Need async webhook orchestration -> pass `callback_url`
- Need a rerun -> pass `callback_url`

## Core workflow

### 1. Find the right scraper

Search the CoreClaw Store by keyword:

```bash
curl -s "https://openapi.coreclaw.com/api/store?search=amazon&limit=5" | jq '.data.scraper[] | {slug, title, description}'
```

What to extract:

- `slug`
- `title`
- `description`

### 2. Read the live scraper contract

Before any run, read the scraper detail:

```bash
curl -s "https://openapi.coreclaw.com/api/scraper?slug=SCRAPER_SLUG" | jq '.data | {version, parameters, readme}'
```

Use the response as the live contract source:

- `data.version` -> required in `scraper/run`
- `data.parameters.system` -> default system values
- `data.parameters.custom` -> live schema for scraper-specific input fields
- `data.readme` -> human-readable scraper instructions

Important mapping rule:

- `/api/scraper` may expose `memory_bytes`
- `/api/v1/scraper/run` uses `memory`
- these refer to the same MB value

### 3. Choose run mode

#### Option A: synchronous run

Use sync mode when the user wants a simple request/response flow and is willing to wait.

- set `is_async` to `false`
- `callback_url` is optional
- use the smallest valid live `custom` payload first when validating a new scraper

```bash
curl -s -X POST "https://openapi.coreclaw.com/api/v1/scraper/run" \
  -H "api-key: $CORECLAW_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "scraper_slug": "SCRAPER_SLUG",
    "version": "VERSION_FROM_DETAIL",
    "is_async": false,
    "input": {
      "parameters": {
        "system": {
          "cpus": 0.125,
          "memory": 512,
          "execute_limit_time_seconds": 1800,
          "max_total_charge": 0,
          "max_total_traffic": 0
        },
        "custom": {
          "START_WITH_THE_MINIMAL_VALID_LIVE_PAYLOAD": true
        }
      }
    }
  }'
```

#### Option B: asynchronous run

Use async mode when the user wants background execution, status polling, or webhook-based orchestration.

- set `is_async` to `true`
- if webhook orchestration is needed, pass `callback_url`
- current runtime accepted missing `callback_url` during the 2026-05 verification pass

```bash
curl -s -X POST "https://openapi.coreclaw.com/api/v1/scraper/run" \
  -H "api-key: $CORECLAW_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "scraper_slug": "SCRAPER_SLUG",
    "version": "VERSION_FROM_DETAIL",
    "is_async": true,
    "input": {
      "parameters": {
        "system": {
          "cpus": 0.125,
          "memory": 512,
          "execute_limit_time_seconds": 1800,
          "max_total_charge": 0,
          "max_total_traffic": 0,
          "proxy_region": "US"
        },
        "custom": {
          "START_WITH_THE_MINIMAL_VALID_LIVE_PAYLOAD": true
        }
      }
    },
    "callback_url": "https://your-callback.example.com/webhook"
  }'
```

Response returns `data.run_slug`. Save it for all later operations.

### 4. Poll run status safely

Use `POST /api/v1/run/detail` to monitor progress:

```bash
CONSECUTIVE_ERRORS=0
while true; do
  RESP=$(curl -s -X POST "https://openapi.coreclaw.com/api/v1/run/detail" \
    -H "api-key: $CORECLAW_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{"run_slug": "RUN_SLUG"}')

  RESP_CLEAN=$(echo "$RESP" | tr -d '\000-\011\013-\037')
  STATUS=$(echo "$RESP_CLEAN" | jq -r '.data.status // empty' 2>/dev/null)

  if [ -z "$STATUS" ]; then
    CONSECUTIVE_ERRORS=$((CONSECUTIVE_ERRORS + 1))
    if [ "$CONSECUTIVE_ERRORS" -ge 5 ]; then
      echo "ERROR: 5 consecutive failures"
      echo "$RESP" | head -c 500
      break
    fi
    sleep 10
    continue
  fi

  CONSECUTIVE_ERRORS=0
  echo "Status: $STATUS"

  case "$STATUS" in
    3) echo "$RESP_CLEAN" | jq '.data | {status, results, duration, usage}'; break ;;
    4) echo "$RESP_CLEAN" | jq '.data | {status, err_msg, duration, usage}'; break ;;
  esac

  sleep 10
done
```

Status codes:

- `1` ready
- `2` running
- `3` succeeded
- `4` failed
- `5` aborting

### 5. Retrieve results

#### Inline analysis: `result/list`

Use this for small datasets and when the agent needs to inspect data directly.

```bash
curl -s -X POST "https://openapi.coreclaw.com/api/v1/run/result/list" \
  -H "api-key: $CORECLAW_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"run_slug": "RUN_SLUG", "page_index": 1, "page_size": 20}'
```

What to read:

- `data.count`
- `data.headers[]`
- `data.list[]`

The fields inside `data.list[]` are not fixed. They depend on the Worker output schema.

#### File delivery: `result/export`

Use this for large datasets or when the user wants a file.

```bash
curl -s -X POST "https://openapi.coreclaw.com/api/v1/run/result/export" \
  -H "api-key: $CORECLAW_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"run_slug": "RUN_SLUG", "format": "csv", "filter_keys": []}'
```

The response returns a temporary `download_url`, not raw records.

### 6. Saved tasks and reruns

#### Run a saved task

`POST /api/v1/task/run` launches a saved Task template.

- `task_slug` required
- `callback_url` required by documented contract

```bash
curl -s -X POST "https://openapi.coreclaw.com/api/v1/task/run" \
  -H "api-key: $CORECLAW_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"task_slug": "TASK_SLUG", "callback_url": "https://your-callback.example.com/webhook"}'
```

#### Re-run an existing run

`POST /api/v1/rerun` starts a new async run using the same parameters as a previous run.

- `run_slug` required
- `callback_url` required and verified by runtime behavior

```bash
curl -s -X POST "https://openapi.coreclaw.com/api/v1/rerun" \
  -H "api-key: $CORECLAW_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"run_slug": "PREVIOUS_RUN_SLUG", "callback_url": "https://your-callback.example.com/webhook"}'
```

### 7. Debugging tools

#### View logs

```bash
curl -s -X POST "https://openapi.coreclaw.com/api/v1/run/last/log" \
  -H "api-key: $CORECLAW_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"run_slug": "RUN_SLUG"}' | jq '.data.list[] | {type, group, content, timestamp}'
```

#### Abort a running task

`POST /api/v1/scraper/abort` aborts a running Worker task.

- send `run_slug` in the JSON body
- success response is `{"code":0,"message":"success","data":null}`
- if abort fails, capture both `run/detail` before/after and the raw abort response

```bash
curl -s -X POST "https://openapi.coreclaw.com/api/v1/scraper/abort" \
  -H "api-key: $CORECLAW_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"run_slug": "RUN_SLUG"}'
```

#### Check account info

```bash
curl -s -X POST "https://openapi.coreclaw.com/api/v1/account/info" \
  -H "api-key: $CORECLAW_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{}' | jq '.data | {balance, traffic, traffic_expiration_at}'
```

## Decision guide

### When to use `result/list`

Use it when:

- the user wants a quick preview
- the dataset is small enough to fit in context
- the agent needs to summarize or transform records inline

### When to use `result/export`

Use it when:

- the user asks for a file
- the dataset is large
- a clean CSV or JSON artifact is needed outside the chat context

## Common failure patterns

### `4000 Invalid request parameters`

Usually means one of these:

- `version` was guessed instead of read from `/api/scraper`
- `input.parameters.custom` does not match the live scraper schema
- sync/async fields are inconsistent
- `rerun` or `task/run` was called without `callback_url`
- `abort` was attempted when the platform did not accept the current task state

### Empty or confusing results

Check:

- whether the run actually succeeded
- whether the user only needs page 1 or all pages
- whether export is more appropriate than inline listing

### Polling loops forever

Check:

- parsing errors caused by control characters
- repeated non-JSON responses
- whether terminal states `3` or `4` are being handled correctly

## Verified coverage and known gaps

Verified by runtime:

- store discovery
- scraper detail and live schema reads
- async run with callback
- async run without callback
- run detail
- run list
- result list
- result export
- run logs
- rerun with callback
- account info

Known gaps:

- sync mode still needs a single stable minimal success example that is easy to replay
- abort needs a stronger positive example under a platform state that definitely supports cancellation
- `task/run` is still documented here according to platform docs but not independently verified in this repo's regression pass

## Minimal acceptance checklist

A correct agent using this skill should consistently do the following:

1. discover a scraper with `/api/store` or accept a provided slug
2. fetch `/api/scraper` before calling `scraper/run`
3. copy `version` from the detail response
4. derive `custom` from the live schema instead of guessing field names
5. choose sync vs async correctly
6. decide whether to send `callback_url` in async mode based on runtime behavior and webhook needs
7. use `run_slug` for follow-up operations
8. choose `result/list` vs `result/export` according to dataset size and user intent
9. use logs or `run/detail` for debugging instead of hallucinating causes

## Additional resources

- OpenAPI spec: [openapi.json](openapi.json)
- CoreClaw Store: https://coreclaw.com/store