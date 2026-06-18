---
title: "Why I Chose AWS Batch for GPU Workloads (And What It Took to Make It Work)"
date: 2026-06-18T15:30:00+05:30
draft: false
tags:
    - aws
    - gpu
    - batch
---

Hi there!

Running GPU workloads in the cloud sounds straightforward until you're staring at a job queue that's slow, expensive, and falling over under load. In this post, I'll walk through how I got GPU batch processing to a place where it was fast, cost-efficient, and something I could actually trust and the three fixes that made the difference.

## The problem: GPU compute that doesn't run 24/7

Not every GPU workload is a steady stream. Mine wasn't. Jobs came in bursts, sometimes a few at once, sometimes nothing for a while. The compute needs were real and heavy, but they weren't constant.

That left me with a few bad options:

- Keep a GPU instance running all the time and bleed money during idle periods.
- Spin up EC2 manually per job and drown in operational overhead.
- Write custom queuing logic to manage concurrency and retries myself.

None of these made sense. What I actually needed was something that could sit quietly until work arrived, process jobs in order without collisions, scale when needed, and not require me to babysit it.

## Why AWS Batch fit

AWS Batch is purpose-built for exactly this kind of workload. It handles the infrastructure lifecycle for you compute comes up when jobs arrive and goes away when they don't. More importantly for GPU work, it supports GPU instance families natively and wraps everything in a container-based job model, which means reproducible, isolated runs without any custom environment setup per job.

A few things stood out:

**Job queues with proper controls.** Batch gives you first-class job queues with priority support, retry strategies, and dependency chaining between jobs. No DIY polling, no race conditions, no dropped work.

**Compute environments with launch templates.** This is where Batch gets interesting. You can attach a launch template to your compute environment, which means full control over the underlying EC2 instances userdata scripts, EBS volumes, ECS agent config, all of it. That ended up being critical, as I'll get to shortly.

**GPU resource targeting.** Job definitions let you specify GPU resource requirements directly, and Batch handles the scheduling logic to match jobs to instances that have what they need. The `g5` instance family was the right fit for my workload high GPU memory, solid throughput.

## Step Functions: the orchestration layer on top

Batch handles running jobs. It doesn't handle *what runs after what*.

That's where AWS Step Functions came in. A complex workload isn't a single job it's a pipeline of jobs with dependencies between them. Step Functions gave me a proper state machine to model that:

- Chain jobs in sequence where outputs feed into the next stage.
- Branch on success or failure without writing polling logic.
- Handle retries with exponential backoff at the orchestration level, not inside job code.
- Get visibility into exactly where a pipeline is at any point in time.

Together, Batch and Step Functions gave me compute and coordination. One handles the *how*, the other handles the *when* and *in what order*.

![Architecture diagram showing jobs flowing from Step Functions orchestration into an AWS Batch job queue and GPU compute environment](assets/architecture.svg)

## Where things got hard (and how we fixed them)

This is the part nobody writes about. The setup above works. What it doesn't tell you is that out of the box, your jobs will be slower than they should be, your costs will be higher than they need to be, and your instances will behave in ways that aren't obvious until you dig into the internals.

Here's what we hit and how we resolved each one.

### Problem 1: Image pull was killing job start times

Every job was pulling the Docker image fresh from ECR before running. For a large GPU image CUDA drivers, ML libraries, everything that pull alone was eating a significant chunk of each job's wall clock time. Multiply that across a queue of jobs and it adds up fast.

The fix was straightforward once we found it, but it's buried in ECS agent configuration: disable image cleanup.

By default, the ECS agent running on the Batch compute environment periodically cleans up Docker images to free disk space. That's fine for long-running services. For batch jobs, it's the wrong behavior you *want* the image cached on the instance so subsequent jobs skip the pull entirely.

The fix is setting `ECS_IMAGE_CLEANUP_ENABLED=false` in the ECS agent config at `/etc/ecs/ecs.config`. The way to inject this reliably is through the userdata script in the launch template attached to the Batch compute environment. The instance bootstraps with this config applied before ECS even starts, so cleanup never runs and the image stays resident.

After this change, the first job on a fresh instance still pulls the image. Every job after that on the same instance skips it entirely.

### Problem 2: Cold instances wiped out the cache benefit

Image caching only helps if the instance is still alive. If the compute environment scaled the instance down between jobs, we were back to a cold start fresh instance, fresh pull, same slow startup.

![Comparison of a cold instance that pulls the image on every job versus a warm instance that hits the cache and starts computing immediately](assets/cold-vs-warm.svg)

The fix here is a scale-down policy configured on the compute environment. By keeping the instance alive for a window after job completion rather than terminating immediately, we preserve the image cache across consecutive jobs. The instance stays warm, the image stays cached, and the next job starts at compute, not at pull.

This is a cost tradeoff you're paying for idle instance time during the scale-down window. But it's a controlled, bounded cost, and the latency savings on subsequent jobs more than justify it for bursty workloads where jobs tend to cluster.

One thing I made sure of: the scale-down window isn't hardcoded. I exposed it as a setting that's adjusted from the app itself. That way I'm not locked into keeping instances warm for a long time when nothing's coming. If I know a quiet period is ahead, I can shorten the window so idle instances spin down quickly; if I'm expecting a burst, I can widen it to keep compute warm and ready. The knob lives where the usage signal is, so the warm-instance cost only gets paid when it's actually going to pay off.

### Problem 3: Multi-GPU instances were being underutilized

The `g5.12xlarge` has four GPUs. Single-GPU jobs only claim one. That means if you're running single-GPU workloads on a multi-GPU instance, three GPUs are sitting idle while you're paying for all four.

The naive read of this is that multi-GPU instances are wasteful for single-GPU jobs. The correct read, once you combine it with the fixes above, is the opposite.

![A single g5.12xlarge with four GPUs: the first job pays the warmup cost while the next three jobs run immediately on the already-cached instance](assets/multi-gpu.svg)

With image caching and a scale-down policy in place, a warm multi-GPU instance is essentially pre-loaded and ready. The moment one job lands on it, the instance is already warm and already has the image. The next three jobs that arrive can each take one of the remaining GPUs no cold start, no image pull, just immediate compute.

What looks like a cost problem becomes a throughput multiplier. The first job pays the warmup cost. The next three ride for free on an instance that's already ready. For workloads that arrive in clusters which mine did this changed the economics of the whole system.

## What this looked like in practice

After all three fixes landed together, the behavior of the system changed noticeably.

The first job on a fresh instance pays a one-time warmup cost. Every subsequent job on that instance starts at compute. The scale-down window keeps the instance alive long enough that bursty workloads almost always land on warm compute. And on multi-GPU instances, that warmup cost is amortized across all four GPUs simultaneously.

The queue moves faster, the per-job cost is lower, and the system handles bursts without the spiky latency that was making results unpredictable.

## The bigger takeaway

AWS Batch gets you most of the way there with managed compute and native job queuing. Step Functions gives you the orchestration layer you need to build real pipelines on top of it. But the difference between "it works" and "it works well" came down to three things that aren't in the getting-started guide:

1. **Disable image cleanup** so your large GPU images stay cached on the instance.
2. **Use a scale-down policy** to keep instances warm between jobs.
3. **Treat multi-GPU instances as a throughput asset**, not a cost liability, once caching is in place.

None of these are obvious from the documentation. All of them made a measurable difference.

---

Thanks for reading! 🎉

If you're building on top of GPU batch infrastructure, these are the first three things I'd look at. If you've hit other rough edges with Batch or have your own tricks, feel free to reach out!
