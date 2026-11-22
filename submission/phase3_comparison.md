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
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->25.9<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.048<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->9.5<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->8<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
Fastest to slowest: local 0.048 s, Spark 9.5 s, cloud 25.9 s. Local vs cloud is the like-for-like pair: the same word count over 7,730 pairs. Locally that's a Python loop in memory. The cloud job pays for 24 authenticated HTTP calls (16 tasks plus 8 retries), function instances starting up, about 110 Cloud Storage reads and writes, the barrier, and some retry backoff since this was my --chaos run. Spark runs a heavier job (three shuffles) on one machine and is still almost 3x faster, mostly start-up and scheduling. So the cloud job's time goes into distribution overhead, not computation.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
4 stages means 3 shuffles: the reduceByKey in term_freq, the reduceByKey in doc_freq, and the join, which in PySpark unions both sides and repartitions them. 8 retries for 16 tasks means 24 attempts, 8 of which failed: 33%, close to the 30% that --chaos 0.3 should give.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
parallelize just creates the source RDD. flatMap, map (in doc_freq, for tf_by_term and in to_row), reduceByKey (twice) and join are transformations: they only add a step to the lineage. collect is the only action. Running build_tfidf_rdd returned instantly and nothing ran, which is why 2b could print the lineage first. collect started job 0, the first job of the session: Spark split it into 4 stages, ran 12 tasks and brought 5,975 rows back in 9.5 s.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
The first reduceByKey's partitions are rebuilt from the shuffle files that stage 0 wrote. If the worker holding one died, Spark would recompute just that partition from those files and rerun the tasks downstream of it, reusing everything else. Recovery starts at the first shuffle boundary (ShuffledRDD[3], after stage 0). Stage 0 itself would rerun only if its shuffle files were lost too. My run shows that reuse: tf feeds both doc_freq and the join, but stage 0 ran once and stage 2 re-read its 51.0 KB.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows: the mr-wordcount workflow (workflow.yaml).<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->The mr-worker Cloud Run function (main.py), one invocation per map or reduce task.<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Cloud Storage: my eurecomgpt-iban-eurecomgpt bucket (input splits, shuffle files and outputs, in one mr-... folder per job).<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->partition() in mapreduce.py: zlib.crc32(word) % 4, called by run_map in main.py.<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
It's the map_phase parallel step itself. A parallel step only finishes when all its branches have, and reduce_phase is simply the next step, so no reducer can start before the last mapper returns.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
A map task always writes the same files for the same split, and upload_from_string replaces the whole object. So a retried task just rewrites identical files, even if the failed attempt had already written some of them. If run_map appended instead, a retry after a partial run would add the same pairs twice and the reducers would over-count those words. The job would still say SUCCEEDED, but matches_local would be false.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
In the execution view I couldn't really tell anything had failed: it looked like a normal run, map branches in parallel then the reducers, because the workflow retries on its own. The failures only showed up in the logs: filtering mr-worker requests on status 503 gave 8 entries among the 200s. The job still SUCCEEDED, with the same 2221 words as the local reference (matches_local: true).
<!--/answer-->
