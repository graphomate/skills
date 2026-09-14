# CFL Patterns Reference

Verified examples from graphomate's internal CFL pattern library.

---

## API quick-clarification: `appendStyle` vs `appendStyles`

These are two distinct methods:

```javascript
// appendStyle(elementName, cssObject) — targets ONE sub-element by name
cell.appendStyle("root", { backgroundColor: "#c6efce" });
cell.appendStyle("textAlignmentWrapper", { justifyContent: "flex-start" });

// appendStyles(object) — targets MULTIPLE sub-elements in one call
cell.appendStyles({
  root: { backgroundColor: "#c6efce" },
  text: { fontWeight: "bold" }
});
```

---

## Data Access Patterns

### Cell value via getData()
```javascript
const value = cell.getData().cellRepresentingDataPoint.value; // number | null
```

### Value at arbitrary position
```javascript
// x = data column index, y = data row index (both excluding headers)
const neighborValue = cell._data.data[x + offset][y].cellRepresentingDataPoint.value;
```

### Row address map (member keys per dimension)
```javascript
const rowAddresses = cell.getMatrixData().rowAddresses; // array indexed by dataRowIndex
const thisRowAddress = rowAddresses[cell.getDataRowIndex()]; // { "Dim.Key": "MemberKey" }
const prevRowAddress = rowAddresses[cell.getDataRowIndex() - 1];
```

### Dimension metadata
```javascript
// From cell's own data context:
const dimensions = cell.getData().dataSet.metadata.dimensions;
// From full matrix:
const dimensions = cell.getMatrixData().metadata.dimensions;

// Each dimension has: .key, .axis ("ROWS"|"COLUMNS"), .axisIndex, .members[]
// Each member has: .key, .name

// Filter to row dimensions only:
const rowDimensions = dimensions.filter(dim => dim.axis === "ROWS");

// Find deepest non-aggregate row dimension for current cell:
const deepestDim = rowDimensions.reduce((max, cur) =>
  cur.axisIndex > max.axisIndex && cell.getAddress()[cur.key] !== "Overall" ? cur : max,
  rowDimensions[0]
);
```

---

## Styling

### Black Pin Heads
```javascript
if (cell.getCellType() === "PIN_CHART") {
  cell.appendStyles({
    pinHeadFill:   { fill: "black" },
    pinHeadBorder: { stroke: "black" }
  });
}
```

### Prevent Chart Overflow into Adjacent Cells
```javascript
if (cell.getCellType() === "PIN_CHART" || cell.getCellType() === "BAR_CHART") {
  cell.appendStyle("root", { overflow: "hidden" });
}
```

### White Labels Inside Bars
```javascript
// When bar value exceeds a % of the max, the label renders inside — make it white.
// Requires CflVariable "Prozent" (0-100)
const maxValue = cell.getDataMetrics().maxValue * (cell.getCflVariable("Prozent") * 0.01);
if (cell.getCellType() === "BAR_CHART" && cell.getValue() > maxValue) {
  cell.appendStyle("root", { color: "white" });
}
```

### Alternating Row Colors
```javascript
if (
  (cell.getCellType() !== "HEADER" || cell.getHeaderAxis() !== "COLUMNS") &&
  cell.getCellType() !== "CANTON_HEADER"
) {
  cell.appendStyle("root", { backgroundColor: "#ffffff" });
  if (cell.getMatrixRowIndex() % 2 === 1) {
    cell.appendStyle("root", { backgroundColor: "#f5f5f5" });
  }
}
```

### Column Header Alignment (left / center / right)
Uses CflVariable `"headerAlignment"` (`"left"`, `"right"`, or default `"center"`).
```javascript
if (cell.getCellType() === "HEADER" && cell.getHeaderAxis() === "COLUMNS") {
  const align = cell.getCflVariable("headerAlignment") || "center";
  if (align === "left") {
    cell.appendStyles({
      textAlignmentWrapper: { justifyContent: "flex-start" },
      sortSignCenterDummy:  { display: "none" }
    });
  } else if (align === "right") {
    cell.appendStyles({
      textAlignmentWrapper: { justifyContent: "flex-start", flexDirection: "row-reverse" },
      sortSignCenterDummy:  { display: "none" }
    });
  }
}
```

### Left-Align All Labels in a Specific Column
```javascript
if (cell.getMatrixColumnIndex() === 1) {
  cell.appendStyle("root",               { justifyContent: "flex-start" });
  cell.appendStyle("textSpacer",         { flexGrow: "0" });
  cell.appendStyle("textAlignmentWrapper", { justifyContent: "left" });
}
```

### Make Result/Sum Rows Bold
```javascript
if (cell.isRowResultCell()) {
  cell.appendStyle("root", { fontWeight: "bold" });
}
```

### Visual Borders / Frames
```javascript
// Vertical divider line after column 1
if (cell.getMatrixColumnIndex() === 1) {
  cell.appendStyle("root", { borderRight: "0.1em solid" });
}

// Horizontal line + background at a specific address intersection
const productGroup = cell.getAddress()["Product_Group"];
const salesRep     = cell.getAddress()["SalesRep"];
if (productGroup === "Overall" && salesRep === "Johnny Marr") {
  cell.appendStyle("root", { borderBottom: "0.05em solid", backgroundColor: "lightgrey" });
}
```

### Visual Spacing Between Rows and Columns
```javascript
// Add white space below specific rows (by matrixRowIndex)
const spacerRows = [5, 9];
if (spacerRows.includes(cell.getMatrixRowIndex())) {
  cell.appendStyle("root", { borderBottom: "4px solid rgb(255,255,255)" });
}

// Add margin before specific columns (set columnMargin to 0 in options first)
const spacerCols = [3, 5, 6];
cell.appendStyle("root", {
  marginLeft: spacerCols.includes(cell.getMatrixColumnIndex()) ? "14px" : "0.2em"
});
```

### Background Image in Header Cell
```javascript
if (cell.getCellType() === "HEADER" && cell.getText() === "Germany") {
  cell.appendStyle("root", {
    backgroundRepeat:   "no-repeat",
    backgroundPosition: "right",
    backgroundSize:     "contain",
    backgroundColor:    "white",
    backgroundImage:    "url('https://example.com/flag-germany.jpg')"
  });
}
```

### Micro Pie Charts (background-image trick)
```javascript
if (cell.getCellType() !== "NUMERIC") return;
if (cell.getDataRowHierarchyLevel() !== cell.getMaxRowHierarchyLevel()) return;
const sumSize  = cell.getDataMetrics().maxValuePerColumnPerHierarchyLevel[cell.getDataColumnIndex()][0];
const dataValue = cell.getValue();
if (dataValue === null) return;
const angle = 360 * (dataValue / sumSize);
cell.appendStyle("root", {
  backgroundRepeat: "no-repeat",
  backgroundSize:   "1.5em 1.5em",
  backgroundImage:  `radial-gradient(circle at 50%, rgba(0,0,0,0), rgba(0,0,0,0) 60%, #444 70%, white 70%),conic-gradient(#aaa ${angle}deg, white 0)`
});
```

### Butterfly Layout (Mirror Two Columns)
```javascript
// Swaps columns 0 and 3 visually. Only works with exactly 2 data columns + 1 header column.
if (cell.getMatrixColumnIndex() === 0) {
  cell.appendStyle("root", { gridColumnStart: "3" });
} else if (cell.getMatrixColumnIndex() === 3) {
  cell.appendStyle("root", { gridColumnStart: "0" });
}
```

---

## Text / Labels

### Rename a Column Header
```javascript
if (
  cell.getCellType() === "HEADER" &&
  cell.getHeaderAxis() === "COLUMNS" &&
  cell.getText() === "Stückzahl"
) {
  cell.setText("Quantity");
}
```

### Conditional Traffic Light (emoji)
```javascript
const upperThreshold = 100000;
const lowerThreshold = 0;
if (cell.getCellType() === "NUMERIC" && cell.getDataRowHierarchyLevel() === cell.getMaxRowHierarchyLevel()) {
  const val = cell.getValue();
  const text = cell.getText();
  if (val < lowerThreshold) {
    cell.setText(text + " 🔴");
  } else if (val < upperThreshold) {
    cell.setText(text + " 🟡");
  } else {
    cell.setText(text + " 🟢");
  }
}
```

### Conditional Value Color by Column Index
```javascript
if (cell.getCellType() === "NUMERIC" && cell.getDataColumnIndex() === 3) {
  cell.appendStyle("root", { color: cell.getValue() < 100 ? "red" : "green" });
}
```

### Append Year from HyperAxis to Column Header (Power BI)
```javascript
if (
  cell.getAddress()["graphomate.internal.measures"] === "Orders.Total Profit" &&
  cell.getCellType() === "HEADER"
) {
  const year = cell.getMatrixData().metadata.dimensions
    .find(dim => dim.key.endsWith("Jahr")).members[0].name;
  cell.setText(cell.getText() + " " + year);
}
```

### Icon Column Using Row Header Name (BMW example pattern)
Displays an image per row by constructing a URL from the row's member name:
```javascript
if (
  cell.getCellType() === "NUMERIC" &&
  cell.getAddress()["graphomate.internal.measures"] === "calculation_1"
) {
  const rowDimensions = cell.getData().dataSet.metadata.dimensions
    .filter(dim => dim.axis === "ROWS");
  const deepestDim = rowDimensions.reduce((max, cur) =>
    cur.axisIndex > max.axisIndex && cell.getAddress()[cur.key] !== "Overall" ? cur : max,
    rowDimensions[0]
  );
  const memberKey  = cell.getAddress()[deepestDim.key];
  const member     = deepestDim.members.find(m => m.key === memberKey);
  const url        = "url('https://example.com/icons/" + member.name + ".png')";
  cell.setText("");
  cell.appendStyle("root", {
    pointerEvents:      "none",
    backgroundRepeat:   "no-repeat",
    backgroundPosition: "right",
    backgroundSize:     "contain",
    backgroundColor:    "white",
    backgroundImage:    url
  });
}
```

---

## Heatmap / Scaling

### Heatmap (diverging color palette, leaf-level only)
Normalizes values at the deepest hierarchy level and maps them to a multi-stop color palette.
Color palettes: https://colorbrewer2.org
```javascript
if (
  cell.getCellType() === "NUMERIC" &&
  cell.getDataRowHierarchyLevel() === cell.getMaxRowHierarchyLevel()
) {
  const m = cell.getDataMetrics();
  const maxVal = m.maxValuePerRowHierarchyLevel[cell.getMaxRowHierarchyLevel()];
  const minVal = m.minValuePerRowHierarchyLevel[cell.getMaxRowHierarchyLevel()];

  // Linear interpolator between uniformly distributed color stops
  const interpolate = function(colors) {
    return function(t) {
      if (t >= 1) return colors[colors.length - 1];
      const i = Math.min(Math.floor(t * (colors.length - 1)), colors.length - 2);
      const f = (t * (colors.length - 1)) % 1;
      return {
        r: Math.round(colors[i].r + (colors[i+1].r - colors[i].r) * f),
        g: Math.round(colors[i].g + (colors[i+1].g - colors[i].g) * f),
        b: Math.round(colors[i].b + (colors[i+1].b - colors[i].b) * f)
      };
    };
  };

  // RdYlGn-4 from colorbrewer2.org
  const palette = interpolate([
    { r: 215, g: 25,  b: 28  },
    { r: 253, g: 174, b: 97  },
    { r: 166, g: 217, b: 106 },
    { r: 26,  g: 150, b: 65  }
  ]);

  const raw = cell.getData().cellRepresentingDataPoint.value || 0;
  // normalize 0→maxVal (use (raw - minVal)/(maxVal - minVal) for min→max range)
  const t = raw / maxVal;
  const c = palette(t);
  cell.appendStyle("root", {
    color:      `rgb(${c.r},${c.g},${c.b})`,
    fontWeight: "bold"
  });
}
```

### Font Size Scaling (linear, proportional to value)
```javascript
if (cell.getCellType() === "NUMERIC") {
  const m        = cell.getDataMetrics();
  const maxVal   = m.maxValue;
  const minVal   = m.minValue;
  const raw      = cell.getData().cellRepresentingDataPoint.value || 0;
  const t        = (raw - minVal) / (maxVal - minVal);
  const minSize  = 8;
  const maxSize  = 15;
  cell.appendStyle("root", { fontSize: ((maxSize - minSize) * t + minSize) + "px" });
}
```

---

## Hiding Rows and Columns

### Hide Overall / Root Node Row (by hierarchy level)
```javascript
if (
  (cell.getCellType() === "HEADER" && cell.getHeaderAxis() === "ROWS" && cell.getHierarchyLevel() === 0) ||
  (["NUMERIC", "BAR_CHART"].includes(cell.getCellType()) && cell.getDataRowHierarchyLevel() === 0)
) {
  cell.appendStyle("root", { display: "none" });
}
```

### Hide Overall by Text / Address Match
```javascript
if (cell.getCellType() === "HEADER" && cell.getHeaderAxis() === "ROWS" && cell.getText() === "Overall") {
  cell.appendStyle("root", { display: "none" });
} else if (cell.getCellType() !== "HEADER" && cell.getCellType() !== "CANTON_HEADER") {
  const rowDimensions = cell.getData().dataSet.metadata.dimensions
    .filter(dim => dim.axis === "ROWS");
  const allOverall = rowDimensions.every(dim => cell.getAddress()[dim.key] === "Overall");
  if (allOverall) {
    cell.appendStyle("root", { display: "none" });
  }
}
```

### Hide Specific Members by Key
```javascript
const memberKeysToHide = ["01Jan", "01Feb"];
if (cell.getCellType() !== "CANTON_HEADER") {
  const addressKeys = Object.values(cell.getAddress());
  if (memberKeysToHide.some(k => addressKeys.includes(k))) {
    cell.appendStyle("root", { display: "none" });
  }
}
```

### Hide Specific Members by Dimension Key
```javascript
const dimensionKey     = "Datenquelle.Region";
const memberKeysToHide = ["DE", "AT"];
if (cell.getCellType() !== "CANTON_HEADER") {
  if (memberKeysToHide.includes(cell.getAddress()[dimensionKey])) {
    cell.appendStyle("root", { display: "none" });
  }
}
```

### Hide Column by Index
```javascript
const columnIndexToHide = 1;
if (
  cell.getCellType() !== "CANTON_HEADER" &&
  !(cell.getCellType() === "HEADER" && cell.getHeaderAxis() === "ROWS") &&
  cell.getMatrixColumnIndex() === columnIndexToHide
) {
  cell.appendStyle("root", { display: "none" });
}
```

### Hide First Data Row
```javascript
if (cell.getDataRowIndex() === 0) {
  cell.appendStyle("root", { display: "none" });
}
```

### Hide Second Header Row
```javascript
if (cell.getMatrixRowIndex() === 1) {
  cell.appendStyle("root", { display: "none" });
}
```

### Hide Rows Where First N Data Columns Are All Null
```javascript
if (
  (cell.getCellType() === "HEADER" && cell.getHeaderAxis() !== "COLUMNS") ||
  cell.getCellType() === "NUMERIC"
) {
  const allData   = cell.getMatrixData().data;
  const rowIdx    = cell.getDataRowIndex();
  const checkCols = 2; // adjust as needed
  let allNull     = true;
  for (let c = 0; c < checkCols; c++) {
    if (allData[c][rowIdx].cellRepresentingDataPoint.value !== null) {
      allNull = false;
      break;
    }
  }
  if (allNull) {
    cell.appendStyle("root", { display: "none" });
  }
}
```

---

## Using Neighbor Cell Values

### Read a Value from a Different Column in the Same Row
```javascript
if (
  (cell.getCellType() === "NUMERIC" || cell.getCellType() === "BACKGROUND_BAR") &&
  cell.getAddress()["graphomate.internal.measures"] === "Sum(Datenquelle.Revenue)" &&
  cell.getAddress()["Datenquelle.Year"] === "2021"
) {
  const offset        = 3; // columns to the right (including hidden columns)
  const neighborValue = cell._data.data[cell.getDataColumnIndex() + offset][cell.getDataRowIndex()]
    .cellRepresentingDataPoint.value;
  if (neighborValue !== null && neighborValue < 0.10 && neighborValue > 0.06) {
    cell.setText("🔴 " + cell.getText());
  }
}
```

---

## Header Manipulation

### Two-Part Column Header with Different Formatting (indicatorSign workaround)

CFL cannot style partial text within a single cell — `setText()` only accepts plain text.
The workaround is to split the header text across two independently styleable slots:
`text` (normal cell text) and `indicatorSign` (normally the collapse indicator).

```javascript
// Example: "Jan 2016" → "Jan" normal, "2016" bold
if (cell.getCellType() === 'HEADER' && cell.getHeaderAxis() === 'COLUMNS') {
  const parts = cell.getText().split(' ');
  if (parts.length > 1) {
    // indicatorSign renders BEFORE text in the DOM → assign first part here
    cell.setIndicatorSign(parts[0]);
    cell.appendStyle('indicatorSign', { fontWeight: 'normal', paddingRight: '4px' });
    // text slot renders second → assign second part here
    cell.setText(parts[1]);
    cell.appendStyle('text', { fontWeight: 'bold' });
  }
}
```

Use `paddingRight` on `indicatorSign` for spacing — `marginLeft` on the `text` slot is
unreliable depending on the flex layout context. If `paddingRight` also has no effect,
use a non-breaking space as a safe fallback: `cell.setIndicatorSign(parts[0] + '\u00A0')`
— non-breaking spaces are never collapsed by the browser regardless of layout.

> **Note:** This workaround repurposes the collapse indicator slot. Do not combine it with
> `hierarchyNodeCollapsible: true` on column headers — the indicator sign would be overwritten.

### Suppress Repeating Row Headers for One Dimension (workaround)
Re-displays suppressed text and adds divider lines for a specific row header column:
```javascript
const TARGET_COL = 1; // zero-based column index of the dimension to un-suppress
if (
  cell.getCellType() === "HEADER" &&
  cell.getHeaderAxis() === "ROWS" &&
  cell.getMatrixColumnIndex() === TARGET_COL
) {
  // Add divider line
  const color     = cell.getProperties().rowDividerColor;
  const thickness = cell.getProperties().rowDividerThickness;
  cell.appendStyle("root", { borderBottom: `solid ${thickness} ${color}` });
  // Re-add text if suppressed
  if (cell.getText() === "") {
    const rowDimensions = cell.data.metadata.dimensions
      .filter(dim => dim.axis === "ROWS")
      .sort((a, b) => a.axisIndex - b.axisIndex);
    const dimension = rowDimensions[TARGET_COL];
    const member    = cell.data.rowAddresses[cell.getDataRowIndex()][dimension.key].member;
    cell.setText(member.name);
  }
}
```

### Highlight a Specific Row by Dimension + Member Key
```javascript
if (cell.getCellType() !== "CANTON_HEADER") {
  if (
    cell.getAddress()["Datenquelle.Region"] === "DE" &&
    cell.getAddress()["Datenquelle.Segment"] === "Overall"
  ) {
    cell.appendStyle("root", { backgroundColor: "papayawhip" });
  }
}
```

### Collapse Indicator Customization (emoji icons)
```javascript
if (cell.getCellType() === 'HEADER' && cell.getHeaderAxis() === 'ROWS') {
  if (cell.isResultCell()) {
    cell.appendStyle('root', { cursor: 'pointer' });
    cell.appendStyle('indicatorSign', { paddingRight: '0.5em' });
    cell.setIndicatorSign(cell.isCollapsed() ? '👉' : '👇');
  }
}
```

### Read Matrix Color Properties (goodColor / badColor)
Uses the matrix's own configured colors instead of hardcoding — stays in sync with the
properties panel automatically:
```javascript
if (cell.getCellType() === 'NUMERIC') {
  const positiveColor = cell.getProperty('goodColor') || 'green';
  const negativeColor = cell.getProperty('badColor') || 'red';
  const val = cell.getValue();
  if (val > 0) {
    cell.appendStyle('root', { color: positiveColor });
  } else if (val < 0) {
    cell.appendStyle('root', { color: negativeColor });
  }
}
```

> ⚠️ The CFL editor example for this pattern contains a bug: `else if (this.getValue() < 0)`
> — mixing the old `this.` API with `cell.`. Always use `cell.` consistently.

### Show Raw Member Keys Instead of Display Names
Replaces header text with the raw data source key (e.g. `"DE"` instead of `"Germany"`).
Uses `cell._data` (internal) for dimension metadata:
```javascript
if (cell.getCellType() === 'HEADER') {
  if (cell.getHeaderAxis() === 'ROWS') {
    const dimensions = cell._data.metadata.dimensions
      .filter(dim => dim.axis === "ROWS")
      .sort((a, b) => a.axisIndex - b.axisIndex);
    cell.setText(cell.getAddress()[dimensions[cell.getMatrixColumnIndex()].key]);
  } else {
    const dimensions = cell._data.metadata.dimensions
      .filter(dim => dim.axis === "COLUMNS")
      .sort((a, b) => a.axisIndex - b.axisIndex);
    cell.setText(cell.getAddress()[dimensions[cell.getMatrixRowIndex()].key]);
  }
}
if (cell.getCellType() === 'CANTON_HEADER') {
  const isLastHeaderRow = cell.getMatrixRowIndex() === cell.getColumnHeaderCount() - 1;
  const rowDimensions = cell.getMatrixData().metadata.dimensions
    .filter(dim => dim.axis === "ROWS")
    .sort((a, b) => a.axisIndex - b.axisIndex);
  if (isLastHeaderRow) {
    cell.cellModel.text = rowDimensions[cell.getMatrixColumnIndex()].key;
  }
}
cell.appendStyle('root', { userSelect: 'all' });
```

> Note: `cell.cellModel.text` is an undocumented internal property used here as the only
> way to set text on `CANTON_HEADER` cells. May break in future versions.

### Tableau: Highlight Column Matching a Date Parameter
```javascript
// Requires CflVariable "CFL_Test" with value like "2024-03" (YYYY-MM)
const targetCol = Number.parseInt(cell.getCflVariable("CFL_Test").split("-")[1]);
if (cell.getMatrixColumnIndex() === targetCol) {
  cell.appendStyle("root", { backgroundColor: "papayawhip" });
}
```

---

## Notes on `cell._data` (internal access)

Some patterns access `cell._data` directly — this is an internal property not in the public API:

- `cell._data.data[col][row]` — raw data array (same as `getMatrixData().data`)
- `cell._data.rowAddresses[rowIdx]` — row address map (same as `getMatrixData().rowAddresses`)

These work but may break in future matrix versions. Prefer the public API equivalents
(`cell.getMatrixData()`) where possible.
