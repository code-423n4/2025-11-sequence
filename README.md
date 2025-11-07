# Sequence audit details
- Total Prize Pool: XXX XXX USDC (Airtable: Total award pool)
    - HM awards: up to XXX XXX USDC (Airtable: HM (main) pool)
        - If no valid Highs or Mediums are found, the HM pool is $0 (🐺 C4 EM: adjust in case of tiered pools)
    - QA awards: XXX XXX USDC (Airtable: QA pool)
    - Judge awards: XXX XXX USDC (Airtable: Judge Fee)
    - Scout awards: $500 USDC (Airtable: Scout fee - but usually $500 USDC)
    - (this line can be removed if there is no mitigation) Mitigation Review: XXX XXX USDC
- [Read our guidelines for more details](https://docs.code4rena.com/competitions)
- Starts XXX XXX XX 20:00 UTC (ex. `Starts March 22, 2023 20:00 UTC`)
- Ends XXX XXX XX 20:00 UTC (ex. `Ends March 30, 2023 20:00 UTC`)

### ❗ Important notes for wardens
(🐺 C4 staff: delete the PoC requirement section if not applicable - i.e. for non-Solidity/EVM audits.)
1. A coded, runnable PoC is required for all High/Medium submissions to this audit. 
    - This repo includes a basic template to run the test suite.
    - PoCs must use the test suite provided in this repo.
    - Your submission will be marked as Insufficient if the POC is not runnable and working with the provided test suite.
    - Exception: PoC is optional (though recommended) for wardens with signal ≥ 0.68.
1. Judging phase risk adjustments (upgrades/downgrades):
    - High- or Medium-risk submissions downgraded by the judge to Low-risk (QA) will be ineligible for awards.
    - Upgrading a Low-risk finding from a QA report to a Medium- or High-risk finding is not supported.
    - As such, wardens are encouraged to select the appropriate risk level carefully during the submission phase.

## V12 findings (🐺 C4 staff: remove this section for non-Solidity/EVM audits)

[V12](https://v12.zellic.io/) is [Zellic](https://zellic.io)'s in-house AI auditing tool. It is the only autonomous Solidity auditor that [reliably finds Highs and Criticals](https://www.zellic.io/blog/introducing-v12/). All issues found by V12 will be judged as out of scope and ineligible for awards.

V12 findings will be posted in this section within the first two days of the competition.  

## Publicly known issues

_Anything included in this section is considered a publicly known issue and is therefore ineligible for awards._

## 🐺 C4: Begin Gist paste here (and delete this line)





# Scope

*See [scope.txt](https://github.com/code-423n4/2025-11-sequence/blob/main/scope.txt)*

### Files in scope


| File   | Logic Contracts | Interfaces | nSLOC | Purpose | Libraries used |
| ------ | --------------- | ---------- | ----- | -----   | ------------ |
| /src/TrailsIntentEntrypoint.sol | 1| **** | 101 | |@openzeppelin/contracts/token/ERC20/IERC20.sol<br>@openzeppelin/contracts/token/ERC20/extensions/IERC20Permit.sol<br>@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol<br>@openzeppelin/contracts/utils/cryptography/ECDSA.sol<br>@openzeppelin/contracts/utils/ReentrancyGuard.sol|
| /src/TrailsRouter.sol | 1| **** | 236 | |@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol<br>@openzeppelin/contracts/token/ERC20/IERC20.sol<br>wallet-contracts-v3/modules/interfaces/IDelegatedExtension.sol<br>tstorish/Tstorish.sol|
| /src/TrailsRouterShim.sol | 1| **** | 30 | |tstorish/Tstorish.sol|
| /src/guards/DelegatecallGuard.sol | 1| **** | 12 | ||
| /src/interfaces/IMulticall3.sol | ****| 1 | 18 | ||
| /src/interfaces/ITrailsIntentEntrypoint.sol | ****| 1 | 5 | ||
| /src/interfaces/ITrailsRouter.sol | ****| 1 | 25 | |wallet-contracts-v3/modules/interfaces/IDelegatedExtension.sol|
| /src/interfaces/ITrailsRouterShim.sol | ****| 1 | 4 | |wallet-contracts-v3/modules/interfaces/IDelegatedExtension.sol|
| /src/libraries/TrailsSentinelLib.sol | 1| **** | 13 | ||
| **Totals** | **5** | **4** | **444** | | |

### Files out of scope

*See [out_of_scope.txt](https://github.com/code-423n4/2025-11-sequence/blob/main/out_of_scope.txt)*

| File         |
| ------------ |
| ./script/TrailsIntentEntrypoint.s.sol |
| ./script/TrailsRouter.s.sol |
| ./script/TrailsRouterShim.s.sol |
| ./test/TrailsIntentEntrypoint.t.sol |
| ./test/TrailsRouter.t.sol |
| ./test/TrailsRouterShim.t.sol |
| ./test/guards/DelegatecallGuard.t.sol |
| ./test/libraries/TrailsSentinelLib.t.sol |
| ./test/mocks/MockERC20.sol |
| ./test/mocks/MockMulticall3.sol |
| ./test/mocks/MockNonStandardERC20.sol |
| ./test/mocks/MockSenderGetter.sol |
| ./test/script/TrailsIntentEntrypoint.s.t.sol |
| ./test/script/TrailsRouter.s.t.sol |
| ./test/script/TrailsRouterShim.s.t.sol |
| ./test/utils/Create2Utils.sol |
| ./test/utils/TstoreUtils.sol |
| Totals: 17 |

