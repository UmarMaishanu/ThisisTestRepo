# Aave V4 contest details

- Join [Sherlock Discord](https://discord.gg/MABEWyASkp)
- Submit findings using the **Issues** page in your private contest repo (label issues as **Medium** or **High**)
- [Read for more details](https://docs.sherlock.xyz/audits/watsons)

# Q&A

### Q: On what chains are the smart contracts going to be deployed?
The initial deployment target is Ethereum mainnet
___

### Q: If you are integrating tokens, are you allowing only whitelisted tokens to work with the codebase or any complying with the standard? Are they assumed to have certain properties, e.g. be non-reentrant? Are there any types of [weird tokens](https://github.com/d-xo/weird-erc20) you want to integrate?
Only explicitly whitelisted ERC20 tokens are accepted. Whitelisting is enforced via governance-controlled configuration and assumes:

- ERC20 compliance without non-standard hooks (no ERC777, no callbacks).
- No fee-on-transfer, rebasing, reflection, or balance-mutation side effects.

Asset onboarding follows the Aave DAO’s technical and risk due diligence process, executed by service providers before governance approval. If the tokens are incompatible with the codebase, then they won't be used.
___

### Q: Are there any limitations on values set by admins (or other roles) in the codebase, including restrictions on array lengths?
Privileged roles are trusted actors (Aave DAO and governance-approved executors). Their parameter changes are considered honest, and misconfiguration is out of scope.

That said, the codebase does enforce specific hard limits on certain configuration values:

- Asset decimals: Assets must fall within a predefined minimum (`MIN_ALLOWED_UNDERLYING_DECIMALS`) and maximum (`MAX_ALLOWED_UNDERLYING_DECIMALS`) decimal range; assets outside these bounds cannot be listed.
- Collateral risk score (`collateralFactor`): Each asset’s collateral risk value is capped by a maximum allowed risk, preventing governance from setting risk scores beyond the upper bound enforced by the protocol.
- Liquidation logic: the maximum liquidation bonus (`maxLiquidationBonus`) is constrained to a protocol-defined minimum; the product of the maximum liquidation bonus and the collateral risk score is bounded by a global upper limit; and the liquidation fee (`liquidationFee`) itself is capped by a protocol‑defined maximum.

Beyond these explicit constraints, most configuration values rely on governance processes rather than on-chain caps, consistent with the “trusted admin” assumption.
___

### Q: Are there any limitations on values set by admins (or other roles) in protocols you integrate with, including restrictions on array lengths?
Protocols the system depends on (such as Chainlink price feeds, listed collateral assets, and other external primitives) are treated as trusted, governance-vetted dependencies. Only whitelisted assets and governance-approved integrations are permitted. These components are assumed to behave consistently with their established interfaces and not introduce arbitrary semantic changes (e.g., unexpected proxy upgrades or non-standard ERC20 behavior).

Evaluation of these dependencies (including oracle configuration, asset onboarding, and integration changes) is handled through Aave’s governance processes and risk frameworks. Under the contest’s trust assumptions, these dependencies are considered stable, predictable, and non-malicious.
___

### Q: Is the codebase expected to comply with any specific EIPs?
EIP712 is implemented for typed signatures used by the Spoke `setUserPositionManeger` intent and for all intents processed through the Signature Gateway. Apart from it, no other EIP compliance requirements beyond standard ERC20 interactions for assets.
For an EIP-violation issue to be valid, it has to qualify for at least Medium severity based on the severity definitions in the "Additional audit info" question.
___

### Q: Are there any off-chain mechanisms involved in the protocol (e.g., keeper bots, arbitrage bots, etc.)? We assume these mechanisms will not misbehave, delay, or go offline unless otherwise specified.
The only off-chain dependency is Chainlink oracle infrastructure. These feeds are assumed to operate correctly, remain available, and not deviate from expected behavior.
___

### Q: What properties/invariants do you want to hold even if breaking them has a low/unknown impact?
Invariants at the the Hub level:

1. Total borrowed assets <= total supplied assets
2. Total borrowed shares == sum of Spoke debt shares
3. Hub added assets >= sum of Spoke added assets (converted from shares)
4. Hub added shares == sum of Spoke added shares
5. Supply share price and drawn index cannot decrease (remains constant or increases)

Issues breaking the above invariants may be considered Medium severity even if the actual impact is Low, considering it doesn't conflict with common sense.
___

### Q: Please discuss any design choices you made.
Aave V4 adopts a **Liquidity Hub** as the canonical accounting layer for all assets: available liquidity, drawn and premium liabilities. User flows are implemented in external **Spokes**, which initiate Hub mutations and perform asset transfers. The Hub enforces global invariants and liquidity provisioning; Spokes handle borrow logic and risk configuration logic.

All Spokes are **governance-permissioned modules**. Governance explicitly authorizes which Spokes may call Hub mutators for each asset and maintains the allowlist. 

**Hub ↔ Spoke Trust Boundary**

The system defines an asymmetric trust model:

- The Hub is the authoritative ledger.
- Permissioned Spokes orchestrate user actions and decide operational details such as:
    - the source and destination addresses for ERC20 transfers between the user and the Hub,
    - when user premium bookkeeping occurs,
    - manage donations within the Spoke.

Spokes operate under Hub-enforced global invariants and per-Spoke caps/flags that throttle or isolate flows without halting the protocol.

**Implications:**

- A malicious or compromised Spoke could misuse its privileges. E.g., drawing all the liquidity up to the draw cap and never return it. This risk is out of scope and mitigated by governance gatekeeping, since only approved Spokes can invoke Hub mutators. Hence, Spokes are considered trusted entities, working in a legit way.

**Collateral Risk Framework**

Each reserve receives a dynamic risk score (0–1000 %) controlled by governance. This score feeds into the final interest rate computation resulting in a risk-adjusted rate: low-risk assets map to lower borrowing costs, and higher-risk assets incur higher effective rates.
___

### Q: Please provide links to previous audits (if any) and all the known issues or acceptable risks.
Shown in the private contest repos.
___

### Q: Please list any relevant protocol resources.
[Aave V4 Documentation](https://aave.com/docs/aave-v4)
[Aave V4 Technical Overview](https://github.com/aave/aave-v4/blob/main/docs/overview.md)
[The Potential of Aave V4](https://governance.aave.com/t/the-potential-of-aave-v4/23150)
___

### Q: Additional audit information.
**Upgradeability / Mutability**
- Hub is immutable; its only mutable surface is through external setters (asset/spoke config) managed by governance process.
- Spoke is upgradeable, allowing parameter tuning or full redeployment while leaving Hub liquidity untouched.

The contest has custom severity definitions that will be considered during judging:
Critical severity:
Direct loss of funds without (extensive) limitations of external conditions. The loss of the affected party must exceed 20% and 100 USD.
Examples: 
- Users lose more than 20% and more than 100 USD of their principal.
- Users lose more than 20% and more than 100 USD of their yield.
- The protocol loses more than 20% and more than 100 USD of the fees.

High severity:
Direct loss of funds without (extensive) limitations of external conditions. The loss of the affected party must exceed 5% and 50 USD.
Examples:
- Users lose more than 5% and more than 50 USD of their principal.
- Users lose more than 5% and more than 50 USD of their yield.
- The protocol loses more than 5% and more than 50 USD of the fees.

Medium severity:
Causes a loss of funds but requires certain external conditions or specific states, or a loss is highly constrained. The loss of the affected part must exceed 1% and 10 USD.
Breaks core contract functionality, rendering the contract useless or leading to loss of funds of the affected party that exceeds 1% and 10 USD.
Note: If a single attack can cause a 1% loss but can be replayed indefinitely, it may be considered a 100% loss and can be medium or higher severity, depending on the constraints.
Examples:
- Users lose more than 1% and more than 10 USD of their principal.
- Users lose more than 1% and more than 10 USD of their yield.
- The protocol loses more than 1% and more than 10 USD of the fees.

Gas optimisation severity:
The report must show how the gas cost for the transaction can be reduced by 5% and the protocol team has to implement a change in the code for the issue to be considered valid. Gas optimisation severity won't have duplicates and only the first one will be considered valid.

Point weights breakdown:
- A critical severity finding is worth 30 points.
- A high severity finding is worth 10 points.
- A medium severity finding is worth 5 points.
- Gas optimisation severity doesn't get points and is a separate severity with a separate pot.


# Audit scope

[aave__aave-v4 @ 6959e3219b5506bf2acae18551cbb2a68a5b8fba](https://github.com/sherlock-scoping/aave__aave-v4/tree/6959e3219b5506bf2acae18551cbb2a68a5b8fba)
- [aave__aave-v4/src/access/AccessManagerEnumerable.sol](aave__aave-v4/src/access/AccessManagerEnumerable.sol)
- [aave__aave-v4/src/hub/AssetInterestRateStrategy.sol](aave__aave-v4/src/hub/AssetInterestRateStrategy.sol)
- [aave__aave-v4/src/hub/Hub.sol](aave__aave-v4/src/hub/Hub.sol)
- [aave__aave-v4/src/hub/interfaces/IAssetInterestRateStrategy.sol](aave__aave-v4/src/hub/interfaces/IAssetInterestRateStrategy.sol)
- [aave__aave-v4/src/hub/interfaces/IBasicInterestRateStrategy.sol](aave__aave-v4/src/hub/interfaces/IBasicInterestRateStrategy.sol)
- [aave__aave-v4/src/hub/interfaces/IHubBase.sol](aave__aave-v4/src/hub/interfaces/IHubBase.sol)
- [aave__aave-v4/src/hub/interfaces/IHub.sol](aave__aave-v4/src/hub/interfaces/IHub.sol)
- [aave__aave-v4/src/hub/libraries/AssetLogic.sol](aave__aave-v4/src/hub/libraries/AssetLogic.sol)
- [aave__aave-v4/src/hub/libraries/Premium.sol](aave__aave-v4/src/hub/libraries/Premium.sol)
- [aave__aave-v4/src/hub/libraries/SharesMath.sol](aave__aave-v4/src/hub/libraries/SharesMath.sol)
- [aave__aave-v4/src/interfaces/IMulticall.sol](aave__aave-v4/src/interfaces/IMulticall.sol)
- [aave__aave-v4/src/interfaces/INoncesKeyed.sol](aave__aave-v4/src/interfaces/INoncesKeyed.sol)
- [aave__aave-v4/src/interfaces/IRescuable.sol](aave__aave-v4/src/interfaces/IRescuable.sol)
- [aave__aave-v4/src/libraries/math/MathUtils.sol](aave__aave-v4/src/libraries/math/MathUtils.sol)
- [aave__aave-v4/src/libraries/math/PercentageMath.sol](aave__aave-v4/src/libraries/math/PercentageMath.sol)
- [aave__aave-v4/src/libraries/math/WadRayMath.sol](aave__aave-v4/src/libraries/math/WadRayMath.sol)
- [aave__aave-v4/src/libraries/types/EIP712Types.sol](aave__aave-v4/src/libraries/types/EIP712Types.sol)
- [aave__aave-v4/src/libraries/types/Roles.sol](aave__aave-v4/src/libraries/types/Roles.sol)
- [aave__aave-v4/src/misc/UnitPriceFeed.sol](aave__aave-v4/src/misc/UnitPriceFeed.sol)
- [aave__aave-v4/src/position-manager/GatewayBase.sol](aave__aave-v4/src/position-manager/GatewayBase.sol)
- [aave__aave-v4/src/position-manager/interfaces/IGatewayBase.sol](aave__aave-v4/src/position-manager/interfaces/IGatewayBase.sol)
- [aave__aave-v4/src/position-manager/interfaces/INativeTokenGateway.sol](aave__aave-v4/src/position-manager/interfaces/INativeTokenGateway.sol)
- [aave__aave-v4/src/position-manager/interfaces/INativeWrapper.sol](aave__aave-v4/src/position-manager/interfaces/INativeWrapper.sol)
- [aave__aave-v4/src/position-manager/interfaces/ISignatureGateway.sol](aave__aave-v4/src/position-manager/interfaces/ISignatureGateway.sol)
- [aave__aave-v4/src/position-manager/libraries/EIP712Hash.sol](aave__aave-v4/src/position-manager/libraries/EIP712Hash.sol)
- [aave__aave-v4/src/position-manager/NativeTokenGateway.sol](aave__aave-v4/src/position-manager/NativeTokenGateway.sol)
- [aave__aave-v4/src/position-manager/SignatureGateway.sol](aave__aave-v4/src/position-manager/SignatureGateway.sol)
- [aave__aave-v4/src/spoke/AaveOracle.sol](aave__aave-v4/src/spoke/AaveOracle.sol)
- [aave__aave-v4/src/spoke/instances/SpokeInstance.sol](aave__aave-v4/src/spoke/instances/SpokeInstance.sol)
- [aave__aave-v4/src/spoke/interfaces/IAaveOracle.sol](aave__aave-v4/src/spoke/interfaces/IAaveOracle.sol)
- [aave__aave-v4/src/spoke/interfaces/IPriceOracle.sol](aave__aave-v4/src/spoke/interfaces/IPriceOracle.sol)
- [aave__aave-v4/src/spoke/interfaces/ISpokeBase.sol](aave__aave-v4/src/spoke/interfaces/ISpokeBase.sol)
- [aave__aave-v4/src/spoke/interfaces/ISpoke.sol](aave__aave-v4/src/spoke/interfaces/ISpoke.sol)
- [aave__aave-v4/src/spoke/interfaces/ITreasurySpoke.sol](aave__aave-v4/src/spoke/interfaces/ITreasurySpoke.sol)
- [aave__aave-v4/src/spoke/libraries/KeyValueList.sol](aave__aave-v4/src/spoke/libraries/KeyValueList.sol)
- [aave__aave-v4/src/spoke/libraries/LiquidationLogic.sol](aave__aave-v4/src/spoke/libraries/LiquidationLogic.sol)
- [aave__aave-v4/src/spoke/libraries/PositionStatusMap.sol](aave__aave-v4/src/spoke/libraries/PositionStatusMap.sol)
- [aave__aave-v4/src/spoke/libraries/ReserveFlagsMap.sol](aave__aave-v4/src/spoke/libraries/ReserveFlagsMap.sol)
- [aave__aave-v4/src/spoke/libraries/UserPositionDebt.sol](aave__aave-v4/src/spoke/libraries/UserPositionDebt.sol)
- [aave__aave-v4/src/spoke/Spoke.sol](aave__aave-v4/src/spoke/Spoke.sol)
- [aave__aave-v4/src/spoke/TreasurySpoke.sol](aave__aave-v4/src/spoke/TreasurySpoke.sol)
- [aave__aave-v4/src/utils/Multicall.sol](aave__aave-v4/src/utils/Multicall.sol)
- [aave__aave-v4/src/utils/NoncesKeyed.sol](aave__aave-v4/src/utils/NoncesKeyed.sol)
- [aave__aave-v4/src/utils/Rescuable.sol](aave__aave-v4/src/utils/Rescuable.sol)


