# chartrix Visual Elements

Every chartrix chart consists of a set of **visual elements** that are controlled
statically at configuration time (config properties in `visual.json`) and can be
overridden dynamically at runtime via **XFL**.

---

## Bridge table: Config → Element → XFL

| Element (visible) | Controlling config property | XFL SemanticType |
|---|---|---|
| Bars / columns | `scenarios[].color`, `charts[].type` | `DATA_BAR` |
| Waterfall bar | `charts[].type: "WATERFALL"` | `WATERFALL_BAR` |
| Pin stem | `charts[].type: "PIN"` | `PIN_BAR` |
| Pin head (marker) | `charts[].type: "PIN"` | `PIN_HEAD` |
| Value label | `valueLabel*` properties | `VALUE_LABEL_BAR` and others |
| Category axis | `categoryAxis*` properties | `CATEGORY_LABEL` |
| Value axis (scale) | `valueAxis*` properties | `AXIS_LABEL` |
| Highlight (span) | `highlights[]` | `HIGHLIGHT_DELTA_LINE`, `HIGHLIGHT_LABEL`, `HIGHLIGHT_CONNECTOR` |
| Separator | `separators[]` | `SEPARATOR` |
| Line chart | `charts[].type: "LINE"` | `LINE_CHART_LINE`, `LINE_CHART_AREA` |
| Chart title | `title` | `CHART_TITLE` |
| Subchart title | `chartsLayout[].title` | `SUB_CHART_TITLE` |
| Background | – (XFL only) | `BACKGROUND_RECT` |

---

## Elements by category

### Data elements

**Bars (`DATA_BAR`)** — main element of all bar/column charts.
- Color: via `scenarios[].color` or XFL `bar.styles = { name: "xfl", styles: "fill: ..." }`
- Type: via `charts[].type` (`BAR`, `COLUMN`, `WATERFALL`, `PIN`, ...)
- Scenarios control the base coloring; XFL can override it per data point.

**Waterfall (`WATERFALL_BAR`)** — stacked deviation bars.
- Requires `charts[].type: "WATERFALL"` on **both** chart entries in the layout.
- `WATERFALL_CONNECTOR`: connecting lines between the bars (addressable via XFL).

**Pin chart (`PIN_BAR` + `PIN_HEAD`)** — stem + marker.
- Via XFL, `PIN_HEAD` supports only `color`, not `styles`.
- Identify the series of a `PIN_HEAD` via `item.address`, not `rowIndex`.

### Labels

**Value labels** — text on or next to data elements.
- Config: `valueLabel*` properties control format, position, visibility.
- XFL: `VALUE_LABEL_BAR`, `VALUE_LABEL_WATERFALL`, `VALUE_LABEL_PIN` etc.
- `viewItem.text` contains the formatted value — can be overridden via XFL.

**Category labels** — labels of the category axis.
- Config: `categoryAxis*` properties.
- XFL: `CATEGORY_LABEL` — text can be changed via `viewItem.text` (e.g. emoji replacement).

### Highlights

Highlights are a **native chartrix feature** — always use `highlights[]` in the
config, **never draw them manually via XFL**.
- The config entries create the `HIGHLIGHT_*` ViewItems.
- XFL can then reposition them dynamically (e.g. point to max/min).
- Without a `highlights[]` entry, no `HIGHLIGHT_*` ViewItems exist in XFL.

### Separators

Dividers between category groups — also **native config**, no manual SVG.
- Config: `separators[]` — creates `SEPARATOR` elements.
- XFL: `SEPARATOR` SemanticType is addressable for styling overrides.

---

## When to use config, when XFL?

| Requirement | Use |
|---|---|
| Color all bars of a scenario the same | Config (`scenarios[].color`) |
| Color bars above a threshold green | XFL |
| Highlight between two fixed points | Config (`highlights[]`) |
| Set a highlight dynamically to max/min | Config (placeholder) + XFL |
| Rename a category label (fixed) | Config (`categoryAxis`) |
| Replace a category label with an emoji (data-driven) | XFL |
| Insert a separator | Config (`separators[]`) |
| Show/hide a separator conditionally | XFL (`SEPARATOR`) |
