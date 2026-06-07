# ASLR Information Leakage Under Same-Host Probing

Address Space Layout Randomization (ASLR) is an operating system defense that hides where a process's code, stack, and heap live in memory. I had recently done some work with the MILP optimization algorithm, and started this project after reading about Windows DLLs having fixed starting addresses - whether ASLR is enabled or not. Computers are not very good at true randomization. I got curious: what could an unprivileged process on a Windows host learn about the address-space layout of an unrelated process running on the same machine?

## Thesis

A controlled probe process X, running on a Windows host, can reduce the conditional entropy of an unrelated co-resident process Y's address-space layout by a quantifiable amount. The leakage comes entirely from architectural observations the probe makes about itself, with no privileged access to Y. It works with ASLR enabled and relies only on information the probe is normally allowed to read. Side channels like CPU cache timing are out of scope.

## Why this matters

ASLR is a standard memory-corruption mitigation on modern Windows. The OS mixes random bits into each region's base address to prevent layout prediction. The mitigation might not be sufficient. Per-boot system DLL sharing, correlated allocator behavior, and predictable internal structures may leak information about the target system.

## Research questions

* What is the conditional entropy of a target process layout given a probe's observations of its own layout and public OS state?
* Which leaks come from per-boot system-wide randomization, and which come from deterministic behavior inside the heap and stack allocators?
* Which leaks result from higher-order correlations between memory objects?
* Can constraint propagation and search optimization produce attempt-ordering strategies that cut the expected number of guess attempts relative to a uniform-random baseline?
* Which mitigations close the largest gaps, and what are their costs?

## Scope

The adversary is a same-user, same-host attacker on a pinned Windows x64 build. The probe is a normal user-mode process with no special privileges. It reads only architectural state and values it is explicitly allowed to read. We make no inferences from hidden CPU hardware timing. The target is a generic user-mode application.

**Stretch goal:** Cross-user, same-host attacks on ARM64 Windows, adding one microarchitectural observation channel.

**Out of scope:** Multiple Windows versions, cross-host adversaries, hypervisor escapes, kernel-space targets, and Kernel ASLR.

## Success criteria

* A reproducible measurement harness that collects boot-fresh samples of process memory layouts, at a sample size justified by the highest-entropy region.
* Per-region conditional entropy estimates with bootstrap confidence intervals.
* At least one optimization-based attempt-ordering result that shows a reduction in expected guess attempts versus a uniform-random baseline.

# Threat Model

## Actors

**Probe:** This is a standard Windows user-mode process. I start it as the same user running the target. It has no administrator rights, no debug rights, and no kernel components.
**Target:** A normal Windows user-mode process started independently by the same user. It has no awareness of the probe.
**Host:** A pinned Windows configuration with ASLR, DEP, and CFG enabled. There is no third-party EDR agent installed.

## Probe capabilities
The probe can read its own PEB, TEB, loaded module list, and heap and stack bases. It can call VirtualQuery on its own address space and read standard file system metadata like system DLL timestamps. It can also spawn child processes, use standard time syscalls, and load arbitrary public DLLs.

## Excluded capabilities
We strictly exclude inference from hidden CPU hardware behavior. The probe does not read secrets from CPU cache timing, branch predictors, or TLBs. We also exclude debug privileges, syscall hooking, API monitors, and kernel drivers. We are not relying on direct information leaks from the target itself, and there is no physical access.

## Target exposure
We assume the target carries an independent memory-corruption vulnerability that gives the attacker control flow if the layout is known. This project does not model how that vulnerability gets triggered. We only measure the probability that the probe's prior inference produces a good enough layout estimate.

## Observables
On each fresh boot, we record the probe's image base, the base address of every loaded DLL, the process heap base, the thread stack base, and the TEB/PEB addresses. We also track fixed-size allocations at known program points, and we record the same observations for a spawned child process. The target's layout is recorded strictly as ground truth for evaluation. 

## Inference target
For each target region, we produce a posterior distribution representing our updated belief about where that region sits based on the probe's observations. We will report the posterior entropy, the information gain compared to maximum theoretical entropy, and the single most likely address under the posterior.

## Reliability metrics
* **Information-theoretic:** Per-region conditional entropy with bootstrap confidence intervals.
* **Operational:** Expected number of attempts before a uniformly random sample from the posterior matches the actual layout, compared against a random baseline.

## Validation methodology
We run a paired probe configuration by launching the probe and target in a known order on a fresh boot. Both record their full layouts. The probe computes a posterior over the target, and we score that posterior against the target's actual layout. 

## Known Risks
* **Sample contamination:** The probe's allocation behavior could alter the target's layout. We handle this by varying the launch order and stratifying the analysis.
* **Virtualization quirks:** Hypervisors might produce different distributions than bare metal. We will validate a subset of measurements on bare metal before finalizing.
* **OS update drift:** Updates can change the loader and break our findings. We pin to a specific Windows build and snapshot the VM to prevent this.
* **Selection bias:** A single target process is not representative. We run the analysis against several distinct target processes to ensure it generalizes.

## What this project is not

* An exploit. Exploitation of results is out of scope.
* A side-channel attack. The goal is to find information leaks without them.
* Vulnerability research against a specific application.

## Repo layout

- `docs/` contains the project plan, threat model, references, and study guide.
- `src/` is where the code will go when I'm done reading up on ASLR.
- `data/` contains collected samples (gitignored).
- `results/` contains analysis outputs (gitignored).
