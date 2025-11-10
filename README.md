# Sequence: Transaction Rails audit details
- Total Prize Pool: $18,500 in USDC
    - HM awards: up to $14,400 in USDC
        - If no valid Highs or Mediums are found, the HM pool is $0
    - QA awards: $600 in USDC
    - Judge awards: $3,000 in USDC
    - Scout awards: $500 in USDC
- [Read our guidelines for more details](https://docs.code4rena.com/competitions)
- Starts November 11, 2025 20:00 UTC
- Ends November 17, 2025 20:00 UTC

### ❗ Important notes for wardens
1. A coded, runnable PoC is required for all High/Medium submissions to this audit. 
    - This repo includes a basic template to run the test suite.
    - PoCs must use the test suite provided in this repo.
    - Your submission will be marked as Insufficient if the POC is not runnable and working with the provided test suite.
    - Exception: PoC is optional (though recommended) for wardens with signal ≥ 0.68.
1. Judging phase risk adjustments (upgrades/downgrades):
    - High- or Medium-risk submissions downgraded by the judge to Low-risk (QA) will be ineligible for awards.
    - Upgrading a Low-risk finding from a QA report to a Medium- or High-risk finding is not supported.
    - As such, wardens are encouraged to select the appropriate risk level carefully during the submission phase.

## V12 findings

[V12](https://v12.zellic.io/) is [Zellic](https://zellic.io)'s in-house AI auditing tool. It is the only autonomous Solidity auditor that [reliably finds Highs and Criticals](https://www.zellic.io/blog/introducing-v12/). All issues found by V12 will be judged as out of scope and ineligible for awards.

[V12 findings can be viewed here.](https://github.com/code-423n4/2025-11-sequence/blob/main/2025-11-sequence-v12-findings.md)

## Publicly known issues

_Anything included in this section is considered a publicly known issue and is therefore ineligible for awards._

- **In‑scope contracts:** `TrailsRouter`, `TrailsRouterShim`, `TrailsIntentEntrypoint`, `TrailsEntrypointV2`, and their libraries. Auditors can interact via the same public API patterns documented (multicall, sweep, injection, EIP‑712 deposit/permit, transfer‑suffix path).  
- **Out of scope:** The closed‑source **Intent Machine** (backend), while auditors can still hit the **[public API interfaces](https://docs.trails.build/api-reference/introduction)** and simulate the flows described in the flow docs.  
- **Context coupling:** These contracts are meant to operate with **Sequence v3 Sapient Signer wallets** (delegatecall extensions; attestation‑based validation); reviewers should model threats with that in mind.

# Overview

Multichain transaction rails to pay, swap, fund, or earn in 1-click with any wallet, token or chain - powered by intents.

## Summary

This transaction rails module is a chain abstraction orchestration protocol that enables 1-click transactions from any wallet seamlessly with unified user liquidity across all chains. In contrast to typical cross-chain solutions, it sources liquidity and aggregates all user balances for every token across every chain in a user’s wallet as options for any transaction on a destination chain. The module is architected as a trustless system that works on top of existing bridging, filler, or solver infrastructure. It is free to integrate and is optimized for a variety of use cases, for example:

- **Pay**: Enable cross-chain, 1-click payments with any token for ecommerce platforms, NFT marketplaces, real-world asset purchases, and minimal-slippage stablecoin transactions.
- **Swap**: Embed low-latency, highly liquid cross-chain token swaps for your applications.
- **Fund**: Maximize TVL and transaction velocity through fully brandable funding widgets for protocol deposits such as perpetual exchanges, chain deposits, and liquidity provisioning.
- **Earn**: Streamline DeFi yield opportunities by enabling deposits into tokenized vaults, lending pools, and yield strategies from any token on any chain

## Links

- **Previous audits:** [Quantstamp, October 2025](https://github.com/0xsequence/trails-contracts/blob/master/audits/quanstamp-audit-2025-10-23.pdf)
- **Documentation:** https://docs.trails.build/
- **Website:** https://trails.build/
- **X/Twitter:** https://x.com/0xsequence

---

# Scope

### Files in scope

| File                                        | nSLOC | 
|---------------------------------------------|-------|
| [src/TrailsIntentEntrypoint.sol](https://github.com/code-423n4/2025-11-sequence/blob/main/src/TrailsIntentEntrypoint.sol) | 101   | 
| [src/TrailsRouter.sol](https://github.com/code-423n4/2025-11-sequence/blob/main/src/TrailsRouter.sol) | 236   | 
| [src/TrailsRouterShim.sol](https://github.com/code-423n4/2025-11-sequence/blob/main/src/TrailsRouterShim.sol) | 30    |
| [src/guards/DelegatecallGuard.sol](https://github.com/code-423n4/2025-11-sequence/blob/main/src/guards/DelegatecallGuard.sol) | 12    |
| [src/interfaces/IMulticall3.sol](https://github.com/code-423n4/2025-11-sequence/blob/main/src/interfaces/IMulticall3.sol) | 18    |
| [src/interfaces/ITrailsIntentEntrypoint.sol](https://github.com/code-423n4/2025-11-sequence/blob/main/src/interfaces/ITrailsIntentEntrypoint.sol) | 5     |
| [src/interfaces/ITrailsRouter.sol](https://github.com/code-423n4/2025-11-sequence/blob/main/src/interfaces/ITrailsRouter.sol) | 25    | 
| [src/interfaces/ITrailsRouterShim.sol](https://github.com/code-423n4/2025-11-sequence/blob/main/src/interfaces/ITrailsRouterShim.sol) | 4     |
| [src/libraries/TrailsSentinelLib.sol](https://github.com/code-423n4/2025-11-sequence/blob/main/src/libraries/TrailsSentinelLib.sol) | 13    |

*For a machine-readable version, see [scope.txt](https://github.com/code-423n4/2025-11-sequence/blob/main/scope.txt)*

### Files out of scope

| File         |
| ------------ |
| [script/\*\*.\*\*](https://github.com/code-423n4/2025-11-sequence/tree/main/script) |
| [test/\*\*.\*\*](https://github.com/code-423n4/2025-11-sequence/tree/main/test) |
| Total Files: 17 |


*For a machine-readable version, see [out_of_scope.txt](https://github.com/code-423n4/2025-11-sequence/blob/main/out_of_scope.txt)*

# Additional context

## Areas of concern (where to focus for bugs)
### A. Delegatecall‑only router pattern
- **Delegatecall enforcement & assumptions.** `TrailsRouter` / `TrailsRouterShim` are designed to be invoked **only via `delegatecall`** from Sequence v3 wallets; direct calls are blocked (e.g., `onlyDelegatecall`). Probe for any call paths that bypass this constraint, or any place wallet‑context assumptions (storage layout, `msg.sender`) can be violated by unintended delegatecalls.  
- **Storage sentinels & `opHash` gating.** Success/failure is tracked via a per‑op storage sentinel keyed by `opHash`; mistakes in setting/clearing, hash collisions, or re‑use could gate fee sweeps incorrectly. Validate namespacing and slot computation (e.g., `TrailsSentinelLib.successSlot(opHash)`), including Cancun tstore vs. sstore fallbacks.  
- **Multicall3 behavior.** The router composes approvals, swaps, bridges via `IMulticall3.aggregate3Value`. Stress revert bubbling, partial‑success semantics (when upstream sets `behaviorOnError = IGNORE`), and ensure approvals can’t be stranded in a half‑updated state.

### B. Balance injection & calldata surgery
- **`injectAndCall` / `injectSweepAndCall`.** Calldata manipulation uses a fixed 32‑byte placeholder and a provided `amountOffset`. Focus on: offset correctness, alignment, endianness, fee‑on‑transfer tokens, and ETH vs ERC‑20 branches (value forwarding vs approval path). Look for out‑of‑bounds writes and incorrect placeholder detection.  
- **Approval handling quirks.** Uses `SafeERC20.forceApprove` (for USDT‑like tokens). Validate no approval race or leftover unlimited approvals after failure paths.

### C. Fee collection & refund semantics
- **Conditional fee sweeps.** `validateOpHashAndSweep(opHash, token, feeCollector)` should only fire when the success sentinel was set by the shim; verify there’s no path to set the sentinel on partial/incorrect success. Ensure `refundAndSweep` cannot under‑refund the user or over‑sweep to fees when origin calls fail.  
- **Destination failures.** When destination protocol calls fail, the intended behavior is to sweep funds to the user *on the destination chain* (no “undo bridge”). Validate this always occurs and can’t be front‑run/griefed into a stuck state.

### D. Entrypoint contracts (two surfaces)
- **`TrailsIntentEntrypoint` (EIP‑712 deposits + optional permits).** Review replay protection, deadline checks, nonces, and the “leftover allowance → `payFee` / `payFeeWithPermit`” pattern so fee collection can’t exceed expectations or happen without user intent. Check reentrancy guard coverage.  
- **`TrailsEntrypointV2` (“transfer‑first”, commit‑prove‑execute).** UX hinges on: (1) extracting the intent hash from the last 32 bytes of calldata on ETH deposits, (2) validating proof/commitment linkage, (3) correct status transitions (`Pending` → `Proven` → `Executed/Failed`). Audit proof validators and signature decoding, expiry logic, and emergency withdrawal gating.

### E. Sapient Signer modules (wallet‑side attestation)
- **Target pinning & `imageHash` matching.** For LiFi, calls must target the immutable `TARGET_LIFI_DIAMOND` and the attestation‑derived `lifiIntentHash` must match the leaf’s `imageHash`. Validate decoding and signer recovery over `payload.hashFor(address(0))`. Any bypass → arbitrary call authorization.

### F. Cross‑chain assumptions
- **Non‑atomicity & monitoring.** Origin/destination legs are decoupled by bridges/relayers. Stress timing windows, reorgs around proofing, dust handling, token decimal mismatches, and MEV on destination protocol interactions (especially with balance injection).

## Main invariants

**Router/Shim invariants**
- Router/RouterShim functions **execute only via `delegatecall`** from a Sequence v3 wallet context. Any direct call must revert via `onlyDelegatecall`.  
- A fee sweep using `validateOpHashAndSweep(opHash, …)` **must** observe `SUCCESS_VALUE` at the sentinel slot computed for that `opHash`; otherwise it reverts and **no fees are taken**.  
- Fallback refund path `refundAndSweep` **only** runs when the immediately previous step reverted under `behaviorOnError = IGNORE` (“onlyFallback” semantics). On success paths, fallback calls are skipped.  
- Balance injection (`injectAndCall`) **must** replace exactly the placeholder bytes at `amountOffset` and use the *current* wallet balance/allowance at call time (ETH via `value`, ERC‑20 via `forceApprove`)—never a guessed amount.

**State/sentinels invariants**
- The success sentinel slot is **namespaced** (no collisions with wallet storage) and keyed by `opHash`; it is set **only** after `RouterShim`’s wrapped call completes successfully.

**Economic invariants**
- On **origin failure**, the user is refunded on origin (funds never bridged), and fees—if collected—come only from remaining balances after refund logic (no user loss beyond quoted fees).  
- On **destination failure**, the user receives tokens on the destination chain via a sweep; no hidden fee collection occurs there beyond the defined sweep step.

**`TrailsIntentEntrypoint` invariants**
- Deposits (`depositToIntent` / `…WithPermit`) **must** match signed EIP‑712 intent (user, token, amount, intentAddress, deadline), with replay blocked by tracked intent hashes and deadline enforced. Reentrancy is guarded.  
- Fee payments (`payFee`, `payFeeWithPermit`) can only move `feeAmount` from the user to `feeCollector` when there is sufficient allowance **or** a valid ERC‑2612 permit for that exact amount by the deadline.

**`TrailsEntrypointV2` invariants**
- **ETH deposits** pull the intent hash from the last 32 bytes of calldata; the committed intent must match sender/token/amount/nonce and be within deadline before proofing. Status transitions are linear: `Pending → Proven → Executed/Failed`. Emergency withdraw is restricted to the deposit owner in Failed/expired states. All state changers are `nonReentrant`.

**Sapient Signer (wallet) invariants**
- For LiFi operations, **every call target** must equal the immutable `TARGET_LIFI_DIAMOND`; recovered attestation signer + decoded LiFi data must hash to the leaf’s `imageHash` for weight to count. Any mismatch rejects the signature.

## All trusted roles in the protocol

| Role | Surface | Authority / Notes |
|---|---|---|
| **Owner** | `TrailsEntrypointV2` | Admin actions incl. `pause`/`unpause` and ownership transfer; emergency/expiry rules enforced at function level. Public flows otherwise permissionless with validation. |
| **Deposit owner (user)** | `TrailsEntrypointV2` | Can `emergencyWithdraw(intentHash)` only when status is Failed or expired; cannot override happy‑path execution. |
| **Relayers / operators** | `TrailsEntrypointV2` | Call `commitIntent`, `prove*Deposit`, `executeIntent`. Unprivileged (validations gate success) but control **liveness/censorship** by choosing to act or not. |
| **Sapient signer keys** | Wallet side | Off‑chain keys that produce attestations. Trust is in correct wallet configuration (`imageHash`) and key custody; module pins LiFi diamond target. |

*(`TrailsRouter` / `TrailsRouterShim` execute under Sequence v3 wallet authority via `delegatecall`; there’s no standalone admin role on these stateless extensions.)*

## Running tests

### Prerequisites

The repository utilizes the `foundry` (`forge`) toolkit to compile its contracts, and contains several dependencies through `foundry` that will be automatically installed whenever a `forge` command is issued.

The compilation instructions were evaluated with the following toolkit versions:

- forge: `1.3.5-stable`

### Tests

This command can be issued to execute any tests within the repository:

```sh
forge test
``` 

### Submission PoCs

The scope of the audit contest involves three distinct contracts, each with its dedicated `**.t.sol` test file.

Wardens are instructed to utilize the respective test suite of the existing ones to illustrate the vulnerabilities they identify, should they be constrained to a single file (i.e. `TrailsIntentEntrypoint` vulnerabilities should utilize the `TrailsIntentEntrypoint.t.sol` file).

If a custom configuration is desired, wardens are advised to create their own PoC file that should be executable within the `test` subfolder of this contest.

All PoCs must adhere to the following guidelines:

- The PoC should execute successfully
- The PoC must not mock any contract-initiated calls
- The PoC must not utilize any mock contracts in place of actual in-scope implementations

## Miscellaneous

Employees of Sequence and employees' family members are ineligible to participate in this audit.

Code4rena's rules cannot be overridden by the contents of this README. In case of doubt, please check with C4 staff.


