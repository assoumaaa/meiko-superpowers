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

---

## 2026-06-14 — SVG over PNG-with-baked-scale for HTML-embedded images (MR #89)

**What:** colcek on `qr.py`: "you can get svg and scale it in html template. you don't need the scale". Original change baked `scale=10` into `qr_code.save(buffer, kind=type, scale=10)`; reviewer wanted the route to return the natural-size SVG and let the consuming `<img style="width: 3cm; height: 3cm">` handle sizing.

**Why it matters:** when the output is an image meant for HTML rendering, prefer the vector form (SVG) and let CSS size it. PNG with a hardcoded scale freezes the resolution at one number — too small at print DPI, too big in DOM. The pattern generalizes beyond QR codes: any generator that supports both raster and vector output should default to vector for HTML consumers.

**Likely target skill:** other (UI/document-generation niche; doesn't fit meiko-python's Flask/ORM scope or meiko-js's React scope)

**Status:** unprocessed

---

## 2026-06-14 — service-specific env vars belong in the service's tf, not the shared config map (MR #89)

**What:** colcek on `tf/config.tf`: "these are specific to `backend` service put it into respective `tf` script as env vars". I added `QR_SERVICE_URL` and `PLATFORM_URL` to the top-level `locals.service_common` map (which becomes the `service-config` ConfigMap mounted by every service). Reviewer wanted them as `env { name = "...", value = "..." }` blocks in `services/backend/tf/replicas.tf`'s container spec — the same place backend already declares `S3_BUCKET`.

**Why it matters:** the top-level `service_common` is for values every service consumes (MQ host, Keycloak base, Redis). Putting service-specific keys there pollutes every other service's environment with strings it doesn't read, and makes the value's owner ambiguous — if QR_SERVICE_URL only backend reads, backend's tf should be where you find it. Same scoping principle as Python imports or app.config: narrowest scope that works.

**Likely target skill:** other (terraform/infra; no skill covers it yet — candidate for a future `meiko-terraform` skill if more patterns accumulate)

**Status:** unprocessed

---

## 2026-06-14 — routes read app.config; env loading lives in app.py with getenv defaults (MR #89)

**What:** colcek on `quote.py:_fetch_quote_qr`: "it is better to use `app.config` for env vars and use a default value such as, `getenv('QR_SERVICE_URL', 'http://qr-code')` so do it in `app.py`". My route did `os.environ['QR_SERVICE_URL']` directly. Reviewer wanted the env loaded into `app.config` inside `init_app` with a `getenv(NAME, default)` fallback, and the route reading `app.config['QR_SERVICE_URL']`.

**Why it matters:** routes that read `os.environ` directly bypass two things — (1) the `app.config` test override mechanism (conftest can set values per-test without touching env), and (2) the central place where defaults are declared. The default matters in production too: `'http://qr-code'` is the k8s service name, so the value works even if someone forgets to wire the ConfigMap. This is already partially in meiko-python's `## Configuration` section but the phrasing is too soft — it says values "belong in `app.config`" but doesn't forbid `os.environ` in routes or call out `getenv(name, default)` with the k8s-service-name idiom.

**Likely target skill:** meiko-python (`## Configuration`)

**Status:** [done] processed 2026-06-14 — tightened meiko-python's `## Configuration` section to make the rules explicit.

---

## 2026-06-14 — split mock patches into per-concern fixtures, DI by name (MR #89)

**What:** colcek on `test_download_quote.py`: "the proper usage is to use separate fixtures and dependency injection because each test can call a different chain of dependencies". I had a single `@pytest.fixture(autouse=True) def setup` that opened TWO `with patch(...)` contexts (template render + requests.get), stashed both mocks on `self`, and let every test reach into `self.render_mock` / `self.qr_get`. Reviewer wanted them split: `render` as a regular fixture injected only by tests that assert on the render call, `qr_get` either same shape or as `autouse` for the network mock — tests inject by parameter name what they need.

**Why it matters:** lumping unrelated patches into one autouse setup hides which test depends on which mock. Splitting them makes the test's dependency chain explicit, lets you reuse the fixture across files that need only one of the two patches, and means a test that only cares about render assertions doesn't get a stale `self.qr_get` floating around. Autouse remains correct for patches that must always be active (network calls, time freezing) — the split is about *naming and granularity*, not *autouse vs not*. Current meiko-test `### Fixtures` section is entirely about data fixtures (SQLAlchemy rows); nothing yet on mock-patch fixtures.

**Likely target skill:** meiko-test (new `### Mock fixtures` subsection)

**Status:** [done] processed 2026-06-14 — added `### Mock fixtures` subsection to meiko-test.

