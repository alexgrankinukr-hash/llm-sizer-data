# LLM Sizer — open data & formulas

This repository holds the **data and the math** behind [LLM Sizer](https://theaibridges.com/tools/llm-sizer), a free tool from [The AI Bridge](https://theaibridges.com) that tells you which open AI models fit your machine and how fast they'll run.

Everything the tool shows traces to a file here:

| File | What it is | How it's maintained |
|---|---|---|
| `FORMULAS.md` | Every formula, constant, assumption and limitation, in plain language, with worked examples and a changelog | Hand-written; updated whenever the tool's math changes (it's part of the definition of done) |
| `machines.json` | Machines the tool knows: memory options, bandwidth, GPU cores, prices, sources; one row per chip bin where bandwidth differs | Hand-curated from Apple's and NVIDIA's spec pages; updated when products change |
| `benchmarks.json` | Measured tokens/sec used to calibrate the speed estimates, one row per source; accelerated rows carry the unaccelerated baseline | Seeded from community measurements; grows with accepted submissions |
| `factors.json` | The efficiency, runtime and acceleration factors the speed formula uses, with sample sizes and ranges | Generated from `benchmarks.json` by the calibration script; never edited by hand |
| `models.overrides.json` | The featured list and the few hand facts the Hugging Face API cannot express (license thresholds, "ships 4-bit only", published active-parameter counts) | Hand-maintained, reviewed on every change |
| `models/index.json` | The search index: one lean row per model (sizes at the reference 4-bit and 8-bit quants, architecture summary, acceleration flags, license) | Exported nightly from the tool's database, which is filled from the Hugging Face API |
| `models/<id>.json` | One full record per model: every quantization with its exact file size and source repository, the architecture fields behind the context-cache formula, draft-model availability, notes | Exported nightly; the `sha` field changes when the content does |

Mirror cadence: the tool's nightly job pushes every file above after its own sanity checks pass, so this repository's commit history is the change log of the data.

## Correcting a number

Open an issue or a pull request here with: machine, model, quantization, context length, runtime (GGUF/MLX), and the number you measured or the source you're citing. Accepted corrections change the file and get a changelog entry in `FORMULAS.md` with your handle as the source. Or use the feedback form in the tool — it attaches your exact configuration. The tool's [About page](https://theaibridges.com/tools/llm-sizer/about) shows every row of these files next to the numbers they produce.

File by file:

- **A machine fact** (memory options, bandwidth, price, status): edit the row in `machines.json` and add the page you are citing to its `sources` list.
- **A speed you measured**: add a row to `benchmarks.json` with `chip`, `bandwidth_gbs`, `memory_gb`, `model`, `quant`, `weights_gb`, `runtime` (`gguf` or `mlx`), `accel` (`none`, `mtp`, `dflash`, `dspark`), `tg` (writing tokens per second), `pp` (prefill tokens per second, optional), `source` (a public link) and `date`. Accelerated rows also carry `baseline_tg`, the same setup without the acceleration. Do not edit `factors.json`: the calibration script regenerates it from the rows.
- **A model fact** the Hugging Face API cannot express (a published active-parameter count, a licence threshold, a hand-maintained build): `models.overrides.json`.
- **A formula or constant**: open an issue. `FORMULAS.md` is mirrored nightly from the tool and is not edited here.

The nightly mirror rewrites every data file and `FORMULAS.md`; it never touches this README.

## Licenses

- Data (`*.json`): [CC BY 4.0](LICENSE-DATA.md)
- Formulas and any scripts: [MIT](LICENSE-CODE.md)
