# Phase 3 — Comparison Writeup

Copy this file to `submission/phase3_comparison.md`, fill in every answer slot, and commit it:

```bash
cp phase-3-mapreduce-spark/comparison_template.md submission/phase3_comparison.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers in section 1 must **match `submission/phase3_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Facts from your runs

Numbers only (e.g. `12`, `20.3`, `0.041`).

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->13.6<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.011<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->5.4<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->7<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
The local word count was fastest (approximately 0.011 s), followed by Spark TF-IDF (5.4 s) and cloud MapReduce (13.6 s). 

The local and cloud word counts are directly comparable because they perform the same computation on the same 60 documents. 

Cloud MapReduce spends substantial time on workflow scheduling, HTTP function invocations, Cloud Storage I/O, and coordination between map and reduce phases. 
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
Spark executed four stages with three shuffle boundaries, caused by the two reduceByKey operations and the join. 

The chaos run produced seven HTTP 503 failures across 16 logical tasks. It represents approximately 43.75% of the original task count, compared with the configured 30% failure probability per attempt. Random variation and repeated failures are possible.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
The RDD transformations are flatMap, map, reduceByKey, and join. 

In term_freq, flatMap generates term-document pairs and reduceByKey counts their occurrences. 

In doc_freq, map generates document-presence indicators and reduceByKey aggregates document frequencies. 

Further map and join transformations combine TF and DF and construct TF-IDF rows. 

build_tfidf_rdd only constructs the lazy execution lineage without running the computation.  

collect action triggers execution of the dependency graph and returns the computed rows to the driver. 

toDebugString inspects the lineage without executing it.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
If a worker lost an output partition from the first reduceByKey, Spark could reconstruct that partition using its lineage. 

Recovery would start at the first reduceByKey shuffle boundary, shown as ShuffledRDD[3] in the notebook. 

Spark could reuse surviving shuffle partitions and recompute the missing partition from its upstream map output. Dependent downstream partitions might then need recomputation, but Spark would not need to restart the entire job.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Google Cloud Workflows (mr-wordcount, defined in workflow.yaml)<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->Cloud Run Functions (mr-worker, implemented in main.py)<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Google Cloud Storage (gs://eurecom-lab-512621-eurecomgpt/)<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->The deterministic partitioning function in mapreduce.py<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The barrier is the completion of the map_phase parallel step in workflow.yaml. Cloud Workflows waits for every parallel map branch to finish before proceeding to reduce_phase, ensuring that all shuffle outputs are available before reducers start.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
Retries are safe because each worker task writes deterministic results to fixed Cloud Storage object paths, overwriting previous output rather than accumulating it. Re-executing the same task therefore produces the same logical result. 

If run_map appended pairs to an existing shuffle file instead, retries could duplicate intermediate key-value pairs. Reducers would then count duplicated values, producing incorrect word frequencies. Overwriting outputs makes repeated execution safe and allows the workflow to recover from transient failures without corrupting the final result.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
With chaos=0.3, Cloud Logging showed seven HTTP 503 responses from mr-worker between 23:16:38 and 23:16:44 UTC. These indicate simulated transient worker failures. 

The workflow's retry policy allowed the execution to finish successfully despite the failures. The final state was SUCCEEDED after 13.6 seconds. 

The cloud job produced 2,221 distinct words, and its output matched the local reference exactly (identical: True).
<!--/answer-->
