# graphomate chartrix – Full Property Reference

This file documents all configurable properties of graphomate chartrix. Use it when you need details beyond what is in SKILL.md.

---

## Category: Chart Specific

#### Chart Types — `charts`
Array of chart type assignments. Each entry targets a subset of data.
- **type**: `BAR` | `LINE` | `PIN` | `WEDGE` | `WATERFALL` | `STACKED_BAR` | `OFFSET_BAR`
- **displayMode**: `BASE` (main value bars) | `DEVIATION` (variance bars)
- **subchartKey**: which layout panel to render in (matches `chartsLayout[].key`)
- **negativeValueIsGood** (boolean): invert deviation color (useful for costs)
- **lineThickness** (number): for LINE type
- **linePattern**: `None` | `Dots` | `Dashes`
- **waterFallType**: `VARIANCE` | `MANUAL` | `AUTOMATIC`
- **pinHeadShape**: `CIRCLE` | `RHOMBUS` | `WEDGE` | `RECT`

#### Chart Layout — `chartsLayout`
Array of subchart panels. Each panel holds one or more chart series.
- **key** (string): unique ID, referenced by `charts[].subchartKey`
- **ratio** (number): relative size of this panel (e.g. 3 for main, 1 for deviation)
- **scaleMode**: `DYNAMIC` | `SHARED` | `FIXED`
- **scaleFixedPositive** / **scaleFixedNegative**: for FIXED mode
- **synchronizeScale** (boolean): link scale across visuals
- **synchronizeScaleGroup** (string): group ID for shared scaling (Comparison Group)
- **enableSemanticAxis** (boolean): show IBCS scenario markers on axis
- **zoomMode**: `NONE` | `CUT_AXIS` | `OUTLIER`
- **enableSeriesLabels** (boolean): show series labels

#### Categories — `categorySemantics`
Assigns IBCS waterfall semantics to specific categories.
- **waterfallElementType**: mark a row as a subtotal, result, etc.
- **negativeDeviationIsGood** (boolean): invert colors for cost rows

---

## Category: Data

#### Scenario Assignment — `scenarios`
Assigns an IBCS scenario ID to a subset of data.
- **scenarioId** (string): `AC` | `PY` | `BU` | `FC` — or any custom ID
- **addressSubset** (object): filter to specific measures/dimensions
- **subChartKeys** (array): limit assignment to specific subcharts

#### Scenarios — `scenarioStyles`
Visual style definitions for each scenario.
- **id** (string): must match a `scenarioId` used in `scenarios`
- **fillColor** (string): hex color for filled bars
- **borderColor** (string): hex color for border
- **borderThickness** (string): e.g. `"1px"`
- **borderDasharray** (string): e.g. `"4 2"` for dashed border (FC style)
- **markerShape**: `RECT` | `RHOMB` | `CIRCLE`
- **barWidth** (number): 0–1, fraction of available space
- **pinWidth** (number): 0–1

**IBCS defaults:**
| ID | fillColor | markerShape | Notes |
|----|-----------|-------------|-------|
| AC | `#404040` | RECT        | Solid dark gray |
| PY (PP) | `#a6a6a6` | RHOMB | Light gray |
| BU | `#404040` | CIRCLE | Hollow in IBCS → set fillColor transparent |
| FC | `#404040` | RHOMB | Hollow dashed → borderDasharray `"4 2"` |

#### Deviation Calculations — `deviationCalculations`
Auto-calculates variance members. No DAX needed.
- **deviationType**: `ABSOLUTE` | `PERCENT`
- **deviationMemberKey** (string): key for the new member (e.g. `"delta_PY"`)
- **deviationMemberName** (string): display label (e.g. `"∆PY"`)
- **targetDimension** (string): dimension to add the new member to (usually `"Measures"`)
- **minuendMember** (string): the "actual" side (e.g. `"Actual"`)
- **subtrahendMember** (string): the "comparison" side (e.g. `"PriorYear"`)

#### Other Calculations
- **customCalculations**: custom formula members
- **nRestCalculations**: top-N + rest grouping
- **dimensionSortOrders**: custom sort order for members
- **valueSortOrder**: sort by value ascending/descending
- **aggregationType**: `SUM` | `MIN` | `MAX` | `CUMULATIVE_MEAN` | `GEOMETRIC_MEAN` | `COUNT` | `NONE`
- **maxNumberOfDataPoints** (number): default 200, increase for large datasets
- **valueFilterConfigs**: filter data by value threshold

---

## Category: Axes

- **orientation**: `HORIZONTAL` (bars go left/right = column chart) | `VERTICAL` (bars go up/down = bar chart)
- **axisThicknessPx** (number): default `2`
- **minimumCategorySizePx** (number): default `20`
- **maximumCategorySizePx** (number): default `150`
- **initialCategoryExpandLevel**: initial expand level for hierarchy

---

## Category: Labels

#### Value Format — `valueFormats`
- **format**: `"number"` | `"percent"` | `"currency"`
- **abbreviation**: `"auto"` | `"K"` | `"M"` | `"B"` | `"none"`
- **fractionalDigits** (number): decimal places
- **scalingFactor** (number): multiply all values (e.g. `0.001` to show thousands)
- **prefix** / **suffix** (string): unit labels
- **localeId** (string): e.g. `"en-US"`, `"de-DE"`
- **nullOutput** (string): text shown for null values
- **negativeSign**: `"minus"` | `"parentheses"`

#### Font
- **fontSize** (string): e.g. `"12px"` — default
- **fontFamily** (string): e.g. `"Arial"` — default
- **fontColor** (string): hex color — default `#000000`

#### Label Collision
- **hideCollidingLabels** (boolean): default `true`
- **collidingCategoryLabelStrategy**: `SCROLL` | `HIDE` | `TRIM_AND_SCROLL` | `TRIM_AND_HIDE`

#### Title
- **titleText** (string): chart title
- **enableTitle** (boolean): show/hide title

---

## Category: Emphasis

- **backgroundColor** (string): chart background color, default `#FFFFFF`
- **highlights**: conditional highlighting rules (XFL-based)
- **separators**: visual separator lines between data groups

---

## Category: Input / Output

- **serverUrl**: graphomate Server URL (for templates and comments)
- **templateName**: name of a saved template to load
- **liveTemplate** (boolean): auto-reload template on server update
- **dataPointCommentRepresentation**: `None` | `Indicator` | `Numbered`

---

## addressSubset – Filtering Reference

`addressSubset` appears in many properties to restrict which data points are affected. It is an object where keys are dimension names and values are **arrays of member key strings** — always an array, even for a single member.

```json
// Apply to ALL data
"addressSubset": {}

// Apply only to "Actual" measure
"addressSubset": { "Measures": ["Actual"] }

// Apply to a specific category
"addressSubset": { "Month": ["Jan"] }

// Apply to a combination
"addressSubset": { "Measures": ["Actual"], "Region": ["North"] }
```

Use `addressSubset` to assign different scenarios, formats, or styles to different parts of the same dataset.
