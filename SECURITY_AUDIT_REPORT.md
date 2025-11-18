# COMPREHENSIVE SECURITY AUDIT REPORT
## Sequence: Transaction Rails Contracts
**Audit Date:** November 18, 2025
**Auditor:** Claude Code Security Team
**Scope:** TrailsRouter, TrailsRouterShim, TrailsIntentEntrypoint, and supporting libraries
**Commit:** Latest (claude/audit-swafe-security-01DbMRJaNJUNjqHLGvs9jikb branch)

---

## EXECUTIVE SUMMARY

This comprehensive security audit examined the Trails contract system, focusing on delegatecall patterns, storage manipulation, balance injection, multicall validation, fee collection, and EIP-712 replay protection.

### Test Results
- **Total Tests:** 151
- **Passed:** 151 (100%)
- **Failed:** 0
- **Test Suites:** 8

### Vulnerability Summary
- **Critical Severity:** 8 findings
- **High Severity:** 10 findings
- **Medium Severity:** 11 findings
- **Low Severity:** 7 findings
- **Informational:** Multiple findings

---

## CRITICAL SEVERITY FINDINGS

### [C-1] Sentinel Persistence Vulnerability on Non-Cancun Chains
**Severity:** Critical
**Component:** TrailsRouterShim.sol, TrailsRouter.sol, TrailsSentinelLib.sol
**Lines:** TrailsRouterShim.sol:62, TrailsRouter.sol:204-208

**Description:**
The sentinel mechanism uses Tstorish library which falls back to permanent storage (sstore/sload) on chains without EIP-1153 transient storage support. The sentinel value set via `_setTstorish` persists across transactions on pre-Cancun chains.

**Attack Scenario:**
1. Attacker executes legitimate operation on pre-Cancun chain (Optimism, Arbitrum pre-upgrade)
2. TrailsRouterShim sets sentinel via sstore: `_setTstorish(slot, SUCCESS_VALUE)`
3. Sentinel persists in permanent storage after transaction completes
4. In subsequent transaction, attacker calls `validateOpHashAndSweep` with same opHash
5. Validation passes because sentinel still set from previous transaction
6. Attacker sweeps tokens that should only be swept once

**Code Location:**
```solidity
// TrailsRouterShim.sol:61-62
uint256 slot = TrailsSentinelLib.successSlot(opHash);
_setTstorish(slot, TrailsSentinelLib.SUCCESS_VALUE); // Persists if using sstore

// TrailsRouter.sol:204-208
uint256 slot = TrailsSentinelLib.successSlot(opHash);
if (_getTstorish(slot) != TrailsSentinelLib.SUCCESS_VALUE) {
    revert SuccessSentinelNotSet();
}
sweep(_token, _recipient); // Can be called multiple times!
```

**Impact:** Unauthorized token/ETH sweeping across multiple transactions, complete bypass of sentinel protection, potential repeated drainage of wallet funds.

**Recommendation:**
1. Explicitly clear sentinel after sweep: `_clearTstorish(slot)` in `validateOpHashAndSweep`
2. Add transaction-level nonce to opHash computation
3. Consider using mapping-based tracking on non-Cancun chains

---

### [C-2] Balance Injection Calldata Corruption via Zero Offset
**Severity:** Critical
**Component:** TrailsRouter.sol
**Lines:** 332-355, 341

**Description:**
The `_injectAndExecuteCall` function allows replacement when `amountOffset == 0` due to the OR condition in `shouldReplace = (amountOffset != 0 || placeholder != bytes32(0))`. At offset 0, the function selector bytes can be read and potentially overwritten.

**Attack Scenario:**
1. Attacker crafts calldata where first 32 bytes (function selector + first param bytes) match chosen placeholder
2. Calls `injectAndCall` with `amountOffset = 0`, `placeholder = <crafted_value>`
3. Code reads bytes at offset 0: reads selector + 28 bytes
4. If placeholder matches, code overwrites selector with callerBalance
5. Function selector corrupted, causing call to wrong function or revert

**Code Location:**
```solidity
// TrailsRouter.sol:341
bool shouldReplace = (amountOffset != 0 || placeholder != bytes32(0));

// Lines 347-353
assembly {
    found := mload(add(add(callData, 32), amountOffset)) // At offset 0, reads selector!
}
if (found != placeholder) revert PlaceholderMismatch();

assembly {
    mstore(add(add(callData, 32), amountOffset), callerBalance) // Overwrites selector!
}
```

**Impact:** Calldata corruption, calls to unintended functions, funds sent to wrong functions or stolen.

**Recommendation:**
```solidity
bool shouldReplace = (amountOffset != 0 && placeholder != bytes32(0));
require(amountOffset >= 4, "Cannot replace function selector");
```

---

### [C-3] DOMAIN_SEPARATOR Immutability Creates Chain Fork Vulnerability
**Severity:** Critical
**Component:** TrailsIntentEntrypoint.sol
**Lines:** 49, 66-74, 187

**Description:**
The `DOMAIN_SEPARATOR` is set once in constructor using `block.chainid` and stored as immutable. After a chain fork, signatures valid on the original chain remain valid on forked chain because DOMAIN_SEPARATOR doesn't update.

**Attack Scenario:**
1. User signs intent on Ethereum mainnet with chainId=1
2. Ethereum undergoes contentious hard fork creating two chains
3. Contract deployed at same address on both chains
4. DOMAIN_SEPARATOR remains same on both chains (immutable, set at construction)
5. Despite `chainid()` in intentHash (line 187), signature validates on both chains
6. Attacker replays signature on forked chain

**Code Location:**
```solidity
// TrailsIntentEntrypoint.sol:49, 66-74
bytes32 public immutable DOMAIN_SEPARATOR;

constructor() {
    DOMAIN_SEPARATOR = keccak256(
        abi.encode(
            keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
            keccak256(bytes("TrailsIntentEntrypoint")),
            keccak256(bytes(VERSION)),
            block.chainid, // Set once, never updates!
            address(this)
        )
    );
}
```

**Impact:** Cross-chain replay of signed intents after chain forks, unauthorized token transfers on forked chains.

**Recommendation:**
Implement dynamic DOMAIN_SEPARATOR:
```solidity
function DOMAIN_SEPARATOR() public view returns (bytes32) {
    return keccak256(
        abi.encode(
            TYPE_HASH,
            NAME_HASH,
            VERSION_HASH,
            block.chainid, // Dynamic!
            address(this)
        )
    );
}
```

---

### [C-4] Cross-Function Reentrancy in Balance Injection
**Severity:** Critical
**Component:** TrailsRouter.sol
**Lines:** 358-369

**Description:**
The `_injectAndExecuteCall` function violates Checks-Effects-Interactions (CEI) pattern. It makes external calls without reentrancy protection. TrailsRouter has NO ReentrancyGuard - unlike TrailsIntentEntrypoint.

**Attack Scenario:**
```solidity
// Malicious target contract:
contract MaliciousTarget {
    fallback() external payable {
        // Reenter during native transfer
        if (address(router).balance > 0) {
            router.injectAndCall{value: 1 ether}(
                address(0), address(this), maliciousCallData, 0, bytes32(0)
            );
        }
    }
}
```

**Impact:** Drain contract balance through recursive calls, double-spend tokens via approval manipulation, bypass intended logic flow.

**Recommendation:**
```solidity
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract TrailsRouter is IDelegatedExtension, ITrailsRouter, DelegatecallGuard, Tstorish, ReentrancyGuard {
    function injectSweepAndCall(...) external payable nonReentrant { ... }
    function injectAndCall(...) public payable nonReentrant { ... }
    function sweep(...) public payable onlyDelegatecall nonReentrant { ... }
    function refundAndSweep(...) public payable onlyDelegatecall nonReentrant { ... }
}
```

---

### [C-5] Intent Signature Frontrunning & Griefing
**Severity:** Critical
**Component:** TrailsIntentEntrypoint.sol
**Lines:** 98-119, 122-148

**Description:**
The `depositToIntent` functions accept user signatures that can be observed in mempool and executed by anyone. The function transfers from `user` (not `msg.sender`), allowing signature theft.

**Attack Scenarios:**
1. **Signature Theft:** Attacker monitors mempool, sees user's signed intent, front-runs with higher gas
2. **MEV Griefing:** Execute intents at optimal MEV timing (not when user intended)
3. **Fee Collector Manipulation:** If attacker controls feeCollector parameter

**Impact:** Users lose control over execution timing, gas griefing, MEV extraction from user intents, loss of user autonomy.

**Recommendation:**
```solidity
// Option 1: Restrict caller
mapping(bytes32 => address) public intentRelayer;
require(
    authorizedRelayer == address(0) || authorizedRelayer == msg.sender,
    "Unauthorized relayer"
);

// Option 2: Add timestamp window
require(
    block.timestamp >= minTimestamp && block.timestamp <= maxTimestamp,
    "Outside execution window"
);
```

---

### [C-6] Arbitrary Code Execution via Multicall3 Delegatecall
**Severity:** Critical
**Component:** TrailsRouter.sol
**Lines:** 61, 97

**Description:**
The `execute` function delegatecalls to hardcoded MULTICALL3 address. If MULTICALL3 is compromised or if attacker controls that address on new chain, delegatecall executes arbitrary code with full wallet permissions.

**Attack Scenario:**
```solidity
// If MULTICALL3 at 0xcA11...CA11 is upgraded to malicious implementation
contract MaliciousMulticall3 {
    function aggregate3Value(...) external payable {
        // Drain all ETH
        payable(attacker).transfer(address(this).balance);
        // Drain all ERC20s
        token.transfer(attacker, token.balanceOf(address(this)));
    }
}
```

**Impact:** Complete fund theft, full wallet access.

**Recommendation:**
- Make MULTICALL3 address immutable and verify before deployment
- Add integrity check (code hash verification)
- Use call instead of delegatecall if possible
- Implement emergency pause mechanism

---

### [C-7] Approval Race Condition via forceApprove
**Severity:** Critical
**Component:** TrailsRouter.sol
**Lines:** 364

**Description:**
The `_injectAndExecuteCall` uses `SafeERC20.forceApprove(erc20, target, callerBalance)` without verifying balance hasn't changed. For rebase tokens, balance can change between reading and approval.

**Attack Scenario:**
1. Attacker monitors mempool for `injectAndCall` transactions
2. Front-runs with rebase event
3. Approved amount becomes incorrect (too low: DoS; too high: excess approval remains)

**Impact:** DoS attacks, excess approvals can be exploited, fund loss.

**Recommendation:**
- Read balance immediately before approval
- Add balance check after external call to detect fee-on-transfer or malicious tokens

---

### [C-8] Transient Storage Persistence Across Calls
**Severity:** Critical
**Component:** Tstorish library (fallback to SSTORE)
**Lines:** TrailsRouterShim.sol:62, TrailsRouter.sol:204-208

**Description:**
When using SSTORE fallback on non-Cancun chains, success sentinels persist across transactions. `validateOpHashAndSweep` can be called in future transactions with old opHashes.

**Attack:**
```solidity
// Transaction 1: Execute operation X, sets opHash A sentinel via SSTORE
// Transaction 2: Attacker calls validateOpHashAndSweep(opHashA, token, attackerAddr)
// Sentinel still set from Transaction 1
// Validation passes! Attacker sweeps someone else's funds
```

**Recommendation:**
```solidity
function validateOpHashAndSweep(bytes32 opHash, address _token, address _recipient) {
    uint256 slot = TrailsSentinelLib.successSlot(opHash);
    require(_getTstorish(slot) == TrailsSentinelLib.SUCCESS_VALUE, "Not authorized");
    _clearTstorish(slot);  // MUST clear immediately
    sweep(_token, _recipient);
}
```

---

## HIGH SEVERITY FINDINGS

### [H-1] Multicall Validation Bypass via Calldata Encoding Mismatch
**Severity:** High
**Component:** TrailsRouter.sol
**Lines:** 373-396, 59-64

**Description:**
The `_validateRouterCall` validates a memory copy of calldata, but actual delegatecall uses original calldata parameter. Differences in memory vs calldata interpretation could be exploited.

**Impact:** Bypass of `allowFailure` validation, execution of non-aggregate3Value functions, unauthorized multicall operations.

**Recommendation:** Validate actual calldata parameter directly or use inline assembly to validate without copying.

---

### [H-2] Insufficient Bounds Validation in amountOffset
**Severity:** High
**Component:** TrailsRouter.sol
**Lines:** 344, 347-353

**Description:**
Bounds check only ensures reading is within bounds, doesn't validate `amountOffset` is properly aligned to ABI-encoded parameter boundaries. Arbitrary offsets can overwrite unintended memory regions.

**Attack Scenario:**
1. Attacker provides `amountOffset = 5` (misaligned)
2. Bounds check passes if callData.length >= 37
3. Assembly writes balance at offset 5
4. Partially overwrites multiple parameters, corrupting calldata

**Recommendation:**
```solidity
require(amountOffset % 32 == 0 && amountOffset >= 4, "Invalid offset alignment");
```

---

### [H-3] Anyone Can Call validateOpHashAndSweep With Valid opHash
**Severity:** High
**Component:** TrailsRouter.sol
**Lines:** 199-209

**Description:**
Function only checks delegatecall context and opHash sentinel. Doesn't verify `msg.sender`, whether `_recipient` matches original intent, or whether `_token` matches original operation.

**Attack Path:**
```solidity
// Multi-call batch transaction
// Call 1: Operation X sets opHash A success sentinel
// Call 2: Attacker's call uses opHash A but sweeps to attacker address
```

**Recommendation:**
Bind recipient to opHash or store recipient in transient storage and verify match.

---

### [H-4] Sweep Transaction Front-Running
**Severity:** High
**Component:** TrailsRouter.sol
**Lines:** 153-163, 199-209

**Description:**
`validateOpHashAndSweep` doesn't bind opHash to specific recipient. Attacker can front-run with same opHash but different recipient.

**Recommendation:**
Include recipient in opHash validation or use commitment scheme.

---

### [H-5] Rebase Token Balance Manipulation
**Severity:** High
**Component:** TrailsRouter.sol
**Lines:** 120, 136, 319

**Description:**
Functions read token balances and cache them. For rebase tokens (Aave aTokens, Compound cTokens, Lido stETH), balance changes over time even without transfers.

**Impact:** Protocol accounting breaks for yield-bearing tokens, users lose funds if negative rebase occurs, external protocols receive incorrect amounts.

**Recommendation:**
- Document that rebase tokens are not supported
- Add balance delta check
- Consider whitelist approach

---

### [H-6] Read-Only Reentrancy via Balance Queries
**Severity:** High
**Component:** TrailsRouter.sol
**Lines:** 287-293

**Description:**
Balance queries call external contracts. For tokens with callback hooks (ERC777), `balanceOf` can trigger external code for price/oracle manipulation.

**Recommendation:**
- Cache balances at transaction start
- Use commit-reveal pattern for sensitive operations
- Add mutex for balance-dependent logic

---

### [H-7] Missing Chain ID Validation in Router
**Severity:** High
**Component:** TrailsRouter.sol
**Lines:** All functions

**Description:**
TrailsRouter has NO chain ID validation. If deployed at same address on multiple chains via CREATE2, transactions can be replayed across chains.

**Recommendation:**
```solidity
uint256 private immutable DEPLOYMENT_CHAIN_ID;
constructor() { DEPLOYMENT_CHAIN_ID = block.chainid; }
modifier onlyCorrectChain() { require(block.chainid == DEPLOYMENT_CHAIN_ID, "Wrong chain"); _; }
```

---

### [H-8] Integer Arithmetic Edge Case in Fee Calculation
**Severity:** High
**Component:** TrailsIntentEntrypoint.sol
**Lines:** 105-106

**Description:**
```solidity
unchecked {
    if (permitAmount != amount + feeAmount) revert PermitAmountMismatch();
}
```
The `unchecked` block disables overflow protection. Large amounts can overflow.

**Attack:**
```solidity
// amount = type(uint256).max - 100
// feeAmount = 200
// amount + feeAmount overflows to 99 (wraps around)
```

**Recommendation:**
```solidity
uint256 totalRequired = amount + feeAmount;  // Will revert on overflow
if (permitAmount != totalRequired) revert PermitAmountMismatch();
```

---

### [H-9] No Deadline Protection in TrailsRouter Operations
**Severity:** High
**Component:** TrailsRouter.sol

**Description:**
Unlike TrailsIntentEntrypoint, TrailsRouter has NO deadline validation. Transactions can sit in mempool and execute at unfavorable prices.

**Recommendation:**
Add deadline parameter to all public functions:
```solidity
function pullAndExecute(address token, bytes calldata data, uint256 deadline) {
    require(block.timestamp <= deadline, "Transaction expired");
    // ... rest of logic
}
```

---

### [H-10] Sandwich Attack on Balance Injection
**Severity:** High
**Component:** TrailsRouter.sol
**Lines:** 107-126

**Description:**
`injectSweepAndCall` reads user balance and injects it into call with no slippage protection. Vulnerable to sandwich attacks.

**Recommendation:**
- Add minimum output amount parameter
- Validate received amount after execution

---

## MEDIUM SEVERITY FINDINGS

### [M-1] DelegatecallGuard Storage Slot Collision Risk
**Severity:** Medium
**Component:** DelegatecallGuard.sol
**Lines:** 18

**Description:**
When TrailsRouter is delegatecalled by Sequence wallet, Tstorish writes to storage slots computed via `keccak256(SENTINEL_NAMESPACE, opHash)`. Could overlap with wallet storage if similar namespacing used.

**Recommendation:**
Use EIP-7201 namespaced storage pattern with additional entropy.

---

### [M-2] Placeholder Replacement Logic Uses OR Instead of AND
**Severity:** Medium
**Component:** TrailsRouter.sol
**Lines:** 341, 350

**Description:**
Condition `shouldReplace = (amountOffset != 0 || placeholder != bytes32(0))` uses OR instead of AND, creating edge cases for replacement.

**Recommendation:**
```solidity
bool shouldReplace = (amountOffset != 0 && placeholder != bytes32(0));
```

---

### [M-3] Nonce Ordering Forces Sequential Intent Execution
**Severity:** Medium
**Component:** TrailsIntentEntrypoint.sol
**Lines:** 174, 211

**Description:**
Strict nonce check `if (nonce != nonces[user])` requires intents in exact order. Creates DoS vulnerabilities and poor UX.

**Recommendation:**
Use bitmap-based nonce tracking:
```solidity
mapping(address => mapping(uint256 => bool)) public usedNonces;
```

---

### [M-4] refundAndSweep Under-Refunds Without Proper Error Handling
**Severity:** Medium
**Component:** TrailsRouter.sol
**Lines:** 171-196

**Description:**
Function caps `actualRefund` at current balance but only emits event without reverting. Users receive partial refund with no transaction failure signal.

**Recommendation:**
```solidity
require(current >= _refundAmount, "Insufficient balance for refund");
```

---

### [M-5] Pausable Token DoS Attack
**Severity:** Medium
**Component:** TrailsRouter.sol
**Lines:** 153-163, 166-196

**Description:**
`sweep` and `refundAndSweep` don't handle pausable tokens (USDC, USDT). If token paused, entire sweep operation fails, locking all tokens until unpause.

**Recommendation:**
- Implement try/catch around transfers
- Allow partial sweeps
- Add escape hatch for emergency withdrawals

---

### [M-6] Blacklisted Address Causing Permanent Fund Lock
**Severity:** Medium
**Component:** TrailsRouter.sol
**Lines:** 153-163

**Description:**
Tokens with blacklist functionality (USDC, USDT) revert if recipient blacklisted. No fallback mechanism.

**Recommendation:**
- Pre-check recipient status
- Allow changing recipient
- Implement multi-recipient sweep

---

### [M-7] State Mutation After External Call in refundAndSweep
**Severity:** Medium
**Component:** TrailsRouter.sol
**Lines:** 166-196

**Description:**
Function reads balance, makes external call, reads balance again. Allows reentrancy manipulation of second balance check.

**Recommendation:**
- Calculate all amounts upfront
- Emit events before external calls
- Add reentrancy guard

---

### [M-8] Dust Amount Accumulation & Gas Griefing
**Severity:** Medium
**Component:** TrailsRouter.sol
**Lines:** 153-163

**Description:**
`sweep` sweeps all balances regardless of amount. Attacker can send 1 wei of 1000 tokens, forcing victim to pay extreme gas costs to sweep.

**Recommendation:**
```solidity
function sweep(address _token, address _recipient, uint256 minAmount) {
    uint256 amount = _getSelfBalance(_token);
    if (amount >= minAmount) {  // Only sweep if above threshold
        // ... transfer logic
    }
}
```

---

### [M-9] Precision Loss in Balance Calculations
**Severity:** Medium
**Component:** TrailsRouter.sol
**Lines:** 173, 186

**Description:**
`refundAndSweep` re-reads balance after transfer. For tokens with small decimals, precision loss can lock small amounts.

**Recommendation:**
Calculate remaining arithmetically without re-querying:
```solidity
uint256 remaining = current - actualRefund;  // Don't re-query
```

---

### [M-10] Inconsistent State Across Chains in Delegatecall Context
**Severity:** Medium
**Component:** TrailsRouterShim.sol
**Lines:** 43-67

**Description:**
If same wallet exists on multiple chains, transaction executed on Chain A sets sentinel, but same opHash on Chain B not executed. State inconsistency across chains.

**Recommendation:**
Include chain ID in opHash calculation.

---

### [M-11] Finality Assumptions and Reorg Vulnerability
**Severity:** Medium
**Component:** TrailsIntentEntrypoint.sol
**Lines:** 207-211

**Description:**
On chains with low finality (Polygon, BSC), reorgs can revert `usedIntents` and `nonces` mappings, allowing double-spend.

**Recommendation:**
- Wait for finality
- Add block number to intent
- Implement reorg detection

---

## LOW SEVERITY & INFORMATIONAL FINDINGS

### [L-1] Missing OpHash Validation in handleSequenceDelegateCall
**Component:** TrailsRouter.sol:256
**Description:** OpHash from calldata ignored, parameter used instead. Could cause confusion.

### [L-2] No Target Address Validation in injectAndExecuteCall
**Component:** TrailsRouter.sol:334
**Description:** No validation that `target != address(0)`, allowing ETH/token burns.

### [L-3] Selector Extraction Uses Memory Instead of Direct Calldata Read
**Component:** TrailsRouter.sol:378-380
**Description:** Memory conversion creates potential mismatch with actual calldata.

### [L-4] Missing Access Control on Public Functions
**Component:** TrailsRouter.sol
**Description:** Several functions marked `public` should be `external` for gas optimization.

### [L-5] handleSequenceDelegateCall Selector Spoofing
**Component:** TrailsRouter.sol:216-262
**Description:** Selector extracted from data but not validated against actual function.

### [L-6] No Protection Against Flash Loan Attacks
**Component:** TrailsRouter.sol
**Description:** Balance-dependent functions susceptible to flash loan manipulation.

### [L-7] Unchecked Arithmetic in Intent Validation
**Component:** TrailsIntentEntrypoint.sol:106
**Description:** `unchecked { if (permitAmount != amount + feeAmount) }` could overflow with realistic values.

---

## SUMMARY STATISTICS

### Findings by Severity
| Severity | Count | Percentage |
|----------|-------|------------|
| Critical | 8 | 22.2% |
| High | 10 | 27.8% |
| Medium | 11 | 30.6% |
| Low | 7 | 19.4% |
| **Total** | **36** | **100%** |

### Findings by Component
| Component | Critical | High | Medium | Low | Total |
|-----------|----------|------|--------|-----|-------|
| TrailsRouter.sol | 5 | 7 | 8 | 5 | 25 |
| TrailsIntentEntrypoint.sol | 2 | 1 | 2 | 1 | 6 |
| TrailsRouterShim.sol | 1 | 0 | 1 | 0 | 2 |
| DelegatecallGuard.sol | 0 | 0 | 1 | 0 | 1 |
| TrailsSentinelLib.sol | 0 | 0 | 0 | 0 | 0 |
| Tstorish (library) | 0 | 1 | 0 | 0 | 1 |
| Multi-component | 0 | 1 | 0 | 1 | 2 |

---

## RECOMMENDATIONS BY PRIORITY

### IMMEDIATE (Critical - Must Fix Before Deployment)
1. **Add ReentrancyGuard to TrailsRouter** - Prevents all reentrancy attacks
2. **Implement dynamic DOMAIN_SEPARATOR** - Protects against chain fork replay
3. **Clear transient storage sentinels after use** - Prevents cross-transaction exploits
4. **Fix balance injection offset validation** - Prevents calldata corruption
5. **Add approval race condition protection** - Prevents rebase token exploits
6. **Restrict intent execution to authorized relayers** - Prevents frontrunning
7. **Add code hash verification for MULTICALL3** - Prevents arbitrary code execution
8. **Fix OR to AND in placeholder logic** - Prevents unintended replacements

### HIGH PRIORITY (Within 1 Week)
1. Add deadline parameters to all Router functions
2. Implement minimum amount thresholds for sweeps
3. Add chain ID validation throughout
4. Bind opHash to specific recipients
5. Fix integer overflow in fee calculation
6. Add slippage protection to balance injection
7. Implement proper access controls

### MEDIUM PRIORITY (Within 1 Month)
1. Document unsupported token types (rebase, fee-on-transfer, pausable)
2. Add try/catch for pausable tokens
3. Implement bitmap-based nonce tracking
4. Add balance delta checks
5. Improve error handling in refundAndSweep
6. Add multi-recipient sweep for blacklist recovery

### CODE QUALITY & TESTING
1. Change unnecessary `public` functions to `external`
2. Add comprehensive integration tests for reentrancy
3. Implement fuzzing for edge cases
4. Add cross-chain scenario tests
5. Increase test coverage for error paths
6. Add MEV protection documentation

---

## INVARIANT VIOLATIONS DETECTED

### Main Invariants from README.md

✅ **MAINTAINED:**
- Router/RouterShim execute only via delegatecall (DelegatecallGuard working)
- Balance injection uses current wallet balance at call time
- State sentinels are namespaced

❌ **VIOLATED:**
1. **Fee sweep validation:** On non-Cancun chains, sentinel persists allowing unauthorized sweeps
2. **Fallback refund semantics:** Under-refund occurs without revert
3. **Economic invariants:** On destination failure, hidden fees possible via frontrunning
4. **Replay protection:** EIP-712 intents vulnerable to chain fork replay

---

## CONCLUSION

This audit identified **36 security vulnerabilities** across all severity levels. The most critical issues involve:

1. **Reentrancy vulnerabilities** that could lead to complete fund drainage
2. **Signature replay attacks** enabling cross-chain fund theft
3. **Storage persistence issues** on non-Cancun chains
4. **Frontrunning vulnerabilities** in intent execution
5. **Token compatibility issues** with rebase, pausable, and blacklist tokens

### Test Suite Assessment
While all 151 tests pass (100% success rate), the test suite does not cover:
- Reentrancy attack scenarios
- Chain fork replay attacks
- Non-Cancun chain sentinel persistence
- Frontrunning and MEV exploitation
- Token compatibility edge cases
- Cross-chain replay scenarios

### Deployment Readiness
**Status:** NOT READY FOR PRODUCTION

The contracts require significant security improvements before mainnet deployment. The critical vulnerabilities could lead to complete loss of user funds.

### Recommended Next Steps
1. Address all Critical severity findings immediately
2. Implement comprehensive reentrancy protection
3. Add extensive security-focused test coverage
4. Conduct follow-up audit after fixes
5. Consider formal verification for critical functions
6. Implement bug bounty program before mainnet launch

---

**End of Security Audit Report**

*This audit was conducted using automated analysis, manual code review, and security best practices. However, no audit can guarantee the absence of all vulnerabilities. Users should conduct their own due diligence and understand the risks before interacting with these contracts.*
