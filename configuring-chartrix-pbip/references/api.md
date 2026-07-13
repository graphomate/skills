# XFL API Reference

## CartesianSplitView (`this.getData()`)

```typescript
interface CartesianSplitView {
  data: CartesianDataRow[];  // one entry per series (row dimension)
}

// Each CartesianDataRow is both an object and an array of cells:
interface CartesianDataRow {
  cellRepresentingDataPoint: DataPoint;  // row-level aggregate point
  [categoryIndex: number]: CartesianDataCell;
}

interface CartesianDataCell {
  cellRepresentingDataPoint: DataPoint;
}

interface DataPoint {
  value: number;
  formattedValue: string;
  // plus address, metadata fields
}
```

**Iteration pattern:**
```javascript
const allSeries = this.getData().data;           // CartesianDataRow[]
const series0 = allSeries[0];                    // first series

// Iterate cells in series0:
const cells = Array.from({ length: Object.keys(series0)
  .filter(k => !isNaN(Number(k))).length }, (_, i) => series0[i]);
```

---

## DataMetrics (`this.getDataMetrics()`)

Pre-computed statistics — prefer this over manual reduce when possible.

```typescript
interface DataMetrics {
  maxValue: number;                        // global max across all series
  minValue: number;                        // global min across all series
  maxValuePerColumn: (number|undefined)[]; // max per category index
  minValuePerColumn: (number|undefined)[]; // min per category index
  maxValuePerRow: (number|undefined)[];    // max per series index
  minValuePerRow: (number|undefined)[];    // min per series index
  extremasPerSeries: {                     // local extrema per cell
    isLocalMin: boolean;
    isLocalMax: boolean;
  }[][];                                   // [seriesIndex][categoryIndex]
  maxValuePerChartLayout: Map<string, number>; // max per sub-chart key
  minValuePerChartLayout: Map<string, number>; // min per sub-chart key
}
```

**Example — find global max without reduce:**
```javascript
const metrics = this.getDataMetrics();
const globalMax = metrics.maxValue;
const series0Max = metrics.maxValuePerRow[0];
```

**Example — mark local extrema:**
```javascript
const extremas = this.getDataMetrics().extremasPerSeries;
this.bars.forEach(bar => {
  const r = bar.rowIndex;
  const c = bar.columnIndex;
  if (r !== undefined && c !== undefined && extremas[r]?.[c]?.isLocalMax) {
    bar.styles = css`fill: #0070c0;`;  // blue highlight
  }
});
```

---

## ChartrixProperties (`this.getProperties()`)

Exposes all chart configuration properties at runtime:

```typescript
type ChartrixProperties = {
  xflRules: XflRule[];          // the scripts themselves
  xflVariables: XflVariable[];  // user-defined key-value pairs
  orientation: "HORIZONTAL" | "VERTICAL";
  chartsLayout: SubChartConfig[];
  charts: EveryChartTypeConfig[];
  highlights: HighlightConfig[];
  scenarios: ScenarioSelection[];
  deviationCalculations: DeviationConfig[];
  valueFormats: ActivatableNumberFormatSelection[];
  fontSize: string;
  fontColor: string;
  // ... all other visual.json properties
};
```

**Example — read orientation:**
```javascript
const isHorizontal = this.getProperties().orientation === "HORIZONTAL";
```

---

## XflVariable (`this.getXflVariable(name)`)

Variables defined in `xflVariables` are accessible by key.
Useful for making scripts configurable without code changes.

```javascript
// In xflVariables: { key: "threshold", value: "5000" }
const threshold = Number(this.getXflVariable("threshold"));
```

---

## Filter helpers

These return a `(item: ViewItem) => boolean` predicate for use with `.filter()`.

### `createFilterByCategoryAddress(addressSubset)`
Matches items in a category address — works for axis labels and non-series items.

### `createFilterByAddress(addressSubset)`
Matches items assigned to a specific data series.

### `createFilterByDataValue(predicate)`
Matches items by their associated data value.

```javascript
// Filter bars in the "Actual" scenario
const isActual = this.createFilterByAddress({ "Datenquelle.Version": ["Actual"] });
this.bars.filter(isActual).forEach(bar => { /* ... */ });

// Filter bars above threshold
const isAbove = this.createFilterByDataValue(v => v > 10000);
this.bars.filter(isAbove).forEach(bar => { /* ... */ });
```

---

## Element creation

All `create*` methods accept an optional `positionReference` ViewItem. When provided,
the new element's coordinate system is relative to the reference item.

```javascript
// Absolute position (no reference)
const label = this.createLabel("Max", 100, 50);

// Relative to a bar
const bar = this.bars[0];
const label = this.createLabel("!", 0, -10, bar);
```

### `createLabel(text, x, y, ref?)`
Returns a `Label` (a `ViewItem` with `text`, `color`, `outlined` properties).

### `createRect(x, y, width, height, ref?)`
Returns a `Rect`.

### `createCircle(x, y, radius, ref?)`
Returns a `Circle`.

### `createImage(url, x, y, width, height, ref?)`
Returns an `Image`. URL must be accessible at render time.

### `createWedge(x, y, width, height, ref?)`
Returns a `Wedge` with a `headSize` property.

---

## Styling with `addStyles()` / `styles`

Chartrix uses CSS-in-JS (Emotion) internally, but **neither the `css` tag function nor
`IbcsColors` are actually available as globals in the live XFL sandbox**, despite being
shown that way in some examples. Assign a plain string (template literal without a `css`
tag) instead, and use raw hex values instead of `IbcsColors.*`:

```javascript
// CONFIRMED WORKING — plain string, no css tag:
bar.styles = `fill: #ff0000;`;

// Or use addStyles() to merge with existing styles:
bar.addStyles(`fill: #ff0000; opacity: 0.5;`);
```

`ReferenceError: css is not defined` / `ReferenceError: IbcsColors is not defined` in the
console confirm this — if you see either, drop the `css` tag and swap `IbcsColors.X` for
its raw hex value (see the table at the end of this file).

Common CSS properties for ViewItems:
- `fill` — fill color of bars/shapes
- `stroke` — border color
- `stroke-width` — border thickness
- `opacity` — transparency (0–1)
- `font-size` — for labels
- `font-weight` — for labels
- `color` — text color for Label (alternatively use `label.color`)

**Known limitation — `styles` silently ignored on some Circle-based markers:**
On `PIN_HEAD`, `styles` and `outlined` have no effect at all — only the `color` property
works (see Common Mistake #5 in `patterns.md`). `DATA_MARK` (line-chart dots) may share
this behavior. If assigning `mark.styles = \`fill: ...; r: 2px;\`` produces no error but
also no visible change, try direct properties instead:

```javascript
mark.color = "#8f8f8f";
mark.radius = 2;   // try `radius`; if that has no effect either, inspect the object
                    // directly (see debugging pattern in patterns.md) to find the
                    // correct property name — it has not been confirmed for DATA_MARK yet.
```

This is unconfirmed for `DATA_MARK` specifically — treat it as the first thing to try,
not a guaranteed fix. If you find the working property, update this note.

**Confirmed: `DATA_MARK` size is not scriptable via XFL at all.** `color` works directly
as a property; `styles`/`addStyles()` are silently ignored (no `styles` property exists
on the object — `Object.getOwnPropertyNames` inspection confirmed this); there is no
`radius`/`r` property either. For size, use the `customCss` config property instead
(real browser CSS, not Emotion/CSS-in-JS — a separate mechanism from `styles`/XFL). The
rendered dots have the CSS class `.gm-circle`, sized via `width`/`height` (not `r`):

```css
/* customCss entry — affects ALL dots (all series), not scriptable per-series */
.gm-circle {
  width: 4px;
  height: 4px;
}
```

`customCss` is a top-level `visual.json` property (array of `{ name, css, enabled }`),
separate from `xflRules` — it's the general escape hatch for anything the XFL element API
doesn't expose a property for, per its own description: "a perfect counterpart to the XFL
when highly complex selectors are needed."

---

## IBCS color constants

Available as `IbcsColors.*`:
```javascript
IbcsColors.BLACK          // "#000000" — text, axes, lines
IbcsColors.DARK_GREY      // "#404040" — actual data (AC)
IbcsColors.LIGHT_GREY     // "#a6a6a6" — prior year (PY)
IbcsColors.GREEN          // "#8cb400" — good variance
IbcsColors.RED            // "#ff0000" — bad variance
IbcsColors.GREEN_ACCESSIBLE // accessible green alternative
IbcsColors.MEDIUM_GREY    // "#8f8f8f" — neutral variance
IbcsColors.BLUE           // "#0070c0" — highlighting
```
