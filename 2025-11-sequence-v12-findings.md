# Missing Zero-Address Validation Leading to Irreversible Token/ETH Burns
- Severity: Low

## Targets
- _transferERC20 (TrailsRouter)
- sweep and refundAndSweep (TrailsRouter)
- refundAndSweep (TrailsRouter)
- sweep (TrailsRouter)
- injectSweepAndCall / _injectAndExecuteCall (TrailsRouter)

## Description

Multiple functions in the TrailsRouter contract allow asset transfers to be directed to the zero address without any validation. Both ERC-20 transfers (via `_transferERC20`) and native ETH transfers (via `_transferNative` and low-level calls) in functions such as `sweep`, `refundAndSweep`, and `injectSweepAndCall` accept user-provided recipient or target addresses without checking that the address is non-zero. Sending to `address(0)` either burns tokens/ETH irreversibly or may revert unexpectedly, resulting in permanent loss of funds or disrupted workflows.

## Root cause

The contract omits explicit checks (e.g., `require(recipient != address(0))`) on user-supplied recipient or target addresses before executing ERC-20 or native asset transfers, enabling zero-address transfers.

## Impact

Attackers or users—either maliciously or by mistake—can specify `address(0)` as the recipient/target in any of the affected functions, causing irreversible burning of native ETH or ERC-20 tokens held by the contract or unexpected transaction reverts. This leads to permanent asset loss and potential denial of service for legitimate users.

---

# Blind Reliance on safeTransfer without Post-Transfer Balance Verification Leading to Fee-on-Transfer Token Misaccounting
- Severity: Low


## Targets
- _safeTransferFrom (used by pullAmountAndExecute and injectSweepAndCall) (TrailsRouter)
- refundAndSweep (TrailsRouter)
- sweep (TrailsRouter)

## Description

Multiple functions in the TrailsRouter contract (`_safeTransferFrom` used by pullAmountAndExecute and injectSweepAndCall, refundAndSweep, and sweep) rely solely on OpenZeppelin’s SafeERC20.safeTransfer or safeTransferFrom returning true, without verifying the actual change in token balances. Fee-on-transfer (deflationary) tokens deduct fees during transfers—and malicious tokens can falsely report success—so the router and its users may receive or forward fewer tokens than assumed. This leads to incorrect downstream logic, misleading events, locked residual balances (“dust”), and unaccounted-for fund losses.

## Root cause

The router’s transfer wrappers trust only the boolean return value of ERC-20 transfer calls and never reconcile balances before and after the transfer. They assume that a non-reverting, true-returning call always moves the full requested amount, ignoring fee-on-transfer token mechanics or malicious override of transfer semantics.

## Impact

1. pullAmountAndExecute/injectSweepAndCall may execute with less than the requested amount, breaking or skewing downstream calls and potentially losing funds. 2. refundAndSweep can under-deliver refunds while emitting events that overstate delivered amounts, misleading recipients and contracts. 3. sweep can lose or lock tokens: part of the swept amount may be deducted as fees or maliciously withheld, and residual ‘‘dust’’ remains forever trapped in the contract. 4. Malicious tokens can exploit this to steal or withhold assets without detection.

---

# Inaccurate Sweep Event Logging for Fee-On-Transfer Tokens
- Severity: Low


## Targets
- sweep (TrailsRouter)

## Description

The Sweep event emits the router’s pre-transfer token balance as the `amount`, but fee-on-transfer or burnable tokens deduct fees during transfer, so the recipient’s actual received amount can be lower than the logged value. This leads to misleading on-chain event data.

## Root cause

Sweep calculates `amount` via `_getSelfBalance` before calling `_transferERC20`, then emits that original balance without verifying the net tokens received after fees. `_transferERC20` simply invokes `SafeERC20.safeTransfer` and does not capture post-transfer balances.

## Impact

Listeners and downstream systems that rely on the Sweep event for transfer accounting will record inflated transfer amounts for tokens with transfer fees or burns, potentially causing misreporting, analytics errors, or incorrect on-chain bookkeeping.

---

# Locked Ether in ERC-20 Branch of pullAndExecute
- Severity: Low


## Targets
- pullAndExecute (TrailsRouter)

## Description

When a user calls pullAndExecute with a non-zero msg.value and a non-zero token address (an ERC-20 token), the function ignores the ETH sent. The ERC-20 branch never validates or refunds msg.value, so any ETH included is retained by the contract permanently unless a privileged sweep or refund function is invoked.

## Root cause

The code only checks and uses msg.value when token == address(0); in the ERC-20 branch, msg.value is neither validated nor refunded, and downstream logic (_safeTransferFrom and delegatecall) makes no use of it.

## Impact

Users can accidentally lock ETH in the TrailsRouter contract with no way to retrieve it, potentially losing funds until a privileged actor executes a sweep or refund function. This may lead to unexpected loss of ETH and reputational damage.

---

# Unhandled Transfer Failures Enable Denial-of-Service in Sweep and Refund Operations
- Severity: Low


## Targets
- _transferERC20 (TrailsRouter)
- sweep (TrailsRouter)

## Description

The TrailsRouter contract’s sweep and refundAndSweep flows rely on internal transfer functions that immediately revert on any transfer failure—whether from native ETH transfers to recipient contracts or ERC-20 transfers via SafeERC20.safeTransfer—without any error handling or fallback logic. This design allows a malicious or misbehaving token or recipient contract to force a revert and block the entire operation.

## Root cause

Both `_transferNative` and `_transferERC20` in TrailsRouter unconditionally bubble up transfer reverts or failures, lacking try/catch, return-value checks, or alternative recovery paths.

## Impact

An attacker controlling a recipient contract or deploying a non-standard ERC-20 token (whose `transfer` always reverts or returns false) can indefinitely block `sweep` and `refundAndSweep` calls, locking funds in the router and preventing legitimate users from retrieving assets (Denial-of-Service).