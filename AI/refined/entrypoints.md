# Entry Point Analysis: ethereum-vault-connector

**Analyzed**: 2026-02-23T21:05:22.6371130+07:00  
**Scope**: `src/` (project contracts only; dependencies under `lib/` excluded)  
**Languages**: Solidity  
**Focus**: State-changing functions only (`view`/`pure` excluded)

## Summary

| Category                     | Count  |
| ---------------------------- | ------ |
| Public (Unrestricted)        | 3      |
| Role-Restricted              | 9      |
| Restricted (Review Required) | 4      |
| Contract-Only                | 4      |
| **Total**                    | **20** |

---

## Public Entry Points (Unrestricted)

State-changing functions callable by anyone (even if effect is conditional).

| Function                                     | File                                  | Notes                                                                                                                             |
| -------------------------------------------- | ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `disableController(address account)`         | `src/EthereumVaultConnector.sol:L576` | Removes `msg.sender` from `accountControllers[account]` if present; otherwise still triggers `requireAccountStatusCheck(account)` |
| `requireAccountStatusCheck(address account)` | `src/EthereumVaultConnector.sol:L875` | May update controller metadata timestamp when checks are not deferred                                                             |
| `forgiveVaultStatusCheck()`                  | `src/EthereumVaultConnector.sol:L915` | Removes `msg.sender` from deferred vault-check set (intended for vaults, but not enforced)                                        |

---

## Role-Restricted Entry Points

### Owner / Address-Prefix Owner

| Function                                                                         | File                                  | Restriction                                                                          |
| -------------------------------------------------------------------------------- | ------------------------------------- | ------------------------------------------------------------------------------------ |
| `setLockdownMode(bytes19 addressPrefix, bool enabled)`                           | `src/EthereumVaultConnector.sol:L327` | `onlyOwner(addressPrefix)`                                                           |
| `setPermitDisabledMode(bytes19 addressPrefix, bool enabled)`                     | `src/EthereumVaultConnector.sol:L348` | `onlyOwner(addressPrefix)`                                                           |
| `setNonce(bytes19 addressPrefix, uint256 nonceNamespace, uint256 nonce)`         | `src/EthereumVaultConnector.sol:L366` | `onlyOwner(addressPrefix)`                                                           |
| `setOperator(bytes19 addressPrefix, address operator, uint256 operatorBitField)` | `src/EthereumVaultConnector.sol:L385` | `authenticateCaller(addressPrefix, allowOperator=false, ...)` (owner-only semantics) |

### Owner / Operator (per-account)

| Function                                                                 | File                                  | Restriction                                                                              |
| ------------------------------------------------------------------------ | ------------------------------------- | ---------------------------------------------------------------------------------------- |
| `setAccountOperator(address account, address operator, bool authorized)` | `src/EthereumVaultConnector.sol:L416` | `authenticateCaller(account, allowOperator=true, ...)` + operator can only change itself |
| `enableCollateral(address account, address vault)`                       | `src/EthereumVaultConnector.sol:L488` | `onlyOwnerOrOperator(account)`                                                           |
| `disableCollateral(address account, address vault)`                      | `src/EthereumVaultConnector.sol:L507` | `onlyOwnerOrOperator(account)`                                                           |
| `reorderCollaterals(address account, uint8 index1, uint8 index2)`        | `src/EthereumVaultConnector.sol:L524` | `onlyOwnerOrOperator(account)`                                                           |
| `enableController(address account, address vault)`                       | `src/EthereumVaultConnector.sol:L557` | `onlyOwnerOrOperator(account)`                                                           |

---

## Restricted (Review Required)

Functions whose access is _intentionally conditional_ (e.g., signature-gated, or authentication is skipped on a specific path).

| Function                                                                                                                                      | File                                  | Pattern                                                        | Why Review                                                                                                                      |
| --------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- | --------------------- | ----------------------------------------------------------------------------------------------------- |
| `receive()`                                                                                                                                   | `src/EthereumVaultConnector.sol:L105` | `if (!executionContext.areChecksDeferred()) revert`            | Accepts ETH only during “checks-deferred” execution context; sending ETH directly otherwise reverts                             |
| `permit(address signer, address sender, uint256 nonceNamespace, uint256 nonce, uint256 deadline, uint256 value, bytes data, bytes signature)` | `src/EthereumVaultConnector.sol:L588` | Signature-gated + `sender == 0                                 |                                                                                                                                 | sender == msg.sender` | Public entrypoint controlled by offchain signature rules; also performs an EVC self-call with context |
| `call(address targetContract, address onBehalfOfAccount, uint256 value, bytes data)`                                                          | `src/EthereumVaultConnector.sol:L671` | Auth skipped when `targetContract == msg.sender`               | “Self-callback” path is permissionless; target contract must perform its own authentication if it relies on `onBehalfOfAccount` |
| `batch(BatchItem[] items)`                                                                                                                    | `src/EthereumVaultConnector.sol:L736` | Per-item auth skipped when `item.targetContract == msg.sender` | Same “self-callback” caveat as `call()` but per batch item                                                                      |

---

## Contract-Only (Internal Integration Points)

Entry points expected to be called by vault/controller contracts (trust-boundary surfaces).

| Function                                                                                            | File                                  | Expected Caller                                                                              |
| --------------------------------------------------------------------------------------------------- | ------------------------------------- | -------------------------------------------------------------------------------------------- |
| `controlCollateral(address targetCollateral, address onBehalfOfAccount, uint256 value, bytes data)` | `src/EthereumVaultConnector.sol:L700` | The _sole enabled controller_ for `onBehalfOfAccount` (`onlyController`)                     |
| `forgiveAccountStatusCheck(address account)`                                                        | `src/EthereumVaultConnector.sol:L884` | The _sole enabled controller_ for `account` (`onlyController`)                               |
| `requireVaultStatusCheck()`                                                                         | `src/EthereumVaultConnector.sol:L906` | The vault itself (`msg.sender`), since EVC calls `IVault.checkVaultStatus()` on `msg.sender` |
| `requireAccountAndVaultStatusCheck(address account)`                                                | `src/EthereumVaultConnector.sol:L925` | The vault itself (`msg.sender`), plus triggers account status checks for `account`           |

---

## Analysis Warnings / Notes

- **Slither not used**: Slither wasn’t available in this environment; results are from manual inspection of `src/`.
- **Externally callable but excluded (simulation-only)**:
  - `batchRevert(BatchItem[] items)` (`src/EthereumVaultConnector.sol:L764`): always reverts (used to surface simulation results).
  - `batchSimulation(BatchItem[] items)` (`src/EthereumVaultConnector.sol:L814`): intended to be non-mutating; performs a reverting `delegatecall` into `batchRevert` and decodes the revert payload.

---

## Files Analyzed

- `src/EthereumVaultConnector.sol` (20 state-changing entry points)
- `src/ExecutionContext.sol` (library; no external/public entry points)
- `src/Set.sol` (library; no external/public entry points)
- `src/TransientStorage.sol` (abstract storage; no external/public entry points)
- `src/Events.sol` / `src/Errors.sol` (events/errors only)
- `src/utils/EVCUtil.sol` (utility base; exposes only `view` external `EVC()`)
