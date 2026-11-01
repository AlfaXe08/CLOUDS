# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
50
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->252.3<!--/answer--> | <!--answer:vm_warm_median_ms-->212.7<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2886.4<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->636.4<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->898.6<!--/answer--> | <!--answer:function_warm_median_ms-->639.0<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
```
vm       cold   252.3 ms  ###
vm       warm   212.7 ms  ###
cloudrun cold  2886.4 ms  ########################################
cloudrun warm   636.4 ms  #########
function cold   898.6 ms  ############
function warm   639.0 ms  #########
```
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
The VM creates nothing automatically: every request is served by the single e2-micro instance, so its hard limit is the CPU and memory of the machine type I chose (--machine-type=e2-micro); handling more load would mean resizing it or adding VMs by hand. Cloud Run automatically starts additional container instances as concurrent requests grow and scales back to zero when idle; its hard limit is a flag I set myself, --max-instances=3. The gen-2 Function also adds managed instances automatically and can scale to zero, but I set no maximum: its limit is the platform default and project quotas, not a value I chose.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
The VM took the most work: I built a custom VPC and subnet, created two firewall rules (ports 22 and 8080, targeted by the echo-http tag), created the e2-micro Debian VM, granted its service account the Artifact Registry reader role, connected over SSH, installed Docker, authenticated it and started the container. I also remain responsible for the OS, Docker and security patches afterwards. Cloud Run needed a single command with my existing image: port 8080, unauthenticated access, min 0 and max 3 instances, while networking, HTTPS and scaling are managed. For the Function I supplied only Python source, the runtime and the entry point; Google built the container and runs it.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-415795400525.us-central1.run.app
LAST DEPLOYED BY: service-415795400525@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-28T14:40:26.127919Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-415795400525.us-central1.run.app
LAST DEPLOYED BY: thomasd0803@gmail.com
LAST DEPLOYED AT: 2026-10-28T14:35:11.332969Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
gcloud run services list shows two services: echo-bot, which I deployed myself, and echo, deployed by the Cloud Functions service account (gcf-admin-robot). So a gen-2 Function is a Cloud Run service that Google creates from my source: it builds a container and runs it on the same infrastructure. That explains why the warm medians are nearly identical (636.4 ms for Cloud Run, 639.0 ms for the Function): same serving path and same HTTPS front end. Both scale to zero, so a request after an idle period needs a new instance. The cold starts differ, though: 2886.4 ms for Cloud Run and 898.6 ms for the Function. The Function's lower cold start may mean its instance had not fully scaled down yet, so I cannot conclude it is genuinely faster. What truly differs is the abstraction: I provide a container image to Cloud Run, but only source code to the Function, and Google builds the container.
<!--/answer-->
