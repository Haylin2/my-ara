# UI cascade debugging & color verification (browser + CDP)

Field-verified recipe for finding which CSS declaration produces a visual color bug (e.g. unreadable active menu item), on this app's stack: MaryUI 2.x (robsontenorio/mary) + daisyUI 5 (+ Tailwind 4) + Livewire 4 `wire:navigate`.

## Resolution chain of the active menu item (measured)

1. MaryUI `MenuItem` sets `aria-current="page"` + `data-current` (Livewire `wire:navigate` marks the current link), and class `mary-active-menu <activeBgColor=default 'bg-base-300'>` when route matches.
2. `.mary-active-menu` has **no CSS at all** — it is only a marker. daisyUI matches the item via its big group selector `.menu :where(li) > :not(ul, menu, .menu-title, details, .btn):active, ..., .menu-active, ..., [aria-current]:not([aria-current="false"], [aria-current=""])` → `color: var(--menu-active-fg)` + `background-color: var(--menu-active-bg)`.
3. `.menu` itself defines the pair as tokens: `--menu-active-fg: var(--color-neutral-content); --menu-active-bg: var(--color-neutral)`. So the active pair is **neutral/neutral-content, NOT base-300/base-content** — the `bg-base-300` class loses on specificity and is dead. Any fix plan that targets `bg-base-300` or base-* tokens is already wrong.
4. Contrast math: daisyUI default (fantasy) neutral pair is oklch(86% 0 0) background vs oklch(85.561% .005 256.847) text — ~1:1, a solid-color bar with invisible text.

## Tracing the winning declaration with CDP

Use the browser tool's `cdp()` helper; the return value is the result object itself (e.g. `d['root']['nodeId']` after `cdp('DOM.getDocument')`).

1. Matched-rule calls throw 'CSS agent was not enabled' — always `cdp('CSS.enable')` first.
2. `node = cdp('DOM.querySelector', nodeId=root, selector='a.mary-active-menu')['nodeId']`.
3. `m = cdp('CSS.getMatchedStylesForNode', nodeId=node)` — iterate `m['matchedCSSRules']`; each rule: `rule['selectorList']['text']`, `rule['style']['cssText']`, `rule['styleSheetId']` (None + empty cssText = UA rules).
4. Custom props: `CSS.getComputedStyleForNode` — computed custom-property values arrive already-substituted per element (e.g. `--menu-active-fg` on `<ul class="menu">`). Walk the ancestor chain (`documentElement → body → div.drawer-side → inner column div → nav → ul.menu → li → details → nested ul → li → a`) sampling tokens AND `color`/`background-color` per level to find where theme dependence breaks.
5. To know WHY a rule matched a daisyUI compound selector, enumerate its alternatives explicitly (`:active` only matches during mousedown; `[aria-current]` is what really matched a Livewire-navigated item); check node attributes (`aria-current`, class list) before assuming class-based matching.

## getComputedStyle traps (verified)

- Do NOT flip `data-theme` and read computed styles inside one synchronous JS pass and trust value identity across iterations — resolved colors can stay pinned to the `:root` default-theme values (serialized as oklab/oklch) while ancestor custom props update correctly. Readings that stayed identical across all 7 themes matched the daisyUI `:root,[data-theme=fantasy]` values exactly, i.e. default-theme contamination in the readout, not real cascade truth.
- Measure each theme through the **real app mechanism**: set `localStorage.setItem('mary-theme', JSON.stringify('<theme>')); localStorage.setItem('mary-class','')` while on an app URL, then do a full reload/navigation per theme, then measure fresh. Report per-theme raw serialized values plus contrast ratios computed offline from the tokens.
- Prefer `document.body.innerText` / `[...document.querySelectorAll(...)]` text probes over pixel checks; use a clipped screenshot only as visual evidence when text data cannot decide.

## Browser-exec tool pitfalls

- A `js()` probe that throws kills the entire call — wrap risky reads (`?.`, try/catch) and fail soft: `document.querySelector('#x')?.value ?? 'no el'`.
- `localStorage` throws SecurityError when the current document is not the app origin (fresh tab/about:blank) — navigate to the app first, then touch storage.
- `pgrep -fa artisan serve` also matches the harness's own bash wrapper command line; match on the concrete binary args instead of trusting any single PID match.
- Vite dev is not needed for measurement, but `vendor/`, `node_modules/`, and `public/build/` must all exist — check those three paths before trusting a served page's built assets.

## Fix-layer guidance (color-contrast)

- Prefer a narrow override scoped to the active menu state over editing vendor code: e.g. a `.menu a[aria-current="page"]` / `.mary-active-menu` rule in `resources/css/app.css` anchored to `--color-base-content`/`--color-base-100` pairs — the toast/alert glass fix in the same file is the precedent (anchors to base-content, measured worst case 6.90:1 across all 7 themes).
- Compute candidate pairs offline from theme tokens (oklch → WCAG relative luminance) BEFORE proposing; acceptance gate is ≥4.5:1 across all 7 themes with hover, submenu header, and glass surfaces visually unchanged.
