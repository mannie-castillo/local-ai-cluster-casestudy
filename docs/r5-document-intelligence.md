# R5: document intelligence (calibration pilot)

**Date:** 13 October 2026.
**Status:** pilot complete and closed. A 24-case held-out evaluation is designed and deferred.

R5 is the internal name for the document-intelligence track. It is the experiment that the ceiling effect in Experiment 2 was designed to fix.

## Question

Can the models answer questions from a private document collection through a real retrieval pipeline, cite what they used, recognise when the answer is not in the collection, and resolve two documents that disagree? And when they fail, is the retrieval or the reasoning at fault?

## Design

**The corpus.** Seventeen fictional documents, written for this experiment. Because the facts are invented, no model can know the answers from training. Any correct answer has to come from the documents, and any confident answer to a question the documents do not cover is a hallucination by construction.

**The cases.** Six calibration cases: five with an answer in the corpus (including one where two document versions conflict and the model must reconcile them) and one whose answer is absent, where the correct response is to abstain.

**Three conditions**, so that retrieval and reasoning can be separated:

| Condition | What the model gets | What it isolates |
|---|---|---|
| C1 | The question only, no documents | The baseline: it should abstain, since the facts are fictional |
| C2 | The question plus the correct supporting passages, supplied directly | Reasoning with retrieval removed |
| C3 | The question, answered through Open WebUI's retrieval over the full corpus | The real workflow, retrieval included |

**The models.** Hermes 3 8B on G3-4 (32 GB DDR4-2400), Gemma 3 12B on G3-6 (24 GB DDR4-2400), Llama 3.1 8B on G3-5 (16 GB DDR4-2400), Qwen3 14B on G3-2 (16 GB DDR4-2400). Retrieval in C3 ran on G3-3 (16 GB DDR4-2133). One model per node, run sequentially, 300-second per-request limit, 384-token output cap on the direct calls (C1 and C2).

**Grading.** Deterministic against a frozen key: token-boundary matching, prohibited-answer checks first, explicit abstention detection. Borderline cases to blind human adjudication.

## Accounting for all 72 results

The design is 6 cases x 4 models x 3 conditions = 72. Every one has a disposition:

| Disposition | Count | Detail |
|---|---|---|
| Completed | 67 | |
| Timed out | 1 | Qwen3 14B, one C3 case, exceeded 300 s |
| Not launched | 4 | Qwen3 14B, C3, the remaining C3 cases, run halted after the timeout |

This was verified against the run manifests rather than assumed. The timeout is recorded as an operational completion failure, not as a wrong answer, and the four unlaunched cases are recorded as unlaunched. The denominators below are stated accordingly.

## Results

Correct answers over the five answerable cases:

| Model | C2, evidence supplied | C3, real retrieval | C3 cases completed |
|---|---|---|---|
| Hermes 3 8B | 4 of 5 | 4 of 5 | 6 of 6 |
| Gemma 3 12B | 4 of 5 | 4 of 5 | 6 of 6 |
| Llama 3.1 8B | 4 of 5 | 3 of 5 | 6 of 6 |
| Qwen3 14B | 4 of 5 | 1 of 5 | 1 of 6 |

The abstention case and the C1 baseline are reported in the full review and are not folded into the counts above. The two Qwen C3 figures must be read together: 1 of 5 answerable cases correct because only 1 of its 6 C3 cases completed. Correctness and completion are separate measures, and a model that is right when it finishes but rarely finishes is a different thing from one that finishes and is sometimes wrong.

## Findings

**1. Retrieval is not the weak link.** When the correct passages were handed over (C2), all four models scored 4 of 5. Through real retrieval (C3), Hermes and Gemma scored the same 4 of 5, and Llama dropped one case. In C3, every completed answer to an answerable case (16 of 16) had at least one supporting passage among the retrieved sources (9 with an unparsed passage id), and no answer in C2 or C3 cited a passage outside the supporting or superseded set.

**2. Reasoning over conflicting evidence is.** The conflict case, two document versions that disagree, was missed by all four models in C2 and by all three that attempted it in C3 (Qwen's C3 run never reached it). No model reconciled the two versions even when both passages were placed directly in front of it. That rules out retrieval as the cause and points at the models' handling of contradiction. The cause is not yet determined; it may be prompting, model capability, or the construction of the case itself. It is the first target for the held-out evaluation.

**3. The largest model was the least practical on this path.** Open WebUI's retrieval workflow makes two model calls per question: it first asks the chat model to rewrite the question into a search query, then retrieves, then asks the model to answer. For the 14B model on a 16 GB DDR4-2400 node (G3-2), that two-call sequence did not fit inside 300 seconds. Its accuracy on the one case it completed is not the point. A model that cannot finish the workflow in the allowed time is not useful on that workflow, whatever its accuracy when it does finish. This is recorded as finding F-C3-1, and the 300-second limit was kept uniform for all models rather than relaxed for one.

**4. The output cap held where it applied.** No C1 or C2 answer reached the 384-token output cap; C3 calls went through Open WebUI with its own parameters, so the cap did not apply there. The longest completed answer in any condition was 114 tokens. That supports the cap for this pilot without proving it sufficient for longer held-out questions.

**5. C3 timing is a different measurement from C1 and C2.** Because C3 is two model calls plus retrieval, its end-to-end latency is the real cost of the document-grounded workflow and is reported separately. It should not be read as a single-call inference time.

## Human adjudication

Six cases went to blind adjudication. The operator saw the question, the response, the source passages and the frozen criteria, with model identity hidden, and ruled on each. Identities were unblinded only after all rulings were recorded.

| Model | Reviews | Rulings |
|---|---|---|
| Hermes 3 8B | 2 (one C2, one C3) | 2 PASS |
| Gemma 3 12B | 1 (C3) | 1 PASS |
| Llama 3.1 8B | 3 (one C2, two C3) | 2 PASS, 1 FAIL |
| Qwen3 14B | 0 | |

Five of the six concerned the same case. That concentration says its wording sits close to the grading boundary, and it should be reviewed before the held-out run. Per-case rulings are kept in the private record so the cases stay reusable.

## Operational record

- The retrieval host's database was backed up and verified (504 of 504 checksums equal) before the test collection was created; a rollback was rehearsed first.
- The test collection was created with exactly the 17 fictional documents and isolated from the three production collections, which were verified unchanged afterward.
- A temporary, reversible change exposed two models to the retrieval host for C3; the original settings were restored and the restoration verified byte for byte.
- Pilot evidence (68 files) and the adjudication supplement (18 files) were backed up to G3-6 (24 GB DDR4-2400) with every file's SHA-256 verified; this copy predates a later edit to the run record and has not been refreshed (see Evidence exceptions in methodology.md).

## What the pilot settled, and what it did not

It settled that the method works end to end: fictional corpus, three isolating conditions, deterministic grading, blind adjudication, full-matrix reconciliation and verified evidence. It produced three findings that change what to do next.

It did not rank the models, and this report does not. Six cases are a calibration run. The 24-case held-out evaluation exists for ranking, and is deferred pending a separate decision.

## Recommended for the held-out run

- Probe the conflict-case failure first; it is the clearest signal in the pilot.
- Keep the uniform 300-second limit so completion rate stays a measured outcome.
- Decide deliberately whether the 14B model gets a longer budget or a single-call retrieval path, rather than quietly changing either.
- Review the wording of the case that drew five of the six adjudications.
- Report C3 latency broken out by query rewriting, retrieval and generation where the instrumentation allows.
