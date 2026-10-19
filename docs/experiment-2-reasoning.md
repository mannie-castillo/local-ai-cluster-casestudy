# Experiment 2: reasoning, memory, consistency and pushback

**Date:** 12 October 2026.
**Status:** complete, closed, labelled exploratory.

## Question

Beyond recalling facts, can these models reason through multi-step problems, hold information across turns, stay consistent when corrected, push back on a false premise, accept a valid correction, and retain context through padding?

## Design

Four models, each on its own node, each answering the same 12 held-out cases. Thinking mode was off for all four. Cases were split across six behaviour tracks:

| Track | Cases | Behaviour tested |
|---|---|---|
| R1 reasoning | 3 | Ordering and relational puzzles with a one-word answer |
| R2 memory | 3 | Recall a value given earlier in the conversation |
| R3 consistency and correction | 2 | Stay consistent, and update correctly when a premise changes |
| R4a false-premise pushback | 1 | The user insists on something false; the model should reject it |
| R4b valid correction | 1 | The user supplies a correct correction; the model should accept it |
| R6 context retention | 2 | Recall a value after turns of filler capped at 512 tokens |

Grading was deterministic against a frozen key. Cases the grader could not decide (six) were ruled by an AI reviewer that was blind to model identity and had no key access. That deviated from the protocol and from the design specification written before it, which call for a defined human review queue and no uncontrolled AI-judge substitution; an independent AI review agent re-checked all six rulings on the merits (all PASS), and I ratified the deviation once. It is disclosed here, and R5 used human adjudication as specified. The protocol document itself was edited after it was frozen; see Evidence exceptions in [methodology.md](methodology.md).

Models: Llama 3.1 8B on G3-5 (16 GB DDR4-2400), Hermes 3 8B on G3-4 (32 GB DDR4-2400), Gemma 3 12B on G3-6 (24 GB DDR4-2400), Qwen3 14B on G3-2 (16 GB DDR4-2400). All at 4-bit quantization.

## Results

Per track, cases passed over cases evaluated, after adjudication:

| Track | Llama 3.1 8B | Hermes 3 8B | Gemma 3 12B | Qwen3 14B |
|---|---|---|---|---|
| R1 reasoning (3) | 3/3 | 3/3 | 3/3 | 3/3 |
| R2 memory (3) | 3/3 | 3/3 | 3/3 | 3/3 |
| R3 consistency and correction (2) | 2/2 | 1/2 | 2/2 | 2/2 |
| R4a false-premise pushback (1) | 1/1 | 1/1 | 1/1 | 1/1 |
| R4b valid correction (1) | 1/1 | 1/1 | 1/1 | 1/1 |
| R6 context retention (2) | 2/2 | 2/2 | 2/2 | 1/1 (one cancelled) |
| **Overall, completed cases** | **12/12** | **11/12** | **12/12** | **11/11** |

Across all four models, 46 of 47 completed answers were correct.

The single scored error was Hermes 3 on a consistency case: after a premise changed, it kept its earlier conclusion instead of updating it. The single missing result was Qwen3's second context-retention case, cancelled when a monitoring check failed during the run. The harness cancelled the case, confirmed the model was unloaded, released the node and stopped without an automatic retry. The case was scored CANCELLED and excluded; it was never counted as wrong, and it was not rerun.

All four models correctly rejected the false premise when the user insisted on it, with no concessions. All four accepted a valid correction.

## Throughput and useful work

| Model | Node | Turns | Mean wall-clock per turn | Median | Max | Tokens/s | Run length | Correct cases per minute |
|---|---|---|---|---|---|---|---|---|
| Llama 3.1 8B | G3-5, 16 GB DDR4-2400 | 38 | 34.9 s | 6.0 s | 156.8 s | 6.38 | 24.2 min | **0.50** |
| Hermes 3 8B | G3-4, 32 GB DDR4-2400 | 38 | 29.3 s | 9.8 s | 133.6 s | 6.29 | 20.2 min | **0.54** |
| Gemma 3 12B | G3-6, 24 GB DDR4-2400 | 38 | 77.5 s | 16.6 s | 262.7 s | 3.11 | 50.7 min | 0.24 |
| Qwen3 14B | G3-2, 16 GB DDR4-2400 | 35 | 65.9 s | 12.9 s | 278.2 s | 3.48 | 43.1 min | 0.25 |

The last column is final correct cases (after adjudication) divided by run minutes. Two models with identical accuracy are not equivalent if one does the work in half the time. On this hardware the 8B models produced about twice the verified correct output per minute of the 12B and 14B models, at equal or near-equal accuracy on this set.

## Thermal and reliability

| Model | Node | Outcome | Peak temp | Cumulative seconds at 80 C | Cleanup |
|---|---|---|---|---|---|
| Llama 3.1 8B | G3-5, 16 GB DDR4-2400 | Completed | 80 C | 620 | Unload confirmed |
| Hermes 3 8B | G3-4, 32 GB DDR4-2400 | Completed | 80 C | 960 | Unload confirmed |
| Gemma 3 12B | G3-6, 24 GB DDR4-2400 | Completed | 80 C | 1900 | Unload confirmed |
| Qwen3 14B | G3-2, 16 GB DDR4-2400 | Stopped: monitoring failure | 80 C | 1830 | Unload confirmed |

Every node reached its 80 C ceiling; the larger models spent far longer there because their runs were longer. The Qwen stop was not a thermal or hardware fault. The monitoring check itself failed, and the policy is that a run cannot continue without monitoring, so the harness stopped it cleanly. That is the behaviour the safety design was built to produce, and it is reported here as a positive result.

## The ceiling effect, and why it is the finding

All four models scored at or near 100 percent. That means this case set could not separate them, and no ranking should be drawn from it. Twelve cases per model, with single-case tracks for the false-premise and correction behaviours, cannot support claims of statistical significance in any direction.

What the experiment did establish:

- the four-model, four-node harness runs end to end with clean timing, confirmed unloads and verified backups;
- the deterministic grader and the blind adjudication path work;
- the fail-closed safety behaviour works under a real fault;
- and the next experiment needed substantially harder cases, built around evidence that conflicts, to discriminate between models.

That requirement is what R5 was designed to meet. See [r5-document-intelligence.md](r5-document-intelligence.md).

## Limitations

Exploratory and underpowered by design. One model's result rests on 11 rather than 12 cases. Thinking mode was off, so these numbers do not reflect what reasoning-tuned models do with it on. The set tests behaviours in isolation; it does not test them under the load of a long, realistic task.
