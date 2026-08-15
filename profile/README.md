# SoroGuard

**AI-assisted adversarial testing and simulation for Soroban smart contracts.**

SoroGuard helps Soroban developers find logic and authorization bugs by generating and executing realistic, adversarial test scenarios. It reads your contract's exported interface, uses AI to suggest scenarios worth testing, scaffolds them into readable Rust tests, and lets you run one-off simulations from the CLI — including in plain English.

> **Not a static analyzer.** SoroGuard doesn't scan your code for vulnerability patterns — [Scout](https://github.com/CoinFabrik/scout-soroban) already does that well. SoroGuard executes your contract against real scenarios to see how it actually behaves. Use both — they cover different parts of the security workflow.

---

## Why

Soroban's SDK gives you solid testing primitives (`testutils`, local sandbox mode), but you still decide what to test and write the setup for every scenario by hand. Auth checks, edge cases, boundary values — it's all boilerplate that's easy to skip under deadline pressure, and skipping it is how bugs make it to mainnet.

SoroGuard closes that gap two ways: deterministic scaffolding for the failure modes every contract shares, and an AI layer that looks at *your specific contract* and suggests what else is worth testing — things a generic template wouldn't think to check.

Importantly: **AI suggests what to test. It never decides what "secure" means.** Every suggested scenario is converted into a real test that actually runs against your contract — the result is determined by execution, not by a model's opinion.

---

## Status

SoroGuard is early-stage.

- **Deterministic scaffolding and CLI simulation** — stable, in daily use
- **AI-assisted features** (scenario suggestions, natural-language simulation, failure explanations) — **actively in development.** The interfaces below reflect the intended design; expect rough edges and rapid iteration.

The core tool works fully with AI disabled — it's an added layer, not a dependency.

## Features

### Deterministic core

- **Auto-scaffolded negative-path tests** — parses your contract's interface and generates `testutils`-based Rust tests for common failure modes per function (unauthorized caller, missing `require_auth`, wrong signer, invalid/boundary inputs)
- **CLI simulation** — run one-off calls against a local sandbox without writing a test file:
  ```bash
  soroguard sim transfer --as unauthorized_user
  soroguard sim withdraw --amount 999999999
  ```

### AI-assisted (in development)

- **Scenario generation** — SoroGuard analyzes your contract's functions and suggests scenarios specific to it, not just generic templates:
  ```text
  $ soroguard generate ./contracts/token --ai

  Analyzing contract...
  12 exported functions found. 5 involve authorization.

  Suggested scenarios:
  ✓ Unauthorized mint
  ✓ Transfer after authorization is revoked
  ✓ Burn exceeding balance
  ✓ Transfer with maximum amount (overflow check)
  ```
  Suggestions are turned into real, executable tests — not left as text.

- **Natural-language simulation** — describe a scenario, SoroGuard runs it:
  ```bash
  soroguard ask "Try to transfer 1000 tokens from an account that hasn't authorized it"
  ```

- **Failure explanations** — plain-language read on why a scenario failed:
  ```text
  Scenario: Unauthorized transfer
  Expected: Authorization failure
  Observed: Transaction succeeded

  Suggested area to investigate:
  Verify the transfer operation requires authorization from
  the account whose balance is being modified.
  ```
  Explanations are a starting point for investigation, not a verdict — always confirm against the actual test output.

## Installation

```bash
cargo install soroguard
```

Requires a Rust environment set up for Soroban development (Rust toolchain + Soroban CLI). AI features require configuring a supported provider — see `docs/ai.md` (coming soon).

## Quick Start

```bash
# Generate a baseline test suite from your contract's interface
soroguard scaffold ./contracts/my_contract
cargo test

# Add AI-suggested scenarios on top
soroguard generate ./contracts/my_contract --ai

# Simulate a specific scenario without writing a test
soroguard sim my_function --as some_address --arg amount=1000

# Or just ask
soroguard ask "What happens if I withdraw more than the account holds?"
```

## How It Works

```text
              Soroban Contract
                     │
                     ▼
           Contract Interface
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
    Deterministic          AI-assisted
     scenarios              scenarios
          │                     │
          └──────────┬──────────┘
                     ▼
              Test Generation
                     │
                     ▼
        Soroban Test Environment
                     │
                     ▼
              Actual Results
                     │
             ┌───────┴───────┐
             ▼               ▼
         Rust Tests     AI Explanation
                        (optional)
```

1. SoroGuard parses your contract's exported functions and interface
2. Deterministic rules generate baseline negative-path scenarios; the AI layer (when enabled) suggests contract-specific ones on top
3. All scenarios — deterministic or AI-suggested — become real Rust tests or CLI simulations
4. Tests run against the local Soroban sandbox; SoroGuard records the actual behavior
5. AI can optionally explain failures in plain language
6. Generated tests are ordinary, readable, editable Rust — never a black box

## Project Structure

```text
soroguard/
├── crates/
│   ├── cli/          # Command-line interface
│   ├── core/          # Test/scenario orchestration
│   ├── parser/        # Soroban contract interface parsing
│   ├── generator/      # Rust test scaffolding
│   ├── simulator/      # Local sandbox execution
│   └── ai/            # AI-assisted scenario generation & explanations (in development)
│
├── examples/
│   └── token/         # Example contract used for development/testing
│
├── tests/
│   └── integration/     # End-to-end SoroGuard tests
│
├── Cargo.toml
├── README.md
└── LICENSE
```

Crates are split so contributors can work on one part of the pipeline — parsing, generation, simulation, or AI — without needing to understand the whole codebase.

## SoroGuard + Scout

| Tool | Focus |
|---|---|
| Scout | Static vulnerability detection |
| SoroGuard | AI-assisted dynamic testing |

Scout flags dangerous code patterns without running the contract. SoroGuard executes it against generated scenarios — deterministic and AI-suggested — to observe actual behavior. Using both gives broader coverage than either alone.

## What SoroGuard Does Not Do

SoroGuard doesn't claim to prove a contract is secure, and doesn't replace manual code review, static analysis, formal verification, or a security audit. AI-generated scenarios and explanations can be incomplete or wrong — they're a testing aid, not authoritative security advice. Treat them as a starting point, and always trust the actual test execution over the AI's commentary.

## Roadmap

**Testing**
- [ ] Auth scenario matrix — generate every "who can call what" combination and test each
- [ ] Property-based fuzzing — randomized inputs checked against invariants (balance never negative, supply conserved)
- [ ] Cross-contract scenario testing — mock a second contract, test call-in/call-out behavior
- [ ] Gas/resource regression tracking
- [ ] Snapshot/diff testing across contract upgrades

**AI**
- [ ] AI-assisted invariant suggestions (properties that should always hold)
- [ ] Context-aware generation from contract docs/comments
- [ ] AI-assisted test prioritization

**Developer experience**
- [ ] CI/CD integration (GitHub Action)
- [ ] VS Code extension

## Contributing

This is early and the roadmap is open. Good first contributions: new negative-path scenario types, additional example contracts, or picking up any unchecked roadmap item — deterministic or AI-side. Check open issues or start a discussion before taking on something larger.

## License

MIT
