---
name: cabinetry-analysis
description: Guidance for implementing the statistical analysis component of an analysis in HEP, using cabinetry. Covers loading a pre-existing configuration or authoring one from scratch, building/fitting a workspace (including rebinning overrides), producing fit-result plots, and validating against a second/pruned workspace.
---

# Cabinetry Analysis

## Overview

This skill provides the necessary information to take an existing pipeline that produces histograms and implement the statistical analysis using `cabinetry`.

## Core workflow

1. Determine whether a `cabinetry` configuration file already exists for this analysis, or needs to be authored from scratch.
2. Either load the existing config, or build a new `.yml` config file from the high-level information (POI, regions, systematics, etc.) provided by the user.
3. Use the `cabinetry` Python package to build the workspace and fit the model to data.
4. Create useful plots of the fit results, such as pull plots, correlations, and post-fit distributions.
5. If there is a second observable/workspace that should give a consistent result (e.g. a cross-check using ML-based features instead of the primary observable), validate against it by matching the fit result onto it.

Below are explanations of these steps in more detail.

## Step 0: existing config, or author from scratch?

Check whether a `.yml`/`.yaml` `cabinetry` config file already exists alongside the pipeline (e.g. passed in as a support file, or referenced by name in the prompt). If it does, **do not rewrite or regenerate it** — load it as-is (see "Build workspace" below) and treat its `Samples`/`Regions`/`Systematics`/`NormFactors` entries as already-decided, matching whatever histogram names were produced upstream. Only proceed to "Create configuration .yml file" below if no config file exists yet and one genuinely needs to be authored.

## Create configuration .yml file

Use the information given to you in the pipeline in order to create a .yml file representing the configuration of the statistical analysis.
Put this new file in the current analysis workspace, do not modify unrelated files, and name the file `config.yml`.
Below is a brief summary of general guidelines you should follow, the syntax and exact structure you should follow for creating the .yml file, and the meanings of every entry in the config file.

General info:
See user prompt for this general info

Syntax to be followed:

```
General:
  Measurement: "<measurement_name>"
  POI: "<POI>"
  HistogramFolder: "<histogram_directory>"
  InputPath: "<hist_root_filename>.root:{SamplePath}{VariationPath}"
  VariationPath: ""

Regions:
  - Name: <Region_1>
  - Filter: <region_selection> [if more than one region]

[add more regions as necessary -- one per analysis region produced upstream, e.g. by coffea-processing's "Handle multiple regions" section]

Samples:
  - Name: "Data"
    SamplePath: "<data_path>"
    Data: True

  - Name: "Signal"
    SamplePath: "<signal_path>"

  - Name: "Background <bkg_1_name>"
    SamplePath: "<bkg_1_path>"

[add more background processes as necessary]

Systematics:
  - Name: "<shape_syst_name>"
    Type: "NormPlusShape"
    Up:
      VariationPath: “<up_variation_suffix>"
    Down:
      VariationPath: “<down_variation_suffix>"

[more sources of systematic uncertainty affecting distribution shape -- one entry per weight/kinematic systematic produced upstream, see corrections-systematics]

  - Name: "<norm_syst_name>"
    Type: "Normalization"
    Up:
      Normalization: <up_shift>
    Down:
      Normalization: <down_shift>
    Samples: "<relevant_samples>"

[more sources of systematic uncertainty affecting only normalization]


NormFactors:
  - Name: <norm_factor_name>
    Samples: <nf_sample>
    Nominal: 1.0
    Bounds: [<nf_lbnd>, <nf_hbnd>]

[more NormFactors which scale specific samples without prior constraints]
```

Meanings of the parameters:
- <measurement_name> = string, name of the measurement being done (Ex: CabinetryHZZAnalysis)
- <POI> = parameter of interest, will be specified by the user for now
- <histogram_directory> = directory where histograms are to be saved, and directory where histograms (or ntuples are to be read from) are taken from for input
- <hist_root_filename> = root file where histograms or ntuples are to be taken from.
- <Region_1> = name of Region 1 (example: signal region. Other regions might be control region, validation region, etc.)
- <region_selection> = criteria events must satisfy to be in this region (example: pT > 100). This parameter is unnecessary if there is only one region.
- <data_path> = path to histogram (the name of the histogram in the .root file) where data is stored (usually it is called just "data". Check the earlier code where the histograms were created.)
- <signal_path> = path to histogram where MC simulated signal is stored (usually it is called just "signal". Check the earlier code where the histograms were created.)
- <bkg_1_name> = name of background process 1 that is being modelled
- <bkg_1_path> = path to histogram where MC simulated background process 1 is stored (usually it will be called the name of the particular background process we are considering, such as "ZZ" or "Z_tt". Check the earlier code where the histograms were created.)
- <shape_syst_name> = unique name identifying the systematic uncertainty (e.g. JES, TauID, PDF). This label corresponds to a single nuisance parameter in the fit and should reflect the physical source of uncertainty.
- <up_variation_suffix> = suffix appended to the nominal histogram path to access the +1σ (upward) variation of this systematic. Must match the naming convention the histograms were actually saved under (see corrections-systematics and hist-plotter -- typically `<name>_up`).
- <down_variation_suffix> = suffix appended to access the −1σ (downward) variation histogram. Together with the up variation, this defines how the shape and normalization change as the nuisance parameter varies.
- <norm_syst_name> = unique name for a normalization-only systematic (e.g. lumi, ZZ_norm). This defines a nuisance parameter that scales yields without altering histogram shapes.
- <up_shift> = fractional change in yield for a +1σ shift (e.g. 0.1 means +10%). This defines how much the sample normalization increases when the nuisance parameter is +1.
- <down_shift> = fractional change for a −1σ shift (e.g. -0.1 means −10%). Typically symmetric with the up shift, but not required to be.
- <relevant_samples> = sample or list of samples affected by this systematic (e.g. "Background ZZ" or ["ttbar", "Wjets"]). This restricts where the nuisance parameter is applied in the model.
- <norm_factor_name> = name of the normalization factor (e.g. mu_signal, ttbar_norm). This becomes a free parameter in the fit and may represent the parameter of interest (POI) or a floating background normalization.
- <nf_sample> = sample (or list of samples) that this NormFactor scales. It defines which components of the model are multiplied by this parameter during the fit.
- <nf_lbnd> = lower bound allowed for the NormFactor during the fit (e.g. 0). This constrains the physically allowed range of the parameter.
- <nf_hbnd> = upper bound for the NormFactor (e.g. 10). This prevents unphysical or numerically unstable solutions during the likelihood maximization.

## Build workspace

The code given to you so far in the pipeline produces histograms, but does not yet include the statistical analysis.
You will now start implementing the statistical analysis with cabinetry, by creating a workspace.
First, check if cabinetry has been imported in the analysis file; if not, import it.
The easiest way to create a cabinetry workspace is to start with a configuration object, loaded from a `.yml` file (either pre-existing, or the one just authored in the previous step):
```
cabinetry_config = cabinetry.configuration.load("<path/to/config/file>")
```
Print the ["Samples"] and ["Systematics"] elements of the config object.

Then build/collect the templates. There are two ways to do this:

- `cabinetry.templates.collect(cabinetry_config)` — reads the histograms exactly as they were saved, with no modification. Use this when the saved binning is already what should be fit.
- `cabinetry.templates.build(cabinetry_config, router=<router>)` — use this instead when a rebinning (or other template-building override) is needed before fitting, e.g. merging bins above some threshold. Build the router first:
  ```python
  rebinning_router = utils.rebinning.get_cabinetry_rebinning_router(cabinetry_config, rebinning=slice(<lower_edge>j, None, hist.rebin(<merge_factor>)))
  cabinetry.templates.build(cabinetry_config, router=rebinning_router)
  ```
  (the exact helper used to build a router may be provided by the project's support package rather than written from scratch — check for one before implementing the rebinning logic manually.)

Either way, follow with:
```
cabinetry.templates.postprocess(cabinetry_config)
```
Then create a workspace object using
```
ws = cabinetry.workspace.build(cabinetry_config)
cabinetry.workspace.save(ws, "<workspace_name>.json")
```
This workspace is what you will use to do the rest of the statistical analysis.

## Fit model to data and create plots

The code given to you so far in the pipeline is written up until the creation of a statistical workspace.
This statistical workspace contains information on the model for our data, and the data itself.
You will now write code to take this workspace and actually perform the fit, generating output plots for the user to analyse the results.

Start a new block of code and implement as follows:

Write the code snippet
```
model, data = cabinetry.model_utils.model_and_data(<workspace>)
```
This creates model and data objects. The argument <workspace> is to be replaced by the name of the cabinetry workspace object previously created.

Then perform the fit using
```
fit_results = cabinetry.fit.fit(model, data)
```

Now we create plots showing the fit results. There are multiple plots we might want to examine.

To make a pull plot:
```
cabinetry.visualize.pulls(
    fit_results, exclude="<POI_name>", close_figure=False, save_figure=False
)
utils.save_figure("pulls")
```
The above code creates both a .pdf and a .png file of the plot. If we want only a .pdf, omit the last line beginning with `utils` and set save_figure to True in the cabinetry.visualize.pulls() line. By default, save as both a .pdf and .png unless otherwise specified.

To make a plot visualizing correlations between parameters:
```
cabinetry.visualize.correlation_matrix(
    fit_results, pruning_threshold=0.15, close_figure=False, save_figure=False
)
utils.save_figure("correlation_matrix")
```
Follow the same changes as before in the case that we want only a .pdf plot.

Now we want to actually visualize our fit with the data. Create a post-fit (and, if useful for comparison, pre-fit) model prediction using:
```
model_prediction = cabinetry.model_utils.prediction(model)
model_prediction_postfit = cabinetry.model_utils.prediction(model, fit_results=fit_results)
```

For the data/model comparison plot, prefer passing the already-loaded `cabinetry_config` directly if it already contains full binning/labeling info for every region — this covers all regions in one call and needs no manual construction:
```
figs = cabinetry.visualize.data_mc(model_prediction_postfit, data, config=cabinetry_config, close_figure=True)
```
If the project's support package provides its own comparison-plot helper (e.g. a grid view showing pre- and post-fit together), consider calling it too, but the `cabinetry.visualize.data_mc` call above should not be skipped.

Only fall back to building a manual `plot_config` object when there is no full config available for this call (e.g. a bespoke workspace without a matching config file): a python dict with a "Regions" key, whose value is a list of `{"Name": <region_name>, "Binning": <list from a linspace>}` dictionaries, one per region, passed as `config=plot_config` to `cabinetry.visualize.data_mc`.

Either way, for each figure, give it a meaningful x-axis label:
```
fig = figure_dict[i]["figure"]
fig.axes[1].set_xlabel("<title of quantity being plotted>")
```
`i` indexes the particular plot being made as listed in the figure dict/list (one per region).

Then save and export each figure:
```
utils.save_figure("<Descriptive name for the plot in this region>")
```

For each new plot being made, start a new code block (separated by line breaks) for better organization.

Save these plots in the same directory as the config file.

## Validate against a second/derived workspace

Some analyses build a second workspace from a different observable (e.g. an ML-based feature) to cross-check the fit result from the primary workspace, rather than fitting it independently. If asked to do this:

1. Load the second config and build/collect its templates and workspace the same way as above (`cabinetry.configuration.load(...)`, `cabinetry.templates.collect`/`.build`, `cabinetry.templates.postprocess`, `cabinetry.workspace.build`).
2. If only a subset of channels from this second workspace are relevant (e.g. only some of many feature channels), prune it down first:
   ```python
   ws_pruned = pyhf.Workspace(<second_workspace>).prune(channels=[<relevant_channel_names>])
   ```
3. Build the model and data for the (pruned) second workspace: `model_2, data_2 = cabinetry.model_utils.model_and_data(ws_pruned)`.
4. Rather than independently re-fitting, match the *primary* fit result onto this second model so the comparison reflects the same fitted parameters:
   ```python
   fit_results_matched = cabinetry.model_utils.match_fit_results(model_2, fit_results)
   ```
5. Produce pre-/post-fit predictions and a `data_mc` comparison plot for the second model the same way as above, using `fit_results_matched` in place of `fit_results`.
