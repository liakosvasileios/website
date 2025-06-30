+++
title = "AVX THROTTLING"
date = 2025-06-30
tags = ["malware", "operating systems", "windows"]
description = "Thoughts about exploiting AVX-induced frequency throttling behavior"
+++

---

# AVX-induced frequency throttling behavior could potentially be exploited as a side-channel attack vector

---

## Background on AVX Frequency Throttling

On Intel Skylake and newer CPUs, executing wide AVX2 (256-bit) or AVX-512 (512-bit) instructions causes the CPU to throttle down its frequency, even when thermal/power limits aren’t reached. This happens to avoid instability due to voltage droop, and the throttling is:
- Instruction-dependent
- Thread-specific
- Time-sensitive

---

## Potential Exploit Mechanism

This behavior introduces a measurable timing side effect based on:
- The type of instructions executed
- How often they're executed
- Whether throttling occurs

This makes it observable by other processes or threads, **even without privileged access**.

This could be used in a side channel as follows:
1. Victim executes AVX-512-heavy instructions → causes CPU to enter a lower turbo bin (L2).
2. Attacker co-located on same physical core (different logical thread) observes a drop in performance due to reduced shared core frequency.
3. By timing specific operations, the adversery can infer:
    - If AVX-512 instructions were being run.
    - Potentially, what kind of data was being processed (if it correlates with AVX usage).
    - Maybe even narrow down specific code paths taken by the victim.

This is similar in concept to:
- Flush+Reload or Prime+Probe cache attacks.
- Branch predictor timing attacks
- Power-based side channels

But instead of cache or branch predictors, it's using dynamic frequency scaling as the source of leakage.

---

## Security Impact
- This attack requires core co-residency (i.e., sharing a core via SMT/Hyper-Threading).
- It's less granular than Spectre/Meltdown because it leaks coarse information (frequency, not memory content).
- But it can be used in combined attacks or side-channel amplifiers, especially for detecting:
    - AVX usage in cryptographic code.
    - Specific workloads running in secure enclaves (e.g., Intel SGX)

---

## Known Research
- "Hertzbleed: Turning Power Side-Channel Attacks
Into Remote Timing Attacks on x86" [Reference](https://www.usenix.org/system/files/sec22-wang-yingchen.pdf)
- AVX Spectre: Leaking Data through AVX Frequency Scaling" [Reference](https://arxiv.org/pdf/2102.06809)
- "PLATYPUS" attack (Leveraged energy measurement registers (RAPL) to observe power behavior of victim code).

---

## Mitigations
- Disable AVX-512 support in BIOS (common in cloud environments).
- SMT mitigation: Use core pinning or disable Hyper-Threading to prevent attacker-victim co-location.
- Noise injection or core isolation for sensitive tasks.
- Intel could (and sometimes does) apply microcode patches to reduce observability or make transitions more deterministic.

--- 

## AVX-induced throttling as a potentialaid in exploiting speculative execution or beanch prediction

Indirectly, as it would not be the primary mechanism for such an exploit like in Spectre or Meltdown.

## What Spectre/Meltdown Rely On
Spectre and Meltdown fundamentally exploit:
- Speculative execution: CPU executes ahead-of-time to gain performance.
- Mispredicted branches: Speculation can be steered into attacker-controlled code paths.
- Side channels (e.g., cache timing): Used to observe microarchitectural effects of speculation (e.g., which cache lines were loaded).
These attacks leak data that should not be **architecturally accessible**.

## Where AVX Frequency Throttling Comes In
AVX throttling:
- Is not speculative
- Does not bypass privilege checks
- Has low resolution (multi-microsecond granularity)
- Is instruction-pattern triggered, not data-dependent

So, it cannot directly be used to:
- Force speculative execution
- Influence branch predictor state
- Read memory bypassing perimissions

**BUT, it can still help in:**

## Possible Roles in a Speculative Execution Attack
1. Covert Channel Enhancement
If two processes (e.g., attacker and victim) are colluding or co-resident:
- AVX throttling can be used as a signal mechanism, where one process modulates CPU frequency through AVX instructions, and another detects it via timing.
- This can exfiltrate secret data leaked speculatively and stored in a register (rather than memory/cache).

2. Speculation Suppression Timing Oracle
AVX throttling may affect timing of speculative execution paths:
- The attacker might use AVX instructions to slow down execution of certain instructions or flush the pipeline more often.
- This could serve as a timing oracle, helping determine if speculation occurred or not.

3. Improved Branch Predictor Side Channels
If a branch predictor is being attacked (e.g., Spectre v1-style), and you want to infer which code path was speculatively taken:
- You could time how long it takes for the victim to recover from AVX throttling (if you suspect they used AVX in the speculative path).
- This could offer indirect confirmation that speculation went through a path involving AVX instructions.

## Hypothetical Example Scenario
Imagine you craft a Spectre v1 gadget that speculatively runs AVX-512 instructions only if a secret bit is 1. Then:
- If the bit is 1 → CPU enters AVX-512 throttled state → attacker sees performance drop on co-located core.
- If the bit is 0 → no AVX → no throttling → attacker sees normal speed.

This creates a bit-leaking side channel based on speculative AVX-induced frequency transitions.
    This has actually been proposed in the "AVX Spectre" paper (2021), where speculative execution is used to induce throttling, and timing is observed externally.

## TL;DR

AVX frequency throttling can act as a low-resolution timing side channel, which might be exploited similarly to cache timing attacks—especially if the attacker shares a physical core with the victim. While not as powerful as Spectre/Meltdown, it adds to the arsenal of side-channel vectors and could help fingerprint or infer workload behavior.
Also, AVX throttling can be abused in conjunction with speculative execution, especially to:

- Observe side effects of speculation

- Build new kinds of covert/co-resident channels

- Enhance traditional Spectre-type attacks with new timing oracles

**But it is not a standalone speculative execution exploit like Spectre/Meltdown. It acts as a side-channel amplifier or observer.**