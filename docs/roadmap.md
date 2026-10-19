# Roadmap

Four completed experiments answer "what can these models do on this hardware" at the level of recall, reasoning, document grounding and demanding practical work. The remaining experiment asks whether larger models, or a model-plus-solver workflow, overcome the weakness the fourth one found. All of it feeds one deliverable: an evidence-based routing system that chooses not only which model answers a request but which verified tool or workflow the task requires.

## Where things stand

| Workstream | Status | Next milestone |
|---|---|---|
| Experiment 1, bake-off | Complete | None |
| Experiment 2, reasoning | Complete, exploratory | None |
| R5, document intelligence | Pilot complete | 24-case held-out evaluation, deferred pending decision |
| Operational Depth-1 | Pilot complete, verified, exploratory | Expand D3 to three or more validated recovery scenarios per model (folded into Experiment 3) |
| Experiment 3, larger models and model-plus-solver | Protocol drafted, awaiting operator review; not run | Resource-fit trial on G3-1 (32 GB DDR4-2400), then the three-capability comparison |
| Model preservation | In progress; last Dell transfer interrupted, not yet accepted | Resume the Dell transfer, then the final acceptance ledger |
| Control-machine independence from the staging drive | Discovered, not done; of 37 references, 32 repointable, 4 gated, 1 historical (no change needed) | Approve the gated items, then a controlled drive-absent test |

## What Operational Depth-1 established

The pilot ran all three tracks on all four models: 12 sessions, 6 independently verified successes, no timeouts or safety stops. Full report in [operational-depth-1.md](operational-depth-1.md). Three things carry forward:

- **All four models failed the same feasible 12-constraint schedule** (a deterministic checker had proved it feasible). The evidence supports investigating a model-plus-solver workflow: the model extracts the constraints, a deterministic solver computes and a verifier checks the plan, and the model explains the verified result. This is an observed limitation on one problem and a design opportunity, not a general claim about models of this size, and the workflow itself has not been tested yet.
- **All four models handled incident diagnosis and revision.** Consistent with every earlier experiment: these models update well on new evidence.
- **Recovery separated the models on one workflow with three faults** (Qwen fixed all three on its first attempt, Llama succeeded on its second, Hermes and Gemma not within three). One workflow, not three independent scenarios. A signal to expand, not a ranking.

The original 12-session scorecard, cases and evidence are immutable. They will not be rescored because a different architecture would have solved D1.

## Experiment 3: larger models, and the model-plus-solver workflow

The central design question, set by Depth-1: for rule-governed tasks, does a larger model help, does a solver help, or both? The experiment separates three capabilities and scores each independently as well as the end-to-end task:

1. **Constraint extraction.** Can the model translate natural-language requirements into correct structured constraints, including dependencies, prohibitions and exceptions? This is the step where the model adds value and where it can silently get a requirement wrong. Smaller and larger models are compared on extraction before any pipeline improvement is attributed to model size.
2. **Constraint solving and verification.** Can a deterministic solver produce a feasible plan from those constraints, and can an independent verifier validate it? The solver guarantees satisfaction whenever extraction was correct and the solver represents the constraints faithfully.
3. **Explanation.** Can the model accurately explain the verified solution without changing or misrepresenting its requirements?

Three arms are compared: smaller models attempting difficult planning directly; larger models attempting the same problems directly; and extraction plus verified solver plus explanation. The original 12-constraint D1 case is a starting diagnostic, not fresh held-out material. Additional independently validated planning cases at graded difficulty, and at least three validated D3 recovery scenarios per model covering distinct failure modes, are prepared before the comparison runs.

Candidates in the 20B to 32B range, each with a hash-verified staged copy on the cluster (gpt-oss 20B and DeepSeek-R1 32B were recorded on G3-1 in the 6 October inventory, to be re-read before use; the others would be placed there under their own approval): gpt-oss 20B, DeepSeek-R1 32B, Mistral Small 24B, Qwen3 30B-A3B, and optionally Qwen3.8 27B. The test node is G3-1 (32 GB DDR4-2400), with matched smaller-model baselines run on the same node so hardware differences cannot be mistaken for model differences. An existing baseline informs the feasibility plan: gpt-oss 20B already ran on G3-1 (32 GB DDR4-2400) in Experiment 1, scoring 27 of 40 at 6.6 tokens per second, so it is a known-feasible first candidate for the resource-fit phase rather than a first attempt at a large model on that machine.

Phases, each with its own approval gate:

1. **Readiness.** Read-only assessment of the node's memory, storage, thermal telemetry and software state, using existing records.
2. **Resource fit.** One model at a time: does it load, how much memory does it use, does it page, what is its practical context ceiling, how fast does it generate. A short bounded run, then a verified clean stop of the inference process (the machine is never shut down or restarted). No benchmarking yet.
3. **The three-capability comparison** on the planning and recovery cases, small and large models on the same node.
4. **Practical value.** Verified task completion per wall-clock minute for every arm. The question is not whether the big model is more accurate but whether it, or the solver workflow, completes more verified work per minute, or enough more correct work to justify being several times slower.

Two constraints shape this experiment. The test node cannot be safely rebooted, so the plan uses a portable, non-installed inference runtime run as a temporary process, which must be shown to start, stop and clean up without touching the existing installation. And the node currently lacks the thermal logging the monitoring policy requires, which must be resolved before any monitored inference.

Generation speed for 20B to 32B models on this node is not yet measured; the resource-fit phase measures it. The resource-fit phase exists to find out which of these models are usable at all before investing a benchmark in them.

## The routing system

The destination. Based on what has been measured so far, the shape is:

- a fast small model (the 4B model in Experiment 1, the 8B models in Experiment 2), where it was among the fastest and most accurate on those sets, as the default for straightforward requests;
- escalation to a 12B or 14B model only for request types where a live cascade test shows it pays (the Experiment 1 cascade was an upper bound);
- the retrieval pipeline for anything that should be grounded in the private corpus;
- a model-plus-solver workflow for exact planning and scheduling, because no model tested did it alone;
- a larger model for the hardest reasoning, or for constraint extraction, only if Experiment 3 shows a measurable gain in verified work;
- and a human approval step for tasks where a wrong answer matters, because the models did not resolve the one conflicting-evidence case tested.

Each rule in that table should trace to a measured result. That is the standard the whole project is held to.

## Model preservation

A side workstream, in progress and not yet accepted (status verified through 14 October 2026, 06:45). The goal is that every operational model has at least one hash-verified copy on a cluster node other than the original download drive, with a cold archive on the Dell (16 GB DDR4-2400) holding the models that had no other durable copy. Most of that is done; the last transfer to the Dell was interrupted at 17 of 83 files (149.3 of 157.2 GB) by an unplanned operating-system restart of the control machine, and it resumes once a free-space minimum on the Dell is decided. The final acceptance ledger, which will list every model, where its verified copies are, and the six models on hold (five pending licence review, one by operator decision), has not been produced yet. Until it is, the original download drive was to stay connected and intact; it was not mounted during the 18 October re-hash. A related sweep found 37 active references to that drive on the control machine and none on the cluster nodes inspected (G3-1 was not inspected, by rule); 32 can be repointed with reversible, hashed changes, 4 are gated pending approval, and 1 is a historical record that only documents a past location and needs no change (32 + 4 + 1 = 37). Independence from the drive is claimed only after a controlled drive-absent test passes.
