# Experimental protocol

Version 0.1 — draft for supervisor review (November 2026).
Project: *Large language models as simulated subjects: diagnosing observation dependence and
contamination in the Ultimatum Game.*

This document fixes, before any data is collected, what will be run, how many times, how the
outputs will be parsed, and — most importantly — which observed patterns will be read as
evidence for which conclusion. Section 8 is the decision rule; it is written in advance so that
the interpretation is not chosen after seeing the results.

---

## 1. Design overview

Two diagnostics share one data collection effort:

- **D1 (observation dependence)** needs *many runs of the same cell* under systematically varied
  nuisance factors (prompt paraphrase, temperature, seed) in order to decompose variance.
- **D2 (contamination)** needs *few runs of many cells* — perturbed game variants and parameter
  points where theory predicts a direction of movement.

The run grid is therefore crossed: every analysis cell is executed across the same nuisance
factors, so D1 can be estimated inside any cell and D2 can be tested across cells with
dependence-corrected standard errors.

### Factors

| Factor | Levels | Purpose |
|---|---|---|
| Model | LLaMA-3.1-8B-Instruct (local); 2–3 closed models | generalisation |
| Role | proposer; responder | both sides of UG |
| Variant | V0, V-L, V-P, V-F, V-ALL (§2) | contamination (D2) |
| Game parameters | pie size, responder outside option, game type (§3) | comparative statics (D2) |
| Prompt paraphrase | 5 semantically equivalent rewordings per cell | dependence (D1) |
| Temperature | 0.0, 0.7, 1.0 | dependence (D1) |
| Seed | fixed list, logged per run | reproducibility, dependence (D1) |
| Context mode | fresh context per run (default); sequential within one session (auxiliary cell) | dependence (D1) |

Default context mode is **fresh context per run**: each decision is elicited in a new session
with no history. The sequential cell is run only for V0 and exists to show how much worse the
dependence problem becomes when runs share a context window — a practice that occurs in
published LLM-as-subject work.

### Elicitation

- **Proposer:** the model states an offer, as an amount out of the pie.
- **Responder:** the **strategy method** — the model states a minimum acceptable offer (MAO)
  rather than responding to a single offer. This yields a continuous response per run instead of
  a binary one, which materially reduces the number of runs needed per cell, and it matches a
  substantial part of the human UG literature. A smaller direct-response cell (accept/reject of a
  presented offer at 10%, 20%, 30%, 40%, 50% of the pie) is retained for comparability with
  studies that do not use the strategy method.

Output is requested in a fixed JSON schema (§5), with free-text reasoning optional and stored but
not used in the primary analysis.

---

## 2. Game variants (contamination instruments)

The canonical formulation carries at least three cues that could trigger retrieval of a memorised
answer rather than a decision: the **name** of the game and its roles, the **round pie** of 100,
and the standard **framing** of the instructions. The design perturbs each cue separately and then
all at once.

| ID | Name / roles | Pie | Framing | What it isolates |
|---|---|---|---|---|
| V0 | "Ultimatum Game", proposer / responder | 100 | canonical instructions | baseline (fully canonical) |
| V-L | no game name, "Player A" / "Player B" | 100 | canonical | label cue |
| V-P | canonical | 97 | canonical | round-number / anchor cue |
| V-F | canonical | 100 | neutral division, or workplace bonus allocation | framing cue |
| V-ALL | no name, A / B | 97 | neutral division | all three removed |

One-factor-at-a-time plus the combined cell is used instead of a full factorial: it identifies
which cue carries the canonical answer, and the V-ALL cell tests whether the effects are additive
or whether a single remaining cue is sufficient to restore the canonical response.

Two framings are prepared for V-F (abstract division of a sum; allocation of a team bonus between
two employees) so that a framing effect cannot be attributed to one particular wording.

**Manipulation check (critical).** Relabelling is only an instrument if it actually conceals the
paradigm. After the decision runs, the relabelled instructions (V-L, V-ALL) are presented to the
same models in a separate session with the question of whether this corresponds to a known
experimental paradigm and, if so, which one. If recognition is high, the relabelling failed, and
those cells cannot be used to separate recall from behaviour — this is reported, not hidden.

---

## 3. Comparative statics (behaviour instruments)

Parameters are varied along directions where the human evidence gives a clear *qualitative*
prediction. The test is directional movement, not agreement with a particular human mean — the
project does not claim that LLM output should match human magnitudes.

| Manipulation | Levels | Prediction if the model is responding to structure |
|---|---|---|
| Pie size | 10, 100, 1000 (same unit) | **relative** offer approximately stable; **absolute** offer scales with the pie |
| Responder outside option | 0, 10%, 30% of the pie | offers rise; responder MAO rises with the outside option |
| Game type | Ultimatum vs **Dictator** (responder cannot reject) | offers fall substantially under Dictator |
| Interaction horizon | one-shot anonymous vs repeated with the same partner | offers higher under repetition |

The pie-size and Dictator manipulations are the strongest instruments and are run on every model.
A model that has memorised "offer 50" as a number rather than "offer about half" as a decision
will fail the pie-size test by holding the absolute figure fixed, and a model reproducing the
canonical UG result without tracking the incentive structure will fail the Dictator test by
offering the same amount when rejection is impossible.

The outside-option and horizon manipulations are run on the local model in full and on closed
models only at the extreme levels, for cost reasons (§6).

---

## 4. Memory probes (recall channel)

Probes measure what is retrievable as *knowledge*, separately from play. Each probe is run 30
times per model, in a fresh context, with temperature fixed at 0.7.

1. **Open recall** — description of the game and of the results typically observed in it.
   Scored for: presence of a numeric modal offer, its value, presence of a rejection-rate claim.
2. **Source recall** — which studies report these results. Scored for correct attribution
   (Güth, Schmittberger & Schwarze 1982 and subsequent literature) and for fabricated citations,
   which are themselves informative.
3. **Recognition under relabelling** — the manipulation check from §2, scored as a binary plus
   the name produced.
4. **Counterfactual knowledge** — what the literature reports for the Dictator game and for
   high-stake replications. This tests whether the comparative-statics predictions are themselves
   retrievable as memorised facts, which would make correct statics weaker evidence of behaviour
   than it first appears. This is the probe most likely to complicate the story, which is why it
   is specified in advance.

Probe scoring is rule-based where possible (numeric extraction, string matching for author names)
with a manually coded sample of 50 responses per model to validate the rules; inter-coder
agreement is not applicable (single coder), so the coding rules are published in
`docs/probe-coding.md` and the coded sample is committed.

---

## 5. Run specification

### Request

Each run is one API/Ollama call with:

- a system message fixing the role and the output schema;
- a user message containing one paraphrase of the variant instructions;
- decoding parameters: `temperature` as per the grid, `top_p = 1.0`, `max_tokens = 512`,
  `seed` from the seed list (local model only — see the caveat below).

### Output schema

```json
{
  "role": "proposer",
  "decision": 40,
  "unit": "points",
  "confidence": 0.0,
  "reasoning": "optional free text"
}
```

Parsing rules: the first valid JSON object in the response is taken. A response is **invalid** if
no JSON object is present, if `decision` is missing or non-numeric, or if it falls outside
`[0, pie]`. Invalid responses are **not** resampled — the per-cell invalid rate is itself a
reported outcome, since silently retrying until a parseable answer appears is one of the practices
this project is arguing against.

### Logging

One record per run, appended to `data/raw/<date>_<cell-id>.jsonl`, containing: model identifier
and digest, variant ID, parameter point, role, paraphrase ID, temperature, seed, prompt hash,
full prompt, raw response text, parsed decision, validity flag, latency, timestamp, and the
config file hash. Raw files are never edited; all cleaning happens on the way to
`data/processed/`.

**Caveat on closed models.** Seeds are not honoured by all providers, provider routing may change
the backing hardware between calls, and model identifiers are not stable over time. Provider
pinning is used where OpenRouter allows it, and the model version string is logged with every
call. Where reproducibility cannot be guaranteed, this is reported as a finding about the
reproducibility of closed-model experiments rather than treated as a nuisance.

---

## 6. Sample sizes

Sample size cannot be fixed before the intra-source correlation is known, because the effective
sample size depends on it. The protocol therefore runs in two stages.

### Stage 1 — pilot

V0 and V-ALL, proposer and responder, local model only:
5 paraphrases × 3 temperatures × 20 runs = **300 runs per cell**.

From the pilot, estimate the variance components (§7) and the intra-source correlation $\rho$,
then compute the design effect for a planned cluster size $\bar{n}$:

$$\text{DEFF} = 1 + (\bar{n} - 1)\,\rho, \qquad N_{\text{eff}} = N / \text{DEFF}$$

### Stage 2 — main grid

Target: detect a shift of 5 percentage points in the mean relative offer between two cells.
Taking the pilot standard deviation $s$, the required *effective* sample per cell at 80% power
and $\alpha = 0.05$ is

$$n_{\text{eff}} \approx 16\,s^2 / \delta^2, \qquad \delta = 0.05$$

and the raw sample per cell is $n = n_{\text{eff}} \times \text{DEFF}$, allocated as
5 paraphrases × 3 temperatures × $n/15$ runs. With a plausible $s \approx 0.12$ and
$\rho \approx 0.15$ at $\bar{n} = 20$, this gives roughly 90 effective and **350–450 raw runs per
cell**; the exact number is fixed after the pilot and recorded here in version 0.2 of this
document before the main runs start.

Cells: 5 variants + 9 comparative-statics points + Dictator + repeated = 16 cells × 2 roles.
On the local model this is on the order of 12,000–14,000 calls, which at observed pilot latency
is feasible overnight in batches. Closed models are run on a **reduced grid** — V0, V-ALL,
Dictator, and the two extreme pie sizes, 100 runs per cell per role — to keep API cost bounded;
the reduced grid is sufficient for the generalisation claim, which is qualitative.

---

## 7. Analysis plan

### D1 — dependence

For each cell, a mixed model on the decision with random intercepts for paraphrase, temperature
block, and seed batch:

- variance decomposition into paraphrase / temperature / residual components;
- intra-source correlation and design effect, reported per model and per cell;
- effective sample size implied by *N* runs;
- side-by-side comparison of three standard errors for the same cross-cell contrast: naive i.i.d.,
  cluster-robust at the paraphrase level, and multilevel. The headline deliverable is the ratio of
  the naive to the corrected standard error — i.e. by what factor significance is overstated when
  runs are treated as independent participants;
- the sequential-context cell is compared against fresh-context to quantify the additional
  dependence introduced by reusing a session.

### D2 — contamination

- **Invariance:** distance between the decision distributions of V0 and each perturbed variant
  (difference in means with dependence-corrected intervals, plus a distributional comparison);
- **Statics:** sign and magnitude of the predicted movements in §3, estimated jointly with
  cell-level random effects;
- **Probes:** recall scores per model, and the manipulation-check recognition rate.

---

## 8. Decision rule (specified in advance)

Each model is placed in one row of the table below. No single instrument is decisive; the reading
comes from the combination.

| Invariant under relabelling | Statics in the predicted direction | Recall probes strong | Reading |
|---|---|---|---|
| yes | yes | either | Decisions respond to structure. Recall may be present but is not what produces the canonical result. |
| no | no | yes | Recall dominates: the canonical answer appears only when canonical cues are present. |
| yes | no | yes | Template reproduction: the canonical output survives relabelling but does not track incentives. The most damaging case for LLM-as-subject use. |
| no | yes | either | Framing sensitivity, **not** contamination. Humans are also not invariant to framing; this is reported as a framing effect, with the human framing literature as the comparison. |

Two caveats are to be stated in the report regardless of outcome:

1. If the manipulation check (§2) shows high recognition of the relabelled instructions, the
   invariance column is uninformative for that model and the conclusion rests on statics and
   probes alone.
2. If probe 4 shows that the comparative-statics predictions are themselves retrievable as
   memorised facts, correct statics are weaker evidence of behaviour than they appear, and the
   claim is narrowed accordingly.

---

## 9. What this protocol does not do

- No new human data is collected. Human benchmarks come from the published literature, and are
  used for *direction* of effects, not for calibration of levels.
- No claim is made about the models' internal states. "Behaviour" here means that decisions move
  with the incentive structure; nothing stronger.
- Market environments and zero-intelligence-agent ablations are out of scope and reserved for
  follow-up work.

---

## 10. Open questions for the supervisor

1. Strategy method as the primary responder elicitation — accepted, or should direct response be
   primary for comparability with the human benchmark literature?
2. Is the two-stage sample size (pilot → design effect → main grid) acceptable, or should a fixed
   *n* be committed to now?
3. Probe 4 (counterfactual knowledge) weakens the comparative-statics instrument by design.
   Keep it in the main protocol, or report it as a robustness section?
4. Reduced grid for closed models — are three models at the reduced grid preferable to two models
   at the full grid?
