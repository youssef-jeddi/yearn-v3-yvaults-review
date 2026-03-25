```
---
chain: "ethereum"
stage: 1
reasons: []
risks: ["L", "H", "M", "H", "M"]
author: ["youssef"]
submission_date: "2026-03-25"
publish_date: "2026-03-25"
update_date: "2026-03-25"
---
```

# Summary

Yearn V3 is a yield aggregation protocol built around yVaults, ERC-4626 compliant smart contracts that accept user deposits and generate yield by allocating funds across multiple yield-generating strategies. Strategies deploy assets to external DeFi opportunities such as lending protocols, liquidity pools, or other yield sources. Users receive vault shares proportional to their deposit, which can be redeemed at any time for the underlying asset plus any earned yield. The V3 vault system is permissionless, allowing anyone to deploy and manage their own vault. Yearn operates its own set of vaults through a dedicated governance structure.

# Ratings

## Chain

Yearn V3 is deployed on different chains. This review is based on the Ethereum mainnet deployment.

> Chain score: Low

## Upgradeability

The vault contracts themselves are immutable : they are deployed as clones from an implementation contract ([`Vault Original`](https://etherscan.io/address/0xd8063123BBA3B480569244AE66BFE72B6c84b00d)), with no proxy pattern and no mechanism to change their bytecode after deployment. This is a strong positive for decentralization.

However, the DADDY position (the ychad.eth 6/9 multisig acting without a timelock) holds ALL roles on every Yearn-managed vault. This grants DADDY the ability to immediately, with no delay:

- **Force-revoke strategies** (`force_revoke_strategy`): Writes off a strategy's entire debt as a loss, which can destroy user funds in some cases.
- **Replace the Accountant** (`set_accountant`): Swap the fee engine with another contract.
- **Shut down vaults** (`shutdown_vault`): Permanently disables deposits.
- **Move funds between strategies** (`update_debt`): Redirect user capital to any strategy.

Adding new strategies is gated behind the STRATEGY_MANAGER position (the Timelock), providing a 7-day delay for this specific action. But the combination of force-revoke and accountant swap means the DADDY multisig can affect user funds without any delay.

> Upgradeability score: High

## Autonomy

The vault contracts themselves do not rely on oracles, they do not perform price-based operations. This is a strong positive.

However, strategies deployed by the vaults invest user funds into external DeFi protocols (lending markets, liquidity pools, etc...). If an underlying protocol is exploited or fails, the vault's strategy could experience losses. These are indirect dependencies inherent to the yield model.

The Keeper (a yHaaS relayer contract) is responsible for triggering `process_report` to realize profits/losses. However, this function can be called by anyone with the `REPORTING_MANAGER` role, and the Keeper position can be reassigned, so there is no single point of failure for reporting.

The strategies dependency on external yield sources means failures could impact unclaimed yield and potentially user principal (depending on the specific strategy). The external protocols themselves vary in decentralization level, but the core vault infrastructure has no oracle or external data dependencies.

> Autonomy score: Medium

## Exit Window

The governance architecture uses a two-layer system:

**Layer 1 = Timelocked (structural changes):** The Role Manager's [`governance`](https://etherscan.io/address/0x88Ba032be87d5EF1fbE87336B7090767F367BF73) is a TimelockController with a 7-day minimum delay. Changes to position holders, position roles, and governance transfer all go through this timelock. The proposer is the [ychad.eth](https://etherscan.io/address/0xFEB4acf3df3cDEA7399794D0869ef76A6EfAff52) 6/9 multisig. The executor is a custom [`TimelockExecutor`](https://etherscan.io/address/0xF8f60BF9456A6e0141149Db2DD6f02C60da5779B) contract governed by BRAIN (3/8 multisig), which restricts execution to two whitelisted addresses. The canceller role is held by the ychad.eth multisig. Adding/removing strategies via the STRATEGY_MANAGER position also goes through the Timelock.

Note: Because BRAIN controls the TimelockExecutor's governance, BRAIN can add or remove executors without going through the timelock.



**Layer 2 = Immediate (vault operations):** The DADDY position (same ychad.eth 6/9 multisig, but acting directly) can execute the most critical vault operations, force-revoking strategies, swapping the accountant, shutting down vaults, and moving debt, with no delay. Users have no advance warning or exit window for these actions.

Since the Upgradeability score is High AND the most impactful permissions are not protected by an exit window, this section receives a High risk score. However, the ychad.eth multisig qualifies as a Security Council, which satisfies the Stage 1 exception: "IF Exit Window receives High risk, THEN a Security Council must be in place with ownership of or veto over permissions."

> Exit Window score: High

## Accessibility

The primary user interface is [yearn.fi](https://yearn.fi). The frontend source code is open-source under a GNU GPL-3.0 license on GitHub ([github.com/yearn/yearn.fi](https://github.com/yearn/yearn.fi)), meaning anyone can fork and self-host it.

Users can also interact directly with the vault contracts via Etherscan's Read/Write Contract interface, Yearn's documentation includes a [guide](https://docs.yearn.fi/getting-started/guides/user-faq) for doing that. 

A single primary UI exists, but a self-hosting option is publicly available.

> Accessibility score: Medium

## Conclusion

The Yearn V3 (yVaults V3) deployment on Ethereum Mainnet achieves a Low centralization risk score for Chain, Medium for Autonomy and Accessibility, and High for Upgradeability and Exit Window. It thus ranks **Stage 1**.

The protocol could reach **Stage 2** by placing all critical vault operations (force-revoking strategies, accountant changes, vault shutdown, and debt management) behind the existing 7-day timelock or an on-chain governance process with at least a 30-day exit window.

# Reviewer's Notes

- **Scope:** This review focuses on Yearn V3 yVaults core infrastructure on Ethereum mainnet. It excludes other Yearn products.
- **Permissionless deployment:** Anyone can deploy V3 vaults via the VaultFactory. This review only covers Yearn-managed vaults (the 37 vaults tracked by the Role Manager).

Important Note : During the review period, on March 22, 2026, BRAIN (3/8 Strategist multisig) executed a [batch transaction](https://etherscan.io/tx/0x9aee2ca423085d94209e61be7ad93d828fe48b59a2af50581bba7cebee7fe03a/advanced) removing all 37 vaults from the Role Manager. This transferred the role_manager position on each vault directly to the ychad.eth multisig (DADDY), placing the vaults outside of the timelocked governance structure analyzed in this review. While this likely represents a migration to a new Role Manager, it confirms the centralization risks identified in our analysis: BRAIN which is a 3/8 multisig that does not meet Security Council requirements, was able to restructure vault governance for all Yearn V3 vaults without any timelock delay or user notification. We can note that users can still interact with the Vaults as the Vault contract itself is immutable.


# Protocol Analysis

The Yearn V3 architecture consists of several interacting contracts:

- **Vault (VaultV3.vy):** ERC-4626 compliant contract that holds user deposits, issues shares, and manages an array of strategies. Each vault is a clone deployed via the VaultFactory. Uses role-based access control system where the `role_manager` assigns specific roles to addresses.

- **TokenizedStrategy:** Base contract for strategies. Strategies are themselves ERC-4626 compliant vaults that deploy assets to a single external yield source.

- **Role Manager:** Central access control hub for all Yearn-managed V3 vaults. Deploys new vaults, assigns roles with `_sanctify()`, and manages vault lifecycle. Uses a two-step governance transfer pattern (`Governance2Step`).

- **Accountant:** Fee engine that calculates management and performance fees during strategy reports. Has hardcoded fee caps (management ≤ 2%, performance ≤ 50%). Does not hold user funds.

- **Vault Factory (VaultFactory.vy):** Deploys new vault clones from the implementation. Permissionless.

- **Registry:** Tracks all vaults on-chain. Used by frontends and integrators to discover vaults.

- **Debt Allocator:** Manages debt allocation ratios between strategies in a vault. Triggers debt updates when strategies differ from target ratios.

- **Keeper (yHaaS):** Relayer contract that triggers `process_report` on vaults to realize strategy gains/losses and distribute profits to depositors.

# Dependencies

**External yield sources (via strategies):** Each strategy deploys assets to one external protocol (Aave, Compound, Curve, etc.). If that protocol fails, the strategy and its vault depositors could suffer losses. These are inherent to the yield aggregation model and vary per strategy.

**No oracle dependency:** The vault contracts do not use oracles. They track strategy performance via `totalAssets()` reported by each strategy, not via external price feeds.

# Governance

## Role-Based Access Control

Yearn V3 uses a role system defined in the Roles library. Each vault maintains a mapping of `address => uint256` where each bit in the uint256 represents a specific permission. The Role Manager assigns these roles during vault deployment via `_sanctify()`.

The governance chain is:

```
ychad.eth Multisig (6/9 Gnosis Safe) → 0xFEB4...ff52
  ↓ proposes actions to
TimelockController (7-day delay) → 0x88Ba...BF73
  ↓ is governance of
Role Manager → 0xb3bd...9a41
  ↓ assigns roles on
All 37 Yearn V3 Vaults on Ethereum
```

Position holders assigned by the Role Manager:

| Position | Address | Type | Timelock? | Roles |
|----------|---------|------|-----------|-------|
| governance (Role Manager) | [`0x88Ba032be87d5EF1fbE87336B7090767F367BF73`](https://etherscan.io/address/0x88Ba032be87d5EF1fbE87336B7090767F367BF73) | TimelockController | YES (is the timelock) | Controls structural changes: setPositionHolder, setPositionRoles, transferGovernance, removeRoles |
| DADDY | [`0xFEB4acf3df3cDEA7399794D0869ef76A6EfAff52`](https://etherscan.io/address/0xFEB4acf3df3cDEA7399794D0869ef76A6EfAff52) | Gnosis Safe 6/9 | NO | ALL, every role on every vault |
| BRAIN | [`0x16388463d60FFE0661Cf7F1f31a7D658aC790ff7`](https://etherscan.io/address/0x16388463d60FFE0661Cf7F1f31a7D658aC790ff7) | Gnosis Safe 3/8 | NO | REPORTING_MANAGER, DEBT_MANAGER, QUEUE_MANAGER, DEPOSIT_LIMIT_MANAGER, DEBT_PURCHASER |
| SECURITY | [`0xe5e2Baf96198c56380dDD5E992D7d1ADa0e989c0`](https://etherscan.io/address/0xe5e2Baf96198c56380dDD5E992D7d1ADa0e989c0) | Gnosis Safe 4/7 | NO | MAX_DEBT_MANAGER |
| KEEPER | [`0x604e586F17cE106B64185A7a0d2c1Da5bAce711E`](https://etherscan.io/address/0x604e586F17cE106B64185A7a0d2c1Da5bAce711E) | yHaaSRelayer contract | NO | REPORTING_MANAGER |
| STRATEGY_MANAGER | [`0x88Ba032be87d5EF1fbE87336B7090767F367BF73`](https://etherscan.io/address/0x88Ba032be87d5EF1fbE87336B7090767F367BF73)| TimelockController | YES | ADD_STRATEGY_MANAGER, REVOKE_STRATEGY_MANAGER |

## Security Council

The ychad.eth multisig is checked against Security Council requirements because, as DADDY, it holds ALL roles on every vault and can immediately execute force-revoke, accountant swap, vault shutdown, and debt reallocation without any timelock. These are the permissions that cause the High Upgradeability and High Exit Window scores. 

The list of signers for the ychad.eth multisig can be found [here](https://docs.yearn.fi/developers/security/multisig).

| Name | Account | Type | ≥ 7 signers | ≥ 51% threshold | ≥ 50% non-insider | Signers public |
|------|---------|------|-------------|-----------------|-------------------|----------------|
| ychad.eth | [`0xFEB4acf3df3cDEA7399794D0869ef76A6EfAff52`](https://etherscan.io/address/0xFEB4acf3df3cDEA7399794D0869ef76A6EfAff52) | Multisig 6/9 | ✅ | ✅ (66%) | ✅ (7/9 non-insider) | ✅ |

Signers (9 total, 2 insiders)

# Contracts & Permissions

## Contracts

| Contract Name | Address | Verified |
|--------------|---------|----------|
| Role Manager (Yearn) | [0xb3bd6b2e61753c311efbcf0111f75d29706d9a41](https://etherscan.io/address/0xb3bd6b2e61753c311efbcf0111f75d29706d9a41) | ✅ |
| Accountant (Yearn) | [0x5A74Cb32D36f2f517DB6f7b0A0591e09b22cDE69](https://etherscan.io/address/0x5A74Cb32D36f2f517DB6f7b0A0591e09b22cDE69) | ✅ |
| Vault Factory (v3.0.4) | [0x770D0d1Fb036483Ed4AbB6d53c1C88fb277D812F](https://etherscan.io/address/0x770D0d1Fb036483Ed4AbB6d53c1C88fb277D812F) | ✅ |
| Vault Original (v3.0.4) | [0xd8063123BBA3B480569244AE66BFE72B6c84b00d](https://etherscan.io/address/0xd8063123BBA3B480569244AE66BFE72B6c84b00d) | ✅ |
| TokenizedStrategy (v3.0.4) | [0xD377919FA87120584B21279a491F82D5265A139c](https://etherscan.io/address/0xD377919FA87120584B21279a491F82D5265A139c) | ✅ |
| V3 Registry (Yearn) | [0xd40ecF29e001c76Dcc4cC0D9cd50520CE845B038](https://etherscan.io/address/0xd40ecF29e001c76Dcc4cC0D9cd50520CE845B038) | ✅ |
| Keeper | [0x52605BbF54845f520a3E94792d019f62407db2f8](https://etherscan.io/address/0x52605BbF54845f520a3E94792d019f62407db2f8) | ✅ |
| TimelockController | [0x88Ba032be87d5EF1fbE87336B7090767F367BF73](https://etherscan.io/address/0x88Ba032be87d5EF1fbE87336B7090767F367BF73) | ✅ |
| TimelockExecutor | [0xF8f60BF9456A6e0141149Db2DD6f02C60da5779B](https://etherscan.io/address/0xF8f60BF9456A6e0141149Db2DD6f02C60da5779B) | ✅ |

## All Permission Owners

| Name | Account | Type |
|------|---------|------|
| governance (TimelockController) | [0x88Ba032be87d5EF1fbE87336B7090767F367BF73](https://etherscan.io/address/0x88Ba032be87d5EF1fbE87336B7090767F367BF73) | TimelockController (7-day delay) |
| DADDY / ychad.eth (proposer + canceller) | [0xFEB4acf3df3cDEA7399794D0869ef76A6EfAff52](https://etherscan.io/address/0xFEB4acf3df3cDEA7399794D0869ef76A6EfAff52) | Multisig 6/9 |
| TimelockExecutor | [0xF8f60BF9456A6e0141149Db2DD6f02C60da5779B](https://etherscan.io/address/0xF8f60BF9456A6e0141149Db2DD6f02C60da5779B) | Custom executor contract (governance: BRAIN 3/8 multisig) |
| BRAIN | [0x16388463d60FFE0661Cf7F1f31a7D658aC790ff7](https://etherscan.io/address/0x16388463d60FFE0661Cf7F1f31a7D658aC790ff7) | Multisig 3/8 (also: feeManager on Accountant, governance on TimelockExecutor) |
| SECURITY | [0xe5e2Baf96198c56380dDD5E992D7d1ADa0e989c0](https://etherscan.io/address/0xe5e2Baf96198c56380dDD5E992D7d1ADa0e989c0) | Multisig 4/7 |
| KEEPER | [0x604e586F17cE106B64185A7a0d2c1Da5bAce711E](https://etherscan.io/address/0x604e586F17cE106B64185A7a0d2c1Da5bAce711E) | yHaaSRelayer contract |
| STRATEGYMANAGER | [0x88Ba032be87d5EF1fbE87336B7090767F367BF73](https://etherscan.io/address/0x88Ba032be87d5EF1fbE87336B7090767F367BF73) | TimelockController (7-day delay) contract |



## Permissions

### Role Manager [(0xb3bd6b2e61753c311efbcf0111f75d29706d9a41)](https://etherscan.io/address/0xb3bd6b2e61753c311efbcf0111f75d29706d9a41)

| Contract | Function | Impact | Owner |
|----------|----------|--------|-------|
| Role Manager | `setPositionHolder(bytes32, address)` | Changes the address assigned to any position (DADDY, BRAIN, etc.). This controls who can perform all permissioned actions across all 37 vaults. | governance (TimelockController) |
| Role Manager | `setPositionRoles(bytes32, uint256)` | Changes what roles a position receives on new and existing vaults. | governance (TimelockController) |
| Role Manager | `transferGovernance(address)` | Starts a 2-step transfer of the Role Manager's governance address. | governance (TimelockController) |
| Role Manager | `removeRoles(address[], address, uint256)` | Removes specific roles from any address on any set of vaults. | governance (TimelockController) |
| Role Manager | `newVault(address, uint256, ...)` | Deploys a new vault, assigns all roles, and sets up the Accountant. | DADDY (Multisig 6/9, no timelock) |
| Role Manager | `removeVault(address)` | Removes a vault from the Role Manager and transfers role_manager to the `ychad.eth` address (DADDY). The vault continues operating but is no longer managed by the Role Manager. | BRAIN (Multisig 3/8, no timelock) |
| Role Manager | `updateDebtAllocator(address, address)` | Changes which Debt Allocator manages a vault's strategy allocations. | BRAIN (Multisig 3/8, no timelock) |

### Vault (each of the 37 vaults, called directly by role holders)

| Contract | Function | Impact | Owner |
|----------|----------|--------|-------|
| Vault | `force_revoke_strategy(address)` | Immediately removes a strategy and records its entire current debt as a loss | DADDY (Multisig 6/9) |
| Vault | `set_accountant(address)` | Replaces the vault's fee engine with any arbitrary contract. | DADDY (Multisig 6/9) |
| Vault | `shutdown_vault()` | Disables new deposits. | DADDY (Multisig 6/9) |
| Vault | `revoke_strategy(address)` | Cleanly removes a strategy. | STRATEGY_MANAGER (TimelockController, 7-day delay) |
| Vault | `transfer_role_manager(address)` | Initiates transfer of the vault's role_manager position. 2-step pattern (must be accepted). | Current role_manager (Role Manager contract) |

### Accountant [(0x5A74Cb32D36f2f517DB6f7b0A0591e09b22cDE69)](https://etherscan.io/address/0x5A74Cb32D36f2f517DB6f7b0A0591e09b22cDE69)

| Contract | Function | Impact | Owner |
|----------|----------|--------|-------|
| Accountant | `updateDefaultConfig(...)` | Changes default fee parameters for all vaults that don't have custom configs. | BRAIN (Multisig 3/8) |
| Accountant | `addVault(address)` / `removeVault(address)` | Controls which vaults this Accountant charges fees for. | feeManager (BRAIN) or vaultManager (role_manager) |


---

## Stage 0 Qualification Check

| Requirement | Met? | Evidence |
|-------------|------|----------|
| Blockchain-based, financial protocol | ✅ | Ethereum smart contracts managing user deposits and yield |
| Assets NOT in custody by centralized entity | ✅ | Assets held in smart contracts and withdrawable by depositors at any time |
| Public documentation exists | ✅ | docs.yearn.fi  |
| Source-available codebase | ✅ | github.com/yearn/yearn-vaults-v3 |
| Verified contracts | ✅ | Check each contract on Etherscan |

## Sources & References
- DeFiScan Framework: https://www.defiscan.info/framework
- DeFiScan Template: https://github.com/deficollective/defiscan
- Yearn V3 Docs: https://docs.yearn.fi/developers/addresses/v3-contracts
- Yearn Frontend: https://github.com/yearn/yearn.fi 
- Yearn Multisig Info: https://docs.yearn.fi/developers/security/multisig
- Permission Scanner: https://github.com/deficollective/permission-scanner
