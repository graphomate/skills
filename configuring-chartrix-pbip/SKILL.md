---
name: graphomate-chartrix
description: >
  Use this skill whenever a user wants to configure, modify, or build a Power BI report
  using the graphomate chartrix visual. Triggers include: any mention of "chartrix",
  "graphomate", "Power BI report", ".pbip", "IBCS chart", "financial chart", "variance chart",
  "waterfall chart in Power BI", or requests to help visualize financial data (P&L, budget vs actual,
  forecast, revenue, cost). Also trigger when the user wants to set up scenarios (AC, PY, BU, FC),
  configure deviations, or make their charts IBCS-compliant. If the user uploads a visual.json or
  .pbip file and asks for help with charts or visuals, always use this skill.
---

# graphomate chartrix – Power BI Skill

This skill helps configure the **graphomate chartrix** custom visual inside Power BI Project (`.pbip`) files so that financial analysts can create clear, IBCS-compliant charts without needing BI experts.

---

## How .pbip files work

A `.pbip` is a **folder** (not a single file). When saved in enhanced report format (PBIR), every visual lives in its own JSON file:

```
MyReport.pbip
MyReport.Report/
  definition/
    pages/
      [PageId]/
        visuals/
          [VisualId]/
            visual.json   ← chartrix config lives here
```

> **First step when a user shares a file:** Ask them to share the relevant `visual.json`. If they share the whole `.pbip` folder, navigate to the correct path above and find the file where `"visualType"` contains `"graphomate"`.

---

## How chartrix config is stored in visual.json

**Critical: the chartrix config is NOT one big JSON blob.** Each property is stored as its own individually stringified JSON value, grouped by category under `visual.objects`.

The structure looks like this:

```json
{
  "visual": {
    "visualType": "graphomatechartrix",
    "objects": {
      "ChartSpecific": [{ "properties": {
        "chartsLayout": { "expr": { "Literal": { "Value": "'[...]'" } } },
        "charts":       { "expr": { "Literal": { "Value": "'[...]'" } } }
      }}],
      "Data": [{ "properties": {
        "scenarios":             { "expr": { "Literal": { "Value": "'[...]'" } } },
        "deviationCalculations": { "expr": { "Literal": { "Value": "'[...]'" } } }
      }}],
      "Emphasis": [{ "properties": {
        "highlights": { "expr": { "Literal": { "Value": "'[...]'" } } }
      }}],
      "Axes": [{ "properties": {
        "orientation": { "expr": { "Literal": { "Value": "'VERTICAL'" } } }
      }}],
      "Labels": [{ "properties": {
        "valueFormats": { "expr": { "Literal": { "Value": "'[...]'" } } }
      }}],
      "InputOutput": [{ "properties": {
        "randomId":           { "expr": { "Literal": { "Value": "'abc123'" } } },
        "allPropertiesCache": { "expr": { "Literal": { "Value": "'{...}'" } } }
      }}]
    }
  }
}
```

Each `Value` contains a JSON array or primitive **stringified with escaped quotes** (e.g. `"[{\"enabled\":true,...}]"`), wrapped in outer single quotes.

---

## addressSubset — critical format rule

`addressSubset` filters which data points a config applies to. The key must be the **fully qualified Power BI field reference** (`"TableName.DimensionName"`), and the value must be an **array of member strings**:

```json
// Apply to ALL data
"addressSubset": {}

// Apply only to "Actual" in the Version column of table "Datenquelle"
"addressSubset": { "Datenquelle.Version": ["Actual"] }

// Apply to a specific Sales Rep
"addressSubset": { "Datenquelle.SalesRep": ["Smith"] }
```

> Always ask the user for their exact Power BI table name if not already known. The table name prefix is required — `"Version"` alone will not work.

---

## Workflow

1. **Understand the data** — Ask: what table/column names are used? What scenarios (Actual, Budget…)? What story should the chart tell?
2. **Read the visual.json** — if the user shares it, parse each property value out of its `Value` field.
3. **Recommend chart type & IBCS setup** — use the guidance below.
4. **Output each property separately** — provide one JSON block per property, with instructions on exactly which group it belongs to in `visual.json`.
5. **Explain changes in plain language** — no technical jargon.

---

## IBCS Quick Reference

| Rule | Guidance |
|------|----------|
| **Scenarios** | Use standard IDs: `AC` (Actual), `PY` (Prior Year), `BU` (Budget/Plan), `FC` (Forecast) |
| **AC style** | Dark gray filled, solid border (`#404040`) |
| **PY style** | Light gray filled (`#a6a6a6`), rhomb marker |
| **BU style** | Transparent fill, solid outlined border, circle marker |
| **FC style** | Transparent fill, dashed border |
| **Deviations** | Always show ∆ bars alongside base values |
| **Orientation** | Time series → `HORIZONTAL`. Structural/ranking (people, regions) → `VERTICAL` |
| **Waterfall** | Use for P&L, contribution margin |

---

## Common Financial Chart Patterns

### Pattern 1: Actual vs. Budget with deviation subchart

This is the most common financial pattern. It requires three linked entries in `charts`, two in `chartsLayout`, and one `deviationCalculations` entry. They are linked by a shared `relationId`.

**Step 1 — Generate a unique relation ID** (use a short alphanumeric string, e.g. `"dev1234567890"`). This same ID is used in `deviationCalculations.relationId`, `deviationCalculations.deviationMemberKey`, and the deviation chart entry in `charts`.

**`ChartSpecific` → `chartsLayout`:**
```json
[
  {
    "enabled": true,
    "key": "base_chart",
    "description": "Base Chart",
    "ratio": 7,
    "scaleMode": "DYNAMIC",
    "zoomMode": "OUTLIER",
    "enableSemanticAxis": false,
    "enableSeriesLabels": true,
    "autoShrinkToMatchMaster": false
  },
  {
    "enabled": true,
    "key": "dev1234567890",
    "description": "∆abs",
    "ratio": 3,
    "scaleMode": "DYNAMIC",
    "zoomMode": "OUTLIER",
    "enableSeriesLabels": true,
    "autoShrinkToMatchMaster": true,
    "scaleMaster": "base_chart"
  }
]
```

**`ChartSpecific` → `charts`:**
```json
[
  {
    "enabled": true,
    "addressSubset": {},
    "type": "BAR",
    "subchartKey": "base_chart",
    "displayMode": "BASE",
    "negativeValueIsGood": false,
    "goodColor": "#8cb400",
    "badColor": "#ff0000",
    "neutralColor": "#8f8f8f",
    "relationId": null,
    "waterFallType": "VARIANCE",
    "wedgeHeadSize": "MIDDLE",
    "wedgeHeadFillColor": "#000000",
    "pinHeadShape": "CIRCLE",
    "lineThickness": 2,
    "linePattern": "None",
    "linePatternOffset": 100,
    "lineShowDiffArea": true,
    "lineDiffAreaDisplayMode": "NONE",
    "lineDiffAreaMinuend": {},
    "lineDiffAreaSubtrahend": {},
    "lineLabelVisibility": "GLOBAL_HIGHLIGHTS",
    "target": {}
  },
  {
    "enabled": true,
    "addressSubset": { "Datenquelle.Version": ["Budget"] },
    "type": "WEDGE",
    "subchartKey": "base_chart",
    "displayMode": "BASE",
    "negativeValueIsGood": false,
    "goodColor": "#8cb400",
    "badColor": "#ff0000",
    "neutralColor": "#8f8f8f",
    "relationId": null,
    "waterFallType": "VARIANCE",
    "wedgeHeadSize": "MIDDLE",
    "wedgeHeadFillColor": "#000000",
    "pinHeadShape": "CIRCLE",
    "lineThickness": 2,
    "linePattern": "None",
    "linePatternOffset": 100,
    "lineShowDiffArea": true,
    "lineDiffAreaDisplayMode": "NONE",
    "lineDiffAreaMinuend": {},
    "lineDiffAreaSubtrahend": {},
    "lineLabelVisibility": "GLOBAL_HIGHLIGHTS",
    "target": {}
  },
  {
    "enabled": true,
    "addressSubset": { "Datenquelle.Version": ["dev1234567890"] },
    "description": "∆abs",
    "relationId": "dev1234567890",
    "type": "BAR",
    "subchartKey": "dev1234567890",
    "displayMode": "DEVIATION",
    "negativeValueIsGood": false,
    "goodColor": "#8cb400",
    "badColor": "#ff0000",
    "neutralColor": "#8f8f8f"
  }
]
```

**`Data` → `deviationCalculations`:**
```json
[
  {
    "enabled": true,
    "deviationMemberKey": "dev1234567890",
    "deviationMemberName": "∆abs",
    "deviationType": "ABSOLUTE",
    "targetDimension": "Datenquelle.Version",
    "minuendMember": "Actual",
    "subtrahendMember": "Budget",
    "addressSubset": {},
    "relationId": "dev1234567890"
  }
]
```

**`Data` → `scenarios`:**
```json
[
  {
    "enabled": true,
    "scenarioId": "AC",
    "addressSubset": { "Datenquelle.Version": ["Actual"] },
    "description": "",
    "relationId": null
  },
  {
    "enabled": true,
    "scenarioId": "BU",
    "addressSubset": { "Datenquelle.Version": ["Budget"] },
    "description": "",
    "relationId": null
  }
]
```

### Pattern 2: Highlights

Use the native `highlights` property — **never use XFL for this**. It draws an ARROW or LINE connector between two specific, named data points.

Key rules:
- Address keys use the **fully qualified** `"TableName.DimensionName"` format.
- Address values are **strings** (not arrays) in highlights — unlike `addressSubset`.
- Only **one highlight entry** is supported (`maxItems: 1`).
- `highlights` targets specific named members — it cannot automatically find "the biggest" at runtime. If the user asks for that, ask which member they want to highlight.

**`Emphasis` → `highlights`:**
```json
[
  {
    "enabled": true,
    "description": "my highlight",
    "type": "ARROW",
    "sourceAddress": { "Datenquelle.SalesRep": "Smith", "Datenquelle.Version": "Actual" },
    "targetAddress": { "Datenquelle.SalesRep": "Jones", "Datenquelle.Version": "Actual" },
    "showAbsoluteDeviation": true,
    "showRelativeDeviation": true,
    "negativeValueIsGood": false,
    "goodColor": "#8cb400",
    "badColor": "#ff0000",
    "neutralColor": "#8f8f8f",
    "relationId": null
  }
]
```

### Pattern 3: Value formatting in thousands

Use `"abbreviation": "thousand"` — do **not** use `scalingFactor`. Add a second format entry for deviation percentages if needed.

**`Labels` → `valueFormats`:**
```json
[
  {
    "enabled": true,
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
  }
]
```

### Pattern 4: Orientation

**`Axes` → `orientation`:**
- Structural/ranking dimensions (Sales Reps, Products, Regions): `"VERTICAL"` → renders as horizontal bar chart
- Time series (months, quarters, years): `"HORIZONTAL"` → renders as column chart

### Pattern 5: P&L Waterfall
- `charts[].type: "WATERFALL"`, `waterFallType: "MANUAL"` for user-controlled directions
- Use `categorySemantics` to mark subtotals and result rows

---

## How to apply the config to visual.json

Always give the user these exact steps.

### Step 1 — Open the report folder
A `.pbip` is a folder. Open it in VS Code (recommended) or Windows Explorer:
```
MyReport.pbip
MyReport.Report/
  definition/
    pages/
      ReportSectionXXXXXX/
        visuals/
          XXXXXXXXXXXXXXXX/
            visual.json    ← edit this file
```

### Step 2 — Find the right visual.json
If there are multiple visuals, open each `visual.json` and look for `"visualType": "graphomatechartrix"`.

### Step 3 — Find the property to update
Each property lives inside its category group. For example, to update `chartsLayout`:
```json
"ChartSpecific": [{ "properties": {
  "chartsLayout": { "expr": { "Literal": { "Value": "' ← YOUR JSON HERE '" } } }
}}]
```

### Step 4 — Replace the value
Each `Value` field contains a stringified JSON array or string wrapped in single quotes: `'[{...}]'`.

Replace **only the content between the outer single quotes** with the new JSON Claude provides. The JSON must be valid — use VS Code's built-in JSON validation to check.

**Property → Group mapping:**
| Property | Group in `objects` |
|---|---|
| `chartsLayout`, `charts`, `categorySemantics` | `ChartSpecific` |
| `scenarios`, `deviationCalculations`, `scenarioStyles` | `Data` |
| `highlights`, `separators`, `backgroundColor` | `Emphasis` |
| `orientation`, `axisThicknessPx` | `Axes` |
| `valueFormats`, `fontSize`, `titleText` | `Labels` |
| `randomId`, `serverUrl`, `templateName` | `InputOutput` |

### Step 5 — Reload in Power BI
Save the file, switch to Power BI Desktop, and click **Reload** when prompted.

> **Tip:** Use VS Code with the Prettier extension to auto-format. If Power BI shows an error after reload, the JSON likely has a syntax issue — check for unescaped quotes.

---

## Communication tips for finance users

- Avoid technical jargon like "addressSubset" — say "which data this applies to" instead.
- Use IBCS scenario names in plain language: Actual, Prior Year, Budget, Forecast.
- When in doubt about chart type, ask: "Is this comparing over time, or comparing people/regions/products?"
- Always confirm the exact Power BI **table name** — it is required as a prefix in all dimension references.
