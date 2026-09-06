---
name: coffea-processing
description: Guidance for processing retrieved data with coffea into histograms - object/event selection (including multiple analysis regions), event weighting, and running the processor locally or with Dask. Pairs with corrections-systematics for anything beyond a single nominal weight per event.
---

# Coffea Processing

## Overview

This skill provides the necessary information to take an existing analysis pipeline up until the point where the data has been retrieved (e.g. via `ServiceX`, or a directly-accessible fileset) and turn it into histograms. The main things to do are: normalize events based on cross-section/luminosity (or data), apply object and event selection (possibly across more than one analysis region), and fill histograms via a class inheriting from a coffea processor.

For anything beyond a single nominal weight per event — scale-factor corrections, or up/down systematic variations — see the `corrections-systematics` skill, which this one defers to.

## Existing analysis pipeline and what you should have

At this point in the analysis pipeline, you should have:

- A `fileset` dictionary mapping dataset/process names to `{"files": [...], "metadata": {...}}`. `metadata` may be as simple as a `dataset_name`, or may carry richer per-sample info such as `process`, `variation`, `xsec`, `nevts` if it was produced by a fileset-construction utility rather than a hand-built dict.
- Knowledge of how many analysis regions are needed (a single signal region is common, but a control-region/signal-region pair, or more, is also common) and what defines each one.

## Choose a schema

Use `coffea.nanoevents.NanoAODSchema` for CMS/ATLAS NanoAOD-style inputs — it adds cross-references between object collections (e.g. `Jet.matched_muon`) and attaches four-vector behavior automatically. Use `BaseSchema` for flat, ntuple-style inputs (e.g. many ServiceX-delivered flat trees) where no such cross-references or object typing are needed. Infer which applies from the input file structure, or ask if unclear; don't default to `BaseSchema` unconditionally. If using `NanoAODSchema` and some branches genuinely won't be used, `NanoAODSchema.warn_missing_crossrefs = False` silences the resulting warnings.

## Normalize events

Prefer per-sample metadata already attached to the fileset, if present — this is the more portable approach and avoids maintaining a separate cross-section table:

```python
x_sec = events.metadata["xsec"]
nevts_total = events.metadata["nevts"]
lumi = <luminosity, in pb^-1>
xsec_weight = x_sec * lumi / nevts_total if events.metadata["process"] != "data" else 1
```

If no such metadata is attached to the fileset (e.g. a hand-built `input_files` dict rather than a construction utility), fall back to an external lookup: write a method that takes a sample name and returns its cross-section from a lookup table (e.g. an `infofile`-style module or a JSON/YAML map), and compute the weight the same way. Either way: data always gets weight 1; MC gets `xsec * lumi / nevts`.

## Select objects and events

Apply object-level requirements (kinematic and ID cuts) to each object collection, then build event-level selection masks from the filtered collections.

For a single combined selection, `ak.sum`/`ak.num` plus boolean array operations are enough. Once there is more than one named cut, or regions are built from combinations of cuts, use `coffea.analysis_tools.PackedSelection` instead of tracking separate boolean arrays by hand — it names each cut and lets regions be defined as boolean combinations of named cuts:

```python
from coffea.analysis_tools import PackedSelection

selections = PackedSelection(dtype='uint64')
selections.add("<cut_name_1>", <boolean mask over events>)
selections.add("<cut_name_2>", <boolean mask over events>)
# combine named cuts into a named region
selections.add("<region_name>", selections.all("<cut_name_1>", "<cut_name_2>"))
```

Note this only defines the final-state selection — it does not yet distinguish signal-like from background-like events; that's the job of the discriminating observable and, later, the fit.

## Handle multiple regions

If the analysis defines more than one region (e.g. a control region and a signal region), loop over them and build one histogram (or histogram-dict entry) per region, rather than writing near-duplicate code per region:

```python
for region in ["<region_1>", "<region_2>", ...]:
    region_mask = selections.all(region)
    region_objects = <object_collection>[region_mask]
    ...
    if region == "<region_1>":
        observable = <region_1's observable expression>
    elif region == "<region_2>":
        observable = <region_2's observable expression>
    hist_dict[region].fill(observable=observable, process=process, variation=variation_name, weight=region_weight)
```

Each region typically has its own observable definition and its own histogram object (see "Create histograms" below), but shares the same event-weight/normalization logic.

## Create the processor class

Create a class named after the analysis (e.g. `TtbarAnalysis`), inheriting from `processor.ProcessorABC`.

The constructor should build the histogram object(s) needed (see "Create histograms" below) and load anything needed for the full run once — such as `correctionlib.CorrectionSet.from_file(...)` (see `corrections-systematics`) — in `__init__`, not per-event in `process`.

Define `process(self, events)`, containing, in order:
1. Event weighting (see "Normalize events").
2. Any object/kinematic systematics setup and looping, if applicable (see `corrections-systematics`) — this wraps the following two steps.
3. Object + event selection, possibly per-region (see above).
4. Observable computation and histogram filling, possibly per-region.

Return a dictionary from `process()`, e.g. `{"nevents": {<dataset_name>: len(events)}, "hist_dict": hist_dict}` — include whatever histogram dict(s) were filled (region histograms, and separately any auxiliary histograms, e.g. for ML input features, if applicable).

### `postprocess()`

Define `postprocess(self, accumulator)`; unless the analysis needs cross-chunk postprocessing beyond what `coffea`'s accumulation already does, this can just `return accumulator`.

## Create histograms

Pick a histogram schema based on how many event categories need to be distinguished.

**Data kept structurally separate from MC** (useful when data will always be treated differently downstream, e.g. plotted as points rather than a stacked bar):
```python
<var>_hist_data = hist.Hist.new.Reg(num_bins, bin_edge_low, bin_edge_high, name="<var>", label="<label>").Weight()
<var>_hist_MC = (
    hist.Hist.new.Reg(num_bins, bin_edge_low, bin_edge_high, name="<var>", label="<label>")
    .StrCat([<process names>], name="dataset")
    .StrCat(["nominal", <systematic variation names>], name="variation")
    .Weight()
)
```

**Unified histogram covering data and every MC process together** (simpler when there's no structural reason to keep data separate — e.g. data is just filled with weight 1 like any other process, and pseudo-data is built later by summing MC):
```python
<region>_hist = (
    hist.Hist.new.Reg(num_bins, bin_edge_low, bin_edge_high, name="observable", label="<label>")
    .StrCat([], name="process", label="Process", growth=True)
    .StrCat([], name="variation", label="Systematic variation", growth=True)
    .Weight()
)
```
Use `growth=True` categorical axes when the full list of process/variation names isn't known up front (e.g. discovered as different fileset entries are processed).

If there are multiple regions, build one histogram per region (e.g. a `dict` keyed by region name), each with its own binning if the observable differs by region. If the analysis also produces auxiliary histograms (e.g. one per ML input feature), build those the same way in a separate dict, generally only in the region(s) where they're meaningful.

## Fill histograms

Fill with `variation="nominal"` and the sample's nominal weight for the central value. For anything beyond that — up/down systematic variations, or a sample that is itself a systematic variation — see `corrections-systematics` for how to compute the weight/variation and what to name it (`<name>_up`/`<name>_down`).

## Choose an executor and run the processor

For local processing:
```python
executor = processor.FuturesExecutor(workers=<NUM_CORES>)
```

For distributed processing with Dask (useful for large-scale runs), register the analysis package with `cloudpickle` so it ships correctly to workers, then use a Dask executor instead:
```python
import cloudpickle
cloudpickle.register_pickle_by_value(<analysis_package>)
executor = processor.DaskExecutor(client=<dask_client>)
```
Choose between the two behind a boolean flag (e.g. `USE_DASK`) defined among the pipeline's global settings, rather than hardcoding one.

Define a `processor.Runner`:
```python
run = processor.Runner(executor=executor, savemetrics=True, metadata_cache={}, chunksize=<CHUNKSIZE>, schema=<schema chosen above>)
```

Before calling the runner, determine the tree name: if the data was retrieved with a raw/uproot-style query (e.g. `UprootRaw`), use the original tree name (or whatever `ServiceX` renamed it to); if retrieved with `func_adl`, `ServiceX` names the returned tree `"servicex"`.

Preprocess, then run, and time it since this is usually the most expensive step:
```python
filemeta = run.preprocess(fileset, treename=tree_name)

t0 = time.monotonic()
all_histograms, metrics = run(fileset, tree_name, processor_instance=<AnalysisClassName>(...))
exec_time = time.monotonic() - t0
print(f"execution took {exec_time:.2f} seconds")
```

`metrics` contains runtime statistics about the processing; log them if the pipeline has a metrics-tracking utility, otherwise they can be discarded.
