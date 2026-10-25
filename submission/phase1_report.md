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
47.7
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
| IaaS VM | <!--answer:vm_cold_ms-->251.7<!--/answer--> | <!--answer:vm_warm_median_ms-->211.1<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->130.4<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->633.0<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->802.9<!--/answer--> | <!--answer:function_warm_median_ms-->693.8<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
```
vm        cold=  251.7ms  warm_median=  211.1ms
          ############
cloudrun  cold=  130.4ms  warm_median=  633.0ms
          ####################################
function  cold=  802.9ms  warm_median=  693.8ms
          ########################################
```

The VM has no real cold start: its container never stops, so its "cold" number is just the first request of the run.
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
On the VM nothing is created automatically: one e2-micro, one container, gunicorn with 1 worker and 8 threads, so 8 requests at a time and the rest wait. With a hundred at once it gets slow or times out. The limit is the machine I picked (--machine-type=e2-micro). Cloud Run starts new instances of my container as the load grows, up to the --max-instances=3 I set, each handling up to 80 requests at once by default. The function works the same way but with 1 request per instance, so a burst creates many instances, up to the default cap of 100.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
The VM was clearly the most work. I had to build the network myself (VPC, subnet, two firewall rules), create the VM with the right tag, give its service account access to Artifact Registry, SSH in, install Docker, log it in to the registry and run the container. My first VM creation failed with "subnetwork not found": I thought the VM command was wrong, but the subnet had never been created, probably because my $REGION variable was empty at that point. And it's a machine I would have to keep updated. For Cloud Run I only wrote the Dockerfile, built the image with Cloud Build and deployed it in one command. The function was the easiest: no Dockerfile, just my source folder and the entry point.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
```
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-834184498166.us-central1.run.app
LAST DEPLOYED BY: service-834184498166@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-25T18:46:31.653183Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-834184498166.us-central1.run.app
LAST DEPLOYED BY: iban.leparoux@eurecom.fr
LAST DEPLOYED AT: 2026-10-25T18:43:27.924927Z
```
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
There are two services in the list, not one: echo-bot, which I deployed, and echo, which is my Cloud Function. It was deployed by the gcf-admin-robot account and has a run.app URL too. So a gen 2 function is really a Cloud Run service underneath: gcloud functions deploy sent my source to Cloud Build, Google's buildpacks turned it into a container image, and that image was deployed on Cloud Run. That's why both cold starts look alike: in both cases Cloud Run starts a container from zero. What really differs is who builds the image (my Dockerfile vs buildpacks, so I don't control what's in it), the default settings (1 request per instance and less memory and CPU for the function, 80 requests per instance for Cloud Run) and what I deploy: source code vs a ready image.
<!--/answer-->
