# Experiment 1: the bake-off

**Dates:** 7 October 2026 (first five runs), 12 October 2026 (three additional models).
**Status:** complete. Evidence note: the raw per-answer files for the 7 October runs were on an unmounted drive during the 18 October re-hash and no second copy was found; see Evidence exceptions in [methodology.md](methodology.md).

## Question

On identical questions, how do the available models compare on accuracy and speed on this hardware, and does a routing strategy that chains a small model to larger ones beat choosing any one model?

## Design

A 40-question set in five categories:

| Category | Questions | What it tests |
|---|---|---|
| Factual recall | 15 | Direct knowledge questions |
| Combining facts | 10 | Answers that require joining two or more facts |
| Numeric | 5 | Answers that are a number |
| Absent answer | 5 | Questions with no valid answer; the correct response is to say so |
| False premise | 5 | Questions built on a wrong assumption; the correct response rejects it |

Every model answered the same 40 questions, one model per node; some runs ran in parallel across nodes. Scoring counted an answer correct against a frozen key; for absent-answer and false-premise questions, inventing an answer or conceding the false premise counted as wrong. The run also recorded invented flags (the model asserting something not in evidence) and wrong refusals (refusing a question that had an answer).

## Results

| Model | Maker | Params | Node | Correct of 40 | Median tokens/s | Median time to first token | Median answer time |
|---|---|---|---|---|---|---|---|
| Gemma 3 12B | Google | 12B | G3-6, 24 GB DDR4-2400 | 32 | 2.85 | 43.4 s | 59.7 s |
| Qwen3 4B | Alibaba | 4B | G3-2, 16 GB DDR4-2400 | 31 | 9.6 | 8.6 s | 11.1 s |
| Qwen3 4B | Alibaba | 4B | G3-5, 16 GB DDR4-2400 | 31 | 9.8 | 8.1 s | 10.7 s |
| Qwen3 14B | Alibaba | 14B | G3-4, 32 GB DDR4-2400 | 29 | 2.8 | 38.3 s | 48.4 s |
| Qwen3 14B | Alibaba | 14B | G3-6, 24 GB DDR4-2400 | 29 | 2.2 | 38.5 s | 50.8 s |
| gpt-oss 20B | OpenAI | 20B (MoE) | G3-1, 32 GB DDR4-2400 | 27 | 6.6 | 25.9 s | 28.2 s |
| Hermes 3 8B | Nous Research | 8B | G3-4, 32 GB DDR4-2400 | 26 | 5.47 | 22.7 s | 30.3 s |
| Llama 3.1 8B | Meta | 8B | G3-5, 16 GB DDR4-2400 | 17 | 5.49 | 17.9 s | 24.3 s |

Per-category correct counts (fact 15 / combine 10 / numeric 5 / absent 5 / false premise 5):

| Model | Fact | Combine | Numeric | Absent | False premise |
|---|---|---|---|---|---|
| Gemma 3 12B | 13 | 6 | 5 | 5 | 3 |
| Qwen3 4B | 13 | 8 | 5 | 5 | 0 |
| Qwen3 14B | 13 | 6 | 5 | 5 | 0 |
| gpt-oss 20B | 12 | 4 | 5 | 5 | 1 |
| Hermes 3 8B | 10 | 5 | 4 | 5 | 2 |
| Llama 3.1 8B | 6 | 2 | 4 | 5 | 0 |

Observations the data supports:

- **The 4B model was the best value by a wide margin.** It matched or beat the 14B model from the same family on accuracy while generating four times faster.
- **Every model handled absent-answer questions.** All eight runs scored 5 of 5 on recognising that a question had no answer. Recognising a false premise was much harder: only Gemma got a majority of those.
- **The mixture-of-experts 20B model generated faster than the dense 14B** (6.6 vs 2.8 tokens/s) despite more total parameters, because only a fraction of its parameters are active for each token. On memory-bound CPU inference, active parameters matter more than total parameters.
- **Llama 3.1 8B scored low on this set** (17 of 40), driven by factual recall and fact combination. It performed well in the later reasoning experiment, which suggests this set exposed a knowledge gap rather than a reasoning one.
- **Same model, two nodes, same accuracy.** The two Qwen3 4B runs and the two Qwen3 14B runs produced identical correct counts on different nodes, which is the expected result and a useful check that the harness is deterministic.

## The routing experiment

The more useful question was whether chaining models beats choosing one. A cascade runs the small model first and escalates to a larger model only when the small model's answer is wrong. Here escalation was decided by the grader, which knows the correct answer, so the cascade figures below are upper bounds for a live cascade; a live version would need its own confidence signal and has not been tested.

| Strategy | Correct of 40 | Total time | Seconds per correct answer |
|---|---|---|---|
| Small model first, escalate to 14B, then 20B | **34** | 1101 s | 32 |
| Small model first, escalate to 14B | 33 | 880 s | 27 |
| Always the 4B model | 31 | 447 s | 14 |
| Always the 14B model | 29 | 1910 s | 66 |
| Always the 20B model | 27 | 1164 s | 43 |

Two conclusions:

- **The cascade was more accurate than any single model** (as an upper bound, since escalation used the grader), including the models it escalated to. The small model answered the easy questions quickly and correctly, and escalation fixed some of what it got wrong. The two-question margin is within noise for 40 questions, and the comparison spans runs on different days.
- **Running one large model on everything was the worst strategy, in two different ways.** The 14B-only option was the slowest, and the 20B-only option was the least accurate. Neither beat the small model alone. On this hardware, model size is a cost to be spent selectively, not a default.

This result is the basis of the routing system the project is working toward.

## Parallelism

Running one 40-answer batch across two nodes instead of one:

| Setup | Nodes | Total time (measured) |
|---|---|---|
| One node | 1 | 343 s |
| Two nodes | 2 | 237 s |

Not a clean 2x, because prompt caching helped the single-node run (15 cached prompts versus 1) and because the batch has fixed overhead, but a real gain on two nodes; scaling beyond two nodes was not measured. Both timings come from a rerun; the first attempt was stopped by the original 80 C auto-stop.

## Hardware effect

The same 14B model ran 24 percent faster on G3-4 (32 GB DDR4-2400, matched 2 x 16 GB, dual channel) than on G3-6 (24 GB DDR4-2400, mismatched 8 GB + 16 GB, partly single channel): 2.77 vs 2.24 tokens per second. The runs were under different conditions (G3-4 rerun alone at its thermal ceiling, G3-6 in a lock-step parallel run), prompt reading speed was the same on both, and the end-to-end gain per answer was about 4 percent. See [hardware.md](hardware.md).

## Thermal

The 14B rerun on G3-4 (32 GB DDR4-2400; its first run was stopped by the original 80 C auto-stop) held the CPU at its 80 C ceiling for most of the run: 177 of 193 temperature samples at 80 C, longest continuous stretch 6.8 minutes. No fault. The per-answer minimum clock sat at about base speed (2,892 MHz) on 30 of 40 answers and slightly below it (2,792 MHz) on 4, consistent with the CPU holding itself at its thermal limit. A clear picture of sustained load on 35-watt silicon.

## Limitations

Forty questions is enough to see large differences and strategy effects; it is not enough to rank models that are a few answers apart. The set is general-knowledge, so it rewards what a model memorised in training as much as how it reasons, which is why Experiment 2 moved to reasoning and R5 moved to a fictional corpus the models could not have seen.
