---
title: "Investigating Terraform Timeouts for AWS S3 Lifecycle Configuration"
date: '2025-12-26T00:00:00-07:00'
draft: false
categories: ["Cloud", "Terraform", "AWS"]
tags: ["Terraform", "AWS", "S3", "Distributed Systems", "Consistency", "Reliability"]
description: "Why Terraform can time out applying S3 lifecycle rules: control-plane eventual consistency, waiter design, and measured propagation delays." 
---

Terraform occasionally times out while applying **S3 bucket lifecycle configuration** (see hashicorp/terraform-provider-aws issue #25939). This write-up explains what’s happening, validates the consistency behavior with small test suites, and outlines polling/waiter strategies that reduce timeouts without hammering APIs.

<!--more-->

## Background: Terraform’s Plan / Apply loop

Terraform’s workflow is:

1. **Plan**: compute the diff between desired config (HCL) and current state.
2. **Apply**: call provider APIs to converge reality to the plan.

![Terraform workflow: plan and apply](/images/terraform-s3/image1.png)

Terraform’s power comes from providers (e.g., AWS), but that also means Terraform inherits the consistency and propagation characteristics of cloud control planes.

## Problem statement

When applying an S3 lifecycle configuration, Terraform may write the lifecycle rule successfully but fail to observe the new configuration consistently before its timeout expires.

The symptom looks like: “we set it, but reads keep returning old or inconsistent state”.

## Root cause: S3 consistency is different for data plane vs control plane

S3’s **data plane** (object PUT/GET/LIST) is strongly consistent (since Dec 2020). But many **control plane** operations (bucket-level configuration metadata) can exhibit propagation delays.

That means:

- **Data plane**: write an object, immediately read it → you get the latest.
- **Control plane**: update bucket lifecycle configuration, immediately read it → you might get *old* config depending on which replica serves the read.

## Why Terraform waiters struggle here

Terraform uses a “waiter” pattern for resources where an immediate read after write may not reflect the update. Two problems show up under eventual consistency:

1. **Stale reads**: a GET after PUT can return the previous config.
2. **Flapping**: one GET might show success, then a subsequent GET might hit a lagging replica and show the old value again.

![Control plane: stale reads](/images/terraform-s3/image2.png)

![Fixed-interval polling](/images/terraform-s3/image3.png)

![Stability checks to avoid flapping](/images/terraform-s3/image4.png)

In practice, a fixed polling interval plus “N consecutive successes” can be:

- **Too slow on the happy path** (wastes time proving stability when the system already converged)
- **Not robust in the tail** (doesn’t adapt when propagation is slower than usual)

## Verification: measuring S3 behavior with simple test suites

To validate the model, I used two small Python test suites:

- **Sequential test**: serial operations (baseline).
- **Concurrent test**: parallel operations (“thundering herd”) to amplify tail behavior.

### Data plane (strong consistency)

Objective: after `put_object`, immediately `get_object`, verify the body.

| Metric | Sequential | Concurrent |
| :---: | :---: | :---: |
| Success rate | 100% | 100% |
| Mean latency | 99.26 ms | 302.38 ms |
| P99 latency | 225.72 ms | 414.45 ms |

![Data plane latency distribution](/images/terraform-s3/image5.png)

### Control plane (eventual consistency)

Objective: after `put_bucket_lifecycle_configuration`, poll `get_bucket_lifecycle_configuration` until the new config is visible.

| Metric | Sequential | Concurrent “racer” |
| :---: | :---: | :---: |
| Success rate | 100% | 100% |
| Mean propagation | 0.07 s | 1.85 s |
| Max propagation | 1.09 s | 11.83 s |
| “Instant” visibility (<1s) | 98% | 52% |

![Control plane propagation delays](/images/terraform-s3/image6.png)

Additional views:

![CDF of propagation time](/images/terraform-s3/image7.png)

![Polling count vs propagation time](/images/terraform-s3/image8.png)

## Optimization: testing waiter strategies under realistic delays

In small lab environments, propagation often looks “instant”, so timeouts are hard to reproduce. To approximate production behavior, I injected synthetic delays sampled from a heavy-tailed percentile distribution (based on reported/observed production-like latencies).

Then I compared a few waiter strategies:

- **Baseline**: fixed polling interval + stability check.
- **Extended timeout**: same strategy, longer maximum wait.
- **Adaptive**: learn a timeout budget from observed percentiles.
- **Hybrid**: front-load fast checks, then back off and apply a stability window.

![Strategy comparison summary](/images/terraform-s3/image9.png)

## Takeaways

- The bug/timeout behavior is explained by **control-plane propagation delay**, not by “S3 being broken”.
- Fixed waiters can be both wasteful (happy path) and brittle (tail).
- Better waiters are possible: adaptive/hybrid schedules can improve success rates while reducing API calls.

## References

- https://github.com/hashicorp/terraform-provider-aws/issues/25939
- https://www.youtube.com/watch?v=E7dWUJD57BU
- https://medium.com/@ksumedh001/5-critical-network-trade-offs-in-system-design-striking-the-right-balance-e2087c9bd032


## Test Suite Overview

The verification process involved:

1. **Sequential Test** (test\_s3\_consistency.py): Baseline measurements with serial operations to establish ground truth.  
2. **Concurrent Test** (test\_s3\_consistency\_concurrent.py): Stress testing with parallel threads to observe behavior under high-concurrency scenarios (Thundering Herd).

Data Plane Verification (Object Consistency)

* **Objective:** Verify that a GET request immediately following a PUT request returns the latest data.  
* **Method:**  
  1. Generate a unique key and body.  
  2. Perform put\_object.  
  3. Immediately perform get\_object and verify the content matches.  
  4. Measure latency and success rate.  
* **Concurrent Scenario:** 100 parallel threads performing atomic write-then-read operations.

Control Plane Verification (Eventual Consistency)

* **Objective:** Measure the propagation delay of bucket configuration changes.  
* **Method:**  
  1. Update the bucket's Lifecycle Configuration using put\_bucket\_lifecycle\_configuration.  
  2. Poll get\_bucket\_lifecycle\_configuration until the new configuration is visible.  
  3. Measure the time (propagation delay) between the PUT and the successful GET matching the new config.  
* **Concurrent "Racer" Scenario:** A "Writer" thread updates the config while a "Poller" thread aggressively polls for the change, using exponential backoff to measure propagation time with high precision.

## Verification Results

The following results are based on the execution of the test suites (N=100 iterations for each test).

Data Plane (Strong Consistency)

| Metric | Sequential Test | Concurrent Test |
| :---: | :---: | :---: |
| **Success Rate** | **100.00%** | **100.00%** |
| **Mean Latency** | 99.26 ms | 302.38 ms |
| **P99 Latency** | 225.72 ms | 414.45 ms |

Observation:

The Data Plane demonstrated Strong Consistency. In 100% of the cases, the GET request immediately following the PUT returned the correct object. Even under concurrent load, while latency increased, consistency was maintained.

![][image5]

Figure 4: Data Plane Latency Distribution

Control Plane (Eventual Consistency)

| Metric | Sequential Test | Concurrent "Racer" Test |
| :---: | :---: | :---: |
| **Success Rate** | 100.00% | 100.00% |
| **Mean Propagation** | 0.07 s | 1.85 s |
| **Max Propagation** | 1.09 s | **11.83 s** |
| **Instant Visibility (\<1s)** | 98.0% | 52.0% |

Observation:

The Control Plane demonstrated Eventual Consistency.

* In the **Sequential** test, changes were often visible almost immediately (Mean: 0.07s), but occasional delays were observed.  
* In the **Concurrent** test, the eventual nature became much more apparent. The mean propagation time increased to 1.85s, with a worst-case delay of **11.83s**.  
* Only 52% of concurrent updates were instantly visible, confirming that control plane operations are not strongly consistent.

![][image6]

Figure 4.2: Control Plane Propagation Delays

Additional Control Plane Insights

The following charts detail the tail latency and the relationship between API calls and propagation time during the concurrent "Racer" test.

![][image7]

Figure 4.3: Cumulative Distribution Function (CDF) of Propagation Time

![][image8]

Figure 4.4: API Polling Count vs. Propagation Time

## Conclusion

The empirical evidence supports the AWS S3 consistency model documentation:

1. **S3 Objects (Data Plane)** exhibit **Strong Consistency**. Applications can rely on reading the latest data immediately after a write.  
2. **S3 Bucket Configurations (Control Plane)** exhibit **Eventual Consistency**. Changes to bucket policies (like Lifecycle rules) may take seconds or even longer to propagate across the system. Applications must be designed to tolerate these delays and should not assume immediate visibility of configuration changes.

# Optimization

## 

## Part 1: Why AWS Learner Lab Cannot Reproduce the Timeout Issue

The Problem: Two Different AWS Environments

**AWS Production Environment (Real World):**

* Massive distributed system spanning multiple data centers  
* Configuration changes must propagate across countless nodes  
* **Eventual consistency**: Changes take time to become visible everywhere  
* Propagation delays range from seconds to minutes (heavy-tailed distribution)

**AWS Learner Lab Environment (Our Testing Environment):**

* Simplified educational environment  
* Smaller scale, fewer nodes  
* Configuration changes propagate **almost instantly** (\< 1 second typically)  
* Cannot reproduce real-world propagation delays

Why This Matters

When you run the baseline test in Learner Lab without simulation:

PUT request → Returns in \~0.5s  
First GET check (3s later) → Configuration already visible ✓  
Result: Success in \~3 seconds, every single time

**But in production** (based on GitHub Issue \#25939 real data):

* 50% of requests take up to **48 seconds** to propagate  
* 95% of requests take up to **156 seconds** to propagate  
* 1% of requests take up to **312 seconds** to propagate

The current Terraform timeout of **180 seconds** means:

* ✓ Covers approximately 95% of cases (P95 \= 156s)  
* ✗ **Fails approximately 5% of the time** when propagation exceeds 180s  
* This causes deployment failures in production\!

## Part 2: Understanding Percentiles (P50, P90, P95, P99)

What Are Percentiles?

Think of percentiles as answering: **"How long do I need to wait to cover X% of all cases?"**

**Simple Example with 100 students' test scores:**

* **P50** (50th percentile/median): 50% scored below this, 50% above  
* **P90** (90th percentile): 90% scored below this, only 10% scored higher  
* **P95** (95th percentile): 95% scored below this, only 5% scored higher

Applied to AWS S3 Propagation Times

From GitHub Issue \#25939 **real production data**:

| Percentile | Time | Meaning |
| ----- | ----- | ----- |
| P50 | 48 seconds | 50% of requests propagate within 48s |
| P75 | 82 seconds | 75% of requests propagate within 82s |
| P90 | 142 seconds | 90% of requests propagate within 142s |
| P95 | 156 seconds | 95% of requests propagate within 156s |
| P99 | 234 seconds | 99% of requests propagate within 234s |
| Max | 312 seconds | Worst case observed |

**Visual Distribution:**

Fast cases (P0-P50):     ████████████░░░░░░░░  0-48s    (50% of requests)  
Moderate (P50-P90):      ████████░░░░░░░░░░░░  48-142s  (40% of requests)  
Slow (P90-P95):          ██░░░░░░░░░░░░░░░░░░  142-156s (5% of requests)  
Very slow (P95-P99):     █░░░░░░░░░░░░░░░░░░░  156-234s (4% of requests)  
Extreme (P99-Max):       ░░░░░░░░░░░░░░░░░░░░  234-312s (1% of requests)

Why This Is Called "Heavy-Tailed"

Most requests are fast (median \= 48s), but the **tail** of slow requests extends much longer:

* P99 (234s) is **4.9 times longer** than P50 (48s)  
* This is why fixed timeouts fail—they cannot accommodate the unpredictable slow cases

## Part 3: The Simulation Solution

How We Simulate Real Production Behavior

Since Learner Lab propagates instantly, we **inject artificial delays** that match the real production distribution:

def \_generate\_propagation\_delay(self):  
    """  
    Generate realistic propagation delay based on percentile distribution  
    """  
    rand \= random.random()  \# Random number 0.0 to 1.0  
      
    if rand \< 0.50:  
        \# 50% of requests: P0-P50 (0-48 seconds)  
        return random.uniform(0, 48\)  
    elif rand \< 0.75:  
        \# Next 25% of requests: P50-P75 (48-82 seconds)  
        return random.uniform(48, 82\)  
    elif rand \< 0.90:  
        \# Next 15% of requests: P75-P90 (82-142 seconds)  
        return random.uniform(82, 142\)  
    elif rand \< 0.95:  
        \# Next 5% of requests: P90-P95 (142-156 seconds)  
        return random.uniform(142, 156\)  
    elif rand \< 0.99:  
        \# Next 4% of requests: P95-P99 (156-234 seconds)  
        return random.uniform(156, 234\)  
    else:  
        \# Rare 1% of requests: P99-Max (234-312 seconds)  
        return random.uniform(234, 312\)

How the Simulation Works Step-by-Step

**Without Simulation (Learner Lab default):**

PUT request → 200 OK (0.5s)  
    ↓  
GET check \#1 (at 3s) → Configuration visible ✓  
    ↓  
SUCCESS (total time: approximately 3s)

**With Simulation (mimicking production):**

PUT request → 200 OK (0.5s)  
    ↓  
Generate simulated delay: 156s \[This would happen in production\]  
    ↓  
GET check \#1 (at 3s) → Simulated: Still propagating ✗  
GET check \#2 (at 6s) → Simulated: Still propagating ✗  
GET check \#3 (at 9s) → Simulated: Still propagating ✗  
    ...  
GET check \#52 (at 156s) → Simulated: NOW visible ✓  
    ↓  
SUCCESS (total time: approximately 156s)

Key Implementation Detail  
def check\_configuration\_match(self, expected\_config):  
    """Override to simulate eventual consistency"""  
      
    \# First check: Generate the delay for this test  
    if not hasattr(self, '\_simulated\_delay'):  
        self.\_simulated\_delay \= self.\_generate\_propagation\_delay()  
        self.\_propagation\_start \= time.time()  
        print(f" \[Sim: {self.\_simulated\_delay:.1f}s\]", end='')  
      
    \# Check if enough time has passed  
    elapsed \= time.time() \- self.\_propagation\_start  
    if elapsed \>= self.\_simulated\_delay:  
        \# Propagation "complete" \- do real AWS check  
        return super().check\_configuration\_match(expected\_config)  
    else:  
        \# Still "propagating" \- return False  
        return False

**What this does:**

1. On first check, generate a random delay (e.g., 156s) matching production distribution  
2. Keep returning `False` (not ready) until that much time has actually elapsed  
3. Only then allow the real AWS check to succeed

This creates **realistic test conditions** that match production behavior\!

## Part 4: Test Design \- Four Strategy Comparison

Strategy 2.1: Baseline (Current Terraform)

**Configuration:**

* Polling interval: Every 3 seconds (fixed)  
* Timeout: 180 seconds

**Behavior:**

* Checks 60 times (180s ÷ 3s \= 60 checks)  
* **Coverage: P95** (156s \< 180s)  
* **Expected failures: approximately 5%** (cases taking \> 180s)

**Theoretical analysis with simulation:**

* Cases under 180s: Success ✓  
* Cases over 180s (5% of requests): **Timeout failure** ✗

Strategy 2.2: Extended Timeout

**Configuration:**

* Polling interval: Every 3 seconds (fixed)  
* Timeout: 600 seconds (10 minutes)

**Behavior:**

* Checks 200 times (600s ÷ 3s \= 200 checks)  
* **Coverage: Beyond P99** (234s \< 600s, even Max 312s \< 600s)  
* **Expected failures: \<1%**

**Trade-offs:**

* ✓ Higher success rate  
* ✗ **No API call reduction** \- still checking every 3s  
* ✗ Wastes time on genuine failures (waits full 10 minutes)

Strategy 2.3: Adaptive Learning (Improved)

**Configuration:**

* Polling interval: Variable (2-15s, adjusts based on elapsed time)  
  * 0-30s: Every 2 seconds (dense early)  
  * 30-60s: Every 4 seconds  
  * 60-120s: Every 8 seconds  
  * 120-180s: Every 10 seconds  
  * 180s+: Every 15 seconds (sparse late)  
* Timeout: Dynamic (300-600s, learned from past 20 tests)

**Innovation: Learns from experience**

\# After each successful test, remember how long it took

self.historical\_times.append(result\['propagation\_time'\])

\# Calculate timeout as: mean \+ 3 standard deviations

timeout \= mean(historical\_times) \+ 3 × stdev(historical\_times)

**Why this works:**

* **Higher minimum (300s vs 180s)**: Covers P90 (142s) during bootstrap, reducing early failures  
* **3×stdev vs 2×stdev**: Covers 99.7% of distribution instead of 95%, handles tail latency better  
* **Faster learning (3 tests vs 5\)**: Enters adaptive mode sooner while still being conservative  
* **Adaptive intervals (2-15s)**: Dense early polling for common cases (P50), sparse late for tail (P99)

**Adaptation examples:**

* If recent tests averaged 50s: Sets timeout ≈ 200-250s (covers typical patterns \+ buffer)  
* If recent tests averaged 150s: Sets timeout ≈ 450-500s (adapts to slower periods)  
* Continuously adjusts to changing AWS behavior patterns

Strategy 2.4: Hybrid Polling

**Configuration:**

| Time Range | Polling Interval | Rationale |
| ----- | ----- | ----- |
| 0-30s | 2 seconds | Most requests complete quickly |
| 30-60s | 4 seconds | Moderate cases |
| 60-120s | 8 seconds | Less common |
| 120-600s | 15 seconds | Rare tail cases |

**Inspiration: "The Tail at Scale" (Jeff Dean)**

* **Dense early polling** captures the common case (P50 @ 48s)  
* **Sparse late polling** handles rare slow cases without excessive API calls

**Example with P95 case (156s propagation):**

Baseline strategy (3s fixed):  
  Total checks: 52 times (156s ÷ 3s)  
    
Hybrid strategy (variable):  
  0-30s:    15 checks (30s ÷ 2s)  
  30-60s:   7 checks (30s ÷ 4s)  
  60-120s:  7 checks (60s ÷ 8s)  
  120-156s: 2 checks (36s ÷ 15s)  
  Total:    31 checks  
    
API call reduction: 40% fewer calls\! (31 vs 52\)

## Part 5: What We're Measuring

Primary Metrics

**1\. Success Rate**

Success Rate \= (successful\_tests / total\_tests) × 100%  
Goal: \>95% (at least P95 coverage)

**2\. Propagation Time Distribution**

From successful tests, calculate:  
\- P50 \= median propagation time  
\- P90 \= 90th percentile  
\- P95 \= 95th percentile  
\- P99 \= 99th percentile

**3\. API Call Efficiency**

Total API calls \= 1 (PUT) \+ number\_of\_GET\_checks  
Average per test \= mean(api\_calls)

Why We Run 30 Iterations Per Strategy

* **Statistical validity**: 30 samples provide reliable percentile estimates  
* **Distribution coverage**: With 30 tests, we expect:  
  * Approximately 15 tests in P0-P50 range (fast)  
  * Approximately 12 tests in P50-P90 range (moderate)  
  * Approximately 3 tests in P90-P99 range (slow)  
  * Approximately 0-1 tests in P99+ range (very slow)

This gives us comprehensive data across the entire distribution\!

## Summary: The Complete Picture

**The Challenge:**

* AWS Learner Lab cannot reproduce real production propagation delays  
* Real production shows heavy-tailed latency (P99 is 5 times longer than P50)

**Our Solution:**

* Inject simulated delays matching empirical production distribution  
* Test four different waiter strategies under realistic conditions

**Expected Outcomes:**

* **Baseline**: Approximately 95% success, but 60 API calls per test  
* **Extended**: Approximately 99%+ success, but still 200 API calls (wasteful)  
* **Adaptive**: Approximately 95%+ success, learns optimal timeout  
* **Hybrid**: Approximately 99%+ success, **40% fewer API calls** (best of both worlds)

This research demonstrates that **smart polling strategies** can simultaneously improve success rates AND reduce API costs\!

![][image9]

# 

# 

# References

[https://medium.com/@ksumedh001/5-critical-network-trade-offs-in-system-design-striking-the-right-balance-e2087c9bd032](https://medium.com/@ksumedh001/5-critical-network-trade-offs-in-system-design-striking-the-right-balance-e2087c9bd032)

[https://www.youtube.com/watch?v=E7dWUJD57BU](https://www.youtube.com/watch?v=E7dWUJD57BU)

# 