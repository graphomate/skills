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

Chartrix uses CSS-in-JS (Emotion) internally, but **the `css` tag function is not
reliably available globally in the XFL sandbox** — empirically confirmed to throw
`css is not defined` in at least one chart context. Use the plain-object form instead,
which works without relying on the tag function being in scope:

```javascript
// Reliable - plain object form:
bar.styles = { name: "xfl", styles: "fill: red;" };

// If `css` happens to be available in your context, this also works - but don't
// depend on it:
// bar.styles = css`fill: red;`;
```

`addStyles()` for merging with existing styles:
```javascript
bar.addStyles({ name: "xfl", styles: "fill: red; opacity: 0.5;" });
```

Common CSS properties for ViewItems:
- `fill` — fill color of bars/shapes
- `stroke` — border color
- `stroke-width` — border thickness
- `opacity` — transparency (0–1)
- `font-size` — for labels
- `font-weight` — for labels
- `color` — text color for Label (alternatively use `label.color`)

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
