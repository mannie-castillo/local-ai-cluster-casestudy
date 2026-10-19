# Hardware

Seven refurbished business mini PCs on a private, offline local network, managed from a separate control machine.

## Software

Windows 11 Pro (23H2 or 22H2) on the six HP nodes and Windows 10 Enterprise on the Dell, as recorded on 6 October 2026. Ollama 0.35.0 on the inference workers G3-2 to G3-6; Open WebUI 0.11.4 on G3-3 (16 GB DDR4-2133) with nomic-embed-text (137M parameters, F16) as the embedding model. Every chat model ran as a 4-bit build from the Ollama library (Q4_K_M, except Hermes 3 8B at Q4_0). The 7 October gpt-oss 20B run on G3-1 (32 GB DDR4-2400) used a temporary Ollama instance started only for that test and stopped afterwards; its version was not separately recorded in the run report.

## Nodes

| Node | Machine | CPU | Cores / threads | Base / turbo | RAM | Memory layout | Storage |
|---|---|---|---|---|---|---|---|
| G3-1 | HP ProDesk 600 G3 Mini | Intel Core i7-7700T | 4 / 8 | 2.9 / 3.8 GHz | 32 GB DDR4-2400 | 2 x 16 GB, dual channel | 256 GB SATA SSD |
| G3-2 | HP ProDesk 600 G3 Mini | Intel Core i7-7700T | 4 / 8 | 2.9 / 3.8 GHz | 16 GB DDR4-2400 | 2 x 8 GB, dual channel | 256 GB SATA SSD |
| G3-3 | HP ProDesk 600 G3 Mini | Intel Core i7-7700T | 4 / 8 | 2.9 / 3.8 GHz | 16 GB DDR4-2133 | 2 x 8 GB at 2133 MT/s, dual channel | 256 GB SATA SSD |
| G3-4 | HP ProDesk 600 G3 Mini | Intel Core i7-7700T | 4 / 8 | 2.9 / 3.8 GHz | 32 GB DDR4-2400 | 2 x 16 GB, dual channel | 256 GB SATA SSD |
| G3-5 | HP ProDesk 600 G3 Mini | Intel Core i7-7700T | 4 / 8 | 2.9 / 3.8 GHz | 16 GB DDR4-2400 | 2 x 8 GB, dual channel | 256 GB SATA SSD |
| G3-6 | HP ProDesk 600 G3 Mini | Intel Core i7-7700T | 4 / 8 | 2.9 / 3.8 GHz | 24 GB DDR4-2400 | 8 GB + 16 GB, flex mode (16 GB dual, 8 GB single) | 256 GB SATA SSD |
| Dell | Dell OptiPlex 5050 Micro | Intel Core i5-7500T | 4 / 4 | 2.7 / 3.3 GHz | 16 GB DDR4-2400 | 2 x 8 GB, dual channel | 240 GB SATA SSD |

All are 35-watt parts with integrated graphics only. The integrated graphics device is disabled on five of the seven nodes and left enabled on the Dell and G3-4 by operator decision; inference is CPU-only on every node. Each node has one 1 Gbps Ethernet link with a reserved address on the private network. The router's WAN port is physically unplugged (my own check, not a network test), so the network has no internet uplink.

## Upgrades

Several of the HP units shipped with an Intel Core i5-6500T. They were upgraded to the i7-7700T, which is close to the top of what the board supports and adds hyper-threading and a higher turbo clock. Two nodes run 32 GB DDR4-2400, the board maximum (G3-1 already had it and G3-4 was upgraded to match), to hold models in the 14B to 27B range. Storage, networking and operating systems were reconfigured on every unit.

## Why memory configuration is expected to matter

Token generation on a CPU is bound by memory bandwidth, not compute: each generated token requires streaming the model's weights through the memory bus. That makes the memory layout expected to be a major variable on this class of machine (memory-bound generation); CPU models were not compared.

Measured in Experiment 1 with the same 14B model:

| Node | Memory | Median tokens/s | Median answer time |
|---|---|---|---|
| G3-4 | 32 GB DDR4-2400, 2 x 16 GB, matched, dual channel | 2.77 | 48.4 s |
| G3-6 | 24 GB DDR4-2400, 8 GB + 16 GB, mismatched, partly single channel | 2.24 | 50.8 s |

Same CPU, same model, same questions: the matched dual-channel node generated text about 24 percent faster. Caveats: the two runs were under different conditions (G3-4 alone at its thermal ceiling, G3-6 in a lock-step parallel run), and prompt reading speed was the same, so the end-to-end gain per answer was about 4 percent.

## Thermal behaviour

Under sustained inference these chips reach their 80 C reporting ceiling quickly and stay there. In Experiment 1, a 14B model on G3-4 (32 GB DDR4-2400) produced Core Temp readings at 80 C in 177 of 193 samples, with the longest continuous stretch at 80 C lasting 6.8 minutes. Nothing failed. On G3-4 the per-answer minimum core clock was about base speed (2,892 MHz) on 30 of 40 answers and slightly below it (2,792 MHz) on 4, consistent with thermal limiting (the performance-limit counters were not read). The monitoring policy therefore records cumulative time at 80 C rather than using a higher stop threshold that could never fire.

## Roles

- **G3-2 (16 GB DDR4-2400), G3-4 (32 GB DDR4-2400), G3-5 (16 GB DDR4-2400), G3-6 (24 GB DDR4-2400)**: inference workers. Each serves one model at a time.
- **G3-3 (16 GB DDR4-2133)**: the retrieval host. Runs Open WebUI with a local embedding model and holds the private document collection. Its web interface is allowed by firewall rule only from the control machine (rule, not a network test).
- **G3-1 (32 GB DDR4-2400)**: reserved for large-model testing (20B to 32B). It has special handling: it is never rebooted, and every change to it needs its own approval.
- **Dell (16 GB DDR4-2400)**: being set up as a cold archive for hash-verified copies of the models that had no other durable copy (transfer in progress, 17 of 83 files at 14 October); until it completes, the original download drive remains the only durable copy of those models. Verified copies of the remaining models are distributed across the worker nodes rather than concentrated here. Six models are on hold (five pending licence review, one by operator decision), and the final archive acceptance ledger is still pending. It runs no inference.
- **G3-6 (24 GB DDR4-2400)** additionally holds the hash-verified evidence backups of every experiment except the 7 October Experiment 1 raw files (see Evidence exceptions); the Operational Depth-1 and R5 pilot copies predate later edits to their run records and have not yet been refreshed; a refresh needs the cluster on and its own approval.
- **Control machine**: an HP Envy laptop that manages every node over the local network. It is never a worker.
