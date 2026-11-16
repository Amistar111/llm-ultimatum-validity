# Prompts

Versioned prompt material for the experiment. Nothing here is edited in place once runs have
been executed against it: a change means a new `id`, so that every logged run can be traced to
the exact text that produced it.

## Files

| File | Contents |
|---|---|
| `base.yaml` | system frame, placeholders, response formats, parsing rules |
| `variants.yaml` | game variants V0 / V-L / V-F / V-ALL and their paraphrases |
| `conditions.yaml` | modifiers applied on top of a variant: game type, stake, anonymity, earned role, horizon, identity |
| `probes.yaml` | memory probes (knowledge elicited outside of play) |

## Identifiers

Every elicitation is identified by a triple that is written into each log record:

```
<variant>.<role>.<paraphrase>     e.g.  v0.proposer.p03
```

plus the condition ids applied to it, e.g. `+stake:1000 +anon:high`.

## Paraphrases

Paraphrases are the clustering unit of the dependence diagnostic: the intra-source correlation
is estimated across them, so the number of paraphrases, not the number of runs, is what buys
precision. Target is **15 per role per variant**; the sets below are the start of that work and
are extended before the pilot.

Rules for writing one: the strategic content, the payoff numbers, the one-shot and anonymity
statements, and the response format must be identical. Only sentence order, register, and
non-strategic wording may change. A paraphrase that adds, removes, or re-weights information is
a new condition, not a paraphrase.

## Sources

Canonical texts are reconstructions of the procedures of the cited studies, written in the
author's own words and citable as "adapted from". Exact original instructions are in the
appendices of the cited papers. Prompts marked `origin: own` are not reconstructions of any
published procedure.
