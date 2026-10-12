# Streamlit explorer colour

Which colours the explorer apps use, what each one means, and the rules that
keep a colour meaning the same thing in every plot. The mechanics of the black
outline wrapper and of the user's colour pickers are in
`streamlit-explorers.md`; layout and wording are in `streamlit-ui.md`.

## Contents
1. Principles
2. The palette
3. Who gets which colour
4. Continuous scales
5. Opacity and edges
6. The user's colours
7. Light and dark theme
8. Checklist

## 1. Principles

- **Colour encodes one thing per plot** — the column named in "Colour by" — and
  that column is named in the legend title. Shape carries a second column
  ("Shape by"); never a second colour scheme.
- **A value keeps its colour everywhere.** `wt` is the same blue in the box
  plot, the scatter, the UMAP, the atlas and the exported report figures. This
  is why no plot builds its own palette.
- **Define colours once.** The constants live in one module
  (`analysis_plots.py`) that both the viewer's matplotlib report and the
  Streamlit apps import; the apps re-export rather than redefine
  (`INK = ap.INK`).
- **Grey means "not a group"**: unknown values, templates, significance bars,
  backgrounds under coloured marks. Never give a real group a grey.
- **Colour is never the only carrier.** Groups are also separated by position
  (x axis), marker symbol, or a label; significance is stars or a p value, not a
  colour.

## 2. The palette

```python
# Categorical, in assignment order.
GENOTYPE_COLORS = ("#2a78d6", "#eb6834", "#1baf7a", "#4a3aa7")   # blue, orange, green, violet
EXTRA_COLORS    = ("#e87ba4", "#008300", "#e34948", "#eda100")   # pink, dark green, red, amber

# Neutrals.
INK        = "#1f1f1e"   # titles, annotations over the plot
INK_MUTED  = "#6b6a65"   # axis titles, tick labels, legend titles
GRID       = "#e4e3de"   # grid lines
SURFACE    = "#ffffff"   # exported figure background
NEUTRAL    = "#8a8983"   # unknown group, significance brackets, template outline
OUTLINE    = "#000000"   # the edge of every mark

# Symbols, in the same assignment order as the colours.
SYMBOLS = ("circle", "square", "diamond", "cross", "x", "triangle-up", "triangle-down", "star")
```

Eight categorical colours, then they repeat. Past eight groups, colour stops
distinguishing — switch what is coloured (see samples, below) or let symbol
carry it.

## 3. Who gets which colour

Assignment is by **position in a fixed order**, never by hashing or by first
appearance in the filtered data — filtering a group out must not recolour the
rest.

| Coloured by | Order | Colour |
|---|---|---|
| Genotype | reference first (`wt`, `ctl`, `sib`), the rest alphabetically, `(unknown)` last | palette by position; `(unknown)` is `NEUTRAL` |
| A condition (Treatment, …) | same rule; `(none)` last | palette by position |
| Genotype × condition (`wt · DMSO`) | condition by condition, genotypes side by side | **the genotype's colour** in every condition — position on the axis separates the conditions |
| Region | the order regions were drawn in | palette by position |
| Cluster (`C1`…) | cluster number | palette by number, so `C1` is always the first blue |
| Sample, up to 8 | grouped by genotype | palette by position |
| Sample, more than 8 | grouped by genotype | **shades of the genotype's colour**: mixed from 45 % towards black to 45 % towards white across the group |

- The reference group is first, so it is always the first blue and always on
  the left.
- Shades-of-the-genotype for samples keeps the genotype readable when the plot
  is coloured by sample.
- One function returns `(colours, order)` for a column (`category_colors(frame,
  by)`); pass both to the plot (`color_discrete_map`, `category_orders`) so
  legend order and colour agree.

## 4. Continuous scales

Hand-built from the palette's blue, so heatmaps belong to the same family:

```python
# Sequential, one hue: light = little, dark = much. Density, magnitude.
DENSITY_SCALE = [[0.0, "#f4f8fd"], [0.2, "#b7d3f6"], [0.4, "#6da7ec"],
                 [0.6, "#2a78d6"], [0.8, "#1c5cab"], [1.0, "#0d366b"]]

# The same in grey: under marks that are themselves coloured by group.
DENSITY_SCALE_GREY = [[0.0, "#f7f7f5"], [0.25, "#d9d8d3"], [0.5, "#b0afa9"],
                      [0.75, "#76756f"], [1.0, "#3a3a37"]]

# Diverging: blue lower, neutral grey equal, red higher. Differences only.
DIFFERENCE_SCALE = [[0.0, "#1c5cab"], [0.25, "#86b6ef"], [0.5, "#f0efec"],
                    [0.75, "#ee9191"], [1.0, "#b8302f"]]

# Short sequential for colouring points by a feature value.
VALUE_SCALE = [[0.0, "#9ec5f4"], [0.5, "#2a78d6"], [1.0, "#0d366b"]]
```

- **Sequential for amounts, diverging only for a signed difference.** A
  diverging scale is always symmetric about zero: `zmin=-span, zmax=+span` with
  `span = max(abs(values))`, so the neutral midpoint *is* "no difference".
- **Pin the range.** Panels that are compared share one colour axis
  (`coloraxis` with `cmin`/`cmax`), and a normalised quantity is fixed to 0–1
  rather than autoscaled, so the same colour is the same value across samples,
  genotypes and regions. Guard an all-zero panel: `zmax=top or 1.0`.
- **Blank where there is no information** (NaN, drawn transparent), not the
  bottom of the scale: no cells is not low intensity.
- Switch a heatmap to the grey ramp when coloured marks are drawn over it; blue
  under blue-and-orange dots hides the dots.
- The colour bar is titled with the unit, on the right side
  (`colorbar={"title": {"text": unit, "side": "right"}}`), and a caption says
  which end means what when it is a difference ("Red: more in mut than in wt
  there; …").
- No rainbow or jet, and no Plotly default scale left in by omission.

## 5. Opacity and edges

- Group fills (violin, box, bar bodies) are the group colour at **~0.6 alpha**,
  with the alpha in the fill colour (`rgba(r,g,b,0.6)`), not as trace opacity —
  trace opacity would fade the black edge too.
- Dots are the solid group colour (opacity ~0.9) with the black rim from the
  house-style wrapper: 1 px, 0.5 px above ~2000 points.
- Filled outlines (region and cell glyphs) take a lighter fill (~0.55 alpha)
  and a thin edge (~0.8).
- Marks laid over a heatmap just to be seen, not to encode a group, are
  near-white (`rgba(255,255,255,0.85)`) with the black rim.
- Lines that are lines — templates, trends, significance brackets — are
  `NEUTRAL`, 1.5 px, and are not outlined.
- Annotation text over the plot is `INK` at 12 px.

## 6. The user's colours

- Every palette is passed through `paint(kind, {value: default})` on its way
  out, which lays the user's picks for that kind over the defaults. A new plot
  that builds its own mapping without `paint` is a plot that ignores the
  sidebar.
- The **kind** is the column name ("Genotype", "Treatment", "Region", "Sample",
  "Cluster"). Use the same kind string in the plot as in the picker, or the pick
  does not reach it.
- Combined groups resolve through their genotype, so recolouring `wt` recolours
  `wt · DMSO` and `wt · drug` together.
- Pickers start at the default colour, and a value is stored only once it
  differs — so improving a default later still reaches users who never picked.
- Choices are remembered per user across workbooks (a JSON file in the app-data
  dir), because `wt` should not have to be recoloured for every experiment.

## 7. Light and dark theme

- Plot backgrounds are transparent (`plot_bgcolor="rgba(0,0,0,0)"`) and charts
  are drawn with `theme="streamlit"`, so they follow the page.
- Colours that must read on both — the template outline, brackets — are the mid
  grey `NEUTRAL`, not ink or white.
- **The black outline is weak on the dark theme.** The house style is designed
  for light; say so when delivering, and do not "fix" it by switching the
  outline to white in one plot.
- Exported matplotlib figures are always on `SURFACE` white with `INK` text,
  whatever the app theme.

## 8. Checklist

- No hex literal in a plot: a named constant from the shared module.
- Mapping built by the shared order/colour helpers, then through `paint(kind, …)`.
- Order passed alongside the colours; reference group first, unknown last and grey.
- Legend title names the coloured column.
- Sequential or diverging chosen for the right reason; range pinned and shared
  across compared panels; NaN blank.
- Fill alpha in the colour, not the trace; figure sent through `chart()`.
- Looked at with a group filtered out (others keep their colours) and with more
  than eight groups.
