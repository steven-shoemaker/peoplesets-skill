---
name: peoplesets
description: Generate sales-demo-grade synthetic HR datasets via the peoplesets API. Use when the user asks for fake HR data, synthetic employees, a demo dataset for people analytics, a workforce dataset for testing, a Workday/BambooHR-shaped sample, an org-chart sandbox, or wants to back a people-analytics demo with realistic-but-fake employees, terminations, promotions, and comp data. Drives the hosted peoplesets service (default `https://www.peoplesets.com`) — translates natural-language briefs into endpoint calls, polls the job, downloads the four parquets, and summarizes shape for the user.
version: 0.1.0
---

# peoplesets

You are translating natural-language briefs about synthetic HR datasets into
calls against the hosted **peoplesets** API. The user wants a dataset that
looks like Workday or BambooHR exports — four tables (employees, events,
comp_events, recruiting) — for sales demos, prototypes, or testing.

## When to use this skill

Triggers include any of:

- "Make me a synthetic HR dataset…"
- "Generate fake employee data for X"
- "Demo dataset for people analytics"
- "Sample workforce data, ~N employees in industry Y"
- "Workday-shaped data for a [tech / retail / healthcare / fintech] company"
- "Test data for an HRIS pipeline"
- "Fake comp data with realistic pay bands"
- "Org chart sandbox"
- "Synthetic employees with [special event / scenario]"

If the user asks for **real** HR data, this skill does not apply.

## Setup

The hosted base URL is **`https://www.peoplesets.com`**. Use that literal
string in every curl. Do not template `$PEOPLESETS_URL` into commands —
when unset it produces `https:///generate-company` and the call fails.
Only use the env var if the user explicitly set one for a self-hosted
instance.

The API key comes from `$PEOPLESETS_API_KEY`. Check it first:

```bash
if [ -z "$PEOPLESETS_API_KEY" ]; then
  echo "PEOPLESETS_API_KEY not set"
fi
```

If unset, stop and tell the user verbatim:

> No API key found. Grab a free one at https://www.peoplesets.com/#get-key
> (30 seconds, no card). Then:
> `export PEOPLESETS_API_KEY=psk_…`
> and ask me again.

Never retry without a key. Every call sends
`Authorization: Bearer $PEOPLESETS_API_KEY`.

## Endpoint catalog

```
GET  https://www.peoplesets.com/industry-packs                    → list curated packs
GET  https://www.peoplesets.com/scenarios                         → list special events
POST https://www.peoplesets.com/generate-company                  → start a sim job
POST https://www.peoplesets.com/apply-scenario                    → pack + 1 scenario
GET  https://www.peoplesets.com/jobs/{job_id}                     → status + meta
GET  https://www.peoplesets.com/jobs/{job_id}/artifacts.zip       → 4 parquets + reports
```

The richest one is `POST /generate-company`. Default to it for most briefs.

## Intent → parameter translation

Build a `POST /generate-company` body from these slots, in priority order:

1. **`industry_pack`** — if the brief names an industry that matches an
   installed pack (`tech_startup`, `retail_chain`, `healthcare_system`,
   plus whatever `GET /industry-packs` returns), use that as the base.

2. **`size`** — **starting** headcount the engine spins up before the
   sim window opens. NOT the final active headcount. Growth hiring over
   the sim window typically pushes the final active count to 2–3× this
   for a 3-year run. When the user says "make me a 1,200-person
   company," they usually mean *final active* — pick `size` accordingly:

   | User says                                | starting `size` | with `simulation_years` |
   | ---------------------------------------- | --------------- | ----------------------- |
   | "tiny startup, ~25"                      | 25              | 1.0                     |
   | "early startup, ~50"                     | 50              | 1.0                     |
   | "Series A ~80 today"                     | 50              | 2.0                     |
   | "Series B, ~250"                         | 150             | 2.0                     |
   | "~1,200-person company"                  | 500             | 3.0                     |
   | "Series C, ~2k"                          | 1,000           | 3.0                     |
   | "mid-market, ~5k"                        | 2,500           | 3.0                     |
   | "enterprise, ~10k"                       | 5,000           | 3.0                     |

   Shortcut: for a target final headcount `N` over 3 years, start at
   `~N/2.5`. Or set `simulation_years=1.0` and use `size=N` directly.

3. **`simulation_years`** — defaults to 3.0 if unspecified.
   Bump to 5.0 if the brief says "5-year history" / "since 2020" / similar.

4. **`country_mix`** — ISO3 → relative weight. Map intent:

   | Phrase                                     | country_mix                                       |
   | ------------------------------------------ | ------------------------------------------------- |
   | "US-only"                                  | `{"USA": 1.0}`                                    |
   | "US + Canada"                              | `{"USA": 0.85, "CAN": 0.15}`                      |
   | "US-headquartered with major India hub"    | `{"USA": 0.55, "IND": 0.35, "GBR": 0.10}`         |
   | "Europe-heavy"                             | `{"DEU": 0.35, "GBR": 0.25, "IRL": 0.20, "USA": 0.20}` |
   | "global"                                   | leave unset (defaults are a mild global mix)      |

   Supported codes: `USA, GBR, CAN, IND, DEU, AUS, BRA, IRL`. Don't invent
   others — the engine will sample only from defaults if the override is
   empty.

5. **`special_events`** — list of scenario names from `GET /scenarios`.
   Common phrasings map to:

   | Phrase                                          | special_events             |
   | ----------------------------------------------- | -------------------------- |
   | "did a RIF" / "layoff last Q3" / "headcount cut"| `["rif"]`                  |
   | "hyper-growth" / "doubled last year" / "Series C burst" | `["hyper_growth"]` |
   | "retention crisis" / "distressed" / "talent exodus" | `["distressed"]`       |
   | "pay equity gap" / "encoded gender pay gap"     | `["pay_gap"]` (set `pay_gap_pct` for size, default 0.05) |

   M&A and Leadership shake-up scenarios are not yet implemented in the
   engine — don't suggest them. If a user asks for "post-merger" or
   "new CEO cleaned house" narratives, tell them the closest current
   approximation is `distressed` or `rif` and the dedicated scenarios
   are on the roadmap.

6. **`seed`** — pass `42` by default so the user can reproduce. If they say
   "give me something different," pick another non-negative int.

7. **`company_name`** — only if explicit. Otherwise leave the engine default.

If the brief is vague on a dimension, leave it unset rather than guess.

## Worked examples

### Example 1 — fintech mid-market, recent RIF

User: *"Make me a 1,200-person fintech dataset. They did a RIF last year — I want it to show in the data."*

`size=500` + `simulation_years=3.0` lands at ~1,200 active by sim end:

```bash
curl -X POST "https://www.peoplesets.com/generate-company" \
  -H "Authorization: Bearer $PEOPLESETS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 500,
    "simulation_years": 3.0,
    "country_mix": {"USA": 0.85, "GBR": 0.10, "IRL": 0.05},
    "special_events": ["rif"],
    "seed": 42
  }'
```

Then poll `/jobs/{job_id}` until `status: "done"`, then GET the zip.

### Example 2 — hospital network, low attrition

User: *"3,000-employee hospital system, realistic clinical/admin split, low turnover."*

`size=1200` + `simulation_years=3.0` lands near 3,000 active:

```bash
curl -X POST "https://www.peoplesets.com/generate-company" \
  -H "Authorization: Bearer $PEOPLESETS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "industry_pack": "healthcare_system",
    "size": 1200,
    "simulation_years": 3.0,
    "seed": 42
  }'
```

### Example 3 — startup with India engineering hub

User: *"Series-A tech startup, ~80 employees, US-headquartered with most engineers in Bangalore."*

A small org over a short window doesn't compound much — `size=50` +
`simulation_years=1.5` lands near 80:

```bash
curl -X POST "https://www.peoplesets.com/generate-company" \
  -H "Authorization: Bearer $PEOPLESETS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "industry_pack": "tech_startup",
    "size": 50,
    "simulation_years": 1.5,
    "country_mix": {"USA": 0.40, "IND": 0.60},
    "seed": 42
  }'
```

### Example 4 — list-then-pick when the user is exploring

User: *"What kinds of fake HR datasets can you build?"*

1. `GET https://www.peoplesets.com/industry-packs` — show the user the names + titles.
2. `GET https://www.peoplesets.com/scenarios` — show the user the scenarios.
3. Wait for them to pick one before submitting a job.

### Example 5 — applying a single scenario to a pack

User: *"Take the retail_chain pack and apply the distressed scenario."*

```bash
curl -X POST "https://www.peoplesets.com/apply-scenario" \
  -H "Authorization: Bearer $PEOPLESETS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "scenario": "distressed",
    "industry_pack": "retail_chain",
    "seed": 42
  }'
```

## Polling

Jobs run asynchronously. Poll every 1–3 seconds:

```bash
curl -s -H "Authorization: Bearer $PEOPLESETS_API_KEY" \
  "https://www.peoplesets.com/jobs/$JOB_ID"
```

Statuses: `pending` → `running` → `done` (or `error`). A typical 500-person
job finishes in 1–3 seconds; a 10,000-person job in 20–40 seconds. Time out
at 120 s and report the failure to the user.

## Downloading

```bash
curl -L -o "peoplesets-$JOB_ID.zip" \
  -H "Authorization: Bearer $PEOPLESETS_API_KEY" \
  "https://www.peoplesets.com/jobs/$JOB_ID/artifacts.zip"
```

The zip contains:

```
employees.parquet      one row per employee (active + terminated)
events.parquet         hire, termination, promotion, perf_review, merit,
                       reorg, layoff, hiring_freeze, leave_start, leave_end
comp_events.parquet    every salary change with old/new amounts
recruiting.parquet     pipeline funnel per hire
summary.md             human-readable rollup
data_dictionary.md     schema reference
charts/*.png           if analytics=true (default)
```

## Summarizing for the user

After the download succeeds, tell the user:

1. **What you built** — pack used (if any), size, country mix, scenario(s).
2. **Shape highlights** — final active headcount, # hires, # terminations,
   # promotions, top departments, country breakdown, anything the brief
   asked for. Pull these from `summary.md` or quickly load the employees
   parquet.
3. **Where the file is** — local path of the zip.
4. **`job_id`** — so they can re-download or rerun deterministically.

Keep the summary short (5–10 lines). Don't show raw JSON unless asked.

## Determinism

Same `(seed, config)` → byte-identical output. When a user wants to
reproduce a dataset, ask for the `job_id` and re-submit the same body. The
service stores no per-user state; reproducibility lives in the request.

## Errors and fallback

- `401` — `PEOPLESETS_API_KEY` is wrong or missing. Tell the user.
- `400` — bad config; the response body explains what failed.
- `409` on `/artifacts.zip` — job not yet done; keep polling.
- `5xx` — service issue; retry once after 5 s, then surface the error.

Never silently retry forever. Bound polling to 120 s and bound retries to 2.

## What not to do

- Don't call the deprecated `POST /jobs` endpoint. Use `/generate-company`.
- Don't fabricate country codes outside the supported set.
- Don't promise the dataset includes things the engine doesn't model
  (e.g. union membership, security clearance) — stick to the schema in
  `data_dictionary.md`.
- Don't use this skill for **real** PII or production HR exports. It only
  generates synthetic data.
