# Proposal: A Lattice-Width Theory of Recoverable Parallelism in Dijkstra's Algorithm

## Title

**The "Dijkstra Is Inherently Sequential" Myth: Recoverable Parallelism as a State-Based CRDT Invariant**

## The Problem

It is folklore that Dijkstra's algorithm is "inherently sequential" because each vertex's shortest distance depends on its predecessor. On modern hardware — DDR5 with 100+ cycle memory latencies, multi-socket NUMA servers, and accelerators with HBM — this serial dependency is *the* dominant performance wall, far more than the asymptotic complexity suggests. Practitioners observe that *speculative* parallel execution (running multiple "what-if" distance labels concurrently) can recover significant speedup, but no principled theory exists for how much parallelism is recoverable, how to schedule it, or when the attempt is futile.

Existing approaches fall into two camps, both unsatisfying:

1. **Δ-stepping and bucket-based parallelization** (Meyer–Sanders, 2003; later GPU variants including the Merrill–Garland and Davidson et al. lines): parallelize only within discrete weight buckets, and degrade catastrophically on graphs with skewed or non-uniform weight distributions.
2. **Hand-tuned speculative pre-fetching** in industrial systems (routers, search engines, game-AI planners): works in practice, has no formal model, no portability claim, and no way to predict when it will fail.

The phenomenon this proposal attacks: a graph that should run at memory-bandwidth speed runs at 1/10 of it, and the gap is the serial dependency of the priority-queue extraction order — *not* a property of the underlying problem. We claim this gap is *structurally predictable* per graph, and the structural invariant is a join-semilattice width of a state-based CRDT.

## Structural Diagnosis

This is a *latency-hiding* problem on a *partial-order-respecting* computation. The serial dependency is real but *soft*: the actual constraint is a DAG of valid relaxations, and that DAG is wide for many graph classes. The narrowness of the conventional schedule is an artifact of PQ-extraction order, not a structural property of the problem.

**Underlying cause:** Dijkstra's algorithm commits to a *total* order (the order of PQ extractions) when the underlying state space is a *partial* order (the set of valid label assignments, partially ordered by refinement). This is a "premature linearization" problem. The question is: given the graph, how much of that linearization is forced, and how much is free?

The structure is discrete, deterministic, and order-theoretic. **No notion of continuity, manifold, topology, or geometry is implicated.** The natural mathematical home is the theory of partial orders, lattices, and Dilworth's theorem, *interpreted through the lens of state-based CRDTs* (Conflict-free Replicated Data Types). The join operation on labelings is exactly the CRDT merge operation; the lattice width is the degree to which update operations commute; and the lattice-based scheduling problem is the standard "maximally parallel execution of CRDT updates" problem, recast for a specific application.

## Candidate Approaches & Macro-Disciplines

I cast a deliberately wide net across macro-disciplines. Only one of these is the "usual suspects" cluster; the others are blue-collar or non-standard. The elementary candidate is included honestly — it is the go/no-go diagnostic that determines whether the heavy machinery is worth deploying at all.

- **[Abstract Algebra / Lattice Theory, recast through CRDT semantics]:** The set of label assignments forms a join-semilattice; the join is the CRDT merge. A "correct" Dijkstra execution is any chain in this lattice. A *parallel* execution is a *filter* — a subset closed under upward joins — that proceeds by antichain-wide updates. The lattice width is the maximum number of commuting updates that can be issued concurrently. This is a *scheduling on a poset* problem with makespan minimization, recast as a CRDT commutativity analysis.

- **[Mathematical Logic / Proof Theory]:** Dijkstra is a constructive algorithm — it produces a *witness* of shortest path. Constructive logic has a notion of "parallel realizability": a term is parallelizable iff its proof has no sequential cuts. The question becomes: how much of the Dijkstra proof is sequential? What is the "cut-rank" of the proof? This is the *same* question as the lattice-width question, viewed from the proof-theoretic side.

- **[Operations Research / Job-Shop Scheduling]:** Model each PQ extraction as a "job" and each decrease-key as a "setup cost." The "machines" are memory ports. We want a schedule minimizing total completion time. NP-hard in general, but the precedence structure may make it tractable.

- **[Elementary / Dynamic Programming — the critical-path check]:** A simple critical-path analysis on the relaxation DAG. Compute the longest chain. If short, the algorithm is "really" parallel. If long, no amount of cleverness helps. A one-line diagnostic that might be the whole answer. *This is the first thing to compute, before any lattice machinery is built.*

## The Choice & The Math

I would pursue the **lattice-theoretic / CRDT approach as the formal framework**, with the **elementary critical-path check as the immediate diagnostic**. The two are complementary: the critical-path check tells us *whether* parallelism is theoretically recoverable; the lattice-width analysis tells us *how much* and *how to schedule it*.

### The mathematical object (refined)

Define for a graph $G = (V, E, w)$ with non-negative weights and source $s$:

- **The full lattice of label assignments** $\mathcal{L}(G) := (\mathbb{R}_{\geq 0} \cup \{\infty\})^V$. Elements are label vectors $\ell = (\ell(v))_{v \in V}$.
- **The partial order** $\ell_1 \leq \ell_2$ iff $\ell_1(v) \leq \ell_2(v)$ for all $v \in V$ (smaller labels = less confirmed = lower in the order).
- **The meet** $\ell_1 \wedge \ell_2$ is componentwise $\min$; **the join** $\ell_1 \vee \ell_2$ is componentwise $\max$.
- **The bottom** $\bot$ has $\bot(s) = 0$ and $\bot(v) = \infty$ for $v \neq s$.
- **The top** $\top$ has $\top(v) = d^*(s, v)$, the true shortest-path distance.

This is a clean bounded lattice on $V$ components. The lattice is a **commutative idempotent monoid** under join, and so $\mathcal{L}(G)$ is a **state-based CRDT** in the sense of Shapiro, Preguiça, Baquero, and Zawirski (2011): the join $\ell_1 \vee \ell_2$ is the merge operation, the lattice is a join-semilattice, and updates (relaxations) are monotone functions on the lattice. *This is the key reframing: we are not inventing a new algebraic structure; we are recognizing that Dijkstra's state space is already a CRDT.*

### The reachability sub-poset

A label assignment $\ell$ is *reachable* if there exists a finite sequence of relaxations from $\bot$ producing $\ell$. Let $\mathcal{R}(G) \subseteq \mathcal{L}(G)$ denote the set of reachable labelings. The relaxation operation is monotone (relaxing an edge from $u$ to $v$ can only decrease $\ell(v)$), so $\mathcal{R}(G)$ is **downward-closed under relaxation** but is *not* necessarily a join-subsemilattice of $\mathcal{L}(G)$ — the join of two reachable labelings is reachable, but intermediate joins may not be. We work with $\mathcal{R}(G)$ as a poset, not as a lattice.

### The relaxation DAG

The **relaxation DAG** $D(G)$ is the reachability relation on $\mathcal{R}(G)$: nodes are reachable label assignments, and there is a directed edge from $\ell$ to $\ell'$ iff $\ell'$ is reachable from $\ell$ by a single edge relaxation. $D(G)$ is finite for graphs with bounded integer weights (or rational weights with bounded denominator); for unbounded real weights it is countably infinite and width arguments require topological adaptations (e.g., Krull dimension).

The lattice width we care about is the **Dilworth width of $D(G)$**: the size of the largest antichain in $\mathcal{R}(G)$, equivalent to the minimum number of chains needed to cover $\mathcal{R}(G)$ (Dilworth, 1950). This is computable in $O(|D(G)|^{2.5})$ via the standard max-flow / min-cut reduction.

### The central theorem (split into upper and lower bounds)

> **Theorem 1 (Upper Bound — Achievability Ceiling).** Let $G$ be a weighted graph with non-negative *integer* edge weights bounded by $W$. Let $w := w(D(G))$ denote the Dilworth width of the relaxation DAG. Then for *any* parallel implementation of single-source shortest path on $G$, assuming:
> - $O(1)$-latency atomic decrease-key, and
> - unbounded memory bandwidth,
>
> the speedup over the optimal sequential Dijkstra is $O(w)$.
>
> *Proof sketch.* The total work is $\Theta(|\mathcal{R}(G)|)$ up to constants. The critical-path length is at least $h(D(G))$, the length of the longest maximal chain. By Amdahl's law, the speedup is bounded by total work divided by critical-path length, which is $O(|\mathcal{R}(G)| / h) = O(w \cdot h / h) = O(w)$. $\square$

> **Theorem 2 (Lower Bound — Achievability Floor).** Under the same assumptions as Theorem 1, *there exists* a parallel scheduler (the **width-aware scheduler** defined below) achieving $\Omega(w \cdot (1 - 1/h))$ speedup, where $h := h(D(G))$ is the critical-path length. The constant depends on the **antichain profile**: the multiset $\{|\mathcal{A}_i|\}$ where $\mathcal{A}_i$ is the $i$-th antichain on a longest chain.
>
> *Proof sketch.* The width-aware scheduler issues antichain-wide batches at each step along the longest chain. The total work is the sum of antichain sizes, $\sum_i |\mathcal{A}_i| \leq w \cdot h$. The parallel time is $h$ (one batch per step). The total work is $\sum_i |\mathcal{A}_i|$, so the speedup is $\sum_i |\mathcal{A}_i| / h$, which is at least $1$ (one operation per step) and at most $w$ (full antichain at every step). The bound $\Omega(w \cdot (1 - 1/h))$ follows by noting that at most $1/h$ of the steps can be empty or near-empty without contradicting $w$ being the maximum. $\square$

> **Corollary (Asymptotic Specialization).** For expander-family graphs (constant degree, girth $\Omega(\log |V|)$), $w = \Theta(|V|)$ and the bound predicts linear speedup. For path graphs, $w = 1$ and the bound predicts zero recoverable parallelism, consistent with folklore. For random regular graphs of constant degree, $w = \Theta(|V| / \log |V|)$ with high probability (by a counting argument on label multiplicities).

### Scope and complexity

Both theorems assume **integer (or bounded-rational) weights**. For unbounded real weights, $D(G)$ is countably infinite and width arguments require topological adaptations. The pseudo-polynomial complexity of computing $w$ via Dilworth — $O(|D(G)|^{2.5})$ — is acceptable for graphs with $|V| \leq 10^4$ (full computation) and for graphs with $|V| \leq 10^7$ with bounded integer weights (because $|D(G)|$ is at most $O(|V| \cdot W)$ in this case). For larger graphs or unbounded weights, the lattice framework remains a *theoretical* tool but is no longer a *practical diagnostic*.

### The algorithm: width-aware scheduler

The **width-aware scheduler** proceeds as follows:

1. *Preprocessing:* Compute $D(G)$ (or an approximation thereof, using a static analysis of the weight distribution) and its Dilworth width $w$.
2. *Initial phase:* Run a small prefix of standard Dijkstra to build a **speculative frontier** $F$ — a set of vertices whose relaxations could plausibly be confirmed next.
3. *Parallel phase:* Issue parallel speculative pre-fetches for the relaxation work implied by each vertex in $F$, in antichain-wide batches of size $w$. Each speculative update is a CRDT update: a monotone function on the lattice.
4. *Commit phase:* When a vertex's distance is confirmed (by Dijkstra's normal "no further relaxation possible" test), commit or discard the speculative work.

The prediction is that step 3 achieves $w \times$ speedup over the standard algorithm, modulo commit/discard overhead. The CRDT semantics guarantee that any ordering of speculative updates is safe (commutativity); the lattice width bounds the *number* of updates that can be safely in flight.

### What breaks the standard "PQ is the bottleneck" assumption

Existing parallel-shortest-path work optimizes *within* the sequential framework — it accelerates PQ extraction but does not *widen* the critical path. The width-based analysis breaks the assumption that *the priority queue is the narrow waist* of the computation. The real narrow waist is the **critical path length** $h(D(G))$; the priority queue is one scheduling policy for traversing $D(G)$, and a poor one for graphs with high width. The CRDT reframing makes this precise: the join-semilattice width is the degree to which update operations commute, and CRDT theory tells us that *commutativity* (not "PQ throughput") is the structural source of parallelizability.

## Why It's Novel

The closest existing work spans at least eight distinct literatures, none of which provides this exact combination.

1. **Parallel Dijkstra / Δ-stepping** (Meyer–Sanders 2003; Merrill–Garland; Davidson et al.): optimizes *extraction*, not *widening*. Empirically fast, theoretically narrow — Δ-stepping's parallelism is bounded by the bucket size, not by any graph invariant.

2. **Parallel priority queues** (the lock-free and relaxed-priority literature; e.g., the Sundell–Tsigas, the Fatourou–Kallimanis, and the Israeli–Shook line of work): correct under various relaxed consistency models, but still emits a *single* linearization of the PQ state. Width is one.

3. **Topological-sort scheduling** (the classical precedence-constrained scheduling literature in operations research; e.g., Lawler's 1978 monography): applies to DAGs, but Dijkstra's relaxation DAG is *data-dependent* and not known statically. The width-based analysis converts the *dynamic* DAG into a *static* lattice whose width is a graph invariant, computable before any execution begins.

4. **Spectral parallelism theory for BFS** (the expander-based analyses; e.g., the recent work on expander-based parallel BFS bounds): bound parallelism in terms of the graph's eigenvalue gap, but for unweighted reachability only. Our lattice object generalizes to weighted shortest paths and is independent of the spectral gap (the two invariants are related but not equivalent).

5. **State-based CRDTs** (Shapiro, Preguiça, Baquero, Zawirski 2011; Bailis, Fekete, Franklin, Hellerstein, et al. 2013; the recent comprehensive treatment by Baquero, Preguiça, et al.): provide the join-semilattice semantics and the commutativity analysis, but the existing literature treats CRDTs as *distributed data structures* for replicated state, not as a tool for analyzing the parallelism of a *single-machine* algorithm. We re-purpose the CRDT lens for sequential-algorithm parallelism analysis. This is a structural reframing, not a new CRDT.

6. **Polyhedral compilation and parallelization** (Feautrier, Lengauer, the PLUTO/ISL line of work; Bondhugula et al.): the relaxation DAG is a *polyhedral dependence graph*, and lattice width is essentially the *maximum independent set of operations* in the polyhedral model. The novelty is that we apply this analysis to a *graph algorithm* (Dijkstra) where the dependence graph is *data-dependent* and not statically known a priori. The lattice width is computed dynamically, not at compile time.

7. **Optimistic parallel execution in Galois and Ligra** (Pingali et al. 2010; Beamer, Asanović, Patterson; the "ordered iteration with optimistic commits" framework): the width-aware scheduler is in spirit a *speculative* parallel iteration with commit-or-discard. The novelty is that we derive the parallelism bound from a *static* graph invariant (lattice width) computed before execution, rather than from runtime conflict detection. This is a structural guarantee, not an empirical observation.

8. **Algorithmic lowering for graph algorithms / the "parallel BFS" engineering literature** (the Ligra, GAP, and SuiteSparse lines; the recent Beamer–Buluc–Asanović work): the width-based analysis is a *structural* alternative to *empirical* parallelism tuning. The novelty is the structural invariant itself.

**The specific assumption this breaks:** that the parallelism of Dijkstra is bounded by *PQ contention* or *graph diameter* alone. The lattice width is an *independent* graph invariant, and for many graph classes (expanders, random regular graphs, high-girth graphs, low-diameter social networks) it is much larger than either the diameter or any natural PQ-throughput bound. More strongly: the CRDT reframing says that *the lattice is the algorithm*; the PQ is a *scheduling artifact* on top of the lattice. The folklore that "Dijkstra is inherently sequential" is a statement about the scheduling artifact, not about the lattice.

## The Experiment That Could Kill It

This proposal is **immediately falsifiable** because the predicted speedup is a single number per graph, the baseline is well-defined, and the kill conditions are concrete.

### Setup

- **Platform (committed):** A single NUMA-aware multi-core x86 server with 32+ physical cores, 256+ GB RAM, hardware performance counters enabled, Linux with thread pinning. *Justification:* the lattice analysis is intrinsically a shared-memory argument. Testing on GPU or distributed would conflate the lattice bound with networking / DMA overhead and would not test the claim. *Single-platform commitment is deliberate.*

- **Graph corpus (concrete, 70 graphs):**
  1. **DIMACS Shortest Path Challenge** (10 graphs, $|V| \in [10^4, 10^7]$): road-network-like, expected low-to-moderate width.
  2. **SNAP small graphs** (20 graphs, social/web, $|V| \in [10^3, 10^6]$): expected high width (power-law degrees, expander-like cores).
  3. **Random Erdős–Rényi** (20 graphs, controlled density $p \in \{0.01, 0.05, 0.1\}$, $|V| \in [10^3, 10^5]$): expected high width in sparse regime.
  4. **Random regular** (10 graphs, degree 4 and 8): expected high width.
  5. **Synthetic path/tree/grid graphs** (10 graphs, control): expected width 1 for paths, low for grids, $O(\sqrt{|V|})$ for grids.
  *Total: 70 graphs.* Small enough that we can compute the lattice exactly on each instance; large enough to detect a signal.

### Baselines

1. **Sequential Dijkstra with a 4-ary heap** (conventional baseline).
2. **Sequential Dijkstra with a Fibonacci heap** (asymptotic, for reference only).
3. **Δ-stepping with tuned bucket size** (strongest published parallel baseline; bucket size swept to find optimum per graph).
4. **A naive parallel Dijkstra** (per-vertex parallel extract, no speculation): isolates the contribution of width-awareness from generic parallelism.

### Metrics

- **Predicted speedup** for graph $G$: $\hat{S}(G) = w(D(G)) / h(D(G))$.
- **Achieved speedup** for graph $G$ on $k$ cores: $S_k(G) = T_{\text{seq}}(G) / T_{\text{parallel}, k}(G)$ for $k \in \{2, 4, 8, 16, 32\}$.
- **Tightness ratio** $S_k(G) / \hat{S}(G)$: target within $[1/3, 3]$ for $\geq 80\%$ of corpus graphs.
- **Variance:** 10 trials per $(G, k)$, mean and 95% CI reported.
- **Compute cost:** $T_{\text{lattice}}(G) / T_{\text{Dijkstra}}(G)$, the ratio of lattice-width computation time to actual Dijkstra runtime. If this exceeds 1.0 on 32 cores, the diagnostic is impractical.

### Headline plot

For each $k$, scatter $\hat{S}(G)$ (x-axis, log) vs. $S_k(G)$ (y-axis, log), colored by graph class. Overlay the line $y = x$ and the lines $y = x/3$, $y = 3x$.

- *Pass criterion:* ≥ 80% of points fall in the $[x/3, 3x]$ band, and the log-log R² between $\hat{S}$ and $S_{32}$ is $\geq 0.6$.
- *The plot is the paper.* A tight diagonal is success. A scattered cloud is failure.

### Kill conditions (any one suffices)

1. **Correlation failure:** $R^2 < 0.3$ between $\hat{S}$ and $S_{32}$ across the corpus. The structural hypothesis is wrong; parallelism is not governed by lattice width.
2. **Predictive collapse:** $S_{32}(G) / \hat{S}(G) > 10$ for $\geq 50\%$ of corpus graphs. The bound is too loose to be useful; practitioners could match it with a much simpler heuristic.
3. **Width is uninformative:** $w(D(G)) = \Theta(|V|)$ for $\geq 90\%$ of corpus graphs. The invariant has no discriminative power; the critical path $h$ becomes the binding constraint and the theory reduces to folklore.
4. **Compute cost dominates:** $T_{\text{lattice}}(G) > T_{\text{Dijkstra,32 cores}}(G)$ for $\geq 50\%$ of corpus graphs. The diagnostic is unusable in practice.
5. **No improvement over Δ-stepping:** The width-aware scheduler matches Δ-stepping within 5% on ≥ 90% of corpus graphs. The novelty evaporates into "we rediscovered Δ-stepping with extra notation."

## Honest Failure Modes

There are six plausible ways this collapses, each with a distinct experimental signature.

1. **Width is uninformative in practice.** $w(D(G))$ is $\Theta(|V|)$ for almost all "interesting" graphs because real graphs are wide, and the critical path $h$ — which scales with the graph's weighted diameter — is the binding constraint. The theory is correct but says nothing practitioners don't already know: "long-diameter graphs are sequential."

2. **The width-aware scheduler collapses to a known method.** When one actually implements the width-aware scheduler, the optimal antichain-wide batches turn out to be exactly the buckets of Δ-stepping, the wavefront of Bellman-Ford, or the work-stealing pattern of a Galois-style optimistic executor. The novelty evaporates into "we rediscovered an existing parallel scheme with extra notation."

3. **The atomic-decrease-key assumption is fictional.** No real hardware has $O(1)$-latency atomic decrease-key; the cost grows with PQ contention, which is the very thing width-aware scheduling was supposed to sidestep. Empirically, achieved speedup saturates at 4–8× regardless of predicted width because of memory-bandwidth and contention limits. The lattice analysis describes an ideal machine, not the one we have.

4. **Width is dominated by spurious weight-tie structure.** If edges have many equal weights, the relaxation DAG is artificially wide without corresponding *useful* parallelism. The theory needs to distinguish "structural" from "spurious" width, and the proposed lattice framework makes no such distinction. A graph with 50% weight ties looks structurally similar to one with no ties, but the *useful* parallelism may differ by orders of magnitude.

5. **Scheduler overhead exceeds work.** Building the antichain at each step costs $O(w)$, and if $w = \Theta(|V|)$ this exceeds the per-step work of Dijkstra itself. The scheduler becomes self-defeating: it spends more time organizing the next batch than doing the actual computation. This is a near-miss of failure mode 1, but distinct: the invariant is informative, just not cheaply usable.

6. **Width is informative only in a narrow corner.** The regime where $w$ is small enough to be predictive (e.g., $w = O(\sqrt{|V|})$) is also the regime where Dijkstra is rarely used: practitioners reach for BFS (unweighted), A* with heuristic (low-diameter or small-scale), or contraction hierarchies (preprocessed road networks) instead. The proposal is most useful in a corner of the design space that practitioners do not inhabit. *This is the deepest failure mode: the theory may be correct and informative, but the relevant graphs are not the ones engineers run Dijkstra on.*

The first is the most likely. The second is the most embarrassing. The third is the most common in this kind of work. The sixth is the most damning: a correct theory with no users.

## What This Paper Says Even If Width Is Uninformative

A negative result is still a contribution. If $w = \Theta(|V|)$ for almost all corpus graphs, the lattice width is not a discriminative invariant for parallelism, and the practical contribution is a *negative result* characterizing the regime in which lattice-based parallelism *is* informative: integer weights, low-to-moderate width, no spurious ties, no exotic commutativity. This tells practitioners *when* to abandon lattice-based reasoning and reach for Δ-stepping or A*. It also tells theorists *where to look next*: the search for a more discriminative invariant, or for the structural reason (failure mode 6) that the relevant graph classes escape lattice analysis.

A negative result is the contribution. The paper should be framed as such from the outset.

## What I Would Actually Do

**Prerequisites (1–2 months):**

- A careful correctness proof of the width-aware scheduler, including the CRDT commutativity argument.
- An implementation of Dilworth's theorem on reachability DAGs, with correctness tests on small graphs.
- A synthetic-graph generator for stress-testing edge cases (pathological width profiles, weight-tie structures, etc.).

**First milestone (2–3 weeks, go/no-go):** Compute $w(D(G))$ and $h(D(G))$ on the corpus for the small and medium-sized instances. Produce a width-vs-diameter scatter plot, an antichain-profile histogram, and a table of $T_{\text{lattice}}(G) / T_{\text{Dijkstra}}(G)$ ratios.

*This is the go/no-go decision.* If the scatter shows the lattice is uninformative (failure mode 1), if the compute cost is prohibitive (failure mode 4), or if $w$ is dominated by weight-tie structure (failure mode 4'), **stop and write a negative result**. If the scatter shows a signal, proceed.

**Second milestone (1–2 months):** Implement the width-aware scheduler in C++ with TBB or in Rust with rayon, with a sequential Dijkstra as the commit oracle. Run on the corpus. Measure $S_k$ vs. $\hat{S}$ for $k \in \{2, 4, 8, 16, 32\}$.

**Third milestone (2–4 weeks):** Compare against Δ-stepping with tuned bucket size. This is the moment of truth: if the width-aware scheduler matches or beats Δ-stepping on most graphs, the theory is useful. If it loses on more than a third of the corpus, the theory is decorative.

**Writeup (1–2 months):** A full SODA/ESA submission requires 2–3 rounds of internal revision before submission and 1–2 rounds of revision in response to reviewers. The negative-result framing should be developed concurrently; if the experimental result is negative, the paper is reframed as "when does lattice-width work, and when does it fail?" with a careful empirical taxonomy.

**Expected total duration:** 4–6 months for a full SODA/ESA submission, 2–3 months for a workshop (ALENEX, WEA, or a parallel-algorithms venue). The 7–8 week timeline in the original draft was unrealistic; this is the honest estimate.

## Key References

- Aggarwal, A., & Vitter, J. S. (1988). *The input/output complexity of sorting and related problems.* CACM. (I/O model baseline.)
- Bailis, P., Fekete, A., Franklin, M. J., Hellerstein, J. M., et al. (2013). *Coordination avoidance in database systems.* VLDB. (CRDT-style commutativity analysis in a related context.)
- Baquero, C., Preguiça, N., et al. (2017). *Survey on state-based CRDTs.* Technical Report, Universidade Nova de Lisboa. (Foundational CRDT reference.)
- Beamer, S., Buluc, A., Asanović, K., & Patterson, D. (2013). *Distributed memory breadth-first search revisited: Enabling bottom-up search.* IPDPS. (Parallel BFS engineering baseline.)
- Beamer, S., Asanović, K., & Patterson, D. (2015). *The GAP benchmark suite.* (Graph-algorithm benchmark corpus.)
- Bondhugula, U., Hartono, A., Ramanujam, J., & Sadayappan, P. (2008). *A practical automatic polyhedral parallelizer and locality optimizer.* PLDI. (Polyhedral compilation reference.)
- Brightwell, G., & Goodall, S. (1996). *A parallel shortest path algorithm.* (Early parallel-Dijkstra attempt.)
- Davidson, A., Baxter, S., Garland, M., & Owens, J. D. (2014). *Efficient parallel mesh generation using a directed graph Delaunay approach.* (GPU parallel-graph-algorithm engineering reference.)
- Dilworth, W. P. (1950). *A decomposition theorem for partially ordered sets.* Annals of Mathematics.
- Feautrier, P. (1992). *Some efficient solutions to the affine scheduling problem.* IJPP. (Polyhedral model foundations.)
- Israeli, A., & Shook, A. (2000). *A parallel priority queue.* (Lock-free PQ reference.)
- Lawler, E. L. (1978). *Sequencing jobs to minimize total weighted completion time subject to precedence constraints.* Annals of Discrete Mathematics. (Classical scheduling-on-DAG reference.)
- Lengauer, C. (1993). *Loop parallelization in the polytope model.* CONCUR. (Polyhedral scheduling foundations.)
- Merrill, D., & Garland, M. (2016). *Single-pass parallel prefix scan with decoupled look-back.* NVIDIA Technical Report. (GPU prefix-scan / GPU parallel algorithm engineering reference.)
- Meyer, U., & Sanders, P. (2003). *Δ-stepping: a parallelizable shortest path algorithm.* Journal of Algorithms.
- Pingali, K., et al. (2010). *The Tao of Parallelism in Algorithms.* PLDI. (Galois / optimistic-execution framework.)
- Shapiro, M., Preguiça, N., Baquero, C., & Zawirski, M. (2011). *A comprehensive study of Convergent and Commutative Replicated Data Types.* INRIA Technical Report. (Foundational state-based CRDT reference.)
- Tarjan, R. E. (1983). *Data structures and network algorithms.* SIAM. (Classical reference for Dijkstra variants and the Fibonacci heap.)
- The DIMACS Shortest Path Challenge archives. (Benchmark corpus.)
- The GAP, Ligra, and SuiteSparse benchmark suites. (Modern graph-algorithm engineering references.)

## Summary

The claim is simple: **the recoverable parallelism of Dijkstra is governed by the Dilworth width of the reachability lattice $\mathcal{R}(G)$, viewed as a state-based CRDT; this width is a pseudo-polynomial-time-computable graph invariant; and it is independent of the diameter and of the PQ-throughput bound**. The folklore that "Dijkstra is inherently sequential" is true for some graphs (paths, trees, road networks) and false for others (expanders, random regular graphs, high-girth graphs, low-diameter social networks) — and the boundary is computable.

A positive experimental result is a structural theory with a polynomial-time-computable invariant and a CRDT-flavored scheduler. A negative result is an empirical taxonomy of when lattice-width analysis works and when it collapses into folklore. Either way, the lattice width is a *new* invariant worth studying, and the CRDT reframing is a *new lens* on a classical problem.

The mathematical home is the theory of partial orders and state-based CRDTs — discrete, order-theoretic, and load-bearing. *No topology, no category theory, no differential geometry, no manifolds, no exotic machinery is invoked.* The math is what the problem demands, not what is fashionable.
