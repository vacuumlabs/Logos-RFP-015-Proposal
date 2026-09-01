# \[PROPOSAL\] RFP-015 — `Synarton`

---

## RFP ID

**RFP-015 — Token Launchpad: Bonding Curve** ([RFP text](https://github.com/logos-co/rfp/blob/master/RFPs/RFP-015-bonding-curve-launchpad.md))

## Your Project Name

`Synarton`

## Team or Organization Name

Vacuumlabs

## Primary Contact

Peter Hucík - peter.hucik@vacuumlabs.com

## Team Members

| \# | Name / pseudonym | Social links | Role | Status |
| :---- | :---- | :---- | :---- | :---- |
| 1 | Uroš Kočišević | [GitHub](https://github.com/kocisevic) · [CV](https://people.vacuumlabs.com/cv/ec6323f2737c840999667d42c7ca29630e99c942a12a5da2e91df295f3d55d0a) | Chief Product Owner | Full-time |
| 2 | Boris Hristov | [GitHub](https://github.com/Soulrealz) · [CV](https://people.vacuumlabs.com/cv/7ca3b02f0626c586367860c6d1cb80811fee74b3162c95f671cdcd8b56cc7ab8) | Tech lead / Senior Full-stack Engineer | Full-time |
| 3 | Goktug Gurbuzturk | [GitHub](https://github.com/goktug-gurbuzturk-wp) · [CV](https://people.vacuumlabs.com/cv/e2ddb12995f434c103c7cb022129c57f87eeab6637d8c2b58540936cd58b19da) | Senior Full-stack Engineer | Full-time |
| 4 | Ladislav Dubravsky | [Github](https://github.com/ladislavdubravsky) · [CV](https://people.vacuumlabs.com/cv/fb3e538365d3272e51fe8ecbc5de6ca209c0c5e260ef73a4a3909edec98e4aa8) | Senior Full-stack Engineer | Full-time |
| 5 | Marek Roštár | [GitHub](https://github.com/RostarMarek) · [CV](https://people.vacuumlabs.com/cv/9c5267a6e45103aa6eb1b05606b5d86701336170e3d3f6209dc6f05e958d397f) | Project Manager / Utility Dev | Full-time |
| 6 | 0xcr1st0f | [CV](https://github.com/cr1st0f/cv/blob/main/cr1st0f_CV_updated.pdf) | Advisor | Part-time |

---

## Project Summary

We propose to build a **privacy-preserving bonding-curve token launchpad on the Logos Execution Zone (LEZ)**: a constant-product AMM with virtual reserves, in the pump.fun lineage the RFP names as its model, with a deterministic supply-driven price trajectory that is fully computable before a sale opens. Creators configure a sale from a token pair, a sale quantity `D`, an optional DEX seed `R`, and the virtual reserves that shape the curve; participants buy and sell on that curve from a public account directly, or privately through the deshield→trade→re-shield pattern, with a mini-app, an SDK, a CLI and sale analytics on top.

**Two things distinguish this proposal, and both are verifiable before you award anything.**

**First, the pricing core already exists in Logos's own repository.** RFP-015's Reference Implementation and `lez-programs/programs/amm/core/src/lib.rs` are algebraically identical — we checked term for term, not by inspection — including the rounding directions F1 mandates, the fee lattice, and the exact-output inverse F1 requires the SDK to expose. **The fee arithmetic in particular is exercised rather than dormant:** `fees` is a parameter of the pool-creation instruction itself (`amm/methods/guest/src/bin/amm.rs`), `assert_supported_fee_tier` gates it on four instructions (`swap`, `add`, `remove`, `sync`), the supported set is `[1, 5, 30, 100]` bps so **a pool cannot be created fee-free**, and the stored rate is passed into `swap_exact_in_amounts` and `swap_exact_out_amounts` on every trade. `price_impact_bps` and `spot_price_q64_64`, which U4 needs, are already written and tested. The curve is multiplication and division on `u128` throughout: no fixed-point exponentiation, no approximation, no convergence argument to audit. We are porting a proven kernel, not deriving a new one.

**Second, the specification contains a defect that makes the sale unclosable, and we can prove it arithmetically.** The supply target is unreachable on integer arithmetic: no integer collateral input yields exactly `D` tokens, because the output lattice near exhaustion is ~2,434 tokens wide and `D` falls strictly inside the gap. This is measured, not argued from one example: across 400 randomly generated parameter sets in the shape the RFP names as its model, **none is closable**. F4's auto-close therefore never fires, **hard requirement R3 is unsatisfiable**, F5 leaves the entire raised collateral permanently locked, and **two of the five test cases S3 explicitly mandates are tests of behaviour the specification cannot produce.** The remedy is a partial-fill clamp built on Logos's own `swap_exact_out_amounts` — whose "the pool never comes up short" rounding is exactly what F1 asks for, and whose failure modes are all excluded — one by F2's own `Vt > D` constraint, the other two by our creation-time validation envelope. F1's "must revert" forbids it, so we request that deviation explicitly rather than shipping a silent divergence.

We found eight defects in total. Four are High or Critical: the unreachable supply target above; a fee that the specification credits to the reserve *and* pays to the treasury, leaving the books 1.0101% ahead of the vault at a 1% rate; **the same fee ambiguity unresolved on the sell side**, where the losing reading drains the collateral vault before a sold-out position can be sold back; and an F5 withdrawal that hands the creator the entire remaining collateral reserve at close while auto-graduation to a DEX is only a *soft* requirement — leaving token holders with no backing, no sell path and no DEX. We set all eight out below with their resolutions. **Where the specification is ambiguous we state the reading we take, the decision that follows and the reasoning behind it** — twelve such readings, each of them a decision Logos can review rather than a question Logos has to answer.

We are equally explicit about what argues against this mechanism. **A bonding curve structurally penalises the privacy path**: price rises with every buy, a private buy is three transactions, and the buyer pays the impact of whatever lands in the gap — 6.19% worse fill with a single buy ahead at sale open. No implementation can remove that. We quantify it, mitigate it, and disclose it to users rather than leaving it for an audit to find.

This proposal delivers the program, SDK, CLI, mini-app, analytics, tests and documentation to LEZ testnet 0.2 in **12 weeks across three engineering streams**, with testnet 0.3 and mainnet as separately-scoped milestones gated on platform availability, since neither has a published date.

---

## Technical Approach

### 3.1 Architecture

```
on-chain (Rust / RISC0 zkVM, built with SPEL)
  └── bonding curve program (immutable)                                ← F1–F7
        state (PDAs): Sale · sale reserve D / DEX seed reserve R buckets
                      · program-owned collateral + token vaults
                      · Config (fee rate, treasury, admin authority)
        pricing: constant product with virtual reserves (Vt, Vc, k)
                 ported from amm_core, + partial-fill clamp (DEV-1)
```

The client side is deliberately shaped so that every surface the RFP requires is a thin consumer of
a single Rust core module. This is not a stylistic preference: U1
(SDK), U2 (mini-app), U3 (CLI) and U8 (analytics) all require the *same* pricing, quoting, saga and
error logic, and the RFP explicitly permits the CLI to be a feature subset of the GUI. Duplicating
that logic across three surfaces is the standard way these deliverables drift.

```
client stack — ONE core module is the single source of truth
  ├── launchpad core module (Rust, FFI, Qt-free)              ← U1
  │     ├── curve math: quotes, price impact, pre-trade summary   ← U4
  │     ├── saga engine: public path + deshield→trade→re-shield   ← U1, U6, U7
  │     │     (buy saga AND sell saga — F3, §3.5.36)
  │     ├── error taxonomy (one enum, one message table)          ← U10
  │     └── chain observer / analytics index                      ← U8
  │           IDL-decoded calldata + per-block account reads
  ├── mini-app GUI (QML + C++, Basecamp-loadable via git repo)     ← U2, U4, U5, U6, U7, U8
  ├── CLI (SPEL-generated + thin wrapper)                          ← U3
  └── SPEL IDL (generated from the program definition)             ← U9
```

**C8 — the IDL is frozen first, and it is the contract that lets three streams work at once.**
U9 requires a SPEL IDL anyway. We generate and freeze it at the end of the program's first
milestone, then build the SDK, CLI, GUI and analytics decoder against it while the on-chain program
is still being hardened. This is the single most important scheduling decision on the client side
and it is what keeps the client streams from serialising behind the program (see Milestones, M1).

### 3.2 Stack

- **On-chain:** Rust compiled to the **RISC0 zkVM**, built with **SPEL** — one program definition generates the IDL (U9), CLI (U3), Rust SDK (U1) and a QML/C++ Basecamp module (U2). Workspace mirrors [`lez-programs`](https://github.com/logos-blockchain/lez-programs)' layered `core`/`program`/`methods/guest` pattern.
- **Client:** a single Qt-free Rust **core module** consumed identically by SDK, CLI, mini-app and the analytics observer; **QML + C++** mini-app loadable in Basecamp from a git repository.
- **Custody:** program-owned **PDA vaults** with PDA-seed-authorized `Token::Transfer`, reusing the proven `lez-programs` idiom — no per-vault private key. ATAs throughout (F7, LP-0014).
- **Arithmetic:** `u128` with U256 intermediates, reusing `amm_core`'s `mul_div_floor` / `mul_div_ceil` verbatim so the rounding direction F1 mandates is inherited rather than re-implemented.
- **Testing:** `proptest` property suites over generated operation sequences for the invariants, plus **two e2e lanes in CI** — an in-process harness and a **standalone-mode sequencer** job (S2) — with CI cloned from `lez-programs` and failing the build on red.
- **Licence:** dual **MIT + Apache-2.0**, as the RFP requires.

### 3.3 Platform dependencies & integration status

Verified against local clones of the live public Logos repositories: `logos-execution-zone` @ `9a7a71a` (2026-08-19) and its release tags `v0.2.1`–`v0.2.4`, `lez-programs` @ `72a3e74` (2026-08-17), `roadmap` @ `77a9418`. **Three entries in the RFP's own Platform Dependencies section do not match the code** — two are stale in opposite directions, and the third names a primitive the prize it cites never delivered. We flag all three, because anyone scoping this RFP will read them as current.

| Dependency | RFP says | Ground truth | Effect on this plan |
| :---- | :---- | :---- | :---- |
| **LP-0013** token authorities | Hard blocker, "currently open", required for the **"transfer-authority primitives"** that custody the sale reserve and the real collateral reserve | **Mis-specified, and closed.** LP-0013 delivered a *mint* authority model — `mint_authority: Option<AccountId>`, `MintWithAuthority`, `SetAuthority`, `SetAuthorityWithAuthority`, all in `lez-programs`' `programs/token/core/src/lib.rs`, prize `[CLOSED]`. **No transfer authority exists anywhere:** `transfer_authority`, `delegate` and `approve` return **0 hits** across `programs/token/`, and the word "transfer" does not occur in the prize text. The RFP's own Resources list titles LP-0013 *"mint authorities"*, contradicting its dependency block | **Not a blocker, and never was.** Custody needs no authority primitive — see the F7 walk (§3.5.11) |
| **LP-0014** ATAs | Listed only under Resources | **Delivered.** Full `programs/ata/` — create/transfer/burn, IDL, private-account ATA creation | Satisfies F7 directly |
| **LP-0015** cross-program calls | "Resolved" | **Partially.** The `ChainedCall` machinery ships and the roadmap milestone is checked, but the prize's own success criteria are unchecked and its distinctive deliverables (entrypoint declaration, call-capability type, `call_program!` macro, docs) are absent upstream | Enough to build on — our buy is 3 chained executions of the 11 available |
| **LP-0012** structured events | **"Resolved"** | **NOT delivered upstream.** `emit_event` → **0 hits**; `TxReceipt` → **0 hits**; the roadmap's `lez_events.md` is a **testnet-0.3 milestone with all 3 items unchecked**. The prize is `[CLOSED]` and was genuinely won — but the code lives in a **contributor fork**, not upstream | **We do not depend on it.** U8 analytics run on the indexer + `getAccountAtBlock` + intra-block replay (C9), with a documented migration when events land |
| **RFP-001** admin authority | Hard blocker, "in development" | **Not delivered.** M3 (final milestone) still open; the enabling PR `spel#233` (opened 2026-06-24) was **closed unmerged on 2026-08-19**; no admin-authority code in `spel` | **We build the admin authority inline**, exactly as the AMM did in `update_config.rs`, and migrate if RFP-001 lands |
| **RFP-004** DEX | Soft dependency (auto-graduation) | Awarded; `lez-programs/programs/amm` exists | Auto-graduation is soft; the collateral-destination parameter (C5) is designed so graduation can be wired in without redesign |
| **LEZ clock** | "Resolved" | **Delivered.** Two caveats: **no monotonicity guard** (the sequencer stamps wall-clock time with no check that it exceeds the parent's), and a **one-block lag** (the clock tx is applied after the user-tx loop, so user transactions in block N read block N−1's values) | **Pricing is time-independent** — spot price is pure state, `Vc/Vt` — but we take the optional end timestamp (C26), so the close path does depend on the clock. Both caveats are bounded there rather than assumed away: the floor is a duration in days, where a one-block lag is noise and a non-monotonic stamp can move a close by at most one block |

**Nothing in the 12-week programme waits on an unshipped platform capability.** That is a deliberate design outcome, not luck: the one requirement that would otherwise have depended on undelivered work — U8 analytics — was re-architected to remove the dependency (C9).

### 3.4 Scope and boundaries

**In scope:** the immutable bonding-curve program (create, buy, sell, auto-close, constrained manual close, withdraw) with virtual reserves, two accounting buckets, the partial-fill clamp and per-swap fee routing; the client core module, SDK, CLI and Basecamp mini-app; the public and private trade paths including private sells; the creator-configurable one-directional mode that C22 relies on; the optional end timestamp with the mandatory minimum-duration check it carries; sale analytics independent of LP-0012; the parameter-validation envelope; tests, benchmarks, README and the privacy/anonymisation document; deployment to testnet 0.2, with testnet 0.3 and mainnet as gated milestones.

**Soft requirements — our position on all three**, since a soft requirement with no stated position is
the same problem as an ambiguity with no stated reading:

| Soft requirement | Position | Why |
| :---- | :---- | :---- |
| **One-directional mode** | **Implemented** (M2) | C22 relies on it as a D-7 mitigation: disabling sell-back removes the reflexive volume that can land in a private buyer's gap. One flag plus one check on the sell path |
| **Optional end timestamp** | **Implemented** (M2, C26) | It carries the RFP's own minimum-duration floor, which is the only mechanism the specification offers against the D-7 price penalty we quantify (§3.5.35). It also bounds the >98% of curves that never reach the supply target, alongside the constrained manual close (C24). Cost: it is the one feature that makes the program depend on the clock (§3.3) |
| **Auto-graduation to DEX** | **Not implemented** — OOS-1 | Execution depends on RFP-004, which is still open. The collateral destination is a creation-time parameter (C5) so graduation can be wired in without redesign |

**Out of scope**, each verified in PR review rather than merely asserted:

| \# | Out of scope | Verification it stayed out |
| :---- | :---- | :---- |
| OOS-1 | Auto-graduation execution into a DEX pool (soft requirement; depends on RFP-004) | No DEX-integration code. The collateral destination is a parameter defaulting to literal F5 precisely because graduation execution is out of scope; the escrow setting holds collateral in the program rather than deploying it |
| OOS-2 | Governance (token / DAO / multisig) | No governance code in the repository |
| OOS-3 | An alternative pricing mechanism under the RFP's deviation standard | Constant-product with virtual reserves only |
| OOS-4 | Admin custody and key-rotation policy | Admin is a single configurable testnet handle |
| OOS-5 | Bot mitigation / access restrictions at the protocol level | None implemented — the RFP states no ecosystem platform does, and directs projects needing it to RFP-016 |
| OOS-6 | Audit fees inside the fixed price | The RFP body states no audit requirement, but the programme's proposal template directs bids to include an audit milestone where the project *"handles user funds"* or *"implements cryptographic primitives"* — this does both. **M9 runs an external tier-1 audit outside the build envelope**, billed at cost; our remediation is quoted when the firm is engaged, so neither sits inside the figure below |
| OOS-7 | Off-chain indexer as a hosted service | The observer is a client-side library, not a subgraph or hosted API |
| OOS-8 | Production fee-rate and treasury policy | Testnet values only; the rate is admin-configured at deployment |

### 3.5 Core mechanics, requirement by requirement


#### 3.5.1 The curve core — what exists, and what is net-new

Two questions decide how much of this RFP is genuinely new work: how much of the curve Logos already
owns, and what the remainder costs. The next two subsections answer them in that order.

#### 3.5.2 The pricing math is already in Logos's own repository

RFP-015's Reference Implementation and `lez-programs/programs/amm/core/src/lib.rs` are not merely
similar — the **pricing** is **algebraically identical**, including the fee lattice on the buy side.
The **fee is not symmetric**, and that is the one place naive reuse goes wrong: the kernel's `fee_bps`
always comes off the *input*, while the RFP charges the sell fee on the *output*, so the sell path calls
the kernel fee-free and applies the fee to `C_out_raw` afterwards (§3.5.6). We verified all of this
term-for-term rather than by inspection:

| RFP-015 Reference Implementation | `amm_core` function | Identity |
| :---- | :---- | :---- |
| `tokens_out = Vt − k/(Vc + C_in)` | `swap_exact_in_amounts` | `Vt − Vt·Vc/(Vc+C) = Vt·C/(Vc+C)` — the same value |
| `C_in = k/(Vt − Q) − Vc` (F1's mandated SDK inverse) | `swap_exact_out_amounts` | `k/(Vt−Q) − Vc = Vc·Q/(Vt−Q)` — the same value |
| `C_out = Vc − k/(Vt + tokens_in)` | `swap_exact_in_amounts`, reserves swapped, **`fee_bps = 0`** | the *pricing* is the same by symmetry; the fee is not (row below) |
| F1 "round against the trader" | out floors (`mul_div_floor`); in ceils (`mul_div_ceil`), documented *"so the pool never comes up short"* | rounding direction already correct |
| **Buy** fee on the input, rounded up | `eff = mul_div_floor(C_in, DEN − fee_bps, DEN)` ⇒ the fee itself rounds up | already correct |
| **Sell** fee on the **output**, rounded up | no kernel equivalent — `fee_bps` is input-side only | **we apply it ourselves**: call the kernel with `fee_bps = 0` for `C_out_raw`, then `fee = ceil(C_out_raw × fee_rate)` |

**Why that last row is worth a sentence rather than a footnote.** Passing `fee_bps` to the kernel on a
sell looks like the obvious reuse and quietly breaks two things: `Vt` would rise by only `99%` of
`tokens_in` while the vault received all of it, leaving **1.0000% of every sell's tokens in neither the
reserve nor the treasury** — the token-side mirror of D-5 — and no collateral fee would be produced at
all, so sells would pay the treasury nothing. On a 30,000,000-token sell at 100 bps that is
300,000,000,000 units unaccounted for. The correct call is one line different, and it is the second of
exactly two places where this kernel's obvious reuse is the wrong reuse (the other is
`price_impact_bps`, §3.5.14).

So the safety-critical arithmetic of this RFP is code Logos already owns, already tests, and has
already shipped in a production pool program. We are porting a proven kernel with the rounding
semantics F1 demands, not deriving a new one. Four further primitives that the requirements name
directly are also already present and reused verbatim: **`spot_price_q64_64`** (U4's spot-price line),
**`price_impact_bps`** (execution slippage — adjacent to U4's price-impact line rather than equal to
it, §3.5.14), `mul_div_floor` / `mul_div_ceil`, and PDA vault custody via `compute_vault_pda` — the
same program-owned-vault idiom, with no per-vault private key.

This is the single largest risk reduction available on RFP-015, and it is why the mechanism carries no
fixed-point-exponentiation risk: **the entire curve is multiplication and division on `u128` with U256
intermediates** — no `pow`, no approximation, no convergence argument to audit.

#### 3.5.3 What is genuinely net-new

Honest accounting of what the AMM does *not* give us:

| Net-new | Why the AMM does not cover it | Size |
| :---- | :---- | :---- |
| **Virtual reserves** (`Vt`, `Vc` init, `k = Vt·Vc` recorded at creation but never an operand, `p₀ = Vc/Vt`) | `grep -i virtual programs/amm` returns **zero hits** — the AMM is a real-reserve pool throughout | The largest single piece; arithmetic, not a new primitive |
| **Two accounting buckets** — sale reserve (starts at `D`, decreases) and DEX seed reserve (starts at `R`, untouched until close) | No analogue; an AMM pool has one reserve per side | Small, but it is where F2/F4/F5 all land |
| **Supply-target auto-close** (F4) | No lifecycle in the AMM — pools do not close | Small, and gated on DEV-1 below |
| **Fee routing to a treasury** | The AMM never moves the fee. `finalize_swap` credits the pool `reserve_a += deposit_a` with the **full** `amount_in`; the fee simply stays in the vault and accrues to LPs. `grep -i treasury programs/` returns **zero hits** — there is no treasury account anywhere in the repository | +1 account, +2 config fields, and a sweep instruction — and it is the exact point where the RFP contradicts itself (D-5) |
| **The D-1 partial-fill clamp** | Needs `swap_exact_out_amounts` in a role the AMM never uses it in | Small; see §3.5.5 |

#### 3.5.4 Execution budget — the public buy costs 3 of 11

A public buy costs **3 of the 11 executions** a transaction gets — top-level `buy`, collateral in,
tokens out. The protocol fee does not add a fourth: it is **accrued in sale state on the buy and
swept to the treasury by a separate instruction**, for a runtime reason set out in §3.5.6 rather
than to save an execution. The guard is **not uniform**: public execution allows 11, the PPE prover
10. The private path is **three separate transactions**, each with its own budget, not one summing
to 6. Full figures and the per-operation table are in §3.5.22 and §3.5.24.

This matters because the invocation budget, not cycles, is the scarce resource in composed LEZ
flows — and it is what makes the D-1 remedy cheap: the clamp adds arithmetic inside an existing
invocation, not a new one.

#### 3.5.5 F1 pricing core

**This is the finding we lead with, because it is fatal, it is arithmetic rather than opinion, and
its remedy is already written in Logos's own code.**

##### The defect (D-1, Critical)

F1 requires that *"if the computed `tokens_out` on a buy would exceed the remaining sale reserve,
the transaction must revert."* F4 requires that the sale *"closes automatically when the sale
reserve is exhausted (all `D` tokens have been sold)."* Taken together, closing the sale requires
some buy to land on **exactly** the remaining reserve. On integer arithmetic, no such buy generally
exists.

Using canonical pump.fun-shaped parameters — `Vt = 1,073,000,191e6`, `Vc = 30e9`,
`D = 793,100,000e6`, fee 100 bps — the effective collateral input that would yield exactly `D`
tokens is **85,005,301,050.330473**, which is not an integer. The two adjacent integers give:

```
eff = 85,005,301,050   →  tokens_out = D −   805    undershoots: 805 units of dust remain unsold
eff = 85,005,301,051   →  tokens_out = D + 1,629    exceeds the reserve → F1 mandates a revert
```

One unit of effective collateral moves `tokens_out` by roughly **2,434 units** at the boundary. The
output lattice is ~2,434× coarser than the input lattice, and `D` falls strictly inside the gap —
805 short on one side, 1,629 over on the other. **1,000,001 integer values of `eff` around the
boundary produce zero hits**, and 1,000,000 integer `C_in` values driven through the fee lattice
(`eff = floor(C_in × (DEN − fee)/DEN)`) produce none either.

**And it is not a property of these parameters, which we establish by measurement rather than by
assertion.** Across **400 randomly generated parameter sets** in the shape the RFP names as its model
— pump.fun-style supply, collateral and virtual-reserve ratios — **none is closable: 0.00%**, with a
median boundary step of 1,590 tokens per unit of effective collateral. Simulating actual demand
rather than sweeping the boundary gives the same answer: over **300 sales driven by randomised buy
sequences, none reached exact exhaustion.** The mechanism is structural — `dQ/d(eff) = k/(Vc + eff)²`,
so near exhaustion the output lattice is three orders of magnitude coarser than the input lattice and
`D` has to be struck by coincidence.

**The consequence chain is what makes this critical, and it terminates in locked funds:**

1. `tokens_out == remaining` is unreachable, so **F4's auto-close never fires**.
2. F4 is the only close condition in the hard requirements — the end timestamp is a *soft*
   requirement, and its own text concedes that "over 98% of bonding curves never reach their supply
   target."
3. **R3 — a hard Reliability requirement — is therefore unsatisfiable.** It requires the close to
   be atomic within the exhausting transaction; the exhausting transaction cannot occur.
4. **F5 gates creator withdrawal on the sale being closed.** So the entire raised collateral is
   permanently locked, on every sale, including the successful ones.

We also note the knock-on for S3: two of its five mandated test cases — "auto-close on supply
target" and "manual close" — test behaviour the specification as written does not support (the
first by D-1, the second by D-3, §3.5.9).

##### Why the boundary cannot be closed from the client

The boundary has been observed before, and the published mitigation is an SDK-side cap: near the
end of a sale, use F1's inverse formula `C_in = k/(Vt − Q) − Vc` with `Q` set to the tokens
remaining, so the buy is sized never to overshoot and never to revert. We want to be precise and
fair about why that does not close the sale, because it is a reasonable first instinct and it is
one line of arithmetic away from being right.

Setting `Q = D` on the parameters above, the exact required effective input is
**85,005,301,050.330473** — the same non-integer. The client must therefore round it, and both
directions fail:

| Rounding of the inverse | `eff` | `tokens_out − D` | Outcome |
| :---- | :---- | :---- | :---- |
| **Ceil** — what F1 mandates (`C_in` rounds up) | 85,005,301,051 | **+1,629** | Exceeds the reserve → F1 forces the revert the cap was meant to avoid |
| Floor | 85,005,301,050 | **−805** | Undershoots → 805 units of dust remain → F4 still never fires |

The reason is structural, not a rounding-policy choice: a client can only choose an **input**, and
every representable input maps to an output on a ~2,434-unit lattice that `D` does not sit on. **No
client-side sizing can close the sale, because the target is not in the image of the function.**
The fix has to be on-chain, in what the program does when the requested output exceeds what
remains.

##### Our remedy — a partial-fill clamp, using Logos's own exact-output function (DEV-1)

When a buy's computed `tokens_out` would exceed the remaining sale reserve, the program **fills
exactly the remainder** instead of reverting:

1. Clamp the output to `remaining` (the sale-reserve bucket balance).
2. Compute the collateral actually required for that output with
   `amm_core::swap_exact_out_amounts(remaining, Vc, Vt, fee_bps)`, which solves the constant
   product for the input and lifts it back through the fee, **rounding up at both steps "so the
   pool never comes up short"** — the AMM's own documented guarantee, and exactly the direction F1
   demands.
3. Charge the buyer only that amount, refund the unused collateral in the same transaction, apply
   the per-swap fee on the charged amount, and trigger F4's close atomically in the same execution.

Three properties make this a remedy rather than a workaround:

- **It is total on this program's parameter domain.** `swap_exact_out_amounts` has exactly three
  `None` branches — `amount_out >= reserve_out`, `reserve_in == 0`, and `fee_bps >= FEE_DENOM` — and
  our creation-time envelope excludes all three. Called as
  `swap_exact_out_amounts(remaining, Vc, Vt, fee_bps)`, `amount_out` is the remaining sale reserve, at
  most `D`, against `reserve_out = Vt`, and **F2 already requires `Vt > D`**; `reserve_in` is `Vc`,
  which we validate strictly positive; and the fee rate is validated strictly below the denominator
  (§3.5.7). So the clamp cannot fail for any reason the function is allowed to fail, on every sale, at
  every point on the curve — the first branch by the RFP's own constraint, the other two by ours.
- **It preserves every rounding guarantee F1 asks for.** Both steps round up against the trader; the
  pool is never short.
- **It makes `tokens_out == remaining` reachable by construction** — restoring F4 and R3 and
  unlocking F5, the whole chain from one change.

**DEV-1 — requested deviation from F1.** F1's *"the transaction must revert"* forbids the clamp, so
we request it explicitly rather than ship a silent divergence. Scope: must-revert is retained
everywhere except the single terminal buy that exhausts the sale reserve, where a partial fill with
refund replaces it. Every other overshoot condition — slippage (F6), insufficient balance, sale
closed — still reverts. If Logos prefers F1 literal, the alternative is to accept that the supply
target is decorative and that closing needs a second mechanism: the end-timestamp soft requirement
promoted to hard, with creator withdrawal re-gated on it. **We implement the clamp.**
F1's must-revert and F4's supply-target close are jointly unsatisfiable on integer
arithmetic, so the clamp is the only resolution that leaves F4, R3 and F5 all deliverable. The
alternative is not a variant of the same build — it abandons the supply-target close and re-gates
withdrawal on a soft requirement — so it is M2's first scope item rather than a late discovery. No
competing proposal requests this deviation; those that reach the boundary at all either schedule
"revert-on-overshoot", shipping the defect, or propose the client-side cap refuted above.

#### 3.5.6 F1 collateral location

##### The contradiction (D-5, High)

Three statements in the RFP disagree about what enters the pool on a buy:

| Source | Says |
| :---- | :---- |
| Fee structure (Design Rationale) | The program deducts `fee = C_in × fee_rate` **before applying the pricing formula**; the effective input is `C_in − fee`; the fee "is transferred atomically to a protocol treasury account" |
| **F1** (hard requirement) | "the real collateral reserve **increases by `C_in`**" |
| Reference Implementation | "`Vc` **increases by `C_in`**" |

The fee cannot both leave to the treasury and be credited to the reserve. Simulated over 200
successive 5-SOL buys on the parameters above at a 1% fee:

| Reading | Booked reserve | Actual vault | Treasury | Shortfall | `k` drift |
| :---- | ---: | ---: | ---: | ---: | ---: |
| **F1 / Reference Impl** (`Vc += C_in`) | 1,000,000,000,000 | 990,000,000,000 | 10,000,000,000 | **10,000,000,000 — 1.0101% of the raise** | 3.5188e+00 % |
| Fee structure (`Vc += eff`) | 990,000,000,000 | 990,000,000,000 | 10,000,000,000 | **0** | 1.6533e-10 % |

Implementing F1 literally produces a booked collateral reserve that exceeds the vault balance by
exactly the cumulative fee — 1.0101% of the raise at a 1% rate, scaling linearly with the rate.
**F5 then lets the creator withdraw "the full real collateral reserve", which is by that margin
unbacked**: the withdrawal either underflows or fails, and the failure surfaces at the worst
possible moment, on a successful sale at close.

##### What we implement (DEV-2)

**Pricing and the reserve both move by `eff = C_in − fee`.** The buyer pays the gross `C_in` into
the collateral vault; the curve books `eff`; the fee is **never credited to the reserve**. This is
the fee-structure reading, it is the only one that is solvent, and it is what the sane
implementations in this space do — but it contradicts F1 and the Reference Implementation as
written, so we state it as a deviation rather than implementing it quietly.

**Where the fee sits between the buy and the treasury, and why it is not moved in the buy.** The
runtime settles this for us. A LEZ transaction carries one post-state per account: the account list
is required to be duplicate-free (`validated_state_diff/mod.rs` rejects a repeated `account_id`
outright), and where two writes to one account do reach the state diff the later **silently
overwrites** the earlier — `state_diff.insert(...)` is a map insert, not a merge, so each chained
call carries its own pre-state and the loser leaves no error behind. A buy that both debited the
buyer for collateral and debited the buyer again for the fee would name the payer twice; a buy that
chained a credit into the vault and also wrote the vault itself would lose one of the two writes with
nothing reported. **So the fee accrues.** The buy books `eff` to the reserve and the remainder to an
accrued-fee counter in sale state, and an authority-gated `sweep_fees` moves accrued fees to the
treasury ATA in its own transaction.

**This is better accounting than an atomic transfer, not a concession to the platform.** Protocol
revenue is recognised in the buy that earns it and moved when it is taken, which is what an accrual
is for; the reserve is never credited with money that is about to leave; and the solvency identity
gains precision rather than losing it. The M2 done gate asserts **collateral vault = booked reserve +
accrued fees, exactly, after every operation** — a three-term equality that fails under the literal
F1 reading in the same way the two-term one did (§3.8), and additionally catches a sweep that takes
more than was accrued. A silent-overwrite failure mode is exactly the kind of defect this gate is
positioned to catch, since the symptom would otherwise be a vault that quietly disagrees with the
books.

**DEV-2 — requested clarification/deviation on F1 and the Reference Implementation.** Read `C_in`
as `eff` in both, i.e. the real collateral reserve and `Vc` each increase by the post-fee amount.
This is the reading we build to: the literal one is insolvent by construction, and the point at
which it fails is F5's withdrawal on a *successful* sale. It is reversible only into that
insolvency, so we treat it as settled rather than optional.

##### The same ambiguity on the sell side (D-8, High)

D-5 is scoped to what enters the pool on a **buy**. The sell side carries the same fee-base
ambiguity, and nothing in the RFP resolves it. The Fee Structure section defines **two** sell-side
quantities: *"the program computes the raw collateral output `C_out_raw` from the pricing formula,
then deducts `fee = C_out_raw × fee_rate` (rounded up); the seller receives `C_out_raw − fee`"*
— Design Rationale, **Fee structure**. `C_out_raw` then appears **nowhere else in the specification**,
and every other site uses the undisambiguated `C_out`:

| Source | Says | Which quantity? |
| :---- | :---- | :---- |
| Design Rationale → **Fee structure** | Defines `C_out_raw` **and** `C_out_raw − fee` | Both, explicitly |
| **F1** — Hard Requirements → Functionality 1 | "the real collateral reserve **decreases by `C_out`**" | Undefined |
| **Reference Implementation** | "`Vc` **decreases by `C_out`**" | Undefined |
| **F6** — Hard Requirements → Functionality 6 | "the transaction reverts if the computed `C_out` is below this minimum" | Undefined — and this one is **user-facing** |
| **F1's rounding mandate** (same requirement) | "on sell, `C_out` rounds down" | Implies the **gross**: the net is already rounded against the trader by the ceiled fee, so a second downward rounding of it would be redundant |

Both readings are available and they are not equivalent. Simulated on D-5's canonical parameters
(`Vt = 1,073,000,191e6`, `Vc = 30e9`, `D = 793,100,000e6`, 100 bps) over a full round trip inside the
F2 envelope — 17 buys of 5 SOL exhausting 99.74% of the sale reserve, then 200 equal sells clearing
the position back into the curve:

| Reading | Booked reserve, end | Actual vault, end | Divergence | `k` drift | Vault runs dry |
| :---- | ---: | ---: | ---: | ---: | ---: |
| **`C_out = C_out_raw`** (gross) | 62 | 62 | **0** | 2.07e-07 % | never |
| `C_out = C_out_raw − fee` (net) | 405,146,053 | **−440,761,672** | **845,907,725 — 1.0000% of sell-side gross** | 1.35e+00 % | **on the 197th of 200 sells** |

Three things the run establishes, and one correction to the obvious framing:

- **The divergence is exactly the cumulative sell-side fee** — asserted in the simulation, not
  inferred: 1.0000% of sell-side gross throughput at 100 bps, scaling linearly with the rate.
- **The sign is the same as D-5's, not the mirror of it.** Debiting the books by the net while the
  vault pays the gross leaves the books *ahead* of the vault — the same direction as crediting the
  gross on a buy. D-8 **compounds** D-5 rather than offsetting it, and a sale with two-way volume
  accrues both. Worth stating plainly, because "the mirror-image defect" is the natural way to
  describe it and it would be the wrong way round.
- **The failure is physical, not merely an accounting mismatch.** The under-debited `Vc` inflates the
  curve — `k` drifts **+1.35%** against the same run's rounding residue of 2.07e-07 %, six orders of
  magnitude apart — so every
  subsequent sell prices *more* collateral out than the pool holds. Above, the vault is exhausted at
  the 197th of 200 sells — counting the first sell whose gross exceeds the vault — so the final sells
  revert against an empty vault, and D-5's failure mode arrives from the sell side too. On D-5's exact methodology, for comparability — 200 successive 5-SOL buys,
  accounting-only, then 200 sells — the divergence is **9,988,867,702**, again 1.0000% of sell-side
  gross, vault exhausted on the 155th sell.

##### What we implement on the sell side, and why it is not a fourth deviation

**`C_out` is the gross, `C_out_raw`.** The real collateral reserve and `Vc` each decrease by the
pricing formula's output; the seller receives `C_out_raw − fee`; the difference accrues to the same
fee counter and leaves on the same sweep, exactly as on the buy side. The gross leaves the reserve
in the sell that earns it, so the three-term identity above holds across both directions.

**This is a stated interpretation rather than a deviation, and the distinction is deliberate.** On
the buy side, F1 names `C_in` — a quantity the Fee Structure defines unambiguously as the gross — so
reading it as `eff` is a genuine departure from the text, which is why DEV-2 exists. On the sell
side, F1 names `C_out`, which the specification never defines against either of the two quantities it
has just created. Choosing the gross resolves an ambiguity *inside* the text rather than departing
from it, it is what F1's own rounding mandate implies (above), and it is the same principle
DEV-2 applies on the other side of the trade: **the curve is credited and debited by what actually
moves through it.** So we book it under DEV-2's rationale and declare no fourth deviation for the
accounting — but we record the reading explicitly, because an implementation that does not say which
`C_out` it chose is one an auditor has to reverse-engineer.

**F6 is the one place a deviation is unavoidable (DEV-3).** If `C_out` is the gross everywhere, then
F6's sell-side floor is literally checked against a number the seller never receives, and a sell can
settle up to the full fee rate below the minimum the seller set — **1.0000% at 100 bps**, the
worst-case floor gap in the simulation above. That is slippage protection silently under-protecting
by exactly the fee, on the one requirement whose entire purpose is to be the user's guard. **We check
the floor against net proceeds**, because the only quantity a seller can reason about is the one that
arrives, and we declare it as **DEV-3** rather than let one symbol quietly carry two meanings. The
SDK's sell quote shows gross, fee and net, so the floor a user sets and the amount they receive are
the same currency.

##### The consequence for the invariant (D-2)

Both the Reference Implementation ("`k = Vt × Vc` is preserved… `k` is computed at creation and
must never change") and R1 (which lists `k` among the state to keep consistent) assert a constant
`k`. **It is not constant, under any correct implementation.** Integer rounding against the trader
necessarily leaves a residue in the pool's favour on every trade, so `k` is **non-decreasing** —
measured drift on a single 5-SOL buy is `+7.35e9`, which is **2.2833e-14 %** of `k` under the correct
fee reading.
That is dust, and it is *the direction solvency requires*: rounding that left `k` exactly constant
would have to round in the trader's favour half the time.

The reason this matters is the comparison: on that same single buy, F1's literal `Vc += C_in` drifts
**1.4306e-01 %** — roughly **thirteen orders of magnitude** larger than the rounding residue, and
systematic rather than dust. Over the 200-buy run below the two readings drift **3.5188e+00 %** and
**1.6533e-10 %** respectively. So "`k` must never change" is not merely imprecise under the literal
reading; it is wrong by ten orders of magnitude on the most generous comparison available, and
correcting the fee base (DEV-2) is also what reduces the invariant claim to something true.

**We therefore restate the invariant as we will implement, test and formally state it:** `k` is
**non-decreasing**, never decreasing, with the per-trade increase bounded by the rounding residue.
This is a strictly stronger safety property than "constant" — a constant `k` permits a decrease;
non-decreasing does not — and it is the property R1's consistency requirement actually needs.

#### 3.5.7 F2 — sale creation, virtual reserves, and the validation the spec omits

Sale creation takes the token pair, sale quantity `D`, optional DEX seed `R`, and the virtual
reserves `Vt`, `Vc`; it transfers `D + R` real project tokens from the creator into program-owned
PDA vaults, records the invariant `k = Vt · Vc`, and initialises the two accounting buckets. The
starting spot price is `p₀ = Vc/Vt`, reported through the existing `spot_price_q64_64`.

Virtual reserves are the net-new core (§3.5.3): the curve prices on `(Vt, Vc)` while custody and the
close condition track the *real* balances. It is the piece with no prior art in `lez-programs`, so
it is where test effort concentrates — sale-reserve monotonicity and DEV-1's exact-boundary
behaviour under property test, and the non-decreasing `k` invariant **asserted in the program on
every operation** rather than only in the suite: the post-trade product is compared against its
predecessor in U256, so a rounding change that moved the curve against the pool would fail the
transaction instead of waiting for a test to notice (C4).

**The validation gap.** F2 states exactly one constraint, `Vt > D`. Nothing bounds `Vc`, `D`, `R` or
the fee rate, and several unvalidated parameter sets are degenerate or hostile: a `Vc` small enough
that `p₀` rounds to zero; `D = 0`; a fee rate at or above the denominator, which
`swap_exact_out_amounts` rejects with `None` and would render sales unfillable. We validate all of
these at creation, in the immutable program, and publish the envelope:

- `Vt > D` — the RFP's own constraint, and DEV-1's totality precondition;
- `D > 0`, `R ≥ 0`. We do **not** impose `Vt ≥ D + R`, which the RFP does not require — `Vt` is a
  pricing parameter, not a deposit (F2.4);
- `Vc > 0` with a documented floor, so `p₀` is representable and non-zero at Q64.64;
- **no constraint on the magnitude of `k`** — see below, it is never a divisor or a stored operand
  the arithmetic depends on;
- fee rate strictly below the denominator, validated against the admin-configured tier set —
  reusing the AMM's fee-tier validation rather than inventing a second policy.

##### `k` is recorded, not computed with — and that is what keeps the parameter space open

The RFP asks for `k = Vt · Vc` **"computed and stored at creation"**, and states separately that
curve state must expose the invariant `k` for reading. Taken as an instruction about representation
rather than about fixity, that is not implementable at the top of the plausible parameter range. For
a pair scaled to eighteen decimals — `Vt = 1e27`, `Vc = 1e24` — the product is `1e51`, and
`u128::MAX` is about `3.4e38`. **A program that makes its arithmetic depend on a stored `k` cannot
serve that pair at all**, and would have to refuse the sale at creation.

**It does not have to.** `k` never appears as an operand in the kernel we port. Each formula folds
into a single `mul_div` whose product is taken in U256 and whose quotient is proven to fit `u128`:

```
buy      tokens_out = Vt − k/(Vc + eff)   =  Vt − mul_div_ceil(Vt, Vc, Vc + eff)
inverse  C_in       = k/(Vt − Q) − Vc     =  mul_div_ceil(Vt, Vc, Vt − Q) − Vc
sell     C_out      = Vc − k/(Vt + t_in)  =  Vc − mul_div_ceil(Vc, Vt, Vt + t_in)
```

That is an algebraic identity, not an approximation: the invariant and every rounding direction are
unchanged. It is also not a construction of ours — it is what `amm_core` already does. There is no
`k` in `amm/core/src/lib.rs`; there is `mul_div_floor` / `mul_div_ceil`, a U256 product, and a
checked narrowing on the quotient. **So the constraint the specification's wording appears to impose
is one the reference kernel already does not carry.**

**What we build.** The invariant is fixed at creation and never changes, which is what the
requirement is *for*. `k` is recorded in sale state as a reported quantity so the state-exposure
requirement is met literally wherever it is representable, and reported as out-of-range where it is
not — a sale is never refused on that ground, because no pricing path reads the field. Everything
that prices reads `Vt` and `Vc` through the identity above. This is a **reading of "computed and
stored", not a deviation** (§3.7): the numbered requirement it comes from constrains the invariant's
fixity and its visibility, and both are satisfied exactly. **R1's "curve state must remain
consistent" is strengthened by this, not weakened** — a `Vt · Vc` that cannot be represented cannot
be asserted against either, so the non-decreasing check runs on the U256 product rather than on a
narrowed field (§3.8).

#### 3.5.8 F3 — both paths, and the private path the spec never specifies

Both paths, program and SDK. The public path is an ordinary LEZ transaction; the private path is
three transactions — deshield (PPE) → trade (public) → re-shield (PPE) — and it works because the
deshield touches only the buyer's private account and a **fresh** public account, which is
uncontended, while the trade is a public transaction the sequencer orders against live state. A
fresh destination is not automatically a *claimable* one, and the distinction is load-bearing: the
runtime mechanism is verified, not assumed, and set out in §3.5.19 and §3.5.32.

**F3 is the only place in the RFP that generalises the pattern to "trade".** Every other instance —
U1, U4–U7, all four Privacy requirements, and the whole Privacy Architecture section including its
interaction flow — is written as deshield→**buy**→re-shield and describes buys only. A private
*sell* is the inverse construction. F3 makes it mandatory; nothing specifies it. **Private sells are
therefore in scope** (§3.5.36): F3 says "trade", and a launchpad whose privacy covers only
the entry is not privacy-preserving, because a private buy followed by a public sell links the
position that was being protected. If Logos intends buys only, this is scope we remove — 1.5–2
dev-weeks off M4 and nothing else changes, because the saga engine is direction-generic (C23). We
build it rather than assume it away, so the reading is visible either way.

#### 3.5.9 Close & Payout

##### F4 — auto-close

The sale closes automatically in the same execution that exhausts the sale reserve; no crank, no
second transaction, no keeper. With DEV-1 in place the exhausting buy exists and the close is
reachable; without DEV-1 it is not (§3.5.5). The close flips sale status, stops further buys (R3),
and makes F5's withdrawal path live. Closing is one state write inside an invocation we are already
paying for, which is why P2 ("close in one transaction") is satisfied trivially rather than by
design effort.

##### F5 — withdrawal, and the defect a non-technical reader should look at first (D-6, High)

F5 as written grants the creator, after close:

- *"The full real collateral reserve."*
- *"The DEX seed reserve `R` tokens **(if not used for auto-graduation)**."*

Auto-graduation — deploying that same collateral plus `R` as DEX liquidity — is a **soft**
requirement, so the two are in direct conflict over the same assets. Both cannot happen. **The spec
caught this for the `R` tokens and missed it for the collateral:** the `R` clause carries the
qualifier "(if not used for auto-graduation)"; the collateral clause carries none.

A hard-requirements-only implementation — what a team building to the letter of this RFP ships —
therefore produces: supply target reached → sale auto-closes → creator withdraws **100% of the
remaining real collateral reserve** → buyers hold tokens with **zero collateral backing, no sell
path, and no DEX pool**.

The sell path is easy to miss: R3 bars further *buys* after close and says nothing about sells, and
sells are paid **out of the very reserve F5 has just emptied**. Auto-graduation would have supplied
the DEX venue, but it is soft and depends on RFP-004. Both holder exits terminate in the same
withdrawn balance. That is strictly worse than pump.fun, the model this RFP cites, where graduation
collateral goes *into* the pool. It compounds with D-3: an unconstrained manual close lets a creator
close a partially-sold curve early, take the raised collateral under F5, and keep the unsold `D` — a
rug vector assembled entirely from hard requirements.

**What we propose.** A contractor should neither quietly redefine where a raise goes nor ship this
as written. We implement F5 with the collateral destination as an **explicit creation-time
parameter**. **The default is literal F5** — the creator withdraws the full real collateral reserve —
with a **graduation-escrow setting** available as the alternative, escrowing the collateral toward DEX
seeding with a defined creator share withdrawable.

**We default to the literal reading deliberately, and the reason is our own scoping.** Auto-graduation
*execution* is out of scope (OOS-1) because it depends on RFP-004, so an escrow default would hold the
raise for a release path this deliverable does not build — permanently locked collateral, which is the
exact failure D-1 makes our headline finding. It would also be a fourth deviation from a hard
requirement, and unlike DEV-1, DEV-2 and DEV-3 it is not forced by a defect: F5 is satisfiable, so we
satisfy it. **Escrow becomes the right default the moment RFP-004 provides somewhere for the collateral
to go**, and the parameter is there so that flip is a setting rather than a redesign. What we will not do
is leave D-6 undisclosed: the U4/U5 surfaces state which destination a sale is configured for before a
buyer commits, so a literal-F5 sale is visible as one. A default is where a contractor puts the burden
of proof, not a limit on what the program can do — and with graduation execution out of scope, that
burden belongs on the escrow setting rather than on the hard requirement.

**F5 names two withdrawals, and a closed sale has three buckets.** The collateral reserve follows the
destination parameter above. The **DEX seed reserve `R`** carries F5's own qualifier — *"if not used for
auto-graduation"* — which under our default is trivially satisfied: auto-graduation execution is out of
scope (OOS-1), so nothing consumes `R` and it returns to the creator in full; under the escrow setting it
is held alongside the collateral toward seeding. The third bucket **F5 does not mention is unsold
sale-reserve `D`**, which any close before exhaustion leaves behind — a normal path now that both the
constrained manual close (C24) and the end timestamp (C26) exist, not just the early-close rug scenario
above. The creator deposited `D + R` at creation (F2), so unsold `D` returns to the creator. We say so
because F5's silence would otherwise strand it: tokens sitting in a program-owned vault that no
instruction is authorised to move.

#### 3.5.10 D-3 and D-4 — two lifecycle gaps we will not paper over

**D-3 — "manual close" is required by two requirements and defined by none.** P2 requires a close in
one transaction *"(manual or auto-triggered)"* and S3 mandates a test case for it, while no
Functionality requirement creates one, says who may call it, or constrains when. Unconstrained, plus
literal F5, it is a rug vector. We implement it **creator-authorised and constrained** — permitted
only after a documented minimum open period, with the collateral destination governed by the same F5
parameter, so early close cannot become a cheaper path to the same withdrawal. **The authority is
the creator and the constraint is a minimum open period** (C24) — the narrowest model that satisfies
P2 and S3 without creating the rug vector, and both the authority and the period are configuration.
S3's mandated test case gets something real to test either way.

**D-4 — the sell curve is not the inverse of the buy curve, though the Design Rationale says it
is.** F1 bars sells from withdrawing more than the **real** reserve holds, while the sell formula
prices against the **virtual** reserve `Vc`. Early in a sale, and after fees have been routed out,
the formula's `C_out` can exceed the real reserve, so the executable price departs from the curve.
Two consequences we handle:

- **Quotes must be executable.** The SDK's sell quote is `min(formula C_out, real reserve)`, so U4's
  summary never shows a price the program will not honour; F6's slippage floor is the on-chain guard.
- **Bucket re-entry is unspecified.** The RFP does not say whether sold-back tokens return to the
  sale reserve `D` — making them re-purchasable and moving the close target back — or to a separate
  pool, which materially changes when F4 fires. We implement re-entry to the **sale reserve** (C24), the
  pump.fun-consistent reading that keeps the supply target meaningful: the alternative makes the
  supply target unreachable for a second, independent reason, since tokens sold back would never
  again count against it. Switching buckets is a one-line change to the close accounting.

#### 3.5.11 F6 and F7 — short, and both satisfied by existing platform capability

**F6 — slippage protection, both directions.** `min_tokens_out` on buy, `min_collateral_out` on
sell; the transaction reverts below either. Mechanically trivial, with three consequences worth
naming: it is what makes DEV-1's terminal partial fill safe (a clamped order either meets the
buyer's minimum or reverts); its sell-side floor is the only on-chain guard against executing a
quote the real reserve can no longer honour (§3.5.10); and **the sell-side floor is checked against
the seller's net proceeds, not the gross `C_out` the formula produces** — DEV-3, for the reason set
out in §3.5.6, since a floor checked on the gross under-protects by exactly the fee rate.

**F7 — ATAs for all token interactions.** Satisfied by the delivered `programs/ata/` — create,
transfer, burn, IDL, private-account ATA creation. `create_associated_token_account` is
**idempotent**: it early-returns on an already-initialised ATA and emits a chained call otherwise,
so a vault that does not yet exist at first buy is not a special case to design around. That matters
because `transfer` *does* refuse an uninitialised recipient (`ata/src/transfer.rs` asserts the
account is not `Account::default()`), and `create` is what closes the gap.

**The RFP's token-authority dependency is mis-specified, and it does not obstruct us.** LP-0013 is
listed under **hard blockers, "currently open"**, and the reason given is that *"token
transfer-authority primitives are required to custody the token sale reserve and the real collateral
reserve."* LP-0013 delivered no such thing. Its subject is the **mint** authority —
`mint_authority: Option<AccountId>`, `MintWithAuthority`, `SetAuthority`,
`SetAuthorityWithAuthority` in `programs/token/core/src/lib.rs`, prize `[CLOSED]` — and the word
"transfer" does not occur in the prize text at all. Nor does the primitive exist by another name:
`transfer_authority`, `delegate` and `approve` return **0 hits** across `programs/token/`. The RFP's
own Resources list titles LP-0013 *"mint authorities"*, so the specification contradicts itself
between its dependency block and its references.

**Custody needs no such primitive, which is why the mis-dependency costs nothing.** Rule 5 forbids a
program from *decreasing* a balance it does not own and says nothing about increasing one, so a vault
that is a **PDA of the launchpad program** is spendable by that program without any delegated
authority. This is the established in-tree pattern rather than a construction of ours:
`programs/amm`'s pool PDA owns its vaults and moves them on every swap. We hold the sale reserve,
the DEX seed reserve and the real collateral reserve the same way (§3.5.7). So neither LP-0013 nor
LP-0014 blocks this work — LP-0014 because it landed, LP-0013 because nothing here ever needed it
(§3.3).

#### 3.5.12 U1 — the SDK, and the one part of it that is not a platform primitive

U1 requires the full lifecycle for both roles across both paths, and on the private path that the
SDK handle **"the atomic deshield (both collateral and gas) as a single indivisible user action."**
That clause is the only genuinely hard part, and it is easy to misread as a platform primitive that
already exists.

**There is no one-shot "deshield token + gas" instruction.** Collateral is a token-program asset;
native gas moves through `authenticated_transfer`. Atomicity has to be *constructed*, not called —
and it is constructible, because a PPE transaction is not single-program: the privacy circuit walks
the whole chained-call chain and verifies each callee. The primary construction is **one PPE
transaction whose entry program emits both deshield legs** into the fresh ephemeral account `A`,
atomic under runtime rollback — 2–3 executions against the PPE cap of 10.

Two facts reduce the risk further, both verified at `9a7a71a` (2026-08-19):

- **The deshield never contends.** It touches only the buyer's private account and a *fresh* public
  account, which has no contender by construction and is claimed by derivation rather than by
  signature — the mechanism, and why the RFP's prescribed pattern works even at sale open when the
  curve is hot, is argued at C14, §3.5.19.
- **The gas leg's size is a runtime property, not a design choice.** The fee model is not
  implemented — `program/mod.rs` still carries `TODO: Make this variable when fees are implemented`.
  **We build the gas leg because U1 names it, and make it conditional on the runtime charging fees**
  (C25). That `TODO` is precisely why the code is written parameterised rather than against an
  assumed rate: whichever way the runtime lands, no rework follows.

The residual question is **proving wall-clock time**, not capability — and it is not an open question
about *whether* the private path is slow. It is. Logos's own `tools/cycle_bench` documents real proving
as *"slow, ~minutes"* per program and PPE composition cases as *"very slow, ~hour"*, and the reason is
structural: `execute_and_prove_with_padded_inputs` adds **each inner program receipt as a circuit
assumption** (`privacy_preserving_transaction/circuit/mod.rs`), so the outer privacy circuit recursively
verifies every program in the chain, and that cost cannot be optimised away by us. **We therefore treat
minutes as a design constant, not a risk to be discovered:** the private path is built as an explicitly
asynchronous flow with per-leg progress and elapsed time from the first commit, never a spinner.

What is genuinely unmeasured is the **marginal** cost of our construction — what the *second* deshield
leg adds to a PPE transaction that would carry one anyway. That is what M0 measures, and the two-leg
fallback is a decision about **atomicity**, not latency. **The question is dormant today:** U1's
atomicity requirement binds *two* asset legs — collateral **and** gas — and there is no user-paid gas
to deshield (C25), so today's deshield is single-asset and atomic in one transaction by construction.
It becomes live only if gas lands **and** the marginal cost proves disproportionate. In that case
splitting the legs preserves U1's "single indivisible user action" — the SDK still presents one
action — but gives up all-or-nothing settlement *across* the two assets, which is a departure from
the stricter reading of the same clause. **We would raise it as a declared deviation at that point
rather than absorb it silently** (§3.7), and disclose the correlation window in the U5 copy. The saga engine is generic over trade direction (C23), so the private-sell path is
incremental rather than a second implementation.

#### 3.5.13 U2, U3 — mini-app and CLI

**U2 — the mini-app.** Both required views — participant (browse sales with spot price, reserve as
a percentage of `D`, collateral raised, price-vs-supply chart; buy; purchase history) and creator
(create with the full parameter set, monitor progress, close, withdraw) — as a QML + C++ module
loadable in Basecamp from a git repository, with build instructions and downloadable assets.

**One required chart is nearly free: the price-vs-supply curve is a pure function of
`(Vt, Vc, k, D)` and needs no history.** It is fully determined at creation — exactly the property
the RFP's Design Rationale claims for a supply-driven close. Only the current-position marker needs
live state, which is one account read.

**The creator view is where the RFP's parameter design meets a usability problem.** A creator must
supply `Vt` and `Vc` — synthetic parameters with no natural units — while the quantities that
matter to them are the starting price `p₀ = Vc/Vt`, the graduation price, and the raise if the sale
completes. All three are closed-form, so we build the view the other way round: the creator enters
the economics they want, the module solves for `Vt`/`Vc` and validates against the F2 envelope
(§3.5.7) before submission. That is a little extra math on quoting code we already have, and it is
the difference between a form that produces sane sales and one that produces `p₀`-rounds-to-zero
sales.

**U3 — the CLI.** The RFP permits a feature subset; we generate it from the SPEL IDL over the same
core entrypoints as the GUI, with a test asserting entrypoint parity so "fewer features" never
becomes "different behaviour".

#### 3.5.14 U4 — the pre-buy summary, and where D-5 becomes user-visible

U4 requires, before every purchase: collateral to spend, exact tokens to be received computed with
the pricing formula, current spot price `Vc/Vt`, price impact as the percentage increase in spot
price after the buy, and **the per-swap protocol fee deducted from collateral**.

Two of those five are already implemented in Logos's own code — `spot_price_q64_64` gives the spot
price, and the token quantity is the same `swap_exact_in_amounts` the program runs, executed
client-side against the same state, so the displayed number is the executed number by construction
rather than by agreement.

**The third is a near-miss worth naming, because the obvious reuse is the wrong number.**
`amm_core::price_impact_bps` looks like U4's price-impact line and is not. It computes
`DEN − DEN·amount_out/(reserve_out·amount_in/reserve_in)` — how far the output falls short of a naive
pre-trade valuation of the input, i.e. **execution slippage**, which the function's own comment calls
*"the fraction of the spot value the trader keeps"*. U4 asks for something else: *"price impact
(percentage increase in spot price after the buy)"*. On the canonical parameters, a 1-SOL buy at sale
open with a 100 bps fee gives **320 bps** from `price_impact_bps` against **670.89 bps** for U4's
metric — a factor of 2.1, because for small trades a spot-price move is roughly twice the execution
shortfall. There is a second trap: the function takes `amount_in`, so passing the gross `C_in` rather
than `eff` folds the fee into the "impact" (417 bps here) and double-counts it against the fee line
U4 requires separately.

**So we compute U4's metric in closed form rather than reuse the wrong one:** the post-buy spot is
`(Vc+eff)²/(Vt·Vc)`, so the increase is `(Vc+eff)²/Vc² − 1`, one U256 expression on top of the kernel
we are already porting. `price_impact_bps` is still reused, for the quantity it actually computes —
the execution-slippage line, labelled as such, next to U4's five required values rather than in place
of one of them.

The fee line is the interesting one. **U4 is where D-5 stops being an accounting subtlety and
becomes something a user reads.** Under the RFP's Fee Structure the fee is deducted from `C_in`
before pricing, so the summary shows a fee and a token quantity priced on `C_in − fee`. Under F1's
literal `Vc += C_in` the reserve is credited with the fee as well, and there is no consistent pair
of numbers to display — the fee shown to the user is simultaneously taken and not taken. Our
DEV-2 resolution makes the summary well-defined: fee deducted, priced on the remainder, one
arithmetic story shown to the buyer and executed on-chain. The sell-side summary carries the same
obligation for the same reason (D-8): it shows the gross, the fee and the net, and the slippage floor
the user sets applies to the net (DEV-3), so no number displayed is one the seller does not receive.

For the private path the summary additionally discloses the price the buyer is *likely* to get
rather than a price we cannot guarantee — see the Privacy section on D-7, which is where the
honest disclosure of the private path's price penalty belongs.

#### 3.5.15 U5, U6, U7 — the three private-path guardrails

Three client-side obligations, all achievable today and all cheap relative to the saga engine they
sit on.

**U5 — privacy disclosure before each buy.** Generated from the *verified* on-chain visibility
surface rather than the requirement's prose — the same account and message model the saga builds —
so the copy cannot drift from what actually appears on-chain. We add one disclosure the requirement
omits, because leaving it out would mislead: the private path's price exposure (D-7, §3.5.35).

**U6 — enforce the atomic deshield.** Enforced in the core module rather than the GUI: the
ephemeral is saga-created, deshield-funded, single-use, and unreachable by any user-initiated
transfer path. Core-level enforcement is what extends the guarantee to the CLI, which U6 does not
mention but which could otherwise reintroduce the linkage.

**U7 — shielded-balance pre-check.** Covers collateral **and** gas within the single deshield, with
an actionable error otherwise. The gas component is meaningful only once the fee model is live, so
the check is parameterised rather than pinned to an assumed rate (C25). Because private sells are in
scope (C6), the check also has a sell form — covering gas plus the *token* leg — which the
requirement, written for buys only, does not state.

#### 3.5.16 U9 — the IDL, and why it is scheduled first

Generated from the program definition, so it is the smallest deliverable in the block — and we
schedule it first regardless of size, for two reasons: it is the interface contract that unblocks
parallel client work (C8; frozen at M1), and **it is the decoder that makes U8 possible without
LP-0012** (§3.5.17). The IDL is load-bearing here, not a documentation artifact.

#### 3.5.17 U8 — sale analytics, and the platform dependency that is not delivered

U8 requires, per sale: total collateral raised, current spot price, supply sold over time as a
progress chart, the price-vs-supply curve with the current position marked, and the number of buy
transactions — **without exposing participant identities or linking buys to accounts.**

##### The dependency the RFP calls resolved, and is not

The RFP's Platform Dependencies section lists **LP-0012 structured event emission** under
**"Resolved"**, and the natural way to build U8 is on it. It is not delivered.

Verified against `logos-execution-zone` @ `9a7a71a` and release tags `v0.2.1`–`v0.2.4`, **2026-08-19**:

- `grep -rn "emit_event" --include=*.rs .` → **0 hits**
- `grep -rn "TxReceipt" --include=*.rs .` → **0 hits**
- the only `*EventRecord` symbol in the tree is `PendingDepositEventRecord`, which is bridge-deposit
  plumbing, unrelated to program event emission
- the roadmap's own `lez_events.md` carries "LEZ publishes events" as a **testnet-0.3 milestone with
  all three checklist items unchecked**: event model spec, emission from block execution, exposure
  through the indexer

One nuance we want to state accurately rather than overstate: the corresponding Lambda Prize *is*
marked `[CLOSED]` and was genuinely won — but the implementation lives in a **contributor fork**, not
upstream. "The prize is closed" and "the platform has events" are both true-sounding and different
claims, and only the first is correct. Worth correcting in the RFP text regardless of who wins the
work, because the RFP's own text invites bidders to price against a capability that does not exist.

##### Our path: analytics that do not depend on LP-0012 at all

LEZ already ships a complete indexer service (`lez/indexer/`) exposing what an analytics observer
needs. Verified at the same commit:

| What we need | What the platform serves today |
| :---- | :---- |
| Per-transaction calldata | `PublicMessage { program_id, account_ids, nonces, instruction_data }` — full instruction data, per transaction |
| Per-sale state over time | RPC `getAccountAtBlock` — account state at **block** granularity |
| Only successful activity | The sequencer **skips** transactions that fail validation, so they never appear in a block at all |

The observer therefore works as: filter transactions by `program_id` → decode `instruction_data`
with the SPEL IDL from U9 → read the sale PDA per block via `getAccountAtBlock` for reserves, spot
price and supply sold → where several buys land in the same block, replay the decoded instructions
sequentially from the block-start state to recover each trade's `tokens_out`, using the same
deterministic pricing function we implement on-chain.

Two properties make this much cheaper than a general event pipeline. **Failed transactions never
appear**, so there is no revert filtering and no speculative state to unwind. And **block-level
state is served directly**, so the observer never re-executes programs — at worst it re-runs our own
pricing function over decoded calldata inside one block.

One honest cost: per-transaction post-states are **not** available for public transactions —
`PublicActionWithID { account_id, post_state }` appears only inside
`PrivacyPreservingMessage.public_actions`. That is precisely why the intra-block replay step exists.
It is real work, not a lookup, and sized as such: **approximately 1 to 1.5 developer-weeks** for the
decoder, a rollup store keyed by sale PDA, and the four U8 series. A task inside a milestone, not a
milestone.

**U8's privacy clause is satisfied by construction** — the only address appearing on the private
path is the single-use ephemeral, the observer aggregates by sale, and the view has no account-keyed
query surface to expose.

**C9 — analytics ship independent of LP-0012, with a documented migration.** When events land, the
ingestion layer is replaced behind a stable interface and the views are unchanged. Nothing in the
delivery plan waits on it. This converts a stale entry in the RFP's own Platform
Dependencies section into a risk we do not carry.

#### 3.5.18 U10 — actionable errors, and the one the specification guarantees

U10 requires clear, actionable messages on failed or rejected buys, and names three examples:
insufficient balance, **supply target already reached**, and slippage exceeded. One error taxonomy
lives in the core module, with a single message table shared by GUI and CLI, and every program
error code mapped to exactly one user-facing message plus a suggested action.

The second named example deserves comment. **"Supply target already reached" is the error that
D-1 causes to fire on every buy that would complete a sale** — under F1 as written, the buy that
should close the sale reverts with precisely this message, forever, and the sale never closes. With
DEV-1 the message becomes truthful: it fires only after the sale has actually closed, and the buy
that exhausts the reserve is partially filled rather than rejected. The requirement's own example
list is, unintentionally, a description of the defect.

Two error classes we add beyond the requirement's list, because the private path makes them
reachable and a bare failure would be actively confusing: a dropped PPE transaction needing a
re-prove (with the reason stated, not just "failed"), and a partial fill on the terminal buy (which
under DEV-1 is a success, and must be presented as one — "your order filled 812 of 1,000 tokens and
closed the sale; the remaining collateral was refunded").

#### 3.5.19 R1 — consistency under concurrent buys

##### Why this is satisfied by construction on the public path

The LEZ sequencer does not execute transactions optimistically or in parallel. It validates each
transaction **against the running state** and then applies it, one at a time
(`lez/sequencer/core/src/lib.rs`: `tx.validate_on_state(state, block_height, timestamp)` followed
by `state.apply_state_diff(validated_diff)` inside the sequential per-transaction loop, verified at
`9a7a71a`, 2026-08-19). Every buy is therefore priced against exactly the curve state that precedes
it in block order. There is no read-modify-write window, no lost update, and no reordering hazard
for the program to defend against.

**Double-spend is excluded one level below us**, by nonce validation and by the fact that a
transaction failing validation is **skipped entirely** — `"failed execution check with error: …,
skipping it"`, and the loop returns without including it. A rejected buy is not an included failed
transaction; it is absent from the block. There is no partial state to reconcile.

So R1's substance for us is not concurrency control — the platform provides it — but **making sure
the program's own accounting is correct against a strictly ordered stream**, which is what our
invariant tests target (§3.8 below).

##### The failure mode this design deliberately avoids (C14)

R1 is easy to satisfy here and hard to satisfy in a neighbouring design, and the difference is
worth stating because it is the reason the RFP's prescribed privacy pattern is shaped the way it
is.

A privacy-preserving execution (PPE) transaction is **pre-state-bound to every public account it
touches**. At validation the runtime rebuilds the expected journal from *live* state and
re-verifies the client's receipt against it; if the live pre-state differs from the one the client
proved against — by even one field, including a nonce — verification fails, and the sequencer skips
the transaction. Proving is client-side and succinct: re-proving costs seconds to minutes.

The bonding curve PDA is the single hottest account in the system, written by every buy, on a sale
whose entire premise is concurrent demand at open. **Any design that puts the buy itself inside a
PPE transaction therefore fails R1 under exactly the load R1 is about** — at most one such buy
survives per block and the rest are silently dropped and must be re-proved.

Our design does not do that, and neither does the RFP's: the private path is
deshield (PPE) → **buy (public)** → re-shield (PPE). The deshield touches only the buyer's private
account and a **fresh** ephemeral account, so it has no contender; the buy is an ordinary public
transaction whose ordering is the sequencer's responsibility. **C14: the buy stays public on both
paths.** We state the mechanism explicitly because the RFP prescribes the pattern without recording
the liveness reason for it, and because that reason is what makes R1 and the privacy requirements
compatible rather than in tension.

**Freshness buys the absence of a contender; it does not by itself buy admission.** The privacy
circuit will not let a program claim an arbitrary public account. In
`privacy_preserving_circuit/src/execution_state.rs` a post-state claiming an account asserts the
account is uninitialised — *"Cannot claim an initialized account"* — and then splits on the claim
kind. `Claim::Authorized` on a **public** account asserts `pre_is_authorized`, failing otherwise with
*"Cannot claim unauthorized account"*; `Claim::Pda(seed)` asserts only that the id matches the
derived PDA, with **no authorization requirement**; and private accounts are exempt entirely, since
*"unauthorized private claiming is intentionally allowed"*. So a fresh public destination that is
neither signed for nor claimed by derivation is refused — and refused inside the circuit, where there
is no receipt to carry the reason.

**Which is why the ephemeral is a derived account, not merely a new one.** Priv4 already requires the
ephemeral never to be reused, and the cheapest way to guarantee that is derivation rather than
convention (§3.5.34). The same derivation supplies the claim: the destination is a per-purchase PDA
of our entry program, claimed by seed, so it needs no signature it does not have. Where a leg
requires the ephemeral itself to sign — the public buy does — the client holds that keypair and
authorises it in the transaction that needs it, which is the `Claim::Authorized` branch satisfied
rather than avoided. **We take both routes deliberately and say which applies to which leg, because
the failure mode of getting this wrong is a refusal with no error attached.** Authorisation is also
monotonic once granted: `validated_state_diff/mod.rs` unions the caller's authorised set so that an
account authorised anywhere in a chain stays authorised for the rest of it.

##### Two precisions we would rather state than have discovered

**R1 names `k` as state that must stay consistent, and `k` is not constant.** We verify it in the
form that is both true and stronger than the Reference Implementation's "must never change": `k`
**non-decreasing**, never decreasing (C4, D-2, argued in §3.5.6). Constant permits a decrease;
non-decreasing does not, and that is what R1's consistency requirement actually needs.

**R1 is a correctness property, not a price guarantee, and we do not blur the two.** A private-path
buyer whose transaction lands behind three public buys receives fewer tokens than their pre-trade
quote suggested. That is *correct* under R1 — the accounting is exact, the invariant holds, nothing
is double-spent — and it is still a real cost to that buyer. It is D-7, it is a property of bonding
curves rather than a bug in this implementation, and it is disclosed and quantified in the Privacy
block rather than being quietly absorbed into a reliability claim.

#### 3.5.20 R2 — atomic revert of a failed buy

> *"A failed buy must revert atomically: the buyer's collateral is not consumed and the curve state
> is unchanged."*

Satisfied by a stronger mechanism than the requirement asks for. The runtime's rollback is
all-or-nothing across the buy's three chained executions, and the sequencer **skips** a transaction
that fails validation rather than including it — so the buyer's collateral is untouched because
**the transaction had no effect at all**, not because we unwind it. There is no intermediate state
in which collateral has moved and tokens have not. This covers the deliberate revert paths that
actually fire in production: slippage (F6), insufficient balance, and buys against a closed sale.

**One interaction with DEV-1 that must not be conflated.** The clamped terminal buy is **partially
filled with a refund — a success, not a failure**, so R2 does not govern it. The refund rides inside
the same atomic unit as the fill: the buyer gets tokens + refund + close, or nothing. U10 must
present it as the success it is (§3.5.18).

#### 3.5.21 R3 — auto-close on supply target: unsatisfiable as specified

> *"Auto-close on supply target: when the final sale tokens are sold and the sale reserve is
> exhausted, the sale must close atomically in the same transaction. No additional close
> instruction should be required; no further buys must be accepted after close."*

**As the hard requirements stand, R3 cannot be satisfied by any implementation.** Its trigger is the
state in which the sale reserve is exactly exhausted, and D-1 shows that state is arithmetically
unreachable under F1's must-revert rule (§3.5.5).

**R3 also closes the obvious escape hatch.** The natural workaround for an unreachable exhaustion
is a separate close instruction — and R3 says *"no additional close instruction should be
required."* We read "should" as a strong preference, not an absolute bar. But the alternative is no
better: the "manual close" P2 and S3 both presuppose is defined by no Functionality requirement at
all (D-3). Under a literal F1 there is no close path that is both reachable and specified.

##### What DEV-1 restores

With the partial-fill clamp (§3.5.5), the terminal buy fills exactly the remaining reserve, so
**exhaustion is reachable by construction** — R3's trigger state exists; **the close is atomic in the
same transaction**, a state write inside an execution we already pay for, so no additional instruction
is required and R3's second clause is met in the strong sense rather than the permissive one; and
**no further buys are accepted after close** — a status check at the top of `buy`, with U10's "supply
target already reached" error, which becomes a truthful message only once the sale can reach that
state.

R3 is the requirement DEV-1 exists to make satisfiable. **The clamp and the end timestamp are two
independent close conditions, not alternatives:** DEV-1 makes the supply target reachable, and the end
timestamp bounds a sale that never gets there. We implement both (C26).

If Logos declines DEV-1, R3 cannot be delivered by us or by anyone, and the alternative scoped in
§3.5.5 applies: the end timestamp is promoted from **soft to mandatory**, so that every sale still has
a reachable close, with F5 re-gated on it. The promotion is necessary because an *optional* end
timestamp cannot carry that load — a creator who simply does not set one leaves a sale that can never
close at all. That route satisfies F4 and F5 while leaving R3 formally unmet. We state that now rather
than deliver a requirements matrix with an untruthful row in it.

#### 3.5.22 P1 — a buy completes within one LEZ transaction

> *"A single buy transaction completes within one LEZ transaction."*

Satisfied on both paths, and the precision matters. **The buy is one transaction, costing 3 of the
11 executions** available to a public transaction — top-level `buy`, collateral in, tokens out. The
AMM's own public swap costs 4 by the same accounting because it moves an LP token as well; ours
carries no fee leg, for the reason in §3.5.6.

**The execution budget is not uniform, and we state which one we spend.** The guard differs by
path: public execution and the in-circuit verifier check `chain_calls_counter <= MAX` *before*
incrementing (**11 executions**), while the PPE prover uses a strict `>=` (**10**). A client-proved
private transaction therefore has one *fewer* execution than a public one — a detail that matters to
anyone budgeting private flows.

**On the private path P1 still holds.** The path is three transactions, but the *buy* is one, which
is what P1 asks about; the deshield is **2–3 executions** — three once a gas leg exists (C25) — and the
re-shield **1–2**, each against the PPE cap of 10, and both are presented as one user action (U1). We say three rather than one because summing them
mis-models the platform — and because the three-transaction shape is what makes R1 satisfiable
(C14). Headroom: 8 of 11 executions unused, and DEV-1's clamp adds arithmetic inside an existing
execution, not a new one. The invocation limit is not a design constraint on this program, which on
LEZ is unusual.

#### 3.5.23 P2 — a close completes within one LEZ transaction

> *"A close transaction (manual or auto-triggered by final buy) completes within one LEZ
> transaction."*

**Auto-close: satisfied, and free.** Under DEV-1 the close is a state write inside an execution the
terminal buy already pays for — no extra execution, transaction or measurable cycle cost. Without
DEV-1 the trigger state is unreachable and there is nothing to time.

**Manual close: P2 presupposes a requirement that does not exist.** The parenthetical *"(manual or
auto-triggered)"* is one of two places the RFP assumes a manual close — S3's test list is the other
— while no Functionality requirement defines who may call it or when (D-3). We implement it
creator-authorised and constrained (§3.5.10, C24), in one transaction, against a specification we
write because the RFP does not. P2 is the requirement most easily mis-satisfied: it is easy to tick "close completes in
one transaction" against an instruction nobody specified.

#### 3.5.24 P3 — documented CU cost per operation

##### We use Logos's own harness

LEZ ships a first-party cycle-benchmark tool, **`tools/cycle_bench`**, and its README states its
purpose exactly: *"Per-program Risc0 cycle counts, prover wall time, PPE composition cost, and
verifier wall time for the built-in LEZ programs. Feeds the fee model (`G_executor`, `G_prove`,
`G_verify`, `S_agg`)."* Verified at `9a7a71a`.

This is the tool Logos itself uses to derive its fee model, so measuring P3 with anything else
produces numbers that are not comparable to the platform's own. We use it, and we report in its units —
which requires one mapping stated up front, because **P3 asks for "compute unit (CU) cost" and LEZ has no
CU denomination yet.** `cycle_bench` reports *executor cycles*, and the fee model it feeds (`G_executor`,
`G_prove`, `G_verify`, `S_agg`) is where cycles would become a charged unit once fees exist (C25). We
therefore report executor cycles as the CU figure, name them as such, and state the conversion as
unresolved at the platform rather than inventing one. It runs in three tiers — executor cycles in seconds, real proving per program in minutes,
PPE composition cases in about an hour — plus a criterion microbenchmark for verifier time, so we
can run the cheap tier continuously and the expensive tiers at milestone boundaries.

**C17 — CU costs are CI-gated, not measured once.** `cycle_bench` writes machine-readable
`target/cycle_bench.json`. We therefore commit the measured envelope as a **regression gate**: the
cheap executor-cycle tier runs in CI and fails the build if any operation's cycle count moves
outside a committed tolerance. P3 asks for a document; a document goes stale on the next commit. A
gate does not, and it costs us almost nothing because the harness already emits the artifact.

##### What we will report

For each of the four named operations — create sale, buy, close sale, withdraw — we publish
executor cycles, the execution count against the 11/10 budget, and the transaction's account set,
against a **pinned testnet version and commit SHA** as P3 requires. We add three lines the
requirement does not ask for, because omitting them would make the document less useful than it
looks:

- **The private path's legs** (deshield, re-shield) measured against the **PPE cap of 10**, not the
  public 11.
- **Proving wall-clock time** for the PPE legs, reported as the *marginal* cost per chained call as well
  as the absolute figure, since the absolute one is a platform property and the marginal one is ours. On
  the private path this, not CU, is what a user actually waits for — and it sizes D-7's exposure
  window.
- **The clamped terminal buy** measured separately from an ordinary buy, since DEV-1 makes it a
  distinct code path.

##### The honest headline: cycles are not the binding constraint here

The curve is multiplication and division on `u128` with U256 intermediates — no fixed-point
exponentiation, no iterative approximation, no convergence argument. Against a 32M-cycle public
execution envelope (`MAX_NUM_CYCLES_PUBLIC_EXECUTION = 1024 * 1024 * 32`), the arithmetic is
negligible; the cycle cost of a buy is dominated by serialisation and account handling, not by
pricing.

**C18 — we budget against executions and proving time, because those are what actually bind.** The
scarce resources on this program are the 11/10 invocation budget (§3.5.4) and, on the private path,
client-side proving wall-clock. Cycles are the plentiful one. We say this rather than presenting a
CU table as though it were the interesting risk, and we note in passing that the cycle limit itself
carries `TODO: Make this variable when fees are implemented` — so the envelope is not yet a metered
resource at all, which is the same runtime gap C25 makes the gas leg conditional on.

#### 3.5.25 The unnumbered mandate — and why it shapes the milestone plan

Before S1, the Supportability block opens with an unnumbered sentence that is a hard requirement in
substance:

> *"Proposals must include separate milestones for testnet 0.2, testnet 0.3, and mainnet
> deployment."*

We count it as such. RFP-015 has **34 numbered hard requirements**; counting this mandate the
honest figure is **35**, and our traceability matrix carries it as a row rather than letting it
fall between the numbering. It is also the requirement that determines milestone *shape*: three of
our milestones exist because this sentence says they must, and two of those three have trigger
dates Logos controls (S6/S7 below, and M7/M8 in the Milestones section).

#### 3.5.26 S1 — deployed and tested on LEZ testnet 0.2

**Satisfiable today.** The 0.2 line is live and moving quickly — **v0.2.1 on 2026-08-02, v0.2.2, v0.2.3,
and v0.2.4, the current release, on 2026-08-07** — so this milestone depends on nothing that is not
already live: the program deployed, the suite green against it, CLI and mini-app driving it end to end,
and the P3 envelope measured with version and commit SHA pinned. **The contrast with 0.3 is the point:**
four point releases landed inside six days while testnet 0.3 stands at 0 of 12 roadmap milestones with
no published date (§3.5.31). We pin to a release and a commit SHA rather than to "testnet 0.2", because
on this cadence the two are not the same statement.

This is the structural reason the 12-week commitment is credible: **everything the RFP asks for
except S6 and S7 can be completed against a platform that exists now.** Nothing in the main
programme waits on an unshipped capability — including U8 analytics, which is why we removed the
LP-0012 dependency (C9).

#### 3.5.27 S2 — end-to-end tests against a standalone sequencer, in CI, green on default

The sequencer's `standalone` feature is available (`lez/sequencer/service`),
so this is CI engineering, not research. Two lanes, both in CI: a **standalone-sequencer lane**
driving create → buy → slippage revert → sell → clamped terminal buy → close → withdraw against a
real sequencer, and an **in-process harness lane** carrying the invariant and property testing,
where we control state and can generate long operation sequences cheaply. CI is cloned from
`lez-programs`' own configuration and fails on red — "green on the default branch" as an enforced
property, not a claim. The same lane carries the R1/R2 e2e obligations, so it is built once.

#### 3.5.28 S3 — a test per hard requirement, and two mandated cases the spec cannot support

> *"Every hard requirement in Functionality, Usability, Reliability, and Performance has at least
> one corresponding test. Test coverage must include: invariant preservation across multiple buys,
> happy-path buy, slippage revert (tokens_out below minimum), auto-close on supply target, manual
> close."*

##### The scope of the obligation, stated precisely

S3 names four blocks — Functionality, Usability, Reliability, Performance — which is **23 hard
requirements** (7 + 10 + 3 + 3). It does **not** name Supportability or Privacy. We read that as
written and do not inflate it, but we note that we test the four Privacy requirements anyway,
because a privacy-preserving launchpad whose privacy properties are untested would be an odd
artifact regardless of what S3 requires. The committed coverage matrix maps all 23 named
requirements to at least one test, CI enforces that the mapped tests exist and are green, and the
matrix is a delivered artifact rather than an assertion in a report.

##### Two of the five explicitly mandated test cases test behaviour the specification does not support

This is the point in the RFP where D-1 and D-3 stop being analysis and become an acceptance
problem, because S3 does not merely permit these tests — it **mandates** them:

| Mandated case | Status |
| :---- | :---- |
| Invariant preservation across multiple buys | **Supported** — covered by the property-test suite (C15); note the invariant we assert is `k` **non-decreasing** (C4/D-2), not constant |
| Happy-path buy | **Supported** |
| Slippage revert (`tokens_out` below minimum) | **Supported** — F6 |
| **Auto-close on supply target** | **Untestable as specified.** The trigger state is arithmetically unreachable under F1's must-revert rule (D-1). A conformant implementation cannot pass this test, because the behaviour it tests cannot occur |
| **Manual close** | **Untestable as specified.** No Functionality requirement defines a manual close — who may call it, or when (D-3). There is no specified behaviour to write a test against |

**With DEV-1 and the constrained manual close of C24, both become testable**, which is the cleanest
possible statement of why those two items are in the bid:

- *Auto-close on supply target* becomes the boundary regression test of Reliability §3.8 — run on
  the exact D-1 parameter set, asserting the sale closes where a literal-F1 implementation reverts
  forever.
- *Manual close* becomes a test of the creator-authorised, constrained close we specify in §3.5.10,
  with negative tests for the rug vector D-6 + D-3 would otherwise create.

Better to show this table now than deliver a coverage matrix at S3 with two rows marked "not
applicable" and a footnote. **A bid that does not request DEV-1 cannot pass its own S3.**

#### 3.5.29 S4 — the README

Four required contents, delivered as four sections: deployment steps (reproducible from a clean
checkout, with the toolchain pinned), program addresses (per testnet version, updated at S6),
creator walkthrough, participant walkthrough — each of the last two written twice, once for the CLI
and once for the mini-app, because S4 requires both surfaces.

One commitment beyond the requirement: the creator walkthrough documents the **parameter derivation**
of C12 — how a creator gets from the economics they want (starting price, graduation price, target
raise) to `Vt` and `Vc` — including the validation envelope of §3.5.7 and what each rejection means.
The RFP's own parameters are the least intuitive surface in the product, and a README that lists
`Vt` and `Vc` without explaining how to choose them is documentation that satisfies S4 and helps
nobody.

#### 3.5.30 S5 — the privacy and anonymisation properties document

S5's four required contents — what observers can see, what the private path protects, trust
assumptions split by enforcer, and what happens when a user bypasses the expected path — are each
written from verified runtime behaviour rather than from the RFP's prose.

**(a) What is visible** — all curve state by design, and per buy the transaction, collateral
amount, tokens received, curve address and ephemeral account. Generated from the same account and
message model the SDK builds, so it cannot drift from reality.

**(b) What is protected** — the buyer's private account and the onward destination of purchased
tokens, because the only address on-chain is the single-use ephemeral. We document why that holds
(Priv4's never-reuse rule) and what it does not cover.

**(c) Trust assumptions, split by enforcer** — the part of S5 with the most value to Logos, because
the split is not obvious:

| Guarantee | Enforced by |
| :---- | :---- |
| Curve pricing, rounding, solvency, close semantics | **The on-chain program** — no client can violate these |
| Atomicity of the deshield legs | **The runtime** — all-or-nothing rollback across the chained call |
| Ephemeral account is single-use and deshield-funded | **The client** (U6, C13) — the program cannot tell a fresh account from a reused one |
| Non-linkage of the buyer's identity | **The client** — it is a property of how the account is funded and reused, not of program state |

The headline: **the program enforces the money, the client enforces the privacy.** A user driving
the program directly with a public account gets correct pricing and no privacy, by design.

**(d) What happens when a user bypasses the expected path:**

- **Funding the ephemeral from an existing address** (what U6 prevents) permanently links that
  identity to every buy from that account — irreversible, and undetectable on-chain.
- **Reusing an ephemeral across purchases** (contra Priv4) turns a set of unlinked buys into a
  traceable cluster.
- **Skipping the re-shield** leaves tokens on the ephemeral, where any later movement links them.
- **Driving the program directly** is supported and fully public — the program offers no privacy
  guarantees on its own.

**(e) Beyond the requirement: the private path's price exposure (D-7)**, quantified with its
mitigations in §3.5.35. Omitting it would make the document misleading by silence.

**(f) Corroboration.** C14's contention analysis is not a speculative reading of the runtime:
**Logos's own roadmap carries "LEZ Resolve State Contention" as an open Parallel Milestone**, its
stated risk being *"State contention is a fundamental challenge in the private execution model"*,
both items unchecked. Our design keeps that open platform problem off a buy's critical path, and S5
says what would change here if the contention work later alters the private execution model.

#### 3.5.31 S6 and S7 — the two milestones whose dates Logos owns

S6 updates and verifies the program on LEZ testnet 0.3; S7 deploys it to LEZ mainnet.

Verified against `logos-co/roadmap` @ `77a9418`: the **"Testnet v0.3 Milestones"** section contains
**12 items, all unchecked** — including "LEZ publishes events" (LP-0012), the capability the RFP's own
Platform Dependencies section calls resolved (§3.5.17). There is **no published target date**, and observed cadence is roughly a
quarter per release cycle (v0.1.2 → v0.2.0 was 2026-04-27 → 2026-07-03).

**One precision we will not overstate.** The **LEZ Crypto Audit** sits under the roadmap's
**"Parallel Milestones"** heading, not under "Required for Mainnet". Inferring that it gates mainnet
is reasonable, but the roadmap does not say so and we will not assert a gate the source document
does not contain. Without inference: mainnet is further out than testnet 0.3, and testnet 0.3 has no
date.

**C19 — S6 and S7 are narrowly-scoped milestones gated on platform availability, sitting
outside the 12-week calendar**, each with its scope agreed up front so neither becomes open-ended
(M7 and M8 in the Milestones section). This is the only honest way to satisfy the unnumbered mandate
and the 12-week expectation at once; the alternative requires asserting a testnet-0.3 date that
exists in no Logos source, and we will not price against a date we cannot cite. If Logos holds an
internal target we have not seen, the trigger simply becomes that date — the scope and the price do
not move, which is the point of pre-pricing them. S6 is also where the C9 analytics migration lands,
if LP-0012 has arrived by then.

#### 3.5.32 Priv1 — both paths, and a non-skippable re-shield

> *"The mini-app and SDK must support both direct public account interaction and the
> deshield→buy→re-shield pattern… the SDK must enforce the complete pattern; the re-shield step
> must not be skippable."*

The U1 saga engine satisfies this: deshield, trade and re-shield are one user-level operation with
no intermediate state the UI returns to — no skip affordance, no partial-completion screen, and no
path through mini-app or CLI that leaves a saga at step 2.

**What "non-skippable" can and cannot mean.** The RFP's own Trust Assumptions concede the limit —
*"The bonding curve program itself cannot enforce the re-shield step."* We will not claim otherwise.
Priv1 is an obligation on **our client**, and we satisfy it there.

**The engineering problem Priv1 creates, which the RFP does not mention: crash recovery.** A saga
that must not be skippable must survive the process dying between steps 2 and 3 — otherwise the
guarantee is defeated by a laptop lid and the user's tokens sit exposed on a public ephemeral
account. So the saga is **durably journalled before the deshield is submitted**, and on restart the
client detects the incomplete saga and completes the re-shield.

This needs one reading on record, and we build on it as a decision: **resuming an interrupted saga is
completing an operation, not reusing an account.** Priv4 forbids reuse *across operations*; finishing
the operation the account was created for is that same operation, and the account acquires no second
counterparty, amount or observer-visible event. The alternative forbids crash recovery outright and
strands the user's funds permanently on a public ephemeral account — the exact exposure Priv4 exists
to prevent — so it cannot be what Priv4 intends. **C20**, and M4's done gate tests it: resumption
must generate no second ephemeral.

#### 3.5.33 Priv2 — the pre-confirmation privacy summary

Satisfied by U5's disclosure surface, generated from the saga's own account and message model so it
cannot drift from what actually appears on-chain (§3.5.15). Priv2 adds the timing obligation —
**before each buy**, not once per session — which we enforce in the core module rather than the GUI,
so the CLI inherits it (C13).

The one addition we make beyond the requirement is stated in §3.5.35: the summary discloses the
private path's **price exposure**, not only its information exposure. A privacy summary that
explains what an observer can see, while staying silent about what the path costs, is incomplete in
a way users would reasonably object to.

#### 3.5.34 Priv3 and Priv4 — target validation and ephemeral hygiene

**Priv3 — validate the re-shield target is a shielded account.** LEZ distinguishes account kinds at
the type level (`InputAccountIdentity::{Public, Private(PrivateWitness)}`),
so this is a well-defined client-side check rather than a heuristic. The SDK validates the
re-shield target **before submitting the transaction** — as the requirement specifies, so the
failure is caught before any funds move rather than after the deshield has already exposed an
amount — and rejects with an explicit, actionable error (U10's taxonomy).

**Priv4 — fresh, single-use ephemeral accounts.** Each private operation generates a new account
with no prior on-chain history; the SDK never selects an existing account for this role, and the
mini-app offers no way to nominate one (U6, enforced in the core so the CLI inherits it).

**Priv4 is also a liveness property, not only a privacy rule — and this is the connection that makes
the whole private path work.** A freshly *derived* account has no contender and carries its own claim,
so the deshield PPE transaction is neither queued behind a competing writer nor refused for want of a
signature (C14, §3.5.19). Priv4's never-reuse rule is therefore what keeps the deshield reliable as
well as unlinkable, and it is why the RFP's prescribed pattern is sound even at sale open when the
curve itself is maximally contended. **Derivation is doing two jobs here, and we would not satisfy
Priv4 by generating a random account instead:** a random one would meet the never-reuse rule and
still have to be claimed some other way.

**C21 — ephemeral account management is restart- and multi-device-safe.** Fresh generation, durable
journalling before submission, resume-on-restart, and no reuse across operations. The account's id
is derivable client-side before the deshield lands, which is what makes the same-block mitigation of
§3.5.35 possible.

#### 3.5.35 D-7 private path cost

**This is the finding that argues against the mechanism we are bidding on. We volunteer it, with
numbers.**

A private buy is three transactions; a public buy is one. On a bonding curve **price rises with
every buy**, so whatever volume lands in the gap between a private buyer's deshield and their buy is
paid for by that buyer — the price impact of the intervening volume. Buy = 1 SOL, 1% fee, canonical
pump.fun-shaped parameters:

| Curve position | 1 buy ahead | 3 ahead | 10 ahead |
| :---- | ---: | ---: | ---: |
| At sale open (`Vc` = 30 SOL virtual) | **6.19%** | 16.97% | 43.02% |
| After 30 SOL raised | 3.21% | 9.18% | 26.25% |
| After 200 SOL raised | 0.86% | 2.55% | 8.13% |

These assume the corrected fee reading (`Vc += eff`, per DEV-2). Under F1's literal `Vc += C_in` the
exposure is marginally **worse**, not better — crediting the gross moves the curve further per buy — and
the difference is immaterial: **6.22 / 17.04 / 43.14** at open, the 30-SOL row unchanged to two decimals,
0.86 / 2.54 / 8.10 after 200 SOL. The finding does not turn on which fee reading is taken.

**Why this is a design tension worth naming.** The RFP acknowledges it only inside a **soft**
requirement — the optional end timestamp, which "must enforce a minimum sale duration … long enough
that latency in the deshield→buy→re-shield privacy path does not systematically disadvantage
private-path buyers on price." The disadvantage exists whether or not an end timestamp is
configured, and **no hard requirement addresses it.** A privacy-preserving launchpad whose mechanism
charges a premium for the privacy path deserves to have that said in the open — and the effect is
largest exactly where privacy matters most, at sale open. **We implement that soft requirement, and its
minimum-duration floor, for exactly this reason** (C26): having quantified the penalty, skipping the one
mitigation the specification offers would be an odd place to stop.

This is not a defect in any implementation but a property of a rising-price mechanism combined with a
multi-transaction privacy path, and no implementation can remove it. What one can do is reduce the
gap, disclose the exposure, and give the creator a tool to blunt it.

##### C22 — three reductions, one disclosure, and the residue that cannot be removed

**Three of the four below reduce the penalty; the fourth discloses what is left.** The residue is
structural rather than an implementation shortfall: price rises with every buy, and a private buy is
irreducibly three transactions, so a gap exists and volume can land in it. Removing it altogether
would take either a different pricing mechanism — which the RFP fixes — or the buy itself inside the
PPE transaction, which fails R1 under exactly the contention R1 exists for (C14, §3.5.19). So we
shrink the exposure, and we state what remains rather than describing a solved problem.

1. **Submit the deshield and the buy in the same block.** Account A's id is derivable client-side
   before the deshield lands, so the buy transaction can be **built and submitted without waiting
   for the deshield's confirmation**. This collapses the exposure window from "one confirmation plus
   client round-trip" to "same-block ordering", which is the single largest available reduction. It
   is not free of risk — if the deshield is delayed the buy fails and must be resubmitted — so it is
   the default with an automatic fallback to sequential submission, and the failure is a clean
   revert (R2), never a partial state.
2. **The one-directional soft mode.** The RFP offers a creator-configurable option disabling
   sell-back during the sale window. We implement it, and we document a use it is not advertised
   for: it removes reflexive sell pressure, which reduces the volume that can land in a private
   buyer's gap. It is a creator's choice with a real cost (no exit liquidity before graduation), so
   we present the trade-off rather than recommending it.
3. **The end timestamp's minimum-duration floor.** The RFP requires that a configured end timestamp
   enforce a minimum sale duration long enough that privacy-path latency does not systematically
   disadvantage private buyers. We implement the timestamp and validate the floor at creation (C26), so
   the mitigation the specification names actually exists in the build rather than only in its text.
4. **Honest disclosure in the pre-buy summary (U4/Priv2).** The private-path summary shows the
   quoted fill and the exposure — that the executed price depends on what lands first, and by
   roughly how much at the sale's current position on the curve. The number is computable from
   exactly the state the summary already reads.

**We would rather lose points for raising this than win by not mentioning it.**

#### 3.5.36 The private path for **sells** — required by F3, specified nowhere

The Privacy block, like the rest of the RFP, is written for buys. Priv1 says
"deshield→**buy**→re-shield"; Priv2 says "before each **buy** operation"; Priv3 validates the
re-shield target "for **purchased** tokens"; Priv4 says "each **buy** from a private account". The
Privacy Architecture section and its step-by-step interaction flow describe buys only.

**F3 requires both.** It is the single place in the specification that generalises the pattern to
"deshield→**trade**→re-shield", and it makes both paths mandatory for the program *and* the SDK.

A private sell is the mirror image, and not a trivial one: deshield **project tokens** to a fresh
account A, sell on the curve, re-shield the **collateral proceeds**. Each Privacy requirement
acquires a second form:

| Req | Buy form (specified) | Sell form (required by F3, unspecified) |
| :---- | :---- | :---- |
| Priv1 | deshield collateral → buy → re-shield tokens | deshield tokens → sell → re-shield collateral |
| Priv2 | privacy summary before each buy | privacy summary before each sell |
| Priv3 | validate the re-shield target for **purchased tokens** | validate the re-shield target for **collateral proceeds** |
| Priv4 | fresh ephemeral per buy | fresh ephemeral per sell |
| U7 | shielded balance covers collateral + gas | shielded balance covers the **token** leg + gas |

**C23 — the saga engine is generic over trade direction**, so the sell path is the same machinery
with the assets swapped rather than a second implementation. That is what keeps this scope
incremental rather than doubling, and it is the reason we can carry the uncertainty cheaply instead
of pricing for the worst case.

**The private sell path is in scope** (M4, C6), because F3 says
"both paths" and "trade", and because a launchpad whose privacy covers only the entry is not a
privacy-preserving launchpad — a user who buys privately and then sells publicly has linked the
position they were protecting. It is a reading of an ambiguity rather than a certainty, so we say
which way we read it: if Logos intends buys only, this is 1.5–2 dev-weeks removed from M4 and nothing
else changes, not scope we quietly keep.

#### 3.5.37 Residual visibility we will document rather than let an auditor find

The RFP's "What is private" list states that "any link between multiple buys by the same buyer" is
protected. That is **true at the address level** — ephemeral accounts share no on-chain
relationship — and we want to record the one caveat that is not addressed by address unlinkability,
because S5 requires a truthful account of what an observer can infer.

A PPE transaction publishes its **public post-states** (`public_actions: Vec<PublicActionWithID>`). So account A's funding is observable, even though the private account that
funded it is not. An observer therefore sees: a fresh account funded with amount *X*, then that
account buying with approximately *X* shortly afterwards. What is cryptographically hidden is *which
private account* originated the funds. What is **not** hidden is the amount-and-timing pattern of
the operation itself, and a distinctive amount repeated across operations is a heuristic
correlation signal even without any address link.

To be clear about the strength of this: it is a **statistical heuristic, not a cryptographic link**,
and it weakens as the anonymity set grows — many concurrent private buys of similar size make it
useless. We document it in S5, we note that round-number amounts are the worst case, and we surface
it in the U5 disclosure rather than leaving a user to assume that "unlinkable" is unconditional. The
RFP's own Design Rationale asks for exactly this kind of honesty in the anonymity discussion, and it
is the sort of item that surfaces in an audit whether or not the bid mentions it first.

### 3.6 Design Decisions


- **C1 — Port the proven kernel; do not re-derive it.** The curve is `amm_core`'s constant-product
  math with virtual reserves layered on — every Reference Implementation formula is algebraically
  identical to a function Logos already ships (§3.5.2). Net-new is the five items in §3.5.3.
- **C2 — Close the sale with a partial-fill clamp (DEV-1).** The supply target is arithmetically
  unreachable under F1's must-revert rule. The remedy is `swap_exact_out_amounts`, whose
  round-up-at-both-steps guarantee is the direction F1 wants, and whose failure modes are all
  excluded — one by F2's own `Vt > D`, the other two by our creation-time envelope (§3.5.5).
- **C3 — Price and book on the post-fee amount (DEV-2), and accrue the fee rather than moving it in
  the trade.** The fee is never credited to the curve. The literal F1 reading is insolvent by
  1.0101% of the raise at a 1% rate, and it is F5's withdrawal that breaks first. The fee accrues in
  sale state and leaves on a separate authority-gated sweep, because a transaction carries one
  post-state per account and a trade that named the payer or the vault twice would lose a write
  silently (§3.5.6). The invariant under test becomes **vault = booked reserve + accrued fees**.
- **C4 — State the invariant as non-decreasing `k`, and enforce it on chain in that form.** Constant
  `k` is unachievable on integers and, as a specification, weaker than what we can guarantee:
  rounding residue accrues to the pool only. The check is a post-state assertion in the program on
  every operation, computed on the U256 product so it holds for parameter sets whose `k` is not
  representable in `u128` (§3.5.7) — strictly stronger than a property test, and it costs a
  comparison.
- **C5 — Collateral destination is a creation-time parameter, defaulting to literal F5.** F5 and the
  auto-graduation soft requirement contradict each other over the same assets (D-6); we make the
  resolution explicit and configurable rather than silently shipping either reading. The default is the
  hard requirement, because graduation execution is out of scope (OOS-1) and escrowing toward a path we
  do not build would lock the raise; the escrow setting is there for when RFP-004 lands. D-6 is handled
  by disclosure, not by overriding F5.
- **C6 — Both trade directions get both paths.** F3 says "trade", so the private sell saga is in
  scope, even though the rest of the RFP describes buys only.
- **C7 — Budget against executions, not cycles.** The public buy is 3 of 11; the private path is
  three transactions, not one, and the PPE prover's cap is 10, not 11. The curve itself is mul/div
  only, so cycles are never the binding constraint on this program.

- **C8 — Freeze the SPEL IDL first.** It is the interface contract that lets SDK, CLI, GUI and the
  analytics decoder proceed in parallel with program hardening (M1 onward).
- **C9 — Analytics do not depend on LP-0012.** Indexer calldata + `getAccountAtBlock` + intra-block
  replay through our own pricing function, with a documented migration to events when they land.
- **C10 — One core module, four consumers.** SDK, CLI, GUI and observer share pricing, sagas and
  the error taxonomy; a test asserts the CLI and GUI call the same entrypoints.
- **C11 — The saga engine is generic over trade direction.** Buy and sell are one implementation,
  so the private-sell scope is incremental cost rather than a second build.
- **C12 — The creator view solves for the parameters, not the other way round.** Creators enter the
  economics they want (`p₀`, graduation price, target raise); the module derives `Vt`/`Vc` and
  validates against the F2 envelope before submission.
- **C13 — Privacy guardrails live in the core, not the GUI.** U6's anti-linkage enforcement and
  U7's balance pre-check are core-module properties, so the CLI cannot reintroduce what the
  mini-app forbids.

- **C14 — The buy is a public transaction on both paths.** PPE transactions are pre-state-bound to
  every public account they touch and are dropped under contention; the curve PDA is the most
  contended account in the system. Keeping the buy public is what makes R1 and the privacy
  requirements simultaneously satisfiable.
- **C15 — Reliability is verified as invariants, not as assertions.** The three properties we test
  continuously are: `k` non-decreasing across every operation; sale-reserve monotonically
  non-increasing until close; and accounted reserves never exceeding vault balances (the solvency
  direction that survives unsolicited transfers into a vault).
- **C16 — Partial fill is a success path, not a failure path.** DEV-1's clamped terminal buy is
  atomic with its refund and its close, and is surfaced to the user as a completed order.

- **C17 — CU costs are a CI regression gate, not a one-off document.** `cycle_bench`'s JSON output
  is committed as an envelope; the build fails when an operation drifts outside tolerance.
- **C18 — Budget against executions and proving time.** Cycles are plentiful for a mul/div curve;
  the 11-public / 10-PPE invocation budget and client-side proving wall-clock are what constrain
  real flows.
- **C19 — Platform-gated milestones are scoped and triggered, not dated.** S6 and S7 carry fixed,
  pre-agreed scope and trigger on platform availability, keeping the 12-week body deliverable and
  the gated tail bounded.

- **C20 — Sagas are durably journalled and resumable.** "Non-skippable" must survive a crash, or it
  is not a guarantee. Resuming an interrupted saga is *completing* an operation, not reusing an
  account, so Priv4 permits recovery; the alternative reading strands user funds on a public account
  permanently.
- **C21 — Ephemeral account management is restart- and multi-device-safe**, with the account id
  derivable before the deshield lands, which is what enables the same-block mitigation.
- **C22 — D-7 is reduced, disclosed, and never claimed to be solved.** Three reductions —
  same-block submission, the one-directional mode, the minimum-duration floor — plus honest
  disclosure of the residue, which is structural and cannot be removed.
- **C23 — The saga engine is direction-generic**, so private sells (C6) are incremental scope.

- **C24 — Manual close is creator-authorised and time-constrained, and sold-back tokens re-enter the
  sale reserve.** P2 and S3 both presuppose a close that no Functionality requirement defines (D-3);
  unconstrained, with literal F5, it is a cheaper path to the D-6 rug. Authority is the creator,
  permitted only after a documented minimum open period, with the collateral destination governed by
  the same F5 parameter (C5) — both configuration.
- **C26 — Both close conditions ship: the supply target and the optional end timestamp.** DEV-1 makes
  the supply-target close reachable; the end timestamp bounds a sale that never reaches it, and carries
  the minimum-duration floor that is the specification's own D-7 mitigation. They are independent, not
  alternatives. Taking the timestamp is what makes the program depend on the clock, whose two caveats are
  bounded by a floor measured in days (§3.3).
- **C25 — The gas leg is built, and conditional on the live fee model.** The runtime does not charge
  fees yet (`TODO: Make this variable when fees are implemented`), so U1's atomic collateral+gas
  deshield and U7's combined pre-check are parameterised by the runtime rather than written against an
  assumed rate. Building both costs nothing extra; assuming either answer would.

### 3.7 Deviations & Readings


We request **three** deviations from hard requirements. All three are forced by defects in the
specification, all three are narrow, and all three are stated here rather than discovered during
delivery.

| # | Requirement | Deviation | Why it is necessary |
| :---- | :---- | :---- | :---- |
| **DEV-1** | **F1** — "if the computed `tokens_out` on a buy would exceed the remaining sale reserve, the transaction must revert" | The single terminal buy that exhausts the sale reserve is **partially filled** at exactly the remainder, with refund, instead of reverting. All other overshoot conditions still revert. | Without it, `tokens_out == remaining` is unreachable (D-1), so F4 never fires, **R3 is unsatisfiable**, and F5 leaves the entire raise permanently locked. |
| **DEV-2** | **F1** and the Reference Implementation — "the real collateral reserve increases by `C_in`" / "`Vc` increases by `C_in`" | Both increase by `eff = C_in − fee`, matching the RFP's own Fee Structure section. | The literal reading books collateral the vault does not hold — 1.0101% of the raise at a 1% fee (D-5) — and F5's "full real collateral reserve" withdrawal is unbacked by that margin. |
| **DEV-3** | **F6** — on sell, "the transaction reverts if the computed `C_out` is below this minimum" | The sell-side slippage floor is checked against the seller's **net proceeds** (`C_out_raw − fee`), not the gross `C_out` the pricing formula produces. | Under the accounting reading the specification's own rounding mandate implies, `C_out` is the gross (D-8). Checking a seller's minimum against a number they never receive under-protects by exactly the fee rate — **1.0000% at 100 bps**, simulated in §3.5.6. |

**Twelve ambiguities, and the readings we take.** Twelve open questions in the RFP change what gets
built. Each is a decision here, with its reasoning, rather than a question returned to Logos — so
anything we read wrongly is visible now rather than at the milestone it lands in. All are reversible
at kickoff as configuration or scope unless the row says otherwise.

| Ambiguity | The reading we take, and why | Decision |
| :---- | :---- | :---- |
| **F1's must-revert vs F4's supply-target close** | Jointly unsatisfiable on integer arithmetic (D-1). The clamp is the only resolution leaving F4, R3 **and** F5 deliverable, and its failure modes are all excluded — one by F2's own `Vt > D`, the other two by our creation-time envelope (§3.5.5). | **The clamp — DEV-1** (C2). *Not* configuration: the alternative abandons the supply-target close and re-gates withdrawal on a soft requirement, so it is M2's first scope item. |
| **The fee base on a buy** | The Fee Structure section is the only self-consistent reading; F1's literal `Vc += C_in` books 1.0101% of the raise the vault does not hold, and F5's withdrawal on a *successful* sale breaks first (D-5). | **Price and book on `eff = C_in − fee` — DEV-2** (C3). Reversible only into insolvency, so treated as settled. |
| **The fee base on a sell** | `C_out` is never disambiguated against the two quantities the Fee Structure defines. The gross is what F1's own rounding mandate implies and the only solvent reading (D-8, §3.5.6). | **Reserve and `Vc` move by the gross `C_out_raw`; seller gets the net; treasury the difference.** A stated interpretation under DEV-2's principle, not a fourth deviation (§3.5.6). |
| **What "computed and stored" binds for `k`** | The clause constrains the invariant's fixity and its visibility as curve state. Read instead as a representation mandate it is unimplementable at the top of the plausible range — an eighteen-decimal pair gives `k = 1e51` against a `u128::MAX` of `3.4e38` — and would force the program to refuse the sale. `amm_core` carries no `k` at all (§3.5.7). | **The invariant is fixed at creation and reported as state; nothing prices from a stored `k`.** Each formula folds into one `mul_div` with a U256 product — an exact identity, so no parameter set is refused on `k`'s magnitude. Not a deviation: fixity and visibility are both met. |
| **F6's `C_out` on a sell** | Under that reading F6's floor is checked against a number the seller never receives, under-protecting by exactly the fee rate. | **The floor is checked on net proceeds — DEV-3.** The sell quote shows gross, fee and net. |
| **Does F3's "trade" require private sells?** | F3 is the one place the RFP generalises to "trade", mandatory for program *and* SDK. Privacy covering only the entry is not privacy (§3.5.36). | **In scope** (C6, M4, §3.5.36). If Logos intends buys only: 1.5–2 dev-weeks off M4, nothing else changes (C11). |
| **Where the collateral goes at close** | F5 and the auto-graduation soft requirement contradict each other over the same assets; the literal reading leaves holders with no backing, no sell path and no DEX (D-6). | **A creation-time parameter defaulting to literal F5**, with a graduation-escrow setting as the alternative (C5, §3.5.9). The default is the hard requirement because graduation execution is out of scope; D-6 is handled by disclosure. No fourth deviation. |
| **Who may call the manual close** | P2 and S3 presuppose it; no Functionality requirement creates it (D-3). Unconstrained, with literal F5, it is a cheaper path to the same rug. | **Creator-authorised, only after a documented minimum open period, destination governed by the F5 parameter** (C24). Sold-back tokens re-enter the **sale reserve** (D-4). Authority and period are configuration. |
| **Is the gas fee model live?** | The runtime does not charge fees yet — `program/mod.rs` carries `TODO: Make this variable when fees are implemented` — so the gas leg's size is a runtime property, not a design choice. | **Build the gas leg, conditional on the runtime charging fees** (C25). That `TODO` is why the code is parameterised rather than written against an assumed rate; no rework either way. |
| **How S6 and S7 are scheduled** | Testnet 0.3 stands at **0 of 12** roadmap milestones with no published date. A dated commitment would assert what no Logos source supports. | **Fixed-scope, triggered by platform availability, outside the 12-week calendar** (C19, M7/M8). An internal target date just becomes the trigger; scope and price do not move. |
| **Does saga resumption count as Priv4 reuse?** | Priv4 forbids reuse *across operations*; completing the re-shield for an operation already begun is that same operation. The alternative forbids crash recovery and strands funds on a public account — the exposure Priv4 exists to prevent (§3.5.32). | **Crash recovery on that basis** (C20). M4's done gate asserts resumption generates no second ephemeral. |
| **What does U1's "atomic deshield … as a single indivisible user action" bind?** | Two properties are bundled in one clause: *atomic* over the collateral and gas legs, and *one user action* at the SDK surface. A single PPE transaction emitting both legs satisfies both readings, which is why we build that. | **We build the construction that satisfies both** (§3.5.12). Dormant today — with no user-paid gas (C25) the deshield is single-asset and atomic by construction. If gas lands **and** M0's marginal-cost sweep forces a split, the single-user-action property survives and cross-leg atomicity does not: **that becomes a declared deviation at that point**, recorded against this row rather than absorbed. |

### 3.8 Testing


RFP-015 imposes **no formal-verification requirement**; R1–R3 are behavioural. We therefore meet
them with a test programme rather than a proof programme, which is a deliberate scoping statement
and not an omission — we note it because our prior LEZ work carried machine-checked FV obligations
and this RFP does not, so we are not pricing a verification lane that no requirement asks for. The
invariants of C15 are nonetheless expressed as **property-based tests over generated operation
sequences** (`proptest`), which is the right instrument for them and is cheap.

| Requirement | Test | Lane |
| :---- | :---- | :---- |
| **R1** | Multi-buy sequences within a single block, asserting each buy prices against its true predecessor state; `k` non-decreasing — asserted on the U256 product, so the check holds for parameter sets whose `k` is not representable in `u128` — and reserve accounting exact across the sequence; concurrent buy + sell interleavings | Property tests + standalone-sequencer e2e |
| **R1** | A buy submitted against stale client state fails cleanly rather than executing at a stale price (the F6 slippage floor is the guard) | Unit + e2e |
| **R2** | Every deliberate revert path — slippage floor, insufficient balance, closed sale — leaves vault balances and curve state bit-identical to the pre-state | Unit + e2e |
| **R2** | Failure injected at each of the three executions in the buy chain; assert the sale account is **byte-identical** before and after in every case, not merely that the fields we thought to check are unchanged | Unit |
| **R1** | **Two-way solvency:** over generated buy/sell sequences interleaved with sweeps, the vault balance equals the booked collateral reserve plus accrued fees, exactly, and the reserve never exceeds the vault — the assertion that fails under either the D-5 or the D-8 losing reading, and fails *twice* under both | Property tests |
| **R3** | The terminal buy exhausts the reserve, closes the sale, and refunds — in one transaction, with no additional instruction | Unit + e2e |
| **R3** | Buys after close are rejected with the U10 message; the close is idempotent and cannot be re-triggered | Unit |
| **R3** | **Boundary regression:** the D-1 parameter set specifically, asserting the sale closes where a literal-F1 implementation would revert forever | Unit — this is the test that proves the defect is fixed |

The last row is the one we would point an auditor at first. It encodes the defect, the parameters
that exhibit it, and the fix, so that a future change which quietly reintroduces the must-revert
behaviour fails CI rather than shipping a launchpad whose sales cannot close.

S3 does not require tests for the Privacy block (§3.5.28). We test it anyway:

| Property | Test |
| :---- | :---- |
| Priv1 | No path through SDK, CLI or mini-app leaves a saga at step 2; kill-the-process-mid-saga test asserts the re-shield completes on restart |
| Priv3 | Re-shield to a public account is rejected **before** submission, with the explicit error |
| Priv4 | Every operation uses a freshly generated account; a test asserts no account id is ever reused across operations, and that saga resumption does not count as reuse |
| U6/Priv4 | No API path exists to fund an ephemeral account from an external address |
| C22 | Same-block deshield+buy submission succeeds; delayed-deshield fallback reverts cleanly (R2) and resubmits |
| D-7 | The disclosed exposure figure matches a simulated fill with *n* buys landing in the gap |

### 3.9 Traceability

All **35** hard requirements — the 34 numbered plus the unnumbered Supportability milestone mandate — are owned by at least one milestone with an objective acceptance test. Zero gaps. `R` = reuse of existing Logos code, `N` = net-new. Section references point to the mechanics above.

**Functionality**

| Req | How it is satisfied | | M |
| :---- | :---- | :---- | :---- |
| **F1** Buy/sell, SDK inverse, rounding, invariant | `amm_core` kernel + virtual reserves; inverse is `swap_exact_out_amounts`; terminal clamp; fee on `eff` (§3.5.5–6) | R+N · **DEV-1, DEV-2** | M1, M2 |
| **F2** Sale creation, `Vt`/`Vc`/`k`, two buckets | Creation instruction, PDA vaults, `spot_price_q64_64`; validation envelope added, and `k` recorded rather than used as an operand, so no parameter set is refused on its magnitude (§3.5.7) | N+R | M1 |
| **F3** Public **and** private paths, program + SDK | Public instructions; three-tx saga, fresh ephemeral per op; sagas for buy **and sell** (§3.5.8) | R+N · **C6** | M2–M4 |
| **F4** Auto-close on reserve exhaustion | Close in the exhausting buy's own execution — **reachable only via DEV-1** | N · **DEV-1** | M2 |
| **F5** Post-close creator withdrawal | Gated on close; **satisfied literally by default** — collateral reserve to the destination parameter (graduation-escrow alternative, C5), DEX seed reserve `R` in full, plus unsold `D` which F5 omits (§3.5.9) | N · **C5, DEV-2** | M2 |
| **F6** Slippage floors both directions | `min_tokens_out` / `min_collateral_out` on **net** proceeds (D-8); also guards clamped fills and non-executable sell quotes | N · **DEV-3** | M2 |
| **F7** ATAs for all token interactions | Delivered `programs/ata/`, and `create` is idempotent so a not-yet-existing vault is not a special case. LP-0014 landed; LP-0013's "transfer-authority" dependency is mis-specified and unneeded — PDA-owned vaults, as `programs/amm` already does (§3.3, §3.5.11) | R | M1 |

**Usability**

| Req | How it is satisfied | | M |
| :---- | :---- | :---- | :---- |
| **U1** SDK, both roles, both paths, atomic collateral+gas deshield | Core module + saga engine; deshield as one PPE tx emitting both legs; fresh derived ephemeral, uncontended and claimable by seed (§3.5.12, §3.5.19) | N on R primitives · **C6, C25** | M3, M4 |
| **U2** Mini-app, participant + creator views, Basecamp | QML+C++ on the core module; price-vs-supply chart closed-form; creator view solves for `Vt`/`Vc` (C12) | N | M3–M5 |
| **U3** CLI, essential ops both roles | SPEL-generated + thin wrapper; test asserts CLI/GUI entrypoint parity | R | M3 |
| **U4** Pre-buy summary incl. price impact and fee | `spot_price_q64_64` + `swap_exact_in_amounts` client-side — two of five values already exist; U4's price impact is computed as `(Vc+eff)²/Vc² − 1`, since `price_impact_bps` measures execution slippage instead (§3.5.14) | **R+N** · fee line needs **DEV-2** | M3 |
| **U5** Privacy disclosure before each buy | Generated from the saga's own account/message model; extended with the D-7 exposure | N | M4 |
| **U6** Enforce atomic deshield, no external funding | Core-module enforcement so the CLI inherits it (C13) | N | M4 |
| **U7** Shielded balance covers collateral + gas | Core pre-check, parameterised by the live fee model (C25); sell form because private sells are in scope (C6) | N · **C25** | M4 |
| **U8** Sale analytics, no identity linkage | Indexer calldata + `getAccountAtBlock` + intra-block replay, decoded by the U9 IDL — **no LP-0012 dependency** (C9, §3.5.17) | N (~1–1.5 wk) | M5 |
| **U9** SPEL IDL | Generated from the program definition; scheduled first as the parallelism contract *and* U8's decoder | R | M1 |
| **U10** Clear actionable errors | One taxonomy shared by GUI and CLI; adds dropped-PPE and partial-fill-success cases | N | M3 |

**Reliability · Performance**

| Req | How it is satisfied | | M |
| :---- | :---- | :---- | :---- |
| **R1** Consistent state under concurrent buys | Sequencer's sequential validate-then-apply; failed txs skipped; buy stays public on both paths (C14, §3.5.19) | R + N tests | M2 · tests M3 |
| **R2** Failed buy reverts atomically | All-or-nothing rollback across the buy's three executions, asserted as byte-equality of the sale account across the failure; failing txs never enter a block | R + N tests | M2 · tests M3 |
| **R3** Atomic auto-close, no extra instruction, no buys after | Close inside the terminal buy — **unsatisfiable as specified (D-1)**, restored by **DEV-1** (§3.5.21) | N · **DEV-1** | M2 · tests M4 |
| **P1** Buy within one LEZ transaction | Public buy = **3 of 11 executions**; private path is three transactions, PPE legs against a cap of 10 | R | M2 · measured M5 |
| **P2** Close within one LEZ transaction | Auto-close is free inside the terminal buy; manual close undefined by the RFP (D-3), so we specify it — creator-authorised and constrained | N · **C24** | M2 · measured M5 |
| **P3** Document CU cost per operation | `tools/cycle_bench` — Logos's own harness — pinned to version + SHA, **committed as a CI gate** (C17) | R + N | M5 |

**Supportability · Privacy**

| Req | How it is satisfied | | M |
| :---- | :---- | :---- | :---- |
| **(unnumbered)** Separate milestones for testnet 0.2, 0.3, mainnet | Three distinct milestones, the latter two gated (C19). Counted as the 35th hard requirement | — | M6 · M7 · M8 |
| **S1** Deployed and tested on testnet 0.2 | Terminal milestone of the 12-week programme; the 0.2 line live since 2026-08-02, v0.2.4 current | — | M6 |
| **S2** E2E vs standalone sequencer, CI green | Two CI lanes, build fails on red; lane shared with R1/R2 — built once | R | M0, M6 |
| **S3** ≥1 test per hard requirement in F/U/R/P (23) | CI-enforced coverage matrix — **two mandated cases untestable as specified** (D-1, D-3), restored by **DEV-1**/**C24** (§3.5.28) | N | M6 |
| **S4** README: deploy, addresses, both walkthroughs | Four sections, both surfaces, toolchain pinned; adds the C12 parameter-derivation guide | N | M6 |
| **S5** Privacy & anonymisation document | Four required contents from verified runtime behaviour; program-enforced vs client-enforced split (§3.5.30) | N | M6 |
| **S6** Verified on testnet 0.3 | Fixed-scope gated milestone (C19); carries the C9 analytics migration if LP-0012 has landed | N | **M7** (gated) |
| **S7** Deployed to mainnet | Fixed-scope gated milestone (C19) | N | **M8** (gated) |
| **Priv1** Both paths; complete pattern; re-shield not skippable | Saga as one user-level operation; durable journalling + resume (C20). Enforceable in our client only — as the RFP's own Trust Assumptions concede | N | M4 |
| **Priv2** Privacy summary before each buy | U5 surface, enforced in the core so the CLI inherits it; extended with D-7 and the amount/timing caveat | N | M4 |
| **Priv3** Validate re-shield target is shielded | Type-level check against `InputAccountIdentity::{Public, Private}`, before submission | R+N | M4 |
| **Priv4** Fresh single-use ephemeral, never reused | Fresh generation per operation; no path to nominate an existing account. Also a **liveness** property (C14, C21) | N · **C20** | M4 |

## Milestones, Payout and Timeline
## Shape of the plan

**M0–M6 sum to 12 weeks**, the upper bound of the RFP's stated 10–12 week estimate. We plan to the
ceiling rather than under it, because two of the three declared deviations (DEV-1, DEV-2) sit in M2's
scope, and if Logos rejects either we would rather absorb the re-scope inside the schedule than
renegotiate it.

**M7 and M8 sit outside the 12-week envelope on Logos's calendar, not ours.** Supportability
mandates separate milestones for testnet 0.2, testnet 0.3 and mainnet; S6 and S7 depend on releases
that do not yet have dates. Testnet 0.3 stands at **0 of 12 roadmap milestones with no published
target**, and observed platform cadence is roughly a quarter per release cycle. Both are therefore
**fixed-scope milestones triggered by platform availability** (C19), so the 12-week body
stays deliverable and the gated tail stays bounded.

**M9 is an audit programme, and it does not fit inside a 12-week build — so we do not claim it does.**
The RFP body states no audit requirement, but the programme's own proposal template asks for an audit
milestone where a project *"handles user funds, implements cryptographic primitives, or requires a
security review before mainnet deployment."* **This project does the first two and we treat the third
as following from them:** the program is immutable and custodies the whole of a sale's collateral, and
it executes inside a zkVM with a privacy circuit on the deshield path. The defects set out in §3.5 are
also exactly the class an independent reviewer is best placed to find a second time. The audit runs on its own calendar against the frozen post-M6 commit, with the
slot reserved at kickoff so the firm's lead time is absorbed before it binds. It is **not part of the
fixed price**: the firm's fee is billed at cost and our remediation is quoted once the firm is
engaged.

**Team:** three senior engineers on three streams throughout — **A** on-chain (Rust/RISC0/SPEL),
**B** client core, SDK, CLI and analytics observer, **C** mini-app and documentation — with the tech
lead carrying stream A, a project manager who also takes utility development across all three, and a
part-time advisor. Every milestone below carries work for all three streams, though not equally:
stream C's load is deliberately uneven (see the schedule risks). **The durations are calendar, not
dev-weeks.**

| Milestone | Deliverable headline | Duration | Payout |
| :---- | :---- | :---- | :---- |
| M0 | Deviation sign-off, scaffolding, CI, marginal-proving + observer spikes | ≈1.5 wk | **$12,000** |
| M1 | Curve core: pricing, virtual reserves, buckets, sale creation — **IDL frozen** | ≈2 wk | **$16,000** |
| M2 | Lifecycle: partial-fill clamp, auto-close, withdrawal, fee routing, slippage | ≈2 wk | **$16,500** |
| M3 | SDK + CLI, full public-path lifecycle, pre-trade summary, error taxonomy | ≈2 wk | **$16,000** |
| M4 | Private path: sagas, durability, Priv1–Priv4, disclosures, D-7 mitigations | ≈2 wk | **$16,500** |
| M5 | Analytics without LP-0012, reliability suite, CU benchmarks + CI gate | ≈1.5 wk | **$12,000** |
| M6 | **Testnet 0.2 deployment**, coverage matrix, README, privacy document | ≈1 wk | **$8,000** |
| **Build total (M0–M6)** | | **12 wk** | **$97,000** |
| M7 *(gated)* | **Testnet 0.3** verification + delta report + C9 analytics migration | ≈0.5 wk, on platform availability | **$1,500** |
| M8 *(gated)* | **Mainnet** deployment + deployment record | ≈0.5 wk, on platform availability | **$1,500** |
| M9 *(audit)* | **External tier-1 audit + remediation** — outside the build envelope | ≈4–6 wk calendar, ≈2 wk ours | **at cost + quoted separately** |

---

**Milestone 0 — Deviation sign-off, scaffolding, CI, and the two measurements that are actually open**

Payout: **$12,000** · Duration: ≈1.5 weeks

Deliverables: **a decision record confirming the three declared deviations and the twelve readings of
§3.7** — written as decisions with their reasoning, for Logos to accept or direct otherwise, with
**DEV-1 (the partial-fill clamp) and DEV-2 (the fee base) taken first** because both sit in M2's
scope and DEV-1 alone is not a configuration change; SPEL monorepo
scaffold (curve program, core module, CLI, mini-app module crates) under dual MIT + Apache-2.0; CI
with two e2e lanes — an in-process harness and a scoped standalone-mode sequencer job; **the
marginal-proving spike** (below); **analytics observer spike** against the live indexer, which validates
C9 before we commit to an LP-0012-independent path; mini-app shell loadable in Basecamp.

*Why this is not a latency benchmark.* That the private path costs minutes is settled by the platform —
`cycle_bench` documents real proving as ~minutes and PPE composition as ~an hour, and the outer circuit
recursively verifies every program in the chain (§3.5.12) — so re-measuring it tells us nothing we can act
on. **The open number is the marginal cost of the second deshield leg**, what U1's atomic collateral+gas
construction adds over a PPE transaction carrying one leg, and Logos's own harness already exposes it:
`cycle_bench --prove --ppe` reports end-to-end `execute_and_prove` wall time, `S_agg`, and **a chain-caller
depth sweep**. We run that sweep against our own entry program rather than invent a benchmark. With no
user-paid gas to deshield yet (C25), a token leg stands in for the gas leg, so the construction is proven
before gas becomes load-bearing. Two instrumentation commitments, because this is the measurement most
easily got wrong: per-leg **elapsed time reported separately from proving time**, so waiting is never
misreported as proving, and bounded polling with the budget recorded beside each figure.

*Every M0 gate closes on a published artifact, not on our word for it.* A milestone whose evidence is
a status report can only be audited by asking us. So each gate below names something committed to the
repository, and **every figure we publish carries the commit it was taken at, the pinned toolchain
versions, and the exact command that produced it** — including the figures that come back worse than we
would like, since a measurement published only when it flatters is not a measurement. Logos can re-run
any of them from a clean checkout without our involvement, and the same convention carries through M5's
per-operation table and M6's deployment record.

Done gate, each item an artifact in the repository at the M0 commit:

- **`spel generate-idl` succeeds**, an empty program builds to a guest binary, and the Basecamp module loads — the IDL and the build log committed, with the SPEL, RISC0 and `lee_core` versions pinned in the lockfile so the build is reproducible rather than merely reported.
- **CI green on the default branch with both e2e lanes** — the in-process harness and the scoped standalone-mode sequencer job. A skipped lane is not a passing lane, and the run is linkable.
- **The observer spike, as a re-runnable script**: it pulls a real transaction from the live indexer, decodes its `instruction_data` against the draft IDL, and reads the touched account via `getAccountAtBlock`. This is the gate that matters most at M0, because it demonstrates the U8 path **with LP-0012 absent** (§3.3) against the live chain rather than against our reading of it — so C9 is validated before M5 depends on it.
- **The chain-caller depth sweep from `cycle_bench --prove --ppe`**, committed with its raw output: a marginal cost per chained call, and a **go/no-go on the single-transaction two-leg deshield** recorded against §3.7's U1 reading, so a split becomes a declared deviation rather than a silent one. Proving time and waiting time reported separately, with the polling budget stated beside each figure.
- **The private path's asynchronous shape fixed in the mini-app shell** — per-leg progress, elapsed time, resumable — rather than deferred to M4, so the UX consequence of a minutes-long proof is settled while it is still cheap to change.
- **The decision record signed off or formally directed otherwise, minuted either way, with DEV-1 and DEV-2 resolved before M2 opens.** This is the one gate we cannot close alone, which is why it is first and why the schedule risks name it.

**Milestone 1 — Curve core: pricing, virtual reserves, accounting buckets, sale creation**

Payout: **$16,000** · Duration: ≈2 weeks

Deliverables: the constant-product kernel ported from `amm_core` with virtual reserves layered on
(F1's pricing, both directions, integer rounding against the trader); sale creation with the full
F2 parameter set, `k` computed and stored at creation, the two accounting buckets (sale reserve `D`,
DEX seed reserve `R`), and real deposit of `D + R` into program-owned PDA vaults; the **parameter
validation envelope** the RFP omits (§3.5.7); ATAs throughout (F7); **the SPEL IDL frozen and
published at the end of this milestone** (C8) — the interface contract that unblocks parallel client
work. Client streams begin against it: core-module quoting (B) and the creator-view parameter
derivation of C12 (C), which is closed-form and needs no chain.

Done gate: an algebraic parity test asserts our pricing matches `amm_core::swap_exact_in_amounts` /
`swap_exact_out_amounts` on shared vectors; a property test over generated operation sequences shows
`k` **non-decreasing** and never decreasing (C4); every degenerate parameter set in the validation
envelope is rejected at creation with a distinct error (`Vt ≤ D`, `D = 0`, `Vc` below the price
floor, `k` outside the working domain, fee rate at or above the denominator); `D + R` custody
verified in PDA vaults; **the IDL is frozen, published, and a client build consumes it.**

**Milestone 2 — Lifecycle: the clamp, auto-close, withdrawal, fee routing, slippage**

Payout: **$16,500** · Duration: ≈2 weeks

*Scope note: this milestone implements DEV-1, DEV-2 and DEV-3. If Logos directs otherwise on DEV-1
or DEV-2 at M0, the deliverables change and we re-baseline here rather than at the end — the DEV-1
alternative is scoped in §3.5.5.*

Deliverables: **the partial-fill clamp (DEV-1)** — the terminal buy fills exactly the remaining sale
reserve via `swap_exact_out_amounts`, with refund, replacing F1's revert on that one path; **F4
auto-close** atomically in the exhausting execution; **F5 withdrawal** gated on close, covering **all three
buckets**: the collateral reserve to the destination parameter (defaulting to literal F5, with the
graduation-escrow setting as the alternative, C5), **the DEX seed reserve `R`** to the creator in full
since nothing consumes it (OOS-1), and **unsold sale-reserve `D`** returned to the creator (§3.5.9);
**constrained creator-authorised manual close** (D-3, C24); **fee accrual plus the authority-gated
sweep to the treasury ATA** with the reserve and `Vc` moving by `eff` on a buy and by the gross
`C_out_raw` on a sell (DEV-2, D-8);
F6 slippage floors both directions, the sell-side floor checked on **net** proceeds (DEV-3);
sell-side bounding against the real reserve (D-4); **the creator-configurable one-directional mode**,
which C22 relies on as a D-7 mitigation and is one flag plus one check on the sell path; **the optional
end timestamp**, with the minimum-duration floor validated at creation inside the F2 envelope (C26). Streams B and C continue on SDK sagas and the participant view.

Done gate: **the D-1 boundary regression test** — on the canonical D-1 parameter set (§3.5.5), the
terminal buy fills the
remainder, refunds the unused collateral and closes the sale in one transaction, where a literal-F1
implementation reverts forever; a test asserts **the vault balance equals the booked reserve plus
accrued fees, exactly**, after 200 buys (DEV-2 — the literal reading is short by 1.0101% at a 1%
fee), and continues to hold across a sweep, so a sweep that took more than was accrued would fail it;
**the same three-term identity still holds after a full buy-then-sell cycle that clears the
position** (D-8 — the losing reading empties the vault before the position clears); withdrawal before close is rejected, and after close it pays the configured collateral
destination, returns the **DEX seed reserve `R`** in full and returns any **unsold sale-reserve `D`** —
asserted as all three vaults draining to exactly zero, so no bucket can be silently left behind;
slippage floors revert below `min_tokens_out` / `min_collateral_out`; a sell exceeding
the real reserve is bounded, not overdrawn.

**Milestone 3 — SDK and CLI: the full public-path lifecycle**

Payout: **$16,000** · Duration: ≈2 weeks

Deliverables: the client core module as the single source of truth — quoting, price impact,
pre-trade summary, transaction building, one error taxonomy (U10); the SDK's full lifecycle for both
roles (discover sales, price, impact, buy, sell, query position; create, close, withdraw), including
**F1's mandated inverse** (exact collateral cost for a requested quantity `Q`, which is
`swap_exact_out_amounts` directly); the CLI over the same entrypoints (U3); the U4 pre-trade summary
with spot price, exact quantity, price impact and the per-swap fee. Stream A hardens the program and
lands the R1/R2 test suites; stream C wires the participant and creator views to the live SDK.

Done gate: the CLI drives a complete sale lifecycle end-to-end against the standalone sequencer in
CI — create, buy, sell, terminal clamped buy, close, withdraw; **the U4 summary's predicted
`tokens_out`, price impact — U4's spot-price definition, not execution slippage — and fee match the
executed on-chain result exactly**, asserted per
transaction rather than by inspection; a test asserts CLI and mini-app call the same core
entrypoints; every U10 error code maps to exactly one message and suggested action, including the
partial-fill success case.

**Milestone 4 — The private path: sagas, durability, disclosures**

Payout: **$16,500** · Duration: ≈2 weeks

Deliverables: the deshield→trade→re-shield saga engine, **generic over trade direction** (C11) so
buy and sell are one implementation; the atomic collateral+gas deshield as one PPE transaction
emitting both legs (U1), with the two-leg split only if M0's marginal-cost sweep forced it;
**durable saga journalling and resume-on-restart** (C20) — the piece the RFP never specifies;
fresh single-use ephemeral accounts (Priv4, C21); re-shield target validation against
`InputAccountIdentity` (Priv3); the U5/Priv2 disclosure surface and the U7 shielded-balance
pre-check; **the C22 D-7 mitigations** — same-block deshield+buy submission with clean fallback, and
the price-exposure disclosure in the pre-trade summary. Stream A lands the R3 close tests and the
invariant suite.

Done gate: a private buy and a private sell each complete end-to-end; **the process is killed
between the trade and the re-shield, and on restart the client detects the incomplete saga and
completes it** — with a test asserting resumption does not generate a second ephemeral account
(C20); a re-shield to a public account is rejected *before* submission with an explicit error; no
ephemeral account id is reused across operations; there is no API or UI path that funds an ephemeral
account from an external address (U6); the same-block deshield+buy path succeeds, and a delayed
deshield reverts cleanly (R2) and resubmits.

**Milestone 5 — Analytics without LP-0012, reliability suite, CU benchmarks**

Payout: **$12,000** · Duration: ≈1.5 weeks

Deliverables: the U8 analytics observer — IDL-driven calldata decoding, per-block sale-PDA reads,
intra-block replay for per-trade quantities, and the four required series (collateral raised, spot
price, supply sold over time, price-vs-supply with current position) plus buy count, with **no
LP-0012 dependency** (C9) and a documented migration path; the analytics views in the mini-app; the
R1–R3 property and failure-injection suites completed; **P3 CU measurement via
`tools/cycle_bench`** across create, buy, close and withdraw, plus the PPE legs against the cap of
10 and proving wall-clock, pinned to a testnet version and commit SHA; **the cycle envelope
committed as a CI regression gate** (C17).

Done gate: the observer reconstructs a known sale's full history from indexer data alone, with
LP-0012 absent, matching on-chain state at every block; a block containing several buys is replayed
to recover each trade's `tokens_out` correctly; the analytics surface exposes no account-keyed
query; `cycle_bench` JSON is committed as the envelope and an injected cycle regression **fails
CI**; the R3 boundary regression from M2 runs in the property suite.

**Milestone 6 — Testnet 0.2 deployment, coverage, documentation**

Payout: **$8,000** · Duration: ≈1 week

Deliverables: **the program deployed and verified on LEZ testnet 0.2** (S1); the S3 coverage matrix
mapping all 23 hard requirements across Functionality, Usability, Reliability and Performance to at
least one test, CI-enforced; the S4 README — deployment steps, program addresses, and creator and
participant walkthroughs for both CLI and mini-app, plus the C12 parameter-derivation guide; **the
S5 privacy and anonymisation document** — what is visible, what is protected, trust assumptions
split into program-enforced versus client-enforced, what happens when a user bypasses the expected
path, the D-7 price exposure, and the residual amount/timing correlation caveat; mini-app build
instructions and downloadable assets (U2).

Done gate: deployed to testnet 0.2 with addresses published in the README; **all five of S3's
mandated test cases pass** — invariant preservation across multiple buys, happy-path buy, slippage
revert, **auto-close on supply target** (reachable only under DEV-1) and **manual close** (specified by us
under C24, since the RFP does not); the coverage matrix has zero unmapped requirements and CI fails if a mapped test is
missing or red; a reviewer following the README from a clean checkout deploys, creates a sale, buys
via CLI and via mini-app, and withdraws; the mini-app loads in Basecamp from the published git repo.

**Milestone 7 *(gated)* — Testnet 0.3 verification**

Payout: **$1,500** · Duration: ≈0.5 week · **Trigger: Logos publishes testnet 0.3**

Deliverables: redeploy and verify on testnet 0.3 (S6); full e2e suite re-run against the new
runtime; P3 cycle envelope re-measured and the CI gate updated; README addresses and version pins
updated; a delta report naming any behavioural change; **and, if LP-0012 has landed by then, the
documented C9 migration of the analytics ingestion layer to native events.**

Done gate: CI green against testnet 0.3; the delta report published; analytics unchanged in output
whether served by the replay observer or by native events.

**Milestone 8 *(gated)* — Mainnet deployment**

Payout: **$1,500** · Duration: ≈0.5 week · **Trigger: Logos opens mainnet deployment**

Deliverables: deployment to LEZ mainnet (S7); deployment verification; final published addresses and
a deployment record.

Done gate: the program is live on mainnet at published addresses, with a verified deployment record
and the README updated.

---

---

**Milestone 9 — Audit programme and remediation** *(audit milestone — runs **outside** the 12-week build envelope)*

Payout: external tier-1 audit fees **billed at cost** (pass-through to Logos); our remediation and
re-verification (≈2 wk) quoted separately once the firm is engaged · Duration: ≈4–6 weeks calendar
after the M6 freeze (firm-dependent), ≈2 weeks of it ours; the slot is reserved at kickoff (M0) so
firm lead time is absorbed before it binds

Deliverables: **primary tier-1 audit** — by one of our two partners, **[Zellic](https://www.zellic.io/)
or [Sherlock](https://www.sherlock.xyz/)**, selected at kickoff by availability — of the immutable
bonding-curve program and the client core, with four named focus areas, each one a place this document
already argues the specification is unsafe: **DEV-1's partial-fill clamp at the exact exhaustion
boundary** (§3.5.5), **the fee accrual and sweep accounting on both sides of the trade** (D-5, D-8,
§3.5.6), **the F5 withdrawal path and the collateral-destination parameter** (D-6, §3.5.9), and **PDA
vault custody across the buy's three chained executions** (§3.5.4). A **second independent review** of
the immutable program by a methodologically distinct firm is available on the same frozen commit if
Logos wants it, and is the one addition we would recommend. Consolidated remediation with auditor
fix-review, then full-suite re-verification against the M2 and M4 done gates.

Done gate: the audit report published with the codebase; **all critical and high findings resolved, or
formally accepted with written rationale**; CI green post-remediation, including the D-1 boundary
regression and the vault-equals-reserve-plus-accrued identity, before any mainnet-deployment
recommendation.

**M9 is numbered last because it is charged last, not because it runs last.** It opens at the M6
freeze and runs in parallel with M7, and its done gate is a **precondition for M8** — we would not
recommend a mainnet deployment of an immutable program that had not been through it.

## Schedule risks we are naming up front

| Risk | Effect | How the plan absorbs it |
| :---- | :---- | :---- |
| **DEV-1 refused** — Logos holds F1's must-revert literal | M2 re-scopes; R3 becomes formally unsatisfiable by anyone and the close moves to the end-timestamp route with F5 re-gated | Put to Logos at M0 as a decision with its alternative already scoped (§3.5.5), so the re-baseline lands in M2 rather than at delivery |
| **DEV-2 or DEV-3 refused** — the literal fee bases stand | The program books reserves the vault does not hold on both sides of the trade (D-5, D-8), and F5's withdrawal fails on a successful sale | We would not ship it silently: the deviation is declared, and the M2 done gate asserts vault-equals-reserve-plus-accrued in both directions, so a refusal is a specification decision on record rather than a defect we absorb |
| **Private sells directed out of scope** (our C6 reading declined) | −1.5–2 dev-weeks on M4; the U8 observer moves from B to C, who has headroom | The saga engine is direction-generic (C23), so the sell path is subtraction rather than rework; it is in the plan, so a reduction is schedule slack, not a renegotiation |
| **Mini-app heavier than estimated** | M3/M4 client streams slip | Stream C runs at ~6.5–7 of 12 weeks deliberately — this is where the buffer lives |
| **The second deshield leg is disproportionately expensive to prove** | U1's single-transaction atomic deshield splits into two PPE transactions | The *absolute* proving cost is a platform constant we design around from day one (asynchronous private path, §3.5.12), so only the **marginal** cost is at risk; measured at M0 with Logos's own chain-caller depth sweep. The split is pre-specified with its disclosed correlation window, and because it gives up cross-leg atomicity it would be **raised as a declared deviation** against §3.7's U1 reading, not absorbed. Dormant while there is no user-paid gas to deshield (C25) |
| **M0's decision record is not answered inside M0** — the RFP states Logos "typically respond within 14 days", and M0 is ≈1.5 weeks | M2 opens without DEV-1/DEV-2 settled, and M2 is where both land | M1 does not depend on either: it is the curve kernel, virtual reserves, buckets, creation and the frozen IDL, none of which either deviation touches. So M1 and M2 swap order if the answer is late, and the 12-week total is unchanged. We flag it because it is the one M0 gate we cannot compel |

## Total Requested Budget (USD)

**$100,000** — the fixed price for M0–M8: **$97,000** across the twelve-week build (M0–M6) plus
**$3,000** across the two gated deployment milestones (M7, M8), which sit on Logos's calendar rather
than ours.

**M9 is additional and not included in this figure:** the external tier-1 audit is **billed at cost
(pass-through to Logos)**, and our M9 remediation and re-verification (≈2 weeks) is quoted separately
once the firm is engaged. We would rather name the audit and leave its number open than fold a
placeholder for someone else's fee into a fixed price.

**So this figure is the sum of every milestone payout that has one.** The template asks that the total
equal the sum of the milestone payouts, and it does, for M0 through M8. M9 is the single exception and
it carries no figure to sum — quoting a number for a firm not yet engaged would make the total less
accurate, not more.

## Relevant Experience

**What we have built on this exact problem.** Vacuumlabs has delivered a **production bonding-curve
token launchpad** end to end. It began as a Solidity system on an EVM L2; we **rewrote the launch
programs for Solana in Rust with Anchor**, and extended the product well past the feature set it
started with. Three parts of it are the same parts RFP-015 asks for:

- **A bonding curve that a launch trades on until it completes**, with per-token curve state, buys priced against it, and a defined completion condition — the mechanism §3.5.5 and §3.5.9 are about.
- **Graduation of a completed launch into a constant-product DEX pool**, including a Meteora constant-product pool on the Solana side. RFP-015 treats auto-graduation as a soft requirement and we place it out of scope (OOS-1) because its execution depends on RFP-004 — but the reason C5 can specify a graduation-escrow destination precisely, and the reason §3.5.9 can state what a graduation hand-off actually needs, is that we have built the far side of it.
- **An indexer that reconstructs launch history from chain data** — curve state per token, every buy against the curve, and graduation events — and serves the product's analytics from it. That is U8's problem, and we solved it the same way C9 does.

The engagement is under NDA, so the description above is deliberately capability-level: no client, no
contract names, no parameters.

**A second launchpad, on Solana.** Vacuumlabs designed and built **Syndicate**, a fully decentralized
fair launchpad on Solana — both frontend and backend, on current Solana dApp tooling. Between the two,
the account-model shape this RFP runs on — PDAs, ATAs, IDL-driven clients, program-owned vaults — is
not something we would be meeting for the first time here.

**Constant-product AMM correctness, examined at invariant level.** We audited
**[WingRiders](https://github.com/vacuumlabs/audits/blob/master/reports/wingriders-v2-v1.0.pdf)**, a
Cardano constant-product AMM DEX, in a full-time engagement of roughly 17.5 person-days. The part that
bears on this proposal is the method rather than the report: we **reverse-engineered the pool's
on-chain pricing independently of its documentation**, extracted its treasury and fee handling, and
**checked the `token1 × token2 = k` invariant against live transaction data** instead of against the
team's account of it. That is the operation §3.5.6 performs on RFP-015's Reference Implementation, and
it is what surfaces the two defects a constant-product launchpad is likeliest to ship with: **an
invariant that does not hold as stated under integer rounding (D-2), and a fee credited in one place
and paid from another (D-5, D-8)**.

Also in the public [audits repository](https://github.com/vacuumlabs/audits): **Liqwid Finance** (the
Agora governance module), **Ardana** (stablecoin primitives) and **VyFinance** (NFT staking with a
custom on-chain RNG in Plutus). Lending, stablecoin and staking rather than AMM work — track record on
on-chain correctness, not domain-matched experience, and we do not present it as more than that.

**Rust systems in production, and indexer-derived state.** **[Autonom](https://www.autonom.cc/)** is a
trustless RWA price oracle Vacuumlabs designed and built from scratch: Rust operator nodes resolving
price requests concurrently across chains and DEXes, connected to pooled economic security through
restaking, live in production behind Adrena ALP2. **[CARP](https://github.com/vacuumlabs/carp)**, built
with dcSpark, is a Cardano chain-indexing and analytics service that reconstructs application-level
state from raw chain data — the same shape as the launchpad indexer above, and the same shape as C9.
We have also delivered multiple production generations of infrastructure and client tooling for
**[API3](https://api3.org/)** (Airnode, ChainAPI, an OEV solution, frontends), substantial work on
**[FunKit](https://fun.xyz/)** including its Solana infrastructure, and Midnight work covering wallet
wrappers, contract registries, transaction preparation, state observation and CLI tooling.

**One boundary we state rather than blur.** The four Cardano engagements were third-party code review,
not systems we authored. They evidence the analytical method this document is built on and genuine AMM
domain knowledge; they are not a claim to have shipped a production AMM of our own. The launchpad work
in the first two paragraphs is ours, built and delivered. Individual track records are linked in Team
Members.

## Post-Delivery Plan

Vacuumlabs maintains the project as a team — support is not tied to any single individual. Support runs through the public GitHub repository (issues and PRs) and the primary contact listed above. Delivery includes handover of everything an operator needs: the README deployment runbook, the published program addresses with their freeze-commit and image-id records, the M9 audit report and remediation record, and the per-operation compute-cost reports. The entire project is open-sourced under MIT and Apache-2.0, so every component is forkable and community-runnable rather than dependent on us.

We provide a **6-month post-delivery support window** from acceptance of the final milestone, **included in the total requested budget**, covering:

- **Defect warranty** — reproducible failures against the delivered test suite, fixed at no charge. A confirmed defect in the deployed bonding-curve program is shipped as a new versioned deployment through the same path as M6/M7/M8, with migration notes for the creators and holders of affected sales.
- **From the M6 testnet-0.2 delivery onward we track every LEZ and SPEL release.** Where a release affects the program or the client surface we port it and publish a compatibility delta **within 10 business days**; independently of releases we publish a compatibility status **at least quarterly**, for which "no action required" is a valid status. The client surface is cheap to keep current by construction: the SPEL IDL is generated and frozen in M1 (U9), and the CLI and Basecamp mini-app both consume the single client core module rather than reimplementing it (C8), so a runtime change is one regeneration plus one module rather than three parallel updates. The lockfile pins the LEZ and SPEL revisions we build against, and the platform-triggered M7/M8 milestones already establish the re-verify-and-redeploy mechanism this maintenance follows.
- **Analytics event-path activation** — when LP-0012's accepted event mechanism merges into mainline LEZ, the U8 observer switches from indexer calldata, `getAccountAtBlock` and intra-block replay to the native events, and the interim state-derived history is retired (C9). The observer's schema is fixed in M1 alongside the IDL, so this is activation, not redesign.
- **Security-report handling** — security-relevant reports follow a documented disclosure path and are **acknowledged within 48 hours** with a fix or a documented mitigation. A launchpad holds other people's collateral, so this is the one commitment that is not best-effort.
- **Integration support for early teams composing against the program.** The graduation seam exists precisely for this: close emits the graduation-relevant state — real collateral, seed reserve `R`, final spot — as a documented payload, so a DEX integration (RFP-004) or a downstream product can consume it without us reopening the program. The fee-rate and treasury admin authority is separable for the same reason.
- **Feature extensions beyond the RFP scope are separate engagements**, not assumed here.

**One handover artifact worth naming:** the M0 decision record — the three declared deviations and the twelve readings, with Logos's confirmation or direction against each — is a delivered artifact, so a future maintainer can see which behaviours were specification decisions rather than implementation choices.

**We can operate Synarton, not only deliver it.** Everything above covers the software itself - maintenance, defects, compatibility, and disclosure. Operating a launchpad is a separate responsibility: holding the relevant admin authority, monitoring live sales, and responding when something goes wrong.

If Logos would like us to take on that operational role, we are happy to scope and price it as a separate engagement. The people who would carry this responsibility are already part of [the team above](#team-members) - Uroš as CPO, 0xcr1st0f on business development and the rest of the development team. This same team covers Synarton's bonding-curve and LBP mechanisms if both are awarded to us.

## Permissions and Consent

- [x] I/We confirm Logos may contact me using the primary contact information provided above for follow-ups and next steps.
- [x] I/We consent to Logos using information from this proposal publicly such as blogs case studies social posts or analytical reporting. Redactions can be requested at any time.

## Program Requirements

- [x] I/We have read and agree to the Logos RFP Terms and Conditions and I understand that no Grant is awarded and no right to payment arises unless a Grant Agreement is executed.
- [x] I/We understand that RFP specifications are proposals rather than instructions, that Logos makes no representations as to their legal or regulatory treatment, and that we are responsible for assessing what we build, deploy or operate and for complying with the laws that apply to us.
- [x] I/We understand this project must be open-sourced under the MIT and Apache 2.0 Licenses unless explicitly approved otherwise.
- [x] I/We are prepared to deliver milestone-based outcomes.
