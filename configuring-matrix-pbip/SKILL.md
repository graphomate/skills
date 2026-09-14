---
name: configuring-matrix-pbip
description: >
  Use this skill whenever a user wants to configure, modify, or build a Power BI report
  using the graphomate matrix visual. Triggers include: any mention of "matrix", "graphomate matrix",
  "gm matrix", "Power BI table", ".pbip", "IBCS table", "financial table", "P&L table",
  "variance table", or requests to help display financial data in tabular form (P&L, budget vs actual,
  forecast, revenue, cost breakdowns). Also trigger when the user wants to set up scenarios
  (AC, PY, BU, FC) in a table, configure deviation columns, number formats, bar charts in cells,
  sparklines, or hierarchy display. If the user uploads a visual.json or .pbip file and asks for
  help with a table visual, always use this skill.
---

# graphomate matrix – Power BI Skill

This skill helps configure the **graphomate matrix** custom visual inside Power BI Project (`.pbip`)
files. The matrix is a table component supporting IBCS-compliant financial reporting with
scenarios, deviation columns, in-cell visualizations, and CFL scripting.

---

## How .pbip files work

A `.pbip` is a **folder** (not a single file). When saved in enhanced report format (PBIR),
every visual lives in its own JSON file:

```
MyReport.pbip
MyReport.Report/
  definition/
    pages/
      [PageId]/
        visuals/
          [VisualId]/
            visual.json   ← matrix config lives here
```

> **First step when a user shares a file:** Ask them to share the relevant `visual.json`.
> Find the file where `"visualType"` contains `"graphomate"` and `"matrix"`.

---

## How matrix config is stored in visual.json

Each property is stored as an individually stringified JSON value, grouped under `visual.objects`.

```json
{
  "visual": {
    "objects": {
      "Data":         [{ "properties": { ... } }],
      "Labels":       [{ "properties": { ... } }],
      "Axes":         [{ "properties": { ... } }],
      "ChartSpecific":[{ "properties": { ... } }],
      "InputOutput":  [{ "properties": { ... } }]
    }
  }
}
```

Each value is a stringified JSON array or primitive wrapped in single quotes:
```json
"deviationConfigs": { "expr": { "Literal": { "Value": "'[{\"enabled\":true,...}]'" } } }
```

## Property → Group mapping

| Property | Group |
|---|---|
| `scenarios`, `scenarioAssignment`, `deviationConfigs`, `customCalculationConfigs` | `Data` |
| `nRestConfigs`, `sortByMember`, `moveDimensionConfigs`, `aggregationConfigs` | `Data` |
| `aggregationType`, `aggregationNodeName`, `followingResults` | `Data` |
| `hyperAxisConfigs`, `removeDimensionConfigs`, `valueFilterConfigs` | `Data` |
| `showScenariosInColumnHeaders`, `globalDataTypes`, `selectionType` | `Data` |
| `showTitle`, `titleText`, `titleFontSize`, `fontSize`, `fontFamily`, `fontColor` | `Labels` |
| `numberFormatAssignment`, `dataTextAlign`, `headerElipsis` | `Labels` |
| `cflRules`, `cflVariables`, `customCss` | `Labels` |
| `hierarchyNode*`, `backgroundColor`, `padding` | `Labels` |
| `showFooter`, `footerText`, `footerFontSize` | `Labels` |
| `columnWidth`, `widthPerColumn`, `columnMargin` | `Axes` |
| `suppressRepeatingColumnHeader`, `columnHeaderTextAlign` | `Axes` |
| `headerRowDividers`, `headerRowDividerThickness`, `headerRowDividerColor` | `Axes` |
| `suppressRepeatingRowHeader`, `rowDividers`, `rowDividerThickness`, `rowDividerColor` | `Axes` |
| `initialRowExpandLevel`, `alternateRowStyling`, `crossTabRowHeader` | `Axes` |
| `barChartAssignment`, `pinChartAssignment`, `sparklineAssignment` | `ChartSpecific` |
| `sparkbarAssignment`, `backgroundBarAssignment`, `hyperTrendAssignment` | `ChartSpecific` |
| `goodColor`, `badColor`, `outlierStyle`, `deviationChartLabelSize` | `ChartSpecific` |
| `backgroundBarOpacity`, `backgroundBarGoodColor`, `backgroundBarBadColor` | `ChartSpecific` |
| `randomId`, `allPropertiesCache`, `editabilityAssignment` | `InputOutput` |
| `liveTemplate`, `commentingBackendServerUrl`, commenting properties | `InputOutput` |

---

## addressSubset — critical format rules

`addressSubset` filters which cells a config applies to. Two key formats:

```json
// Apply to ALL cells
"addressSubset": {}

// Apply to a specific dimension member (fully qualified PBI reference)
"addressSubset": { "Datenquelle.Version": ["Actual"] }

// Apply to a specific Power BI measure
"addressSubset": { "graphomate.internal.measures": ["Sum(Datenquelle.Revenue)"] }

// Apply to a calculated/deviation member by its memberKey
"addressSubset": { "graphomate.internal.measures": ["calculation_1"] }
```

> **`graphomate.internal.measures`** is the special key for Power BI measure columns.
> Use the full DAX expression string (e.g. `"Sum(Datenquelle.Revenue)"`) as the member value.
> Calculated members (`customCalculationConfigs`) and deviation members (`deviationConfigs`)
> are referenced by their `memberKey` / `deviationMemberKey`.

---

## Scenarios

**`Data` → `scenarioAssignment`** — assigns IBCS scenario styles to measures or members:

```json
[
  {
    "enabled": true,
    "scenarioId": "AC",
    "addressSubset": { "graphomate.internal.measures": ["Sum(Datenquelle.Revenue)"] },
    "description": "",
    "relationId": null
  },
  {
    "enabled": true,
    "scenarioId": "BU",
    "addressSubset": { "graphomate.internal.measures": ["Sum(Datenquelle.Budget)"] },
    "description": "",
    "relationId": null
  }
]
```

| Scenario ID | Meaning | IBCS Style |
|---|---|---|
| `AC` | Actual | Dark gray, filled |
| `PY` | Prior Year | Light gray |
| `BU` | Budget/Plan | Outlined |
| `FC` | Forecast | Dashed |

**`Data` → `globalDataTypes`** — defines the scenario style catalogue (generated by matrix UI):
```json
{
  "timestamp": 1,
  "datatypes": [
    { "short": "AC", "color": "#222222", "filltype": "filled", "shape": "rect", "patterntype": "solid" },
    { "short": "BU", "color": "#222222", "filltype": "empty",  "shape": "rect", "patterntype": "solid" },
    { "short": "FC", "color": "#222222", "filltype": "empty",  "shape": "rect", "patterntype": "dashed" },
    { "short": "PY", "color": "#999999", "filltype": "filled", "shape": "rect", "patterntype": "solid" }
  ]
}
```

---

## Deviation Columns

Adds calculated difference columns (absolute or %) to the table.

**`Data` → `deviationConfigs`:**
```json
[
  {
    "enabled": true,
    "deviationMemberKey": "dev_ac_bu",
    "deviationMemberName": "∆abs",
    "deviationType": "ABSOLUTE",
    "targetDimension": "Datenquelle.Version",
    "minuendMember": "Actual",
    "subtrahendMember": "Budget",
    "addressSubset": {},
    "description": "",
    "relationId": "dev_ac_bu"
  }
]
```

> `deviationType`: `"ABSOLUTE"` or `"PERCENT"`
> The `deviationMemberKey` becomes the member reference for use in other configs
> (e.g. `addressSubset: { "graphomate.internal.measures": ["dev_ac_bu"] }`).

---

## Custom Calculations

Formula-based members using measure references with `${...}` syntax.

**`Data` → `customCalculationConfigs`:**
```json
[
  {
    "enabled": true,
    "memberKey": "gm_pct",
    "memberName": "GM %",
    "targetDimension": "graphomate.internal.measures",
    "expression": "(${Sum(Datenquelle.Revenue)} - ${Sum(Datenquelle.COGS)}) / ${Sum(Datenquelle.Revenue)}",
    "addressSubset": {},
    "description": "",
    "relationId": null
  }
]
```

> Reference Power BI measures with their full DAX string wrapped in `${...}`.

---

## Number Formatting

**`Labels` → `numberFormatAssignment`:**
```json
[
  {
    "enabled": true,
    "description": "Thousands",
    "addressSubset": {},
    "localeId": "en-US",
    "format": "number",
    "abbreviation": "thousand",
    "fractionalDigits": 1,
    "prefix": "",
    "suffix": "",
    "scalingFactor": 1,
    "thousandSeparator": null,
    "decimalSeparator": null,
    "totalDigits": null,
    "zeroOutput": null,
    "nullOutput": "",
    "infinityOutput": "∞",
    "roundingMethod": "halfAwayFromZero",
    "negativeSign": "minus",
    "explicitPositiveSign": false,
    "relationId": null
  },
  {
    "enabled": true,
    "description": "Percent for calculated member",
    "addressSubset": { "graphomate.internal.measures": ["gm_pct"] },
    "localeId": "en-US",
    "format": "percent",
    "fractionalDigits": 1,
    "abbreviation": "auto",
    "scalingFactor": 1,
    "prefix": "", "suffix": "",
    "thousandSeparator": null, "decimalSeparator": null,
    "totalDigits": null, "zeroOutput": null,
    "nullOutput": "", "infinityOutput": "∞",
    "roundingMethod": "halfAwayFromZero",
    "negativeSign": "minus",
    "explicitPositiveSign": false,
    "relationId": null
  }
]
```

`abbreviation` options: `"auto"`, `"thousand"`, `"million"`, `"billion"`, `"none"`
`format` options: `"number"`, `"percent"`

---

## In-Cell Visualizations

### Bar Charts

**`ChartSpecific` → `barChartAssignment`:**
```json
[
  {
    "enabled": true,
    "description": "Bar for deviation",
    "addressSubset": { "graphomate.internal.measures": ["dev_ac_bu"] },
    "axisScenarioId": "",
    "scenarioId": "",
    "comparisonGroup": "",
    "showLabels": true,
    "centerAxis": false,
    "negativeValueIsGood": false,
    "useOutlierThreshold": false,
    "positiveOutlierThreshold": 1000,
    "negativeOutlierThreshold": -1000,
    "useSpecificGoodColor": false,
    "goodColor": "#8cb400",
    "useSpecificBadColor": false,
    "badColor": "#ff0000",
    "relationId": null
  }
]
```

### Background Bars

**`ChartSpecific` → `backgroundBarAssignment`** — bars behind numeric values for visual scaling:
```json
[
  {
    "enabled": true,
    "description": "Background for PY",
    "addressSubset": { "graphomate.internal.measures": ["Sum(Datenquelle.PriorYear)"] },
    "negativeValueIsGood": false,
    "useOutlierThreshold": false,
    "useSpecificGoodColor": false,
    "goodColor": "#4dacc6",
    "useSpecificBadColor": false,
    "badColor": "#c6674d",
    "useSpecificOpacity": true,
    "opacity": 0.2,
    "relationId": null
  }
]
```

---

## Column Widths

**`Axes` → `widthPerColumn`** — array of widths by column index:
```json
[160, -1, -1, -2, -1]
```

- Positive integer: fixed pixel width
- `-1`: auto width
- `-2`: meaning TBD (observed in real configs — appears to collapse/hide column)

---

## Hierarchy Display

| Property (`Labels`) | Default | Purpose |
|---|---|---|
| `hierarchyNodeBold` | `true` | Bold text for hierarchy nodes |
| `hierarchyNodeIndentation` | `"1.2em"` | Indentation per level |
| `hierarchyNodeCollapsible` | `true` | Show expand/collapse indicator |
| `hierarchyNodeRowDividers` | `true` | Divider lines below hierarchy nodes |
| `initialRowExpandLevel` | `"\"null\""` | Collapse to level on load (`"\"null\""` = fully expanded) |
| `showRootNodeRow` (`Data`) | `true` | Show/hide top-level aggregate row |
| `followingResults` (`Data`) | `false` | `true` = totals appear after children |

> Note: `initialRowExpandLevel` in visual.json is double-stringified: `'"null"'` for null,
> `'"2"'` to collapse to level 2.

---

## Sorting

**`Data` → `sortByMember`** — define manual member order for a dimension:
```json
[
  {
    "enabled": true,
    "description": "",
    "dimensionKey": "Datenquelle.Product",
    "memberKeys": ["Apple juice", "Beer", "Cola", "Overall"]
  }
]
```

---

## CFL Scripting

`cflRules` and `cflVariables` live in the **`Labels`** group.
For writing scripts → use the **writing-matrix-cfl skill**.

Config-level setup:
```json
// Labels → cflRules
[{ "enabled": true, "name": "My Rule", "script": "// JS here using cell.xxx()" }]

// Labels → cflVariables
[{ "key": "threshold", "value": "0.1" }]
```

> Note: single quotes inside scripts must be doubled (`''`) because the entire value
> is wrapped in outer single quotes in the Power BI `Value` field.

---

## How to apply config to visual.json

### Step 1 — Find the file
Open the `.pbip` folder in VS Code. Navigate to:
```
MyReport.Report/definition/pages/[PageId]/visuals/[VisualId]/visual.json
```
Identify the matrix by `"visualType"` containing `"matrix"`.

### Step 2 — Replace a value
Locate the property in its group and replace the content between the outer single quotes:
```json
"deviationConfigs": { "expr": { "Literal": { "Value": "'← YOUR JSON HERE '" } } }
```

### Step 3 — Reload in Power BI Desktop
Save → switch to Power BI Desktop → click **Reload** when prompted.

> **Tip:** Use VS Code with Prettier. JSON syntax errors cause Power BI to silently revert to defaults.

---

## Communication tips

- Say "the table" or "the matrix", not "the visual".
- Avoid saying `addressSubset` — say "which columns/rows this applies to".
- Confirm the exact Power BI **table name** and **measure name** (DAX expression) before writing config.
- For CFL scripting requests → use the **writing-matrix-cfl skill**.
