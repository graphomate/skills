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
    bar.styles = `fill: #8cb400;`; // IbcsColors.GREEN
  } else {
    bar.styles = `fill: #ff0000;`; // IbcsColors.RED
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
    bar.styles = `fill: #0070c0;`; // IbcsColors.BLUE
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
  label.color = "#0070c0"; // IbcsColors.BLUE
}
```

---

## Pattern 5: Filter by address (scenario-specific coloring)

Color only the "Budget" bars differently:

```javascript
const isBudget = this.createFilterByAddress({ "Datenquelle.Version": ["Budget"] });

this.bars.filter(isBudget).forEach(bar => {
  bar.styles = `fill: transparent; stroke: #404040; stroke-width: 2px;`; // IbcsColors.DARK_GREY
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
    bar.addStyles(`fill: #0070c0;`); // IbcsColors.BLUE
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

## Pattern 9: Style DATA_MARK dots of a specific series (line chart)

`rowIndex` on `DATA_MARK` is the **category index** (e.g. week), not the series index —
it restarts at 0 for every series. Series identification must go through `viewItem.key`
(format `data:lineDot:catIndex{N}:serIndex:{S}`) or through `viewItem.address`.

**Which one to use depends on how the second series was created:**
- **Series created via a dimension in the series feed** (e.g. a `Version` field with
  members `Actual`/`Budget`): `address` contains that dimension and can be filtered on
  directly, e.g. `address.Version === "Budget"`.
- **Series created via two measures in the measure feed** (e.g. `Revenue` and `Cost`
  both in Values): there is no extra dimension in `address` to filter on — the measure
  name itself is the series discriminator, under the
  `"graphomate.internal.measures"` address key.

The **`serIndex` from `key` works in both cases** and is the more robust, general-purpose
way to target "the second series", regardless of how it was created:

```javascript
// Robust: works whether series came from a dimension or from two measures
this.dataMarks.forEach(mark => {
  if (mark.semanticType !== "DATA_MARK") return;
  if (!mark.key || !mark.key.includes("serIndex:1")) return; // 0 = first series, 1 = second

  mark.color = "#8f8f8f";
  mark.radius = 2; // unconfirmed property name — see Common Mistake #7
});
```

```javascript
// Alternative when series come from a dimension (e.g. Version) — more readable,
// but only works for the dimension-based case, not the two-measures case:
this.dataMarks.forEach(mark => {
  if (mark.semanticType !== "DATA_MARK") return;
  if (mark.address?.Version !== "Budget") return;

  mark.color = "#8f8f8f";
});
```

```javascript
// Alternative when series come from two measures — filter on the measure name instead:
this.dataMarks.forEach(mark => {
  if (mark.semanticType !== "DATA_MARK") return;
  if (mark.address?.["graphomate.internal.measures"] !== "Cost") return;

  mark.color = "#8f8f8f";
});
```

**Sizing note:** `DATA_MARK` size cannot be set per-item via XFL (no `radius`/`r`
property, `styles` is ignored — confirmed by property inspection, see Pattern 10). To
change dot size, use the `customCss` config property targeting the `.gm-circle` CSS
class instead — see Common Mistake #10. This only applies uniformly to all series, not
per-series like `color` above.

---

## Pattern 10: Debugging — inspect a viewItem when styling has no visible effect

If a `styles` (or other property) assignment produces **no error and no visible change**,
don't guess further — log the actual object first. `console.log` is available in XFL and
shows up in the browser console:

```javascript
// 1. Inspect one representative item's data (key, address, rowIndex, etc.)
this.dataMarks.slice(0, 3).forEach((m, i) => {
  console.log(`Mark ${i}:`, "semanticType=", m.semanticType, "rowIndex=", m.rowIndex,
    "key=", m.key, "address=", JSON.stringify(m.address));
});

// 2. Enumerate ALL own + inherited properties of one item — reveals the real
//    writable properties when the docs are unclear or wrong:
const mark = this.dataMarks.find(m => m.semanticType === "DATA_MARK");
if (mark) {
  const props = new Set();
  let obj = mark;
  while (obj) {
    Object.getOwnPropertyNames(obj).forEach(p => props.add(p));
    obj = Object.getPrototypeOf(obj);
  }
  console.log("DATA_MARK properties:", Array.from(props));
}
```

Use this whenever a mutation is silently ignored — it has already uncovered that
`rowIndex` on `DATA_MARK` is a category index, not a series index, and it's the fastest
way to confirm whether a property like `radius` actually exists before trying more guesses.

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

**7. `rowIndex` on `DATA_MARK` is a category index, not a series index**
Unlike `DATA_BAR`, where `rowIndex` identifies the series, `DATA_MARK.rowIndex` is the
*category* index (e.g. week/period) and restarts at 0 for every series. Use `key`
(`serIndex:{N}`) or `address` to identify the series instead — see Pattern 9.

**8. `styles` may be silently ignored on Circle-based markers**
Confirmed for `PIN_HEAD` (only `color` works, not `styles`/`outlined`); suspected but
**not yet confirmed** for `DATA_MARK`. If a `styles` assignment produces no error but
also no visible change, don't keep guessing CSS properties — use the debugging pattern
(Pattern 10) to enumerate the object's real properties first.

**9. `css` tag function and `IbcsColors` are not global in the sandbox**
Despite being usable in some documentation examples, both throw `ReferenceError` in the
live XFL sandbox. Use plain template-literal strings for `styles` and raw hex values
instead of `IbcsColors.*`.

**10. `DATA_MARK` size (line-chart dots) is not scriptable via XFL**
Confirmed by inspecting a `DATA_MARK` instance's own + inherited properties: there is no
`radius`/`r` property, and `styles`/`addStyles()` have no effect (no `styles` property
exists on the object at all). Only `color` is a real, working property. For size, use the
`customCss` config property instead, targeting the `.gm-circle` CSS class:

```css
.gm-circle {
  width: 4px;
  height: 4px;
}
```

This is real browser CSS (not the Emotion/CSS-in-JS used by `styles`), configured as a
separate `customCss` array entry in `visual.json` (`{ name, css, enabled }`), not inside
`xflRules`. It applies uniformly to all series — there is currently no way to size dots
per-series only. Worth flagging as a graphomate feature request if per-series sizing is
needed.
