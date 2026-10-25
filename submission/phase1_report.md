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
49
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
| IaaS VM | <!--answer:vm_cold_ms-->252.2<!--/answer--> | <!--answer:vm_warm_median_ms-->210.9<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2252.7<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->634.3<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1004.2<!--/answer--> | <!--answer:function_warm_median_ms-->634.3<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
[OK  ] vm        cold=  252.2ms  warm_median=  210.9ms
            #############
[OK  ] cloudrun  cold= 2252.7ms  warm_median=  634.3ms
            ########################################
[OK  ] function  cold= 1004.2ms  warm_median=  634.3ms

            ########################################
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
Compute Engine does not create new VMs automatically. With 1, 10, or 100 requests, the same e2-micro VM handles them. Its CPU and memory limit its performance.

Cloud Run creates new container instances when traffic increases. I set the minimum to 0 and the maximum to 3 instances. This limit comes from my deployment flags.

Cloud Functions Gen2 also creates new function instances automatically when needed. Its limits depend on the function settings and Google Cloud quotas. I did not set a specific maximum instance count.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
Compute Engine needed the most work. I created a VPC, subnet, firewall rules, and an e2-micro VM. I also set IAM permissions, installed Docker, and ran the container. I must manage the VM and its operating system myself.

Cloud Run was easier. I built a Docker image, pushed it to Artifact Registry, and deployed it with port 8080 and scaling settings. Google manages the servers and containers.

Cloud Functions Gen2 was the simplest to deploy. I uploaded Python source code and selected the Python 3.11 runtime and HTTP entry point. I also fixed a Cloud Build permission problem during deployment.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-582902874274.us-central1.run.app
LAST DEPLOYED BY: service-582902874274@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-25T13:24:50.489378Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-582902874274.us-central1.run.app
LAST DEPLOYED BY: yuxiang.ren@eurecom.fr
LAST DEPLOYED AT: 2026-10-25T13:04:07.782794Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The output shows two Cloud Run services: `echo-bot` and `echo`. I deployed `echo-bot` myself, but Google created `echo` when I deployed Cloud Functions Gen2. This shows that Cloud Functions Gen2 runs on Cloud Run.

Both services use containers and can automatically create new instances. They have similar systems, so their cold start and warm request times can be similar.

In my results, both had a warm median of 634.3 ms. However, Cloud Run's first request took 2252.7 ms, while the Function took 1004.2 ms.

The main difference is deployment. For Cloud Run, I built and deployed a Docker image. For Cloud Functions Gen2, I only provided Python code and a function entry point. Google handled the build.
<!--/answer-->
