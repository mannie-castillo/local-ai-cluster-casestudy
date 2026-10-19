# Methodology

This page describes the measurement discipline applied to every experiment. The point of all of it is the same: a result should be trustworthy by construction, so that a reader who disagrees with a conclusion can at least agree on what was measured.

## Isolation

Each node runs one model at a time. Nodes are 35-watt CPUs that share no CPU or memory, so a model's latency is its own, not a function of what another node was doing. Runs on one node are sequential; Experiment 2 and the Experiment 1 parallel check ran separate nodes at the same time. The control machine orchestrates but never runs inference.

Before a run, the node is **booked**. The booking is a lock that prevents any other job (including maintenance work such as model copies) from touching that node. After the run, the model is unloaded and the unload is **confirmed** by querying the inference server and checking that no model remains resident. Only then is the booking released. Every run record includes the unload confirmation and the release.

## Frozen artifacts

Test cases, answer keys, grading criteria, prompts and harness scripts are **frozen** before a run: written, hashed with SHA-256, and recorded. A freeze package is reviewed and approved before the first case is launched, and the package is backed up to a second node with its hashes verified file by file.

Two rules follow from freezing:

- **Unchanged-script rule.** Every approved script's hash is re-checked on disk immediately before it runs. A changed script is a new artifact and needs a new approval.
- **Key isolation.** Answer keys never appear in a model-facing prompt, are never sent to an external service, and are never shown to an AI grading agent. The grader is a deterministic script that compares model output to the frozen key.

## Conditions and design matrix

An experiment is designed as a full matrix (for R5: 6 cases x 4 models x 3 conditions = 72 results) and reconciled against that matrix afterward. Every expected result is given one disposition:

- completed and gradable;
- completed but needs human review;
- timed out (exceeded the per-request limit);
- cancelled (the harness stopped it, for example after a monitoring failure);
- not launched (the run was halted before reaching it);
- invalid.

A timeout is recorded as an operational completion failure, not as a wrong answer. Nothing is dropped silently, and nothing is rerun to improve a number; the three reruns (Experiment 1: the G3-4 14B run and the two-node speed test, both restarted after the original 80 C auto-stop; Operational Depth-1: G3-4 after a harness error) replaced stopped launches and are disclosed in their reports. The report carries separate denominators for attempted, completed, gradable and correct, because a model that is accurate when it finishes but often fails to finish is a different thing from a model that finishes and is sometimes wrong.

## Grading and adjudication

Grading is deterministic: token-boundary matching against the frozen key, with prohibited-answer checks (for false-premise and absent-answer cases) applied first, and abstention detected explicitly. Cases the grader cannot decide are routed to **human adjudication**. The operator sees the question, the model's response, the relevant source material and the frozen criteria, with the model's identity hidden, and rules PASS, FAIL or INVALID. Model identities are unblinded only after the ruling is recorded. No AI agent substitutes for that decision, with one disclosed exception: in Experiment 2 the six borderline cases were ruled by an AI reviewer blind to model identity and without key access. That deviated from the frozen protocol; I ratified it once after an independent AI review re-checked all six rulings, and R5 used human adjudication as specified.

## Safety and monitoring

Every run operates under a thermal and resource policy (in the 7 October Experiment 1 runs, temperatures were read only on G3-2 and G3-4). These CPUs report a maximum of 80 C and sit there under sustained load, so the policy records time at that ceiling rather than pretending a higher stop threshold could fire. A per-request limit of 300 seconds applies uniformly to every model in an experiment; it is never relaxed for one model to improve its score, because the limit is part of what is being measured.

The harness is designed to **fail closed**. During Experiment 2, a monitoring check failed mid-run on one node. The harness cancelled the in-flight case, confirmed the model was unloaded, released the booking and stopped without an automatic retry. The cancelled case was excluded from scoring rather than counted as wrong. That event is reported as a result: it is evidence the safety discipline works.

## Evidence and backup

Raw outputs, run logs, scorecards, reconciliation records and adjudication decisions are written to a protected evidence root and then backed up to a second node (except the 7 October Experiment 1 raw files; see Evidence exceptions). The backup is verified by file count, byte count and SHA-256 per file, and a dated receipt is produced (the R5 and Operational Depth-1 pilot backups have console logs, not receipts). Backups of earlier experiments are never overwritten.

Before any experiment writes to a shared service (for example, creating a test document collection in the retrieval host), that service's database is backed up and the backup verified, a rollback is rehearsed, and the production collections are checked afterward to confirm they are unchanged.

## Evidence exceptions (disclosed 18 October 2026)

On 18 October 2026 every reachable evidence file was re-hashed against the checksum lists written when each run was recorded. Apart from expected changes (a regenerated Python cache file, a runtime configuration file, and the documented grader replacement), the mismatches and process exceptions found are listed below. They are disclosed rather than corrected:

- **Experiment 2 protocol document.** It was modified 4.6 minutes after its Freeze-1 hash was recorded and about 11 minutes before the first pilot run. No record explains or approves the change, and the frozen version is not preserved on the control machine. The test cases, the private grading key and the run matrix still match their frozen hashes. The grader was replaced before the core run by a hardened version, which was documented and reviewed.
- **Experiment 2 adjudication queue.** The checksum recorded for the adjudication queue file does not match the file, whose timestamp is earlier than the checksum list. The cause is unknown. The outcome of the six rulings (all PASS) is also recorded in the results report.
- **Experiment 1, 7 October runs.** The raw per-answer files for the Qwen3 4B, Qwen3 14B and gpt-oss 20B runs, and the cascade inputs, are on an external drive that was offline during this check. Those rows trace to the run report, which was reviewed when it was written, but they could not be re-hashed, and no second-node copy was found (the recovered copy of the G3-6 evidence mirror does not contain them; G3-6 itself was not inspected on 18 October).
- **Operational Depth-1 run record.** It was edited after its evidence manifest was written; 83 of the 85 files in that manifest match (the two that differ are the edited run record itself and a regenerated Python cache file).
- **Pre-registration.** Experiment 1 had no standalone pre-registered protocol; its question set was frozen and hashed before the run. Operational Depth-1's design was declared in its run record before launch, not frozen as a separate package. The freeze-package description above applies in full to Experiment 2 and R5.
- **R5 adjudication packet.** The script that rendered the blinded adjudication packet read the private key before its review gate. A retrospective key-isolation review was recorded.
- **Backups.** The second-node backups of the Operational Depth-1 and R5 pilot evidence predate later edits to their run records and have not yet been refreshed; a refresh needs the cluster on and its own approval.

## Review gates

A separate reviewing agent, with read access only, checks freeze packages, scripts, launches and results at each gate before the next step proceeds. It approved or blocked steps at each gate and caught several defects, including a capacity calculation that mixed units and a test case whose distractor accidentally equalled another case's correct answer. The operator approves every change to the environment and every live inference run. The reviewing agent is itself an AI agent, and AI coding agents also wrote the harnesses and graders and ran the tests; the README section on how AI agents were used gives the division of work. Design-only work (planning the next experiment) proceeds without touching any node.

## What this discipline does not do

It does not make small samples large. Six or twelve cases per model surface findings and validate a method; they do not support confident rankings. The reports say so wherever it applies, and avoid rankings where the data does not justify them.
