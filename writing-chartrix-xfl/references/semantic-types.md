# XFL SemanticType Catalogue

Every entry in `this.viewModel` carries a `semanticType` string that identifies
what kind of visual element it represents. Use this to target elements precisely
in your XFL scripts via `switch (item.semanticType)` or direct filtering.

---

## Data Elements

| SemanticType | Description | Key properties |
|---|---|---|
| `DATA_BAR` | Standard bar or column — the primary data element in BAR/COLUMN charts | `bar.styles`, `bar.color`, `rowIndex`, `columnIndex` |
| `WATERFALL_BAR` | Waterfall segment (requires `charts[].type: "WATERFALL"` on both chart entries) | `bar.styles`, `bar.color`, `rowIndex`, `columnIndex` |
| `WATERFALL_CONNECTOR` | Connecting line between waterfall bars | `line.styles` |
| `PIN_BAR` | The stem of a PIN chart element | `bar.styles`, `bar.color` |
| `PIN_HEAD` | The marker (head) of a PIN chart element — supports `color` only, not `styles` | `color` — use `item.address` for series identification, not `rowIndex` |
| `LINE_CHART_LINE` | The connecting line in a LINE chart | `line.styles` |
| `LINE_CHART_AREA` | The filled area under a LINE chart line | `area.styles` |

---

## Labels

| SemanticType | Description | Key properties |
|---|---|---|
| `VALUE_LABEL_BAR` | Value label attached to a `DATA_BAR` | `text`, `label.styles` |
| `VALUE_LABEL_WATERFALL` | Value label on a waterfall bar | `text`, `label.styles` |
| `VALUE_LABEL_PIN` | Value label on a PIN element | `text`, `label.styles` |
| `CATEGORY_LABEL` | Category axis label (X axis for columns, Y axis for bars) | `text` — overwrite to rename or replace with emoji |
| `AXIS_LABEL` | Value axis (scale) tick label | `text`, `label.styles` |
| `CHART_TITLE` | Top-level chart title | `text`, `label.styles` |
| `SUB_CHART_TITLE` | Title of an individual sub-chart in a multi-chart layout | `text`, `label.styles` |

---

## Highlights

Highlights are always defined in `highlights[]` config first — XFL can then
reposition or restyle the elements they generate. You cannot create highlight
elements purely via XFL without a corresponding config entry.

| SemanticType | Description | Key properties |
|---|---|---|
| `HIGHLIGHT_DELTA_LINE` | The bracket/span line of a highlight | `line.styles` |
| `HIGHLIGHT_LABEL` | The label showing the delta value of a highlight | `text`, `label.styles` |
| `HIGHLIGHT_CONNECTOR` | Connector line from highlight to data point | `line.styles` |

---

## Separators

Separators are always defined in `separators[]` config first.

| SemanticType | Description | Key properties |
|---|---|---|
| `SEPARATOR` | Vertical (column) or horizontal (bar) divider between category groups | `line.styles` — use XFL to conditionally show/hide |

---

## Background

| SemanticType | Description | Key properties |
|---|---|---|
| `BACKGROUND_RECT` | Full-chart background rectangle — only addressable via XFL, no config counterpart | `rect.styles` |

---

## Usage pattern

```javascript
this.viewModel.forEach(item => {
  switch (item.semanticType) {
    case 'DATA_BAR':
      // target all bars
      break;
    case 'VALUE_LABEL_BAR':
      // target all bar value labels
      break;
    case 'CATEGORY_LABEL':
      // rename or restyle category axis labels
      break;
  }
});
```

For the mapping between config properties and SemanticTypes, see `elements.md`.
For the full `this.viewModel` item shape and available API methods, see `api.md`.
