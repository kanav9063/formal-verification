# EXECUTION PLAN: Learned Proof Search for Lean4 Theorem Proving

**Project codename:** CoProver
**Target paper:** "CoProver: Co-Training Proof Generation and Search Heuristics for Automated Theorem Proving"
**Target venues:** ICML 2027 (deadline ~Jan 2027), NeurIPS 2027 (deadline ~May 2027), ICLR 2028 (deadline ~Oct 2027)
**Timeline:** 20 weeks (March 15 – August 2, 2026)
**Status:** Week 0 — starting from zero code

---

## THE THESIS (One Paragraph)

Every current neural theorem prover — DeepSeek-Prover-V2, Goedel-Prover-V2, BFS-Prover, Kimina-Prover — trains the **tactic generator** (the LLM that proposes proof steps) but uses a **fixed search strategy** (beam search, BFS, or MCTS with hand-tuned heuristics). The search strategy decides *which proof states to explore next*, and it's arguably more important than the individual tactics — a mediocre tactic generator with brilliant search beats a brilliant generator with naive search. We propose co-training both: a tactic generator LLM and a lightweight **proof state value model** that learns to score and prioritize proof states. The generator gets better at producing tactics; the value model gets better at predicting which states lead to completed proofs; they improve each other through iterative training on Lean4's verified feedback.

---

## WHY THIS IS NOVEL (Positioning Against Existing Work)

| System | What they train | Search strategy | Our difference |
|--------|----------------|-----------------|----------------|
| DeepSeek-Prover-V2 | Generator (GRPO) | Fixed tree search | We also train the search heuristic |
| Goedel-Prover-V2 | Generator (SFT + self-correction) | Fixed best-of-N | We learn which states to explore |
| BFS-Prover | Generator (expert iteration) | Fixed best-first search | We learn the priority function |
| Kimina-Prover | Generator (SFT + long CoT) | Fixed beam search | We learn beam scoring |
| GAR | Generator + problem composer (adversarial) | Fixed search | We co-train search, not problems |
| AlphaProof | Generator (RL) | MCTS with fixed heuristics | We learn MCTS value function |
| **CoProver (ours)** | **Generator + value model** | **Learned, co-trained** | **—** |

**The closest related work is AlphaZero/MuZero applied to theorem proving.** AlphaProof reportedly uses MCTS, but with hand-tuned heuristics (not published in detail). BFS-Prover showed simple best-first search is competitive. We take this further: learn the "best-first" scoring function and co-train it with the generator.

---

## ARCHITECTURE

```
┌──────────────────────────────────────────────────┐
│                   CoProver                        │
│                                                   │
│  ┌─────────────────┐    ┌──────────────────────┐ │
│  │ Tactic Generator │    │  Proof State Value   │ │
│  │   (7B LLM)       │    │   Model (small MLP   │ │
│  │                   │    │   or 1B transformer)  │ │
│  │ Input: proof state│    │ Input: proof state    │ │
│  │ Output: next tac  │    │ Output: P(provable)   │ │
│  └────────┬──────────┘    └──────────┬───────────┘ │
│           │                          │              │
│           ▼                          ▼              │
│  ┌─────────────────────────────────────────────┐   │
│  │          Learned Best-First Search           │   │
│  │  1. Start from goal state                    │   │
│  │  2. Generator proposes K tactics             │   │
│  │  3. Apply each → new proof states            │   │
│  │  4. Value model scores each state            │   │
│  │  5. Expand highest-scoring state next        │   │
│  │  6. Repeat until QED or budget exhausted     │   │
│  └────────────────────┬────────────────────────┘   │
│                       │                             │
│                       ▼                             │
│              ┌─────────────────┐                    │
│              │   Lean4 Kernel   │                    │
│              │  (verification)  │                    │
│              └────────┬────────┘                     │
│                       │                             │
│              pass/fail + error msgs                  │
│                       │                             │
│                       ▼                             │
│  ┌─────────────────────────────────────────────┐   │
│  │           Co-Training Loop                   │   │
│  │  • Successful proofs → SFT data for gen     │   │
│  │  • All states on successful paths →         │   │
│  │    positive examples for value model         │   │
│  │  • All states on failed paths →             │   │
│  │    negative examples for value model         │   │
│  │  • Iterate: better search → more proofs →   │   │
│  │    better training data → better models      │   │
│  └─────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
```

---

## BENCHMARKS

| Benchmark | Size | Difficulty | Status (SOTA, Mar 2026) | Our target |
|-----------|------|-----------|------------------------|------------|
| MiniF2F-test | 244 | Olympiad | 90.4% (Goedel-V2 pass@32) | Near-saturated. Use for validation only. |
| ProofNet-test | 186 | Undergrad math | 25.8% (GAR) | Secondary benchmark |
| PutnamBench | 658 | Competition | 86/658 solved (Goedel-V2) | **Primary benchmark** — most headroom |
| SorryDB | 1000+ | Real-world Lean | New (Mar 2026) | **Primary benchmark** — real-world signal |

**Why PutnamBench + SorryDB:** MiniF2F is near-saturated. PutnamBench is the new frontier for hard math. SorryDB is brand new (March 2026) and represents real-world Lean usage — proving this works on practical code, not just competition math, is a much stronger story.

---

## WEEK-BY-WEEK PLAN

### Phase 0: Foundation (Weeks 1-3) — Mar 15 – Apr 5

**Goal:** Go from zero Lean knowledge to running an existing prover on a benchmark.

#### Week 1 (Mar 15-22): Learn Lean4
- **Day 1-2:** Install Lean4 via elan. Work through [Theorem Proving in Lean4](https://leanprover.github.io/theorem_proving_in_lean4/) chapters 1-3.
  ```bash
  curl https://elan.lean-lang.org/install.sh -sSf | sh
  ```
- **Day 3-4:** Work through [wellecks/ntptutorial](https://github.com/wellecks/ntptutorial) — specifically Part 1 (Lean basics) and Part 2 (neural theorem proving basics).
- **Day 5-6:** Read GPT-f paper (intro + method + results). Read LeanDojo paper (intro + approach).
- **Day 7:** Write 10 simple Lean4 proofs by hand. Understand: what is a tactic? what is a proof state? what does the Lean kernel actually check?
- **Deliverable:** Can write basic Lean proofs. Understands tactic mode, proof states, type checking.

#### Week 2 (Mar 22-29): Set Up LeanDojo + Run ReProver
- **Day 1-2:** Clone and install LeanDojo. Extract data from a small mathlib4 subset.
  ```bash
  git clone https://github.com/lean-dojo/LeanDojo
  cd LeanDojo && pip install -e .
  ```
- **Day 3-4:** Run the ReProver baseline on MiniF2F. Understand the data format: (proof state, tactic, next state) triples.
  ```bash
  git clone https://github.com/lean-dojo/ReProver
  ```
- **Day 5-7:** Study the proof search implementation in ReProver. This is a best-first search with a fixed priority. Document exactly how it works: what is the search tree structure? how are states scored? how are tactics ranked?
- **Deliverable:** ReProver running on MiniF2F. Documented understanding of the search loop.

#### Week 3 (Mar 29 – Apr 5): Run BFS-Prover / Understand Expert Iteration
- **Day 1-3:** Read BFS-Prover paper carefully. Clone the repo. Understand the expert iteration pipeline: prove → collect data → retrain → prove harder things.
  ```bash
  git clone https://github.com/deepseek-ai/BFS-Prover  # or equivalent
  ```
- **Day 4-5:** Read DeepSeek-Prover-V1.5 (RLPAF method section) and V2 (subgoal decomposition).
- **Day 6-7:** Read GAR paper (co-training framework). Document specifically how GAR differs from what we're doing:
  - GAR: co-trains problem *composer* + solver (adversarial curriculum)
  - CoProver: co-trains search *heuristic* + generator (cooperative improvement)
- **Deliverable:** Clear understanding of the entire landscape. Written 2-page "related work" draft.

**Phase 0 exit criteria:** Can run ReProver on MiniF2F, get a pass rate, and explain exactly how every component works.

---

### Phase 1: Baseline + Data Pipeline (Weeks 4-6) — Apr 5 – Apr 26

**Goal:** Build the data infrastructure and establish baselines we'll improve against.

#### Week 4 (Apr 5-12): Data Extraction Pipeline
- Extract proof state trajectories from mathlib4 using LeanDojo:
  - Input: theorem statement
  - Output: sequence of (proof_state, tactic_applied, resulting_proof_state) triples
- Build dataset of ~50K theorem trajectories with full state traces
- Key: we need not just the tactics, but the **intermediate proof states** — these are training data for the value model
- Store as: `{theorem_id, step_idx, proof_state_text, tactic, next_state_text, is_on_successful_path: bool}`
- **Deliverable:** Dataset of 50K+ proof trajectories with full state information.

#### Week 5 (Apr 12-19): Proof State Embedding
- Design the proof state representation for the value model:
  - **Option A (simple):** Use the generator LLM's last-layer embedding of the proof state text
  - **Option B (better):** Train a small encoder (e.g., 125M parameter transformer) on proof states
  - **Option C (cheapest):** Use a frozen sentence transformer (e.g., E5-large) on proof state text
- Start with Option A (parasitic on the generator's representations) — simplest to implement
- Build a feature extraction pipeline: proof_state_text → embedding vector → value prediction
- **Deliverable:** Proof state embedder that produces fixed-size vectors from Lean proof states.

#### Week 6 (Apr 19-26): Baseline Measurements
- Run ReProver on MiniF2F-test and PutnamBench (subset). Record:
  - Pass rate at pass@1, pass@8, pass@32
  - Average number of nodes expanded per proof
  - Average time per proof attempt
  - Distribution of proof lengths
- Run a frontier LLM (e.g., DeepSeek-Prover-V2-7B if weights available, or Claude/GPT via API) on the same benchmarks with simple beam search
- Run the "Minimal Agent for ATP" (arxiv 2602.24273) baseline — it's specifically designed as a reference
- **Deliverable:** Baseline table with 3+ methods on 2+ benchmarks. This becomes Table 1 of the paper.

**Phase 1 exit criteria:** Have training data, have embeddings, have baselines to beat.

---

### Phase 2: Value Model (Weeks 7-10) — Apr 26 – May 24

**Goal:** Build and train the proof state value model — the core novel component.

#### Week 7 (Apr 26 – May 3): Value Model v0 (Supervised)
- Architecture: MLP head on proof state embeddings
  - Input: proof state embedding (from Week 5)
  - Output: scalar in [0, 1] — predicted probability this state leads to a completed proof
- Training data: from the extracted trajectories
  - States on successful proof paths → label 1.0
  - States on failed proof paths → label 0.0
  - (Later: use softer labels based on distance to QED)
- Train on 80% of mathlib trajectories, validate on 20%
- Metric: does the value model actually predict which states are closer to completion?
- **Deliverable:** Trained value model v0. Validation accuracy on held-out trajectories.

#### Week 8 (May 3-10): Integrate Value Model into Search
- Replace ReProver's fixed priority function with the learned value model
- Search procedure:
  1. Start with initial proof state (the theorem goal)
  2. Generator proposes top-K tactics (K=32)
  3. Apply each tactic → K new proof states
  4. Value model scores each new state
  5. Add to priority queue (sorted by value score)
  6. Pop highest-scoring state, go to step 2
  7. Stop on QED or budget exhausted (N=512 node expansions)
- **This is "learned best-first search"** — the simplest possible instantiation of the idea
- **Deliverable:** Working search loop with learned value function. Can run on MiniF2F.

#### Week 9 (May 10-17): First Results + Debugging
- Run learned search on MiniF2F-test and PutnamBench
- Compare against baselines from Week 6
- Expected: modest improvement (5-15% relative) from value model, even without co-training
- If no improvement: debug the value model
  - Is it actually discriminative? (check accuracy on held-out states)
  - Is the search using it correctly? (log search decisions)
  - Are the embeddings meaningful? (visualize with t-SNE)
- **Deliverable:** First results table. Learned search vs. fixed search.

#### Week 10 (May 17-24): Value Model Improvements
- Better labels: instead of binary 0/1, use:
  - `value = 1.0 / (1 + steps_remaining_to_QED)` for successful paths
  - `value = 0.0` for failed paths
  - Or: Monte Carlo returns — what fraction of rollouts from this state succeed?
- Better architecture: try a small transformer (125M) trained directly on proof state token sequences, with a scalar value head
- Better features: include meta-info in the state representation:
  - Number of remaining goals
  - Goal complexity (token length, depth of terms)
  - Which tactics have been tried and failed
- **Deliverable:** Improved value model v1. Updated results.

**Phase 2 exit criteria:** Learned search demonstrably outperforms fixed search on at least one benchmark.

---

### Phase 3: Co-Training Loop (Weeks 11-14) — May 24 – Jun 21

**Goal:** Build the iterative co-training pipeline — the key contribution.

#### Week 11 (May 24-31): Expert Iteration for Generator
- Implement expert iteration (following BFS-Prover):
  1. Run current generator + value model on unsolved theorems
  2. Collect successful proof trajectories (new training data)
  3. Fine-tune generator on new data (SFT)
  4. Go to step 1
- Start with DeepSeek-Prover-V2-7B (or Goedel-Prover-V2-7B) as base generator
- Use LoRA for efficient fine-tuning (don't full-finetune a 7B)
- **Deliverable:** Expert iteration pipeline running end-to-end. Generator improves over iterations.

#### Week 12 (May 31 – Jun 7): Co-Training: Value Model Updates with Generator
- After each expert iteration round:
  1. Generator is updated → probes different parts of the search space
  2. Collect ALL trajectories (successful + failed) from the new generator
  3. Retrain value model on the expanded trajectory dataset
  4. The value model now reflects the new generator's behavior
- The key insight: **the value model must track the generator.** A value model trained on old generator trajectories gives bad scores for new generator states.
- Implement round-robin training:
  - Round 1: Train generator → collect data → train value model
  - Round 2: Use new value model in search → train generator on new proofs → train value model
  - Round 3: Repeat
- **Deliverable:** Full co-training loop. Both models improve per round.

#### Week 13 (Jun 7-14): Scaling the Loop
- Run 5-10 rounds of co-training on progressively harder theorem sets:
  - Round 1-2: Easy mathlib theorems (provable in ≤5 steps)
  - Round 3-5: Medium mathlib theorems (5-15 steps)
  - Round 6-10: Hard theorems (MiniF2F, PutnamBench-level)
- Track per-round metrics:
  - Theorems solved (cumulative)
  - Average proof length
  - Search efficiency (nodes expanded per proof)
  - Value model accuracy on held-out states
- Plot the co-training curves — this becomes Figure 2 of the paper
- **Deliverable:** Co-training convergence curves showing both models improve.

#### Week 14 (Jun 14-21): RL Fine-Tuning (Optional but Powerful)
- Replace SFT expert iteration with GRPO (Group Relative Policy Optimization):
  - For each theorem, sample K proof attempts
  - Reward: +1 for completed proof, 0 for failure
  - GRPO update on the generator
- Also try: reward shaping using the value model
  - Instead of sparse 0/1 reward, give intermediate reward based on value model predictions
  - "You reached a state with value 0.8 — that's better than your average attempt"
  - This is the RLPAF idea (DeepSeek-V1.5) but with a LEARNED value function instead of just Lean's pass/fail
- **Deliverable:** RL-trained generator. Compare: SFT expert iteration vs. GRPO vs. GRPO+value-shaped rewards.

**Phase 3 exit criteria:** Co-training demonstrably outperforms training either model alone. This is THE result.

---

### Phase 4: Ablations + Benchmarking (Weeks 15-17) — Jun 21 – Jul 12

**Goal:** Generate all the results for the paper.

#### Week 15 (Jun 21-28): Full Benchmark Evaluation
Run the final CoProver model on all benchmarks:

| Benchmark | Metric | Method |
|-----------|--------|--------|
| MiniF2F-test | pass@1, @8, @32 | CoProver vs. baselines |
| ProofNet-test | pass@1, @8, @32 | CoProver vs. baselines |
| PutnamBench | problems solved @32, @128, @512 | CoProver vs. baselines |
| SorryDB (snapshot) | problems solved @32 | CoProver vs. baselines |

Baselines to compare against:
1. ReProver (retrieval-augmented, fixed search)
2. BFS-Prover (expert iteration, fixed BFS)
3. DeepSeek-Prover-V2-7B (if weights available)
4. Goedel-Prover-V2-7B/8B (open source)
5. Generator-only (our generator, but with fixed BFS — no value model)
6. Value-only (ReProver generator, but with our learned search)

**The ablation that matters most:** #5 vs. full CoProver. This isolates the value model's contribution.

#### Week 16 (Jun 28 – Jul 5): Ablation Studies
- **Ablation 1: Co-training vs. separate training**
  - Train generator and value model independently (no co-training loop)
  - vs. CoProver with co-training
  - Expected: co-training wins because the value model adapts to the generator
- **Ablation 2: Value model architecture**
  - MLP on frozen embeddings vs. small transformer vs. same-model value head
- **Ablation 3: Search budget**
  - How does performance scale with # nodes expanded? (64, 128, 256, 512, 1024)
  - Key result: CoProver should reach the same performance as baselines with FEWER nodes (more efficient search)
- **Ablation 4: Co-training rounds**
  - Performance after 1, 2, 5, 10 rounds of co-training
  - Show diminishing but positive returns
- **Ablation 5: SFT vs. GRPO vs. GRPO+value-shaping**
  - Which training method for the generator works best?

#### Week 17 (Jul 5-12): Analysis + Figures
- **Search efficiency analysis:** For proofs found by both CoProver and baselines, compare # nodes expanded. CoProver should explore fewer dead ends.
- **Qualitative examples:** Pick 3-5 theorems that CoProver solves but baselines don't. Show the search tree. Annotate where the value model made a critical correct decision.
- **Failure analysis:** Pick 3-5 theorems CoProver fails on. Why? (generator limitation? value model misjudges? fundamentally hard?)
- **Value model interpretability:** What proof state features correlate with high value? (fewer remaining goals? shorter terms? presence of certain tactics?)
- Generate all plots: co-training curves, pass rate vs. search budget, ablation bar charts

**Phase 4 exit criteria:** All results tables and figures complete.

---

### Phase 5: Paper (Weeks 18-20) — Jul 12 – Aug 2

#### Week 18 (Jul 12-19): Paper Draft v1
- Structure:
  1. **Introduction** (1.5 pages): The gap — everyone trains generators, nobody trains search. We do both.
  2. **Related Work** (1 page): LeanDojo/ReProver, DeepSeek-Prover, BFS-Prover, GAR, AlphaProof
  3. **Method** (3 pages): Architecture, value model, co-training loop, search procedure
  4. **Experiments** (3 pages): Benchmarks, baselines, main results, ablations
  5. **Analysis** (1 page): Efficiency, qualitative examples, failure cases
  6. **Conclusion** (0.5 pages)
- Total: ~10 pages + appendix

#### Week 19 (Jul 19-26): Paper Draft v2 + Polish
- Revise based on self-review
- Add appendix: hyperparameters, full results tables, additional examples
- Make all figures publication-quality

#### Week 20 (Jul 26 – Aug 2): Final Paper + Open-Source Release
- Final revisions
- Prepare code release: clean repo, README, reproducibility instructions
- Push to arXiv
- Write Twitter thread + blog post
- Submit to venue (or hold for ICML/NeurIPS deadline)

---

## COMPUTE REQUIREMENTS

| Component | Requirement | Cost estimate |
|-----------|-------------|---------------|
| Lean4 proof checking | CPU-heavy, parallelizable | Free (local or cloud CPUs) |
| LeanDojo data extraction | ~8 hours on 32-core machine | $20-50 cloud |
| Generator fine-tuning (7B, LoRA) | 1x A100 80GB, ~6 hours per round | $10-15/round |
| Value model training | 1x A100 or even V100, ~1 hour | $2-5/round |
| Proof search (evaluation) | CPU for Lean + GPU for inference | $50-100 per full benchmark run |
| 10 co-training rounds | ~$200-300 total GPU | |
| Full benchmark eval (all methods) | ~$300-500 | |
| **Total estimate** | | **$500-1000** |

Can be done on a single A100 instance rented on Lambda Labs / RunPod / vast.ai. Or: use university cluster if available (UCSB has GPU resources via CNSI/CSIL).

---

## RISK REGISTER

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Value model doesn't improve search | Medium | High | Ablate thoroughly. Even small improvement is publishable if search efficiency gains are shown. |
| Co-training doesn't beat independent training | Medium | High | Fall back to "learned search heuristic" (without co-training) as contribution. Still novel. |
| Can't match SOTA on MiniF2F | High | Low | MiniF2F is saturated. Focus on PutnamBench and SorryDB where there's headroom. |
| Compute insufficient for 10 rounds | Low | Medium | Use LoRA, smaller base models (1.5B), fewer rounds. |
| LeanDojo/tooling breaks or is outdated | Medium | Medium | LeanDojo is actively maintained. SorryDB team (Mar 2026) uses it. Have fallback to direct Lean REPL interaction. |
| GAR paper scoops us | Low | Medium | GAR co-trains problems, not search. Different contribution. Cite and differentiate. |

---

## KEY PAPERS TO CITE (Updated March 2026)

### Directly relevant (the "ancestors" of this work)
1. **LeanDojo** (Yang et al., NeurIPS 2023) — infrastructure we build on
2. **ReProver** (Yang et al., NeurIPS 2023) — baseline prover
3. **DeepSeek-Prover-V1.5** (Xin et al., ICLR 2025) — introduced RLPAF
4. **DeepSeek-Prover-V2** (Ren et al., 2025) — subgoal decomposition, current SOTA at time of writing
5. **BFS-Prover** (Xin et al., ACL 2025) — best-first search baseline, expert iteration
6. **Goedel-Prover-V2** (Lin et al., 2025) — scaffolded data synthesis, self-correction, 90.4% MiniF2F
7. **GAR** (Wang et al., 2025/2026) — adversarial co-training of composer + solver. Closest related work.
8. **Kimina-Prover** (MoonshotAI, 2025) — first large formal reasoning model
9. **AlphaProof** (DeepMind, Nature 2025) — RL + Lean4 for IMO
10. **Aristotle** (Achim/Tenev et al., 2025) — IMO gold, formal + informal reasoning
11. **SorryDB** (Letson et al., 2026) — real-world Lean benchmark
12. **Minimal Agent for ATP** (Requena et al., 2026) — simple agentic baseline

### RL co-training (from your existing research — the bridge)
13. **PRIME** — process reward co-training
14. **RL Tango** — generator-verifier co-evolution
15. **GAR/CURE/MARS** — related RL paradigms

### Foundational
16. **GPT-f** (Polu & Sutskever, 2020) — first LLM proofs
17. **AlphaZero** (Silver et al., 2018) — learned value function for search (our inspiration)
18. **Lean-STaR** (Lin et al., ICLR 2025) — interleaved thinking

---

## DECISION LOG

| Date | Decision | Rationale |
|------|----------|-----------|
| 2026-03-14 | Software FV (Lean4), not hardware | Verification is ms (not hours), 200K+ training data exists, RL proven to work, direct relevance to top labs. Hardware comes later. |
| 2026-03-14 | Co-train search heuristic + generator | Novel gap: everyone trains generators with fixed search. GAR trains composers, not search. AlphaZero-style learned value for theorem proving is unexplored. |
| 2026-03-14 | PutnamBench + SorryDB as primary benchmarks | MiniF2F is 90.4% saturated. PutnamBench has headroom. SorryDB is real-world and brand new. |
| 2026-03-14 | Start with 7B base model + LoRA | Practical for single-GPU training. Goedel-V2 showed 7B can be competitive with better training. |
| 2026-03-14 | 20-week timeline to paper | Aggressive but feasible. Phase 0 (learning) is 3 weeks. Core implementation is 11 weeks. Paper is 3 weeks. |

---

## IMMEDIATE NEXT ACTIONS (This Weekend)

1. **Install Lean4:**
   ```bash
   curl https://elan.lean-lang.org/install.sh -sSf | sh
   source ~/.profile
   lean --version
   ```

2. **Clone ntptutorial:**
   ```bash
   cd ~/formal-verification
   git clone https://github.com/wellecks/ntptutorial
   ```

3. **Read Kleppmann blog (30 min):** `papers/kleppmann-ai-formal-verification.txt`

4. **Work through ntptutorial Part 1** (Lean basics, 2 hours)

5. **Read GPT-f paper** intro + section 3 (1 hour)

That's 4-5 hours of work. By Sunday night you should be able to write a basic Lean proof and understand the proof state → tactic → proof state loop that the entire project builds on.
