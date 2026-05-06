# Methodology: Long-Context Retrieval Degradation Across Frontier LLMs

**Status:** Draft v0.1
**Author:** Mahesh Dasari
**Last updated:** May 2026

---

## 1. Motivation

Frontier LLMs now advertise context windows of 200K, 1M, and beyond. Engineers building RAG pipelines, document analysis tools, codebase reasoning systems, and long-conversation agents rely on these models to faithfully retrieve information from anywhere in the provided context. In practice, retrieval accuracy degrades as context grows and as the target moves away from the start or end of the window — but the shape of that degradation is poorly characterized for current models.

Existing public benchmarks have known limitations:
- **Needle-in-a-Haystack (NIAH)** variants are widely used but most published runs are stale (run on prior model generations) and typically test only single-needle retrieval at a small number of depths
- **RULER** (Hsieh et al., 2024) is more rigorous but rarely re-run against new models on release
- **LongBench** and similar suites mix retrieval with summarization and reasoning, making it hard to isolate retrieval-specific behavior

As of 2026, the field lacks a current, reproducible, multi-model picture of:

- Where retrieval accuracy actually breaks down at modern context lengths (up to 1M tokens)
- How accuracy varies by depth (where the target sits in the context)
- How retrieval interacts with distractors (other plausible-looking content)
- How variance differs across runs (single-shot benchmarks underreport instability)
- How cost-sensitive deployments should choose between cheap and flagship models for long-context tasks

This benchmark addresses that gap.

## 2. Research Questions

The eval is designed to answer five concrete questions:

1. **Length effect:** How does retrieval accuracy degrade as context length increases from 8K to the model's stated maximum?
2. **Depth effect:** Within a given context length, does retrieval accuracy depend on the position of the target ("needle") in the context (the "lost in the middle" effect)?
3. **Distractor effect:** When the context contains plausible distractor information, does the model retrieve the correct target or the distractor?
4. **Variance:** How stable is retrieval performance across multiple runs of the same prompt at temperature 0?
5. **Cost-quality frontier:** Where does each model sit on the cost-vs-accuracy curve at long contexts?

## 3. Task Definition

The core task is needle-in-a-haystack retrieval, with deliberate extensions beyond the canonical version.

### Single-needle task

A "haystack" of natural-language text contains a single inserted "needle" — a factual statement that does not appear elsewhere in the haystack. The model is asked a question whose only correct answer comes from the needle. The model passes if it produces the correct answer.

### Distractor-needle task

The haystack contains the target needle plus 1–3 distractor needles — statements similar in form but factually different from the target. The question targets one specific needle. The model passes if it produces the correct target answer and does not substitute information from a distractor.

### Multi-hop needle task

Two needles must be combined to answer the question. For example: "Vellora is the capital of the Northern Reach" + "The Northern Reach was annexed by the Greycoast Empire in 1923" → "What empire annexed the country whose capital is Vellora, and when?" The model passes if it produces the correct combined answer.

All needles and questions are factual and verifiable; ambiguous or opinion-based questions are excluded.

## 4. Dataset Design

### Haystack corpus

Haystacks are constructed from a base corpus of long-form English text drawn from public-domain sources:
- Project Gutenberg classics (narrative fiction): *War and Peace*, *Moby-Dick*, *Les Misérables*, *Anna Karenina*
- Wikipedia long articles (expository nonfiction): selected for length and topical diversity
- US federal public-domain documents (technical reference)

Multiple genres are included to test whether retrieval performance varies by surrounding text type. All haystack content is verified to not contain information that conflicts with inserted needles via substring search on key entities.

### Needle construction

Needles are short factual statements crafted to be:
- **Self-contained** (interpretable without surrounding context)
- **Distinctive** (unlikely to be confused with surrounding text)
- **Verifiable** (have one objectively correct answer)
- **Absent from the base corpus** (verified via substring search)

Example needle:
> *"The Brindlemark Treaty was signed in 1847 by representatives of three city-states: Vellora, Saintwick, and Carnholm."*

Example question:
> *"In what year was the Brindlemark Treaty signed, and which city-states were represented?"*

Names are deliberately fictional to prevent the model from answering from prior knowledge. Each needle includes a list of "key entities" that an answer must contain to score correct.

### Schema

Each needle is stored as a JSONL row with this schema:

```json
{
  "id": "n001",
  "category": "single | distractor | multihop",
  "needle_text": "string",
  "distractors": ["optional list for distractor tasks"],
  "secondary_needle": "optional, for multihop only",
  "question": "string",
  "canonical_answer": "string",
  "key_entities": ["list of strings that must appear in correct answer"],
  "fact_type": "date | named_entity | numerical | quote | sequence | definition",
  "rationale": "why this needle was included",
  "difficulty": "easy | medium | hard"
}
```

### Test grid

Each needle is tested across a grid of conditions:

| Dimension | Values | Notes |
|---|---|---|
| Context length | 8K, 32K, 128K, 256K, 500K, 1M tokens | Where supported by model |
| Needle depth | 0%, 25%, 50%, 75%, 100% | Position from start of context |
| Task type | single, distractor, multi-hop | Per needle category |
| Repetitions | 3–5 per cell | Captures variance |

For v1.0 release, the dataset targets **50 unique needle/question pairs** distributed as:
- 30 single-needle (varied fact types)
- 15 distractor-needle
- 5 multi-hop

This generates approximately 3,000–7,500 evaluation cells per model depending on grid scope.

### Public vs. private split

80% of the dataset (40 needles) is released publicly. 20% (10 needles) is held as a private test set, committed to a private branch, and not made public. The private set is used to detect contamination in future model releases — if a future model scores notably better on public than private examples, that suggests training data leakage.

## 5. Models Under Test

Initial release evaluates the following models across their accessible context lengths:

| Provider | Model | Max context (approx.) | Tier |
|---|---|---|---|
| Anthropic | Claude Opus 4.7 | 200K+ | Flagship |
| Anthropic | Claude Sonnet 4.6 | 200K+ | Mid |
| Anthropic | Claude Haiku 4.5 | 200K | Cheap |
| OpenAI | GPT-5 | 128K–1M | Flagship |
| OpenAI | GPT-5 mini | 128K | Cheap |
| Google | Gemini 2.5 Pro | 1M+ | Flagship |
| Google | Gemini 2.5 Flash | 1M+ | Cheap |
| Open-weight | Llama 3.3 70B (via Together AI) | 128K | Open |

Where context length grid cells exceed a model's stated max, those cells are reported as "not applicable" rather than failures.

Model versions, dates, and exact API model identifiers are recorded for every run. Future model releases trigger re-runs.

## 6. Prompting

All models receive the same prompt structure:

```
[Long context haystack with embedded needle]

---

Question: {question}

Answer the question based only on the information in the text above.
If the answer is not in the text, say "Not found in the provided context."
Be concise.
```

Sampling parameters held constant across models:
- Temperature: 0 (deterministic)
- Top-p: 1.0
- Max output tokens: 500
- No system prompt overrides

This standardization means model differences reflect model behavior, not prompt tuning. Per-model prompt tuning is left for future work.

## 7. Scoring

### Two-stage scoring

Each model output is scored as **correct**, **partial**, **incorrect**, or **refused** using a two-stage approach:

**Stage 1 — Exact-match check:** The output is checked for the presence of all `key_entities` (case-insensitive substring match). If all key entities are present and the model has not refused, the output scores **correct** without further processing. This handles the majority of clear-cut cases cheaply.

**Stage 2 — LLM-as-judge:** Outputs failing exact-match are sent to an LLM judge with the question, canonical answer, key entities, and model output. The judge produces a structured verdict.

The judge prompt and rubric are versioned and published in the repo (`prompts/judge_v1.txt`).

### Judge model

Claude Opus 4.7 is used as the judge model. This choice introduces a known correlation with judge bias; we mitigate via human validation (below) and by reporting full inter-rater agreement statistics.

### Judge validation

Before publishing any leaderboard, the LLM-as-judge is validated against human judgment on a 100-output sample drawn across all models and conditions. The human rater (the project author) rates each output blind to the judge's verdict. Inter-rater agreement is reported using Cohen's kappa.

**Acceptance threshold: kappa ≥ 0.80.**

If kappa falls below this threshold, the judge prompt is revised and validation is repeated. The judge is not used for final scoring until the validation threshold is met. The validation sample, ratings, and full confusion matrix are published as part of the release.

### Distractor scoring

For distractor-needle tasks, outputs are additionally classified as:

- **Correct:** retrieved the target needle's information
- **Distracted:** retrieved a distractor needle's information
- **Hybrid:** combined target and distractor information
- **Other failure:** unrelated answer or no answer

This finer-grained scoring surfaces a specific failure mode that simple correct/incorrect scoring hides.

## 8. Metrics Reported

For each (model × context length × depth × task type) cell, we report:

- **Accuracy:** mean correctness across repetitions, with 95% Wilson confidence intervals
- **Variance:** standard deviation across repetitions
- **Distraction rate** (distractor task only)
- **Refusal rate:** % of cells where the model declined to answer
- **Cost per run:** average input + output token cost in USD

### Headline visualizations

- **Accuracy heatmaps:** context length × depth, one panel per model
- **Length curves:** accuracy vs context length, all models on one chart
- **Depth curves:** accuracy vs needle position, faceted by length
- **Variance plot:** confidence interval widths by condition
- **Cost-quality plot:** per-correct-answer cost vs accuracy by model

## 9. Reproducibility

All of the following are public and version-controlled in the repository:

- Full source code for haystack construction, needle insertion, model calls, and scoring
- 80% of dataset (held-out 20% described above)
- LLM-as-judge prompt with version tag
- Judge validation results (judge-vs-human agreement on the 100-sample set, including the disagreement examples)
- Per-run logs in JSONL format with full input/output, model version, timestamp, and cost
- Analysis notebooks generating every published chart
- Exact dependency versions (`pyproject.toml`, lock file)

Anyone with API access can clone the repo and re-run the eval against new models or new model versions:

```bash
git clone <repo>
cd longcontext-bench
uv sync
cp .env.example .env  # add API keys
python -m lcbench.run --model claude-opus-4-7 --grid v1
```

## 10. Cost Budget

Estimated cost across all models for the v1.0 grid:

| Component | Estimated cost |
|---|---|
| Cheap-tier models (Haiku, Flash, GPT-5 mini, Llama 3.3) | $100–250 |
| Flagship models (Opus, Sonnet, GPT-5, Gemini Pro) | $250–600 |
| Judge runs (Opus on failed-exact-match outputs) | $80–150 |
| Development and debugging | $50–100 |
| **Total estimated** | **$480–1,100** |

Cost reduction strategies actually employed:

- **Response caching** by `(model_id, prompt_hash)` tuple to avoid duplicate API calls during debugging and re-runs
- **Tiered execution** — cheap models run first to validate harness; flagship runs only once methodology is locked
- **Reduced grids during development** — full grids only for published runs
- **Researcher access programs** — applications submitted to Anthropic and OpenAI in week 1 of project

Actual costs are tracked in `costs.md` and updated as runs complete.

## 11. Limitations

This eval has known limitations, stated upfront:

- **English only.** Results do not generalize to other languages. Cross-lingual long-context behavior is left for v1.2.
- **Synthetic needles.** Real-world long-context tasks (legal documents, codebases, technical manuals) have different statistical properties than narrative haystacks with fictional needles. Domain-specific haystacks are planned for v2.0.
- **Single retrieval task family.** This eval measures one specific capability (factual retrieval); it does not measure summarization, reasoning over long context, code understanding, or in-context learning.
- **Provider-side variance.** The same model accessed through different APIs or with different sampling parameters may produce different results. We use temperature 0 and report all parameters, but provider-side stochasticity may persist.
- **Snapshot in time.** Models update. Results are valid for tested model versions on tested dates. Re-runs accompany major model releases.
- **Judge dependency.** LLM-as-judge introduces a small but real correlation with the judge model's biases. We mitigate via human validation, but the dependency remains.
- **Single human rater.** Judge validation uses one human rater (the author). Multi-rater validation with inter-human kappa would strengthen confidence and is left for v1.1.
- **Prompt sensitivity.** Results may vary with prompt wording. We do not currently measure how much prompt phrasing changes the findings.
- **Tokenization differences.** "1M tokens" means different things across providers due to different tokenizers. We use each provider's own tokenizer for length budgeting, which means comparisons across providers at the same nominal length are approximate.

## 12. Ethical and Safety Considerations

This eval does not generate harmful content, does not test refusal behavior, and does not require red-teaming. Haystack content is drawn from public-domain sources. No personally identifiable information appears in needles. The eval does not provide capabilities uplift; it measures existing model retrieval behavior. No human subjects are involved beyond the project author's own ratings during judge validation.

## 13. Roadmap

| Version | Scope | Target |
|---|---|---|
| **v1.0** | English single + distractor + multi-hop, 8 models | Initial public release |
| **v1.1** | Multi-rater judge validation; improved variance estimates | First update |
| **v1.2** | One non-English language (likely Spanish or French) | Cross-lingual track |
| **v2.0** | Domain-specific haystacks (legal, code, technical docs) | Major expansion |
| **v2.1** | Adversarial / paraphrased needles | Robustness track |

## 14. How to Contribute

Issues and pull requests are welcome. Contributions specifically valued:

- Additional needle/question pairs (especially testing different fact types)
- Independent reproductions on new models
- Translation of methodology and select needles to other languages
- Bug reports on harness or judge prompt
- Multi-rater validation contributions

See `CONTRIBUTING.md` for details.

## 15. How to Cite

If you use this work, please cite as:

> Dasari, M. (2026). *Long-Context Retrieval Degradation Across Frontier LLMs* (Version 1.0) [Software]. GitHub: [repository URL].

A BibTeX entry will be provided in the repo on release.

## 16. Acknowledgments

This methodology builds on and is indebted to:

- Greg Kamradt's original Needle-in-a-Haystack work, which established the basic experimental paradigm
- The RULER benchmark (Hsieh et al., 2024) for rigorous long-context evaluation methodology and the importance of distractor and multi-hop variants
- Liu et al. (2023), *Lost in the Middle: How Language Models Use Long Contexts*, for documenting depth-dependent retrieval degradation
- Anthropic's published evaluations of Claude long-context capability
- The UK AI Safety Institute's Inspect AI framework, used as a structural reference for harness design
- The broader open-source LLM evaluation community

---

## Appendices (to be added)

**Appendix A:** Final judge prompt (added after judge validation)
**Appendix B:** Dataset statistics (added after dataset finalization)
**Appendix C:** Judge validation results — kappa score and disagreement examples (added after week 5)
**Appendix D:** Full results tables (added after v1.0 runs complete)
