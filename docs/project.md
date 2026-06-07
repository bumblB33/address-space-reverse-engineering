# Project Plan, Targets, and Methods

## Phases

### Sampling

Build the probe and collect the dataset.

* **Deliverables:** A probe binary that logs all observables to a structured format like JSONL. An automated harness to boot the VM, run the probe, log output, and reset. A first-pass dataset for validation and a full-pass dataset for the final analysis.
* **Exit criteria:** Sample collection runs completely unattended. Empirical entropy estimates for system DLLs match the published literature. Sample integrity is verified against duplicate boots and truncated logs.

### Inference

Implement the inference method and get measurements.

* **Deliverables:** The implemented inference tool, a per-region conditional entropy table with bootstrap confidence intervals, documentation of the strongest correlation paths, and a held-out evaluation against ground-truth layouts.
* **Exit criteria:** Posterior entropy meaningfully drops below prior entropy for at least one region. The methodology is reproducible.

### Optimization

Implement the constraint and attempt-ordering layers.

* **Deliverables:** Constraint propagation to produce feasible layouts and an optimal attempt ordering for at least one region, compared against the baselines.
* **Exit criteria:** The ordering strategy yields a measurable reduction in expected attempts. If a simple greedy approach turns out to be optimal, cite a theorem proving the cost-model conditions hold.

### Writeup

* **Deliverables:** Write-up on threat model, methodology, results, mitigations, and related work. All code works on a fresh VM clone.
* **Exit criteria:** A reader can replicate results within the time budget.


## Metrics and targets

### Sampling

* **First-pass sample size:** Enough to confirm the harness works and the per-region distributions look plausible. Initial estimate is a few thousand boots, pending dry runs.
* **Full-pass sample size:** Enough boots to estimate conditional entropy with bootstrap confidence intervals tight enough to pick out meaningful effects on the highest-entropy region. The inference phase dictates actual CIs.
* **Matching threshold:** Entropy estimates fall within the bootstrap CI of published estimators. Anything outside that CI is flagged for follow-up.

### Inference

* **Number of distinct target processes:** An isolated test target plus at least two third-party applications, to evaluate whether the result generalizes.
* **Held-out evaluation split:** 80/20 train/test. 80% for fitting, 20% held out for evaluation.
* **Reportable-result threshold:** At least one region where posterior entropy sits below prior entropy, with the bootstrap CIs of the two estimators not overlapping at 95% confidence.

### Optimization

* **Baselines:** uniform random ordering, unweighted descending-probability ordering, closed-form optimums.
* **Operational reduction threshold:** At least one region where the optimized attempt ordering yields an expected-attempts reduction with CIs that do not overlap the baselines.
* **Cost model:** TBD during optimization.

## Method survey

Decisions wait until some first-pass data is available.

### Subproblem one: probabilistic inference of the target layout from probe observations

**Inputs:** Samples of probe and target layouts collected across fresh boots. The target layout is ground truth used only for evaluation; for inference we have only the probe layout.
**Output:** A posterior distribution over a structured but discrete space.
**Side outputs:** Per-region entropy estimates with confidence intervals, mutual information metrics, identification of the strongest correlations.

Candidate approaches:

* **Direct empirical estimation:** Treat each observable pair as a joint distribution and estimate by counting. Great when marginal cardinalities are small; fails for high-entropy regions where samples are sparse.
* **Bayesian network:** Model the joint distribution over memory objects to represent kernel allocation dependencies. Generalizes well but structure learning is difficult.
* **Information-theoretic estimators:** Non-parametric mutual-information estimators. Avoids assuming the joint's shape but yields numbers, not a posterior we can sample from.
* **Discriminative machine learning:** Train a model to predict the target region from probe observables. Scales well but is a black box and prone to overfitting.
* **Hybrid explicit allocator model:** Write a model of Windows allocator behavior and run an exact Bayesian update on its parameters. Highly defensible but significant engineering effort.

Evaluation criteria: defensibility to technical reviewers, sample efficiency (prefer methods that share statistical strength across regions), interpretability, and a tractable posterior the downstream optimizer can sample from.

Recommended starting point: direct empirical estimation for low-entropy regions, then a Bayesian network or hybrid allocator model where structure matters. Independently validate with information-theoretic estimators; keep discriminative ML strictly as a baseline.

### Subproblem two: optimization given the posterior

Constraint propagation to narrow the feasible layouts, plus attempt ordering to minimize expected attempts to a first hit.

Candidate approaches:

* **Mixed-Integer Linear Programming:** Good for linear integer optimization, awkward for bit-level facts.
* **Constraint programming:** Good for integer constraints carrying logical structure.
* **Satisfiability Modulo Theories (SMT):** Excellent for mixing bitvectors, modular arithmetic, and integer constraints. The natural choice for bit-level address constraints.
* **Optimal search theory:** Sometimes sorting by descending posterior probability is provably optimal under the cost model, and no solver is needed.

Evaluation criteria: fit to problem structure (SMT fits bit-level constraints well), tractability at realistic sizes (verify with a quick spike before committing), and reproducibility (prefer open-source solvers).

Recommended starting point: for constraint propagation, start with SMT and bitvector theory. For attempt ordering, first check whether descending-probability ordering is provably optimal; if not, fall back to constraint programming or MILP.

### Pre-implementation prototypes

A short prototype per subproblem before committing to a stack: build the inference path with empirical estimation on one low-entropy region to check whether the confidence intervals are trustworthy; build SMT-based constraint propagation for one realistic correlation path to check speed; write out the optimal-search analysis for a simple cost model to see whether greedy ordering is optimal.
