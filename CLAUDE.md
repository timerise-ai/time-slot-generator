# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An [Agent Skill](https://agentskills.io) package: markdown only. There is no `package.json`, build or dev
server here, and nothing in this repository executes. It teaches an agent to build a strategy-driven
time-slot generator with availability in *someone else's* **Next.js App Router** app, on Firestore or
Supabase/Postgres. The version of record is the git tag.

The commands and code in `references/` describe the app the agent will generate, not this repository. The
one thing checked here is that the templates compile and their tests pass, in a scratch project; the recipe
is below.

The skill was written by the engineer who owns the module, and audited against the earlier implementation.
`references/provenance.md` is the rationale layer and the ledger of that audit: what the audit changed and
how the templates verify it, what was kept deliberately with the reason it is safe, and what was designed
here and has never run in production. Read it before "simplifying" anything.

Sibling directories under `../` (`booking-kiosk`, `calendar-scheduling-links`, `bookable-events`, and so on)
are other skills, not dependencies. They are referenced by name from `SKILL.md` and `README.md`. The
standard every skill follows is `../skills/STANDARD.md`.

## Structure

- `SKILL.md`: entry point, loaded whole on every activation, so it stays between 130 and 160 lines, the
  closing index line aside. The frontmatter `description` is the trigger surface. It carries a short
  adaptation contract table; the full contract is `references/adaptation.md`.
- `README.md`: the human-facing front door, in the section order every Timerise skill shares.
- `CHANGELOG.md`: Keep a Changelog, newest release first. The version lives here, in the README's
  current-release line, and in the git tag, and the three agree.
- `LICENSE`: MIT, identical to the siblings'.
- `references/*.md`: one concern per file. `adaptation.md` the seam and the rename table;
  `strategy-model.md`, `timezones.md`, `generators.md` and `availability.md` the pure engine;
  `supabase-backend.md` and `firestore-backend.md` the stores; `nextjs-routes.md` the route handlers;
  `testing.md` the suite; `provenance.md` the ledger.
- `evals/`: `prompts.md` holds what an operator types after installing, in their words; the first prompt
  is the agent eval run before every release. Every other file there is one eval run: measured frontmatter
  that is never edited, then the notes of the person who ran it. Add a prompt rather than rewording one that
  has results. The procedure is section 10 of the index's STANDARD.md.
- `.github/workflows/agent-eval.yml`: the caller of the index's reusable eval workflow, copied verbatim from
  section 10 of the standard and run on every published release and on a maintainer's dispatch. It is the
  same in every skill; never edit it, and never add a trigger on `push` or `pull_request`.

## Editing conventions

- **Code blocks name their destination on the first line**: `// lib/slots/generators.ts: description`, or
  `// app/api/slots/route.ts: description`. The schema is the one `sql` block in `supabase-backend.md`. The
  fifteen TypeScript files are spread across the references; list them with
  `grep -rn '^// \(lib\|app\)/' references`. Engine files import each other as `./<file>` with no extension;
  the routes import `@/lib/slots/...`.
- **The code blocks are compiled and run.** Write each block to its path in a scratch project (never this
  repo) with `next`, `react`, `@supabase/supabase-js`, `firebase-admin` and `@types/node` borrowed from any
  project, and a `tsconfig.json` that is `strict`, sets `noUncheckedIndexedAccess` and maps
  `"@/*": ["./*"]`. Then:

  ```bash
  npx tsc --noEmit -p .
  for tz in UTC Europe/Warsaw Pacific/Auckland America/Los_Angeles; do TZ=$tz node --test lib/slots/engine.test.ts; done
  ```

  For the second line, Node 22.18 or later, with the extensionless imports rewritten to `.ts` in a copy.
  The routes also build with `next build` in a copy of `../skills/eval/fixture` with no environment set.
  `schema.sql` must apply to a scratch Postgres with a stub `auth.uid()`, the `anon`, `authenticated` and
  `service_role` roles, and Supabase's default grants on `public` (all tables and functions to all three).
  There, `anon` must read no `services` row and be refused `book_slots`, and as `service_role` a second
  `book_slots` call on a full slot, or on an overlapping slot of another ID, must raise
  `slot-has-not-enough-capacity`. Do not add a `package.json` or test config to this repository.
- **Never change the slot ID input string** (`md5("<utcStartISO>;<duration>;<strategyId>")`, built in
  `slot-id.ts` in `generators.md`). Every consumer's existing bookings depend on it, and a test pins it.
- **Test and entry counts are claims.** The suite has 23 tests; the `SKILL.md` description, the README and
  `CHANGELOG.md` repeat the number, and `testing.md` states it. `provenance.md` states its own entry count
  (32). Change a suite or the ledger, change the numbers.
- **Keep the three indexes in sync** with `references/`: the quick start and the reference directory in
  `SKILL.md`, and the file table in `README.md`.
- **Identifiers are shared across files.** `generateSlots`, `buildAvailability`, `listServiceSlots`,
  `prepareBookingSlots`, `resolveSlotsForBooking`, `toSlotOutput`, `SlotStore`, `BookableService`,
  `validateStrategy`, `normalizeStrategy`, `saveStrategy`, `assertCapacityInTransaction`, `md5SlotId`,
  `slotIdInput`, `loadBookableService`, `errorResponse`, the error codes and `book_slots`. Rename in all files
  or none.
- **Two kinds of name.** The domain words the host renames (Service, Strategy, Booking, Host, Location,
  Resource, BusyBlock, project) are the rename table in `adaptation.md`. The identifiers above are the
  authoring contract of this repository.
- **The non-negotiables are never presented as optional.** The five hard rules in `SKILL.md`, the five
  non-negotiables in `README.md` and the list in `adaptation.md` are one list, in one order. Changing one is a
  MAJOR release.
- **Measured and design numbers are load-bearing.** `maxHorizonDays` 731 and `maxSlots` 50000 in
  `DEFAULT_LIMITS`, the 30-value Firestore `in` limit, `MAX_VIEWPORT_DAYS` 62 and `MAX_SLOTS_PER_BOOKING` 50
  in the routes, and the 100-second timeout in the ledger. Change one, change every file that states it.
- **Do not remove the odd-looking parts.** Each is a ledger entry in `provenance.md`.
- **Mark additions as additions.** Anything designed in the skill and never run in the earlier
  implementation belongs in "Designed here, not yet run in production" in `provenance.md`.
- **Evals are not skill content.** A new prompt or an eval result is committed as `chore(evals): ...`, never
  causes a version bump and never rides in a release commit. After each eval, the README's eval badges point
  at the newest prompt 1 result per agent, updated in the commit that adds the run's notes.
- **Origin wording.** Call the codebase the audit ran against "the earlier implementation", and follow
  section 2 of the standard: none of its banned words, no denials, no corpus figures, and no defect counts or
  failure stories on the front door (README intro, `SKILL.md`, the description, this file).
- **Plain punctuation.** No em-dashes, en-dashes, arrows in prose, middle dots, ellipsis characters or smart
  quotes anywhere in this repository's markdown; diagrams are plain ASCII. `LC_ALL=C grep -rn '[^ -~	]'
  --include='*.md' .` must print nothing. Prose wraps at 110 columns; table rows and commands stay on one
  line.
- **Claims are verifiable.** A changed factual claim says how it was verified. Never from memory.
- **Releases.** Patch for template fixes and wording, minor for a new reference, strategy feature or backend,
  under section 9 of the standard. Commits follow Conventional Commits, and no file or commit message names a
  tool or a model as author.
