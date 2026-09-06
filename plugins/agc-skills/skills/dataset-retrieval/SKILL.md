---
name: dataset-retrieval
description: Guidance for retrieving data from dataset names using func_adl and ServiceX, producing a cached fileset ready for coffea processing. Covers both a hand-built file-list dict and a metadata-rich fileset built via a construction utility.
---

# Dataset Retrieval

## Overview

This skill provides the necessary information to take an existing analysis pipeline up until the point where the datasets to process, and the variables and selections needed downstream, have been decided. It builds a `func_adl` query, delivers it with `ServiceX`, and produces a `fileset` dictionary of cached file locations ready to be handed off to `coffea` for processing.

## Existing analysis pipeline and what you should have

At this point in the analysis pipeline, you should have one of:

- A simple dictionary (e.g. `input_files`) mapping each dataset category (observed data, the signal process, and each background process) to a list of file paths/URLs or rucio dataset identifiers, **or**
- A richer `fileset` dictionary already built by a fileset-construction utility (e.g. `<package>.file_input.construct_fileset(...)`), where each entry already carries `{"files": [...], "metadata": {"process": ..., "variation": ..., "xsec": ..., "nevts": ..., ...}}` — this is common when the analysis has multiple simulated variations per process (e.g. `ttbar__nominal`, `ttbar__scaledown`) in addition to plain signal/background/data categories. If such a utility and its driving config (e.g. number of files per sample, an access-point/site setting) are already available, use it rather than hand-building the file lists — it's the source of the `xsec`/`nevts` metadata that `coffea-processing`'s normalization step relies on.

Either way, you should also have:
- Knowledge of the name of the tree in the input files (e.g. `"mini"`, `"Events"`), and which branches/columns are available.
- Knowledge of the event-level selection needed to define the final state under study (e.g. requiring a specific number of leptons or jets), and of every variable needed downstream: quantities used to compute the discriminating variable(s), quantities used for event weighting (Monte Carlo weight, scale factors), and any systematics computed at the query stage rather than later (e.g. because the ingredients needed for the variation would otherwise have to be carried through the whole pipeline unnecessarily).
- Global settings decided earlier in the pipeline, such as whether to bypass ServiceX's local cache (e.g. `IGNORE_CACHE`), and whether delivered files should be downloaded locally or accessed remotely by URL (e.g. `USE_SERVICEX_DOWNLOAD`).

## Core workflow

1. Write a function that builds a `func_adl` query: select the tree, apply the event-level selection, and select the dictionary of output columns (including any systematics computed at this stage).
2. Build a bundle/list of `Sample` entries, one per dataset category (or per fileset entry, if using a metadata-rich fileset), each pointing at that category's files and using the same query.
3. Call `servicex.deliver()` once on the bundle to retrieve/cache the data, choosing a delivery mode appropriate to the pipeline's settings.
4. Turn the `results` returned by `deliver` into a `fileset` dictionary in the format `coffea` expects (or update the existing fileset's file lists in place, if one was already built), and print it to confirm the data are ready.

Below are explanations of the steps in more detail.

## Write the func_adl query

Define a function named to describe what it selects (e.g. `get_<object>_query`), taking no arguments, and returning the query object:

```python
def get_<object>_query():
    """Performs event selection with func_adl transformer: <describe the event-level cut>.
    Also selects all columns needed further downstream for processing & histogram filling.
    """
    from servicex import query as q
    return q.FuncADL_Uproot().FromTree('<tree_name>')\
        .Where(lambda event: <event-level selection expression>).Select(
        lambda e: {
            "<output_column_1>": e.<branch_1>,
            "<output_column_2>": e.<branch_2>,
            # ... every column needed downstream ...
            # systematic variations that are simple functions of other branches
            "<systematic_variation_name>": <expression combining branches, e.g. e.branch_a * e.branch_b * 1.1>,
        }
    )
```

Guidelines:
- Use `q.FuncADL_Uproot().FromTree(<tree_name>)` for flat ROOT ntuples (this is the AGC-style workflow used here). If the input files are instead ATLAS xAOD PHYSLITE/PHYS derivations, use the `servicex` skill instead, which covers `FuncADLQueryPHYSLITE`/`FuncADLQueryPHYS` and the xAOD data model.
- The `.Where(...)` here is an event-level filter that runs before column selection; keep it to the minimal cut needed to define the final state (e.g. exact/minimum object multiplicity). Do not attempt signal/background discrimination here, and do not attempt to define multiple analysis regions here — region logic belongs downstream, in `coffea-processing`, once the data has been processed.
- `.Select(...)` must return a single flat dictionary of column names; never return a nested dictionary.
- Every column later needed by the discriminating-variable calculation, the event weighting, or the histogram filling must be listed in the output dictionary — only these columns are shipped back for downstream processing. If some columns are only needed conditionally (e.g. only when an optional downstream task like ML inference is enabled), gate the extra columns behind that same flag rather than always including them.
- Compute cheap systematic variations directly in this dictionary when they are simple algebraic combinations of other output columns (e.g. a scale factor's up/down variation), so unnecessary intermediate columns don't need to be carried downstream. More involved systematics (ones that require object-level recomputation or full reprocessing) belong in `coffea-processing`/`corrections-systematics` instead — don't try to force those into the query.

If a simpler, expression-based alternative is preferred for flat ntuples with straightforward branch names and no object-level filtering, an `UprootRaw` query can be used instead of `func_adl`:

```python
def get_<object>_query_uproot_raw():
    """Performs event selection with uproot-raw transformer: <describe the event-level cut>."""
    from servicex import query as q
    return q.UprootRaw([{'treename': '<tree_name>',
                         'expressions': ['<output_column_1>', '<output_column_2>', ...],
                         'aliases': { '<output_column>': '<branch_name_or_expression>', ... }
                        }])
```

`expressions` lists every output column name, and `aliases` maps any column whose name differs from the branch name (or that is a derived expression, such as a product of scale factors, or a boolean cut over multiple branches) to that expression string. `UprootRaw` cuts are also capable of expressing compound event-level filters (e.g. counts of objects passing per-object cuts) as a single string expression, not just simple column renames — prefer `func_adl` when the filtering logic reads more naturally as nested `.Where(...)`/lambda expressions; prefer `UprootRaw` when it's simpler to express as a flat boolean expression string.

If both approaches are implemented, decide which to use with a boolean flag defined among the earlier global settings (e.g. `USE_SERVICEX_UPROOT_RAW`), and pick the query accordingly:

```python
query = get_<object>_query_uproot_raw() if USE_SERVICEX_UPROOT_RAW else get_<object>_query()
```

## Build the bundle and deliver with ServiceX

Use the query and the file lists decided earlier to build a `ServiceX` bundle. Each dataset category (or each entry of an already-built fileset) becomes one `Sample`:

```python
import servicex

bundle = {
    'Sample': [
        {
            'Name': <name>,
            'Query': query,
            'Dataset': servicex.dataset.FileList(<that entry's file list>),
            'IgnoreLocalCache': IGNORE_CACHE,
        }
        for <name>, <entry> in <input_files_dict_or_fileset>.items()
    ],
}
```

Replace `<input_files_dict_or_fileset>` with whichever of the two forms described in "Existing analysis pipeline" applies — for a metadata-rich fileset, iterate its entries and pull each one's file list from `<entry>["files"]`. Use `servicex.dataset.FileList(...)` when datasets are given as explicit file paths/URLs; use `servicex.dataset.Rucio(<did>)` instead when a dataset is identified by a rucio DID.

Choose a delivery mode based on the pipeline's settings rather than hardcoding one:
- `'General': {'Delivery': 'LocalCache'}` downloads and caches files locally — a reasonable default for smaller runs or when files need to be re-read many times.
- `'General': {'Delivery': 'URLs'}` (or omitting `General` entirely, if that's the query framework's default) returns remote-access URLs without downloading — useful for large datasets where local disk/caching isn't desired.

If the pipeline has a flag like `USE_SERVICEX_DOWNLOAD`, use it to pick between these rather than assuming one unconditionally.

Time and execute the delivery:

```python
import time
t0 = time.time()

results = servicex.deliver(bundle)

print(f"execution took {time.time() - t0:.2f} seconds")
```

## Build or update the fileset

If no fileset existed yet, build one from `results`:
```python
fileset = {name: {"files": results[name], "metadata": {"dataset_name": name}} for name in results}
```

If a metadata-rich fileset already existed (built by a construction utility before this query stage), update its file lists in place instead of replacing the whole structure, so the existing per-sample metadata (`process`, `variation`, `xsec`, `nevts`) is preserved:
```python
for name in fileset:
    fileset[name]["files"] = results[name]
```
This is also the point to switch any downstream tree-name setting from the original tree name to whatever `ServiceX` renamed it to (commonly `"servicex"`), if the pipeline has such a setting.

## Confirm the fileset

Print the `fileset` object in its own step to confirm every dataset category resolved to a set of cached file locations:

```python
fileset
```

This `fileset` dictionary is what should be handed off next (e.g. to the `coffea-processing` skill) — its keys are the dataset/process names, and each value has `"files"` (the cached file locations returned by ServiceX) and `"metadata"` (containing at least `"dataset_name"` or `"process"`, used downstream to identify which category of events is being processed, plus `"xsec"`/`"nevts"`/`"variation"` if available).
