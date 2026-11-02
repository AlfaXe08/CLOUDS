# Phase 2 — Reflection: where does parallelism stop helping?

Copy this file to `submission/phase2_reflection.md`, fill in every answer slot, and commit it:

```bash
cp phase-2-tiny-gpt/reflection_template.md submission/phase2_reflection.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or your threading plot freely.

Numbers in section 1 must **match `submission/phase2_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->292.249<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.205<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->495874.8<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->12506.1<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.5097<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2849.181 ms, 2 threads: 2435.795 ms, 4 threads: 2557.68 ms, 8 threads: 2422.061 ms. Fastest: 8 threads (2422.061 ms), but only 0.6% faster than 2 threads, which is within measurement noise.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive version spends its time in the Python interpreter, not in arithmetic: for each of the 65,536 (i, j) pairs it decodes bytecode, checks types and allocates objects, then does a single scalar multiply-add (292.249 ms). The vectorized version makes one call into OpenBLAS 0.3.27, which uses AVX2/AVX-512 SIMD and FMA instructions to process up to 8 doubles per instruction in cache-sized blocks: 1.205 ms, 242.5× faster.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
The curve flattens at 2 threads: 2849.181 ms (1) → 2435.795 ms (2), then 2557.68 ms (4) and 2422.061 ms (8), no real gain past 2. The runtime has nproc = 2 vCPUs, but lscpu shows 1 physical core with 2 hyperthreads, so both threads share the same AVX/FMA units: only 1.17× instead of 2×. Beyond 2, threads are oversubscribed: they wait for a core, context-switch and synchronize after every parallel operation, which costs more than it brings.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
A training step is dominated by dense matrix multiplications (QKV and MLP linears, Q·Kᵀ, att·V) over 32×256 tokens: thousands of independent multiply-adds that the T4’s 2,560 CUDA cores run in parallel (SIMT) with high memory bandwidth: 495,874.8 ms → 12,506.1 ms for 100 steps (39.7×). Token-by-token generation with batch size 1 would not benefit: each step is tiny and sequential, so kernel-launch and Python overhead dominate.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Each experiment hit a different limit. Vectorization barely hit one: it did not add workers, it removed interpreter overhead, so the same core went from 292.249 ms to 1.205 ms (242.5×) by processing 8 doubles per AVX-512 instruction. Threads hit three limits at once. Hardware we do not have: nproc reports 2 vCPUs, but lscpu shows 2 hyperthreads of 1 physical core sharing the same FMA units, so 2 threads gave only 1.17× (2849.181 → 2435.795 ms). Work that cannot be divided: batch sampling and Python dispatch stay serial whatever the thread count (Amdahl’s law). Coordination that costs more than it saves: at 4 threads the step got slower (2557.68 ms), as oversubscribed threads context-switch and synchronize. The GPU was bound by data movement: only 39.7× (495874.8 → 12506.1 ms) despite 2,560 cores, because every step the CPU builds the batch, copies it to the device and launches many small kernels, so the GPU often waits.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would rerun the thread sweep on a machine with 8 real physical cores and hyperthreading disabled. If 2 threads then give close to 2× and the curve keeps falling up to the core count, the 1.17× I measured was caused by sharing one hyperthreaded core. If it still flattens early, the real limit is serial work (Amdahl) or memory bandwidth.
<!--/answer-->
