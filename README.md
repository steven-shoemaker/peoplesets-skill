# peoplesets-skill

A [Claude skill](https://docs.claude.com/en/agents-and-skills/skills) that lets
Claude generate sales-demo-grade synthetic HR datasets in one prompt — no
prompt engineering, no API wiring.

Install:

```bash
npx skills add stevenshoemaker/peoplesets-skill
```

…or download [`peoplesets.skill`](https://peoplesets.com/peoplesets.skill) and
drop the folder into `~/.claude/skills/`.

Then in any Claude session, say something like:

> *make me a 1,200-person fintech that did a Q3 RIF*

Claude calls the [peoplesets](https://peoplesets.com) API, runs the sim,
downloads four parquets (employees, events, comp_events, recruiting), and
hands you the zip.

## Setup

The skill needs one environment variable:

```bash
export PEOPLESETS_API_KEY=psk_…
```

Grab a free key at <https://peoplesets.com/#get-key>. It's shown inline,
no card, no email confirmation step.

`PEOPLESETS_URL` defaults to `https://peoplesets.com` — only set it if
you're pointing at a self-hosted instance.

## What you can ask for

The skill translates natural-language briefs into the right API calls.
Examples that work today:

- *"a 75-person Series-A startup, US-only"*
- *"a 5,000-employee retail chain across the US and Canada with realistic frontline turnover"*
- *"a 3,000-employee hospital network with low attrition"*
- *"a US-headquartered tech company with most engineers in Bangalore"*
- *"a SaaS company that did a Q3 RIF and is now in retention crisis"*

Industries (`industry_pack`): `tech_startup`, `retail_chain`,
`healthcare_system`. Scenarios (`special_events`): `rif`,
`hyper_growth`, `m_and_a`, `distressed`, `leadership_shake_up`.
Countries (`country_mix`): `USA`, `GBR`, `CAN`, `IND`, `DEU`, `AUS`,
`BRA`, `IRL`.

See <https://peoplesets.com/docs> for the full catalog + schema.

## How it works

The skill is a single `SKILL.md` that gives Claude the API surface, the
intent-translation rules, and a handful of worked examples. Claude calls
the HTTP endpoints directly using the bash/curl tools it already has —
no Node, Python, or other runtime required on your machine.

The API is open source… well, sort of:
- Engine: closed for now (built by [Steven Shoemaker](https://stevenshoemaker.me), People Analytics at Deel).
- Skill: this repo. MIT-licensed. PRs welcome.

## License

MIT.
