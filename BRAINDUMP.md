# Brain Dump: AlphaProof for Hardware

## The Thesis
The software FV + AI world has AlphaProof, DeepSeek-Prover, Harmonic ($1.45B), Theorem ($6M).
The hardware FV + AI world has... nothing ground-up. Just LLM wrappers around JasperGold.
Gap = opportunity.

## Why Ground-Up (Not JasperGold Wrapper)
- JasperGold is for humans. AI needs fast, programmable, rich-feedback verification.
- JasperGold is slow (min-hrs per property). Open-source bounded checking = seconds.
- JasperGold is a black box. ABC exposes solver internals = richer RL signal.
- JasperGold requires $$$$ licenses. Open-source = deploy anywhere, train anywhere.
- Can't co-train with JasperGold. CAN co-train with your own solver.

## The Stack (All Open-Source)
- Yosys: SystemVerilog → gate-level → AIG
- ABC: Model checking (IC3/PDR, BMC, k-induction)
- Z3: SAT/SMT solving
- sv2v: SystemVerilog → Verilog conversion
- PyVerilog / Slang: Verilog parsing

## Why Hardware FV Has Advantages Over Software FV
1. Deterministic oracle (pass/fail/counterexample, not just pass/fail)
2. Richer feedback (counterexample = concrete signal trace)
3. Clearer ROI ($1M+ per silicon bug vs deploy-a-patch for software)
4. FV already standard in chip design (not trying to convince industry to adopt)
5. Bounded state spaces (hardware is finite, software can be infinite)

## Why Hardware FV Is Harder Than Software FV
1. Verification is SLOW (model checking hits state space explosion)
2. No training data (no "hardware mathlib" — all proprietary)
3. Verilog/SV is a messy language (vs Lean4 which is clean)
4. Properties are design-specific (less transfer than math theorems)
5. Reward signal is sparse ("inconclusive" = no signal)

## Solutions to the Hard Problems
1. SLOW → Bounded model checking for RL training (seconds). Full proofs for validation only.
2. NO DATA → Synthetic Verilog generator. Build hardware mathlib from scratch.
3. MESSY LANGUAGE → Yosys handles synthesis. Work at AIG level, not source level.
4. DESIGN-SPECIFIC → Start with universal patterns (deadlock, liveness, protocol compliance).
5. SPARSE REWARD → Extract intermediate solver state (partial invariants, lemmas, states explored).

## The RL Co-Training Connection
- Generator: LLM that produces properties + proof strategies
- Verifier: ABC/Z3 model checker
- RLPAF: RL from proof assistant (solver) feedback
- Co-evolution: generator gets better at properties, learned heuristics guide solver
- Same paradigm as AlphaProof, DeepSeek-Prover, but for hardware

## Killer Demo (Month 1)
1. Take PicoRV32 (simple open-source RISC-V core, ~3000 LOC Verilog)
2. Yosys synthesizes to AIG
3. LLM generates 50 property candidates for the ALU
4. ABC checks each one (seconds per property with bounded checking)
5. Show: "AI generated N correct, meaningful, PROVEN properties"
6. Compare to what a junior verification engineer would write
7. Blog post + GitHub repo + Twitter thread

## What Makes This a Paper
- "AlphaHardware: RL-Guided Formal Property Verification with Open-Source Solvers"
- Contribution 1: First RL-from-verifier-feedback system for hardware FPV
- Contribution 2: Synthetic hardware training data generation pipeline
- Contribution 3: Rich intermediate solver feedback for RL (not just pass/fail)
- Submit to: DAC, ICCAD, FMCAD, or ML venue (ICML, NeurIPS)

## Competitive Landscape
- Synopsys (JasperGold) — adding AI features but can't open-source
- Cadence (VC Formal) — same
- Harmonic ($1.45B) — software FV only, not hardware
- Theorem ($6M) — software FV only
- Nobody doing ground-up AI-native hardware FV

## Open Questions
- Can bounded model checking give enough signal for RL? (need to test)
- How hard is synthetic Verilog generation? (probably medium — FSMs, FIFOs, buses)
- Does the Yosys → ABC pipeline handle enough SV for real designs? (yes for most)
- What's the right RL algorithm? (GRPO? PPO? AlphaZero-style MCTS?)
- Who's the customer? (fabless chip companies? verification teams? EDA vendors?)
