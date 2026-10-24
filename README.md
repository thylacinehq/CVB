# CVB, the Constraint-Violation Benchmark

A benchmark that tests whether coding agents honor project constraints when those constraints arrive as memory instead of orders.

## What it does

Teams record a constraint once and expect every future AI generation to honor
it. "We standardized on httpx after requests caused the March
socket-exhaustion incident." In practice that constraint reaches the model two
very different ways:

1. as explicit instructions pasted into a prompt, or
2. as ambient memory context injected by a memory layer, a CLAUDE.md
   file, or RAG.

Instruction-following benchmarks (IFEval, AgentIF) only measure the first.
CVB measures both, plus a no-context baseline, and reports the difference.

## Status

Version 2.0.0, prompt version 2.0. One full run is published (2026-07-30 to 2026-08-02, Groq). It covers single-turn code generation only. See Results and Limitations below.

## Stack

Python 3.10+, `httpx`, `pytest`. Any OpenAI-compatible endpoint.

## Arms

| Arm | Context the model receives | What it measures |
|-----|---------------------------|------------------|
| `cold` | task only | baseline violation rate |
| `mandated` | task + constraints framed as explicit orders | instruction-following ceiling |
| `incentivized` | task + the same constraints embedded in an ambient project-memory narrative, never framed as orders | memory adherence |

**The headline metric is the gap.** `mandated strict accuracy − incentivized
strict accuracy`. It measures how much adherence dies when rules live in
memory instead of orders.

Prompt templates are frozen strings in [`cvb/prompts.py`](cvb/prompts.py)
(`PROMPT_VERSION = "2.0"`). The incentivized narrative is built
deterministically from the scenario file. No LLM touches scoring or context
construction. Scenario lint bans imperative wording ("must", "required",
"rule", "you should") from constraint text, so the incentivized arm stays
genuinely non-directive.

## Scoring

- 35 scenarios, 7 categories x 5 (library-choice, security, encoding-io,
  style-architecture, error-handling, concurrency, logging-testing).
- Each scenario is a natural coding task that tempts the default violating
  behavior, plus 2-4 constraints. Every constraint carries its own
  deterministic regex checks.
- Reported per arm: **strict accuracy** (all constraints of a run honored)
  and **per-constraint accuracy** (IFEval-style).
- No LLM judge. Temperature 0. Runs are cheap and exactly reproducible.

## Results

**Run of 2026-07-30 – 2026-08-02.** Groq API, temperature 0, 3 runs per
arm, 35 scenarios, prompt version 2.0, 1260 records total. Raw JSON in
[`results/2026-07-29-groq/`](results/2026-07-29-groq/). Strict accuracy is the
share of runs with every constraint honored.

| model | cold | mandated | incentivized | gap (mandated − incentivized) |
| --- | --- | --- | --- | --- |
| `llama-3.1-8b-instant` | 0.229 | 0.914 | 0.857 | **+0.057** |
| `llama-3.3-70b-versatile` | 0.229 | 0.952 | 0.981 | **−0.029** |
| `openai/gpt-oss-120b` | 0.638 | 0.962 | 0.952 | **+0.010** |
| `qwen/qwen3.6-27b` | 0.476 | 1.000 | 1.000 | **0.000** |

What the numbers say:

1. **The scenarios genuinely tempt violations.** Cold accuracy is 0.23 for
   both Llamas. Without context, models default to the violating pattern
   (naive `datetime.now()`, `shell=True`, no locks, `print` logging).
2. **Context injection is worth +60–75 points.** Every arm that carries the
   constraints massively beats cold. That holds whether they arrive as
   orders or as memory. The main battle is getting constraints into context
   at all.
3. **The mandated-vs-incentivized gap is small and model-dependent.** The
   weakest model (8B) loses 5.7 points when constraints arrive as ambient
   memory instead of orders. The 70B model actually adheres *better* to
   memory framing (−2.9). qwen3.6-27b is perfect under both framings
   (gap 0.0). For current mid-size models, non-directive memory context is
   roughly as effective as explicit instructions in single-turn generation.

Per-category tables: [`results/2026-07-29-groq/report.md`](results/2026-07-29-groq/report.md).

## Limitations

- **Regex checks are conservative.** They catch the canonical violation and
  miss others. A pass means "no detected violation", which falls short of
  proof of compliance.
- **Baselines differ by model.** A stronger model violates less cold. The
  gap metric is within-model, which controls for this, but absolute rates
  are not comparable across models.
- **Weak models can fail checks for competence reasons** (broken code)
  instead of disobedience. Checks target the violating pattern and ignore
  code quality, and per-constraint accuracy limits the blast radius, but the
  confound does not fully vanish.
- **Scenario leakage.** Public benchmarks enter training data. Scenarios
  are versioned and results date-stamped; treat future scores accordingly.
- Single-turn code generation only. No tool use, no retrieval. That
  isolates adherence from retrieval quality.

## Why this exists

- Conversational-recall benchmarks are the wrong yardstick for developer
  memory. An independent audit of LoCoMo found ~6.4% of its answer key
  wrong, and its LLM judge accepted 63% of intentionally wrong answers.
- PROJECTMEM (arXiv 2606.12329) established the Memory-as-Governance
  framing. It ships no constraint-adherence evaluation. CVB fills that
  slot.
- Motivating product: [HCR](https://github.com/PantheraLabs/HybridCognitiveRuntime),
  a developer memory layer whose job is exactly the incentivized arm. The
  benchmark imports nothing from it and runs without it.

## Run it

```bash
pip install -e .
export GROQ_API_KEY=...   # or any OpenAI-compatible endpoint via --base-url

python -m cvb.runner --scenarios scenarios --dry-run          # list matrix
python -m cvb.runner --scenarios scenarios --runs 3 \
    --json results/out.json                                   # ~945 calls
python -m cvb.report results/out.json --markdown results/out.md
```

Default models are probed from the live Groq catalog (first three available
of: `llama-3.3-70b-versatile`, `llama-3.1-8b-instant`, `openai/gpt-oss-120b`,
`qwen/qwen3.6-27b`, `openai/gpt-oss-20b`).
Override with `--models`. Point anywhere OpenAI-compatible with
`--base-url` / `--api-key-env`.

Prompt changelog: `2.0` (2026-07-29), initial three-arm templates.

## Project structure

```
cvb/         runner, prompts, scenario loader, checks, report
scenarios/   35 scenario JSON files
tests/       pytest suite (pip install -e ".[dev]" && pytest)
results/     raw JSON and report for the 2026-07-29-groq run
notes/       design notes for v2
docs/        index.html
```

## Contributing

Issues and pull requests are welcome. New scenarios need deterministic regex checks and must pass the scenario lint (no imperative wording in constraint text). Run `pytest` before opening a PR.

## License

MIT. See LICENSE.
