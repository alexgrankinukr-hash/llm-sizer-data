# LLM Sizer — open data & formulas

This repository holds the **data and the math** behind [LLM Sizer](https://theaibridges.com/tools/llm-sizer), a free tool from [The AI Bridge](https://theaibridges.com) that tells you which open AI models fit your machine and how fast they'll run.

Everything the tool shows traces to a file here:

| File | What it is | How it's maintained |
|---|---|---|
| `FORMULAS.md` | Every formula, constant, assumption and limitation, in plain language, with worked examples and a changelog | Hand-written; updated whenever the tool's math changes (it's part of the definition of done) |
| `machines.json` | Machines the tool knows: memory options, bandwidth, GPU cores, prices, sources | Hand-curated; Phase 1 scaffold (3 rows) — the full table lands with the launch |
| `benchmarks.json` | Measured tokens/sec used to calibrate the speed estimates, one row per source | Seeded from community measurements; grows with accepted submissions |
| `models/` | Per-model records (quantization sizes, architecture, acceleration flags) exported nightly from the Hugging Face API | Automated from launch onward |

## Correcting a number

Open an issue or a pull request here with: machine, model, quantization, context length, runtime (GGUF/MLX), and the number you measured or the source you're citing. Accepted corrections change the file and get a changelog entry in `FORMULAS.md` with your handle as the source. Or use the feedback form in the tool — it attaches your exact configuration.

## Licenses

- Data (`*.json`): [CC BY 4.0](LICENSE-DATA.md)
- Formulas and any scripts: [MIT](LICENSE-CODE.md)
