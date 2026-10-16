# time-slot-generator

[![Agent Skills](https://img.shields.io/badge/Agent_Skills-open_format-059669)](https://agentskills.io)
[![skills.sh](https://img.shields.io/badge/skills.sh-npx_skills_add-059669)](https://www.skills.sh)
[![Claude Code](https://img.shields.io/badge/Claude_Code-compatible-059669)](https://docs.claude.com/en/docs/claude-code/skills)
[![Codex CLI](https://img.shields.io/badge/Codex_CLI-compatible-059669)](https://developers.openai.com/codex/skills)
[![Gemini CLI](https://img.shields.io/badge/Gemini_CLI-compatible-059669)](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/skills.md)

An [Agent Skill](https://agentskills.io) that teaches an agent to build a bookable time-slot generator in a
TypeScript backend: slots generated from strategies (daily ranges, Nth weekday of the month, one-off slots,
whole days), authored in local wall-clock time, stored as UTC instants, and correct across DST, far-from-UTC
zones and any server time zone. Availability subtracts bookings in seats, blackouts, hosts' other
commitments, shared resources and working hours. Firestore and Supabase/Postgres are both carried.

Slots are never stored. Strategies are, and slots are regenerated on every read and found again by a
deterministic ID when someone books. So the generator has to be a pure function whose output does not
depend on the viewport, the server's time zone or the time of year. Otherwise a slot that was on screen is
gone when the guest presses "book". The skill is built around that constraint.

It was extracted from the slot engine of a production booking API and audited before extraction. The audit
ran the engine against DST, far-from-UTC and viewport cases. Eleven defects it found are fixed in the templates
and each is pinned by a test: weekdays dropped from monthly rules, days lost at DST changes, limits that
moved with the viewport, and busy time read in the wrong zone, among others. On ordinary windows the
templates still produce the original slot IDs, so existing bookings keep resolving.
[`references/provenance.md`](references/provenance.md) records what changed, what was kept deliberately
and why.

## Install

```bash
npx skills add <owner>/time-slot-generator
```

Or copy this folder into your agent's skills directory (`~/.claude/skills/`, `~/.codex/skills/`,
`~/.gemini/skills/`).

## Contents

| File | What it answers |
|---|---|
| [`SKILL.md`](SKILL.md) | When to use it, critical facts, hard rules, the seam contract |
| [`references/strategy-model.md`](references/strategy-model.md) | Strategy types, bounds, limits, types, validation |
| [`references/timezones.md`](references/timezones.md) | UTC vs wall clock, DST policy, Intl helpers, legacy normalizer |
| [`references/generators.md`](references/generators.md) | The engine, slot identity, horizon, cost |
| [`references/availability.md`](references/availability.md) | Bookings, blocks, resources, working hours, the booking path |
| [`references/firestore-backend.md`](references/firestore-backend.md) | Collections, indexes, store, transaction capacity check |
| [`references/supabase-backend.md`](references/supabase-backend.md) | Schema, RLS, store, `book_slots` |
| [`references/testing.md`](references/testing.md) | The suite and the differential check |
| [`references/provenance.md`](references/provenance.md) | Defects fixed, choices kept, additions |

## License

MIT
