# SoroGuard


## 🔍 What is SoroGuard?

**SoroGuard** is a **zero-install, Soroban-native vulnerability scanner** that plugs into any CI/CD pipeline with a single line of YAML. It catches the security pitfalls that generic EVM tools miss entirely — because Soroban's storage model, authorization framework, and cross-contract execution semantics are fundamentally different.

We ship as a **pre-built static binary** (no Rust toolchain required), a **GitHub Action**, a **VS Code extension**, and a **community YAML rule registry** so that anyone — regardless of compiler internals knowledge — can contribute new detectors.

> SoroGuard is explicitly **complementary to [CoinFabrik Scout](https://github.com/CoinFabrik/scout)**, not a replacement. Scout is the deep static analysis layer. SoroGuard is the zero-friction CI layer. **Use both for defence in depth.**

<br/>

## 🆚 How SoroGuard fits the ecosystem

| | CoinFabrik Scout | **SoroGuard** |
|---|---|---|
| **Install** | Requires local Rust toolchain | ✅ Zero-install single binary |
| **Rules** | Dylint compiler plugins (Rust) | ✅ YAML pattern matching |
| **Scope** | Multi-chain (Ink!, Soroban, EVM) | ✅ Soroban-only, deep domain |
| **Contributor bar** | Rust compiler internals | ✅ YAML + basic Rust |
| **CI integration** | Manual setup | ✅ 1-line GitHub Action |
| **PR badges** | — | ✅ Native Wave-compatible badges |
| **VS Code** | — | ✅ Inline squigglies as you type |

<br/>

---

## ⚡ Quickstart

### GitHub Action _(recommended)_

Drop this into any repo that contains Soroban contracts:

```yaml
# .github/workflows/soroguard.yml
name: SoroGuard Security Scan

on: [push, pull_request]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: soroguard/action@v1
        with:
          fail-on: high          # critical | high | medium | low
          post-pr-comment: true  # posts a findings summary comment on every PR
          badge: true            # updates the SoroGuard status badge
```

<br/>

### CLI

```bash
# Download binary (Linux x86_64)
curl -sSL https://github.com/soroguard/soroguard/releases/latest/download/soroguard-linux-x86_64 \
  -o soroguard && chmod +x soroguard

# Scan all contracts in a directory
./soroguard scan contracts/

# Output as SARIF (for GitHub Advanced Security code scanning)
./soroguard scan contracts/ --format sarif -o results.sarif

# Scan a single file with verbose output
./soroguard scan contracts/token.rs --verbose

# Filter by severity
./soroguard scan contracts/ --min-severity high
```

<br/>

### VS Code Extension

Install the **SoroGuard** extension from the VS Code Marketplace. Inline squigglies appear on vulnerable patterns as you type, with hover cards explaining each finding and a suggested fix — without leaving your editor.

```
ext install soroguard.soroguard-vscode
```

<br/>

---

## 🛡 Detectors

**20 detectors** across four vulnerability families. Every detector ships with a vulnerable fixture and a fixed fixture in `contracts/fixtures/` that you can run against locally.

<br/>

### 🔐 Access Control

| ID | Description | Severity |
|---|---|---|
| `SG-A001` | Missing `require_auth()` on privileged entrypoint | 🔴 Critical |
| `SG-A002` | Admin key stored in temporary storage (survives 0 ledgers after TTL) | 🔴 Critical |
| `SG-A003` | Role check performed **after** state mutation | 🟠 High |
| `SG-A004` | Missing `require_auth_for_args()` on batch operations | 🟠 High |
| `SG-A005` | Unrestricted contract upgrade entrypoint | 🔴 Critical |

> **Why this matters**: Soroban uses an explicit `require_auth()` model instead of EVM's implicit `msg.sender`. Developers migrating from EVM patterns routinely forget to call it — leaving privileged entrypoints wide open.

<br/>

### 📦 Storage Lifecycle

| ID | Description | Severity |
|---|---|---|
| `SG-C001` | Persistent storage entry missing `extend_ttl()` call | 🔴 Critical |
| `SG-C002` | Balance or critical state stored in temporary storage (~80 ledger TTL) | 🔴 Critical |
| `SG-C003` | TTL extension in hot path — should be conditional | 🟡 Medium |
| `SG-C004` | Unbounded map growth in persistent storage | 🟠 High |
| `SG-C005` | Instance storage used for per-user data (should be persistent) | 🟡 Medium |

> **Why this matters**: Soroban's three-tier storage model (temporary, persistent, instance) has no EVM equivalent. Storing balances in temporary storage or forgetting to extend TTL silently deletes state — no exception, no event, no trace.

<br/>

### ➕ Arithmetic & Overflow

| ID | Description | Severity |
|---|---|---|
| `SG-M001` | Unchecked integer arithmetic — use `checked_add` / `checked_mul` | 🟠 High |
| `SG-M002` | Division before multiplication (precision loss) | 🟡 Medium |
| `SG-M003` | Casting `i128` → `u64` without bounds check | 🟠 High |
| `SG-M004` | Fixed-point scaling constant mismatch (7 vs 8 decimals) | 🟠 High |
| `SG-M005` | Timestamp used as randomness source | 🟡 Medium |

<br/>

### 🔗 Cross-Contract & Events

| ID | Description | Severity |
|---|---|---|
| `SG-E001` | Cross-contract call result ignored (no `?` or match) | 🟠 High |
| `SG-E002` | Unchecked error propagation in cross-contract invocation | 🟠 High |
| `SG-E003` | Missing event emission on state-changing operation | 🟡 Medium |
| `SG-E004` | Hard-coded contract ID (should use env address) | 🟡 Medium |
| `SG-E005` | Auth context not propagated to sub-invocation | 🟠 High |

> **Why this matters**: Soroban's host function call model means errors from sub-invocations don't automatically bubble. An ignored `Result` is a silent failure that leaves state inconsistent.

<br/>

---

## 📋 PR Badge & Comment

When `post-pr-comment: true` is set, SoroGuard posts a structured comment to every PR so reviewers see findings without leaving GitHub:

```
## SoroGuard scan results

✅ 0 critical   ✅ 0 high   ⚠️ 2 medium   ℹ️ 1 low

| Detector | File              | Line | Severity |
|----------|-------------------|------|----------|
| SG-C003  | contracts/token.rs | 47  | Medium   |
| SG-M002  | contracts/amm.rs   | 112 | Medium   |
| SG-E003  | contracts/token.rs | 89  | Low      |

[View full SARIF report →](https://github.com/...)
```

Badge updates are Wave-compatible and render on your repo's README and Drips profile automatically.

<br/>

---

## 🗂 Community Rule Registry

Rules live in `registry/rules/` as **human-readable YAML files**. No Rust compiler internals required — anyone who understands a vulnerability pattern can contribute a new detector.

### Rule Schema

```yaml
id: SG-C001
name: missing-ttl-extension
severity: critical           # critical | high | medium | low | info
family: storage              # auth | storage | math | interop
description: >
  Persistent storage entries expire if TTL is not extended.
  Contracts that write to persistent storage must call
  env.storage().persistent().extend_ttl() to prevent silent data loss.
message: "Persistent storage write without extend_ttl() call detected"
fix_hint: >
  Add `env.storage().persistent().extend_ttl(key, min_ttl, max_ttl);`
  after each set() call, or in a dedicated housekeeping function.
pattern:
  kind: call_expression
  callee_matches: "\.set\("
  context: persistent_storage
  missing_sibling:
    callee_matches: "extend_ttl"
    within_scope: function
references:
  - https://developers.stellar.org/docs/smart-contracts/storage/ttl
test_contracts:
  vulnerable: contracts/fixtures/SG-C001-vulnerable.rs
  fixed:      contracts/fixtures/SG-C001-fixed.rs
```

Every merged rule is automatically bundled into the next binary release.

<br/>

---

## 🏗 Architecture

```
soroguard/
├── src/
│   ├── ast/                    # syn-based AST parsing
│   ├── detectors/              # Built-in Rust detector implementations
│   └── engine/                 # YAML rule engine (pattern matcher)
│
├── registry/
│   └── rules/                  # Community YAML rules (one file per detector)
│
├── contracts/
│   └── fixtures/               # 40 test contracts (vulnerable + fixed per detector)
│
├── vscode-extension/           # VS Code LSP extension (TypeScript)
├── action/                     # GitHub Action entrypoint (shell + binary)
│
└── .github/
    └── workflows/
        └── self-scan.yml       # SoroGuard scanning its own contracts on every push
```

The binary embeds all built-in detectors **and** loads YAML rules from `registry/rules/` at startup, so rule updates ship without recompiling. The VS Code extension runs the same binary in language server mode and surfaces findings as diagnostics.

<br/>

---

## 🧪 Dogfooding

SoroGuard scans its own contracts on every push via `.github/workflows/self-scan.yml`. The `contracts/fixtures/` directory contains intentionally vulnerable contracts — clone the repo and run a scan locally as your first exercise:

```bash
# Should produce a CRITICAL finding
./soroguard scan contracts/fixtures/SG-A001-vulnerable.rs
# → [CRITICAL] SG-A001: Missing require_auth() on fn transfer (line 23)

# Should be clean
./soroguard scan contracts/fixtures/SG-A001-fixed.rs
# → No findings. Clean.

# Run all fixtures as a test suite
./soroguard scan contracts/fixtures/ --format summary
# → 20 vulnerable contracts: 20 findings | 20 fixed contracts: 0 findings ✓
```

<br/>

---

## 🤝 Contributing

SoroGuard is a community-first project. All four contribution tiers produce value — from newcomers to deep experts.

### Tier 1 — Test contract pairs _(good first issue)_

Each detector needs a `vulnerable.rs` and a `fixed.rs` fixture. These are self-contained, independently verifiable by CI, and labelled `good-first-issue` in the tracker.

**What you need:** intermediate Rust + basic Soroban SDK familiarity.

```bash
# File naming convention
contracts/fixtures/SG-XXXX-vulnerable.rs   # must trigger the rule
contracts/fixtures/SG-XXXX-fixed.rs        # must be clean
```

Open issues: [`label:test-contract`](https://github.com/soroguard/soroguard/issues?label=test-contract)

<br/>

### Tier 2 — YAML rule authoring

Some detectors have Rust implementations but no community YAML equivalent. Write the YAML rule so future contributors can extend the pattern without touching Rust.

```bash
# Branch naming
git checkout -b rule/SG-XXXX-short-name
```

<br/>

### Tier 3 — Detector documentation

Write the MDX doc page for each rule: what it catches, why it matters, a vulnerable example, a fixed example, and when to use `#[allow(soroguard::SG-XXXX)]` to suppress.

<br/>

### Tier 4 — Novel detector proposals

Research a Soroban-specific vulnerability class not yet covered. Propose a new `SG-XXXX` with full YAML + test pair. Bonus: implement the Rust detector too.

<br/>

### Contribution workflow

```
1. Fork → branch (rule/SG-XXXX-name or fix/short-description)
2. Add rule YAML  →  registry/rules/SG-XXXX.yaml
3. Add fixtures   →  contracts/fixtures/SG-XXXX-{vulnerable,fixed}.rs
4. Run tests      →  cargo test
5. Open PR        →  self-scan runs automatically, badge updates
```

See [CONTRIBUTING.md](CONTRIBUTING.md) for the complete guide, style conventions, and review SLA.

<br/>

---

## 🌊 Wave 4 Contribution Guide

SoroGuard is participating in **[Drips Wave 4](https://drips.network)**. Every merged contribution earns drip weight proportional to tier and impact.

| Tier | Task | Difficulty | Drip weight |
|---|---|---|---|
| 1 | Test contract pair (1 vulnerable + 1 fixed) | ⭐ Beginner | Base |
| 2 | YAML rule for existing Rust detector | ⭐⭐ Intermediate | 2× |
| 3 | MDX documentation page for a rule | ⭐⭐ Intermediate | 2× |
| 4 | Novel detector proposal + implementation | ⭐⭐⭐ Advanced | 4× |

Issues ready for Wave 4 are tagged `wave-4` in the tracker.

<br/>

---

## 💡 Why Soroban needs its own tooling

Soroban introduces vulnerability classes with **no EVM equivalent**. Generic auditing tools built for Ethereum will not catch these:

### 1. Storage TTL expiry (`SG-C001`, `SG-C002`)
Soroban has three distinct storage tiers:

| Tier | Default TTL | Use case |
|---|---|---|
| **Temporary** | ~80 ledgers (~7 days) | Short-lived session state |
| **Persistent** | Manual extension required | Balances, config, positions |
| **Instance** | Contract lifetime | Admin keys, global params |

Storing balances in temporary storage, or forgetting to call `extend_ttl()` on persistent entries, **silently deletes on-chain state**. No revert. No event. No warning. This is invisible to any EVM-based scanner.

### 2. Authorization model (`SG-A001`)
EVM uses implicit `msg.sender` for access control — every call knows its caller automatically. Soroban uses an **explicit `require_auth()` model** where the developer must invoke it manually. Developers coming from EVM habitually skip this step, leaving privileged functions callable by anyone.

### 3. Cross-contract error propagation (`SG-E002`)
Soroban's host function call model means errors from sub-invocations **do not automatically bubble**. A missing `?` or unmatched `Result` silently discards errors, leaving contract state partially mutated with no signal to the caller.





