---
name: corrections-systematics
description: Guidance for applying correctionlib-based corrections and implementing systematic uncertainties (weight, object/kinematic, and sample-level) when filling histograms in a HEP analysis pipeline. Use alongside coffea-processing whenever a histogram needs more than a single nominal weight per event.
---

# Corrections and Systematics

## Overview

This skill covers how to apply per-event corrections and propagate systematic uncertainties into histograms, using `correctionlib` for scale-factor-like corrections, and covering the different structural kinds of systematic variation that show up in most particle-physics analyses:

- **Weight systematics** — same objects and event selection, just a reweighted fill.
- **Object/kinematic systematics** — a rescaled or resmeared object that can change which events pass selection, so the whole selection + observable calculation has to be redone.
- **Sample-level ("two-point") systematics** — the variation is a wholly separate simulated dataset rather than something computed inline.

Use this skill together with `coffea-processing` when the processor needs to fill histograms with more than a single "nominal" weight per event.

## Core workflow

1. Classify each systematic uncertainty into one of the three categories above — this determines where in the processing loop it needs to be handled.
2. Load corrections from a `correctionlib` file, if one is provided.
3. Implement weight systematics: evaluate a correction up/down and multiply it into the nominal per-event weight when filling histograms.
4. Implement object/kinematic systematics: rescale/resmear the relevant object collection *before* selection, and rerun the entire selection → observable → fill sequence for each variation.
5. Implement sample-level systematics: if the input dataset/fileset entry itself represents a variation, fill its histogram entries directly under that sample's own variation label — do not also apply the weight/kinematic loop on top of it.
6. Adopt a consistent variation-naming scheme (`<name>_up` / `<name>_down`) so a downstream statistical-analysis config (see `cabinetry-analysis`) can find the corresponding histograms by suffix.

## 1. Classify the systematic

Ask: does this uncertainty change *which particles pass selection or what the observable's value is* (object/kinematic), or does it only change *how much each already-selected event counts* (weight)? Or is it represented as an *entirely separate sample* rather than a per-event computation (sample-level)?

- **Weight systematic** — e.g. a b-tagging efficiency scale factor, a pileup reweighting, a lepton ID/trigger scale factor. Computed from quantities of the *nominal* selected objects; only multiplies into the fill weight. Cheapest to compute — selection is only ever run once (for nominal).
- **Object/kinematic systematic** — e.g. jet energy scale/resolution, muon momentum scale. Rescales an object's kinematics *before* the object/event selection is applied, so it can change which objects/events pass. Requires rerunning selection + observable calculation once per variation.
- **Sample-level systematic** — e.g. a renormalization/factorization scale variation, an alternate generator/parton-shower sample, or a detector mis-modeling sample. Delivered as its own dataset with its own `xsec`/`nevts`, tagged with a variation name in its metadata rather than computed on the fly.

## 2. Load corrections with correctionlib

```python
import correctionlib
cset = correctionlib.CorrectionSet.from_file("<corrections_file>.json")
```

`cset` maps correction name to a `Correction` object. Before calling `.evaluate(...)` on one, check its expected inputs — correctionlib schemas are not uniform: some take `(systematic_name, "up"/"down", <variable>)`, others take direct physics inputs like `(eta, pt)` for a plain scale factor with no variation axis. Inspect `cset["<name>"].inputs` (each has a `.name` and `.description`) rather than guessing the argument order, or check the documentation/README accompanying the corrections file if one exists.

If no corrections file is provided and one needs to be built from scratch (e.g. via `correctionlib.schemav2`), ask the user for the functional form of each correction (flat/binned scale factor, formula-based, etc.) rather than inventing values.

## 3. Weight systematics

Evaluate the correction for each direction and multiply it into the nominal weight when filling:

```python
for direction in ["up", "down"]:
    wgt_variation = cset["<correction_name>"].evaluate("<correction's variation argument>", direction, <input_array_from_nominal_objects>)
    hist_dict[region].fill(
        observable=observable, process=process,
        variation=f"<systematic_name>_{direction}", weight=nominal_weight * wgt_variation,
    )
```

`<input_array_from_nominal_objects>` should come from the *already-selected, nominal* objects for the current region — do not recompute the selection for weight systematics.

If a weight systematic should only apply to specific processes (e.g. a background-specific scale uncertainty), guard it with a check on `process` before adding it to the list of systematics to loop over, rather than computing it unconditionally and discarding the result.

## 4. Object/kinematic systematics

These require an outer loop over variations that reruns selection, since the set of objects/events passing selection can itself change:

```python
# compute the rescaled/resmeared quantity once, up front
events["<syst>_up"] = <scale factor, e.g. 1.03, or an array from a smearing helper>

kinematic_systs = ["<syst>_up", ...]
syst_variations = ["nominal"] + kinematic_systs + weight_systs   # only expand beyond "nominal" for samples whose own metadata variation is "nominal" -- see section 5

for syst_var in syst_variations:
    objects = events.<Collection>
    if syst_var in kinematic_systs:
        objects["<kinematic_field>"] = objects.<kinematic_field> * events[syst_var]

    # ... redo the full object + region selection here using `objects` ...
    # ... recompute the observable here ...

    if syst_var in weight_systs:
        # see "Weight systematics" above -- loop over up/down and fill
        ...
    else:
        variation_name = syst_var  # "nominal", or "<syst>_up"/"<syst>_down" for a kinematic variation
        hist_dict[region].fill(observable=observable, process=process, variation=variation_name, weight=nominal_weight)
```

## 5. Sample-level ("two-point") systematics

If the fileset/dataset entry being processed is itself a systematic variation (check e.g. `events.metadata["variation"]`), do not run the object/kinematic/weight-systematics loop on it — just run the nominal-style selection once and fill the histogram directly under that sample's own variation name:

```python
variation = events.metadata["variation"]  # e.g. "nominal", "scaledown", "ME_var"
if variation == "nominal":
    # expand syst_variations as in section 4 and compute inline systematics
    ...
else:
    # run nominal-style selection only, then:
    hist_dict[region].fill(observable=observable, process=process, variation=variation, weight=nominal_weight)
```

This avoids double-counting: a `scaledown` sample shouldn't also have b-tagging or jet-energy variations computed on top of it in the same histogram, since that would conflate two different sources of uncertainty into one set of bins.

## 6. Naming convention

Use `<name>_up` / `<name>_down` for every two-sided systematic, whether weight-based or kinematic. This lets a downstream `cabinetry` YAML config reference the corresponding up/down histograms purely by string suffix (see `cabinetry-analysis`) without per-systematic special-casing.
