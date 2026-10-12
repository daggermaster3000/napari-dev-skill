# Streamlit explorer UI: layout, controls, wording

How the explorer apps are laid out and how they talk. Colour has its own file
(`streamlit-colour.md`); data, statistics and tests are in
`streamlit-explorers.md`. Follow these when adding a tab, a control or a new app,
so the apps keep reading as one tool.

## Contents
1. Page skeleton
2. Sidebar: what the whole app is looking at
3. Tabs: one question each
4. Inside a tab: controls left, result right
5. Choosing the control
6. Empty, missing and failed states
7. Wording
8. Figures: size, margins, legends, hover
9. Tables and downloads
10. Widget keys and reruns
11. Checklist

## 1. Page skeleton

Every app has the same order, top to bottom:

```python
st.set_page_config(page_title="Region explorer", layout="wide")
st.title("Region explorer")

with st.sidebar:                      # 1. where the data comes from
    st.header("Workbook")
    ...
if source is None:                    # 2. nothing loaded: say what to load, stop
    st.info("Load the workbook an Analysis run wrote — …")
    st.stop()

with st.sidebar:                      # 3. which rows
    st.header("Rows")
    ...
st.caption(f"{len(rows)} region row(s) from {n} sample(s), …")   # 4. what is left
if rows.empty:
    st.warning("No rows left after filtering.")
    st.stop()

compare_tab, relate_tab, table_tab = st.tabs(["Compare", "Relate", "Table"])  # 5.
```

- `layout="wide"` always: the plot is the point and the controls sit beside it.
- The one-line **caption of what survived the filters** sits directly under the
  title, above the tabs, so it is true for every tab and always in view.
- Guard with `st.stop()` early. Everything below a guard can assume its data.

## 2. Sidebar: what the whole app is looking at

The sidebar holds only things that change **every tab at once**. Sections, in
order, each under `st.header`:

1. **Workbook** — a file uploader, then "…or a path on this machine" (prefilled
   from `argv` when the viewer launched the app). Optional companion files
   (cell outlines) below, found automatically beside the workbook when a path
   was typed.
2. **Colours** — one collapsed expander (see `streamlit-colour.md`).
3. **Rows** — multiselects for Regions, Genotypes, Samples, all selected by
   default. Filtering removes; nothing is selected "in".
4. **Cells** — only when the workbook has cells: a size measure and a
   "keep from … to …" range slider, with a caption of how many are kept.

A control that affects one tab does not go in the sidebar, however global it
feels while writing that tab.

## 3. Tabs: one question each

- A tab answers one question ("are the groups different?", "what do the cells
  look like?"), named in one or two plain words: `Compare`, `Relate`, `Cells`,
  `Atlas`, `Explain`, `Plots`, `Table`.
- **`Table` is always last**: the filtered rows as they are, plus a CSV download.
  It is the way out of the app and the check on everything the plots claim.
- A tab that needs something the workbook lacks still appears, and says what it
  needs in one `st.info` (see §6). Tabs do not come and go with the data.
- A simple app (`simple_explorer`) is a subset of the full one's tabs with the
  same names and the same shared functions — not a second design.

## 4. Inside a tab: controls left, result right

```python
left, right = st.columns([1, 3])
with left:
    st.subheader("Compare")          # names the group of controls
    ...
with right:
    chart(figure, width="stretch")
    st.caption(...)                  # what was dropped, what the dots are
    st.dataframe(...)                # the numbers behind the picture
    st.download_button(...)
```

- Always `[1, 3]`. Controls read top to bottom in the order the decision is
  made: **what** (measure, region, variable) → **how it is grouped** (group by,
  normalise to) → **how it is drawn** (shape, dots, bars) → **how it is tested**.
- Several groups of controls in one column get a `st.subheader` each ("Mean
  shape", "Heatmap", "Cells"), not dividers.
- Rarely-touched settings go in a collapsed `st.expander` at the end of their
  group ("Fine-tune columns", "Every sample"). The title carries the count when
  there is one: `f"{len(used)} features used"`.
- Under the plot, in this order: notes about the plot, the tables it summarises,
  downloads. Side-by-side secondary results use `st.columns(2)` or `[3, 2]`.

## 5. Choosing the control

| The choice is | Use |
|---|---|
| 2–3 fixed options, all worth seeing | `st.radio(..., horizontal=True)` |
| one of many, or one of the data's columns | `st.selectbox` |
| several of the data's values | `st.multiselect`, all selected by default |
| on / off | `st.checkbox` with the label saying what *on* does ("Show the dots") |
| a number with a sensible range | `st.slider` (min, max, default, step all given) |
| a number with no range (a seed) | `st.number_input(step=1)` |
| a group of columns | one checkbox per group with its size: `"Shape (12)"` |

- **Defaults must give a useful plot with no clicks.** Pick the most likely
  variable (`default_variable`), put the most likely denominator first, start
  filters wide open.
- Clamp ranges to the data: `st.slider("Neighbours", 2, max(3, min(100, n)),
  min(15, n))`. A slider that can be dragged to an impossible value is a bug
  report waiting.
- A control that does nothing right now is **disabled, not hidden**
  (`disabled=not warp`), so the layout does not jump; when a whole option is
  impossible, say so in a caption where the control would be ("Outlines are
  drawn in 2D only.").
- `help=` holds the *why* and the trade-off, in a sentence or two. The label
  stays short.

## 6. Empty, missing and failed states

One vocabulary, used the same way everywhere:

| Situation | Call | Then |
|---|---|---|
| nothing loaded yet; a tab needs a sheet or package the user does not have | `st.info` — say what to load or install | `st.stop()` at page level; skip the tab body otherwise |
| the current choices leave nothing to draw | `st.warning` — say what to change | skip the plot |
| something broke that the user cannot fix by choosing differently | `st.error` — say what, with the exception text | `st.stop()` or skip |
| a file path that is not there | `st.sidebar.error`, next to the box it was typed in | |
| a partial result (some rows dropped, some outlines missing) | `st.caption` under the plot | draw the rest |
| an action finished | `st.success`, or `st.warning` if any item failed | show the per-item table |

- Never draw an empty chart. Never let a traceback reach the page for a
  condition the data can produce (too few rows, a constant column, one group).
- A fallback is announced, not silent: "Embedding skipped (…); showing a scatter
  plot instead."
- Messages name the thing: `f"UMAP needs at least {MIN_EMBED_ROWS} rows (samples
  × regions)."`, not "Not enough data."

## 7. Wording

- **Labels are nouns or short noun phrases in sentence case**: "Group by",
  "Normalize to", "Smoothing radius (µm)", "Cells to use". Units in brackets in
  the label.
- **The same thing has the same label in every tab and app**: "Colour by",
  "Shape by", "Group by", "Variable", "Region". Grep before inventing one.
- Options are words a biologist would say, not parameter names: "Significant
  only / All pairs / Off", "Stars / p value", "Hidden / Dots / Outlines".
- Captions are full sentences that state a fact about what is on screen:
  how many, what was left out, what a dot is, what the test ran on. No
  instructions, no enthusiasm.
- Counts get a thousands separator and say "of" the total: `f"{kept:,} of
  {total:,} cells kept."` Plurals as `row(s)`.
- Placeholders for "no value" are bracketed and consistent: `(none)`,
  `(unknown)`, `(nothing)`, `(all cells)`. They sort last.
- Sheet and file names are quoted as the user will see them: “Region features”,
  `cell_outlines.npz`.
- Say what a statistic means where it is shown: "Stars: * < 0.05, ** < 0.01,
  *** < 0.001; ns otherwise." — in the `help` of the control that turns it on.

## 8. Figures: size, margins, legends, hover

- Every figure goes through `chart(figure, width="stretch")`, never a raw
  `st.plotly_chart` (the wrapper applies the house style).
- **Titles say what is plotted and of what**, on the figure itself
  (`title=...`), so a screenshot stands alone.
- Heights are fixed numbers, not left to Plotly: ~520 for a comparison, ~620 for
  an embedding or scatter, 220 for a small supporting chart, `max(320, 30 * n +
  120)` when the row count drives it, `height * rows` for a grid of panels.
- Margins are tight and explicit: `{"l": 10, "r": 10, "t": 50–60, "b": 10}` with
  `automargin` on the axes. Leave `b: 40` when a horizontal legend sits below.
- Where plots must line up or keep true proportions (shape glyphs in data
  space), fix the plot area instead: constant `PLOT_W, PLOT_H` and a `MARGIN`
  dict, `height = MARGIN["t"] + PLOT_H + MARGIN["b"]`.
- Anything spatial has **equal axes** (`scaleanchor`, `scaleratio=1`) and a
  scale bar; a transparent `plot_bgcolor` so it sits on either theme.
- Legends: a title naming the column; `groupclick="toggleitem"` when traces are
  grouped; horizontal under the plot for panel grids (`orientation="h", y=0,
  yanchor="top", itemsizing="constant"`). One legend entry per group, not per
  trace — set `legendgroup` and `showlegend=False` on the repeats. A comparison
  plot whose x axis already names the groups has no legend.
- Marker sizes: 9 for sample dots, 7–8 for secondary points, 5 for dense
  per-cell summaries, 3 for clouds of thousands. Use `render_mode="webgl"` for
  per-cell scatters.
- Hover names the row first (`%{text}`), then the value to four significant
  figures (`:.4g`), with the group in `<extra>`. Traces that only carry a shape
  (the violin body under its dots) get `hoverinfo="skip"`.
- Jitter is seeded (`np.random.default_rng(0)`): the same data draws the same
  picture on every rerun.
- Cap what is drawn and say so: at most 8 significance bars, `MAX_CELL_GLYPHS`
  outlines, 40 colour pickers — each with a caption giving "N of M".

## 9. Tables and downloads

- `st.dataframe(frame, hide_index=True, width="stretch")`. Format numbers with
  a Styler: `{:.4g}` for measurements, `{:.3g}` for p values, `{:.1f}` for
  percentages.
- A table has a one-line caption above it saying what a row is and how to read
  the key column.
- Every plot's underlying values are downloadable from under it: "These values
  (CSV)". Labels say what and in which format; file names carry the variable or
  region (`f"{region}_template.csv"`).

## 10. Widget keys and reruns

- Give a `key` to every widget whose label could repeat across tabs — in
  practice all of them. Prefix with the tab: `plot-group-by`, `cell-dims`,
  `atlas-heat`, `shap-target`. Tests address widgets by these keys.
- Keys for generated widgets include what generated them:
  `f"color-{kind}-{value}"`, `f"g-{name}"`.
- Expensive steps are `@st.cache_data(show_spinner="Registering outlines…")`
  functions of plain arguments; the spinner text says what is happening, with an
  ellipsis.
- A change should cost one rerun. Only call `st.rerun()` after writing state
  that widgets above have already read, and only when something changed.
- Anything costly and optional sits behind an explicit opt-in ("Score them"
  checkbox), not on by default.

## 11. Checklist

- Sidebar holds only whole-app controls; the survivors caption is under the title.
- New tab: one question, `[1, 3]` columns, subheader on the left, `Table` still last.
- Opens to a useful plot with no clicks; sliders clamped to the data.
- Every empty / missing / failed path shows the right one of info, warning,
  error, caption — no empty chart, no traceback.
- Labels reuse the existing words; captions state facts with counts.
- Figure through `chart()`, fixed height, explicit margins, titled legend.
- Values behind the plot downloadable; every widget keyed with its tab prefix.
- Colours through `paint()` (see `streamlit-colour.md`).
