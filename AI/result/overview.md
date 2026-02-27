# Ethereum Vault Connector (EVC) Overview

## Core Purpose

Provide a single on‑chain gateway for users and contracts to:

- manage groups of 256 “accounts” under one owner,
- enable/disable vaults as **collaterals** or **controllers**,
- perform **checks‑deferred** operations (batching multiple calls with status verification),
- execute **owner/operator‑signed messages** via `permit`,
- and enforce fine‑grained **access control** across all of the above.

This greatly simplifies integration with a variety of vault protocols while encapsulating security patterns.

## Actors & Roles

- **Owner** (EOA/contract) – owns up to 256 accounts; configures operators, controllers, collaterals, modes.
- **Account** – one of 256 derived addresses; used to hold vault balances and authenticate actions.
- **Controller Vault** – a vault contract that, when enabled, can check an account’s status and control collaterals.
- **Collateral Vault** – a vault whose balances are managed under the current controller.
- **Operator** – address authorized by an owner to act for specific accounts (bit‑mask).
- **External caller** – any contract or user using EVC’s `call`, `batch`, `permit`, etc.
- **Vault itself** – calls back into EVC to perform its own status checks or forgive deferred checks.

## Contracts

- `EthereumVaultConnector.sol` – core logic, authentication, batching, permit, deferred‑check machinery.
- `TransientStorage.sol` – storage for temporary state used during checks‑deferred calls.
- `ExecutionContext.sol` – bit‑packed context flags (deferred, checks‑in‑progress, operator authenticated…).
- `Set.sol` – lightweight fixed‑size set library used for controllers/collaterals/temporary lists.
- Interfaces (`IEthereumVaultConnector`, `IVault`, `IERC1271`) define the extensible API surface.

## Terminology

- **Address prefix** – first 19 bytes of an address; groups 256 sibling accounts.
- **On‑behalf‑of account** – the target account checked/authenticated during a call.
- **Checks‑deferred call** – a call where account/vault status checks are postponed until the outermost call returns.
- **Control collateral** – invoking a collateral via the enabled controller; restricted to the single controller active during a batch.
- **Permit** – owner/operator‑signed message that lets off‑chain authorizations execute on‑chain.

## Key Invariants

- Each owner controls exactly one set of 256 accounts (prefix).
- At most 10 collaterals and 10 controllers may be enabled per account in transient context.
- Only one controller may be active when deferred checks are finally executed.
- Status checks must either succeed or the transaction reverts; deferred checks accumulate up to 10 unique addresses.
- Lockdown / permit‑disabled modes can only be toggled by the owner (not via permit or deferred calls).
- `permit` nonces are monotonic per prefix & namespace; setting to `max` invalidates all.

## Main assets

- `address(this).balance` (Ether forwarded through EVC)
- Lists of vault addresses stored per account: `accountCollaterals[account]`, `accountControllers[account]`
- Transient sets for deferred status checks.
- Owner/operator records & bit‑fields.

## Happy Paths

1. **Add collateral** – owner/operator → `enableCollateral(account, vault)`: record vault, perform account‑status check via current controller.
2. **Switch controller** – owner/operator → `enableController(account, vault)`: adds to list; if during checks‑deferred, mark it for later validation.
3. **Batch interaction** – user → `batch([...])`:
   1. Each item authenticates caller (owner/operator/controller).
   2. Calls external vaults while `executionContext` defers checks.
   3. Outer `batch` exit triggers status checks for all accounts/vaults seen.
4. **Signed execution** – owner signs a `permit` payload; any `sender` can call `permit()` and the EVC self‑calls with context set to the signer.
5. **Vault status check** – vault contract calls `requireVaultStatusCheck()` inside its logic to assert health; may later `forgiveVaultStatusCheck()` if appropriate.
6. **Forgive deferred account** – controller vault after a liquidation call executes `forgiveAccountStatusCheck` to remove an address from the pending set.

Admin actions (changing operators, nonce, lockdown) are separate from these flows.

## External Dependencies

- Vault contracts implement `IVault` and must honour `STATUS_CHECK_MAGIC_VALUE` for status checks.
- ECDSA/`IERC1271` used for permit signatures.
- Owners/operators manage keys off‑chain.
- No external oracles; all logic lives within vaults and EVC.

> 💡 The EVC acts as middleware enforcing cross‑vault access control and batching. Vaults plug in by calling status‑check methods or forgiving checks.
