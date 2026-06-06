---
name: meiko-js
description: Use before writing or reviewing any JavaScript/JSX/TS/TSX in a Meiko frontend — components, hooks, pages, dialogs.
---

# Meiko JavaScript / React Conventions

Follows [Mozilla JS style guide](https://developer.mozilla.org/en-US/docs/MDN/Writing_guidelines/Writing_style_guide/Code_style_guide/JavaScript). Formatter is `prettier` with each frontend project's `.prettierrc.js`; the `format` hook in this plugin runs `prettier --write` automatically on every Edit/Write to `.js`/`.jsx`/`.ts`/`.tsx`.

## Naming

| Element | Convention | Notes |
|---|---|---|
| Variables | `camelCase` | Noun or adjective; no hungarian notation |
| Constants | `SNAKE_CASE` | Global scope unless they need hooks |
| Functions/Methods | `camelCase` | Start with verb: `get`, `set`, `is`, `has` |
| Classes | `PascalCase` | Abbreviations all-caps, acronyms PascalCase |
| Private members | `#memberName` | Use `#` prefix |
| Event callback props | `onChange`, `onDelete` | Start with `on` |
| Event callback handlers | `handleChange`, `handleDelete` | Start with `handle` |

- **No** `is_` or `f` prefix on booleans (use `userActive` not `isUserActive`)
- **No** abbreviations unless obvious
- **No** single-letter names except in lambdas and for-of/for-in loops
- **No** Hungarian notation

## Strings

- String literals: double quotes `"text"`
- Interpolation: template literals `` `${value}` ``

## Equality and null/undefined

- **Always** use `??` (nullish coalescing), **never** `||` for null/undefined fallback (because `0` and `""` are valid values you'd accidentally fall through on).
- Use `??=` for assignment with fallback.
- **Never `!=` / `==`** — always strict `!==` / `===`. ESLint is the enforcement; the rule above it is "don't disable `eqeqeq`."
- For "null or undefined": `isNil(x)` / `!isNil(x)` from lodash. Don't write `x == null`.
- For specifically `null`: `x === null` / `x !== null`.
- For specifically `undefined`: `x === undefined` / `x !== undefined`.

## Translations

- **All user-facing literal strings** must be wrapped in `t("...")` from `react-i18next` (or `<Trans>` for JSX interpolation). This includes constants in modules ("Manufacturer", "Status", button labels, dropdown labels) — not just inline JSX text. If the string is shown to a human, it goes through `t()`.
- Memoize translated values when they're computed once and reused. `t()` reads from i18next state, so embedding it in a derived value re-evaluates per render. Wrap in `useMemo`:

  ```jsx
  const label = useMemo(() => t("Manufacturer"), [t]);
  const filters = useMemo(() => buildFilters(t), [t, /* deps */]);
  ```

- Translation files live at `<frontend>/public/locales/<lang>/translation.json` (e.g. `services/platform/src/public/locales/tr/translation.json`).
- The English source string is the translation key. When you add a key, add a matching entry to **every non-English `translation.json` that already has values** in the project. Keys sorted alphabetically.
- Keep a space after punctuation in translated strings: `t("Manufacturer, model")` not `t("Manufacturer,model")`. Whitespace differences create separate translation entries.

## Use existing utilities first

Before writing an inline implementation, look for an existing utility:

- **`lodash`** — `isEmpty`, `isNil`, `isEqual`, `omitBy`, `isUndefined`, `pick`, `omit`, `groupBy`, `keyBy`, `mapValues`, `sortBy`, `cloneDeep`, `set` (path-based write), `get` (path-based read).
- **Project `validation.js`** — `nonEmptyString` and other domain-specific checks. Use these instead of `value && value.trim()`.
- **Project redux state** — for cross-entity display ("show merchant name on terminal row"), pull from the redux slice (`useSelector(state => state.merchants.byId[id])`) instead of asking the backend to `JOIN`.

Patterns that come up repeatedly:

```jsx
// don't reach for: !str || !str.trim()
nonEmptyString(value);             // from validation.js
// don't reach for: value === null || value === undefined
isNil(value);
// don't reach for: a.length === b.length && a.every(...)
isEqual(new Set(a), new Set(b));   // unordered set comparison
// don't reach for: Object.fromEntries(Object.entries(o).filter(...))
omitBy(params, isUndefined);
// don't reach for: arr.slice().sort((a, b) => ...)
sortBy(items, ["fieldA", "fieldB"]);
// don't reach for: spread chains for nested updates
set(cloneDeep(state), "path.to.field", value);
```

## React hooks discipline

### `useMemo`

Reach for it when:
- Building a derived array/object the render depends on: `columns`, `filtering`, `options`, `rows`.
- Computing a value that depends on `t(...)` or other slow lookups.
- The value would change identity on every render and feed into a memoized child.

```jsx
const columns = useMemo(() => buildColumns(t), [t]);
const options = useMemo(
    () => sortBy(terminals ?? [], ["serialNumber", "terminalId"]).map(toOption),
    [terminals],
);
```

If you reach for `useEffect` to derive state from props, you usually want `useMemo` instead. Effects are for side effects (fetches, subscriptions, syncing external stores), not for "computing X from Y."

### `useCallback`

Reach for it when a child component is memoized (via `React.memo` or as a `useMemo` dependency) AND the handler identity matters. Otherwise it adds noise without benefit. When you do use it, **the dependency list is part of the contract**: a missing dep gives stale closures. If you're using `fetchDeviceTypes` inside, it's in the deps.

### `useEffect`

For: triggered fetches, subscriptions, DOM imperative work, sync with external systems. Not for: computing derived state, replacing `useMemo`.

```jsx
// good: fetch when something opens
useEffect(() => {
    if (open) fetchOptions();
}, [open]);

// bad: derive in effect what useMemo could do
useEffect(() => {
    setLabel(t("Manufacturer"));      // ← useMemo(() => t("Manufacturer"), [t])
}, [t]);
```

## State hygiene

- **Don't reach for `useState` when a derived const or `useMemo` works.** State has a re-render cost and a synchronization cost; only adopt it when the value can't be computed from other state/props.
- **Loading state should hide the UI until dependent data is ready.** If a component needs `deviceTypes` AND `accounts` to render meaningfully, show a spinner/skeleton until both are loaded — don't render with empty arrays and flash a "no data" UI for 200ms.
- **Action menus: one state, not N booleans.** For dialogs/menus with multiple branches, prefer a single tagged-action state over a boolean per option:

  ```jsx
  const [action, setAction] = useState();
  // openers
  onEditSettings: (terminal) => setAction({ terminal, name: "editSettings" }),
  onChangePassword: (terminal) => setAction({ terminal, name: "changePassword" }),
  // renderer
  {action?.name === "editSettings" && <EditSettingsDialog ... />}
  ```

- **Path-based form updates**: `set(cloneDeep(state), path, value)` from lodash for nested form changes. The handler is `(path, value) => setState(s => set(cloneDeep(s), path, value))` and `path` can be `"type"` or `"contactDetails.email"`.

## Component design

- **Default values via destructuring**:
  ```jsx
  function Avatar({ size = 44, ...rest }) { ... }
  ```
  not `props.size || 44` (which fails on `0`).
- **Drop unused props.** If a prop appears in the signature but no logic uses it, remove it from the destructure and let `...rest` forward it. Carrying `value, onChange, multiValued` in the signature with no consumer is misleading.
- **`onError` propagation** beats inline catch. Each component throws/propagates; the parent owns the error UX (toast, dialog). Centralizing reduces duplicated retry logic.
- **Dialog actions**: `onPositiveClicked` / `onNegativeClicked` for the standard pair. For multi-step dialogs, replace `actions` with `slots` so next/prev buttons can be placed contextually.
- **Status indicators**: `<Chip>` with color over plain text. A green Chip reads as "active" without forcing the user to read the translated word; the color is the signal, the label is the affordance.
- **Delete actions wear `error` color.** Reds for destructive — never gray-on-gray.
- **Navigation uses `<Link>`**, not `<Button onClick={() => navigate(...)}>`. Right-click "open in new tab" matters; middle-click on a button is a dead end.
- **`<Autocomplete>` callbacks**: `(_, option) => option?.value`, not `(value) => ...`. First arg is the change event you usually don't need; second is the selected option.
- **MUI spacing uses integer multipliers**: `mb: 2` (= `theme.spacing(2)`) not `mb: "1rem"`. Pull from the theme; don't hardcode units.

Structure components in this order:
  1. states
  2. selectors
  3. hooks
  4. memos
  5. callbacks
  6. effects
  7. return / render

Component names are `PascalCase`. Boolean props default to `false` so they can be passed as shorthand: `<Input disabled />`.

## Destructuring

- Destructure object fields when accessing them
- Destructure function/method arguments; use `...rest` (not `...props` or `...kw_args`)
- Boolean args default to `false` so they can be passed as shorthand: `<Input disabled />`

## API call shape

- **Don't send immutable fields in `PATCH` bodies.** `siteId`, `merchantId`, `accountId` etc. are set at create-time; including them in the PATCH payload invites the server to either silently ignore them or worse, accept the change. Send only mutable fields.
- **Don't pass `fetch` methods as props.** If a component needs payments, the payments endpoint is `GET /quote/:id/payments` — call it from the component (or a hook). Passing a parent's fetcher through props couples consumers to the parent's API shape.
- **For "list X of Y", paginate X.** Don't make the backend `JOIN` Y rows into every X response just to show "Y name" in the column. Pull Y from the redux store on the frontend; the join happens client-side, pagination semantics stay clean.

## DataGrid columns

- **Right-align number columns** (`align: "right"`, `headerAlign: "right"`) so the digits line up and values are easy to compare down the column.
- **Currency and price can share one column** (e.g. show `£12.34`), but **sort by the numeric price only** — never by the formatted currency string.

## Project structure

- **Pages live under `views/`, not `components/`.** A "view" is bound to a route; a "component" is reusable across views. If the file ends up in `routes/quote/payments.jsx`, the rendered shell belongs in `views/quote/Payments.jsx`. Some legacy files still sit under `components/` — that's the bug, not the precedent.
- **`views/` index doesn't re-export pages.** Each page is encapsulated under its folder (or parent component), and sibling views reach each other via relative imports. The top-level `views/index.js` should not list every page.
- **Shared state across siblings → Context, not prop drilling.** If two branches of the tree need the same filter state, lift to a Provider at the common ancestor. Don't thread through 6 levels.
- **URL query params → centralized provider hook.** A `QuickFilterProvider` at the route root reads/writes the URL (`?q=...`), exposes a tuple `[quickFilter, setQuickFilter]`, and pages consume `useQuickFilter()`. Don't each parse `useSearchParams` and re-encode.

## Storybook

New visual components ship with a Storybook story in the same change. The story is the contract: if you can't write one, the component's API isn't clear yet.

## Tooling

- Linter: ESLint
- Formatter: Prettier (`tabWidth: 4`, `semi: true`, `singleQuote: false`, `printWidth: 120`, `useTabs: true`)
- Import sorting: `@meikosoft/import-sort-style`. If saving doesn't strip extra blank lines / regroup imports, the plugin isn't running — fix the editor config rather than committing manual fixes.
