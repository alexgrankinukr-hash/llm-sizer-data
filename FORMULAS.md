# LLM Sizer — Methodology & Assumptions

**Version 0.7 · 2026-09-04 · Status: pre-launch draft.** This is the public, plain-language explanation of every number LLM Sizer shows. It is the single source of truth: the tool's About page renders it and the open-data repo's `FORMULAS.md` is a nightly copy of it, and **any change to a formula, constant, or data source in the tool must be reflected here first** (that rule is part of the project's definition of done). Comments and corrections: the feedback form in the tool, or an issue on the open-data repository. Changelog at the bottom.

---

## 1. What the tool answers — and what it doesn't

LLM Sizer answers three questions for a machine you own or are considering: **which open AI models fit in its memory, how fast they will write answers, and what you'd have to change to make a model fit.** Every answer is a calculation from published data, calibrated against real measurements, and every estimate is labeled as one.

It does **not** run a benchmark on your machine, judge model quality (we show quantization-quality notes where quantizers publish them, nothing more), or cover image/video generation models (those are compute-bound and need a different method).

### How to read the table

Machines are rows (one row per memory size, grouped by model line: "Mac mini · M6" covers the 16, 24 and 32 GB configurations), models are columns (one column per model at a chosen quantization and context length). Each cell is one of three marks:

- **Gold sphere: runs.** The whole configuration fits in the memory available for AI with at least 10 % headroom. The sheet behind the cell says how much of the available memory it uses.
- **Gold ring: runs with a compromise.** Something has to give, and the cell already picked the cheapest concession by our ranking (section 5): close the apps you reserved memory for, raise the macOS memory limit, shorten the context, or drop to a smaller build, in that order of preference. A cell that fits with under 10 % headroom is also a ring, labelled *tight*. The sheet lists the chosen fix, up to three alternatives (never "an earlier option plus something extra"), and applies any of them to the column with one tap.
- **Grey bar: doesn't fit.** Nothing at or above the quality floor (2-bit builds by label, adjustable in Advanced) fits even with every concession. The sheet shows the nearest miss: the smallest allowed build and how much memory it would still need.

Everything that configures the chart lives in the Settings panel (a column on the right on wide screens, a button above the chart on narrow ones): the machines and which of their memory sizes to show, the models (each column is one model at one build and one context, set when you add it and editable in the list, which you can also reorder by dragging), and two switches. **"I'll also use it for work"** reserves 16 GB for your own apps (adjustable under Advanced); it is off by default, so the default table is the best case with nothing else running. **"macOS memory limit"** shows the share of RAM macOS lets the GPU use (67 % under 36 GB, 75 % from 36 GB); the sheet's fix and Advanced can lift it to the override (section 4). Speed numbers are always for the configuration that fits: a ring's speed is the speed of its fix, not of the build you asked for. Column headers carry the model's total and active parameters ("180B · 6B active" for a mixture-of-experts model), the build (Q4 by default) and the context length (32K by default). Four tiles above the chart switch between the fit table, the speed bars for one machine, the buying view (one model across machines, cheapest first) and the memory chart, which shows what each model needs at its build and context, split into weights, context cache and buffers, with one machine's limit drawn as a reference line.

Every table state is a link (`?s=`), the share image is rendered from the same math on the server, and a custom model or machine you type in travels inside the link so the person you send it to sees exactly your table.

Every number in the sheet behind a cell links to its source: the weights to the file's repository on Hugging Face, the machine to its row in the sources table (bandwidth, prices, spec pages), the context cache and the available memory to the sections of this page that compute them, and the speed to the factor it used, with a sentence saying whether that factor was measured on this chip, assumed, or taken from the chip's tier.

## 2. Words we use

- **Model / weights** — the file you download. Its size is the single biggest number in every fit calculation.
- **Quantization (Q4, Q8, FP8, MLX 4-bit…)** — storing the weights with fewer bits per number. Q4 ≈ 4.5 bits per weight and is the common local default; Q8 ≈ 8 bits and is near-lossless; FP8/BF16 are the "full" formats. Smaller quant = smaller file = faster, with some quality loss below Q4.
- **Context window** — how much text the model can hold "in mind" at once (your prompt + its answer + any documents). Measured in tokens; 1 token ≈ ¾ of an English word.
- **Context cache (KV cache)** — memory the model uses to remember the context while it works. It grows with the context length and is the part most fit charts ignore.
- **Unified memory** — on Apple silicon (and boxes like the DGX Spark), CPU and GPU share one pool. A model must fit in the part the GPU is allowed to use.
- **Bandwidth (GB/s)** — how fast the chip can read its own memory. It decides writing speed.
- **Tokens per second (tok/s)** — how fast the answer appears. **Decode** = writing the answer; **prefill** = reading your prompt before the first word appears.
- **Active parameters** — in mixture-of-experts (MoE) models only part of the model works on each token; that part is what gets re-read per word and what sets the speed.

## 3. Where the data comes from, and how fresh it is

| Data | Source | Refresh |
|---|---|---|
| Model weight sizes per quantization | The exact file sizes published on Hugging Face by the quantizers people use (Unsloth, Bartowski, LM Studio community, mlx-community, and llama.cpp's own ggml-org) and by the model makers, read through the Hugging Face API | nightly |
| Model architecture (layers, attention type, experts, context maximum, MTP heads) | Each model's `config.json` on Hugging Face; a small hand-maintained overrides file for facts the config can't express (license thresholds, "ships 4-bit only", published active-parameter counts) | nightly |
| Machines (memory options, bandwidth, GPU cores, prices) | Apple's and NVIDIA's published specifications, with the source link per row; one row per chip bin where bandwidth differs (for example the M5 Max 32-core at 460 GB/s and 40-core at 614 GB/s) | by hand, when products change |
| Measured speeds used for calibration | The community llama.cpp Apple-silicon benchmark table (M1 → M5 Max, updated 2026-08-25), published MLX / MTP / DFlash measurements, and later community submissions through the tool | as available; each row carries its source |

**How models get in.** Every night a job lists what the five quantizer organisations above published or updated, finds the maker's original repository for each (from the quantizer's own "base model" note, otherwise by name), and reads that model's `config.json` and file list. It keeps everything from the last twelve months that has at least a billion parameters and a thousand downloads; the ~25 **featured** models (hand-picked, current releases only) are always refreshed. Repositories that are edited variants — uncensored or "abliterated" builds, mixed-precision experiments, draft models — are skipped by name and logged. A model released today is normally in the tool tomorrow, with its real file sizes.

**What can go wrong, and the safety net.** Before new data goes live it is compared with what is live now: if a featured model lost its quantizations, a known file changed size by more than a tenth, a model's architecture could suddenly not be read, or the catalog shrank, the update is held and a person looks at it. The page always shows the date of the data it is using. The full export (search index, one file per model, machines, benchmarks, factors, this document) is mirrored nightly to the public data repository.

## 4. How much memory is actually available for AI

**Macs (Apple silicon).** macOS does not let the GPU use all of the unified memory. By default it allows about **⅔ on machines under 64 GB and ¾ on 64 GB and above**. Power users can raise this limit from Terminal (`sudo sysctl iogpu.wired_limit_mb=…`) — it works and it's common, but it's unsupported by Apple, so the tool treats it as an explicit toggle, off by default.

On top of that, the machine has to run macOS itself and — if it's your everyday computer — your apps. We assume:

- **macOS base:** 6 GB, always.
- **Your apps** (the "I'll also use this machine for work" toggle): a budget of **8 GB (light) / 16 GB (typical, the default when the toggle is on) / 24 GB (heavy)** — a browser with many tabs alone is 5–7 GB. Adjustable in Advanced.

```
available = min( RAM × GPU-limit ,  RAM − 6 GB − apps )
```

The `min` matters: on large machines the ¼ that macOS holds back already covers the OS and your apps; on small machines it doesn't, and your apps become the real limit. Example: a 32 GB Mac mini used for work → `min(32 × 0.67 = 21.4, 32 − 6 − 16 = 10)` → **10 GB** for AI. That's why a model that "fits 32 GB" on a spec sheet doesn't fit a 32 GB machine you work on.

**Other platforms** — see §7.

## 5. How much memory a model needs

```
needed = weights + context cache + working buffers
```

- **Weights** = the published file size of the quantization you picked (not an estimate).
- **Working buffers** = `max(1 GB, 5 % of weights)`, capped at 16 GB — activations and the runtime's scratch space.
- **Context cache** = *bytes per token × context length*. Bytes per token depends on the model's attention design, which we read from its config:

| Attention design (how we detect it) | Cache per token, per attention layer | Typical 128K cost |
|---|---|---|
| Classic grouped-query attention (default) | `2 × kv_heads × head_dim × bytes` | tens of GB on big models |
| Partial-attention hybrids — only some layers keep a cache (`layer_types` lists linear/full layers, or `full_attention_interval`) | classic formula on the attention layers only; linear layers keep a small constant state | Qwen 3.8 27B: 16 of 64 layers → **≈ 4 GB** |
| Compressed / latent attention (MLA: `kv_lora_rank`; DeepSeek V4: one 512-wide KV head) | `(latent_width + rope_width) × bytes` | GLM-5.2 (78 layers): **≈ 6 GB**; DeepSeek V4 Flash: ≈ 3 GB; GLM-5.3 Flash: ≈ 0.7 GB |
| Sliding-window layers (`sliding_window`, Gemma-style) | classic formula, but the window (e.g. 1,024 tokens) caps the length | negligible |
| Key = Value sharing (`attention_k_eq_v`, Gemma 4) | halves the classic formula; Gemma 4's few global layers use their own wider heads (`num_global_key_value_heads × global_head_dim`) | Gemma 4 26B-A4B: 5 global layers → **≈ 0.7 GB** at 128K |

`bytes` is the cache precision: **8-bit by default** (1 byte), FP16 (2 bytes) or 4-bit (0.5) selectable in Advanced. Most 2026 models use compressed or hybrid attention, so long context is far cheaper than older fit charts assumed. Models whose design we can't read from the config are marked **"architecture assumed conventional"** and use the classic formula — the pessimistic choice.

**Verdicts:**
- **Runs** (solid dot) — `needed ≤ 90 %` of available.
- **Tight** (ring) — it fits, but with under 10 % headroom: `90 % < needed ≤ 100 %`. Real sessions grow; treat it as "will work, watch it."
- **Runs with a compromise** (ring) — the asked-for configuration doesn't fit, but a change does, and the cell says which. The tool tries, alone and in combination: closing your work apps (only when the work toggle is on), the macOS memory-limit override (only on Macs, and only when the GPU limit is what binds), a shorter context (the next chips down: 128K → 32K → 8K), a smaller quantization of the same format, and any hand-maintained special build (below). Every combination that fits gets a cost — closing apps 1 · the override 2 · each context step 4 · a smaller quant 8 (9 under 3 bits) · a special build 16 · tightness ½ — and the cheapest wins; ties go to the higher-quality (more bits) and longer-context option. Up to three genuinely different alternatives are kept for the tooltip ("or: at 32K context instead of 128K").
- **Doesn't fit** (dash) — nothing allowed fits at any context. The cell still explains itself: "the smallest allowed build, UD-IQ2_XXS (2-bit) at 238 GB, needs about 256 GB at 128K; this machine offers 250 GB at most."

**The quality floor.** The automatic search never proposes a build under **2 bits per weight, judged by the file's label** (`IQ1_*` / `Q1_*` are out; `IQ2_*`, `Q2_*`, `MLX-2bit` are in). The label is used rather than the measured bits-per-weight because dynamic quantizations keep some tensors at higher precision — a 1-bit-labelled build of a large mixture-of-experts model reads as 2.3 effective bits. When only a sub-floor build would load, the dash says so ("a 1.3-bit build (466 GB) would load, quality unknown"); Advanced lets you lower the floor and pick such builds by hand.

**Special builds.** A few community builds are not plain quant files — Qwen 3.8 Flash Next's 4-bit build keeps 45.8 GB in memory and pages its 51 GB n-gram table from SSD. Those are hand-maintained entries (resident size, source, and the measurement that justifies them) and appear in the search as their own compromise: "with the SSD-paged build (45.8 GB resident)". Their speed is the community measurement, shown as such.

## 6. Speed

### 6.1 Writing speed (decode) — the number that matters day to day
To write each token, the machine re-reads the model's active weights plus the context cache. So the ceiling is bandwidth divided by bytes read per token, and real machines reach a fraction of that ceiling:

```
bytes per token = active_params × bytes per weight (quant) + cache bytes per token × context length
decode tok/s    = bandwidth × efficiency ÷ bytes per token
```

**Efficiency** is measured, not assumed: `calibrate.py` reads every plain llama.cpp row in `benchmarks.json`, computes `measured tok/s ÷ (bandwidth ÷ weights)` per chip, and writes the result to `factors.json` (median per chip, then the median per tier as the fallback):

| Chip tier | Efficiency (tier median) | Measured chips |
|---|---|---|
| Apple base chips (M1–M5) | **0.79** | 0.76–0.83 |
| Apple Pro | **0.74** | 0.69–0.82 (M5 Pro 0.82) |
| Apple Max | **0.63** | 0.58–0.82 (M5 Max **0.82**, three rows) |
| Apple Ultra | **0.44** | 0.40–0.45 (M3 Ultra 0.44, three rows) — the multi-die design pays a consistent penalty |
| NVIDIA DGX Spark (CUDA, unified) | **0.40** | one dense-70B measurement |

A chip with its own measured rows uses its own value (so an M5 Max is 0.82, not the tier's 0.63). Chips without a measured row inherit the tier median and are labeled **"estimate — unmeasured"**: M6 and M6 Pro take the base and Pro medians. **M5 Ultra is the one explicit assumption:** the M5 generation measured about 1.27× its predecessor on the Max tier (M5 Max 0.82 vs M4 Max 0.65), so the M5 Ultra is set to the Ultra median × that lift, **0.55**, capped at 0.6, until a measured row exists. For mixture-of-experts benchmark rows the efficiency is computed on the *active* weights the machine re-reads per token, not the whole file — the M3 Ultra DeepSeek row only makes sense that way. The exact numbers, their sample sizes and ranges are in `factors.json`, which is regenerated whenever `benchmarks.json` changes. The chip name is matched without its GPU bin (`M5 Max (40-core GPU)` → `M5 Max`); a machine whose platform has no factor yet (AMD ROCm boxes) gets no speed number rather than an invented one. Because the context cache is in the formula, speed visibly falls as the context grows — more for classic-attention models, barely for hybrids.

### 6.2 Runtime: GGUF (llama.cpp / LM Studio) vs MLX
The baseline above is llama.cpp with a GGUF file. Apple's MLX runtime (MLX files, used by LM Studio's MLX engine, Ollama's MLX backend, mlx-lm) is faster on Apple silicon; the gap depends on the model:

| Model type | MLX factor | Evidence (2026 reports) |
|---|---|---|
| Dense, under ~14B | **≈ 1.0–1.2×** | roughly tied in most tests |
| Dense, 14B and up | **≈ 1.15×** (1.1–1.2) | consistent 10–20 % lead |
| Mixture-of-experts | **≈ 1.9×** (1.5–3×) | Ollama on M5 Max, Qwen3.5-35B-A3B: 58 → 112 tok/s; M4 Pro, Qwen3-Coder-30B-A3B: 43 → 130 |

Also: MLX files are 5–13 % smaller for the same nominal quant (the tool uses the actual MLX file size when you pick MLX), and MLX prefill is 30–40 % faster. **Caveat shown in the tool:** at very long contexts (~30K+ tokens) llama.cpp with flash attention can be faster than MLX — the factor applies in full up to 24K tokens and fades linearly to 1.0 at 36K, so a speed bar never jumps at a single token.

### 6.3 Acceleration: MTP and draft models (DFlash / DSpark)
Some models can write several tokens per step. Two ways: **MTP** (multi-token-prediction heads built into the model — we detect them in the config) and **draft models** (a small helper model proposes a block of tokens the big model verifies in one pass; DFlash and DSpark drafts exist for many popular models — we find them on Hugging Face daily). Output is identical; only speed changes. The tool offers these only for models that actually have them, and shows the result as a range beside the baseline, never as the headline:

| Method | Factor | Evidence |
|---|---|---|
| MTP (native heads) | **1.6–2.6×** (median 2.1) | 9B on M4 mini 14.4 → 23.0; Qwen 3.8 27B on M4 Pro 7 → 18.3 (MTPLX) |
| DFlash / DSpark draft, dense model | **2.0–3.6×** (median 2.8) | Qwen 3.8 27B 4-bit on M4 Pro 14.7 → 33.8; 8-bit 8.4 → 30.5; Gemma 4 12B 17.8 → 49.4; Qwen3-8B 13.7 → 45.8 (mlx-dspark, mlx-dflash) |
| DFlash / DSpark draft, MoE with under ~6B active | **1.1–1.3×** | Qwen3.6-35B-A3B 86.9 → 114.5 — little to gain when the model is already fast |

These ranges are computed by `calibrate.py` from the paired rows in `benchmarks.json` (each accelerated row records the unaccelerated speed of the same setup), so they move as measurements are added.

Gains depend on the task (code and math accept longer drafts than chat) and on quant (8-bit targets accept more than 4-bit).

### 6.4 Reading speed (prefill) and time to first answer — v1.5
Prefill is limited by compute, not bandwidth. We calibrate it per chip from the measured `pp512` rows of the same community table (M5 Max: 3,220 tok/s on a 7B model vs 886 on M4 Max — Apple's neural accelerators are measurable), scaled by active parameters, and show **time to first token for 8K / 32K / 128K prompts**. Chips without measured prefill rows are labeled estimates.

### 6.5 "Feels like" — the translation next to every speed

| tok/s | label |
|---|---|
| under 5 | painful — slower than reading |
| 5–12 | usable for short answers |
| 12–30 | comfortable chat |
| 30–60 | fast — agents and coding feel fine |
| over 60 | cloud-like |

## 7. Other machines: the same math, different facts

| Platform | Available memory | Speed rule |
|---|---|---|
| Mac (unified) | §4 | §6 with the Apple tier factor |
| Prebuilt unified boxes (DGX Spark, AMD Halo) | RAM − ~8 GB for the OS and runtime; no Apple-style GPU limit; work-apps budget still applies | bandwidth × platform factor (CUDA ≈ 0.55–0.65 on small models, 0.40 on large dense; ROCm ≈ 0.45) |
| Custom PC (cards × count + system RAM) — v2 | Σ VRAM − ~1 GB per card; system RAM only as slow overflow (flagged) | **slowest card's bandwidth × factor — splitting a model across cards does not add writing speed**; NVLink tensor-parallel gains (~1.3–1.6×) as an optional toggle |
| Mac clusters (Thunderbolt 5 / RDMA) — later | Σ pools − per-machine reserves | per-machine speed, not summed; clustering helps prefill more than decode |

One sentence to remember: **memory adds up across cards and boxes; speed doesn't.**

## 8. Known limitations (read before arguing with a number)

1. **Everything speed-related is an estimate** unless a measured row exists for exactly your chip, model, quant and runtime — then we show the measurement next to the estimate.
2. **Fits ≠ runs well.** A model that loads at 2-bit, or crawls at long context, is running the way a car runs in first gear. Rings tell you the compromise.
3. **Sustained load is real.** Long inference sessions run the GPU flat out; laptops get hot and may throttle. The tool doesn't model thermals.
4. **The macOS limit override** is unsupported by Apple; the tool defaults to the official behavior.
5. **Quantization quality** varies by quantizer and method (dynamic "UD" quants keep more accuracy at the same size); we show quantizers' own quality notes where published and don't invent scores.
6. **MoE active parameters** are computed from the config; makers' published figures take precedence when they differ.
7. **Models newer than the data date** aren't in the tool yet; custom entries are the workaround.
8. **"Architecture assumed conventional"** entries use the pessimistic classic cache formula until someone tags the architecture.
9. **Quick custom models** (name, parameters, file size) get an assumed design: about `32 × (parameters ÷ 8B)^0.4` layers with 8 KV heads of 128, all caching. Enter the real fields in the full form for exact context math.

## 9. Worked examples

All numbers below come from the live data (file sizes read from Hugging Face, factors from `factors.json`) and are reproduced by the engine's tests.

**A. 32 GB Mac mini (M6), Qwen 3.8 27B at Q4 (LM Studio's Q4_K_M, 16.8 GB), 32K context.** Available with the default GPU limit = min(32 × 0.67, 32 − 6 − 0) = **21.4 GB**. Needed = 16.8 (weights) + 1.15 (cache: 32 KB/token × 32K, plus the hybrid layers' constant state) + 1 (buffers) = **19.0 GB** → **runs** at 88 %. Speed ≈ 7.5 tok/s ("usable"; the M6 has no measured row yet, so it carries the estimate label). Turn the work toggle on: available = min(21.4, 32 − 6 − 16) = **10 GB** → the ring says *"runs if you close your apps"*; the tooltip's alternative is a 2-bit build that fits with the apps open, tight.

**B. 256 GB Mac Studio (M5 Ultra), GLM-5.3 Flash at Q4 (UD-Q4_K_XL, 199.7 GB), 128K context.** Available with the default GPU limit = 192 GB; needed = 199.7 + 0.8 (cache — compressed attention on 11 of 45 layers) + 10 (buffers) = **210.5 GB** → ring: *"needs the macOS memory-limit override"*; with the override, available = 250 GB → **runs** at 84 %. With the work toggle on as well (available 234 GB) it still runs, at 89.96 % — the page shows the percentage so nobody argues about a rounding. Speed: 18B active × 4.97 effective bits ≈ 11.2 GB read per token; 1,200 × 0.55 (M5 Ultra, unmeasured estimate) ÷ 11.2 ≈ **59 tok/s** at 8K and ≈ 56 tok/s at 128K; with MLX (≈ 1.9× for a mixture of experts) far more; with its MTP heads, more again. The same model on a DGX Spark: Q4 does not fit (120 GB available after the 8 GB reserve); its 2-bit build (UD-IQ2_XXS, 102 GB) runs at ≈ **17 tok/s** — our earlier video graphic said "~14 tok/s at Q4", which was neither the right quant nor, with the calibrated 0.40 efficiency, the right number.

**C. 512 GB Mac Studio, GLM-5.2 at Q4 (UD-Q4_K_XL, 467.3 GB), 128K context, override on.** Needed = 467.3 + 5.9 (cache — compressed attention) + 16 (buffers, capped) = **489.2 GB**; available = 506 GB → **tight** (97 %) — and a Reddit user measured ~473 GB in use on exactly this setup, which is the kind of confirmation that turns an estimate into a measurement. Without the override (available 384 GB) the cheapest fix is the override itself; the alternative is the 3-bit build (UD-IQ3_S, 309 GB), which runs at 86 %.

**D. 256 GB Mac Studio, GLM-5.2, 128K.** A **dash**. The smallest 2-bit build (UD-IQ2_XXS, 238.5 GB) needs about 256 GB with its cache and buffers, and the override allows 250 GB at most; the 1-bit builds are below the quality floor. The tooltip says exactly that, and notes that a 1-bit build (217 GB) would load, tight, if you lower the floor. (Our video's chart showed a ring here because it ignored buffers.)

**E. 512 GB Mac Studio, Kimi K3 (2.8T parameters).** A **dash**: the smallest 2-bit build is 711 GB. Its 1.3-bit build (466 GB) would load at 96 % with the override — quality unknown, and only if you lower the floor in Advanced.

**F. 64 GB MacBook Pro (M5 Max), Qwen 3.8 Flash Next.** Every regular 2-bit-and-up build starts at 75 GB, so none fits. The hand-maintained SSD-paged build (45.8 GB resident) fits with the override: 45.8 + 0.5 + 2.3 = 48.5 GB against 58 GB available → ring: *"with the SSD-paged build; needs the memory-limit override"*, speed shown as the community's measured 36 tok/s.

## 10. Changelog

- **0.7 — 2026-09-04.** The memory chart became "How much memory?": absolute bars of what a configuration needs (weights + context cache + buffers), machine-independent, so the same model at two builds or two context lengths can be compared; one machine's limit is a dashed reference line. No formula or constant changed.
- **0.6 — 2026-09-03.** The simple UI: all configuration moved into a settings panel (machines with tickable memory sizes, models as one-build-one-context columns, drag to reorder, the two switches, Advanced folded in); four chart tiles (fit table, speed, which machine, memory); the memory breakdown became a full chart; a Feedback button and a closable notify card replaced the forms under the table. Section 1's reading guide updated. No formula or constant changed.
- **0.5 — 2026-09-03.** The About page (`/tools/llm-sizer/about`) renders this document with a FAQ and the sources behind every machine, benchmark row and factor; every number in a cell's sheet now links to its source (section 1). Correction recipe by file (section 11). Colours inside the tool darkened for contrast (the video's palette stays on the filming canvas). Cross-check against the charts that aired in the Apple-Macs video (2026-08-29), reproduced through the tool's own table path and kept as a test: the tool disagrees in 14 of 108 cells, none from a formula error. The reasons: the chart drew "does not fit at Q4" as a dash where the tool shows the cheapest compromise as a ring (a 2-bit or 3-bit build, a shorter context, the memory-limit override, or closing your apps in the realistic chart); the chart assumed the three-quarters memory limit on a 32 GB Mac where macOS allows two thirds (Qwen 3.8 27B at Q4 + 128K needs 22.2 GB against 21.4); Qwen 3.8 Flash Next's Q4 file is 119 GB (the n-gram table is inside it), not the ≈ 85 GB the chart assumed; GLM-5.3 Flash at Q4 needs the override on a 256 GB Studio once buffers count; GLM-5.2's 2-bit build does not fit 256 GB with its cache and buffers, and on 512 GB it needs the override and lands at 97 % (tight, a ring); and the speed table's flat 0.48 efficiency is now per chip (section 6.1). No formula or constant changed.
- **0.4 — 2026-09-03.** The app. New "How to read the table" subsection (section 1): the three marks, the two switches, tight cells as rings, speed shown for the configuration that fits, links and custom entries. Fix descriptions now separate the tight note with a semicolon instead of a dash; the alternatives list never contains a superset of an earlier alternative. No formula or constant changed.
- **0.3 — 2026-09-02.** The engine. Verdicts now have four states (runs / tight / compromise / doesn't fit) with the fix search's costs written down; the 2-bit quality floor (by label) and hand-maintained special builds added; MLX factor fades 24K → 36K instead of dropping at 30K; efficiency computed on active weights for mixture-of-experts benchmark rows (Ultra tier 0.44, M5 Ultra assumption 0.55); chip names matched without their GPU bin; ROCm machines get no speed number. Worked examples rewritten from the live data: Qwen Q4 is 16.8 GB (LM Studio's file), GLM-5.3 Flash ≈ 59 tok/s at 8K on the M5 Ultra, its Q4 does not fit the DGX Spark (the 2-bit build runs ≈ 17 tok/s), GLM-5.2 on 256 GB is a dash once buffers are counted, Kimi K3 and Flash Next examples added.
- **0.2 — 2026-09-02.** Data pipeline built. Section 3 now describes how models are discovered (five quantizer organisations, twelve-month window, featured list, name-based exclusions) and the publish-time safety net. Efficiency constants are now generated by `calibrate.py` from `benchmarks.json` into `factors.json` (tier medians 0.79 / 0.74 / 0.63 / 0.45 / 0.40; measured chips use their own value, M5 Max 0.82); M5 Ultra documented as an explicit 0.57 assumption. Acceleration ranges computed from paired rows. Gemma 4 cache corrected to its global-layer heads (≈ 0.7 GB at 128K, not 1.3). Machines catalog: one row per bandwidth bin (M5 Max 460 vs 614 GB/s; M6 mini 153 GB/s at 16 GB vs 170 at 24/32 GB). Spark speed in example B corrected from ~14 to ~11 tok/s.
- **0.1 — 2026-09-02.** First draft, written from the Phase 0 research spike: efficiency by chip tier from the community llama.cpp table (M1–M5 Max), context-cache formulas by attention design read from model configs, MLX runtime factors from 2026 community reports, MTP/DFlash factors from MTPLX and mlx-dspark measurements.

## 11. How to correct us

Use the feedback form in the tool (it attaches your exact configuration) or open an issue on the open-data repository, ideally with: machine, model, quant, context, runtime, and the number you measured. Accepted measurements go into the calibration data with your handle as the source, and the relevant row above changes with a changelog entry.

File by file, in the open-data repository:

- **A machine fact** (memory options, bandwidth, price, status): edit the row in `machines.json` and add the page you are citing to its `sources`.
- **A speed you measured**: add a row to `benchmarks.json` with `chip`, `bandwidth_gbs`, `memory_gb`, `model`, `quant`, `weights_gb`, `runtime` (gguf or mlx), `accel` (none, mtp, dflash, dspark), `tg` (writing tokens per second), `pp` (prefill, optional), `source` (a public link) and `date`; accelerated rows also carry `baseline_tg`, the same setup without the acceleration. Do not edit `factors.json`: it is regenerated from the rows.
- **A model fact** the Hugging Face API cannot express (a published active-parameter count, a licence threshold, a hand-maintained build): `models.overrides.json`.
- **A formula or constant**: open an issue. `FORMULAS.md` is a nightly copy of this document and is not edited in the data repository.
