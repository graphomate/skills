---
name: writing-matrix-cfl
description: >
  Use this skill whenever a user wants to write, debug, or understand CFL scripts for the
  graphomate matrix visual. CFL (Cell Formatting Language) is a JavaScript-based scripting
  dialect that runs per-cell at render time, enabling conditional formatting, dynamic colors,
  custom text, icons, and data-driven styling. Use this skill for any request involving
  coloring cells by value, bolding rows dynamically, showing traffic lights, setting background
  colors based on thresholds, adding custom formatted values, changing scenarios at runtime,
  hiding rows or columns, or any scripted customization of a matrix cell. Triggers include:
  "CFL", "cfl script", "cflRules", "color cells dynamically", "conditional formatting",
  "highlight rows", "traffic light", "bold if negative", "cell color", "hide row", "hide column",
  "heatmap", "scripting in matrix", or whenever the user pastes or asks about a script for
  the matrix. Always use this skill before writing or modifying any matrix scripting code.
---

# graphomate matrix – CFL Scripting Skill

CFL (Cell Formatting Language) is a JavaScript dialect executed inside graphomate matrix at
render time. The script runs **once per cell**. The cell context is accessed via the **`cell`
object** — all API methods are called as `cell.getValue()`, `cell.getCellType()`, etc.

> **Official API reference**: https://public.graphomate.com/matrix/cfl-doc/interfaces/cflenvironment.html
> **Verified code patterns**: see `references/patterns.md` — read it for any non-trivial task

---

## How CFL scripts are stored in visual.json

CFL properties live in the **`Labels`** group:

```json
"Labels": [{ "properties": {
  "cflRules":     { "expr": { "Literal": { "Value": "'[{\"enabled\":true,\"name\":\"My Rule\",\"script\":\"...\"}]'" } } },
  "cflVariables": { "expr": { "Literal": { "Value": "'[{\"key\":\"threshold\",\"value\":\"0.1\"}]'" } } }
}}]
```

**CflRule shape:** `{ "enabled": true, "name": "descriptive name", "script": "// JS code" }`

**CflVariable shape** — read via `cell.getCflVariable("key")`:
`{ "key": "myKey", "value": "0.1" }` — value is always a string; parse with `Number()` if needed.

> ⚠️ **Single quotes inside scripts stored in visual.json must be doubled** (`''`) because the
> entire JSON value is wrapped in outer single quotes in the Power BI `Value` field.

---

## Styling: `appendStyle` vs `appendStyles`

These are two distinct methods — use the right one:

```javascript
// appendStyle(elementName, cssObject) — ONE sub-element
cell.appendStyle("root", { backgroundColor: "#c6efce", fontWeight: "bold" });
cell.appendStyle("textAlignmentWrapper", { justifyContent: "flex-start" });

// appendStyles(object) — MULTIPLE sub-elements in one call
cell.appendStyles({
  root: { backgroundColor: "#c6efce" },
  text: { fontWeight: "bold" }
});
```

CSS property names are **camelCase**: `backgroundColor`, `fontWeight`, `fontSize`, `borderBottom`.

**Sub-element names by cell type:**

| Cell type | Available sub-elements |
|---|---|
| `NUMERIC` / `DATA` | `root`, `editableInput` |
| `HEADER` | `root`, `text`, `textAlignmentWrapper`, `indicatorSign`, `scenarioBar`, `scenarioBarBorder`, `scenarioBarFill`, `sortSign`, `sortSignCenterDummy` |
| `BAR_CHART` / `PIN_CHART` | `root`, `pinHeadFill`, `pinHeadBorder` |
| `BACKGROUND_BAR` | `root`, `backgroundBar` |

---

## The `cell` API — reference

### Getters

| Method | Returns | Notes |
|---|---|---|
| `cell.getValue()` | `number \| null` | Raw numeric value |
| `cell.getText()` | `string` | Display text |
| `cell.getAddress()` | `object` | `{ "Table.Dim": "MemberKey" }` — measures use `"graphomate.internal.measures"` |
| `cell.getCellType()` | `CellType` | See enum below |
| `cell.getScenario()` | `Scenario \| undefined` | Assigned scenario |
| `cell.getCflVariable(key)` | `unknown` | JSON-parsed CflVariable |
| `cell.getProperty(name)` | `unknown` | Any matrix property by name |
| `cell.getProperties()` | `object` | All matrix properties |
| `cell.getData()` | `CartesianDataCell` | `.cellRepresentingDataPoint.value`, `.dataSet.metadata.dimensions[]` |
| `cell.getMatrixData()` | `CartesianSplitView` | `.data[col][row]`, `.rowAddresses[rowIdx]`, `.metadata.dimensions[]` |
| `cell.getDataMetrics()` | `DataMetrics` | See metrics section below |
| `cell.getValueByDataIndices(x, y)` | `number \| null` | Value at col x, row y |
| `cell.getDataRowIndex()` | `number` | Row index (excluding headers) |
| `cell.getDataColumnIndex()` | `number` | Column index (excluding headers) |
| `cell.getMatrixRowIndex()` | `number` | Row index (including headers) |
| `cell.getMatrixColumnIndex()` | `number` | Column index (including headers) |
| `cell.getDataRowHierarchyLevel()` | `number` | Hierarchy depth of current row |
| `cell.getMaxRowHierarchyLevel()` | `number` | Max hierarchy depth in matrix |
| `cell.getHierarchyLevel()` | `number` | Level of a header cell |
| `cell.getHeaderAxis()` | `string` | `"ROWS"` or `"COLUMNS"` |
| `cell.getColumnWidth(idx?)` | `number` | Width in px |
| `cell.getEditable()` | `boolean` | Whether cell is editable |
| `cell.getIndicatorSign()` | `string` | Current collapse character |
| `cell.getStyles()` | `BaseCflStyles` | Current styles object |
| `cell.getCssClasses()` | `string[]` | Current CSS classes |

### Boolean checks

| Method | Purpose |
|---|---|
| `cell.isResultCell()` | Cell is an aggregate/total |
| `cell.isRowResultCell()` | Cell's row is a result row |
| `cell.isColumnResultCell()` | Cell's column is a result column |
| `cell.isNextRowResultCell()` | Next row is a result row |
| `cell.isCollapsed()` | Header cell is collapsed |
| `cell.isSelectedBy(name)` | Cell is in a named assignment (matched by `description` field) |

### Setters

| Method | Purpose |
|---|---|
| `cell.appendStyle(element, css)` | Merge CSS into one sub-element |
| `cell.appendStyles(obj)` | Merge CSS into multiple sub-elements |
| `cell.setStyles(obj)` | Replace all styles (destructive) |
| `cell.addCssClass(cls)` | Append CSS class |
| `cell.setText(text)` | Override displayed text |
| `cell.setScenario(id)` | Change scenario at runtime |
| `cell.setAxisScenario(id)` | Change axis scenario (bar chart cells) |
| `cell.setShowScenarioBar(bool)` | Show/hide scenario bar in headers |
| `cell.setIndicatorSign(char)` | Override collapse indicator character |

---

## CellType enum

```
"NUMERIC"        — numeric data cell (most scripts target this)
"HEADER"         — row or column header cell
"CANTON_HEADER"  — top-left corner cell (no data address — always skip)
"BAR_CHART"      — in-cell bar chart
"PIN_CHART"      — in-cell pin chart
"BACKGROUND_BAR" — background bar
"DATA"           — generic data cell
"BASE"           — base cell
```

---

## DataMetrics

```javascript
const m = cell.getDataMetrics();
m.maxValue                                          // global max
m.minValue                                          // global min
m.maxValuePerRow[rowIdx]                            // max in row
m.maxValuePerColumn[colIdx]                         // max in column
m.maxValuePerRowHierarchyLevel[level]               // max at hierarchy level
m.maxValuePerColumnPerHierarchyLevel[col][level]    // max per column per level
m.minValuePerRowHierarchyLevel[level]               // min at hierarchy level
```

---

## Data Access

```javascript
// Cell's own value (alternative to getValue()):
cell.getData().cellRepresentingDataPoint.value

// Full matrix data array [col][row]:
cell.getMatrixData().data[col][row].cellRepresentingDataPoint.value

// Row address map for any row:
cell.getMatrixData().rowAddresses[rowIdx]  // { "Table.Dim": "MemberKey" }

// Dimension metadata:
cell.getData().dataSet.metadata.dimensions  // [{key, axis, axisIndex, members:[{key, name}]}]
cell.getMatrixData().metadata.dimensions
```

---

## Standard Script Structure

```javascript
// 1. Guard: skip cell types this script doesn't apply to
if (cell.getCellType() !== "NUMERIC") return;

// 2. Get value; guard for null
const val = cell.getValue();
if (val === null) return;

// 3. Identify cell (by address, index, result status, etc.)
const address = cell.getAddress();
if (address["Datenquelle.Version"] !== "Actual") return;

// 4. Apply styling
cell.appendStyle("root", { backgroundColor: "#c6efce", color: "#276221" });
```

---

## Scope separation

| Task | Tool                                       |
|---|--------------------------------------------|
| Color/format cells at runtime | **CFL (this skill)**                       |
| Configure scenarios, deviations, number formats | **configuring-matrix-pbip skill**          |
| Bar charts, sparklines, pin charts in cells | **configuring-matrix-pbip skill**                    |
| Global CSS overrides | `customCss` property (configuring-matrix-pbip skill) |

---

## Key gotchas

- Context object is **`cell`**, not `this` — use `cell.getValue()`, never bare `getValue()`.
- **`appendStyle(element, css)`** vs **`appendStyles({element: css})`** are different methods — don't confuse them.
- `CANTON_HEADER` cells have no data address — always skip them: `if (cell.getCellType() === "CANTON_HEADER") return;`
- `cell.getAddress()` measure key is `"graphomate.internal.measures"`, not a table-qualified name.
- Single quotes in scripts inside visual.json `Value` fields must be **doubled** (`''`).
- Multiple rules run in array order; later rules can overwrite earlier styling on the same cell.
- `cell._data` (internal) works but may break in future versions — prefer public API.

---

## Where to find patterns

`references/patterns.md` contains verified, ready-to-use scripts for:
- Traffic lights, heatmaps, font size scaling
- Hiding rows/columns (by member key, by index, by null check, by Overall)
- Black pin heads, overflow prevention, white bar labels
- Column header alignment (left/center/right via CflVariable)
- Alternating row colors, borders/frames, visual spacing
- Butterfly layout, micro pie charts, background images in cells
- Neighbor cell value access, icon columns, header text manipulation
- Suppress repeating row headers workaround
- `getData().dataSet.metadata.dimensions` patterns for dynamic row dimension access
