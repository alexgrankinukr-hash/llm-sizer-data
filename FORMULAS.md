# LLM Sizer — Methodology & Assumptions

**Version 0.1 · 2026-09-02 · Status: pre-launch draft.** This is the public, plain-language explanation of every number LLM Sizer shows. It is the single source of truth: the tool's About page and the open-data repo's `FORMULAS.md` are generated from it, and **any change to a formula, constant, or data source in the tool must be reflected here first** (that rule is part of the project's definition of done). Comments and corrections: the feedback form in the tool, or an issue on the open-data repository. Changelog at the bottom.

---

## 1. What the tool answers — and what it doesn't

LLM Sizer answers three questions for a machine you own or are considering: **which open AI models fit in its memory, how fast they will write answers, and what you'd have to change to make a model fit.** Every answer is a calculation from published data, calibrated against real measurements, and every estimate is labeled as one.

It does **not** run a benchmark on your machine, judge model quality (we show quantization-quality notes where quantizers publish them, nothing more), or cover image/video generation models (those are compute-bound and need a different method).

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
| Model weight sizes per quantization | The exact file sizes published on Hugging Face by the quantizers people use (Unsloth, Bartowski, LM Studio community, mlx-community) and by the model makers, read through the Hugging Face API | daily |
| Model architecture (layers, attention type, experts, context maximum, MTP heads) | Each model's `config.json` on Hugging Face; a small hand-maintained overrides file for facts the config can't express (licenses, "ships 4-bit only") | daily |
| Machines (memory options, bandwidth, GPU cores, prices) | Apple's and NVIDIA's published specifications, with the source link per row | by hand, when products change |
| Measured speeds used for calibration | The community llama.cpp Apple-silicon benchmark table (M1 → M5 Max, updated 2026-08-25), published MLX / MTP / DFlash measurements, and later community submissions through the tool | as available; each row carries its source |

The data date is shown on the page. A model released today is normally in the tool tomorrow, with its real file sizes.

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
| Key = Value sharing (`attention_k_eq_v`, Gemma 4) | halves the classic formula | — |

`bytes` is the cache precision: **8-bit by default** (1 byte), FP16 (2 bytes) or 4-bit (0.5) selectable in Advanced. Most 2026 models use compressed or hybrid attention, so long context is far cheaper than older fit charts assumed. Models whose design we can't read from the config are marked **"architecture assumed conventional"** and use the classic formula — the pessimistic choice.

**Verdicts:**
- **Runs** — `needed ≤ 90 %` of available.
- **Runs with a compromise** (the ring) — it fits only if something gives, and the cell says what: a smaller quantization, a shorter context, closing your work apps, or enabling the macOS memory-limit override. The tool searches the model's smaller quants and shorter contexts for the best combination that runs and shows it.
- **Doesn't fit** — no quantization of this model fits at any context.

## 6. Speed

### 6.1 Writing speed (decode) — the number that matters day to day
To write each token, the machine re-reads the model's active weights plus the context cache. So the ceiling is bandwidth divided by bytes read per token, and real machines reach a fraction of that ceiling:

```
bytes per token = active_params × bytes per weight (quant) + cache bytes per token × context length
decode tok/s    = bandwidth × efficiency ÷ bytes per token
```

**Efficiency** is measured, not assumed, from the community llama.cpp table (same model, every chip):

| Chip tier | Efficiency | Evidence |
|---|---|---|
| Apple base chips (M1–M5) | **≈ 0.78** | 0.76–0.83 measured |
| Apple Pro | **≈ 0.75** | 0.69–0.82 |
| Apple Max | **≈ 0.62** (M5 Max: **0.74**) | 0.58–0.74 |
| Apple Ultra | **≈ 0.43** | 0.40–0.45 — the two-die design pays a consistent penalty |
| NVIDIA DGX Spark (CUDA, unified) | ≈ 0.40 | dense 70B measurement |

Chips without a measured row (M5 Ultra, M6 at launch) inherit their tier's factor and are labeled **"estimate — unmeasured."** Because the context cache is in the formula, speed visibly falls as the context grows — more for classic-attention models, barely for hybrids.

### 6.2 Runtime: GGUF (llama.cpp / LM Studio) vs MLX
The baseline above is llama.cpp with a GGUF file. Apple's MLX runtime (MLX files, used by LM Studio's MLX engine, Ollama's MLX backend, mlx-lm) is faster on Apple silicon; the gap depends on the model:

| Model type | MLX factor | Evidence (2026 reports) |
|---|---|---|
| Dense, under ~14B | **≈ 1.0–1.2×** | roughly tied in most tests |
| Dense, 14B and up | **≈ 1.15×** (1.1–1.2) | consistent 10–20 % lead |
| Mixture-of-experts | **≈ 1.9×** (1.5–3×) | Ollama on M5 Max, Qwen3.5-35B-A3B: 58 → 112 tok/s; M4 Pro, Qwen3-Coder-30B-A3B: 43 → 130 |

Also: MLX files are 5–13 % smaller for the same nominal quant (the tool uses the actual MLX file size when you pick MLX), and MLX prefill is 30–40 % faster. **Caveat shown in the tool:** at very long contexts (~30K+ tokens) llama.cpp with flash attention can be faster than MLX — the factor is applied only up to 30K and flattened to 1.0 above it.

### 6.3 Acceleration: MTP and draft models (DFlash / DSpark)
Some models can write several tokens per step. Two ways: **MTP** (multi-token-prediction heads built into the model — we detect them in the config) and **draft models** (a small helper model proposes a block of tokens the big model verifies in one pass; DFlash and DSpark drafts exist for many popular models — we find them on Hugging Face daily). Output is identical; only speed changes. The tool offers these only for models that actually have them, and shows the result as a range beside the baseline, never as the headline:

| Method | Factor | Evidence |
|---|---|---|
| MTP (native heads) | **1.6–2.6×** | 9B on M4 mini 14.4 → 23.0; Qwen 3.8 27B on M4 Pro 7 → 18; 2.24× on M5 Max (MTPLX) |
| DFlash / DSpark draft, dense model | **2.0–3.6×** | Qwen 3.8 27B 4-bit on M4 Pro 14.7 → 33.8; 8-bit 8.4 → 30.5; Gemma 4 12B 17.8 → 49.4 (mlx-dspark) |
| DFlash / DSpark draft, MoE with a small active set | **1.1–1.3×** | Qwen3.6-35B-A3B 86.9 → 114.5 — little to gain when the model is already fast |

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

## 9. Worked examples

**A. 32 GB Mac mini (M6), used for work, Qwen 3.8 27B at Q4, 32K context.** Available = min(32 × 0.67, 32 − 6 − 16) = **10 GB**. Needed = 17.1 (weights) + 1.1 (cache: 32 KB/token × 32K) + 1 (buffers) = **19.2 GB** → doesn't fit while you work. Toggle "work machine" off: available 21.4 GB → **runs** (90 % rule: 19.2 ≤ 19.3 — tight). The ring says: *"runs if you close your apps."*

**B. 256 GB Mac Studio (M5 Ultra), GLM-5.3 Flash at Q4, 128K context.** Available with the default GPU limit = 192 GB; needed = 199.7 + 0.7 (cache) + 10 (buffers) = **210 GB** → ring: *"needs the macOS memory-limit override"*; with the override, available = 250 GB → runs. Speed: 18B active × 0.56 bytes ≈ 10 GB per token; 1,200 × 0.50 (M5 Ultra, tier estimate) ÷ 10 ≈ **60 tok/s**, labeled estimate; with MLX ≈ 1.9× for MoE → up to ~110; with its MTP heads, more.

**C. 512 GB Mac Studio, GLM-5.2 at Q4, 128K context.** Needed = 467 + 5.9 (cache — compressed attention) + 16 (buffers, capped) = **489 GB**; available with override = 506 GB → **runs, tight** — and a Reddit user measured ~473 GB in use on exactly this setup, which is the kind of confirmation that turns an estimate into a measurement.

## 10. Changelog

- **0.1 — 2026-09-02.** First draft, written from the Phase 0 research spike: efficiency by chip tier from the community llama.cpp table (M1–M5 Max), context-cache formulas by attention design read from model configs, MLX runtime factors from 2026 community reports, MTP/DFlash factors from MTPLX and mlx-dspark measurements.

## 11. How to correct us

Use the feedback form in the tool (it attaches your exact configuration) or open an issue on the open-data repository, ideally with: machine, model, quant, context, runtime, and the number you measured. Accepted measurements go into the calibration data with your handle as the source, and the relevant row above changes with a changelog entry.
