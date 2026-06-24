---
name: chartrix-xfl
description: >
  Use this skill whenever a user wants to write, debug, or understand XFL scripts for the
  graphomate chartrix visual. XFL is a JavaScript-based scripting dialect that allows
  runtime manipulation of the chartrix viewModel — use this skill for any request involving
  dynamic highlighting, conditional bar coloring, custom labels, data-driven annotations,
  or any scripted customization of a chartrix chart. Triggers include: "XFL", "xfl script",
  "xflRules", "highlight the max/min", "color bars dynamically", "add a label to", "manipulate
  the viewmodel", "auto highlight", "scripting in chartrix", or whenever the user pastes or asks
  about a `this.viewModel.forEach` / `this.getData()` snippet. Always use this skill before
  writing or modifying any chartrix scripting code, even if the request seems straightforward.
---

# graphomate chartrix – XFL Scripting Skill

XFL is a JavaScript dialect executed inside graphomate chartrix at render time. Each script
runs in an environment where `this` exposes the full API. The script manipulates `viewModel`
items (visual elements like bars, labels, lines) or creates new ones.

> **Full API reference**: see `references/api.md`
> **SemanticType catalogue**: see `references/semantic-types.md`
> **Code patterns & examples**: see `references/patterns.md`
>
> **Ergänzender Skill**
> XFL greift auf Elemente zu, die durch die chartrix-Config erzeugt werden.
> Für statische Grundkonfiguration (Szenarien, Chart-Typen, Highlights, Separatoren)
> → **graphomate-chartrix Skill** nutzen. Übersicht Config → Element → SemanticType
> → `graphomate-chartrix/references/elements.md`

---

## How XFL scripts are stored in visual.json

XFL scripts live in the `xflRules` property. In `visual.json`, this is stored as a stringified
JSON array — same escaping rules as all other chartrix properties:

```json
"InputOutput": [{ "properties": {
  "xflRules": { "expr": { "Literal": { "Value": "'[{\"enabled\":true,\"name\":\"My Rule\",\"script\":\"...\"}]'" } } },
  "xflVariables": { "expr": { "Literal": { "Value": "'[{\"key\":\"threshold\",\"value\":\"1000\"}]'" } } }
}}]
```

**XflRule shape:**
```json
{ "enabled": true, "name": "descriptive name", "script": "// JS code here" }
```

**XflVariable shape** — read in scripts via `this.getXflVariable("key")`:
```json
{ "key": "myThreshold", "value": "5000" }
```

Multiple rules are supported and run in array order. Disable a rule by setting `"enabled": false`.

---

## The `this` API — quick reference

| Method / Property | Returns | Purpose |
|---|---|---|
| `this.viewModel` | `ViewItem[]` | All rendered elements |
| `this.bars` | `Rect[]` | Pre-filtered: bars only |
| `this.labels` | `Label[]` | Pre-filtered: labels only |
| `this.dataMarks` | `ViewItem[]` | Pre-filtered: data marks |
| `this.lines` | `Line[]` | Pre-filtered: lines only |
| `this.getData()` | `CartesianSplitView` | Raw data (series → cells) |
| `this.getDataMetrics()` | `DataMetrics` | Pre-computed min/max statistics |
| `this.getProperties()` | `ChartrixProperties` | All chart config properties |
| `this.getXflVariable(name)` | `unknown` | Read an XflVariable by key |
| `this.getDataPoint(seriesIdx, catIdx)` | `DataPoint \| undefined` | Single data point by index |
| `this.createFilterByCategoryAddress(subset)` | `ViewItemFilter` | Filter function matching by category address (incl. axis labels) |
| `this.createFilterByAddress(subset)` | `ViewItemFilter` | Filter function matching by series address |
| `this.createFilterByDataValue(predicate)` | `ViewItemFilter` | Filter function matching by data value |
| `this.createLabel(text, x, y, ref?)` | `Label` | Create new label element |
| `this.createRect(x, y, w, h, ref?)` | `Rect` | Create new rectangle |
| `this.createCircle(x, y, r, ref?)` | `Circle` | Create new circle |
| `this.createImage(url, x, y, w, h, ref?)` | `Image` | Create new image element |
| `this.createWedge(x, y, w, h, ref?)` | `Wedge` | Create new wedge marker |

> Full API details and data structure shapes → `references/api.md`

---

## ViewItem — writable properties

Every element in `viewModel` is a `ViewItem`. The most commonly mutated properties in XFL:

```typescript
viewItem.semanticType          // SemanticType enum string — for identification
viewItem.key                   // string — unique ID, useful for matching
viewItem.x.scaledValue.dataPoint      // DataPoint — X-axis anchor
viewItem.x.scaledValue.zeroReference  // DataPoint — X-axis zero/origin anchor
viewItem.y.scaledValue.dataPoint      // DataPoint — Y-axis anchor
viewItem.y.scaledValue.zeroReference  // DataPoint — Y-axis zero/origin anchor
viewItem.styles                // SerializedStyles (CSS-in-JS) — visual overrides
viewItem.text                  // string — label text (Label items only)
viewItem.color                 // string — color (Label, Line items)
viewItem.rowIndex              // number — which data row this belongs to
viewItem.columnIndex           // number — which data column this belongs to
viewItem.zIndex                // number — rendering order
```

Absolute pixel helpers (read-only):
```javascript
viewItem.getAbsoluteXPosition()   // left edge in px
viewItem.getAbsoluteYPosition()   // top edge in px
viewItem.getAbsoluteWidth()       // width in px
viewItem.getAbsoluteHeight()      // height in px
```

---

## getData() data structure

```
CartesianSplitView
  .data: CartesianDataRow[]         ← one per series (row)
    [i]: CartesianDataRow
      .cellRepresentingDataPoint: DataPoint   ← the row-level aggregate
      [j]: CartesianDataCell                  ← individual cells
        .cellRepresentingDataPoint: DataPoint
          .value: number
          .formattedValue: string
          ... (address, etc.)
```

**Access pattern used in the AutoHighlight example:**
```javascript
const series0 = this.getData().data[0];       // first series
const cell = series0[2];                       // third category
const dp = cell.cellRepresentingDataPoint;     // the DataPoint
dp.value                                       // numeric value
```

---

## Workflow: writing an XFL script

1. **Understand what should change** — identify which `SemanticType`(s) to target.
   See `references/semantic-types.md` for the full list with descriptions.

2. **Access the data** — use `this.getData()` for raw values or `this.getDataMetrics()`
   for pre-computed statistics (min, max, per series, per category).

3. **Iterate `this.viewModel`** — use `forEach` with a `switch` on `semanticType` or
   filter helpers (`this.bars.filter(...)`) for targeted access.

4. **Mutate or create** — modify existing `viewItem` properties in-place, or call
   `this.createLabel(...)` etc. to add new elements. New elements are automatically
   added to the viewModel.

5. **Test in chartrix** — paste the script into the XFL editor and observe the result.

> Code patterns for common tasks → `references/patterns.md`
