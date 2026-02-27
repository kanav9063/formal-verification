# Formal Verification + AI for Hardware

## Vision
Build AlphaProof for hardware — an AI-native formal verification system using open-source tools.

## Architecture
```
LLM Agent (Generator)
    → Generates SVA properties
    → Generates proof strategies
    → Refines based on solver feedback
         ↓
Open-Source Verification Stack
    → Yosys (SV → AIG synthesis)
    → ABC (model checking, IC3/PDR)
    → Z3/CaDiCaL (SAT/SMT)
    → Rich intermediate feedback
         ↓
RL Training Loop
    → Bounded model checking (fast, seconds)
    → Rich signal (partial proofs, invariants, counterexamples)
    → Synthetic RTL training data generation
```

## Key Insight
JasperGold is a product for human engineers. This is built for AI agents.
- Humans need GUIs, scripts, licenses
- AI needs fast, programmable, rich-feedback verification
- Open-source solvers (ABC, Z3) provide exactly this

## Stack
- **Synthesis:** Yosys (SV → gate-level → AIG)
- **Model Checking:** ABC (IC3/PDR, bounded model checking)
- **SAT/SMT:** Z3, CaDiCaL, MiniSat
- **LLM:** Frontier models for property generation
- **RL:** veRL / TRL for training

## Roadmap
1. **Month 1:** PoC — Yosys + ABC + LLM on simple Verilog modules
2. **Month 2-3:** Synthetic training data + first RL loop
3. **Month 4-6:** Scale to real RISC-V cores (PicoRV32, VexRiscV)
4. **Month 7-12:** Co-training (generator + solver heuristics evolve together)

## Research Connection
- Same generator/verifier loop as AlphaProof (DeepMind) and DeepSeek-Prover
- RL from verifier feedback (RLPAF) applied to hardware
- Hardware has structural advantages: deterministic oracles, counterexample feedback, $1M+ per silicon bug

## Deep Research
See: https://github.com/kanav9063/kanav-research/tree/master/formal-verification-ai-agents

## Related
- [The Agentic Mullet: Code in the Front, Proofs in the Back](https://www.amplifypartners.com/blog-posts/the-agentic-mullet-code-in-the-front-proofs-in-the-back) (Amplify Partners, Feb 2026)
- [AI will make formal verification go mainstream](https://martin.kleppmann.com/2025/12/08/ai-formal-verification.html) (Kleppmann, Dec 2025)
- AlphaProof (Nature 2025), DeepSeek-Prover-V2, AutoVerus (OOPSLA 2025)
