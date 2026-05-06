## Long-Context Retrieval Bench

Measuring how frontier LLMs degrade at retrieval as context grows from 8K to 1M tokens.

**Status:** 🚧 In active development. Initial results expected July 2026.

## What this measures

Three retrieval task types across a grid of context lengths and needle depths:
- **Single-needle retrieval** — finding one fact in a long context
- **Distractor-needle retrieval** — finding the target fact when plausible decoys are present
- **Multi-hop retrieval** — combining facts from two separate locations

Tested across Claude (Opus, Sonnet, Haiku 4.x), GPT-5 family, Gemini 2.5, and open-weight models.

## Why this exists

Existing long-context benchmarks are stale or narrow. Engineers building RAG and document analysis systems need current, reproducible data on where retrieval actually breaks. This repo provides that.

## Read the methodology

See [METHODOLOGY.md](./METHODOLOGY.md) for the full experimental design.

## Status

| Phase | Status |
|---|---|
| Methodology | ✅ Drafted |
| Seed dataset | 🚧 In progress |
| Harness | ⏳ Planned |
| Judge validation | ⏳ Planned |
| Initial runs | ⏳ Planned |

## License

MIT (code) · CC BY 4.0 (dataset, when released)

## Citation

Dasari, M. (2026). *Long-Context Retrieval Degradation Across Frontier LLMs* (Version 0.1) [Software]. GitHub.
