---
name: using-meiko-superpowers
description: Use at the start of any Meiko task to know which Meiko-specific skill to reach for. Routes to meiko-python (Flask/SQLAlchemy backend), meiko-js (React frontend), and meiko-test (pytest + Cypress). Invoke before writing any Meiko model, endpoint, component, or test.
---

# Using Meiko Superpowers

You're shipping production code for a multi-service Meiko deployment (Python/Flask/SQLAlchemy backend across `payment`, `platform`, `mhub`, `service-mes`, `common`, plus React frontends in `frontend/services/*`, `mhub/apps/*`, and `platform/src/`). The values doc `CLAUDE.md` (loaded automatically) carries the workflow and security baseline; the skills below carry the operational conventions.

## Principles

### When to invoke which skill

- **`meiko-python`** — before writing or reviewing any `.py` file: Flask routes, SQLAlchemy models, schemas, fixtures, business logic. Carries naming, walrus, SQLAlchemy 2.0 querying, the paginated-endpoint split, and the black/flake8 tooling.
- **`meiko-js`** — before writing or reviewing any `.js`/`.jsx`/`.ts`/`.tsx` file: React components, hooks, pages, dialogs, DataGrid columns, utilities. Carries naming, null handling (`??` not `||`, `isNil` not `== null`), translations, destructuring, component section order, DataGrid alignment, and the prettier/ESLint tooling.
- **`meiko-test`** — before writing or reviewing any test: pytest under `services/*/src/tests/` or Cypress under `services/*/src/cypress/e2e/`. Carries TDD discipline, the per-endpoint coverage checklist, fixture rules (relationships not FK IDs, single `session.commit()`, only required+asserted fields), assertion patterns (`'field' not in resp.json` for cleared values), and Cypress patterns.

More skills will be added over time. When no specific skill applies, fall back to the values in `CLAUDE.md` and the closest analogous code in the repo (the "follow existing conventions" principle).

### Test-first discipline (TDD)

Write the test before the implementation. The cycle is: **write failing test → implement → refactor.** This applies to every new feature, every changed business rule, every non-trivial bugfix.

- **Unit tests** test public methods in isolation; don't test interactions with other units or libraries.
- **Integration tests** test interactions between units/services; stub or mock external dependencies; cover edge cases.
- **Acceptance tests** test common user paths and the error states a user can trigger; don't test edge cases.

**No feature, endpoint, or non-trivial bugfix is complete without tests.** The suite must be green before handoff, and the new behavior must be covered by *new* tests — not just existing tests that happen to still pass. The conventions for how each kind of test is shaped (pytest layout, fixtures, assertions, Cypress patterns) live in `meiko-test`; the discipline of writing them first lives here.

### Follow nearby code, not memory

For every new test, endpoint, fixture, or component, find the closest analogous file in the repo and copy its conventions. Import order, response shapes, fixture style, file placement, variable casing — copy the local pattern. The `CLAUDE.md` "Follow existing conventions" section is normative: this is how Meiko avoids style drift across services.

### Read CLAUDE.md first

`CLAUDE.md` is the umbrella values doc, loaded automatically. It carries the git workflow, security baseline, and the meta-principle of following existing conventions. Skill files don't repeat what's in CLAUDE.md — they extend it with the language-specific detail. Both layers apply.

### Cross-harness behavior

This plugin works in Claude Code, Codex, and Gemini CLI. Skill invocation differs by harness:

- **Claude Code:** use the `Skill` tool with the skill name (no leading slash).
- **Codex:** use the `skill` tool the same way.
- **Gemini CLI:** skills activate via `activate_skill`; this entry skill plus `CLAUDE.md` are loaded automatically via `GEMINI.md`.

### The learning loop

Every correction is data. If you get pushed back on for a Meiko-specific reason ("we don't do it that way here"):

1. Append an entry to `skills/_journal.md` with date, the concrete case, and what should have happened.
2. When the journal hits 5+ entries, fold the lessons back into the relevant skill file (`meiko-python` / `meiko-js` / `meiko-test`) so future sessions inherit the lesson.

Don't let the journal become a graveyard.

## What good looks like

- A new endpoint where the swagger docstring, auth decorator, response codes, and error wording match the closest existing endpoint in the same service — and the matching test file under `tests/<feature>/` covers the happy path, every documented error, and every conditional branch in the handler.
- A React component where `states / selectors / hooks / memos / callbacks / effects` appear in that order, user-facing strings are wrapped in `t(...)`, and the matching `translation.json` files have new keys added in alphabetical order.
- A SQLAlchemy fixture that uses `parent_obj=cls.parent` (relationship), commits once at the end, populates only the fields the test asserts on, and lets the unit-of-work handle FK sync.
- A commit message that references the issue: `(#301) lastSeenAt computed column is added`.

## What bad looks like

- A new endpoint that invents its own response shape because "it's cleaner."
- A test fixture that pre-generates `uuid4()` IDs just to avoid a second commit.
- `value || defaultValue` in JS where `??` is required (because `0` and `""` are valid).
- A user-facing string hardcoded in JSX.
- A commit on `develop` that doesn't reference an issue number.
- A green test run on a polluted dev DB that you've been reusing across features.
