# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An [Agent Skill](https://agentskills.io) package: markdown only. There is no `package.json`, no build and
nothing here executes. It teaches an agent to build a strategy-driven time-slot generator with availability
in *someone else's* TypeScript backend, on Firestore or Supabase/Postgres.

The code blocks in `references/` are the templates an agent copies. They are kept as complete files (one
fenced block per file, headed by a `// slots/<file>.ts` comment) so they can be extracted verbatim and
checked. Do not add a `package.json` or test config to make them runnable here.

## Structure

- `SKILL.md`: the router, loaded on every activation. Keep it at or under 150 lines. The `description`
  frontmatter is the activation trigger. The body holds when/when-not, the architecture, seven critical
  facts, five hard rules, the Adaptation Contract and the reference directory.
- `references/`: one question per file. `provenance.md` is the rationale layer; read it before
  "simplifying" anything, because most odd-looking choices are listed under "Kept deliberately".

## Verifying a change

Template edits must be re-verified the way 0.1.0 was:

1. Extract each `// slots/*.ts` block and the SQL block into a scratch project.
2. `tsc --strict` passes, and also passes with `--noUncheckedIndexedAccess`. Borrow `firebase-admin`,
   `@supabase/supabase-js` and `@types/node` from any project.
3. `node --test` passes under `TZ=UTC`, `Europe/Warsaw`, `Pacific/Auckland` and `America/Los_Angeles`.
   Rewrite the extensionless imports to `.ts` for Node's type stripping.
4. `schema.sql` applies to a scratch Postgres with a stub `auth.uid()`.

Never change the slot ID input string (`<utcStartISO>;<duration>;<strategyId>`). Every consumer's
existing bookings depend on it.

## Releases

Bump `CHANGELOG.md` with Keep a Changelog headings. Patch for template fixes, minor for new strategy
features.
