# Operational Depth-1: demanding practical tasks (pilot)

**Date:** 13 October 2026.
**Status:** pilot complete, independently verified, evidence backed up (the backup predates a later record edit; see methodology.md). Exploratory: one scenario per track.

## Question

Experiments 1, 2 and R5 measured recall, reasoning and document grounding. This one asks the harder operational question: can the models complete difficult multi-step work, recognise when a task cannot be done, and recover when something fails, rather than produce a convincing answer?

## Design

Three task families, each one scenario, run on all four models, one node and one model at a time:

| Track | Task | What counts as success |
|---|---|---|
| D1, complex planning | Produce a schedule that satisfies 12 interacting constraints | Every constraint satisfied, checked by a deterministic verifier. An exhaustive checker established beforehand that the problem was feasible |
| D2, incident diagnosis | Diagnose an incident from evidence that partly conflicts, then revise when new evidence arrives | The correct cause identified after the revision |
| D3, workflow recovery | One synthetic workflow containing three faults has failed in a simulator; diagnose and fix all three, with up to three corrective attempts | Full recovery verified by the simulator within the three-attempt budget |

Every success was independently verified by a deterministic check, not by reading the model's own account of what it did. The primary metric is **Verified Task Resolution Efficiency (VTRE)**: verified successful tasks per minute of wall-clock time.

Models: Qwen3 14B on G3-2 (16 GB DDR4-2400), Hermes 3 8B on G3-4 (32 GB DDR4-2400), Llama 3.1 8B on G3-5 (16 GB DDR4-2400), Gemma 3 12B on G3-6 (24 GB DDR4-2400).

## Results

All 12 sessions (4 models x 3 tracks) completed. No timeouts, no safety stops, every node confirmed unloaded and released. One earlier launch on G3-4 stopped on a harness request error before any generation; the harness was fixed under review and rerun, and the failed launch is recorded and excluded. Reviewed fixes were also made to the D3 simulator during the pilot.

| Model | D1 planning | D2 diagnosis | D3 recovery | Verified tasks per minute |
|---|---|---|---|---|
| Llama 3.1 8B | FAIL | PASS | PASS, second attempt | **0.57** |
| Qwen3 14B | FAIL | PASS | PASS, first attempt | 0.35 |
| Hermes 3 8B | FAIL | PASS | FAIL, three attempts | 0.25 |
| Gemma 3 12B | FAIL | PASS | FAIL, three attempts | 0.14 |
| **Total** | **0 of 4** | **4 of 4** | **2 of 4** | **6 of 12 in 20.3 minutes, about 0.30 per minute** |

![Operational Depth-1: verified tasks per minute](../figures/opd1-vtre.svg)

## Findings

**D1: all four models failed the same feasible scheduling problem.** None satisfied all 12 constraints. The problem was not impossible; the exhaustive checker had confirmed it was feasible. This is the most important result in the project so far, and the right reading of it is architectural rather than a verdict on the models. Exact multi-constraint satisfaction is a search problem, and the result suggests that a language model generating free text is the wrong instrument for it, in the same way that a person reasoning aloud is the wrong instrument for solving a 12-constraint timetable in their head, and that a deterministic solver is the right one. Two limits on that reading: one failed case does not establish that larger models cannot solve it, and the complete model-plus-solver workflow has not been tested yet. The result supports investigating that architecture; it does not yet prove it.

**D2: every model handled the diagnosis.** All four identified the actual cause after correctly revising their initial diagnosis when the new evidence arrived. This is consistent with Experiment 2, where every model correctly rejected a false premise, and with R5, where the models handled retrieval well. These models are good at taking in new evidence and updating.

**D3: the models separated on recovery, but on one workflow.** D3 was a single synthetic workflow containing three faults, not three independent scenarios, and each model had up to three corrective attempts. Qwen resolved all three faults on its first attempt. Llama succeeded on its second attempt. Hermes and Gemma did not complete the recovery within three attempts. One workflow is not enough to establish general recovery reliability, so this is a signal, not a ranking. It is the track most worth expanding.

**The efficiency column is the practical one.** Llama produced the most verified work per minute because it succeeded on two tracks and is fast. Qwen succeeded on the same two tracks but at roughly half the throughput, because it is a larger model on a 16 GB DDR4-2400 node (G3-2). The two 8B models are not interchangeable: Llama recovered the workflow and Hermes did not.

## What this changes about the architecture

Before this pilot the routing question was which model should answer a request. D1 shows that for rule-governed tasks the question is which model plus which verified tool. The architecture this points at:

1. The model interprets the request and extracts the constraints into a structured form.
2. A deterministic solver computes a plan that satisfies them, and an independent verifier checks it.
3. The model explains the verified result without altering it.

In that design the solver guarantees constraint satisfaction whenever the constraints were extracted correctly and the solver represents them faithfully. The remaining risk moves to step one: misunderstanding or omitting a requirement. That is where a larger model may add measurable value, and it is now the central question for Experiment 3.

## Qualifications

- One scenario per track. D1 and D2 are single cases run across four models, and D3 is one workflow with three faults. The findings are meaningful and preliminary, and no model ranking should be drawn from them.
- D1 is an observed limitation on one 12-constraint problem and a design opportunity. It is not proof that models of this size generally cannot plan.
- The figures here were cross-checked on 14 October 2026 against the preserved 12-session scorecard (SHA-256 B63F699A...5EE16C) and match it exactly. The evidence backup (86 files, including the manifest that lists the other 85) is hash-verified on G3-6 (24 GB DDR4-2400); this copy predates a later edit to the run record and has not been refreshed (see Evidence exceptions in methodology.md).

## What follows

- Experiment 3 is being redesigned around this finding: score constraint extraction, constraint solving and explanation as three separate capabilities, and compare smaller and larger models on extraction before attributing any pipeline gain to model size.
- D3 should have at least three independently validated recovery scenarios per model, covering distinct failure modes, before stronger conclusions are drawn about recovery reliability.
- The original 12-session scorecard, cases and evidence stay immutable. They will not be rescored because a different architecture would have solved D1.
