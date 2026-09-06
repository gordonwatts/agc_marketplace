---
name: hist-plotter
description: Guidance for plotting histograms produced by a coffea processor - stacked per-region plots, systematic-variation comparison plots, and grids of auxiliary/feature histograms - and saving them (figures and a .root file) in preparation for statistical analysis.
---

# Histogram Plotter

## Overview

This skill provides the necessary information to take an existing analysis pipeline up until the point where histograms have been produced (e.g. by `coffea-processing`), and turn them into saved figures and a `.root` file ready for the statistical-analysis step (see `cabinetry-analysis`).

## Existing analysis pipeline and what you should have

At this point in the analysis pipeline, you should have `all_histograms` — a dict of `hist.Hist` objects. Depending on how the processor was written, this may be a single histogram, one per analysis region, and/or a further dict of auxiliary histograms (e.g. one per ML input feature).

## Core workflow

1. Check whether the analysis's support package already provides plotting/saving helpers (e.g. a `save_figure`, `plot_errorband`, or `save_histograms` utility) — prefer these over hand-rolled `plt.savefig`/`uproot.recreate` code, since they typically already handle consistent styling, saving both `.pdf` and `.png`, and building pseudo-data. Only fall back to the manual approach below if no such helper exists.
2. Produce a stacked per-process plot of each region's nominal histogram.
3. If there is more than one systematic variation, produce comparison plots overlaying nominal against each variation (grouped by systematic category) rather than stacking them.
4. If there are auxiliary histograms (e.g. one per ML input feature), plot each — a grid of subplots if there are many.
5. Save every figure as both `.pdf` and `.png`, in a figures directory (create it first if it doesn't exist).
6. Save histograms to a `.root` file for the statistical-analysis step.

## Plot a stacked per-process histogram

Make sure `mplhep`, `hist`, and `matplotlib` are imported.

If data is filled into the same histogram as MC (unified schema — see `coffea-processing`), plot it stacked directly:
```python
<hist_obj>[<slice for the nominal variation>, :, "nominal"].stack("process")[::-1].plot(stack=True, histtype="fill", linewidth=1, edgecolor="grey")
plt.legend(frameon=False)
plt.xlabel("<observable label>")
```

If data is kept in a separate histogram from MC (data/MC-split schema), plot data as points on top of the stacked MC instead:
```python
mplhep.histplot(all_histograms['data'], label='Data', color='black', histtype='errorbar')
hist.Hist.plot1d(all_histograms['MC'][:, :, 'nominal'], stack=True, histtype='fill', color=[<one color per process>])
```
Add an MC statistical-uncertainty error band if the support package provides one (e.g. a `plot_errorband` utility).

If there are multiple regions, repeat this once per region — with a title/xlabel describing that region — rather than only plotting one.

## Plot systematic-variation comparisons

For each systematic that's meaningful to inspect visually, overlay nominal and each variation as line plots for a single representative process (not stacked, since the goal is to compare shapes rather than yields):

```python
<hist_obj>[<slice>, "<process>", "nominal"].plot(label="nominal", linewidth=2)
<hist_obj>[<slice>, "<process>", "<variation_name>"].plot(label="<readable label>", linewidth=2)
# ... one line per variation to compare ...
plt.legend(frameon=False)
plt.xlabel("<observable label>")
plt.title("<systematic category> variations")
```
Group systematics that are conceptually related into the same comparison plot (e.g. all b-tagging variations together, all jet-energy variations together) rather than making one plot per individual systematic.

## Plot auxiliary/feature histograms

If the analysis produces one histogram per auxiliary variable (e.g. ML input features), loop over them and plot each; use a subplot grid rather than one figure per feature if there are more than a handful:
```python
fig, axs = plt.subplots(<rows>, <cols>, figsize=(<w>, <h>))
for i, feature_name in enumerate(<feature_names>):
    row, col = <index into axs>
    <hist_obj>[feature_name][:, :, "nominal"].stack("process").project("observable").plot(
        stack=True, histtype="fill", linewidth=1, edgecolor="grey", ax=axs[row, col]
    )
    axs[row, col].legend(frameon=False)
```

## Save figures

Prefer a support-package helper if one exists (it typically saves both formats and applies consistent styling):
```python
utils.save_figure("<descriptive plot name>")
```
Otherwise, save both formats manually into the figures directory:
```python
plt.savefig("<figures_dir>/<name>.pdf")
plt.savefig("<figures_dir>/<name>.png")
```

## Save histograms to disk

Prefer a support-package helper if one exists — it may already know how to serialize multi-axis `hist.Hist` objects and build pseudo-data from the sum of MC:
```python
utils.file_output.save_histograms(<hist_dict>, "<name>.root")
```

If no such helper exists, write directly with `uproot`, looping over every process (and every systematic variation that needs to be available downstream) explicitly:
```python
with uproot.recreate("<name>.root") as f:
    f["data"] = all_histograms['data']
    f["<process>"] = all_histograms['MC'][:, <process>, 'nominal']
    f["<process>_<systematic>_up"] = all_histograms['MC'][:, <process>, '<systematic>UP']
    f["<process>_<systematic>_down"] = all_histograms['MC'][:, <process>, '<systematic>DOWN']
    # repeat for every process, and every systematic each process needs
```
Match the naming convention used when the systematics were filled (see `corrections-systematics`) so the downstream `cabinetry` config's `VariationPath` entries line up with these histogram names.

For each new plot or file being produced, start a new code block (separated by an empty line) for better organization.
