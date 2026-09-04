# LLM Sizer — Methodology & Assumptions

**Version 0.9 · 2026-09-05 · Status: pre-launch draft.** This is the public, plain-language explanation of every number LLM Sizer shows. It is the single source of truth: the tool's About page renders it and the open-data repo's `FORMULAS.md` is a nightly copy of it, and **any change to a formula, constant, or data source in the tool must be reflected here first** (that rule is part of the project's definition of done). Comments and corrections: the feedback form in the tool, or an issue on the open-data repository. Changelog at the bottom.

---

## 1. What the tool answers — and what it doesn't

LLM Sizer answers three questions for a machine you own or are considering: **which open AI models fit in its memory, how fast they will write answers, and what you'd have to change to make a model fit.** Every answer is a calculation from published data, calibrated against real measurements, and every estimate is labeled as one.

It does **not** run a benchmark on your machine, judge model quality (we show quantization-quality notes where quantizers publish them, nothing more), or cover image/video generation models (those are compute-bound and need a different method).

### How to read the table

Machines are rows (one row per memory size, grouped by model line: "Mac mini · M6" covers the 16, 24 and 32 GB configurations), models are columns (one column per model at a chosen quantization and context length). Each cell is one of three marks:

- **Gold sphere: runs.** The whole configuration fits in the memory available for AI with at least 10 % headroom. The sheet behind the cell says how much of the available memory it uses.
- **Gold ring: runs with a compromise.** Something has to give, and the cell already picked the cheapest concession by our ranking (section 5): close the apps you reserved memory for, raise the macOS memory limit, shorten the context, or drop to a smaller build, in that order of preference. A cell that fits with under 10 % headroom is also a ring, labelled *tight*. The sheet lists the chosen fix, up to three alternatives (never "an earlier option plus something extra"), and applies any of them to the column with one tap.
- **Grey bar: doesn't fit.** Nothing at or above the quality floor (2-bit builds by label, adjustable in Advanced) fits even with every concession. The sheet shows the nearest miss: the smallest allowed build and how much memory it would still need.

Everything that configures the chart lives in the Settings panel (a column on the right on wide screens, a button above the chart on narrow ones): the machines and which of their memory sizes to show, the models (each column is one model at one build and one context, set when you add it and editable in the list, which you can also reorder by dragging), and two switches. **"I'll also use it for work"** reserves 16 GB for your own apps (adjustable under Advanced); it is off by default, so the default table is the best case with nothing else running. **"macOS memory limit"** shows the share of RAM macOS lets the GPU use (67 % under 36 GB, 75 % from 36 GB); the sheet's fix and Advanced can lift it to the override (section 4). The context cache is counted at FP16 by default, as every runtime stores it unless told otherwise (section 5). Speed numbers are always for the configuration that fits: a ring's speed is the speed of its fix, not of the build you asked for. Column headers carry the model's total and active parameters ("180B · 6B active" for a mixture-of-experts model), the build (Q4 by default) and the context length (32K by default). Four tiles above the chart switch between the fit table, the speed bars for one machine, the buying view (one model across machines, cheapest first) and the memory chart, which shows what each model needs at its build and context, split into weights, context cache, buffers and, for MLX files at long context, prompt scratch, with one machine's limit drawn as a reference line.

Every table state is a link (`?s=`), the share image is rendered from the same math on the server, and a custom model or machine you type in travels inside the link so the person you send it to sees exactly your table.

Every number in the sheet behind a cell links to its source: the weights to the file's repository on Hugging Face, the machine to its row in the sources table (bandwidth, prices, spec pages), the context cache and the available memory to the sections of this page that compute them, and the speed to the chip profile it used (effective bandwidth and fixed cost per token), with a sentence saying whether that profile was fitted from this chip's own rows, assumed, or taken from the chip's tier.

### What kind of number is this

Every figure the tool shows is one of five kinds, and the sheet behind a cell says which:

- **Exact file data.** Weight sizes are the bytes Hugging Face reports for the file you would download; machine memory and bandwidth are the makers' specifications. Nothing is estimated.
- **Derived from the architecture.** The context cache is computed from the model's `config.json` (attention design, layers, heads) and the context length: exact for a design we can read, the pessimistic classic formula for one we cannot ("architecture assumed conventional").
- **Fitted to measurements.** Speed. Each chip's effective bandwidth and fixed cost per token are fitted to the community llama.cpp table; the architecture and runtime terms are fitted to measured runs and judged on runs the fit never saw (section 6.1). Every fitted term carries the number of rows it rests on, and a term with too few rows is labelled assumed.
- **Measured.** A hand-maintained build's speed on the machine class it was measured on.
- **Policy margins.** The reserves for macOS and your apps, the working buffers and the MLX prompt scratch are safety margins chosen from owners' observations, not measurements of your machine (sections 4 and 5). They are the numbers most worth arguing with, and section 8 says how far off they have been seen to be.

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
| Measured speeds used for calibration | The community llama.cpp Apple-silicon benchmark table (M1 → M5 Max, updated 2026-08-25), the llama.cpp maintainers' DGX Spark benchmark (build b7941, February 2026), published MLX / MTP / DFlash measurements, and later community submissions through the tool | as available; each row carries its source |

**How models get in.** Every night a job lists what the five quantizer organisations above published or updated, finds the maker's original repository for each (from the quantizer's own "base model" note, otherwise by name), and reads that model's `config.json` and file list. It keeps everything from the last twelve months that has at least a billion parameters and a thousand downloads; the ~25 **featured** models (hand-picked, current releases only) are always refreshed. Repositories that are edited variants — uncensored or "abliterated" builds, mixed-precision experiments, draft models — are skipped by name and logged. A model released today is normally in the tool tomorrow, with its real file sizes.

**What can go wrong, and the safety net.** Before new data goes live it is compared with what is live now: if a featured model lost its quantizations, a known file changed size by more than a tenth, a model's architecture could suddenly not be read, or the catalog shrank, the update is held and a person looks at it. The page always shows the date of the data it is using. The full export (search index, one file per model, machines, benchmarks, factors, this document) is mirrored nightly to the public data repository.

## 4. How much memory is actually available for AI

**Units first.** Memory sizes are quoted in binary gigabytes: a "32 GB" Mac has 32 GiB, which is **34.4 GB** in the decimal gigabytes file sizes use (× 1.0737). The tool converts the machine's size before subtracting anything, so a 48 GB MacBook Pro is 51.5 GB, a 128 GB machine 137.4 GB, a 512 GB Studio 549.8 GB. The DGX Spark's 128 GB is binary too.

**Macs (Apple silicon).** macOS does not let the GPU use all of the unified memory. By default it allows **⅔ on machines under 36 GB and ¾ from 36 GB**: Metal's recommended working set is logged at two thirds on 16 and 32 GB machines (10,922 and 21,845 MiB) and three quarters on 36, 48, 64 and 128 GB machines (28,991 / 38,655 / 49,152 / 98,304 MiB). Power users can raise this limit from Terminal (`sudo sysctl iogpu.wired_limit_mb=…`) — it works and it's common, but it's unsupported by Apple, so the tool treats it as an explicit toggle, off by default.

On top of that, the machine has to run macOS itself and — if it's your everyday computer — your apps. We assume:

- **macOS base:** 6 GB with the default GPU limit. Once the limit is lifted, macOS is the only reserve left and on big machines it keeps more: the larger of 6 GB and **5.5 % of the memory** (a 512 GB Studio keeps about 30 GB, which is what owners running with the override report). A policy margin.
- **Your apps** (the "I'll also use this machine for work" toggle): a budget of **8 GB (light) / 16 GB (typical, the default when the toggle is on) / 24 GB (heavy)** — a browser with many tabs alone is 5–7 GB. Adjustable in Advanced.

```
available = min( RAM × GPU-limit ,  RAM − 6 GB − apps )          default limit; RAM in decimal GB (nominal × 1.0737)
available = RAM − max( 6 GB , 5.5 % × RAM ) − apps                with the override
```

The `min` matters: on large machines the ¼ that macOS holds back already covers the OS and your apps; on small machines it doesn't, and your apps become the real limit. Example: a 32 GB Mac mini (34.4 GB) used for work → `min(34.4 × ⅔ = 22.9, 34.4 − 6 − 16 = 12.4)` → **12.4 GB** for AI; with nothing else running, **22.9 GB**. That's why a model that "fits 32 GB" on a spec sheet doesn't fit a 32 GB machine you work on. Available memory by size, nothing else running: 16 GB → 11.2 · 24 → 17.2 · 32 → 22.9 · 36 → 29.0 · 48 → 38.7 · 64 → 51.5 · 128 → 103.1 · 256 → 206.2 · 512 → 412.3. With the override: 32 → 28.4 · 64 → 62.7 · 128 → 129.9 · 256 → 259.8 · 512 → 519.5.

**Other platforms** — see §7.

## 5. How much memory a model needs

```
needed = weights + context cache + working buffers + prompt scratch (MLX files above 8K of context)
```

- **Weights** = the published file size of the quantization you picked (not an estimate).
- **Working buffers** = `1.5 GB + 1 % of weights` — activations and the runtime's scratch space. A policy margin set from owners' peak readings: about 1.1 GB above the file on a 17 GB dense model, about 0.4 GB above weights and cache on a 224 GB mixture-of-experts file (the earlier `5 % of weights` rule charged that machine 11 GB).
- **Prompt scratch** (MLX files only) = `0.15 GB per 1K tokens of context beyond 8K`. MLX allocates working memory to process a long prompt in one pass, and owners' peak readings grow at about that rate on a 27B and a 397B model between 8K and 128K (3.6 GB at 32K, 18 GB at 128K); GGUF runtimes process the prompt in batches inside the buffers. A policy margin; it is its own segment in the memory bar and follows the file that runs, so a GGUF fix on an MLX column drops it.
- **Context cache** = *bytes per token × context length*. Bytes per token depends on the model's attention design, which we read from its config:

| Attention design (how we detect it) | Cache per token, per attention layer | Typical 128K cost |
|---|---|---|
| Classic grouped-query attention (default) | `2 × kv_heads × head_dim × bytes` | tens of GB on big models |
| Partial-attention hybrids — only some layers keep a cache (`layer_types` lists linear/full layers, or `full_attention_interval`) | classic formula on the attention layers only; linear layers keep a small constant state | Qwen 3.8 27B: 16 of 64 layers → **≈ 8.7 GB** at FP16 (4.4 GB with an 8-bit cache) |
| Compressed / latent attention (MLA: `kv_lora_rank`; DeepSeek V4: one 512-wide KV head) | `(latent_width + rope_width) × bytes` | GLM-5.2 (78 layers): **≈ 11.8 GB**; DeepSeek V4 Flash: ≈ 6.5 GB; GLM-5.3 Flash: ≈ 1.5 GB (all FP16) |
| Sliding-window layers (`sliding_window`, Gemma-style) | classic formula, but the window (e.g. 1,024 tokens) caps the length | negligible |
| Key = Value sharing (`attention_k_eq_v`, Gemma 4) | no saving: one projection produces keys and values, but the runtime stores both; Gemma 4's few global layers use their own wider heads (`num_global_key_value_heads × global_head_dim`) | Gemma 4 26B-A4B: 5 global layers → **≈ 2.9 GB** at 128K (1.4 GB with an 8-bit cache) |

`bytes` is the cache precision: **FP16 by default** (2 bytes), which is what llama.cpp, LM Studio, Ollama and MLX store unless told otherwise; 8-bit (1 byte) or 4-bit (0.5) selectable in Advanced. Most 2026 models use compressed or hybrid attention, so long context is far cheaper than older fit charts assumed. Models whose design we can't read from the config are marked **"architecture assumed conventional"** and use the classic formula — the pessimistic choice.

**Verdicts:**
- **Runs** (solid dot) — `needed ≤ 90 %` of available.
- **Tight** (ring) — it fits, but with under 10 % headroom: `90 % < needed ≤ 100 %`. Real sessions grow; treat it as "will work, watch it."
- **Runs with a compromise** (ring) — the asked-for configuration doesn't fit, but a change does, and the cell says which. The tool tries, alone and in combination: closing your work apps (only when the work toggle is on), the macOS memory-limit override (only on Macs, and only when the GPU limit is what binds), a shorter context (the next chips down: 128K → 32K → 8K), a smaller quantization of the same format, and any hand-maintained special build (below). Every combination that fits gets a cost — closing apps 1 · the override 2 · each context step 4 · a smaller quant 8 (9 under 3 bits) · a special build 16 · tightness ½ — and the cheapest wins; ties go to the higher-quality (more bits) and longer-context option. Up to three genuinely different alternatives are kept for the tooltip ("or: at 32K context instead of 128K").
- **Doesn't fit** (dash) — nothing allowed fits at any context. The cell still explains itself: "the smallest allowed build, UD-IQ2_XXS (2-bit) at 711 GB, needs about 724 GB at 128K; this machine offers 520 GB at most."

**The quality floor.** The automatic search never proposes a build under **2 bits per weight, judged by the file's label** (`IQ1_*` / `Q1_*` are out; `IQ2_*`, `Q2_*`, `MLX-2bit` are in). The label is used rather than the measured bits-per-weight because dynamic quantizations keep some tensors at higher precision — a 1-bit-labelled build of a large mixture-of-experts model reads as 2.3 effective bits. When only a sub-floor build would load, the dash says so ("a 1.3-bit build (466 GB) would load, quality unknown"); Advanced lets you lower the floor and pick such builds by hand.

**Special builds.** A few community builds are not plain quant files — Qwen 3.8 Flash Next's 4-bit build keeps 45.8 GB in memory and pages its 51 GB n-gram table from SSD. Those are hand-maintained entries (resident size, source, and the measurement that justifies them) and appear in the search as their own compromise: "with the SSD-paged build (45.8 GB resident)". Their speed is the community measurement, shown only on the platform and memory size it was taken on (a 64 GB MacBook Pro); on other machines the build has no speed figure rather than a borrowed one.

## 6. Speed

### 6.1 Writing speed (decode) — the number that matters day to day
To write each token, the machine re-reads the model's active weights plus the context cache, and then does a fixed amount of work that does not shrink with a smaller file: launching the kernels of every layer, routing between experts, attending over the context. The time per token is the sum of the two, and the speed is its inverse:

```
GB per token = active_params × bytes per weight (quant) + cache bytes per token × context length
ms per token = read + fixed + context
    read     = GB per token ÷ effective bandwidth × 1000 × read factor   (K-quant GGUF files on Apple silicon 1.25, everything else 1)
    fixed    = (chip fixed cost + architecture cost) × MLX factor         (GGUF files 1; section 6.2)
    context  = (attention cost + MLX attention cost) × context ÷ 32K
tok/s        = 1000 ÷ ms per token
```

Until version 0.8 the tool used `bandwidth × efficiency ÷ GB per token`, one efficiency factor per chip. That factor was calibrated on a 3.8 GB read (Llama-2-7B at Q4_0) and carried the fixed cost inside it, so it was wrong in both directions away from that size: models with a small active set (a 2 to 4 GB read per token) came out 1.5 to 3 times too fast, and 8-bit dense models on the Ultra chips 0.55 to 0.85 times too slow. The fixed-cost model separates the two.

**Chip profiles** are fitted, not assumed. For every chip, `calibrate.py` takes the F16, Q8_0 and Q4_0 rows of the community llama.cpp table (13.5, 7.2 and 3.8 GB reads of the same 7B model) and fits a straight line of milliseconds per token against gigabytes read: the slope is the effective bandwidth, the intercept the chip's fixed cost. The effective bandwidth is stored as a share of the chip's specification, because that share carries across the GPU bins of one chip (an M4 Max 32-core at 410 GB/s fits 0.88, the 40-core at 546 GB/s 0.90; an M3 Max 30-core 0.96 against the 40-core's 0.98) while the GB/s does not.

| Chip | Effective bandwidth | Share of spec | Fixed cost | Rows |
|---|---:|---:|---:|---:|
| M1 | 61 GB/s | 0.90 | 8.4 ms | 2 |
| M1 Pro | 190 GB/s | 0.95 | 7.2 ms | 3 |
| M1 Max | 363 GB/s | 0.91 | 6.4 ms | 6 |
| M1 Ultra | 639 GB/s | 0.80 | 5.8 ms | 3 |
| M2 | 94 GB/s | 0.94 | 5.3 ms | 3 |
| M2 Pro | 191 GB/s | 0.95 | 5.8 ms | 3 |
| M2 Max | 381 GB/s | 0.95 | 5.1 ms | 3 |
| M2 Ultra | 700 GB/s | 0.88 | 5.0 ms | 3 |
| M3 | 98 GB/s | 0.98 | 8.2 ms | 2 |
| M3 Pro | 142 GB/s | 0.94 | 5.9 ms | 3 |
| M3 Max | 390 GB/s | 0.98 | 5.2 ms | 3 |
| M3 Ultra | 677 GB/s | 0.83 | 5.2 ms | 3 |
| M4 | 104 GB/s | 0.87 | 4.9 ms | 3 |
| M4 Pro | 251 GB/s | 0.92 | 4.3 ms | 3 |
| M4 Max | 494 GB/s | 0.90 | 4.2 ms | 3 |
| M5 | 148 GB/s | 0.97 | 5.8 ms | 2 |
| M5 Pro | 309 GB/s | 1.01 | 2.6 ms | 3 |
| M5 Max | 516 GB/s | 0.84 | 0.5 ms | 3 |
| NVIDIA DGX Spark | 258 GB/s | 0.94 | 2.6 ms | 2 |

Two-row chips (M1, M3, M5, the Spark) have a line through two points and no slack; the rest fit within 2 % except the M1 Max (8 %, six rows from several machines) and the M5 Max (6 %). The Ultra chips read at 80 to 88 % of their specification: the multi-die design pays a consistent price. Tier medians stand in for chips without rows and are labelled **"estimate — unmeasured"**: base 0.94 / 5.8 ms, Pro 0.95 / 5.8 ms, Max 0.91 / 5.1 ms, Ultra 0.83 / 5.2 ms; M6 and M6 Pro take the base and Pro medians. **The M5 Ultra is the one explicit assumption:** 85 % of its 1,200 GB/s (the Ultra tier's share) and 1.5 ms of fixed cost (between the M5 Pro's 2.6 and the M5 Max's 0.5), until a measured row exists. The chip name is matched without its GPU bin (`M5 Max (40-core GPU)` → `M5 Max`); a platform without a profile (AMD ROCm boxes) gets no speed number rather than an invented one.

**Architecture cost** is the extra fixed time a model's design adds per token, on top of the chip's own. The class comes from the config: a dense model adds nothing; a mixture of experts adds its routing; one with linear-attention layers (the Qwen 3.5, 3.6 and 3.8 mixtures, Nemotron 3) adds those layers' recurrent state; one with compressed (latent) attention (GLM-5.x, Kimi, DeepSeek-R1) adds the decompression; DeepSeek V4 is its own class, set by hand in the overrides file, because its FP4 experts and 512-wide attention measure far above the others. Each cost is the median over the measured GGUF rows of its class at short context, after the read and the chip's fixed cost are subtracted:

| Class | Cost per token | Rows | Range of the rows |
|---|---:|---:|---|
| dense | 0 ms | by definition | |
| mixture of experts | 1.7 ms | 16 | 0.0 – 4.4 ms |
| mixture of experts with linear-attention layers | 5.8 ms | 2 | 5.2 – 6.4 ms |
| mixture of experts with latent attention | 11.7 ms | 2 | 11.0 – 12.4 ms |
| DeepSeek V4 | 17.0 ms | 3 | 13.3 – 32.7 ms |

**Read factor.** On Apple silicon, GGUF K-quant and IQ files (`Q4_K_M`, `UD-Q4_K_XL`, `IQ2_M` …) read about 25 % slower than the plain `Q4_0`, `Q8_0`, `F16` and MXFP4 formats, because their dequantization is heavier on Metal. One dense row measures it (1.25, kept as an assumption until more rows exist), and it applies only on Apple silicon: no CUDA row measures it, so the Spark reads K-quants at the plain rate. MLX files read at the plain rate.

**Context cost.** Beyond the cache re-read (already in GB per token), attending over a long context costs time that does not show up in the bytes: 5 ms per token per 32K of context for latent-attention models (an assumption; see section 8 on the rows that could have fitted it), nothing for grouped-query attention, and 6.4 ms per 32K on MLX (fitted on 14 rows of two MLX sweeps to 128K, section 6.2). Because the cache is also in the read, speed visibly falls as the context grows: more for classic-attention models, less for hybrids.

**How the fit is judged.** The terms above are fitted on the training split of `validation.json` (73 measured runs from the llama.cpp maintainers and eleven community contributors, each row with its source, grade and context length) by coordinate descent on medians, and judged on a held-out split the fit never saw: every Gemma 4 row and every row from the omlx benchmark site, split by submission so that a contributor's sweep is never half in each. On the held-out rows the median error is a factor of **1.14** (90th percentile 1.29); the multiplier model scored 1.21 (90th percentile 2.66) on the same rows. On the training rows: 1.06 against 1.33. The release rule was a held-out median under 1.25 and no measured row getting worse than the multiplier by more than 1.30; both held. `factors.json` carries every term with its row count, range and source, and `calibrate.py --report` regenerates the row-by-row comparison. Chips whose profile is a tier median are flagged in the sheet.

### 6.2 Runtime: GGUF (llama.cpp / LM Studio) vs MLX
The baseline above is llama.cpp with a GGUF file. Apple's MLX runtime (MLX files, used by LM Studio's MLX engine, Ollama's MLX backend, mlx-lm) changes the **fixed cost** per token, not the read: the measured gains sit where the fixed cost dominates (a 3B-active mixture of experts on an M4 Max: 71 → 130 tok/s) and vanish where the read dominates (8-bit files and big dense models: 0.96 to 1.2×), and on the M1 generation MLX is slower on mixtures (0.8×). So the tool multiplies the fixed cost by a factor per model group, fitted on the MLX rows of the training split:

| Model type on MLX | Fixed-cost factor | Rows | Evidence |
|---|---:|---:|---|
| Dense | 0.99 | 5 | Qwen 27B and Qwen 32B on the M1 Ultra, M3 Ultra and M5 Max: MLX 4-bit lands where a plain Q4_0 would; its lead over Q4_K_M is the K-quant read factor, not an MLX bonus |
| Mixture of experts under 6B active, M4 and newer | 0.5 | 1, assumed | Qwen3.5-35B-A3B on the M4 Max: 71 → 130 tok/s |
| Mixture of experts under 6B active, M1 to M3 | 1.2 | 3 | Qwen3-30B-A3B on the M1 Ultra 83 → 68; gpt-oss-120b and Qwen3-Coder-Next on the M3 Ultra |
| Other mixtures of experts | 0.77 | 10 | Qwen3.5-397B, Mixtral, Nemotron 3, GLM-5.2, Kimi K2.6 and DeepSeek V4 Flash on the M3 Ultra and M5 Max |

Also: MLX files are 5–13 % smaller for the same nominal quant (the tool uses the actual MLX file size when you pick MLX), and MLX prefill is 30–40 % faster. **Long context:** MLX pays about 6.4 ms per token per 32K of context on top of the cache re-read (fitted on the Qwen 32B and Qwen3.5-397B sweeps to 128K), which is why llama.cpp with flash attention catches up and passes it somewhere past 30K tokens; the crossover falls out of the two terms instead of the hard-coded 24K–36K fade of earlier versions.

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
| Mac (unified) | §4 | §6 with the chip's own profile, or its tier's |
| Prebuilt unified boxes (DGX Spark, AMD Halo) | RAM (binary, converted) − ~8 GB for the OS and runtime; no Apple-style GPU limit; work-apps budget still applies | the same fixed-cost model with the box's own profile: the DGX Spark reads at 258 GB/s (94 % of its 273) with 2.6 ms of fixed cost, from two dense llama.cpp maintainer rows; the K-quant read factor is not applied on CUDA (unmeasured); ROCm has no profile, so no number is shown yet |
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
10. **Two-point chip profiles.** The M1, M3, M5 and the DGX Spark have only two dense rows, so their line has no slack and no error estimate; the Spark's rows are the maintainers' own, the Macs' come from one community table.
11. **Thin classes and one-row terms.** The linear-attention and latent-attention mixture costs rest on two rows each, DeepSeek V4 on three (range 13 to 33 ms); the K-quant read factor and the MLX factor for small mixtures on M4 and newer rest on one row each and are labelled assumed; the latent-attention context cost is an assumption. These are the terms a single new measurement can move.
12. **The held-out rows cover no latent-attention mixture and no mixture on the Spark**, so the 1.14 median is a statement about dense models, Gemma's mixtures and the Qwen and Nemotron hybrids on Apple silicon. Every row came from a public post; none was measured under controlled conditions.
13. **Rows the model gets wrong by 2× or more.** MiniMax-M2.1 at 30K and 146K of context (mlx-lm issue 763) runs at half the model's prediction on both runtimes; Kimi K2.6 at 10K and 20K on a 512 GB Studio with a 440 GB build runs at a third and a fifth. Both look like memory pressure and are kept as report-only rows; a long-context term that catches them needs cleaner measurements.
14. **Memory margins are policy, and their residuals are known.** Against owners' peak readings the tool over-estimates by 0.5 to 1.9 GB on 17 to 30 GB dense GGUF files, by 3.7 GB on a 27B MLX file at 64K (the scratch term grows faster than that sweep did), and under-estimates a 31B GGUF file at 37K by 2.4 GB. Read a tight verdict with that in mind.

## 9. Worked examples

All numbers below come from the live data (file sizes read from Hugging Face, factors from `factors.json`) and are reproduced by the engine's tests.

**A. 32 GB Mac mini (M6), Qwen 3.8 27B at Q4 (LM Studio's Q4_K_M, 16.8 GB), 32K context.** The machine has 34.4 GB in file-size units. Available with the default GPU limit = min(34.4 × ⅔, 34.4 − 6 − 0) = **22.9 GB**. Needed = 16.8 (weights) + 2.2 (cache: 64 KB/token at FP16 × 32K, plus the hybrid layers' constant state) + 1.7 (buffers: 1.5 GB + 1 %) = **20.7 GB** → **tight** at 90 %, a ring. Speed ≈ 6.5 tok/s ("usable"; the M6 has no measured row yet, so it uses the base-tier profile and carries the estimate label). Turn the work toggle on: available = min(22.9, 34.4 − 6 − 16) = **12.4 GB** → the ring says *"runs if you close your apps; tight"*; the tooltip's alternative is a 2-bit build that fits with the apps open.

**B. 256 GB Mac Studio (M5 Ultra), GLM-5.3 Flash at Q4 (UD-Q4_K_XL, 199.7 GB), 128K context.** Available with the default GPU limit = ¾ of 274.9 GB = **206.2 GB**; needed = 199.7 + 1.5 (cache — compressed attention on 11 of 45 layers, FP16) + 3.5 (buffers) = **204.8 GB** → **tight** at 99 %, a ring. With the override, available = 274.9 − max(6, 15.1) = 259.8 GB → runs at 79 %; with the work toggle on as well (243.8 GB) it runs at 84 %. Speed on the M5 Ultra (assumed profile: 1,020 GB/s effective, 1.5 ms): 18B active × 4.97 effective bits ≈ 11.2 GB read per token at 8K → 13.8 ms read + 1.5 ms chip + 11.7 ms latent-attention mixture + 1.3 ms context ≈ 28 ms → **≈ 35 tok/s** at 8K, ≈ 31 at 32K and ≈ 21 at 128K; with MLX the fixed part shrinks to 0.77× (≈ 39 tok/s at 8K); with its MTP heads, more again. The same model on a DGX Spark: 129.4 GB available after the 8 GB reserve; Q4 does not fit; the 3-bit build (UD-IQ3_XXS, 120.4 GB) fits tight at 96 %, at ≈ **15 tok/s** (the Spark's 258 GB/s effective and 2.6 ms, plus the same 11.7 ms architecture cost), and the 2-bit build (UD-Q2_K_XL, 108.7 GB) is the alternative at ≈ 16 tok/s.

**C. 512 GB Mac Studio, GLM-5.2 at Q4 (UD-Q4_K_XL, 467.3 GB), 128K context, override on.** Needed = 467.3 + 11.8 (cache — compressed attention, FP16) + 6.2 (buffers: 1.5 GB + 1 % of 467) = **485.2 GB**; available = 549.8 − max(6, 30.2) = 519.5 GB → **tight** (93 %). Without the override (available 412.3 GB) the cheapest fix is the override itself; the alternative is the 3-bit build (UD-IQ3_S, 309 GB), which runs comfortably.

**D. 256 GB Mac Studio, GLM-5.2, 128K.** A **ring**: with the override (259.8 GB available) the 2-bit build UD-IQ2_M (238.6 GB) fits with its cache and buffers (254.2 GB) at 98 %, tight; the alternatives are the smaller 2-bit UD-IQ2_XXS, also tight, and a 32K context. Without the override nothing fits, so the fix is the override plus the 2-bit build.

**E. 512 GB Mac Studio, Kimi K3 (2.8T parameters).** A **dash**: the smallest 2-bit build is 711 GB and needs about 724 GB, against 520 GB at most with the override. Its 1.3-bit build (466 GB) would load at 92 % with the override, tight — quality unknown, and only if you lower the floor in Advanced.

**F. 64 GB MacBook Pro (M5 Max), Qwen 3.8 Flash Next.** The machine has 68.7 GB; three quarters of it is **51.5 GB**. Every regular 2-bit-and-up build starts at 75 GB, so none fits. The hand-maintained SSD-paged build (45.8 GB resident) fits without the override: 45.8 + 0.9 + 2.0 = 48.6 GB → ring: *"with the SSD-paged build; tight"* (94 %), speed shown as the community's measured 36 tok/s because that measurement was taken on a 64 GB MacBook Pro; on any other machine the build shows no speed figure.

## 10. Changelog

- **0.9 — 2026-09-05.** The speed model is now a fixed-cost model: milliseconds per token = a read of the bytes per token at the chip's effective bandwidth + a fixed cost (the chip's, fitted from the F16/Q8_0/Q4_0 rows of the community table, plus an architecture-class cost fitted on measured runs) + a context term; tok/s is its inverse (section 6.1). MLX now scales the fixed cost by model group instead of multiplying the speed, GGUF K-quant files on Apple silicon carry a 1.25 read factor, and the MLX long-context penalty is a fitted term instead of a 24K–36K fade (section 6.2). Fitted on 73 measured runs and judged on 19 held-out ones: median error ×1.14 against ×1.21 for the multiplier model, with no row getting worse. What moves: mixtures of experts with a small active set lose the 1.5–3× the old model gave them (GLM-5.3 Flash on the M5 Ultra ≈ 35 tok/s, was 59; its 3-bit build on the Spark ≈ 15, was 25), 8-bit dense files on the Ultras gain, and the Mac-vs-Spark contrast on GLM-5.3 Flash shrinks from about 4× to about 2.3×. Memory margins became explicit policy (sections 4 and 5): buffers are 1.5 GB plus 1 % of the weights instead of 5 % capped at 16 GB (a 467 GB file is charged 6.2 GB, not 16), MLX files carry 0.15 GB of prompt scratch per 1K tokens beyond 8K as their own bar segment, and with the GPU limit lifted macOS keeps the larger of 6 GB and 5.5 % of the memory (30 GB on a 512 GB Studio). New subsection in section 1 on the five kinds of number. Worked examples rewritten: A is tight at 90 % (the buffers grew), B is tight at 99 % without the override, C at 93 %, E's nearest miss is 724 against 520 GB. The aired-chart cross-check disagrees in 17 of 108 cells (the realistic chart's 48 GB Qwen cell is tight at 92 %). Data: `validation.json` (120 measured speed and memory rows with sources, grades and the train/holdout split) joins the mirrored files; `benchmarks.json` gains the F16 and Q8_0 rows per chip (role `floor-fit`, outside the retired multiplier's medians).
- **0.8 — 2026-09-05.** Verified defects from two independent audits against measured runs. Memory sizes are now converted from the binary gigabytes Apple and NVIDIA quote to the decimal gigabytes file sizes use before any reserve is subtracted (32 GB = 34.4 GB, 512 GB = 549.8 GB); the macOS GPU limit is ⅔ under 36 GB and ¾ from 36 GB, as Metal's logged working set shows, not ¾ from 64 GB (36 and 48 GB Macs gain about a fifth of their usable memory). The context cache defaults to FP16, which is what every runtime stores unless told otherwise (it was 8-bit); Gemma 4's shared key/value projection no longer halves the stored cache (26B-A4B ≈ 1.4 GB at 8-bit, 2.9 GB at FP16, was 0.7); a hand-maintained build's measured speed is shown only on the machine class it was measured on. DGX Spark efficiency is 0.75 from five llama.cpp maintainer rows (build b7941); it was 0.40 from a single row that was recorded as a 40 GB Q4 llama.cpp run but was in fact LMSYS's Llama 3.1 70B at FP8 under SGLang (kept in the data for provenance, excluded from calibration) — every Spark speed rises about 1.9×. Mistral Small 4's active parameters are 6.5B per its model card (the config-derived estimate was 8.0B). Worked examples rewritten: the 256 GB Studio fits GLM-5.2's 2-bit build with the override (D is a ring), the 64 GB MacBook Pro runs Flash Next's SSD build without the override (F), the Spark fits GLM-5.3 Flash's 3-bit build (B). The aired-chart cross-check now disagrees in 16 of 108 cells, none from a formula error. The MLX fade end is single-sourced at 36K (the data file said 30K).
- **0.7 — 2026-09-04.** The memory chart became "How much memory?": absolute bars of what a configuration needs (weights + context cache + buffers), machine-independent, so the same model at two builds or two context lengths can be compared; one machine's limit is a dashed reference line. No formula or constant changed.
- **0.6 — 2026-09-03.** The simple UI: all configuration moved into a settings panel (machines with tickable memory sizes, models as one-build-one-context columns, drag to reorder, the two switches, Advanced folded in); four chart tiles (fit table, speed, which machine, memory); the memory breakdown became a full chart; a Feedback button and a closable notify card replaced the forms under the table. Section 1's reading guide updated. No formula or constant changed.
- **0.5 — 2026-09-03.** The About page (`/tools/llm-sizer/about`) renders this document with a FAQ and the sources behind every machine, benchmark row and factor; every number in a cell's sheet now links to its source (section 1). Correction recipe by file (section 11). Colours inside the tool darkened for contrast (the video's palette stays on the filming canvas). Cross-check against the charts that aired in the Apple-Macs video (2026-08-29), reproduced through the tool's own table path and kept as a test: the tool disagrees in 14 of 108 cells, none from a formula error. The reasons: the chart drew "does not fit at Q4" as a dash where the tool shows the cheapest compromise as a ring (a 2-bit or 3-bit build, a shorter context, the memory-limit override, or closing your apps in the realistic chart); the chart assumed the three-quarters memory limit on a 32 GB Mac where macOS allows two thirds (Qwen 3.8 27B at Q4 + 128K needs 22.2 GB against 21.4); Qwen 3.8 Flash Next's Q4 file is 119 GB (the n-gram table is inside it), not the ≈ 85 GB the chart assumed; GLM-5.3 Flash at Q4 needs the override on a 256 GB Studio once buffers count; GLM-5.2's 2-bit build does not fit 256 GB with its cache and buffers, and on 512 GB it needs the override and lands at 97 % (tight, a ring); and the speed table's flat 0.48 efficiency is now per chip (section 6.1). No formula or constant changed.
- **0.4 — 2026-09-03.** The app. New "How to read the table" subsection (section 1): the three marks, the two switches, tight cells as rings, speed shown for the configuration that fits, links and custom entries. Fix descriptions now separate the tight note with a semicolon instead of a dash; the alternatives list never contains a superset of an earlier alternative. No formula or constant changed.
- **0.3 — 2026-09-02.** The engine. Verdicts now have four states (runs / tight / compromise / doesn't fit) with the fix search's costs written down; the 2-bit quality floor (by label) and hand-maintained special builds added; MLX factor fades 24K → 36K instead of dropping at 30K; efficiency computed on active weights for mixture-of-experts benchmark rows (Ultra tier 0.44, M5 Ultra assumption 0.55); chip names matched without their GPU bin; ROCm machines get no speed number. Worked examples rewritten from the live data: Qwen Q4 is 16.8 GB (LM Studio's file), GLM-5.3 Flash ≈ 59 tok/s at 8K on the M5 Ultra, its Q4 does not fit the DGX Spark (the 2-bit build runs ≈ 17 tok/s), GLM-5.2 on 256 GB is a dash once buffers are counted, Kimi K3 and Flash Next examples added.
- **0.2 — 2026-09-02.** Data pipeline built. Section 3 now describes how models are discovered (five quantizer organisations, twelve-month window, featured list, name-based exclusions) and the publish-time safety net. Efficiency constants are now generated by `calibrate.py` from `benchmarks.json` into `factors.json` (tier medians 0.79 / 0.74 / 0.63 / 0.45 / 0.40; measured chips use their own value, M5 Max 0.82); M5 Ultra documented as an explicit 0.57 assumption. Acceleration ranges computed from paired rows. Gemma 4 cache corrected to its global-layer heads (≈ 0.7 GB at 128K, not 1.3). Machines catalog: one row per bandwidth bin (M5 Max 460 vs 614 GB/s; M6 mini 153 GB/s at 16 GB vs 170 at 24/32 GB). Spark speed in example B corrected from ~14 to ~11 tok/s.
- **0.1 — 2026-09-02.** First draft, written from the Phase 0 research spike: efficiency by chip tier from the community llama.cpp table (M1–M5 Max), context-cache formulas by attention design read from model configs, MLX runtime factors from 2026 community reports, MTP/DFlash factors from MTPLX and mlx-dspark measurements.

## 11. How to correct us

Use the feedback form in the tool (it attaches your exact configuration) or open an issue on the open-data repository, ideally with: machine, model, quant, context, runtime, and the number you measured. Accepted measurements go into the calibration data with your handle as the source, and the relevant row above changes with a changelog entry.

File by file (the canonical copies of `machines.json`, `benchmarks.json` and `models.overrides.json` live in the tool's repository and are mirrored to the open-data repository every night; propose a change as an issue or pull request there and it lands in the tool):

- **A machine fact** (memory options, bandwidth, price, status): edit the row in `machines.json` and add the page you are citing to its `sources`.
- **A speed you measured**: add a row to `benchmarks.json` with `chip`, `bandwidth_gbs`, `memory_gb`, `model`, `quant`, `weights_gb` (decimal GB; `active_weights_gb` too for a mixture-of-experts model), `runtime` (`llama.cpp-gguf`, `mlx`, or another runtime's name — only llama.cpp rows calibrate the efficiency), `accel` (`none`, `mtp`, `dflash`, `dflash2`, `dspark`), `tg` (writing tokens per second), `pp` (prefill, optional), `source` (a public link), `date`, and where you have them `build` (the runtime build) and `context_tokens` (tokens already in context when you measured); accelerated rows also carry `baseline_tg`, the same setup without the acceleration; rows meant for the chip-profile fit (F16, Q8_0 and Q4_0 reads of one dense model at short context) carry `role: "floor-fit"`. A measurement of a whole configuration (a model at a quant, runtime and context, with tokens per second or peak memory) goes into `validation.json` with a `grade` (A: the maintainers' or a reproducible tool's number; B: a community post with the setup stated; C: a headline number) and a `split` (`train`, `holdout`, or `report` for rows the fit must not learn from). Do not edit `factors.json`: it is regenerated from the rows.
- **A model fact** the Hugging Face API cannot express (a published active-parameter count, a licence threshold, a hand-maintained build): `models.overrides.json`.
- **A formula or constant**: open an issue. `FORMULAS.md` is a nightly copy of this document and is not edited in the data repository.
