# napari-dev

A [Claude Code](https://claude.com/claude-code) skill for building and debugging
napari-based microscopy apps and plugins.

It collects practical rules that came out of building a full microscopy viewer:
dock panels and window sizing on laptops, Labels/Points/Image layers, mouse and
keyboard interaction, thread workers, multiscale/dask/.ims data, units and scale
bars, Cellpose segmentation, macOS app bundles, and companion Streamlit
explorers for the exported data.

## Install

Clone into your personal skills directory:

```sh
git clone https://github.com/daggermaster3000/napari-dev-skill.git ~/.claude/skills/napari-dev
```

Or into a single project:

```sh
git clone https://github.com/daggermaster3000/napari-dev-skill.git .claude/skills/napari-dev
```

Claude Code picks the skill up automatically when you work on code that imports
`napari`, `qtpy` or `cellpose`, or when you describe a napari symptom ("the
window can't be resized", "the scale bar says pixels").

## Contents

| File | Covers |
| --- | --- |
| `SKILL.md` | How to work, the rules that bite most often, done-checklist |
| `references/windows-and-docks.md` | Dock minimum sizes, scroll areas, screen fitting |
| `references/interaction.md` | Button slots, canvas clicks, label editing, thread workers, shortcuts |
| `references/data-and-io.md` | Lazy reading, pyramid levels, scale and units, HDF5 writes |
| `references/cellpose.md` | Device choice, µm sizes, 2D + stitch, model families |
| `references/packaging-and-startup.md` | Splash, launchers, macOS `FileOpen` events |
| `references/streamlit-explorers.md` | Shared reader module, caching, palettes, plot house style |
| `references/testing.md` | Driving the real UI, screenshots, headless checks |
