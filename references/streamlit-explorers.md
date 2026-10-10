# Streamlit explorers over exported microscopy data

The viewer writes a workbook (per-sample, per-region, per-cell sheets); small
Streamlit apps plot and test it. Patterns that worked:

## Structure

- One shared module (`explorer_common.py`) with the cached readers
  (`@st.cache_data`), all statistics and all palettes, imported by every app, so
  two apps never disagree about a p value or a colour. No Streamlit UI in it
  beyond the cached readers and small sidebar helpers.
- Readers normalise identity columns to `str` (`Sample`, `Region`) and label
  blanks (`"(unknown)"`, `"(none)"`) so they become their own group instead of
  NaN.
- Merging sheets (Objects + Cell shapes + Cell intensities): drop columns the
  left frame already has (except the keys) before merging, or sample labels come
  back as `_x/_y`.

## Group by and normalise

- Detect group columns: Genotype plus any text column constant within a sample.
  Offer each, and "Genotype × condition" as a combined column ("wt · DMSO").
- Order groups: reference groups first (`wt`, `ctl`, `sib`), "(none)" last;
  combined groups condition by condition with genotypes side by side; keep each
  genotype's colour across conditions.
- Parametrise every `groupby(["Sample", "Genotype"])` by the chosen column; the
  mixed model copies the group into its own formula column, so names with spaces
  are fine.
- "Normalize to" dropdown: divide the variable by any other numeric column row
  by row (zero denominator → NaN, not inf); put the most likely denominator (e.g.
  whole-brain area "WB") first. Name the result `"A / B"`.
- Whole-brain area: prefer a drawn region called WB, else the union of outlines
  (rasterise polygons; nested regions must not be summed), else the sum.

## User-chosen colours that apply everywhere

- Route **every** palette through one function, e.g.
  `paint(kind, {value: default_colour})`, which overlays the user's choices for
  that kind (Genotype, a condition, Region, Sample, Cluster). Grep for every place
  a colour map is built (`color_discrete_map`, palette dicts, `styles[g][0]`) and
  replace them — one missed site is one plot that ignores the choice.
- Keep the choices in `st.session_state`, loaded once from a JSON file in the
  user's app-data dir and written back on change, so they persist across
  workbooks and sessions. Tests redirect the file.
- Sidebar: a kind selector and one `st.color_picker` per value, each starting at
  the default colour; record a value only when it differs from the default (or
  was already recorded), `st.rerun()` after a change, and a Reset button that
  clears the kind and the pickers' session keys. A pick should cost exactly one
  extra rerun — count the calls in a test to rule out loops.
- Unify defaults too: the same column must get the same default colours in the
  box plot, the scatter and the UMAP.

## Graph aesthetics (the house style)

The user's preferred look, for every plot — interactive and exported:

- **Every mark outlined in black** over its group's colour: violins, boxes, bars,
  dots (sample dots, scatters, UMAPs, cell plots), filled shapes (region/cell
  glyphs, thinner edge ~0.8). Group fills fairly solid (~60 % opacity) so the
  colour holds next to the black; dot rims ~1 px, thinner (~0.5) above ~2000
  points so dense clouds do not turn to ink.
- **Violins without an inner box plot** (`box_visible=False`); the shape, a black
  mean/median line and the individual dots carry the information.
- Leave alone: plain lines (trend lines, templates, contours) and heatmaps — a
  "black outline" on a line or an image means nothing.

Apply it in **one place**, not per plot, so no chart can miss it:

```python
OUTLINE = "#000000"

def outlined(figure):
    for t in figure.data:
        if t.type in ("violin", "box"):
            t.update(line={"color": OUTLINE, "width": 1.5})
            if t.type == "violin":
                t.update(meanline={"color": OUTLINE})
        elif t.type in ("bar", "histogram"):
            t.update(marker_line_color=OUTLINE, marker_line_width=1)
        elif t.type in ("scatter", "scattergl", "scatter3d"):
            if getattr(t, "fill", None) not in (None, "none", ""):
                t.update(line={"color": OUTLINE, "width": 0.8})
            elif "markers" in (t.mode or "markers"):
                n = len(t.x) if t.x is not None else 0
                t.update(marker_line_color=OUTLINE, marker_line_width=0.5 if n > 2000 else 1)
    return figure

def chart(figure, **kwargs):          # replace every st.plotly_chart( with chart(
    return st.plotly_chart(outlined(figure), **kwargs)
```

Styling last means per-plot leftovers (old white marker rims) are overridden.

Matplotlib report figures get the same look: `edgecolors="#000000"` on scatter
dots and legend markers, violin bodies with `set_facecolor(to_rgba(color, 0.6))`,
`set_edgecolor(OUTLINE)` and `set_alpha(None)` — `set_alpha` fades the outline too,
so put the alpha in the face colour. Very large point clouds (tens of thousands
of cells) stay un-outlined; outline their summary markers instead.

Checking it: without `kaleido` plotly cannot export PNGs; inspect the traces in
`AppTest` chart specs (every violin/box/bar/marker line colour is `#000000`,
`box.visible` is false) and look at the matplotlib PNGs directly. Black outlines
are hard to see on Streamlit's dark theme — say so when delivering.

## Spatial heatmaps on a registered template

- Density: histogram cell positions on a grid over the template, Gaussian-smooth,
  divide by pixel area.
- Intensity: Gaussian-weighted mean = smooth(sum of values) / smooth(counts);
  blank (NaN) where the smoothed count is below a fraction of one cell — no cells
  says nothing about intensity. Average samples with `nanmean`.
- Normalise intensities to *all processed cells* (1st→0, 99th→1 percentile of the
  unfiltered workbook) and fix the colour scale to 0–1, so colours are comparable
  across samples, genotypes and regions.
- Symmetric test shapes can register flipped 180°; use asymmetric outlines in
  tests that check direction.

## Testing apps headless

```python
from streamlit.testing.v1 import AppTest
sys.argv = ["apps/app.py", workbook]        # apps read the path from argv
at = AppTest.from_file("apps/app.py", default_timeout=300).run()
assert not at.exception
at.selectbox(key="group-by").set_value("Treatment").run()
at.color_picker(key="color-Genotype-wt").set_value("#ff00aa").run()
at.button(key="colors-reset-Genotype").click().run()
```

- Plotly specs in `at.get("plotly_chart")[i].proto.spec` pack numpy arrays as
  `{"dtype": ..., "bdata": base64, "shape": "r, c"}`; decode with
  `np.frombuffer(base64.b64decode(bdata), dtype)` before checking values.
- Build a synthetic workbook with a known effect (e.g. brighter on the right in
  mutants) and assert the sign of the result, not just "no exception".
- The first run in a fresh process can be slow (numba/UMAP compilation); give it
  a generous timeout before suspecting a rerun loop.
