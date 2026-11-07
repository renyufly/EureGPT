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
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->463.005<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.059<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->426246.8<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->16779.1<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4166<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
"1": 2042.461,
"2": 2738.893,
"4": 2021.159,
"8": 2312.847
The fastest configuration was 4 threads
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->The naive implementation spends most of its time executing many small, sequential operations, with poor utilization of CPU vector units and repeated memory accesses.
The vectorized implementation processes entire matrices using optimized BLAS routines, which exploit SIMD instructions, cache-friendly memory access, and efficient matrix multiplication.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
4 threads were fastest. The Colab runtime provides only 2 vCPUs, so using 8 threads oversubscribes the available CPU resources. Threads compete for CPU time, cache capacity, and memory bandwidth, while synchronization and scheduling introduce additional overhead. 
Consequently, increasing thread count does not guarantee better performance.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
Transformer training involves large matrix multiplications in attention and feed-forward layers, which can be executed simultaneously across thousands of GPU threads. 
GPUs also provide high memory bandwidth and specialized hardware for efficient tensor operations. 
However, multiplying two small 10×10 matrices once would not benefit from this acceleration, because GPU kernel-launch and data-transfer overhead could exceed the computation time.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Parallelism stops helping when hardware limits, coordination overhead, or unavoidable sequential work outweigh additional computation.

1.Vectorized attention — sequential work and memory movement: 
Vectorization reduced execution time from 463.005 ms to 1.059 ms. SIMD and optimized matrix operations eliminated much of the inefficient execution, but remaining sequential operations and memory access impose limits on further acceleration.

2.CPU threading — hardware capacity and coordination: 4 threads were fastest at 2021.159 ms, but 8 threads took 2312.847 ms. With only two available vCPUs, extra threads compete for CPU time, caches, and memory bandwidth. Scheduling and synchronization overhead can outweigh the benefits of parallel execution.

3.GPU training — data movement and sequential dependencies: The GPU completed 100 steps in 16779.1 ms versus 426246.8 ms on CPU. Parallel matrix multiplication benefits greatly from GPU hardware, but CPU–GPU transfers, kernel launches, and sequential dependencies limit potential speedup.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would repeat the CPU threading sweep on a machine with 8 dedicated CPU cores instead of 2 vCPUs. I expect 8 threads to perform better because hardware contention should decrease. If 8 threads still performed worse than 4, I would reconsider CPU core availability as the main bottleneck and investigate synchronization and memory bandwidth.
<!--/answer-->
