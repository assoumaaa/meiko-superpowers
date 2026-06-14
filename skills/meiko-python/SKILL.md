---
name: meiko-python
description: Use before writing or reviewing any Python in a Meiko service — Flask routes, SQLAlchemy models, business logic, schemas.
---

# Meiko Python Conventions

Follows [PEP8](https://peps.python.org/pep-0008/). Linter is `flake8`, formatter is `black` — both configured to line length 88. The `format` hook in this plugin runs `black` automatically on every Edit/Write to a `.py` file, so you don't need to format manually after edits.

## Naming

| Element | Convention | Notes |
|---|---|---|
| Variables | `snake_case` | Noun or adjective; no hungarian notation |
| Constants | `SNAKE_CASE` | Global scope; no unit in name, use comment instead |
| Functions/Methods | `snake_case` | Start with verb: `get_`, `set_`, `is_`, `has_` |
| Classes | `PascalCase` | Abbreviations all-caps (`HTTPException`), acronyms PascalCase (`NasaWeatherData`) |
| Private methods | `_method_name` | Single leading underscore; no double underscore |
| Event callbacks | `on_message` | Start with `on_` |

- **No** `is_` or `f` prefix on booleans (use `user_active` not `is_user_active`)
- **No** abbreviations unless obvious (e.g. `org` is fine)
- **No** single-letter names except in lambdas and comprehensions
- **No** Hungarian notation (e.g. `num_users` is wrong; use `user_count`)

## Strings

- String literals: single quotes `'text'`
- Interpolation: `f-string` with double quotes `f"{value}"`
- Multi-line strings: triple double quotes `"""`

## Walrus operator

Use the walrus operator (`:=`) when binding a value inside a condition where the bound name is only used in the branch that depends on it. This keeps the assignment and the truthiness check on one line and scopes the variable to its actual use.

```python
# preferred
if payment := quote.confirmationPayment:
    do_something(payment)

if no := kwargs.pop('no', None):
    filters.append(Quote.no.ilike(f"%{no}%"))

# avoid when the binding will be used after the conditional
payment = quote.confirmationPayment  # ok — payment used below regardless
log.info("payment=%s", payment)
if payment:
    ...
```

Do not use the walrus inside expressions where it harms readability (deeply nested comprehensions, complex boolean chains).

## Imports

If a symbol is re-exported from a package's `__init__.py` (e.g. `app.models.QuotePaymentType`), import from the package, not the submodule. Don't fragment imports across multiple lines from the same package.

Clean up unused imports before pushing. The `format` hook runs `black` but doesn't catch them; `flake8` does (`F401`). The cleanest signal that an import is dead is: remove it and re-run `flake8` — if nothing breaks, it was dead.

## Class Methods

- Getters: use `@property`, named as a noun (e.g. `active`)
- Setters: `set_` prefix with `@<prop>.setter`
- Private methods: start with `_`

Don't ship empty base classes "for future use." Add the base when there's a second subclass that justifies it.

## SQLAlchemy ORM Entities

- Entity property names: `camelCase` (for auto schema mapping)
- Column names: `snake_case`

```python
class User:
    id = Column()
    createdAt = Column('created_at')  # property camelCase, column snake_case
```

### Money types

Always `Decimal` for monetary amounts, never `Float`. Float introduces rounding errors that propagate into invoicing and reconciliation. Map via `Column(Numeric(precision, scale))` so the ORM yields `Decimal` on read.

### Encrypted nested fields

For nested objects that need encryption-at-rest, use `Encrypted(Nested(SubSchema))` (not `Nested(Encrypted(...))`). The outer `Encrypted` wraps the whole nested field so encryption applies to the serialized blob, and the inner Marshmallow field remains a `Field` so `inner` resolves correctly.

## SQLAlchemy Querying (2.0 style)

Use `select()` with `Session` methods. The legacy `Query` API is not allowed.

- Use `stmt` as the variable name for select statements
- `db.session.scalars(stmt).all()` → ORM objects
- `db.session.execute(stmt).all()` → row/tuple projections
- `db.session.scalar(stmt)` → single scalar (e.g. COUNT)
- `db.session.get(Model, id)` → get by primary key
- Wrap raw SQL: `db.session.execute(text("..."))`
- Pagination: `paginate(db.session, stmt)` (not `paginate(query)`)

## Flask routes

### Response shape

- **Don't write explicit `200` after `return jsonify(...)`** — Flask defaults to 200. The explicit number is noise.

  ```python
  # good
  return jsonify({'projects': ProjectSchema().dump(account.projects, many=True)})

  # bad
  return jsonify({'projects': ...}), 200
  ```

- When you change a response code (`201 → 200`, `200 → 400`, etc.), update the swagger docstring AND the matching test description in the same change. Stale docstrings mislead reviewers and CI reports.

### Validation and aborts

- **Validate input at the boundary; trust it downstream.** The standard pattern is one `try/except ValidationError` around `schema.load(...)` that calls `abort(400, description=str(ex))`. After that, the data is clean — no more try/except for `ValidationError` further down.

  ```python
  try:
      account = AccountSchema().load(req, session=db.session)
  except ValidationError as ex:
      abort(400, description=str(ex))
  # ...business logic runs after validation, not before
  ```

  Run business logic *after* parsing, not before. Putting checks ahead of `schema.load` means you're checking unvalidated input.

- **Always pass `description=` to `abort`.** Bare `abort(400)` produces an empty error body that's useless to the client. Be specific: `abort(400, description='Project was not assigned to the account')`.

- **400 means invalid input/state.** Don't use 500 for cases the client caused, and don't use `abort` for control-flow — if the situation is "log and skip," log and skip, not abort.

- **Immutable fields in `PATCH`.** Identifiers tying a row into the data graph (`siteId`, `merchantId`, `accountId`, etc.) must not change via PATCH. Check the request body for these keys and `abort(400)` if they appear and differ — silently accepting them can cascade into device/site mismatches that take hours to clean up.

### HTTP requests

Pass JSON as `json=`, not `data=json.dumps(...)`. The library handles serialization and sets `Content-Type` for you.

```python
# good
resp = requests.post(url, json=payload, timeout=10)

# bad
resp = requests.post(url, data=json.dumps(payload), headers={'Content-Type': 'application/json'})
```

## Code shape

### Inline single-use helpers

If a function is called from one place, the caller is the right home. Extracting it just for the name is premature abstraction. Extract once a second call site (or a test that benefits from isolating it) appears.

### Shared helpers stay generic

A helper used from many call sites (`check_device_point`, `apply_quote_filters`, etc.) must not contain endpoint-specific branches. If only one route needs the special case, the route handles it; the helper stays narrow. The opposite — adding `if endpoint == 'x'` inside a shared helper — makes every other call site fragile to changes in the special-case branch.

### Common filters become one helper

When two routes filter on the same set of fields (`from/toDate`, `quote_id`, `payment_type`, etc.), build one generic filter helper they share. The shape is usually `def apply_filters(stmt, **kwargs)` returning the modified `stmt`. Two near-duplicate filter blocks drift independently within a quarter.

### Don't `session.add()` what relationships will cascade

In tests and fixtures, assigning to a relationship attribute (e.g. `cls.boat.quotes.append(cls.quote)`) implicitly adds the related row. You don't need `session.add(cls.boat)` separately. Let SQLAlchemy's unit-of-work do it.

## Paginated list endpoints

Split paginated list endpoints into a private cached getter and a public route handler. The getter is `_get_<plural>(args={})` decorated with `@cache.memoize(timeout=3600)` — it builds the query, calls `paginate`, and returns the `jsonify({...})` payload. The route handler `get_<plural>()` carries `@app.route`/`@auth_required`/`@paginated` and just does `return _get_<plural>(args=request.args)`.

## Configuration

App-level settings (healthcheck paths, feature flags, integration URLs, etc.) belong in `app.config[...]`. Infrastructure (Dockerfile, k8s) should reference the same key so changing the value once changes it everywhere:

```python
app.config['HEALTHCHECK_PATH'] = '/terminal/healthcheck'
```

Don't hardcode the path in three places.

### Loading env vars

Load environment-derived config in `app.py`'s `init_app(app)`, using `getenv(NAME, default)` with the k8s service name as the default. Routes read from `app.config[NAME]`, **never** `os.environ[NAME]` directly.

```python
# app/app.py — init_app
app.config['QR_SERVICE_URL'] = getenv('QR_SERVICE_URL', 'http://qr-code')
app.config['PLATFORM_URL'] = getenv('PLATFORM_URL', 'http://platform')
```

```python
# app/routes/quote.py — usage
qr_service_url = app.config['QR_SERVICE_URL']  # ✅
qr_service_url = os.environ['QR_SERVICE_URL']  # ❌ — bypasses app.config override and the central default
```

Why the default matters:
- **Tests**: pytest conftests bootstrap the app without calling `init_app`. The default (or a value explicitly set on `app.config` in conftest) keeps lookups from KeyError-ing. Set the same key in conftest's test-client fixture so `app.config['QR_SERVICE_URL']` resolves under test.
- **Production**: the default is the k8s service name (`http://qr-code`, `http://platform`) — the route works even before the ConfigMap is wired, and matches what a `kubectl get svc` would show.

If only one service consumes the env var, put the `env { name = "...", value = "..." }` block in that service's own `tf/replicas.tf` (next to where it declares `S3_BUCKET`, etc.), not in the top-level `tf/config.tf` `service_common` map. `service_common` is for keys every service reads (MQ, Redis, Keycloak base).

## Tooling

- Linter: `flake8` (`--max-line-length=88`, `--ignore=E722,E203,W391,W503`)
- Formatter: `black` (`--skip-string-normalization`, `--line-length=88`)
