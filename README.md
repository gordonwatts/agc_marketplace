# AGC Skills Marketplace

`agc_skills` is a Codex plugin marketplace for reusable Analysis Grand
Challenge (AGC) workflow guidance. It currently publishes the `agc-skills`
plugin.

## Included skills

- `dataset-retrieval` — retrieve and cache datasets with func_adl and ServiceX
- `coffea-processing` — process events and fill analysis histograms with coffea
- `corrections-systematics` — apply corrections and model systematic variations
- `hist-plotter` — plot and serialize histograms for statistical analysis
- `cabinetry-analysis` — build, fit, visualize, and cross-check cabinetry models

These skills were migrated from `Agc_claude/.claude/skills` in the
`agc_with_llms` repository. The migration preserves their domain guidance and
replaces the one Claude-specific output-directory reference with
workspace-neutral wording.

## Install from a local checkout

From this repository's root:

```shell
codex plugin marketplace add .
codex plugin add agc-skills@agc_skills
```

## Install from GitHub

After publishing this repository as `agc_marketplace`:

```shell
codex plugin marketplace add OWNER/agc_marketplace
codex plugin add agc-skills@agc_skills
```

Replace `OWNER` with the GitHub account or organization that owns the
repository.

## Repository layout

```text
.
|-- .agents/plugins/marketplace.json
|-- plugins/agc-skills/
|   |-- .codex-plugin/plugin.json
|   `-- skills/
`-- README.md
```

The marketplace identifier is `agc_skills`; the installable plugin identifier
is `agc-skills`.

## Validate

The files are compatible with the Codex plugin and skill validators. If the
Codex skill-authoring tools are installed locally, validate the plugin and each
skill before publishing.

## Work Copied From

This is based on the [work done by Matthieu Foresi](https://github.com/eigenforesi/agc_with_llms).
