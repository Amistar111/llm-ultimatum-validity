# llm-ultimatum-validity
 
**Large language models as simulated subjects: diagnosing observation dependence and contamination in the Ultimatum Game**
 
Research project (НИР), 1st year, MSc Big Data and Machine Learning, ITMO University.
Supervisor: Valeria Volokha (CNC Lab). Author: Amina Marnova.
 
---
 
## 1. What this project is about
 
LLMs are increasingly used as *simulated subjects* in economic experiments: a model is prompted
with an experimental protocol, run many times, and the resulting distribution of decisions is
treated as if it were a sample of human participants. The appeal is obvious — a thousand runs
cost less than a single lab session.
 
This project does not propose a new game or a new model. It asks a narrower and more basic
question:
 
> **Under what conditions are the *conclusions* drawn from LLM-generated experimental data valid?**
 
Note the phrasing. Validity is a property of the inference, not of the model. A model is neither
"valid" nor "invalid"; a conclusion drawn from its outputs can be either.
 
The project answers this through **two diagnostics**, both run on the Ultimatum Game (UG) — the
most canonical, most documented, most replicated bargaining game in experimental economics, and
therefore the hardest case for an LLM to fail in an interesting way.
 
## 2. The two diagnostics
 
### Diagnostic 1 — Observation dependence
 
1,000 runs of one model are not 1,000 independent participants. Repeated sampling from a single
model with a fixed prompt produces observations that share a common source: the same weights,
the same system prompt, often the same decoding parameters. Standard inferential statistics
assume independence; applied to such data they understate standard errors and overstate
significance.
 
This diagnostic quantifies that distortion:
 
- estimate the within-model / between-run dependence structure (intra-source correlation,
  variance decomposition across prompt seeds, temperature settings, and model identity);
- compare naive significance (treating runs as i.i.d.) against estimates that account for
  dependence (clustered / multilevel specifications, design effect);
- report the effective sample size implied by *N* runs — i.e. how many independent observations
  a thousand runs are actually worth.
The deliverable is not "LLMs are bad". It is a correction factor and a stated procedure for
reporting significance when the data source is a single model.
 
### Diagnostic 2 — Contamination
 
Suppose an LLM reproduces the canonical UG result: proposers offer close to 40–50%, responders
reject low offers. Two explanations are observationally identical in the canonical setting:
 
- **(a) behaviour** — the model is in some sense *playing* the game, and its decisions respond to
  the game's incentives and structure;
- **(b) recall** — the model has memorised the canonical result from its training corpus, in
  which the Ultimatum Game and its typical outcomes appear thousands of times, and is
  reproducing a remembered number.
Distinguishing (a) from (b) is the core methodological work of this project, and the part
most likely to be challenged in review. Three instruments are used jointly:
 
1. **Perturbed game variants.** The canonical formulation is compared against versions in which
   surface features are changed while the strategic structure is preserved: renamed roles and
   game label (no "Ultimatum", no "proposer/responder"), non-round pie sizes (e.g. 97 units
   instead of 100), and alternative framings (neutral / exchange / division wording).
   A model that *behaves* should be largely invariant to relabelling; a model that *recalls*
   should degrade when the canonical cues are removed.
2. **Comparative statics.** Parameters are varied along directions where theory and the human
   evidence give a known qualitative prediction (pie size, outside options, stake magnitude,
   one-shot vs repeated framing). The test is whether the model's responses move in the right
   direction, not whether they match a particular human mean.
3. **Memory probes.** Direct elicitation of what the model knows about the game and its
   published results, separately from playing it, to establish how much of the canonical answer
   is retrievable as knowledge.
No single instrument is decisive. The claim the project is willing to defend is a joint one:
invariance under relabelling *plus* correct comparative statics *plus* a demonstrable
recall channel together identify which of (a) and (b) is doing the work.
 
## 3. Models
 
| Model | Access | Role |
|---|---|---|
| LLaMA-3.1-8B-Instruct | local, Ollama | primary model — full control over seeds, temperature, and system prompt; fully reproducible |
| 2–3 closed-weight models | OpenRouter | external comparison — checks whether findings are specific to one open model |
 
The local model is the workhorse precisely because the first diagnostic requires control over
the sampling process. Closed models are used for generalisation, with the caveat that their
versioning and decoding internals are not under the experimenter's control — itself a
reproducibility finding worth reporting.
 
## 4. Relation to prior work
 
This project builds on the author's undergraduate research on human expectations of LLM
behaviour in the Ultimatum Game ("Mind the Gap"), and reuses parts of that experimental
code and data. The contribution here is methodological rigour applied to the *use* of LLMs as
subjects, not a new behavioural result.
 
Market / zero-intelligence-agent ablations in the spirit of Gode & Sunder are deliberately out
of scope for this project and are reserved for follow-up work.
 
## 5. Repository structure
 
Planned layout (directories appear as the corresponding stage is implemented):
 
```
.
├── src/                 # experiment runner, model clients, prompt construction
│   ├── games/           # canonical UG and perturbed variants
│   ├── clients/         # Ollama and OpenRouter interfaces
│   └── analysis/        # dependence estimation, comparative statics, probe scoring
├── configs/             # experiment configs (model, variant, parameter grid, seeds)
├── prompts/             # prompt templates, versioned
├── data/
│   ├── raw/             # raw model outputs (one file per run batch, never edited)
│   └── processed/       # parsed decisions, tidy format
├── notebooks/           # exploratory analysis and figure generation
├── results/             # tables and figures used in the report
├── docs/                # literature base, protocol, report drafts
└── tests/
```
 
Raw model outputs are append-only: every batch is stored with the full config that produced it
(model identifier, prompt version, temperature, seed, timestamp), so any figure can be traced
back to the exact calls behind it.
 
## 6. Reproducing the experiments
 
> Not yet runnable — this section is the intended interface and will be filled in as `src/` lands.
 
```bash
# environment
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
 
# local model
ollama pull llama3.1:8b
 
# credentials for closed models (never committed)
cp .env.example .env    # then add OPENROUTER_API_KEY
 
# run one experiment
python -m src.run --config configs/ug_canonical_llama31.yaml
 
# reproduce the analysis
python -m src.analysis.dependence --input data/processed/
python -m src.analysis.statics    --input data/processed/
```
 
## 7. Status and roadmap
 
- [x] Topic finalised with supervisor; literature base assembled (23 curated papers)
- [x] Repository initialised
- [ ] Experimental protocol written up (`docs/protocol.md`): variants, parameter grid, sample sizes
- [ ] Prompt templates and parser for model decisions
- [ ] Canonical UG runs on LLaMA-3.1-8B, with seed and temperature grid
- [ ] Perturbed variants (relabelling, non-round pie, framing)
- [ ] Memory probes
- [ ] Dependence analysis: variance decomposition, design effect, effective N
- [ ] Comparative statics analysis
- [ ] Closed-model comparison via OpenRouter
- [ ] Report and defence presentation (winter defence, January 2027)
## 8. Terminology used in this repository
 
The project is deliberate about wording, and the same conventions apply throughout the code,
docs, and report:
 
- **conclusions** are valid or invalid — models are not;
- **dependence** of observations (not "independence"), because dependence is what is being
  measured;
- **simulated subjects** (not "replacement for human subjects");
- **diagnostics** (not "validity conditions"), since the project tests specific failure channels
  rather than systematically enumerating all factors that could affect validity.
## 9. License
 
MIT — see [LICENSE](LICENSE).
 
