# codeRED

**Built in 3 hours at the UBS Innovation Challenge — Scenario 1: Ticket Intelligence and Theme Detection**

> From noise to insight: helping risk teams understand what's really going wrong.

codeRED turns a noisy stream of operational tickets into evidence-backed themes and
root-cause candidates — without letting anything, human or model, claim a cause it can't
prove. The core bet: a deterministic engine decides what the data actually supports, and
anything uncertain gets routed to a human instead of guessed at.

## The problem

Risk and ops teams drown in tickets. The expensive failure mode isn't "we didn't notice
the pattern" — it's "we noticed a pattern and confidently blamed the wrong cause." Any
system that auto-summarizes incidents has to resist the temptation to sound certain about
things it can't actually verify.

## What this does

1. **Clusters tickets into themes** using normalized text matching against a controlled
   rule book (keywords + system) — not a black box, every match is traceable to a term in
   the source text.
2. **Correlates each cluster against a real change/deployment log**, only accepting a
   change as candidate evidence if it matches on system, region, timing, and text overlap.
3. **Grades its own confidence** and routes accordingly:

   | State | Cluster score | What it's allowed to say | Route |
   |---|---:|---|---|
   | `KNOWN` | ≥ 0.60 | Describe the pattern and cite ranked evidence | Evidence review |
   | `KNOWN-VARIANT` | 0.30–0.59 | Describe the pattern, but not as a certainty | Human review |
   | `UNKNOWN` | < 0.30 | Describe the tickets only — no cause claim allowed | Human expert, always |

4. **Never touches ground truth.** Every field prefixed `_` (including the evaluation
   label `_gt_theme`) is stripped before the engine ever sees the data. What it finds, it
   finds from title/description/system/region/time alone.

The design intent is a **closed-world grounding contract**: an LLM layer could sit on top
of this to turn verified clusters into prose, but it would only ever be allowed to narrate
evidence the deterministic layer already proved — never invent a cause, an ID, or a system
that isn't in the evidence pack. That validator/LLM layer is the natural next step and is
specced in [`docs/GROUNDING_SOLUTION.md`](docs/GROUNDING_SOLUTION.md); what ships in this
repo today is the grounding engine itself, tested end to end.

## Try it

```bash
python scripts/grounded_cluster.py \
  --tickets data/tickets.json \
  --changes data/changes.json \
  --out data/grounded_clusters.json
```

No dependencies beyond the Python standard library — `python3 --version` and you're
running. Point `--tickets` at any of the ten scenario files in `data/` to see a specific
case (known pattern, novel pattern, recurring pattern, singleton noise, etc).

### Run the test suite

```bash
python -m unittest discover -s tests/grounding -p 'test_*.py' -v
```

Ten scenarios, ten passes. Each one asserts the engine's *behavior*, not just its output —
e.g. the novel-IBAN scenario must land in `UNKNOWN` with confidence capped at `0.30` and
`cause_claim_allowed: false`, and the singleton-noise scenario must refuse to collapse
unrelated tickets into one fabricated incident. Full results in
[`tests/grounding/GROUNDING_TEST_REPORT.md`](tests/grounding/GROUNDING_TEST_REPORT.md).

### Regenerate the synthetic dataset

```bash
python gen/generate.py
```

Deterministic (seeded), so regenerating reproduces the same 1,401 tickets and 40 change
records every time.

## Why the confidence math is strict

```text
keyword_score         = matched rule terms / total rule terms
system_score          = 1.0 if ticket.system is in the rule's allowed systems else 0.0
ticket_category_score = 0.7 * keyword_score + 0.3 * system_score

evidence_strength     = min(1.0, accepted_candidate_changes / 2)
evidence_confidence    = 0.5 + 0.5 * evidence_strength
confidence             = min(cluster_category_score, evidence_confidence)
```

Good keyword similarity alone can't buy high confidence — it's capped by how much real
supporting evidence (a matching, in-window, same-system change record) actually exists.
An `UNKNOWN` cluster is hard-capped at `0.30` confidence regardless of how clean the text
match looks, specifically so a well-written but unprecedented ticket can't sneak into an
automated path by sounding familiar. Full derivation in
[`docs/GROUNDING_SOLUTION.md`](docs/GROUNDING_SOLUTION.md).

## The synthetic dataset

[`data/tickets.json`](data/tickets.json) and [`data/changes.json`](data/changes.json)
simulate a 90-day operational ticket stream with planted signal buried in realistic noise:

- **1,401 tickets**, **40 change records**, 90-day window
- 3 planted root-cause stories (a high-volume login regression, a recurring APAC FX
  mismatch, a low-volume novel IBAN validation bug) layered over 7 background issue types
  and a bucket of genuine one-off singletons
- The change log includes decoys and innocent nearby changes on purpose — correlation has
  to survive noise, not just succeed in a clean room
- The novel IBAN case is deliberately hard for keyword matching, to prove the engine
  escalates instead of forcing a fit

Schema and generation details: [`docs/SCHEMA.md`](docs/SCHEMA.md) ·
[`docs/LANE_B_DATA.md`](docs/LANE_B_DATA.md).

## Project layout

```
scripts/grounded_cluster.py   the grounding + clustering engine (stdlib only)
gen/generate.py               deterministic synthetic ticket/change generator
data/                         the dataset + 10 scenario slices used by the tests
tests/grounding/              behavioral test suite, one file per scenario
docs/                         design docs: schema, grounding contract, scoring, demo script
```

## Background

Built solo-team in a 3-hour hackathon sprint against the brief in
[`Case_information.md`](Case_information.md). [`docs/PROJECT_SUMMARY.md`](docs/PROJECT_SUMMARY.md)
is the longer writeup of the workflow and the human/AI division of labor; the original
build plan (including the originally-scoped Streamlit dashboard and LLM narration layer)
is in [`PLAN.md`](PLAN.md) for anyone who wants to see where this goes next.
