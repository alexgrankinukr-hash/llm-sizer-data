# models/

`index.json` is the search index (one row per model); `<id>.json` is the full record for that model. Both are written by the nightly export from the tool's database, which the discovery job fills from the Hugging Face API (five quantizer organisations plus the featured list; models from the last twelve months with at least a billion parameters). Field meanings and formulas: `FORMULAS.md` §3 and §5. Every quantization row names the Hugging Face repository the size was read from.
