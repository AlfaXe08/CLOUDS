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
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->38.9<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.005<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->3.1<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->11<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
Ranking: local 0.005 s < Spark 3.1 s < cloud MapReduce 38.9 s. The like-for-like pair is local vs cloud: the same word count, with the same primitives, over the same 60 documents. The local run is pure CPU work in one process. The cloud job spends almost all of its time on coordination: 16 HTTP function invocations (some of them cold starts), about 65 Cloud Storage reads and writes for the splits, shuffle files and outputs, and workflow scheduling. My submitted run also had `--chaos 0.3`, so it includes retries with exponential backoff (a run without chaos took 32.2 s). Spark does strictly more work (two `reduceByKey`s and a `join`), yet finishes in 3.1 s on one machine, mostly in JVM and task-scheduling overhead. So the cloud job's 38.9 s is distribution overhead, not computation: this corpus is far too small to amortise it.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
4 stages means 3 shuffles: the `reduceByKey` in `term_freq` (line 31), the `reduceByKey` in `doc_freq` (line 42), and the `join` in `build_tfidf_rdd` (line 54). In the `--chaos 0.3` run I counted 11 retries (HTTP 503 responses of `mr-worker` in Cloud Logging) for 16 tasks, i.e. 27 attempts. That means 11/27 ≈ 41% of attempts failed, or 11/16 ≈ 0.69 retries per task. With p = 0.3, the expectation is 16 · 0.3/0.7 ≈ 7 retries (30% of attempts). 11 is higher than that, but within normal random variation for such a small sample.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
`parallelize` creates the source RDD from the Python list. The transformations are lazy and each returns a new RDD: `flatMap` and `reduceByKey` (in `term_freq`), `map` and `reduceByKey` (in `doc_freq`), and `map`, `join` and `map(to_row)` (in `build_tfidf_rdd`). The only action is `collect` (in `build_tfidf_rows` / notebook 2c). `toDebugString` is neither: it only prints the recorded lineage. Calling `build_tfidf_rdd` returned instantly: it only recorded the lineage graph, and no task ran. `collect` made Spark split that graph into 4 stages, run 12 tasks, and ship 5975 rows back to the driver in 3.1 s.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
The first `reduceByKey` (`term_freq`) is the shuffle boundary between stage 0 and stages 1/2 (`ShuffledRDD[3]` in the lineage). If the worker holding one of its output partitions died, Spark would recompute only that partition. It would re-fetch that partition's blocks from stage 0's shuffle files, and reuse every other partition and every completed shuffle output. Only if some of stage 0's shuffle files were lost with that worker would Spark re-run just those stage-0 tasks from `ParallelCollectionRDD[0]`. It never reruns the whole job.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows (`mr-wordcount`, defined in `phase-3-mapreduce-spark/workflow.yaml`)<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->Cloud Run function (`mr-worker`, implemented in `phase-3-mapreduce-spark/main.py` using `mapreduce.py`)<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Google Cloud Storage (`<project>-eurecomgpt` bucket with `<job>/input/`, `shuffle/`, and `output/` prefixes)<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->`partition(word, num_reducers)` in `phase-3-mapreduce-spark/mapreduce.py` (`zlib.crc32(word) % num_reducers`, a hash stable across machines), called by `run_map` in `main.py` to pick each pair's shuffle file<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The barrier is the `map_phase` step using `parallel` with `for` in `workflow.yaml`: a `parallel` step only completes when all of its branches have completed, so Cloud Workflows waits for every map branch before proceeding to `reduce_phase`.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
Overwriting shuffle objects at deterministic keys makes retries safe because re-running a failed map task produces the exact same state. If `run_map` appended instead of overwriting, retried map attempts would duplicate `(key, value)` pairs in shuffle files, inflating word counts and breaking exactness.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
With `--chaos 0.3`, the execution log and Logs Explorer showed 11 HTTP 503 simulated worker crashes retried automatically with exponential backoff by Cloud Workflows. The job still succeeded (38.9 s vs 32.2 s without chaos) and returned the exact same 2,221 distinct words and top terms as the local reference.
<!--/answer-->
