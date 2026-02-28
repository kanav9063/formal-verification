# Formal Verification for Software (Lean4 + AI) — Full Brain Dump

*Everything we discussed about pursuing formal verification on the software/math side.*

---

## Why Software FV Over Hardware FV (For Now)

| Dimension | Software (Lean) | Hardware (Yosys/ABC) |
|-----------|----------------|---------------------|
| Verification speed | Milliseconds | Seconds-hours |
| Training data | 200K+ theorems (mathlib4) | Nothing public |
| RL feasibility | **Proven** (AlphaProof, DeepSeek) | Unproven |
| Competition | High but tractable | Low but hard to produce results |
| Path to paper | 3-6 months | 12-18 months |
| Relevance to 3 labs | **Direct** (DeepMind built AlphaProof) | Tangential |
| Community | Exploding (Lean4, LeanDojo) | Tiny, corporate |

**Bottom line:** Lean gets you a paper faster, the labs care about it directly, and the infrastructure already exists. Do Lean first → get credibility → bring it to hardware later.

---

## The Landscape (From Deep Research)

### Key Systems (Ranked by Impact)

| System | Org | Year | Achievement |
|--------|-----|------|-------------|
| AlphaProof | DeepMind | 2025 | IMO silver medal via RL + Lean4. Nature paper. |
| DeepSeek-Prover-V2 | DeepSeek | 2025 | SOTA: 88.9% on MiniF2F. Open-source. Recursive subgoal decomposition. |
| LeanDojo / ReProver | Caltech | 2023 | Foundational open-source infra. NeurIPS. |
| GPT-f | OpenAI | 2020 | First LLM-generated proofs in Metamath. Seminal paper. |
| AutoVerus | Microsoft | 2025 | >90% verified Rust code via Verus. OOPSLA. |
| LeanCopilot | lean-dojo | 2024 | LLM proof suggestions directly in Lean IDE. |
| Baldur | FSE | 2023 | Whole-proof generation + error-driven repair. |
| COPRA | UT Austin | 2024 | In-context learning (GPT-4, no fine-tuning) for proofs. |
| Lean-STaR | CMU | 2025 | Interleaved NL thinking + formal tactics. ICLR. |
| APOLLO | 2025 | 84.9% MiniF2F with compiler-guided repair. |

### Key Techniques

| Technique | What It Does | Used By |
|-----------|-------------|---------|
| **RLPAF** | RL from proof assistant feedback (pass/fail as reward) | AlphaProof, DeepSeek-V1.5 |
| **Retrieval-Augmented Proving** | Retrieve relevant lemmas, then generate tactics | LeanDojo/ReProver |
| **Autoformalization** | Natural language → formal Lean statement | AlphaProof (80M statements) |
| **Subgoal Decomposition** | Break hard proofs into easier lemmas | DeepSeek-V2 |
| **Whole-Proof Generation** | Predict entire proof at once, repair from errors | Baldur |
| **Interleaved Thinking** | NL chain-of-thought between formal tactics | Lean-STaR |
| **Multi-Agent Proving** | Multiple agents generate candidate proofs | AutoVerus (10 agents) |
| **Test-Time RL** | Improve during inference via problem variations | AlphaProof |
| **Compiler-Guided Repair** | Use Lean error messages to iteratively fix proofs | APOLLO |

### Key Repos

| Repo | Stars | What It Does |
|------|-------|-------------|
| leanprover-community/mathlib4 | 2,941 | THE training data (200K+ theorems) |
| lean-dojo/LeanCopilot | 1,225 | LLM proof assistant in Lean IDE |
| lean-dojo/LeanDojo | 770 | ML infrastructure for theorem proving |
| lean-dojo/ReProver | 316 | RAG-based theorem prover |
| google-deepmind/alphageometry | 4,788 | Neuro-symbolic geometry prover |
| wellecks/ntptutorial | 178 | Best tutorial for getting started |

### Startups / Companies

| Company | Funding | Thesis |
|---------|---------|--------|
| Harmonic AI | $295M, $1.45B val | Hallucination-free AI via Lean4 verification. Aristotle chatbot. |
| Theorem | $6M (YC, Khosla) | Fractional proof decomposition. Found bugs at Anthropic. |
| Logical Intelligence | Undisclosed | Aleph agent — automates FV end-to-end. |
| AWS | (internal) | Cedar authorization policies verified in Lean. Production. |

---

## The Research Opportunity: What's NOT Saturated

### MiniF2F is dying (88.9%)
DeepSeek-Prover-V2 nearly saturated MiniF2F. The field needs:
- Harder benchmarks (PutnamBench: only 49/658 solved)
- Real software verification benchmarks (not just math)
- Cross-domain generalization tests

### What Nobody Has Done Well Yet

1. **RL Co-Training for Theorem Proving**
   - Current systems: RL trains the generator, verifier (Lean) is frozen
   - Novel: co-train the proof SEARCH HEURISTICS alongside the generator
   - The generator learns to write better proofs; learned heuristics guide the search to explore more promising branches
   - This is YOUR RL co-training thesis applied to Lean
   - Nobody has published this cleanly

2. **Specification Generation (the real bottleneck)**
   - Everyone focuses on "prove this theorem." Who writes the theorem?
   - For software: automatically generate formal specs from informal requirements
   - For math: propose interesting conjectures worth proving
   - Underexplored, high impact

3. **Beyond Math → Real Code Verification**
   - AutoVerus showed Rust verification is possible
   - But nobody has combined LeanDojo-style infrastructure with real software verification
   - seL4 LLM work (Feb 2026) just started this — early mover advantage

4. **Retrieval + Subgoal Decomposition (novel combination)**
   - DeepSeek-V2: subgoal decomposition (break proof into lemmas)
   - ReProver: retrieval (find relevant existing lemmas)
   - Combining both: decompose → retrieve relevant lemmas for each subgoal → prove
   - Nobody has published this combination cleanly

---

## Concrete Research Plans (Pick One)

### Plan A: RL Co-Training for Lean Proofs (Most Novel)
**Your unique angle. Maps directly to RL co-training research.**

**Idea:** Don't just train the proof generator — also train a learned proof search heuristic that improves alongside it.

**Architecture:**
```
Generator (LLM)
  → Produces proof tactic candidates
Heuristic Model (smaller model)  
  → Scores/ranks which tactics to try first
  → Guides beam search / MCTS
Lean Kernel
  → Verifies proof attempts
  → Feedback trains BOTH models
```

**Why this is novel:**
- AlphaProof uses RL but with fixed MCTS heuristics
- DeepSeek-Prover uses RL but the search strategy is fixed
- Nobody co-trains the generator AND the search heuristic

**Infrastructure:** Build on LeanDojo. Use mathlib4 theorems for training. Benchmark on MiniF2F, PutnamBench.

**Timeline:** 4-6 months to first results. Paper-ready in 6-8 months.

**Target venue:** ICML 2027, NeurIPS 2027, or ICLR 2027.

### Plan B: Retrieval-Augmented Subgoal Proving (Most Practical)
**Combines two proven techniques nobody has merged well.**

**Idea:** When proving a hard theorem, decompose into subgoals (DeepSeek-V2 style), then for each subgoal retrieve the most relevant lemmas from mathlib4 (ReProver style), then prove each subgoal with retrieval context.

**Why this could work:**
- Subgoal decomposition reduces complexity
- Retrieval reduces hallucination (the model sees real, verified lemmas)
- Together they should outperform either alone

**Infrastructure:** LeanDojo + ReProver + DeepSeek-Prover-V2 (open-source).

**Timeline:** 3-4 months to results. Paper-ready in 5-6 months.

**Target venue:** NeurIPS 2027 or EMNLP 2027.

### Plan C: Open-Source Benchmark for Code Verification (Most Impactful)
**The community needs this. Nobody has built it.**

**Idea:** Create a benchmark suite for LLM-guided formal verification of REAL code (not just math). Targets: Rust (via Verus), Python (via type-checking + contracts), or general (via Dafny).

**What it includes:**
- 100+ programs with formal specifications
- Baseline results from frontier LLMs (zero-shot + few-shot)
- Evaluation framework: spec validity, proof completeness, code correctness
- Difficulty tiers (trivial → research-level)

**Why this matters:**
- VeriBench/Vericoding showed only ~12% full verification
- The field needs better benchmarks to drive progress
- First good benchmark = high citation paper

**Timeline:** 2-3 months to build. Paper-ready in 4 months.

**Target venue:** FMCAD, ASE, or ISSTA.

---

## What You'd Need to Learn

### Must-Have (2-4 weeks)
- **Lean4 basics** — syntax, tactics, proof mode
  - Start: [Theorem Proving in Lean4](https://leanprover.github.io/theorem_proving_in_lean4/)
  - Then: [Mathematics in Lean](https://leanprover-community.github.io/mathematics_in_lean/)
- **LeanDojo setup** — install, extract data, run ReProver
  - Repo: github.com/lean-dojo/LeanDojo
  - Tutorial: wellecks/ntptutorial

### Good-to-Have (4-8 weeks)
- **PyTorch training loops** — for RL fine-tuning (the "rebuild your hands" plan)
- **GRPO / PPO implementation** — for the RL loop
- **mathlib4 familiarity** — understand how theorems are structured

### Already Have
- RL co-training theory (from your research)
- Multi-agent orchestration (from ChipAgents)
- Research taste (knowing what's interesting)
- Strong writing (for the paper)

---

## The Citation Tree (Know Your Ancestors)

```
GPT-f (2020, OpenAI) ← THE founding paper
    │
    ├──► Autoformalization (2022) → AlphaProof (2025, Nature)
    │                                     │
    │                                     └──► DeepSeek-Prover-V2 (SOTA)
    │
    ├──► LeanDojo (2023) → ReProver → LeanCopilot → COPRA
    │
    ├──► Baldur (2023) → whole-proof generation
    │
    ├──► DeepSeek-Prover-V1.5 (2024) → introduced RLPAF
    │
    ├──► Lean-STaR (2024) → interleaved thinking
    │
    └──► AutoVerus (2024) → verified Rust code
```

**Your paper would cite:** LeanDojo, DeepSeek-Prover-V1.5/V2, ReProver, and your RL co-training papers (PRIME, RL Tango). You'd be at the intersection of two lineages.

---

## The Pitch (For Grad School Apps / Lab Applications)

> "The software world proved that AI + formal verification works — AlphaProof solved IMO problems, Harmonic built a $1.45B company, and DeepSeek-Prover-V2 achieves 88.9% on MiniF2F. All use RL from verifier feedback with frozen verifiers. My research extends this with co-training: the proof generator and the proof search heuristic evolve together, avoiding the reward hacking that occurs with frozen verifiers (as shown in the RL co-training literature). I have unique domain expertise from building multi-agent formal verification systems for semiconductor design, and I'm applying those insights to software theorem proving."

---

## Key Sources

### Must-Read Papers
1. AlphaProof — Nature 2025
2. DeepSeek-Prover-V2 — arXiv 2504.21801
3. LeanDojo — NeurIPS 2023
4. DeepSeek-Prover-V1.5 — ICLR 2025
5. Lean-STaR — ICLR 2025
6. PRIME (RL co-training) — from your existing research

### Must-Read Posts
1. Kleppmann — "AI will make formal verification go mainstream" (Dec 2025)
2. Amplify Partners — "The Agentic Mullet" (Feb 2026)
3. Congdon — "The Coming Need for Formal Specification" (Dec 2025)

### Must-Use Tools
1. LeanDojo — github.com/lean-dojo/LeanDojo
2. mathlib4 — github.com/leanprover-community/mathlib4
3. ntptutorial — github.com/wellecks/ntptutorial
4. ReProver — github.com/lean-dojo/ReProver

### Full Deep Research
- GitHub: https://github.com/kanav9063/kanav-research/tree/master/formal-verification-ai-agents
- Notion: saved under Research page
- 34 total sources: 15 web | 6 HN threads | 12 papers | 8 repos
