# Skill Journal

Append-only log of corrections, surprises, and "I should have known that" moments.
Process when this file has 5+ unprocessed entries.

Format per entry:

```
## YYYY-MM-DD — short title

**What:** concrete case, verbatim snippet or phrase
**Why it matters:** the underlying pattern, not the surface
**Likely target skill:** <skill name> | other
**Status:** unprocessed
```

Mark `Status: [done] processed in commit abc1234` once folded into a skill.

---

## 2026-06-06 — folded 685 colcek MR comments into meiko-python / meiko-js / meiko-test

**What:** Pulled all 254 MRs authored by assoumaaa where colcek (Can Olcek) was the reviewer; extracted 685 inline review comments from him (426 on .jsx, 234 on .py). Clustered recurring patterns and added them as principles to the topic skills.

**Why it matters:** Colcek's reviews concentrate the senior-reviewer wisdom that wasn't yet in CLAUDE.md — concrete patterns like "use `json=` not `json.dumps`", "`Decimal` not `Float` for money", "no need for `200` after `jsonify`", "always `abort(400, description=...)` never bare", "immutable IDs must not be in PATCH bodies", "`useMemo` for derived/`t()` values not `useEffect`", "lodash/`validation.js` before inline implementations", "pages under `views/` not `components/`", "frontend redux state instead of backend joins for cross-entity display." These principles transfer across the codebase and were appearing again and again in his comments.

**Likely target skill:** meiko-python (Flask response/abort shape, `json=`, `Decimal`, code-shape rules, encrypted-nested-fields, healthcheck config), meiko-js (state hygiene, `useMemo`/`useCallback` discipline, lodash-first, component design patterns, project structure, PATCH/pagination shape, storybook), meiko-test (count assertions, `freezegun`, "look at the nearest existing test first," "don't `session.add()` what relationships will cascade", "docstrings stay in sync with response codes")

**Status:** [done] processed 2026-06-06 — added sections to all three topic SKILL.md files; added meta-principles "Reuse before invent; inline before extract" + "Existing utilities" bullet to CLAUDE.md.

