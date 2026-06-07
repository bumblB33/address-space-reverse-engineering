# Project Concepts - Study Guide

## Introduction

This guide covers Address Space Layout Randomization (ASLR) on Windows. The goal is to measure how much randomness actually survives in practice and how much one program can infer about another.

The main point is that ASLR is less secure than expected due to shared DLLs, and biases in memory allocation create subtle leaks. Accurate measurement is difficult, and brute-forcing high-randomness regions especially so.

## Part 1: Essential Concepts

### What is ASLR?

Address Space Layout Randomization (ASLR) randomizes where programs and libraries load in memory. This prevents attackers from predicting the address they need to hijack during an exploit. If the address changes every time a program runs, a wrong guess usually crashes the target instead of granting control.

Entropy measures the amount of randomness in bits. A region with n bits of entropy has 2^n equally likely places it could be.

- 8 bits = 256 possibilities (guessable quickly).
- 19 bits = roughly 524,000 possibilities (annoying but not hopeless).
- 24+ bits = millions of possibilities (brute force is impractical).

Rule of thumb: An attacker needs roughly 2^(n-1) guesses to find a value with n bits of remaining randomness. Every bit removed from the defense halves the attacker's work.

### The Alignment Trap

Windows does not place images at byte-random addresses. It aligns them to 64 KB boundaries (multiples of 0x10000). This means the lowest 16 bits of an address are always zero and carry no randomness.

If the OS spreads images over a 1 TB range (2^40 bytes), the actual number of distinct slots is 2^40 / 2^16, which equals 2^24. This results in 24 bits of entropy, not 40. Whenever your calculated range and actual entropy do not match, alignment is usually the cause.

### Key Terminology

- Image Base: The starting address of a loaded .exe or .dll.
- DLL: A shared library. Windows attempts to load each one once per boot session and share it across all processes.
- Heap: Memory allocated by the application (malloc/HeapAlloc). This is per-process.
- Stack: Per-thread scratch memory for function calls and local variables.
- PEB (Process Environment Block): A per-process structure holding the image base and list of loaded modules.
- TEB (Thread Environment Block): A per-thread structure holding stack bounds and a pointer back to the PEB. On x64 systems, the CPU GS register points to the TEB.
- Loader: The Windows component that maps images into memory and selects their addresses.
- KnownDLLs: Core system DLLs pre-loaded into a shared section at boot to improve speed and prevent shadowing attacks.

### Information Theory Basics

- H(Y): The total entropy of the target's layout (the secret).
- I(X; Y): Mutual information between the probe's layout (X) and the target's layout (Y). This represents the bits of the target you learn by looking at your own process.
- The Goal: Minimize I(X; Y). If your probe shares a DLL base with the target, you learn exactly where that DLL is in the target, leaking all its entropy.

### Leak Mechanisms

There are two main channels through which ASLR entropy leaks:

#### Channel A: Shared DLL Bases

This is the strongest and most well-known leak. A specific DLL receives one random address per boot session. Every process loading that DLL gets the same address until the system reboots.

Because Windows uses position-dependent code for many DLLs, it cannot share the same physical copy across different virtual addresses. Therefore, if your probe reads where its own ntdll.dll landed, it knows exactly where ntdll.dll is in any other process running on the same machine during the same boot session. For these shared system DLLs, residual entropy is effectively zero.

#### Channel B: Allocator Bias

Private regions like the heap and stacks are unique to each process and cannot be read directly by a probe. However, the memory allocator is not perfectly uniform. It hands out memory in a deterministic order, making the distribution predictable even when no single address is shared.

For example, on older Windows versions, the PEB was guessable 25% of the time, whereas a uniform distribution would expect 6.25%. This proves that effective entropy is often lower than the theoretical log2(range). 

### The Statistics Problem

Estimating entropy is notoriously difficult. The obvious method is to reboot many times, record where a region lands, build a histogram, and calculate entropy. This is called the plug-in estimator, and it is biased downward.

With K possible values and N samples, the estimator underestimates true entropy by roughly (K-1)/(2N). There is no unbiased estimator for entropy. To get a reliable estimate, you need sample sizes far larger than the number of possible outcomes (N >> K).

A 24-bit region has roughly 16.7 million possible values. Achieving N >> K would require tens of millions of reboots, which is physically impractical. In this undersampled regime, standard confidence intervals become misleadingly tight and incorrect.

**Strategy for Measurement:**

- Measure empirical entropy only for low-entropy regions (like the EXE base) where data collection is feasible.
- Use parametric models to calculate entropy analytically for high-entropy regions.
- Use better estimators like NSB (Nemenman-Shafee-Bialek) where empirical estimation is necessary.

## Part 2: Checkpoints and Exercises

### 1. Virtual Memory and x64 Address Space

- Primary Source: Memory Management chapter in Windows Internals.
- Secondary Source: User-mode virtual address space overview on Microsoft Learn.
- Exercise: Write a C program to print your own image base, heap base, TEB, and PEB addresses. Run this across several reboots and track which region of the address space each falls into.
- Checkpoint: Explain the standard x64 layout, which bits are randomizable, and why the HIGH_ENTROPY_VA flag matters for x64 versus x86.

### 2. Process Structures (PEB, TEB, EPROCESS)

- Primary Source: Processes chapter in Windows Internals.
- Secondary Source: Vergilius Project for PEB/TEB layouts on your pinned build.
- Exercise: Attach WinDbg to a running process and manually walk the module list. Find the three module lists by name.
- Checkpoint: Locate the base of a loaded module from inside the process without using documented APIs. Understand why multiple module lists exist and know the offset of ImageBaseAddress in the PEB.

### 3. The Loader and DLL Loading

- Primary Source: Image loader section in the Processes chapter of Windows Internals.
- Secondary Source: Chappell loader notes and Yosifovich System Programming examples.
- Exercise: Set a breakpoint on ntdll!LdrLoadDll in a fresh process. Step through a load event and trace the chosen address back to where the decision was made.
- Checkpoint: Explain what LdrpFindOrMapDll does. Describe how a DLL's preferred base interacts with ASLR and why system DLLs receive special treatment via KnownDLLs.

### 4. KnownDLLs and System DLL Relocation

- Primary Source: KnownDLLs articles by Geoff Chappell.
- Secondary Source: Windows Internals section on session-level KnownDLLs.
- Exercise: Use WinObj to list the KnownDlls section. Dump headers for each entry with dumpbin /headers and check ASLR flags. Compare base addresses across different running processes.
- Checkpoint: Understand why critical system DLLs share a base across all processes in a single boot session and when that base changes.

### 5. ASLR Mechanics

- Primary Source: Microsoft Learn Exploit Protection reference for ASLR.
- Secondary Source: Methodology section in the Binosi et al. CCS 2024 paper.
- Exercise: Find registry settings for image relocation. Build a probe that samples your own ntdll.dll base across many reboots to compute empirical bit-entropy. Compare this against documented budgets.
- Checkpoint: Explain exactly when system DLL bases and per-process randomizable regions are selected. Know the entropy budget for each region under both High Entropy and legacy modes.

### 6. Memory Manager Behavior

- Primary Source: VAD trees and allocation policies in Windows Internals.
- Secondary Source: Recent talks by Windows memory manager researchers.
- Exercise: Have your probe call VirtualAlloc repeatedly without a preferred address and log the bases. Repeat with MEM_TOP_DOWN and compare distributions. Then load a non-known DLL multiple times and log results.
- Checkpoint: Explain bottom-up versus top-down allocation. Describe how the kernel places non-preferred allocations and whether allocations within or across processes are correlated.

### 7. High-Entropy ASLR

- Primary Source: Microsoft Learn pages on /HIGHENTROPYVA.
- Secondary Source: Current literature on low-entropy DLLs.
- Exercise: Build two DLLs differing only in the HIGH_ENTROPY_VA flag. Load them into your probe across multiple reboots. Compute empirical bit-entropy and compare with documentation.
- Checkpoint: Explain why some modern DLLs still have low entropy. Describe how LARGEADDRESSAWARE interacts with HIGH_ENTROPY_VA and how to identify a low-entropy DLL in a binary you do not control.

## Part 3: Background Knowledge and Data

### x64 ASLR Entropy Budgets

Based on Matt Miller (MSRC) and recent empirical studies (CCS 2024):

| Region | Entropy (Bits) | Type | Notes |
| --- | --- | --- | --- |
| DLL Base (High Entropy, >4GB) | 19 | Documented | MSRC 2013 |
| DLL Base (Legacy, <4GB) | 14 | Documented | MSRC 2013 |
| EXE Base (High Entropy, >4GB) | 17 | Documented | MSRC 2013 |
| EXE Base (Legacy, <4GB) | 8 | Documented | Matches Vista measurements |
| Bottom-up Allocations (Heap/Stack) | 24 | Documented | 1 TB range, 64KB aligned |
| Top-down Allocations (MEM_TOP_DOWN) | 17 | Documented | 8 GB range |

Note on Alignment: The 64 KB alignment fixes the lowest 16 bits. A 1 TB range yields 24 bits, not 40.

### Recent Empirical Findings (Windows 11)

Data from Binosi et al., "The Illusion of Randomness" (ACM CCS 2024), measured on Windows 11 x64:

- Thread Stack: ~28.15 bits.
- Heap (Small malloc): ~26 to 32 bits (size-dependent).
- Heap (Large malloc): ~22 to 29 bits.
- VirtualAlloc (Large): ~18.6 to 19.7 bits.
- EXE Base: ~16.99 bits.
- DLL Base: ~18.97 bits.

**Key Qualitative Findings:**

- Windows 11 achieves up to 31 bits of entropy in peak areas, but the most critical areas (EXE and DLLs) remain at around 19 bits.
- Allocations follow a triangular (Irwin-Hall) distribution rather than a uniform one, creating a predictable most-common value that reduces the effort needed to de-randomize sections.
- No significant correlations were found between regions on Windows that aid an attacker, unlike on Linux.

**Outstanding Gaps:**

- PEB and TEB: These remain unquantified for x64 in current public literature. No official Microsoft documentation provides a specific bit count for these structures on x64.

### Flags and Dependencies

- /HIGHENTROPYVA: Allows placement above 4 GB. Requires /LARGEADDRESSAWARE and /DYNAMICBASE. Enabled by default for 64-bit images.
- Low Entropy DLLs: Some modern DLLs remain low-entropy if built without /HIGHENTROPYVA (or with :NO). Check PE characteristics using dumpbin /headers.

### The Two Leaks Explained

- Channel A (Shared Bases): A DLL gets one random base per boot. All processes share it. If your probe loads kernel32.dll, you know the address in the target. Residual entropy is near zero for these modules.
- Channel B (Allocator Bias): Private memory is per-process. The leak is statistical bias in how the allocator distributes addresses. Even if addresses differ, the probability distribution is skewed, making some addresses much more likely than others.

### Legal Observation Methods (Same User, Non-Admin)

You can legally observe your own process memory using documented APIs:

- Walk VA Space: VirtualQuery + MEMORY_BASIC_INFORMATION.
- Image Base: GetModuleHandle(NULL).
- Loaded Modules: EnumProcessModulesEx + GetModuleInformation.
- Stack Bounds: GetCurrentThreadStackLimits.
- Heaps: GetProcessHeap().
- PEB Access: NtCurrentTeb()->ProcessEnvironmentBlock.

Warning on Undocumented Internals: While intrinsics like __readgsqword(0x60) work on x64 to access the PEB, specific offsets are undocumented and build-specific. Do not hardcode offsets across different Windows builds. Always verify offsets for your pinned build.

Threat Model Note: A same-user process can technically query another sibling process using OpenProcess(PROCESS_QUERY_INFORMATION) and VirtualQueryEx. While legal, this is considered "direct observation." In rigorous studies, this path is reserved for ground-truth verification, while the "probe" should rely only on inference methods to simulate real-world attack constraints.

## Open Questions and Next Steps

### Pending Items

- O1: PEB/TEB Quantification: We lack specific entropy numbers for PEB and TEB on x64.
    - Action: Sample NtCurrentTeb() offsets across many reboots (target >100,000 samples). Look for bias patterns.
- O2: Loader Internals: Exact function names for the loader in modern Windows (Win11) need confirmation against current symbols, as older references refer to Win8-era functions.
    - Action: Verify kernel function names using symbol servers or tools like pdbextract.
- O3: Per-Boot Sharing Confirmation: Verify empirically that image bases remain fixed across processes in a single boot session on your specific build.

### Try This At Home!

- Print Your Own Layout: Create a C program to print image base, PEB, TEB, heap, and stack. Run across reboots to confirm terrain.
- Two-Process Comparison: Launch a probe and a child process. Compare ntdll and kernel32 bases within the same boot. They should be identical (Channel A proof).
- Low-Entropy Measurement: Focus empirical entropy calculation on the EXE base or small stack slots where sample size N is greater than possible values K. Use NSB estimator.
- Allocator Distribution: Loop VirtualAlloc calls and log distributions for bottom-up vs. top-down allocation. Test for non-uniformity (Channel B).
- Correlation Analysis: Calculate mutual information between regions (e.g., stack vs. heap) on your collected dataset.

## References and Related Work

### Papers

#### `[BINOSI-CCS]`
Binosi, Barzasi, Carminati, Zanero, Polino. "The Illusion of Randomness: An Empirical Analysis of Address Space Layout Randomization Implementations." ACM CCS, 2024. arXiv:2408.15107.

* **Summary:** Most direct prior work; its boot-fresh sampling method is this project's conceptual starting point.
* **Covers:** Sample-size justification, correlation-entropy definition and computation, the paths they found, their proof-of-concept structure, and the "Partial-VM" strategy.
* **Notes:** Light on Windows; assumes an attacker who collects samples with a debugger. We use an unprivileged external probe and add a Bayesian inference layer that outputs posteriors.

#### `[BRIZENDINE-IEEE]`
Brizendine, Rimal. "Predictable Paths: Novel ASLR Bypass Methods and Mitigations." IEEE Access, 2025.

* **Summary:** Establishes the deterministic Windows structures that leak module bases; needed to reason about PEB walks and what an external probe can learn.
* **Covers:** The deterministic offset chain from the PEB through the loaded-module lists, what makes their variants work, and their `[WINDBG]` validation.
* **Notes:** Assumes the attacker already has a memory-read primitive. We quantify what is learnable *before* that primitive exists.

#### `[JANG-DRK]`
Jang, Lee, Kim. "Breaking Kernel Address Space Layout Randomization with Intel TSX." ACM CCS, 2016. Also the BlackHat USA 2016 whitepaper.

* **Supporting.** Establishes a genre of microarchitectural side-channel ASLR attack. Good for threat-model precision. Out of scope for v1; possible stretch goal.

#### `[EVTYUSHKIN-BTB]`
Evtyushkin, Ponomarev, Abu-Ghazaleh. "Jump Over ASLR: Attacking Branch Predictors to Bypass ASLR." IEEE/ACM MICRO, 2016.

* **Supporting.** A second microarchitectural side-channel genre. Same role as `[JANG-DRK]`: threat-model precision, out of scope for v1.

#### `[SHACHAM-BRUTEFORCE]`
Shacham, Page, Pfaff, Goh, Modadugu, Boneh. "On the Effectiveness of Address-Space Randomization." ACM CCS, 2004.

* **Supporting.** The foundational ASLR paper. Defines the standard threat model and the brute-force baseline everyone compares against (expected attempts ~ 2^(n-1) for n residual bits).

#### `[MARCO-OFFSET2LIB]`
Marco-Gisbert, Ripoll. "On the Effectiveness of Full-ASLR on 64-bit Linux." 2014.

* **Supporting.** The earliest clear example of correlation collapsing ASLR on Linux.

#### `[GRAS-ANC]`
Gras, Razavi, Bosman, Bos, Giuffrida. "ASLR on the Line: Practical Cache Attacks on the MMU." NDSS, 2017.

* **Supporting.** Breaks ASLR from a heavily constrained environment like JavaScript. Sets up the "what can a weak attacker learn?" framing we adapt.

### Books

#### `[WININTERNALS]`
Windows Security Internals, 1st Edition. James Forshaw. 2024 - No Starch Press.

* **Summary:** Primary source on Windows memory management; the project relies on knowing exactly where entropy enters and where layout stays deterministic.
* **Covers:** Virtual address space layout, VAD trees, allocation policies, and the loader's handling of system DLL relocation and default heap randomization.

#### `[ARCH]`
Computer Architecture: From the Stone Age to the Quantum Age, Charles Fox. 2024 - No Starch Press.

* **Summary:** Background reference on computer architecture and the memory hierarchy.

### Web resources

* **`[CHAPPELL]`**: Geoff Chappell's notes on Windows internals at chappell.com.au. Reference specific articles by title. Practical reference for loader internals.
* **`[VERGILIUS]`**: The Vergilius Project. Used for looking up structure layouts across Windows builds.
* **`[MS-DOCS]`**: Microsoft Learn and Microsoft Docs.

### Tools

* **`[WINDBG]`**: WinDbg, including the Preview version.
* **`[PROCHACKER]`**: Process Hacker. Also known as System Informer.
* **`[Z3]`**: The Z3 SMT solver.
* **`[ORTOOLS]`**: Google OR-Tools. Covers CP-SAT and MILP.
* **`[GUROBI]`**: The Gurobi MILP solver. Commercial, with a free academic license.
* **`[NPEET]`**: Non-Parametric Entropy Estimation Toolbox for Kraskov-family estimators.
* **`[PGMPY]`**: The Python library for probabilistic graphical models.

### Further reading

* **Empirical Studies:** Mandiant ("Six Facts about ASLR"), Whitehouse (Vista analysis).
* **Theory:** Cover & Thomas ("Elements of Information Theory"), Paninski ("Estimation of Entropy").
* **Official Docs:** Larry Osterman blog (KnownDLLs).
