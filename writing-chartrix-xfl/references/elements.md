# chartrix Visual Elements

Jedes chartrix-Chart besteht aus einer Menge von **visuellen Elementen**, die
zur Konfigurationszeit (Config-Properties in `visual.json`) statisch gesteuert
und zur Laufzeit via **XFL** dynamisch überschrieben werden können.

Diese Datei beschreibt Elemente aus der **Config-Perspektive**.
Für XFL-seitige Laufzeit-Adressierung → `chartrix-xfl/references/semantic-types.md`

---

## Brückentabelle: Config → Element → XFL

| Element (sichtbar) | Steuernde Config-Property | XFL SemanticType |
|---|---|---|
| Balken / Säulen | `scenarios[].color`, `charts[].type` | `DATA_BAR` |
| Wasserfall-Balken | `charts[].type: "WATERFALL"` | `WATERFALL_BAR` |
| Pin-Stab | `charts[].type: "PIN"` | `PIN_BAR` |
| Pin-Kopf (Marker) | `charts[].type: "PIN"` | `PIN_HEAD` |
| Wertebeschriftung | `valueLabel*` Properties | `VALUE_LABEL_BAR` u.a. |
| Kategorieachse | `categoryAxis*` Properties | `CATEGORY_LABEL` |
| Werteachse (Skala) | `valueAxis*` Properties | `AXIS_LABEL` |
| Highlight (Spanne) | `highlights[]` | `HIGHLIGHT_DELTA_LINE`, `HIGHLIGHT_LABEL`, `HIGHLIGHT_CONNECTOR` |
| Separator | `separators[]` | `SEPARATOR` |
| Linienchart | `charts[].type: "LINE"` | `LINE_CHART_LINE`, `LINE_CHART_AREA` |
| Chart-Titel | `title` | `CHART_TITLE` |
| Sub-Chart-Titel | `chartsLayout[].title` | `SUB_CHART_TITLE` |
| Hintergrund | – (XFL only) | `BACKGROUND_RECT` |

---

## Elemente nach Kategorie

### Datendarstellung

**Balken (`DATA_BAR`)** — Hauptelement aller Bar/Column Charts.
- Farbe: über `scenarios[].color` oder XFL `bar.styles = css\`fill: ...\``
- Typ: über `charts[].type` (`BAR`, `COLUMN`, `WATERFALL`, `PIN`, ...)
- Szenarien steuern die Grundfärbung; XFL kann pro Datenpunkt überschreiben.

**Wasserfall (`WATERFALL_BAR`)** — Gestapelte Abweichungsbalken.
- Benötigt `charts[].type: "WATERFALL"` auf **beiden** Chart-Einträgen im Layout.
- `WATERFALL_CONNECTOR`: Verbindungslinien zwischen den Balken (XFL-adressierbar).

**Pin Chart (`PIN_BAR` + `PIN_HEAD`)** — Stab + Marker.
- `PIN_HEAD` unterstützt via XFL nur `color`, keine `styles`.
- Serienerkennung am `PIN_HEAD` über `item.address`, nicht `rowIndex`.

### Beschriftungen

**Wertebeschriftungen** — Text auf/neben Datenelementen.
- Config: `valueLabel*`-Properties steuern Format, Position, Sichtbarkeit.
- XFL: `VALUE_LABEL_BAR`, `VALUE_LABEL_WATERFALL`, `VALUE_LABEL_PIN` etc.
- `viewItem.text` enthält den formatierten Wert — via XFL überschreibbar.

**Kategorielabels** — Beschriftung der Kategorieachse.
- Config: `categoryAxis*`-Properties.
- XFL: `CATEGORY_LABEL` — Text via `viewItem.text` änderbar (z.B. Emoji-Ersatz).

### Highlights

Highlights sind eine **native chartrix-Funktion** — immer `highlights[]` in der
Config nutzen, **niemals manuell per XFL zeichnen**.
- Die Config-Einträge erzeugen die `HIGHLIGHT_*` ViewItems.
- XFL kann diese dann dynamisch repositionieren (z.B. auf Max/Min zeigen).
- Ohne `highlights[]`-Eintrag existieren keine `HIGHLIGHT_*` ViewItems im XFL.

### Separatoren

Trenner zwischen Kategoriegruppen — ebenfalls **native Config**, kein manuelles SVG.
- Config: `separators[]` — erzeugt `SEPARATOR`-Elemente.
- XFL: `SEPARATOR` SemanticType adressierbar für Styling-Overrides.

---

## Wann Config, wann XFL?

| Anforderung | Mittel |
|---|---|
| Alle Balken einer Szene gleich färben | Config (`scenarios[].color`) |
| Balken oberhalb Schwellwert grün färben | XFL |
| Highlight zwischen zwei fixen Punkten | Config (`highlights[]`) |
| Highlight dynamisch auf Max/Min setzen | Config (Platzhalter) + XFL |
| Kategorielabel umbenennen (fix) | Config (`categoryAxis`) |
| Kategorielabel durch Emoji ersetzen (datengetrieben) | XFL |
| Separator einfügen | Config (`separators[]`) |
| Separator bedingt ein-/ausblenden | XFL (`SEPARATOR`) |
