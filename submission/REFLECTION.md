# Reflection — Lab 20 (Personal Report)

---

**Họ Tên:** Tran Quang Long - 2A202600304
**Cohort:** A20-K1
**Ngày submit:** 2026-05-06

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

- **OS:** Linux 6.8.0-1044-azure (x86_64)
- **CPU:** AMD EPYC 9V74 80-Core Processor
- **Cores:** 2 physical / 2 logical
- **CPU extensions:** AVX2
- **RAM:** 7.8 GB
- **Accelerator:** CPU only (no discrete accelerator)
- **llama.cpp backend đã chọn:** CPU
- **Recommended model tier:** TinyLlama-1.1B

**Setup story:** I used GitHub Codespaces to ensure a clean, isolated environment. Since I was running on a virtualized cloud CPU, I focused on optimizing thread allocation and batch sizes for CPU-bound inference rather than GPU offloading.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf | 2047 | 443 / 592 | 46.5 / 50.7 | 3289 / 3436 / 3438 | 21.5 |
| tinyllama-1.1b-chat-v1.0.Q2_K.gguf | 1074 | 722 / 914 | 60.4 / 62.6 | 4435 / 4712 / 4725 | 16.6 |

**Một quan sát:** Surprisingly, the Q4_K_M model had a higher decode rate than Q2_K. On this specific cloud vCPU, the larger model weights likely aligned better with the processor's AVX2 registers, whereas the heavily compressed Q2_K might have incurred higher dequantization overhead.

---

## 3. Track 02 — llama-server load test

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 0.13 | 14000 | 53000 | 53000 | 0 |
| 50 | 0.11 | 29000 | 48000 | 48000 | 0 |

**KV-cache observation:** Peak `llamacpp:kv_cache_usage_ratio` hit 1.0 under 50-user load. This saturation led to significant queueing delays and client disconnects, as the 2048-token context window was fully exhausted across the 4 parallel slots.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** stub: localhost only
- **N17 (Data pipeline):** stub: in-memory dict
- **N18 (Lakehouse):** stub: SQLite
- **N19 (Vector + Feature Store):** stub: TOY_DOCS

**Nơi tốn nhiều ms nhất:**
- retrieve: 0.0 ms
- llama-server: 8472.0 ms

**Reflection:** The bottleneck is entirely the CPU-bound LLM inference. In a production scenario, retrieval time would be negligible compared to the seconds required for token generation on this hardware.

---

## 5. Bonus — The single change that mattered most

**Change:** Reducing `LAB_N_THREADS` from 2 down to 1.

**Before vs after:**

before (2 threads): 21.5 tokens/s
after (1 thread): 23.0 tokens/s
speedup: ~1.07x

**Tại sao nó work:** 
On this specific 2-vCPU Azure instance, the overhead of the OS trying to synchronize the TinyLlama-1.1B weights across two virtual cores was greater than the parallel processing gain. By forcing the work onto a single thread, I eliminated context switching and cache-coherency overhead, leading to a measurable 7% improvement in throughput. This demonstrates that "more threads" is not a universal fix for small models on limited silicon.

---

## 6. (Optional) Điều ngạc nhiên nhất

It was shocking to see how quickly latency exploded from 3 seconds to over 40 seconds just by adding 10 concurrent users. It really emphasized the importance of continuous batching and PagedAttention in production systems.
