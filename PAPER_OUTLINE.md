# CoProver: Co-Training Proof Generation and Search Heuristics for Automated Theorem Proving

---

## Abstract (~250 words)

Neural theorem provers have achieved remarkable progress on formal mathematics benchmarks, with systems like DeepSeek-Prover-V2 and Goedel-Prover-V2 reaching near-saturation on MiniF2F. However, all existing systems share a fundamental asymmetry: they invest heavily in training the **tactic generator** (the LLM that proposes proof steps) while relying on **fixed, hand-designed search strategies** (beam search, best-first search, or MCTS with static heuristics) to navigate the proof space. We argue that the search strategy — which determines *which proof states to explore* — is at least as important as the quality of individual tactics. We introduce **CoProver**, a framework that co-trains two models: (1) a tactic generator that proposes proof steps, and (2) a **proof state value model** that learns to predict which intermediate states are most likely to lead to completed proofs. The value model guides a learned best-first search, focusing computation on promising proof branches. Critically, both models are trained iteratively: better search discovers more proofs, yielding richer training data, which improves both the generator and the value model. On PutnamBench, CoProver solves [X] problems at pass@32 compared to [Y] for the best baseline, while expanding [Z]% fewer search nodes. On SorryDB, a real-world benchmark of incomplete Lean proofs, CoProver achieves [X]% compared to [Y]% for [baseline]. Ablations confirm that co-training outperforms training either model independently, and that the learned value function provides consistent gains across search budgets. We release all code, trained models, and evaluation infrastructure.

---

## 1. Introduction (1.5 pages)

### Opening hook (2 paragraphs)
- The success story: neural theorem proving has gone from GPT-f's first machine-generated proofs (2020) to AlphaProof's IMO silver medal (2025) to Goedel-Prover-V2's 90.4% on MiniF2F (2025). The generator-verifier paradigm works: an LLM proposes proof tactics, a formal kernel (Lean4) verifies them, and RL/expert iteration closes the loop.
- The hidden assumption: every one of these systems trains better and better generators while keeping the search strategy fixed. DeepSeek-Prover-V2 uses fixed tree search. BFS-Prover uses fixed best-first search. Kimina-Prover uses fixed beam search. This is like training a better chess move proposer while always using depth-2 minimax — you're leaving enormous performance on the table by not also learning *how to search*.

### The gap (1 paragraph)
- In game-playing AI, the breakthrough was AlphaZero: co-training a policy network (move generator) AND a value network (position evaluator). The value network made search exponentially more efficient — instead of evaluating millions of positions, you evaluate thousands, guided by learned heuristics. Theorem proving has adopted the policy network (tactic generators) but not the value network. We fill this gap.

### Our contribution (1 paragraph, numbered list)
1. **Learned proof state value model** — a lightweight model that scores intermediate proof states by their likelihood of leading to a completed proof, used to guide best-first search.
2. **Co-training framework** — an iterative pipeline where the tactic generator and value model improve together: better search → more proofs → richer training data → better models.
3. **Search efficiency gains** — CoProver achieves comparable or better performance to baselines while expanding significantly fewer search nodes, demonstrating that learned search heuristics reduce wasted computation.
4. **State-of-the-art on [benchmark]** — [X] on PutnamBench / SorryDB, improving over [baselines].

### Framing paragraph
- We evaluate on PutnamBench (hard competition math) and SorryDB (real-world Lean tasks), deliberately avoiding the near-saturated MiniF2F. We show that co-training provides consistent gains across benchmarks, search budgets, and base models. We release code, models, and evaluation infrastructure to support future work on learned search for theorem proving.

---

## 2. Related Work (1 page)

### 2.1 Neural Theorem Proving
- **Tactic-level generation:** GPT-f (2020), LeanDojo/ReProver (2023), DeepSeek-Prover V1.5/V2 (2024-2025), Goedel-Prover V1/V2 (2024-2025), Kimina-Prover (2025).
- **Whole-proof generation:** Baldur (2023), Lean-STaR (2025), subgoal decomposition in DeepSeek-V2.
- **Agentic approaches:** Minimal Agent for ATP (2026), COPRA (2024), multi-agent proving in AutoVerus (2025).
- **Key point:** All train the generator. Search is always fixed (beam, BFS, greedy).

### 2.2 Search in Theorem Proving
- **Best-first search:** ReProver, BFS-Prover — expand the node with highest *cumulative log-probability* of the tactics on its path. This is a proxy for value, not a learned value.
- **MCTS:** AlphaProof (reportedly). Uses random rollouts or fixed heuristics for value estimation.
- **Key point:** No published system learns the value function for proof search and co-trains it with the generator.

### 2.3 Co-Training and Adversarial Training
- **GAR (2025/2026):** Co-trains problem *composer* and solver — adversarial curriculum. Different axis: they make harder problems, we make smarter search.
- **Expert iteration:** BFS-Prover, AlphaZero — solve problems, train on solutions, repeat. We extend this: also update the *value model* each round.
- **RL co-training in reasoning:** PRIME, RL Tango — co-train generator and verifier. Our value model is not a verifier (Lean is the verifier) — it's a *search heuristic*.

### 2.4 Learned Value Functions in Search
- **AlphaGo/AlphaZero (2016-2018):** Policy + value network co-trained via self-play. Direct inspiration.
- **MuZero (2020):** Learned dynamics model + value function for planning.
- **Key point:** The policy+value paradigm revolutionized game-playing AI. We bring it to theorem proving.

---

## 3. Method (3 pages)

### 3.1 Background: Tactic-Level Theorem Proving
- Formal setup: a theorem in Lean4 defines an initial proof state (the goal). A tactic transforms one proof state into zero or more new proof states. A proof is complete when all goals are discharged (QED). The Lean kernel verifies each step.
- Proof search as a tree: root = initial goal; edges = tactics; nodes = proof states; leaves = QED or failure.
- Standard approach: generator LLM produces candidate tactics, search strategy determines expansion order.

### 3.2 Proof State Value Model
- **Input:** A proof state $s$ (text representation of remaining goals, hypotheses, and context).
- **Output:** $V(s) \in [0, 1]$ — estimated probability that state $s$ lies on a path to a completed proof.
- **Architecture options:**
  - (a) MLP head on frozen LLM embeddings of the proof state
  - (b) Small transformer encoder (125M) with scalar value head
  - (c) Value head on the generator LLM itself (shared backbone)
- **Training signal:** From completed proof trajectories:
  - States on successful proof paths → positive examples (label based on proximity to QED)
  - States on failed/abandoned paths → negative examples (label 0)
  - Label scheme: $y = \gamma^{d}$ where $d$ = remaining steps to QED, $\gamma$ = discount factor

### 3.3 Learned Best-First Search
```
Algorithm 1: Learned Best-First Search
────────────────────────────────────────
Input: initial state s₀, generator G, value model V, budget N
Output: proof or FAIL

1: Q ← priority queue with (V(s₀), s₀)
2: for i = 1 to N do
3:   s ← Q.pop()                           // highest value state
4:   tactics ← G.generate(s, k=K)          // K candidate tactics
5:   for τ in tactics do
6:     s' ← lean.apply(s, τ)               // apply tactic, get new state
7:     if s' = QED then return SUCCESS
8:     if s' ≠ ERROR then
9:       Q.push(V(s'), s')                  // score and enqueue
10:  end for
11: end for
12: return FAIL
```
- **Comparison to standard BFS:** Standard BFS uses cumulative log-probability $\sum \log P_G(\tau_i | s_i)$ as priority. We use $V(s')$ — a direct estimate of "how close is this state to a proof?" This is strictly more informative because it accounts for the global proof landscape, not just local tactic probability.

### 3.4 Co-Training Loop
```
Algorithm 2: Co-Training
────────────────────────────────────
Input: theorem set T, base generator G₀, initial value model V₀
Output: trained Gₙ, Vₙ

1: for round r = 1 to R do
2:   // SEARCH: use current models to attempt proofs
3:   trajectories ← SearchAll(T, Gᵣ₋₁, Vᵣ₋₁)
4:
5:   // UPDATE GENERATOR: fine-tune on successful proofs
6:   successful ← {(s, τ) | (s, τ) ∈ trajectories, leads to QED}
7:   Gᵣ ← FineTune(Gᵣ₋₁, successful)      // SFT or GRPO
8:
9:   // UPDATE VALUE MODEL: train on ALL trajectories
10:  states⁺ ← {s | s on successful path}     // positive
11:  states⁻ ← {s | s on failed path}          // negative
12:  Vᵣ ← Train(Vᵣ₋₁, states⁺, states⁻)
13:
14:  // KEY: value model adapts to new generator's behavior
15: end for
```
- **Why co-training matters:** A value model trained on generator G₀'s trajectories gives poor predictions for generator G₃'s states — the distribution shifts as the generator improves. Co-training keeps the value model calibrated.
- **Curriculum:** Early rounds use easy theorems (≤5 steps). Later rounds add harder theorems. The value model learns coarse signals first, then refines.

### 3.5 Generator Training Variants
- **SFT Expert Iteration:** Fine-tune on successful proof trajectories (simplest).
- **GRPO:** Group Relative Policy Optimization with binary reward (proof found / not found).
- **GRPO + Value-Shaped Reward:** Use value model predictions as intermediate rewards — dense signal instead of sparse proof completion.

---

## 4. Experimental Setup (1 page)

### 4.1 Benchmarks
- **PutnamBench** (658 problems): Hard competition math. SOTA: 86/658 (Goedel-V2). Significant headroom.
- **SorryDB** (1000+ tasks): Real-world incomplete Lean proofs from GitHub. New (March 2026). Tests practical utility.
- **MiniF2F-test** (244 problems): Olympiad math. Report for comparability but near-saturated (90.4%).
- **ProofNet-test** (186 problems): Undergraduate math. Secondary benchmark.

### 4.2 Baselines
1. **ReProver** — retrieval-augmented generation, fixed best-first search (LeanDojo)
2. **BFS-Prover** — expert iteration, fixed BFS (DeepSeek)
3. **DeepSeek-Prover-V2-7B** — GRPO-trained generator, fixed tree search
4. **Goedel-Prover-V2-7B** — SFT + self-correction, fixed best-of-N
5. **CoProver (generator only)** — our generator with fixed BFS (no value model)
6. **CoProver (value only)** — ReProver generator with our learned search (no generator training)

### 4.3 Metrics
- **Pass@K** (K = 1, 8, 32): fraction of theorems proved within K attempts
- **Search efficiency**: average nodes expanded per proof (lower = more efficient)
- **Solve rate vs. compute**: performance as a function of search budget (nodes expanded)

### 4.4 Implementation Details
- Base generator: [DeepSeek-Prover-V2-7B / Goedel-Prover-V2-7B]
- Value model: [architecture choice from ablations]
- Fine-tuning: LoRA (rank 16), learning rate [X], [X] epochs per round
- Co-training: [R] rounds, [X] theorems per round
- Search budget: 512 node expansions per theorem
- Infrastructure: LeanDojo for Lean interaction, [X] GPUs, [X] hours total

---

## 5. Results (2 pages)

### 5.1 Main Results

**Table 1: Main results on PutnamBench and SorryDB**

| Method | PutnamBench (solved/658) | SorryDB (pass@32) | MiniF2F-test (pass@32) |
|--------|-------------------------|-------------------|----------------------|
| ReProver | [X] | [X] | [X] |
| BFS-Prover | [X] | [X] | [X] |
| DeepSeek-Prover-V2-7B | [X] | [X] | [X] |
| Goedel-Prover-V2-7B | [X] | [X] | [X] |
| CoProver (gen only) | [X] | [X] | [X] |
| CoProver (val only) | [X] | [X] | [X] |
| **CoProver (full)** | **[X]** | **[X]** | **[X]** |

Key findings:
- CoProver (full) outperforms all baselines on [benchmark] by [X]%
- The value model alone (val only) improves over fixed search by [X]%
- The generator alone (gen only) improves over baselines by [X]%
- Co-training provides [X]% additional gain over either component independently

### 5.2 Search Efficiency

**Figure 1: Solve rate vs. search budget (nodes expanded)**

[Plot: X-axis = nodes expanded (64, 128, 256, 512, 1024), Y-axis = pass rate. Lines for CoProver, BFS-Prover, ReProver.]

- **Key result:** CoProver at 128 nodes matches BFS-Prover at 512 nodes — **4x more search-efficient**.
- The value model's benefit grows with harder problems (where search matters more).

### 5.3 Co-Training Dynamics

**Figure 2: Performance per co-training round**

[Plot: X-axis = round (1-10), Y-axis = theorems solved. Two lines: "co-trained" vs. "independently trained".]

- Both generator and value model improve each round
- Co-training outperforms independent training starting from round 2
- Gap widens with more rounds — the models compound each other's improvements
- Diminishing returns after round [X]

### 5.4 Ablation Studies

**Table 2: Ablations on MiniF2F-test**

| Ablation | pass@32 | Δ vs. full |
|----------|---------|------------|
| Full CoProver | [X] | — |
| No co-training (independent) | [X] | -[X]% |
| No value model (fixed BFS) | [X] | -[X]% |
| Binary labels (0/1) vs. soft | [X] | -[X]% |
| MLP value head vs. transformer | [X] | -[X]% |
| SFT vs. GRPO | [X] | ±[X]% |
| GRPO + value-shaped reward | [X] | +[X]% |

---

## 6. Analysis (1 page)

### 6.1 What Does the Value Model Learn?
- Proof states with fewer remaining goals → higher value (expected)
- Proof states after `simp`, `ring`, `norm_num` → higher value (these often close goals)
- Proof states with very long/complex goal terms → lower value (harder to close)
- [Surprise finding: TBD from experiments]

### 6.2 Qualitative Examples
- **Example 1:** A PutnamBench problem where the value model correctly identifies a key intermediate lemma state that baselines skip over
- **Example 2:** A SorryDB task where learned search finds a 12-step proof while BFS finds a 28-step proof (more efficient path)
- **Example 3:** A failure case — value model is overconfident on a dead-end state

### 6.3 When Does the Value Model Help Most?
- Larger search spaces (harder problems) → larger gains
- Longer proofs → larger gains (more branching decisions)
- Problems with "tricky" intermediate states (require non-obvious lemmas) → largest gains
- Easy problems (provable in 1-3 steps) → minimal difference

---

## 7. Discussion & Future Work (0.5 pages)

### Limitations
- Value model adds inference overhead (though offset by fewer nodes expanded)
- Co-training requires multiple rounds (more total compute than single-round training)
- Value model quality depends on having enough successful trajectories — cold-start problem on very hard theorems
- Currently tactic-level only — doesn't handle whole-proof generation or subgoal decomposition (orthogonal, could be combined)

### Future Work
- **Self-play for theorem proving:** Value model enables full AlphaZero-style MCTS with learned value estimates and policy improvement via self-play proof search.
- **Learned curriculum:** Use value model to identify "frontier" theorems — just barely too hard for current system — for optimal training.
- **Transfer to hardware FV:** Apply the learned search paradigm to hardware formal verification, where search efficiency is even more critical (model checking is orders of magnitude slower than Lean).
- **Scaling:** Co-train with larger generators (32B, 70B) and study scaling laws of value model accuracy vs. size.

---

## 8. Conclusion (0.5 pages)

We presented CoProver, the first system to co-train a tactic generator and a proof state value model for Lean4 theorem proving. By learning *what to search* alongside *how to prove*, CoProver achieves [state-of-the-art / competitive] results on PutnamBench and SorryDB while being [X]x more search-efficient than fixed-strategy baselines. Our results suggest that the proof search strategy is an underexplored axis of improvement in neural theorem proving, and that the AlphaZero paradigm — jointly learning a policy and value function — transfers effectively from game-playing to mathematical reasoning. We release all code and models to support future work.

---

## Appendix

### A. Hyperparameters
[Full table of all hyperparameters for reproducibility]

### B. Additional Results
[Full per-problem breakdown, additional benchmarks]

### C. Value Model Architecture Details
[Detailed architecture diagrams, embedding dimensions, training curves]

### D. Proof Examples
[5-10 full proof transcripts showing CoProver's search trace vs. baselines]

### E. Compute Budget
[Detailed compute costs per component, total training time]
