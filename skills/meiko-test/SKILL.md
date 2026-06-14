---
name: meiko-test
description: Use before writing or reviewing any Meiko test — pytest (backend) or Cypress (frontend).
---

# Meiko Testing Conventions

The TDD discipline ("write the test first; nothing ships without tests") lives in the entry skill `using-meiko-superpowers`. This skill is about how a Meiko test is *shaped* once you've decided to write one — layout, fixtures, assertions, time handling, mocks.

## Per-endpoint coverage checklist

For each new backend endpoint, cover at minimum:
- The happy path (200/201/204 with expected response shape)
- Every documented error response (404, 403, 409, etc.)
- Every conditional branch you added in the handler (e.g. `if depositReceived`, `if workGroupId`)

## Look at the nearest existing test first

Before inventing a fixture shape, mock layout, or assertion pattern, grep for the closest existing test in the same service. If a similar route, similar fixture relationship, or similar assertion already exists, copy its convention — that's how Meiko stays consistent across services. The pattern you remember from another project may already be stale.

## Backend tests (pytest)

### Layout & auth

- Location: `services/<svc>/src/tests/<feature>/test_<verb>_<noun>.py` (e.g. `tests/quote/test_unconfirm.py`).
- Class-based: extend the feature's project base (e.g. `BaseQuote` in `tests/quote/__init__.py`) so customer/boat/quote fixtures are reused. Add per-test-class fixtures with `@pytest.fixture(autouse=True, scope='class')` for data unique to that file.
- Auth: decorate the test class with `@authorized()` (from `tests/__init__.py`). For role-gated endpoints, write a second class with `@authorized(roles=['<other-role>'])` that asserts `403`.

### Fixtures: minimal and idiomatic

- **Only populate fields that are required by the model OR that the test specifically depends on.** Drop everything else. Examples of fields to skip: `boat`/`startOn`/`finishOn` on a quote when no assertion touches them, `confirmedCurrency='EUR'` when the model already defaults to `'EUR'` and you only assert it gets cleared to `None`, payment `currency`/`type`/`date` when they have defaults and aren't asserted on. Keep a field only if (a) the model requires it (NOT NULL FK), (b) at least one assertion depends on the value, or (c) it's needed to exercise a branch in the code under test (e.g. `receipt={...}` to cover the `if payment.receipt:` path).
- **Construct rows via relationships, never via explicit `*Id` FK assignment.** Use `QuotePayment(quote=cls.quote, ...)`, not `QuotePayment(quoteId=cls.quote.id, ...)` unless the parent was committed in an earlier fixture (the `cls.quote.id` pattern in `test_delete_payment.py` only works because `BaseQuote` committed it first).
- **Don't call `session.add()` for rows a relationship will cascade.** Assigning to a relationship (e.g. `cls.boat.quotes.append(cls.quote)`, or `cls.quote = Quote(customer=cls.customer, boat=cls.boat)`) implicitly adds the related rows. Adding `session.add(cls.customer)` and `session.add(cls.boat)` on top is redundant — let SQLAlchemy's unit-of-work do it.
- **Do not call `session.flush()` in fixtures.** Use `session.add_all([...])` + `session.commit()` once and let SQLAlchemy's unit-of-work handle dependency order and FK sync. Two commits are acceptable only when the model has an unusual relationship that SA can't auto-sync from relationship-side assignment (e.g. an inverted relationship using `foreign(OtherTable.id) == ThisTable.fkColumn`); in that case, commit once to materialize IDs, then set the FK and commit again. Do not pre-generate UUIDs with `uuid4()` just to avoid the second commit.
- **Shared setup goes in fixtures; scenario-specific mutations can live in the test body.** Prefer fixtures for state every test in the class uses. For state that only one test needs — especially mutations whose effect IS what the test is checking — in-test `session.commit()` is fine and used elsewhere in Meiko (see `payment/services/backend/src/tests/organization/hierarchy/test_hierarchy.py`). Pattern: set `db.session.expire_on_commit = False`, grab `session = db.session()`, mutate, `session.commit()`. If a direct attribute write on a fixture-set class attribute doesn't persist (typically when the model has an unusual relationship like `foreign(Other.id) == This.fkCol`), re-fetch the row with `Model.query.get(self.fixture_obj.id)` and mutate the fetched instance — this guarantees the object is in the current session's identity map.
- **Imports must follow the package's `__init__.py` re-exports.** If `app.models.__init__.py` re-exports `QuotePaymentType`, import it from `app.models`, not from the submodule. Don't fragment imports.

### Mock fixtures

The data-fixture rules above are about SQLAlchemy rows. **Mock-patch fixtures** — `patch.object(...)`, `patch('module.symbol')` — follow a different rule: **one fixture per concern, inject by parameter name only what the test asserts on.**

Wrong:

```python
@pytest.fixture(autouse=True)
def setup(self):
    app = self.client.application
    with patch.object(app.template, 'render', return_value=BytesIO(b'PDFBYTES')) as render_mock, \
         patch('app.routes.quote.requests.get') as qr_get:
        qr_get.return_value = MagicMock(ok=True, content=b'...')
        self.render_mock = render_mock
        self.qr_get = qr_get
        yield
```

Right:

```python
@pytest.fixture(autouse=True)
def qr_get(self):
    # autouse — protects against accidental real network calls
    with patch('app.routes.quote.requests.get') as qr_get_mock:
        qr_get_mock.return_value = MagicMock(ok=True, content=b'<svg></svg>')
        yield qr_get_mock

@pytest.fixture
def render(self):
    # opt-in — only tests that assert on render call inject it
    app = self.client.application
    with patch.object(app.template, 'render', return_value=BytesIO(b'PDFBYTES')) as render_mock:
        yield render_mock

def test_calls_qr_service_with_quote_url(self, qr_get, render):
    ...
    qr_get.assert_called_once()

def test_embeds_qr_in_template(self, render):
    ...
    assert render.call_args.kwargs['data']['qrCode'].startswith('data:image/svg+xml;base64,')
```

Rules:
- **One fixture per patched symbol.** Don't bundle unrelated `with patch(...)` contexts into a single `setup`.
- **`autouse=True` only for patches that must always be active** — network calls (so tests can't reach the real service), time freezing, anything that produces nondeterminism or external side effects if not patched.
- **Inject by parameter name to access the mock.** Tests that only need the patch *active* (no assertions on it) don't need to inject — autouse handles activation. Tests that assert on call args / return values inject by name and reference the parameter, not `self.<mock_name>`.
- **No `self.<mock>` attribute stashing.** It hides which test depends on which mock and forces every test to share one chain of dependencies.



- For present values: `assert resp.json['field'] == value`, or `assertIsSubsetOf({...}, resp.json)` from `meiko_test` for checking multiple fields at once.
- For null/cleared values: `assert 'field' not in resp.json`. Marshmallow omits null `auto_field`s from the dump entirely, so a cleared field is *absent* from the response, not `None`. Do not use `data.get('field') is None` — the membership check is the Meiko-wide convention (see `tests/work_group/test_delivery_same_as_pickup.py`, mhub/payment/platform tests for the same pattern).
- For confirming an auto-generated value exists: `assert resp.json.get('id') is not None` is fine (e.g. checking the server assigned a UUID).
- **Assert counts on side effects.** When the route under test creates/deletes rows, assert the row count and the relationship is correct, not just the response status. "It returned 200" is not the same as "it created exactly 3 orders" — match the pattern used by other tests in the same feature.

### Time-sensitive logic

Use `freezegun` (`from freezegun import freeze_time`) to freeze time around assertions that depend on "today" / "now" / time deltas. Check existing tests in the same service for the decorator/context pattern they use — don't invent a new one.

### Docstrings stay in sync

When the route's response code or shape changes (e.g. `201 → 200`, added a field, removed a 409 branch), update the test description (and the swagger docstring in the route) in the same change. Stale test descriptions mislead CI failure triage.

### External services

- S3 / external services are already mocked via `MagicMock()` in `tests/conftest.py`; do not re-mock locally.

## Frontend tests (Cypress)

- Location: `services/platform/src/cypress/e2e/<feature>/<feature>.cy.js`.
- Cover golden-path user flows and the error states the user can trigger from the UI (e.g. trying to unconfirm a quote that has a work group).
