# Lovelace card sizing, and what it means for `unifi-device-card`

Research notes. This document answers how Home Assistant sizes a Lovelace card, what a custom card
can control, and what stops `unifi-device-card` from using its own model layouts today.

**This document changes no code.** It ends with options and trade-offs.

---

## 0. Scope, sources, and how to read the citations

### Versions

| Item | Version | Note |
|---|---|---|
| Home Assistant Core | 2026.8.1 | The instance that produced the measurements in the task |
| Home Assistant frontend | **20260729.6** | Pinned by `homeassistant/components/frontend/manifest.json` at core tag `2026.8.1`: `"requirements": ["home-assistant-frontend==20260729.6"]` |
| Fork branch | `feat/poe-and-models` | Head `a8f5fcf`, based on `origin/develop` |

Every frontend citation below points at the tag **20260729.6**, the build that this instance runs.
The sizing files at frontend `dev` (commit `ea0ceec`, 2026-08-13) are byte-identical to that tag for
`types.ts`, `compute-card-grid-size.ts`, `hui-card.ts`, `hui-grid-section.ts`, `hui-panel-view.ts`,
`hui-card-layout-editor.ts`, `compute-card-size.ts`, `hui-iframe-card.ts`, `hui-clock-card.ts` and
`hui-button-card.ts`. The three files that differ (`hui-sections-view.ts`, `hui-masonry-view.ts`,
`hui-tile-card.ts`) differ only outside the sizing code.

### Citation form

- `frontend:src/panels/lovelace/…:NN` — file and line in home-assistant/frontend at tag 20260729.6.
- `fork:src/…:NN` — file and line in this repository at `a8f5fcf`.
- `devdocs:…` — home-assistant/developers.home-assistant, page path.

### Fact markers

- **[V]** Verified. Read directly in source, or read in the official developer documentation.
- **[I]** Inference. Arithmetic or reasoning built on top of **[V]** facts. Marked as such.

---

## 1. The `getGridOptions()` contract

### 1.1 The interface [V]

```ts
export interface LovelaceGridOptions {
  columns?: number | "full";
  rows?: number | "auto";
  max_columns?: number;
  min_columns?: number;
  min_rows?: number;
  max_rows?: number;
}

export interface LovelaceCard extends HTMLElement {
  …
  getCardSize(): number | Promise<number>;
  /** @deprecated Use `getGridOptions` instead */
  getLayoutOptions?(): LovelaceLayoutOptions;
  getGridOptions?(): LovelaceGridOptions;
  setConfig(config: LovelaceCardConfig): void;
}
```

`frontend:src/panels/lovelace/types.ts:59-78`

`getCardSize()` is **required**. `getGridOptions()` and `getLayoutOptions()` are both optional.

### 1.2 What Home Assistant does when a card implements neither [V]

```ts
public getElementGridOptions(): LovelaceGridOptions {
  if (!this._element) return {};
  if (this._element.getGridOptions) {
    const options = this._element.getGridOptions();
    // Some custom cards might return undefined, so we ensure we return an object
    return options || {};
  }
  if (this._element.getLayoutOptions) {
    const options = migrateLayoutToGridOptions(this._element.getLayoutOptions());
    return options;
  }
  return {};
}
```

`frontend:src/panels/lovelace/cards/hui-card.ts:79-99`

This confirms the first given fact. A card that implements only `getCardSize()` produces `{}`.
`unifi-device-card` is exactly this case: `getCardSize()` sits at `fork:src/unifi-device-card.js:223`,
and a search for `getGridOptions` or `getLayoutOptions` across `fork:src/*.js` returns no hit.

The empty object has a second, visible effect. The card layout editor shows a warning when the
element options are empty:

```ts
${this._defaultGridOptions && Object.keys(this._defaultGridOptions).length === 0
  ? html`<ha-alert alert-type="info">
      ${this.hass.localize("ui.panel.lovelace.editor.edit_card.layout.no_grid_support")}
    </ha-alert>`
  : nothing}
```

`frontend:src/panels/lovelace/editor/card-editor/hui-card-layout-editor.ts:92-103`

The English text is *"This card does not fully support resizing yet and may not display correctly with
custom sizes."* (`frontend:src/translations/en.json:9504`).

### 1.3 Defaults [V]

```ts
export const DEFAULT_GRID_SIZE = {
  columns: 12,
  rows: "auto",
} as CardGridSize;
```

`frontend:src/panels/lovelace/common/compute-card-grid-size.ts:30-33`

```ts
const rows = options.rows ?? DEFAULT_GRID_SIZE.rows;
const columns = options.columns ?? DEFAULT_GRID_SIZE.columns;
```

`frontend:src/panels/lovelace/common/compute-card-grid-size.ts:46-47`

So `{}` means **12 columns and automatic height**. The developer documentation states the same:
*"If you don't define this method, the card will take 12 columns and will ignore the rows of the
grid."* (`devdocs:docs/frontend/custom-ui/custom-card.md`, section *Sizing in sections view*).

There is **no** default for `min_columns`, `max_columns`, `min_rows` or `max_rows` in the code. The
developer documentation claims `min_columns` and `min_rows` default to `1`. That claim is a
description of the visible result, not of the code. See section 1.4.

### 1.4 Clamping [V]

```ts
const clampedRows =
  typeof rows === "string" ? rows : conditionalClamp(rows, minRows, maxRows);
const clampedColumns =
  typeof columns === "string" ? columns : conditionalClamp(columns, minColumns, maxColumns);
```

`frontend:src/panels/lovelace/common/compute-card-grid-size.ts:53-59`

```ts
export const conditionalClamp = (value: number, min?: number, max?: number) => {
  let result: number;
  result = min != null ? Math.max(value, min) : value;
  result = max != null ? Math.min(result, max) : result;
  return result;
};
```

`frontend:src/common/number/clamp.ts:4-9`

Three consequences:

1. An undefined bound clamps nothing.
2. The string literals `"full"` and `"auto"` bypass clamping completely. `max_columns` cannot cap
   `columns: "full"`, and `max_rows` cannot cap `rows: "auto"`.
3. A second clamp happens in CSS, not in JavaScript. See section 1.9.

### 1.5 The merge with the user configuration — `max_columns` caps the user [V]

```ts
public getGridOptions(): LovelaceGridOptions {
  const elementOptions = this.getElementGridOptions();
  const configOptions = this.getConfigGridOptions();
  const mergedConfig = { ...elementOptions, ...configOptions };
  return mergedConfig;
}
```

`frontend:src/panels/lovelace/cards/hui-card.ts:68-76`

The merge is per key, and the configuration wins per key. This creates a trap that matters for any
change to this card:

> If the card returns `max_columns: 12` and the user writes `grid_options: {columns: 24}`, the merged
> object is `{max_columns: 12, columns: 24}`. `conditionalClamp` then reduces the width back to 12.
> The user cannot make the card wider than the card allows, unless the user also overrides
> `max_columns`. **[V]** — this follows directly from the merge at `hui-card.ts:71-74` and the clamp at
> `compute-card-grid-size.ts:56-59`.

The editor enforces the same bound in its picker: `.columnMin=${gridOptions.min_columns}` and
`.columnMax=${gridOptions.max_columns}` (`frontend:…/hui-card-layout-editor.ts:144-145`), and the
picker clamps the drag target with the same helper
(`frontend:src/components/ha-grid-size-picker.ts:164-168`).

`min_columns` has the mirrored effect. It raises a user value that is too small.

### 1.6 `GRID_COLUMN_MULTIPLIER` and precise mode [V]

```ts
export const GRID_COLUMN_MULTIPLIER = 3;
```

`frontend:src/panels/lovelace/common/compute-card-grid-size.ts:4`

The constant has exactly two jobs.

**Job 1 — the migration factor.** The old grid was 4 columns wide. The new grid is 12 columns wide.
`migrateLayoutToGridOptions` multiplies the three column fields by 3 and passes the three row fields
through unchanged:

```ts
const gridOptions: LovelaceGridOptions = {
  columns: multiplyBy(options.grid_columns, GRID_COLUMN_MULTIPLIER),
  max_columns: multiplyBy(options.grid_max_columns, GRID_COLUMN_MULTIPLIER),
  min_columns: multiplyBy(options.grid_min_columns, GRID_COLUMN_MULTIPLIER),
  rows: options.grid_rows,
  max_rows: options.grid_max_rows,
  min_rows: options.grid_min_rows,
};
```

`frontend:src/panels/lovelace/common/compute-card-grid-size.ts:11-28`

`multiplyBy` passes a string through untouched, so `grid_columns: "full"` stays `"full"`
(`compute-card-grid-size.ts:6-9`).

**Job 2 — the editor step size.** The width slider moves in steps of 3, not 1:

```ts
.step=${this._preciseMode ? 1 : GRID_COLUMN_MULTIPLIER}
```

`frontend:…/hui-card-layout-editor.ts:146`

`isPreciseMode` is a pure test, not a setting:

```ts
export const isPreciseMode = (options: LovelaceGridOptions) =>
  typeof options.columns === "number" && options.columns % 3 !== 0;
```

`frontend:src/panels/lovelace/common/compute-card-grid-size.ts:40-41`

Precise mode turns itself **on** when a column count is not a multiple of 3
(`hui-card-layout-editor.ts:259-263`). When the user turns it **off** again, the editor rounds the
saved value **up** to the next multiple of 3 (`hui-card-layout-editor.ts:347-356`).

`isPreciseMode` runs against the card element options as well. A card that returns a default such as
`columns: 7` forces the editor into precise mode for that card
(`hui-card-layout-editor.ts:252-256`).

The developer documentation states the same preference as a recommendation: *"For the number of
columns, it's `highly` recommended to use multiple of 3 for the default value (`3`, `6`, `9` or
`12`)"* (`devdocs:docs/frontend/custom-ui/custom-card.md`).

**`GRID_COLUMN_MULTIPLIER` never scales what a card returns from `getGridOptions()`.** It applies
only to `getLayoutOptions()` output and to the editor slider.

### 1.7 The deprecated `getLayoutOptions()` [V]

`getLayoutOptions()` is checked only when `getGridOptions()` is absent
(`frontend:src/panels/lovelace/cards/hui-card.ts:82-97`). Its result passes through
`migrateLayoutToGridOptions`. The `console.warn` that told developers to migrate is commented out,
with the reason *"Disabled for now to avoid spamming the console"*
(`hui-card.ts:88-92`). The same migration runs on the user configuration key `layout_options`
(`hui-card.ts:102-110`), and the layout editor deletes `layout_options` whenever it saves
(`hui-card-layout-editor.ts:376-378`).

Do not implement `getLayoutOptions()` in new code. It buys nothing that `getGridOptions()` does not
buy, and it costs a multiplication by 3.

### 1.8 How `getGridOptions()` and `getCardSize()` interact [V]

They do not interact. They serve different views, and neither reads the other.

- `getCardSize()` is consumed only through `computeCardSize()`
  (`frontend:src/panels/lovelace/common/compute-card-size.ts:5-23`), and the only view that calls it
  is the masonry view (`frontend:src/panels/lovelace/views/hui-masonry-view.ts:222`).
- `getGridOptions()` is consumed by the grid section
  (`frontend:src/panels/lovelace/sections/hui-grid-section.ts:109-111`), by the view footer
  (`frontend:src/panels/lovelace/views/hui-view-footer.ts:151-152`, rows only), and by the layout
  editor (`hui-card-layout-editor.ts:231`).

`computeCardSize` wraps the call in a 500 ms timeout and falls back to `1` on error or timeout
(`compute-card-size.ts:10-15`). Keep `getCardSize()` cheap and synchronous.

### 1.9 How wide a section grid is [V]

The section grid is built from three CSS custom properties.

```css
:host {
  --base-column-count: 12;
  --row-gap:     var(--ha-section-grid-row-gap, 8px);
  --column-gap:  var(--ha-section-grid-column-gap, 8px);
  --row-height:  var(--ha-section-grid-row-height, 56px);
}
.container {
  --grid-column-count: calc(var(--base-column-count) * var(--column-span, 1));
  display: grid;
  grid-template-columns: repeat(var(--grid-column-count), minmax(0, 1fr));
  grid-auto-rows: auto;
}
.card {
  grid-row: span var(--row-size, 1);
  grid-column: span min(var(--column-size, 1), var(--grid-column-count));
}
.card.fit-rows {
  height: calc((var(--row-size, 1) * (var(--row-height) + var(--row-gap))) - var(--row-gap));
}
.card.full-width { grid-column: 1 / -1; }
```

`frontend:src/panels/lovelace/sections/hui-grid-section.ts:214-263`

`--column-size` and `--row-size` come straight from the computed grid size, and only when they are
numbers:

```ts
const { rows, columns } = computeCardGridSize(gridOptions);
…
style=${styleMap({
  "--column-size": typeof columns === "number" ? columns : undefined,
  "--row-size":    typeof rows    === "number" ? rows    : undefined,
})}
class="card ${classMap({
  "fit-rows":   typeof rows === "number",
  "full-width": columns === "full",
})}"
```

`frontend:src/panels/lovelace/sections/hui-grid-section.ts:109-128`

`--column-span` is set by the view, and the view clamps it first:

```ts
const columnSpan = Math.min(section.config.column_span || 1, contentColumnCount);
…
style=${styleMap({ "--column-span": columnSpan, "--row-span": rowSpan })}
```

`frontend:src/panels/lovelace/views/hui-sections-view.ts:271-283`

`contentColumnCount` comes from the view width and from the view configuration:

```ts
const maxColumns = this._config?.max_columns ?? DEFAULT_MAX_COLUMNS;   // DEFAULT_MAX_COLUMNS = 4
const maxColumnCount = clamp(this._columnsController.value ?? 1, 1, maxColumns);
…
const columnCount = Math.max(Math.min(this._maxColumns, totalSectionCount), 1);
const contentColumnCount = hasSidebar ? Math.max(1, columnCount - 1) : columnCount;
```

`frontend:src/panels/lovelace/views/hui-sections-view.ts:41, 161-166, 195-202`

and `_columnsController` measures the real width against `--column-min-width` (320 px by default):

```ts
const columns = Math.floor((totalWidth - padding + columnGap) / (minColumnWidth + columnGap));
return Math.max(columns, 1);
```

`frontend:src/panels/lovelace/views/hui-sections-view.ts:103-106`, with
`--column-min-width: var(--ha-view-sections-column-min-width, 320px)` at line 542 and
`--column-max-width: var(--ha-view-sections-column-max-width, 500px)` at line 541.

**The formula.** Put together **[I]**:

```
view columns   = clamp( floor((viewWidth - padding + 32) / (320 + 32)), 1, view.max_columns ?? 4 )
section tracks = 12 × min(section.column_span ?? 1, view columns)
card span      = min( computeCardGridSize(options).columns , section tracks )
```

This reproduces the measurement given in the task. A section with `column_span: 4` in a view with
`max_columns: 4` on a wide screen gives `--column-count: 4` and `12 × 4 = 48` tracks, and a card with
`{}` options spans `min(12, 48) = 12` tracks — exactly one quarter of the section, which is one view
column. The observed 446 px is the width of 12 tracks plus 11 gaps at that viewport **[I]**.

**Three practical rules that follow [I]:**

1. A card that returns nothing is always exactly **one view column** wide, whatever `column_span` is.
2. A `columns` number is relative to the **section**, not to the view. The same `columns: 24` fills a
   `column_span: 2` section and half fills a `column_span: 4` section.
3. Returning a number larger than the grid is **safe**. The CSS `min()` at
   `hui-grid-section.ts:246` clamps the span to the track count. No overflow happens. Only
   `columns: "full"` is span-independent by design.

**Rows.** `rows: "auto"` sets no `--row-size` and adds no `fit-rows` class, so the card keeps its
natural height inside one auto-sized grid row. A numeric `rows: N` fixes the height to
`N × (56 + 8) − 8` px. For `N = 2` that is 120 px, which matches the worked example in the developer
documentation.

### 1.10 Which Home Assistant version introduced it [V]

| Fact | Evidence |
|---|---|
| `getGridOptions` absent at frontend `20241002.0` and `20241002.4` | GitHub contents API on `src/panels/lovelace/types.ts` at those tags: zero matches |
| Present at frontend `20241106.0` | Same query at that tag: two matches |
| Core `2024.10.0` pins `home-assistant-frontend==20241002.2` | `homeassistant/components/frontend/manifest.json` at core tag `2024.10.0` |
| Core `2024.11.0` pins `home-assistant-frontend==20241106.0` | Same file at core tag `2024.11.0` |
| Frontend PR that converted all cards | [home-assistant/frontend#22676](https://github.com/home-assistant/frontend/pull/22676) *"Use grid options instead of layout options for all cards."*, merged 2024-11-05 |

**`getGridOptions()` first shipped in Home Assistant 2024.11.0.**

For background: `getLayoutOptions()` arrived earlier, through
[frontend#20001](https://github.com/home-assistant/frontend/pull/20001) (merged 2024-03-07), which
lands in core `2024.3.x` (`2024.3.0` pins `20240306.0`). The `return options || {}` guard for custom
cards that return `undefined` came later, through
[frontend#25760](https://github.com/home-assistant/frontend/pull/25760) (merged 2025-06-12).

**Compatibility note for this fork.** A card that adds `getGridOptions()` stays safe on older
frontends. Any Home Assistant before 2024.11 simply never calls the method, because the property test
at `hui-card.ts:82` fails. No error results.

---

## 2. Views other than sections

### 2.1 Masonry — no lever at all [V]

The masonry view never calls `getGridOptions()`. A search for `getGridOptions` across
`frontend:src/` returns no hit inside `hui-masonry-view.ts`. The only card call it makes is:

```ts
const cardSizeProm = computeCardSize(el);
…
this._addCardToColumn(columnElements[getColumnIndex(columnSizes, cardSize as number)], index, …);
```

`frontend:src/panels/lovelace/views/hui-masonry-view.ts:222, 233`

`getColumnIndex` uses the size only to choose **which column** a card goes into:

```ts
const getColumnIndex = (columnSizes: number[], size: number) => {
  let minIndex = 0;
  for (let i = 0; i < columnSizes.length; i++) {
    if (columnSizes[i] < 5) { minIndex = i; break; }
    if (columnSizes[i] < columnSizes[minIndex]) { minIndex = i; }
  }
  columnSizes[minIndex] += size;
  return minIndex;
};
```

`frontend:src/panels/lovelace/views/hui-masonry-view.ts:20-35`

**Column width is fixed and card-independent** [V]:

```css
.column { flex: 1 0 0; max-width: 500px; min-width: 0; }
@media (max-width: 599px) { .column { max-width: 600px; } }
```

`frontend:src/panels/lovelace/views/hui-masonry-view.ts:305-307, 335-337`

The column count comes only from media queries and from the sidebar state:

```ts
this._mqls = [300, 600, 900, 1200].map((width) => window.matchMedia(`(min-width: ${width}px)`));
…
const newColumns = Math.max(1, matchColumns - Number(!this.narrow && this.hass!.dockedSidebar === "docked"));
```

`frontend:src/panels/lovelace/views/hui-masonry-view.ts:104, 269-273`

**Answer: there is no lever.** In masonry a card gets a column of `min(500px, viewport share)` and
cannot ask for more. `getCardSize()` changes vertical packing only. A card can still measure its own
width with a `ResizeObserver` and adapt its content — that is what this card already does — but it
cannot change the box it lives in.

One dead end worth recording. The card dispatches `iron-resize` after it measures itself
(`fork:src/unifi-device-card.js:258`). **Nothing in the frontend listens to `iron-resize` any more.**
A repository-wide search across frontend 20260729.6 returns zero hits. The event was removed by
[frontend#18813](https://github.com/home-assistant/frontend/pull/18813) *"Cleanup iron resize event"*
(2023-11-29). That dispatch is a no-op today.

### 2.2 Panel view [V]

`hui-panel-view.ts` calls neither `getGridOptions()` nor `getCardSize()`. It renders one card and sets
`card.layout = "panel"` (`frontend:src/panels/lovelace/views/hui-panel-view.ts:107`). The host is
`display: block; height: 100%` and the view strips the card border, radius and shadow
(`hui-panel-view.ts:122-136`). A card in panel view is full width by construction.

The card can read this: `hui-card` forwards the string to the element as `element.layout`
(`frontend:src/panels/lovelace/cards/hui-card.ts:135, 229`) and also sets the legacy
`element.isPanel = this.layout === "panel"` (`hui-card.ts:231`). Built-in cards branch on it, for
example `const ignoreAspectRatio = this.layout === "panel" || this.layout === "grid";`
(`frontend:src/panels/lovelace/cards/hui-iframe-card.ts:71`).

### 2.3 Sidebar view and other legacy types [V]

`hui-sidebar-view.ts` contains no `computeCardSize`, no `getCardSize` and no `getGridOptions`. It is a
two-column flex layout with `max-width: 1620px` on the main area and `max-width: 380px` on the sidebar
(`frontend:src/panels/lovelace/views/hui-sidebar-view.ts:217, 224`). No card-side lever exists.

`getViewType()` resolves an unset `type` to `panel` when `panel: true`, to `sections` when `sections`
is present, to `masonry` when `cards` is present, and to `sections` otherwise
(`frontend:src/panels/lovelace/views/get-view-type.ts`).

### 2.4 Stack cards swallow grid options [V]

None of `hui-stack-card.ts`, `hui-vertical-stack-card.ts`, `hui-horizontal-stack-card.ts`,
`hui-grid-card.ts` or `hui-conditional-card.ts` implements `getGridOptions()` or forwards a child's
options. Each implements `getCardSize()` only. A `unifi-device-card` nested inside a stack therefore
has **no** influence on its own grid size — the stack decides.

### 2.5 The view footer uses rows only [V]

```ts
const gridOptions = card.getGridOptions();
const { rows } = computeCardGridSize(gridOptions);
```

`frontend:src/panels/lovelace/views/hui-view-footer.ts:151-152`. The footer discards `columns`.

---

## 3. Reference cards, and when Home Assistant re-queries

### 3.1 The patterns worth copying [V]

**`hui-tile-card.ts:120-142` — options computed from the card configuration:**

```ts
public getGridOptions(): LovelaceGridOptions {
  const columns = 6;
  let min_columns = 6;
  let rows = 1;
  const featurePosition = this._config && this._featurePosition(this._config);
  const featuresCount = this._config?.features?.length || 0;
  if (featuresCount) {
    if (featurePosition === "inline") { min_columns = 12; }
    else { rows += featuresCount; }
  }
  if (this._config?.vertical) { rows++; min_columns = 3; }
  return { columns, rows, min_columns, min_rows: rows };
}
```

Note the idiom: `rows` and `min_rows` carry the **same** value. The card asks for a height and refuses
anything smaller, but leaves the ceiling open.

**`hui-iframe-card.ts:129-136` — the "I am wide" pattern:**

```ts
public getGridOptions(): LovelaceGridOptions {
  return { columns: "full", rows: 4, min_columns: 3, min_rows: 2 };
}
```

This is the closest reference for `unifi-device-card`. `columns: "full"` is section-span-independent
and needs no measurement.

**`hui-clock-card.ts:46-…` — a lookup table over configuration variants**, one branch per
`clock_style` × `clock_size` pair, each returning its own `{min_rows, rows, columns, min_columns}`,
and `min_rows` raised by one when a title is present.

**`hui-button-card.ts:117-134` — two shapes, chosen by content:**

```ts
public getGridOptions(): LovelaceGridOptions {
  if (this._config?.show_icon && (this._config?.show_name || this._config?.show_state)) {
    return { rows: 2, columns: 6, min_columns: 2, min_rows: 2 };
  }
  return { rows: 1, columns: 3, min_columns: 2, min_rows: 1 };
}
```

**Common properties of all four [V]:**

- Every one is **synchronous and pure**. Each reads only `this._config`.
- **None reads `this.hass`.**
- **None measures the DOM.**
- None caches. The method is cheap enough to run on every render.

### 3.2 Is there a signal a card must fire? [V]

**There is no dedicated event, and none is needed for the common case.** Home Assistant re-queries
`getGridOptions()` far more often than a card needs.

The call sits inside the grid section `render()`, inside a `repeat()` body:

```ts
(_cardConfig, idx) => {
  const card = this.cards![idx];
  card.layout = "grid";
  const gridOptions = card.getGridOptions();
  const { rows, columns } = computeCardGridSize(gridOptions);
```

`frontend:src/panels/lovelace/sections/hui-grid-section.ts:106-111`

The `repeat` key controls DOM reuse only. The body runs for every card on every render. So the
question becomes: when does `hui-grid-section` render?

`hui-grid-section` declares `hass`, `lovelace`, `index`, `viewIndex`, `isStrategy`, `cards` and
`importOnly` as reactive properties, plus the `_config` and `_dragging` state
(`hui-grid-section.ts:43-60`). Any change re-renders. Two of those change often:

**Path A — every `hass` update.** `hui-view.update()` forwards `hass` to each section on every change
(`frontend:src/panels/lovelace/views/hui-view.ts:215-232`), and `hui-section.update()` forwards it on
to the layout element:

```ts
if (changedProperties.has("hass")) {
  this._cards.forEach((element) => { element.hass = this.hass; });
  this._layoutElement.hass = this.hass;
```

`frontend:src/panels/lovelace/sections/hui-section.ts:170-174`

**Path B — the `card-updated` event.** `hui-section` listens on each `hui-card` and replaces the array
identity, which re-renders the grid section:

```ts
element.addEventListener("card-updated", (ev: Event) => {
  ev.stopPropagation();
  this._cards = [...this._cards];
});
```

`frontend:src/panels/lovelace/sections/hui-section.ts:80-83`

`hui-card` fires `card-updated` itself on a configuration update, on `ll-upgrade`, on `ll-rebuild` and
on a preview change (`hui-card.ts:118, 151, 160, 219`).

**The escape hatch, if one is ever needed [V].** The listener above is registered on the `hui-card`
element, and `hui-card.createRenderRoot()` returns `this` (`hui-card.ts:47-49`), so the custom card is
a light-DOM child of `hui-card`. A `card-updated` event that bubbles from the custom card reaches that
listener. Home Assistant's own `fireEvent` defaults to `bubbles: true, composed: true`
(`frontend:src/common/dom/fire_event.ts:76-82`). A custom card can therefore force an immediate
re-query with:

```js
this.dispatchEvent(new Event("card-updated", { bubbles: true, composed: true }));
```

This mechanism is verified in source but is **not documented** as public API. Treat it as a last
resort. `ll-rebuild` is a heavier alternative that rebuilds the element, and its listener is
registered `{ once: true }` (`hui-card.ts:155-163`), so it works one time only.

**The editor re-queries separately.** `hui-card-layout-editor` builds its own hidden `hui-card`,
listens for `card-updated`, and refreshes the defaults through `getElementGridOptions()`
(`hui-card-layout-editor.ts:207-232`).

### 3.3 What this means for a measuring card [I]

A card that wants to derive `getGridOptions()` from a **measured** width faces an ordering problem.
`getGridOptions()` decides the width, and the measurement reads the result of that decision. Feeding
the measurement back creates a loop: narrower measurement, narrower request, narrower measurement.

The safe design is the one all four reference cards use: derive grid options from the **configuration
and the device model only**, never from a measurement, and let the card's own internal repacking
handle whatever width it actually receives.

---

## 4. The card's internal conflict

### 4.1 The pipeline, in order [V]

1. `getDeviceLayout(device, discoveredPortsRaw)` returns the model layout, which carries `kind`,
   `frontStyle`, `rows`, `portCount`, `specialSlots` and `theme`
   (`fork:src/model-registry.js:1440-1537`). Registry hits, the gateway fallback and the generic
   switch fallback all pass through `applyRj45LayoutHints`
   (`model-registry.js:1466, 1471, 1496, 1504, 1528`).
2. `applyRj45LayoutHints` adds one boolean:
   `rj45_odd_even: isSwitchOrGateway && !isExcluded && numberedRj45Count > 8`
   (`fork:src/model-registry.js:70-80`).
3. **The override** (`fork:src/helpers.js:2045-2052`):

   ```js
   const configuredPortsPerRow = Number.parseInt(cardConfig?.ports_per_row, 10);
   const hasConfiguredPortsPerRow = Number.isFinite(configuredPortsPerRow) && configuredPortsPerRow > 0;

   if (hasConfiguredPortsPerRow) {
     layout = applyPortsPerRowOverride(layout, configuredPortsPerRow);
   } else if (type === "switch") {
     layout = applyPortsPerRowOverride(layout, 8);
   }
   ```

4. `applyPortsPerRowOverride` rebuilds `rows` from `portCount` and replaces `frontStyle`
   (`fork:src/model-registry.js:82-104`).
5. `filterPortsByLayout` keeps only ports that appear in `rows` or in `specialSlots`
   (`fork:src/helpers.js:1981-1991`).
6. At render time `_buildEffectiveRows` repacks the rows against the measured width
   (`fork:src/unifi-device-card.js:1051-1086`), and `_applyOddEvenRows` can reorder them
   (`fork:src/unifi-device-card.js:1107-1129`).

### 4.2 Finding 1 — the hardcoded `8` overwrites every switch model [V]

`applyPortsPerRowOverride` does not merge. It replaces:

```js
export function applyPortsPerRowOverride(layout, portsPerRow) {
  if (!portsPerRow || portsPerRow < 1 || layout.kind !== "switch") return layout;
  const portCount = layout.portCount;
  const newRows = [];
  for (let i = 0; i < portCount; i += portsPerRow) {
    newRows.push(range(i + 1, Math.min(i + portsPerRow, portCount)));
  }
  const frontStyleMap = { 4: "grid-4", 6: "six-grid", 7: "ultra-row", 8: "eight-grid", 12: "quad-row" };
  return { ...layout, rows: newRows, frontStyle: frontStyleMap[portsPerRow] || `grid-${portsPerRow}` };
}
```

`fork:src/model-registry.js:82-104`

So for **every** device classified as a switch, and unless the user sets `ports_per_row`, the declared
`rows` and the declared `frontStyle` are thrown away and replaced by rows of 8 and
`frontStyle: "eight-grid"`.

The live example is the head commit of this branch. `a8f5fcf` changed `US24P250` to
`frontStyle: "quad-row"` with `rows: [range(1, 12), range(13, 24)]`
(`fork:src/model-registry.js:327-336`). Step 3 above turns that back into
`[1-8], [9-16], [17-24], [25-26]` with `frontStyle: "eight-grid"` before anything renders. **The
declared layout never reaches the DOM.** The commit message already suspected the width path. The
actual blocker is one line earlier.

The guard `layout.kind !== "switch"` means gateways and access points are untouched. Only switches are
affected.

### 4.3 Finding 2 — `frontStyle` does not control columns at all [V]

This is the most important finding for the question as asked, and it is easy to miss.

The CSS declares `--udc-cols` in twelve places, one per front style:

```css
.frontpanel.single-row .port-row,
.frontpanel.gateway-single-row .port-row { --udc-cols: 8; }
.frontpanel.dual-row .port-row      { --udc-cols: 8; }
.frontpanel.gateway-rack .port-row  { --udc-cols: 8; }
.frontpanel.gateway-compact .port-row { --udc-cols: 5; }
.frontpanel.six-grid .port-row      { --udc-cols: 6; }
.frontpanel.eight-grid .port-row    { --udc-cols: 8; }
.frontpanel.quad-row .port-row      { --udc-cols: 12; }
.frontpanel.ultra-row .port-row     { --udc-cols: 7; }
.frontpanel.grid-4 .port-row        { --udc-cols: 4; }
.frontpanel.grid-5 .port-row        { --udc-cols: 5; }
.frontpanel.grid-9 .port-row        { --udc-cols: 9; }
.frontpanel.grid-10 .port-row       { --udc-cols: 10; }
```

`fork:src/unifi-device-card.js:1628-1675`

and the renderer writes it inline twice (`fork:src/unifi-device-card.js:2533, 2725`).

**No rule reads it.** `grep -rn "udc-cols" src/` returns 14 writes and zero occurrences of
`var(--udc-cols`. The same holds for the built bundle: `dist/unifi-device-card.js` contains 14
occurrences of `udc-cols` and 0 occurrences of `var(--udc-cols`.

The actual layout is plain flex wrap:

```css
.port-row { display: flex; flex-wrap: wrap; gap: 4px 6px; width: 100%; align-items: flex-start; }
.port { … width: calc(var(--udc-port-size) - 2px); flex: 0 0 calc(var(--udc-port-size) - 2px); … }
```

`fork:src/unifi-device-card.js:1608-1613, 1966-1973`

**Consequences [V/I]:**

- Ports per visual line are decided by flex wrap at the measured width, not by `frontStyle`. **[V]**
- `frontStyle` still controls the panel background family and the access-point rendering
  (`fork:src/unifi-device-card.js:1677-1688, 2580-2581`). **[V]**
- Each entry in `layout.rows` becomes one `.port-row` element
  (`fork:src/unifi-device-card.js:2714-2728`), so `rows` **does** control where the row breaks are —
  which is exactly what the `US24P250` commit wanted. **[V]**
- The gap between wrapped lines inside one `.port-row` is 4 px, and the gap between two `.port-row`
  elements is 3 px (`.frontpanel { gap: 3px }`, `fork:src/unifi-device-card.js:1577-1580`). The two
  are close enough that an unwanted wrap looks almost the same as a declared row break. **[I]**

So the question *"how do I make `frontStyle` take effect"* has a two-part answer: `rows` is the field
that matters, and `frontStyle` has no column effect today. Any fix should either wire `--udc-cols` up
or accept that it is decoration.

### 4.4 Finding 3 — the `frontStyle` guard in `_shouldUseOddEvenRows` is unreachable [V]

```js
_shouldUseOddEvenRows(ctx, numbered) {
  if (!ctx || (ctx.type !== "switch" && ctx.type !== "gateway")) return false;
  if (this._config?.force_sequential_ports === true) return false;
  if (ctx?.layout?.rj45_odd_even === true) return true;
  if (ctx?.layout?.rj45_odd_even === false) return false;
  const frontStyle = String(ctx?.layout?.frontStyle || "");
  if (["dual-row", "six-grid", "eight-grid", "quad-row"].includes(frontStyle)) return false;
  …
}
```

`fork:src/unifi-device-card.js:1089-1104`

`applyRj45LayoutHints` always writes `rj45_odd_even` as a real boolean, never `undefined`
(`fork:src/model-registry.js:70-80`), and every switch and gateway path in `getDeviceLayout` runs it.
`applyPortsPerRowOverride` spreads `...layout`, so the flag survives the override. As a result one of
the two tests at lines 1093-1094 always returns, and the `frontStyle` list at line 1100 never runs.

Its comment describes the intended protection for `quad-row`. That protection does not exist. This
matters directly for the `US24P250` change: once the hardcoded 8 is gone, the odd/even reorder will
still fire, because `rj45_odd_even` is `true` for 24 numbered ports.

Note that for `US24P250` the odd/even reorder is the **wanted** result. The physical panel is odd on
top and even below, and `_applyOddEvenRows` turns `[1..12], [13..24]` into odds then evens
(`fork:src/unifi-device-card.js:1107-1129`). **[I]** — this reads correctly from the code, but it is
not confirmed against a rendered card.

### 4.5 Finding 4 — `portCount` includes special slots [V]

The registry comment is explicit:

```
// rows: only contains *LAN* RJ45 port numbers.
// portCount: total physical ports including special slots.
```

`fork:src/model-registry.js:118-119`

`applyPortsPerRowOverride` builds its rows from `portCount`
(`fork:src/model-registry.js:85-89`), so it pulls the SFP port numbers into the numbered grid. For
`US24P250` (`portCount: 26`, two SFP slots) the override produces a fourth row `[25, 26]`. Those two
slots are then removed again at render time, because `visibleNumbered` excludes any port that a
special slot already claims (`fork:src/unifi-device-card.js:2699-2704`), and the row map filters the
misses out (`fork:src/unifi-device-card.js:2717-2719`). The row is dropped only because empty rows are
filtered (`fork:src/unifi-device-card.js:2728`).

The override therefore works today, but by accident rather than by design. Any change that keeps it
should be aware of this.

### 4.6 Does `_buildEffectiveRows` already degrade gracefully? Yes — with one gap [V]

```js
_buildEffectiveRows(ctx, numbered) {
  const baseRows = (ctx?.layout?.rows || []).map((row) => [...row]);
  …
  const fitCols = this._maxFittableColumns();

  if (!baseRows.length) {
    if (!Number.isFinite(fitCols) || extraPorts.length <= fitCols) return [extraPorts];
    const packed = [];
    for (let i = 0; i < extraPorts.length; i += fitCols) packed.push(extraPorts.slice(i, i + fitCols));
    return packed;
  }

  const rows = baseRows.map((row) => [...row]);
  if (extraPorts.length) rows[rows.length - 1].push(...extraPorts);
  const widestRow = rows.reduce((max, row) => Math.max(max, row.length), 0);
  if (!Number.isFinite(fitCols) || widestRow <= fitCols) return rows;

  const packedRows = [];
  for (let i = 0; i < orderedPorts.length; i += fitCols) packedRows.push(orderedPorts.slice(i, i + fitCols));
  return packedRows;
}
```

`fork:src/unifi-device-card.js:1051-1086`

**The repack is real. The declared rows do degrade on their own.** When the widest declared row does
not fit the measured panel, the function discards the declared grouping and repacks every numbered
port into rows of `fitCols`.

The measurement matches the CSS exactly:

```js
const portSize = this._portSize();          // default 36, clamped to 24…52
const horizontalPadding = 24;
const gap = 6;
const slotWidth = Math.max(1, portSize - 2);
const available = panelContentWidth > 0 ? panelContentWidth : Math.max(180, hostWidth - horizontalPadding);
return Math.max(1, Math.floor((available + gap) / (slotWidth + gap)));
```

`fork:src/unifi-device-card.js:430-449`, against `.port { flex: 0 0 calc(var(--udc-port-size) - 2px) }`
and `.port-row { gap: 4px 6px }` (`fork:src/unifi-device-card.js:1608-1613, 1972-1973`).

`_measuredFrontPanelContentWidth()` subtracts the real computed padding from the real panel width
(`fork:src/unifi-device-card.js:417-428`), so the fallback constant `24` is used only before the first
paint.

**Three gaps to know before relying on the repack:**

1. **The first render has no measurement.** Before the panel exists, `_measuredFrontPanelContentWidth`
   returns 0 and `_maxFittableColumns` returns `Infinity`
   (`fork:src/unifi-device-card.js:437-439`). The declared rows render unpacked for one frame.
   `_finalizeRender()` then measures in a `requestAnimationFrame` and re-renders once the panel width
   moved by at least 1 px (`fork:src/unifi-device-card.js:261-272`). A `ResizeObserver` on the host
   covers later width changes (`fork:src/unifi-device-card.js:170-179`). **[V]**
2. **An explicit `ports_per_row` disables the repack entirely.** `_maxFittableColumns` returns
   `Infinity` in that case, by design and with a comment
   (`fork:src/unifi-device-card.js:431-434`). A user who sets `ports_per_row` and then narrows the
   card gets overflow. That behaviour exists today and is unchanged by anything discussed here. **[V]**
3. **The repack flattens the model grouping.** When the declared rows do not fit, the result is a
   uniform `fitCols` grid, not a proportionally shrunken version of the declared layout. **[V]**

**Direct answer to the question in the task:** yes, removing or narrowing the hardcoded `8` is safe
with respect to *overflow*. The repack catches every case where the declared rows are too wide, on
every render after the first frame, in masonry and in sections alike. What it does not preserve is the
*declared grouping* — a narrow card shows the model layout only when the model layout fits.

### 4.7 The width numbers, concretely [I]

All arithmetic below uses the default `port_size` of 36 px
(`fork:src/unifi-device-card.js:369-373`), so each slot is 34 px wide with a 6 px gap, a 40 px pitch.

**Panel width needed for K ports in one row:**

```
content = K × 34 + (K − 1) × 6
card    = content + 28      (frontpanel padding 12px 14px 10px → 28px horizontal)
```

| K | content | card width |
|---|---|---|
| 6 | 234 px | 262 px |
| 8 | 314 px | 342 px |
| 12 | 474 px | **502 px** |
| 16 | 634 px | 662 px |
| 24 | 954 px | 982 px |

**Width a card actually receives in a sections view.** A view column is between
`--column-min-width` 320 px and `--column-max-width` 500 px
(`frontend:src/panels/lovelace/views/hui-sections-view.ts:541-542`). Inside a `column_span: 1`
section, a track is `(W − 11 × 8) / 12`, so:

| View column | Track | 12 columns (the `{}` default) | Ports per row from `_maxFittableColumns` |
|---|---|---|---|
| 320 px | 19.3 px | 320 px | `floor((292 + 6) / 40)` = **7** |
| 400 px | 26.0 px | 400 px | `floor((372 + 6) / 40)` = **9** |
| 500 px | 34.3 px | 500 px | `floor((472 + 6) / 40)` = **11** |

**The conclusion that matters:** 12 ports in one row need 502 px of card width, and one view column is
at most 500 px. **A 24-port switch drawn as two rows of twelve never fits inside one view column, at
any viewport size.** It needs `columns: "full"` inside a section with `column_span ≥ 2`, or a wider
`--ha-view-sections-column-max-width`, or a smaller `port_size`.

That is the honest boundary of what `getGridOptions()` can achieve here. Grid options can ask for the
widest box available. They cannot make a fixed-pitch port grid fit a box that is too small.

### 4.8 Enumerated options for the hardcoded `8`

All options touch the same block, `fork:src/helpers.js:2048-2052`.

---

**Option A — delete the `else if` branch.**

```js
if (hasConfiguredPortsPerRow) {
  layout = applyPortsPerRowOverride(layout, configuredPortsPerRow);
}
```

- **Gain.** Every declared `rows` and `frontStyle` takes effect. The `US24P250` commit starts working.
  Smallest possible diff — three lines removed.
- **Cost.** Visible change for every switch user on a wide card. A `USW-48` declared as `quad-row`
  now shows 4 rows of 12 instead of 6 rows of 8, whenever 12 ports fit (≥ 502 px). Narrow cards are
  unaffected, because the repack takes over.
- **Risk.** The generic fallback `defaultSwitchLayout` produces `single-row` with **all** ports in one
  row for any port count that is not 8, 16, 24 or 48 (`fork:src/model-registry.js:54-67`). For an
  unknown 28-port switch that is one row of 28 = 1,090 px. The repack fixes it after the first frame,
  but the first frame overflows. Today the hardcoded 8 hides this.
- **Verdict.** Correct in intent, and the first-frame overflow is the only real objection.

---

**Option B — apply the default only when the model declares nothing useful.**

```js
} else if (type === "switch" && !(layout?.rows?.length > 1)) {
  layout = applyPortsPerRowOverride(layout, 8);
}
```

- **Gain.** Every multi-row model keeps its declared layout. Every `single-row` fallback keeps today's
  behaviour exactly, including the first frame. `defaultSwitchLayout` already returns multi-row shapes
  for 16, 24 and 48 ports, so those adopt their declared grouping too.
- **Cost.** One extra condition, and a new implicit rule: *"one declared row means undecided"*. That
  rule is true today for every registry entry, but nothing enforces it.
- **Risk.** Low. No path loses a safety net.
- **Verdict.** The most defensible minimal change. It is strictly narrower than Option A and fixes
  the same bug.

---

**Option C — use the default only while the width is unknown.**

Move the decision into `_buildEffectiveRows`, and use 8 as the pre-measurement `fitCols` instead of
`Infinity`.

- **Gain.** Solves the first-frame overflow for every layout, declared or generated.
- **Cost.** Larger diff, and it touches the render path rather than the data path. It also changes
  behaviour when `ports_per_row` is set, unless the `Infinity` short-circuit is kept ahead of it.
- **Risk.** Medium. `_buildEffectiveRows` also feeds `_estimateCardSize`
  (`fork:src/unifi-device-card.js:240`), so a changed pre-measurement result changes `getCardSize()`
  and therefore masonry packing.
- **Verdict.** A good complement to Option B, not a replacement for it.

---

**Option D — make 8 a floor rather than an override.**

Keep the declared rows, and split only rows wider than 8.

- **Gain.** No row ever exceeds 8 before measurement.
- **Cost.** It still destroys `quad-row`, which is the exact case that motivated this work. A row of
  12 becomes 8 + 4.
- **Verdict.** Rejected. It does not solve the stated problem.

---

**Option E — gate on `frontStyle`.**

```js
} else if (type === "switch" && layout?.frontStyle === "single-row") {
```

- **Gain.** Same effect as Option B for every current registry entry.
- **Cost.** It couples a data decision to a presentation field that, per section 4.3, currently has no
  layout effect. It also misses `gateway-generic` and other fallbacks.
- **Verdict.** Works, but Option B expresses the same intent against the field that actually matters.

---

## 5. What `AGENTS.md` and CI require of any change

Read from `fork:AGENTS.md` and `fork:.github/workflows/`.

### 5.1 Branch policy [V]

- `develop` is the working branch. Do not open, target or base changes on `main` unless the user asks
  (`AGENTS.md:23-27`). `main` is the release branch.
- The current branch `feat/poe-and-models` is based on `origin/develop`, which fits.

### 5.2 `/dist` policy [V]

- `/src` is the source of truth. `/dist` is generated (`AGENTS.md:37-48`).
- Do not hand-edit `/dist` (`AGENTS.md:20, 46`).
- CI rebuilds and commits it. `build.yml` runs `npm run build` and then
  `git add dist/unifi-device-card.js` followed by a `build: update dist [skip ci]` commit, on pushes to
  `main` and `develop` that touch `src/**`, `build.js`, `package.json`, `package-lock.json` or the
  workflow itself.
- In pull-request review, `/dist` changes are expected artifacts and must not be described as the
  meaningful diff (`AGENTS.md:50-55`).

### 5.3 Size and compatibility expectations [V]

- Prefer small, backwards-compatible changes (`AGENTS.md:18`).
- Prefer `/src` changes (`AGENTS.md:19`).
- Do not refactor unrelated code, and match the existing style (`AGENTS.md:29-35`).
- Keep configuration options backwards compatible. New options must be optional with sensible
  defaults (`AGENTS.md:82-84`).
- Prefer graceful fallbacks over hard failures, and avoid noisy errors for missing optional entities
  (`AGENTS.md:85-86`).
- Keep editor behaviour stable, and preserve translations unless user-facing text really changes
  (`AGENTS.md:87-88`).
- Do not touch `/screenshots` (`AGENTS.md:21, 70-75`). If a change alters the rendering, mention that
  the owner can need to refresh screenshots by hand — do not refresh them.

### 5.4 Documentation rules [V]

- Update `README.md` **only** when installation, usage, configuration, features or visible behaviour
  changes (`AGENTS.md:65`). Any change in section 4.8 alters visible behaviour, so the README
  configuration section for `ports_per_row` needs a look.
- Update `CHANGELOG.md` when the change belongs in release notes (`AGENTS.md:66`). Keep entries
  concise and factual (`AGENTS.md:67`). The existing format is
  `## [v0.7.92-dev]` followed by `### 🐛 Bug Fixes` and `### ✨ Improvements` bullet lists.
- The release workflows parse the changelog. `release.yml` extracts the block for the version with an
  `awk` script that matches a heading like `## [1.2.3]`, `## [v1.2.3]`, `## 1.2.3` or `## v1.2.3`.
  `release-dev.yml` matches any `## ` heading containing the base version. **Keep the heading format
  as it is, or release notes come out empty.**
- Do not rename or delete `README.md`, `CHANGELOG.md` or `.github/workflows/*` (`AGENTS.md:78`).
- Do not change version numbers or packaging (`AGENTS.md:80`).

### 5.5 The four workflows a pull request meets [V]

| Workflow | Trigger | What it does |
|---|---|---|
| `build.yml` | push to `main` or `develop` touching `src/**`, `build.js`, `package.json`, `package-lock.json` or itself, plus manual | Node 24, `npm ci`, `npm run build`, commits `dist/unifi-device-card.js` back with `[skip ci]` |
| `hacs-validate.yml` | **every push and every pull request** | `hacs/action@main` with `category: plugin` |
| `release.yml` | manual only | Validates `v1.2.3`, refuses an existing tag, extracts changelog, builds, commits dist, tags, creates a release with `dist/unifi-device-card.js` as the only asset |
| `release-dev.yml` | manual only | Same on `develop`, tag `v<version>-dev`, marked pre-release |

**`hacs-validate.yml` is the only job a pull request must pass.** There is no lint job, no test job and
no type check. `package.json` declares exactly one script, `"build": "node build.js"`, and one
dev dependency, `esbuild ^0.25.2`.

`build.js` bundles `src/unifi-device-card.js` with esbuild: `format: "esm"`, `target: "es2020"`,
`bundle: true`, and it replaces `__VERSION__` with the computed version string. Because there is no
test suite, `AGENTS.md:92` applies: perform a careful static review of the changed files, and say
explicitly when verification could not be completed.

**Local check before any pull request:**

```bash
npm ci && npm run build && git diff --stat dist/
```

---

## 6. Recommendation

### 6.1 The two problems are independent

They are worth separating, because they have different risk profiles.

- **Problem 1 — the card is one view column wide in a sections view.** It is a Lovelace API gap. The
  card implements no `getGridOptions()`.
- **Problem 2 — declared model layouts never render.** It is a bug inside this card, at
  `fork:src/helpers.js:2051`. It is independent of the view type, and it affects masonry users too.

Fix Problem 2 first. It is smaller, it needs no new API, and it helps every user. Problem 1 changes
how existing dashboards look, so it deserves its own release note.

### 6.2 For Problem 2 — Option B, plus a note

Take **Option B** from section 4.8: keep the hardcoded `8` only when the layout declares fewer than
two rows.

| Property | Result |
|---|---|
| Diff size | One condition on one line in `fork:src/helpers.js` |
| Backwards compatible | Yes for every `single-row` fallback, which is the majority of registry entries (32 of them) |
| Fixes `US24P250` | Yes |
| Fixes `quad-row`, `eight-grid`, `dual-row`, `six-grid` models | Yes — 35 entries in total |
| Narrow cards | Unchanged. `_buildEffectiveRows` repacks, verified in section 4.6 |
| First-frame overflow | Unchanged for `single-row`. New, and brief, for multi-row models wider than the card |

Add **Option C** only when the first frame turns out to flicker in practice. It is a real risk but a
one-frame one, and `_finalizeRender` already corrects it inside a `requestAnimationFrame`.

Two things to check while making that change, both from section 4:

- `_shouldUseOddEvenRows` reaches its `frontStyle` guard never (section 4.4). Once multi-row layouts
  survive, odd/even reordering applies to them. For `US24P250` that is the wanted result. For
  `six-grid` it is worth a second look.
- `--udc-cols` is written and never read (section 4.3). Wiring it up is a separate, optional change.
  Leave it out of a minimal fix.

### 6.3 For Problem 1 — add `getGridOptions()`, derived from the model

Follow the `hui-iframe-card` pattern: pure, synchronous, configuration-driven, no measurement. Section
3.3 explains why a measured value must not feed back into this method.

The shape that fits this card:

| Device type | `columns` | `rows` | `min_columns` | Reason |
|---|---|---|---|---|
| Switch, gateway | `"full"` | `"auto"` | a number derived from the widest declared row | Span-independent, and section 4.7 shows that a fixed number cannot guarantee any port count |
| Access point | `12` | `"auto"` | modest | The access-point rendering is a centered graphic, not a port grid |

**What each part costs:**

- **`columns: "full"`** — the only value that reliably gives the card the whole section. Cost: in a
  `column_span: 4` section, the card jumps from one quarter of the width to the full width. That is a
  visible change for existing dashboards, and it belongs in the changelog. Users can override it with
  `grid_options: {columns: N}`, because the configuration wins the merge (section 1.5).
- **`rows: "auto"`** — already the default (section 1.3). Stating it explicitly costs nothing and
  documents the intent. Do **not** return a numeric `rows`: the card height depends on the selected
  port and on how many rows the panel needs, so a fixed height would clip.
- **`min_columns`** — real value in the editor, real risk in production. It stops a user from dragging
  the card down to an unusable width. It also **raises** an existing saved `grid_options.columns`
  that is below the new floor (section 1.5), which silently widens cards that users deliberately made
  narrow. Choose a conservative floor, or leave `min_columns` out of the first release.
- **`max_columns`** — do not set it. Section 1.5 shows that it caps what the user can configure, and
  this card has no upper width at which it stops improving.

**Cost of doing nothing:** the card keeps one view column in sections view, and the editor keeps
showing *"This card does not fully support resizing yet…"*.

### 6.4 The combination that makes the card self-dimensioning

| Layer | Change | Effect |
|---|---|---|
| 1 | Option B in `fork:src/helpers.js` | The model's declared rows survive |
| 2 | `getGridOptions()` returning `columns: "full"`, `rows: "auto"` | Sections view gives the card the whole section instead of one column |
| 3 | Existing `_buildEffectiveRows` repack | Whatever width arrives, the rows fit — no change needed |
| 4 | Optional: Option C | Removes the one-frame overflow before the first measurement |

Layers 1 and 3 also help masonry users. Layer 2 helps only sections view, because masonry offers no
lever at all (section 2.1).

**What this combination still cannot do.** In masonry, and in a `column_span: 1` section, the card
gets at most 500 px, and a 12-port row needs 502 px (section 4.7). The declared two-rows-of-twelve
layout will still repack there. The only remaining levers are user-side: a wider section, a smaller
`port_size`, or the `--ha-view-sections-column-max-width` theme variable.

### 6.5 Documentation and release notes for whichever option is chosen

- `CHANGELOG.md` needs an entry, because both problems produce user-visible changes
  (`AGENTS.md:66`). Keep the existing heading format so the release workflows keep parsing it.
- `README.md` needs an edit only for layer 2, in the configuration section around `ports_per_row`,
  and only when the described behaviour changes (`AGENTS.md:65`).
- Screenshots can need a manual refresh. Say so, and do not touch `/screenshots`
  (`AGENTS.md:70-75`).

---

## 7. What could not be verified

1. **No rendered-card verification.** Every statement about this fork comes from reading source. No
   card was loaded in a browser, and no layout was measured live. The width table in section 4.7 is
   arithmetic over verified CSS, not a measurement.
2. **The 446 px measurement was not reproduced.** The *ratio* is confirmed against the CSS — a `{}`
   card spans 12 of 48 tracks, one quarter. The absolute pixel value depends on the viewport, which is
   unknown here.
3. **`_applyOddEvenRows` output for `US24P250`** is read from code, not observed. Section 4.4 marks it.
4. **HACS validation was not run.** `hacs/action@main` needs the GitHub Actions environment.
5. **No build was run.** No `npm ci` and no `npm run build` in this session, per the instruction to
   change nothing.
6. **The `card-updated` escape hatch was not tested.** The listener registration, the light-DOM
   parenting and the `fireEvent` defaults are all verified in frontend source. The end-to-end path was
   not exercised in a browser.
7. **The `maxColumnsContext` route was not explored in depth.** The sections view provides a
   `@lit/context` context named `"lovelace-max-columns"`
   (`frontend:src/panels/lovelace/common/context.ts:3`,
   `frontend:src/panels/lovelace/views/hui-sections-view.ts:69-71`), and its only consumer in the
   frontend is `conditional-listener-mixin.ts:55`. A custom card can request it through the standard
   `context-request` DOM event, but it carries the **view** column count, not the section's
   `column_span`, and it is internal and undocumented. It was not tested.
8. **Which exact frontend release first bundled PR #22676 was checked by file content, not by release
   notes.** The tag `20241106.0` contains `getGridOptions` and `20241002.4` does not, and core
   `2024.11.0` pins `20241106.0`. Intermediate tags between those two dates were not each checked.
9. **Behaviour on Home Assistant versions other than 2026.8.1** was not tested. The compatibility
   claim in section 1.10 rests on the property test at `hui-card.ts:82`, which is read from source.
