# XFL Code Patterns & Examples

## Pattern 1: Auto-highlight between min and max of series 1
*(The reference script — annotated)*

This is the canonical example. It dynamically spans the highlight element between
the highest and lowest value of the first data series.

```javascript
// 1. Get all cells of series 0 (index 0 = first series)
const series1Data = this.getData().data[0];

// 2. Find the max data point using reduce
const maxDataPoint = series1Data.reduce((maxDataPoint, currentCell) => {
  const currentDatapoint = currentCell.cellRepresentingDataPoint;
  if (maxDataPoint === undefined) return currentDatapoint;
  if (maxDataPoint.value > currentCell.cellRepresentingDataPoint.value) return maxDataPoint;
  return currentDatapoint;
}, series1Data[0].cellRepresentingDataPoint);

// 3. Find the min data point
const minDataPoint = series1Data.reduce((minDataPoint, currentCell) => {
  const currentDatapoint = currentCell.cellRepresentingDataPoint;
  if (minDataPoint === undefined) return currentDatapoint;
  if (minDataPoint.value < currentCell.cellRepresentingDataPoint.value) return minDataPoint;
  return currentDatapoint;
}, series1Data[0].cellRepresentingDataPoint);

// 4. Format the label text (absolute + percent deviation)
const divValue = new Intl.NumberFormat("en-GB", {
  notation: "compact", compactDisplay: "short",
  maximumFractionDigits: 1, minimumFractionDigits: 1,
}).format(maxDataPoint.value - minDataPoint.value);

const divValuePerc = new Intl.NumberFormat("de-DE", {
  style: "percent", signDisplay: "exceptZero",
  maximumFractionDigits: 1, minimumFractionDigits: 1,
}).format((maxDataPoint.value - minDataPoint.value) / minDataPoint.value);

const labelText = `${divValue}\n${divValuePerc}`;

// 5. Mutate the highlight view items
this.viewModel.forEach((viewItem) => {
  switch (viewItem.semanticType) {
    case "HIGHLIGHT_DELTA_LINE":
      // The delta line spans from minDataPoint to maxDataPoint
      viewItem.x.scaledValue.dataPoint = minDataPoint;
      viewItem.x.scaledValue.zeroReference = maxDataPoint;
      viewItem.y.scaledValue.dataPoint = minDataPoint;
      viewItem.y.scaledValue.zeroReference = maxDataPoint;
      break;
    case "HIGHLIGHT_LABEL":
      // Same positioning as the line, plus the text
      viewItem.x.scaledValue.dataPoint = minDataPoint;
      viewItem.x.scaledValue.zeroReference = maxDataPoint;
      viewItem.y.scaledValue.dataPoint = minDataPoint;
      viewItem.y.scaledValue.zeroReference = maxDataPoint;
      viewItem.text = labelText;
      break;
    case "HIGHLIGHT_CONNECTOR":
      // Two connectors: one at source (min), one at target (max)
      if (viewItem.key.startsWith("highlight:line:connector:target")) {
        viewItem.x.scaledValue.dataPoint = maxDataPoint;
        viewItem.y.scaledValue.dataPoint = maxDataPoint;
      } else {
        viewItem.x.scaledValue.dataPoint = minDataPoint;
        viewItem.y.scaledValue.dataPoint = minDataPoint;
      }
      break;
  }
});
```

**Prerequisite**: A `highlights` entry must exist in `visual.json` (even with placeholder
addresses). The XFL script overrides the positions at runtime. If no highlight config
exists, there are no HIGHLIGHT_* viewItems to manipulate.

---

## Pattern 2: Conditional bar coloring

Color bars above a threshold green, below red. Uses `getDataMetrics()` for efficiency.

```javascript
this.bars.forEach(bar => {
  if (bar.semanticType !== "DATA_BAR") return;
  const dp = bar.dataPoint?.cellRepresentingDataPoint;
  if (!dp) return;

  if (dp.value > 0) {
    bar.styles = css`fill: ${IbcsColors.GREEN};`;
  } else {
    bar.styles = css`fill: ${IbcsColors.RED};`;
  }
});
```

---

## Pattern 3: Highlight local extrema (max per series)

Color the highest bar in each series using pre-computed extremas:

```javascript
const extremas = this.getDataMetrics().extremasPerSeries;

this.bars.forEach(bar => {
  if (bar.semanticType !== "DATA_BAR") return;
  const r = bar.rowIndex;
  const c = bar.columnIndex;
  if (r === undefined || c === undefined) return;

  if (extremas[r]?.[c]?.isLocalMax) {
    bar.styles = css`fill: ${IbcsColors.BLUE};`;
  }
});
```

---

## Pattern 4: Add custom annotation labels

Add a "▲ Max" label above the highest bar:

```javascript
const series0 = this.getData().data[0];

// Find max data point
let maxDp = series0[0].cellRepresentingDataPoint;
let maxBar = null;

this.bars.forEach(bar => {
  if (bar.semanticType !== "DATA_BAR" || bar.rowIndex !== 0) return;
  const dp = bar.dataPoint?.cellRepresentingDataPoint;
  if (dp && dp.value >= maxDp.value) {
    maxDp = dp;
    maxBar = bar;
  }
});

if (maxBar) {
  // Create label relative to the max bar
  const label = this.createLabel("▲ Max", 0, -15, maxBar);
  label.color = IbcsColors.BLUE;
}
```

---

## Pattern 5: Filter by address (scenario-specific coloring)

Color only the "Budget" bars differently:

```javascript
const isBudget = this.createFilterByAddress({ "Datenquelle.Version": ["Budget"] });

this.bars.filter(isBudget).forEach(bar => {
  bar.styles = css`fill: transparent; stroke: ${IbcsColors.DARK_GREY}; stroke-width: 2px;`;
});
```

---

## Pattern 6: Read XflVariable for configurable threshold

```javascript
// In xflVariables: { key: "highlightThreshold", value: "50000" }
const threshold = Number(this.getXflVariable("highlightThreshold")) || 0;

this.bars.forEach(bar => {
  if (bar.semanticType !== "DATA_BAR") return;
  const value = bar.dataPoint?.cellRepresentingDataPoint?.value;
  if (value !== undefined && value > threshold) {
    bar.addStyles(css`fill: ${IbcsColors.BLUE};`);
  }
});
```

---

## Pattern 7: Modify value label text

Append a unit or change formatting of existing value labels:

```javascript
this.labels.forEach(label => {
  if (label.semanticType !== "VALUE_LABEL_BAR") return;
  if (label.text) {
    label.text = label.text + " k€";
  }
});
```

---

## Pattern 8: Auto-highlight between max values of two different series

Useful for showing spread between AC and BU at their respective peaks:

```javascript
const data = this.getData().data;
const series0 = data[0];  // e.g. Actual
const series1 = data[1];  // e.g. Budget

const getMax = (series) => series.reduce((max, cell) => {
  const dp = cell.cellRepresentingDataPoint;
  return (max === undefined || dp.value > max.value) ? dp : max;
}, undefined);

const maxAC = getMax(series0);
const maxBU = getMax(series1);

this.viewModel.forEach(viewItem => {
  switch (viewItem.semanticType) {
    case "HIGHLIGHT_DELTA_LINE":
    case "HIGHLIGHT_LABEL":
      viewItem.x.scaledValue.dataPoint = maxBU;
      viewItem.x.scaledValue.zeroReference = maxAC;
      viewItem.y.scaledValue.dataPoint = maxBU;
      viewItem.y.scaledValue.zeroReference = maxAC;
      if (viewItem.semanticType === "HIGHLIGHT_LABEL") {
        const diff = maxAC.value - maxBU.value;
        viewItem.text = new Intl.NumberFormat("de-DE", {
          notation: "compact", maximumFractionDigits: 1
        }).format(diff);
      }
      break;
    case "HIGHLIGHT_CONNECTOR":
      const isTarget = viewItem.key.startsWith("highlight:line:connector:target");
      viewItem.x.scaledValue.dataPoint = isTarget ? maxAC : maxBU;
      viewItem.y.scaledValue.dataPoint = isTarget ? maxAC : maxBU;
      break;
  }
});
```

---

## Pattern 9 (open limitation): centering a highlight label — nothing tried works reliably

`HIGHLIGHT_LABEL` is documented as "positioned like HIGHLIGHT_DELTA_LINE" (`x/y.scaledValue.dataPoint`
+ `zeroReference`). In practice, none of the following reliably centered the label text on the
delta line:

1. **Synthetic midpoint `DataPoint`** (`dataPoint = zeroReference = average of min/max`) — anchor
   point looked plausible but the rendered text still sat offset toward one side.
2. **Custom `createLabel()` anchored to the delta line**, offset by half the line's own
   `getAbsoluteWidth()`/`getAbsoluteHeight()` — anchors on the line's midpoint correctly, but the
   label's *own* text box still grows off to one side from that anchor (not centered on itself).
3. **Reading the new label's own `getAbsoluteWidth()`/`getAbsoluteHeight()` right after
   `createLabel()`, to compensate for its own box** — **crashes the whole visual**:
   `Uncaught TypeError: Cannot read properties of undefined (reading 'getDataLength')` at
   `Label.getAbsoluteWidth`. A freshly created element isn't mounted/measured yet within the same
   script execution — don't call geometry getters on same-run-created elements.
4. **CSS centering via `styles`** (`text-anchor: middle; dominant-baseline: middle`) — did not
   visibly change the label position. Also confirmed here: the `css` tag function is **not**
   globally available in this sandbox (`css is not defined`), contradicting the `api.md` styling
   section — use the plain-object form instead: `label.styles = { name: "xfl", styles: "..." }`.
   Even with that fix, the text-anchor styling had no visible centering effect on `HIGHLIGHT_LABEL`
   text specifically — unclear whether `text-anchor`/`dominant-baseline` are respected on Label
   elements at all in this renderer.

**Net result**: reliably centering a highlight label's text is currently unsolved. If you need
this, the pragmatic workaround is to accept the offset, or fall back to a fixed/tuned pixel nudge
for the specific chart layout at hand rather than a general solution. Revisit if graphomate ships
an explicit centering property, or worth raising as a feature request.

---

## Common mistakes to avoid

**1. Iterating `getData().data` cells**
`CartesianDataRow` is array-like but not a plain array. Use a numeric index loop or
`Array.from()` — don't assume `.length` works reliably with `for...of`.

```javascript
// Safe iteration over cells in series 0:
const series = this.getData().data[0];
let i = 0;
while (series[i] !== undefined) {
  const dp = series[i].cellRepresentingDataPoint;
  // ...
  i++;
}
```

**2. Missing null checks on `dataPoint`**
Not every `ViewItem` has a `dataPoint`. Always check before accessing `.value`:
```javascript
const value = bar.dataPoint?.cellRepresentingDataPoint?.value;
if (value === undefined) return;
```

**3. Forgetting `rowIndex`/`columnIndex` checks**
Bars from different series share the same `semanticType`. Filter by `rowIndex` to
target a specific series:
```javascript
this.bars.forEach(bar => {
  if (bar.rowIndex !== 0) return;  // only first series
  // ...
});
```

**4. Highlight manipulation without a highlights config entry**
HIGHLIGHT_* viewItems only exist if a `highlights` config entry is present in
`visual.json`. A placeholder entry with any two addresses is sufficient — the XFL
script overrides positions at runtime.

**5. `styles` und `rowIndex` auf PIN_HEAD**
`PIN_HEAD` (Kreis-Marker im Pin Chart) ignoriert `styles` und `outlined` vollständig.
Nur `color` hat Wirkung. Außerdem ist `rowIndex` nicht gesetzt — Serienerkennung
immer über `address`:
```javascript
this.viewModel.forEach(item => {
  if (item.semanticType !== "PIN_HEAD") return;
  const version = item.address ? Object.values(item.address).join("|") : "";
  if (version.includes("FC")) item.color = "white"; // hollow-Simulation
});
```

**6. `getXflVariable` returns string**
All variable values are strings. Always convert explicitly:
```javascript
const n = Number(this.getXflVariable("myNumber"));
const b = this.getXflVariable("myFlag") === "true";
```

**7. Don't assume `rowIndex`/`columnIndex` = series/category without verifying**
`bar.rowIndex`/`bar.columnIndex` semantics are not fixed across chart types/orientations
— in one (vertical waterfall) context `rowIndex` was confirmed to be a *category* index,
not a series index. Filtering `this.bars` on an assumed index (e.g. `rowIndex === 0`
meaning "first series") can silently mix bars from different series/categories and
produce wrong values with no error. If you need a specific series' bars, prefer
matching against known values/addresses from `this.getData()` (which is unambiguous),
or empirically dump `key`/`rowIndex`/`columnIndex`/`address` for all bars first
(`this.bars.forEach(b => console.log(b.key, b.rowIndex, b.columnIndex, b.address))`)
to confirm the mapping for that specific chart before relying on it.
