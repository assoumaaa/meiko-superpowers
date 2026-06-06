# Meiko Development Guidelines

This file contains company-wide conventions and rules for all Meiko projects. Each project may have its own `CLAUDE.md` with project-specific additions, but these rules always apply. Detailed language- and test-specific guidance lives in the skills under `skills/`; this file carries the values, the workflow, and the security baseline that cross all of them.

## Follow existing conventions

Before writing new code — in tests or in production code — read at least one nearby example of the same kind of change and follow its conventions exactly. This applies to:

- **Imports** — group/order/source. If a symbol is re-exported from a package's `__init__.py` (e.g. `app.models.QuotePaymentType`), import from the package, not the submodule. Don't fragment imports across multiple lines from the same package.
- **Database setup in tests** — use relationships and a single `session.commit()` at the end of the fixture; do not `session.flush()` in fixtures unless a specific relationship genuinely requires the FK in advance.
- **Endpoint/handler structure** — mirror the closest existing endpoint for swagger docstring style, response codes, auth decorator placement, error message wording, and S3/file handling.
- **File/folder placement and naming** — put new files where the existing pattern places them (`tests/<feature>/test_<verb>_<noun>.py`, `pages/<Feature>/<Dialog>.jsx`, etc.) with names that match the existing scheme. Pages live in `views/`, reusables in `components/` — don't put route-bound shells under `components/`.
- **Variable naming** — match the patterns in the existing file (camelCase vs snake_case, boolean naming without verb prefixes, etc.) even when the language allows otherwise.
- **Existing utilities** — before writing an inline implementation, check what already exists: `lodash` (`isEmpty`, `isNil`, `isEqual`, `omitBy`, `pick`, `sortBy`, `cloneDeep`, `set`), the project's `validation.js` (`nonEmptyString`), redux slices for cross-entity lookups. Inline reimplementations look identical in the diff but drift in maintenance.

When in doubt, find the nearest analogous change in the repo (grep for similar route handlers, similar dialog components, similar test files) and copy the convention from there. Do not invent a new pattern when an existing one already covers the case.

## Reuse before invent; inline before extract

Two anti-patterns to watch for, both of which compound across services:

- **Reinventing a utility.** `_.isEmpty`, `_.isNil`, `_.sortBy`, `_.set`, `nonEmptyString` from `validation.js`, etc. — if it exists, use it. The reviewer's "why didn't you use lodash?" comment is asking the right question.
- **Pre-extracting helpers.** A function called from one place is at the wrong altitude. Keep it inline until a second call site or test makes the extraction pay rent. Three near-identical lines is better than a premature abstraction.

The mirror image also applies: if you find yourself adding a special-case branch inside a shared helper (`check_device_point`, generic filter helpers, dialog-action handlers), stop — the special case belongs in the caller, the helper stays narrow.

## Desired Code Properties

Code should be:
- **Readable** — names should communicate intent without needing comments
- **Simple** — prefer the simplest solution; avoid premature abstraction
- **Consistent** — follow the conventions below, even if you disagree with them
- **Secure** — never commit secrets, never log sensitive data, follow OWASP principles
- **Testable** — write code that can be unit tested without touching external systems

## Git Workflow

- Branch naming: `feature/kebab-case-title` or `fix/bug-title`
- Always branch from `develop` and keep it up-to-date before branching
- Commit messages must reference the issue: `(#301) lastSeenAt computed column is added`
- Multiple issues: `(#301, #309) Migration scripts are created`
- Merges require a Merge Request with a reviewer assigned in GitLab
- Enable "Delete source branch" when creating the MR

## Security

- **Never** commit passwords, API keys, or secrets to Git — use 1Password
- **Never** include sensitive data (passwords, personal data) in logs
- Production data must not be exported from its environment; use synthetic data for development
- Sensitive data (GDPR-covered) must be stored in encrypted columns
- Use standard log levels: `DEBUG, INFO, WARNING, ERROR, FATAL`
- Stay current on OWASP Top 10 and share significant security updates via Slack

## Skills

Detailed, language-specific conventions live as skills under `skills/`. Each has a `SKILL.md` written as principles, not checklists. The entry point — `using-meiko-superpowers` — explains when to reach for each.

- **`meiko-python`** — Flask/SQLAlchemy backend conventions: naming, walrus, SQLAlchemy 2.0 querying, paginated-endpoint split, black/flake8 tooling. Invoke before writing any `.py` file.
- **`meiko-js`** — React/JS frontend conventions: naming, null handling, translations, component section order, DataGrid alignment, prettier/ESLint tooling. Invoke before writing any `.js`/`.jsx`/`.ts`/`.tsx` file.
- **`meiko-test`** — TDD discipline, pytest layout/fixtures/assertions, Cypress E2E patterns, the per-endpoint coverage checklist. Invoke before writing any test.

**Always invoke the relevant skill before acting**, even if you "know" the topic. Skills evolve from real feedback; what you remember from last week may already be stale.

## Hooks (machine-enforced rules)

- **`git push` is gated.** You may not push without explicit approval in the current turn. This is a rule, not a principle.
- **Python edits run `black` automatically**, and **JS/JSX/TS/TSX edits run `prettier --write` automatically**, via PostToolUse hook. Don't skip them. If formatting fails, that's a signal to look at the file.

## When skills and instructions conflict

User instructions (this file, direct messages, project-level CLAUDE.md) win over skills. Skills win over default behavior. If a per-service or per-project CLAUDE.md sets stricter or different rules, follow the more specific file.

## The learning loop

Every correction is data. When you get pushed back on for a Meiko-specific reason ("we don't do it that way here"), log it in `skills/_journal.md` with date + concrete example + why it matters. Periodically fold the journal back into the relevant skill so the plugin gets sharper.

If the journal has 5+ unprocessed entries, surface that fact at the start of the next session.
