# Formal Verification — Learning Path (From Zero)

## What Is Formal Verification?
Using math to PROVE software/hardware has no bugs, instead of just testing and hoping.
- Testing: "I tried 1000 inputs and none broke it" → maybe correct
- Formal verification: "I proved it works for ALL possible inputs" → definitely correct

---

## Reading Order (Don't Skip Ahead)

### Level 1: Conceptual Foundation (1-2 hours)

#### 📄 Martin Kleppmann — "AI will make formal verification go mainstream"
- **Link:** https://martin.kleppmann.com/2025/12/08/ai-formal-verification.html
- **PDF:** `papers/kleppmann-ai-formal-verification.pdf`
- **Why:** The blog post that started the current wave. Non-technical. Explains WHY formal verification matters NOW.
- **Read time:** 20 min

#### 📄 Software Foundations — Volume 1, Chapter 1 (Basics) only
- **Link:** https://softwarefoundations.cis.upenn.edu/lf-current/Basics.html
- **Why:** Gold standard textbook. Chapter 1 introduces the core idea: programs and proofs are the same thing (Curry-Howard correspondence).
- **Read time:** 45 min. Don't do exercises, just read for the IDEA.

---

### Level 2: Understand Lean4 (2-3 hours)

#### 📄 "Theorem Proving in Lean4" — Chapters 1-3
- **Link:** https://leanprover.github.io/theorem_proving_in_lean4/
- **Why:** Lean4 is THE language the field converged on. These chapters teach: what a theorem IS (a type), what a proof IS (a program), how the checker works (typechecking).
- **Read time:** 2 hours. Don't try to master — just get the mental model.

#### 📄 "The Hitchhiker's Guide to Logical Verification" — Chapters 1-3
- **Link:** https://github.com/blanchette/interactive_theorem_proving_2024
- **Why:** More beginner-friendly than Software Foundations. Uses Lean4 directly.

---

### Level 3: AI + Formal Verification (3-4 hours)

#### 📄 GPT-f (Polu & Sutskever, OpenAI, 2020) — THE FOUNDING PAPER
- **Link:** https://arxiv.org/abs/2009.03393
- **PDF:** `papers/gpt-f-2020.pdf`
- **Why:** First proof that LLMs can generate valid formal proofs. Found NEW proofs humans hadn't written. Everything builds on this.
- **Read:** Intro + Section 3 (method) + Section 5 (results). ~5 pages of core content.

#### 📄 LeanDojo (Yang et al., NeurIPS 2023) — THE INFRASTRUCTURE
- **Link:** https://arxiv.org/abs/2306.15626
- **PDF:** `papers/leandojo-2023.pdf`
- **Why:** Built the tools to train AI on Lean proofs. Turned theorem proving into a standard ML problem.
- **Read:** Intro + Section 2 (background) + Section 3 (approach).

#### 📄 DeepSeek-Prover-V1.5 (ICLR 2025) — RL MEETS FORMAL VERIFICATION
- **Link:** https://arxiv.org/abs/2408.15795
- **PDF:** `papers/deepseek-prover-v1.5-2024.pdf`
- **Why:** Introduced RL from proof assistant feedback (RLPAF). Lean says pass/fail → RL reward → model improves. THE intersection of your RL research and FV.
- **Read:** Intro + Section 3 (RLPAF method).

---

### Level 4: The Full Landscape (2-3 hours)

#### 📄 "A Survey on Deep Learning for Theorem Proving" (Li et al., COLM 2024)
- **Link:** https://arxiv.org/abs/2404.09939
- **PDF:** `papers/survey-dl-theorem-proving-2024.pdf`
- **Why:** THE survey paper. Covers everything. Use as reference when you see terms you don't know.
- **Read:** Sections 1-3 for the landscape map.

#### 📄 DeepSeek-Prover-V2 (2025) — CURRENT SOTA
- **Link:** https://arxiv.org/abs/2504.21801
- **PDF:** `papers/deepseek-prover-v2-2025.pdf`
- **Why:** 88.9% on MiniF2F. Key innovation: break hard proofs into subgoals, solve each one.
- **Read:** Intro + Section 2.

---

### Level 5: Software Verification (Optional, 2 hours)

#### 📄 AutoVerus (Microsoft, OOPSLA 2025) — VERIFIED RUST CODE
- **Link:** https://arxiv.org/abs/2409.13082
- **PDF:** `papers/autoverus-2025.pdf`
- **Why:** LLMs generating verified Rust code. Practical side of FV.

#### 📄 Vericoding Benchmark (Sep 2025)
- **Link:** https://arxiv.org/abs/2509.22908
- **PDF:** `papers/vericoding-2025.pdf`
- **Why:** Coined "vericoding" vs "vibecoding." Benchmarks LLM-generated verified code.

#### 📄 AlphaProof (DeepMind, Nature 2025) — IMO SILVER MEDAL
- **Link:** https://deepmind.google/blog/ai-solves-imo-problems-at-silver-medal-level/
- **Why:** The landmark result. AI solved IMO problems via RL + Lean4.

---

## What You DON'T Need Yet
- ❌ Coq, Isabelle, Agda (older — Lean4 won)
- ❌ Category theory or advanced math
- ❌ SAT/SMT solver internals (Z3 is a black box)
- ❌ Type theory beyond basics

## Total Time: ~15-20 hours over 1-2 weeks

## Quick Reference
```
Kleppmann blog → WHY formal verification matters now
Software Foundations Ch.1 → WHAT formal verification is
Lean4 tutorial Ch.1-3 → HOW proofs work in code
GPT-f → CAN AI do formal proofs? (yes)
LeanDojo → HOW to train AI on proofs
DeepSeek-V1.5 → RL + proof assistant feedback
Survey paper → THE full landscape
DeepSeek-V2 → CURRENT state of the art
AutoVerus → PRACTICAL verified code
```
