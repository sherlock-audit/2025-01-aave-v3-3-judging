# Issue M-1: Liquidators can routinely leave deficit not formed setting gas cost to cover normal liquidations only 

Source: https://github.com/sherlock-audit/2025-01-aave-v3-3-judging/issues/197 

## Found by 
hyh

### Summary

Liquidators can use tx gas limit to accept a loss instead of performing bad debt cleanup. When the latter is not profitable all the liquidators will avoid the last, `MIN_LEFTOVER_BASE` sized, part of the collateral in their calls and if one's call is about to be executed with higher gas usage this essentially means that a liquidator is not first and will obtain only `MIN_LEFTOVER_BASE` liquidation bonus (LB), paying the full deficit formation cost. If that bears a loss higher than a loss of just paying the non-deficit liquidation gas price the liquidators will routinely set the gas limit to cover it only, removing the possibility of bad debt clearing.

### Root Cause

Bad debt cleanup liquidations and normal liquidations have substantially different gas costs, while former is far less profitable for a liquidator (it's lower collateral left at that point). So in a competing environment where it's profitable to leave some collateral intact, the first liquidator get the main funds, while the late transactions will be executed at a loss, not being able to stop the execution otherwise, they can set gas costs to cover normal liquidations only whenever this loss be still more profitable than receiving a liquidation bonus off the small collateral leftover, while performing full bad debt clearing.

These situations will essentially mean that borrower has paid full extra cost in the form of the nearly full position LB, while bad debt wasn't cleared and might not be cleared at all in the future, depending of the market state. I.e. liquidators can get the extra LB, creating more bad debt comparing to 3.2 and not clearing it. 

The creating more bad debt part comes from the fact that incentive to leave the smallest possible collateral leftover comes from the existence of bad debt cleanup logic, not present before, i.e. since 3.3 for one-collateral-many-debts borrowers if a liquidator don't leave something, they have to pay for `_burnBadDebt()`:

[LiquidationLogic.sol#L408-L410](https://github.com/sherlock-audit/2025-01-aave-v3-3/blob/main/aave-v3-origin/src/contracts/protocol/libraries/logic/LiquidationLogic.sol#L408-L410)

```solidity
    if (hasNoCollateralLeft && userConfig.isBorrowingAny()) {
>     _burnBadDebt(reservesData, reservesList, userConfig, params.reservesCount, params.user);
    }
```

Which can be very costly (the `492078 - 392439` gas cost difference is multiplied by the number of additional active bad debt reserves):

[Pool.Operations.json#L4-L9](https://github.com/sherlock-audit/2025-01-aave-v3-3/blob/main/aave-v3-origin/snapshots/Pool.Operations.json#L4-L9)

```json
> "liquidationCall: deficit on liquidated asset": "392439",
> "liquidationCall: deficit on liquidated asset + other asset": "492078",
  "liquidationCall: full liquidation": "392439",
  "liquidationCall: full liquidation and receive ATokens": "368797",
  "liquidationCall: partial liquidation": "383954",
  "liquidationCall: partial liquidation and receive ATokens": "360308",
```

### Internal Pre-conditions

Borrower being liquidated has one collateral, with other supplies being not used as collaterals, and a number of debts. This can be quite common since 3.3 spreading over many debt reserves comes as a natural remedy from `50%` rule change from being debt reserve wise to be the whole position wise. Liquidating one debt will bring HF up and can block the liquidations of all the other debts, while having one debt reserve a borrower can be liquidated fully at once.


### External Pre-conditions

Gas price is high on liquidations and it's calm conditions level remains high enough so `LB * MIN_LEFTOVER_BASE` isn't enough to cover for gas costs of the deficit formations of all debt reserves of the borrower (this is done all in one atomically).

### Attack Path

Liquidators restrict gas spend of the transactions, so if they are winning the first place in a block it will be executed, but if they are not it will be reverted bearing the fixed loss of non-deficit liquidation gas budget. Whichever it will be depends solely on the state of the pool, tx parameters are the same.

This allows liquidators to receive nearly full position LB, but not paying for deficit formation, essentially stealing from the borrowers 3.3 introduced extra LB.

### Impact

Since allowing the `50%` of all the total debt to be liquidated at once is a extra payoff for liquidators at the expense of the borrowers in order to provide bad debt clearing and deficit formation, the failure to do so is a direct loss for the borrowers from the 3.3 release. I.e. in the described circumstances nothing changes bad debt vice, it's still unrealized and require manual DAO intervention (repaying on behalf), but borrowers now lose more LB to liquidators.

### PoC

1. L1 borrower Bob have only one collateral, `totalAmount` of it, and have a number of different debt reserves, say `7`. A market wide down move just happened resulting with increased gas price and his collateral losing value sharply so it will no longer cover his biggest debt reserve's position after an addition of the liquidator incentive, i.e. his cumulative debt position becomes liquidable with bad debt coming from all the reserves.

2. Liquidators Alice and Max both calculated the expected profit of two variants: liquidating the whole collateral and leaving `MIN_LEFTOVER_BASE` of it. Since the gas costs are elevated the total cost of bad debt cleanup for Bob's position is greater than liquidator incentive coming from `MIN_LEFTOVER_BASE` part of the collateral, so both Alice and Max simultaneously run liquidations calls with `debtToCover = totalAmount - MIN_LEFTOVER_BASE`, i.e. both want to leave the `MIN_LEFTOVER_BASE` part of Bob's collateral intact.

3. Alice outbid Max for the block placement and run the liquidation, leaving Max tx to deal with only `MIN_LEFTOVER_BASE` of collateral, while it also having `debtToCover = totalAmount - MIN_LEFTOVER_BASE`. It will be reduced to `MIN_LEFTOVER_BASE` and Max's tx will be executed at loss. However, the loss comes from the extra gas costs of bad debt cleanup.

4. Knowing that, both Alice and Max will set gas limit for their transactions to cover non-bad-debt case only. Say, in line with the [latest snapshot](https://github.com/sherlock-audit/2025-01-aave-v3-3/blob/main/aave-v3-origin/snapshots/Pool.Operations.json#L4-L9) if all the no-bad-debt liquidations cost around `400k` and each bad debt clean-up cost around `100k` per asset, both Alice and Max can set the limit at `410k` if paying that much without receiving liquidation bonus if less expensive than clearing all the bad debt assets with the full cost, while receiving the `MIN_LEFTOVER_BASE * LB` payoff (let's omit protocol part for simplicity). What is more profitable between these two depends on the exact gas cost, price per action in the release version, native token price, collateral LB and the number of debt reserves Bob have.

5. Depending on these variables bad debt might be profitable or not to be cleared even when gas price abates to its current norm.

### Mitigation

Consider evaluating bad debt clearing scenarios for the various min gas prices, collateral LBs and big enough number of debt assets to be cleared. That is, LB coming from the one size fits all `MIN_LEFTOVER_BASE` has to cover not just a normal liquidations, but a cleanup of may, e.g. `7-10` debt assets, with high enough low activity gas price, e.g. `15 Gwei`, and a low enough stable collateral asset LB, e.g. `4.5%`, in an environment of elevated ETH prices, e.g. ETH at `5k`, `10k`. Now it looks to be somewhat lower than needed, e.g. liquidating `7` assets at `15 Gwei` at the ETH price as of time of this report costs nearly twice more than `4.5% * 1000 = 45 USD` LB.

Bearing in mind the outlined necessity to cover for these inherently costly scenarios consider rising `MIN_BASE_MAX_CLOSE_FACTOR_THRESHOLD`.

There are some additional considerations:

* `MIN_LEFTOVER_BASE` being higher than current minimal profitably liquidable debt only forces liquidation to be less fragmented, as low value positions can still be liquidated in full once `HF <= CLOSE_FACTOR_HF_THRESHOLD`: it forbids the case when `HF > CLOSE_FACTOR_HF_THRESHOLD` and `0.5 * totalDebtInBaseCurrency < MIN_LEFTOVER_BASE`, and it's profitable to liquidate `0.5 * totalDebtInBaseCurrency`, but not profitable to liquidate `totalDebtInBaseCurrency - MIN_LEFTOVER_BASE`, i.e. min `MIN_LEFTOVER_BASE` leftover requirement pushes the liquidation amount lower than `50%` of the whole portfolio and out of the profitable range so liquidation won't happen until HF deteriorates further, which puts some pressure on the overall health of the protocol

* `MIN_LEFTOVER_BASE` being lower than current minimal profitably liquidable debt directly allows the creation of many leftover positions that won't be liquidated and also allows for more cases of leaving some collateral behind just to avoid bad debt cleanup. When `MIN_LEFTOVER_BASE` is big it forbids some material share of such cases (a position will generate bad debt if collateral be zeroed, so the liquidator would like to leave some, but can't as it violate the `MIN_LEFTOVER_BASE` leftover: this is as frequent as `MIN_LEFTOVER_BASE` is big, say zero requirement won't catch any such cases) and liquidators either game the system as described or pay for bad debt cleanup as designed

That is, from overall design perspective it's a tradeoff, but bigger `MIN_BASE_MAX_CLOSE_FACTOR_THRESHOLD` and `MIN_LEFTOVER_BASE = MIN_BASE_MAX_CLOSE_FACTOR_THRESHOLD / 2` look more plausible.


# Issue M-2: Liquidator can avoid resolving bad debt with dust supply/transfer while seizing all the borrower's collateral 

Source: https://github.com/sherlock-audit/2025-01-aave-v3-3-judging/issues/203 

## Found by 
bughuntoor, hyh, pkqs90

### Summary

Whenever a position with one collateral and many debt reserves is up for liquidation with claiming all the collateral, its debt reserves will be deemed bad debt and cleaned up during `executeLiquidationCall()`, which will bear a significant cost for a liquidator.

To avoid that the liquidator can create an additional collateral for the borrower. For example, can send a dust amount of non-isolated mode aToken not used by them yet, enabling it as a collateral via `executeFinalizeTransfer()`. This can be used by liquidators routinely as profit enhancement and leaves all the bad debt intact with deficit not formed.

### Root Cause

When `vars.totalCollateralInBaseCurrency > vars.collateralToLiquidateInBaseCurrency` in `executeLiquidationCall()` it is `hasNoCollateralLeft == false` and bad debt clean-up is avoided. For a bad debt bearing position with many debt reserves, liquidators will do that as long as this is profitable, which can frequently be the case on L1.

### Internal Pre-conditions

Borrower being liquidated has one collateral, with other supplies being not used as collaterals, and a number of debts. This can be quite common since 3.3 spreading over many debt reserves comes as a natural remedy from 50% rule change from being debt reserve wise to be the whole position wise.

### External Pre-conditions

Gas price is high on liquidations so paying for dust aToken transfer or supply on behalf is cheaper than covering gas costs of the deficit formation for all debt reserves of the borrower.

### Attack Path

The goal is to trigger `setUsingAsCollateral(id, true)` with some additional action that doesn't require borrower participation. This can be supply with `onBehalfOf = borrower` on straightforward aToken transfer, which is cheaper:

`executeFinalizeTransfer()`, [SupplyLogic.sol#L216-L230](https://github.com/sherlock-audit/2025-01-aave-v3-3/blob/main/aave-v3-origin/src/contracts/protocol/libraries/logic/SupplyLogic.sol#L216-L230):

```
      if (params.balanceToBefore == 0) {
        DataTypes.UserConfigurationMap storage toConfig = usersConfig[params.to];
        if (
          ValidationLogic.validateAutomaticUseAsCollateral(
            reservesData,
            reservesList,
            toConfig,
            reserve.configuration,
            reserve.aTokenAddress
          )
        ) {
>>        toConfig.setUsingAsCollateral(reserveId, true);
          emit ReserveUsedAsCollateralEnabled(params.asset, params.to);
        }
      }
```

### Impact

Since allowing the 50% of all the total debt to be liquidated at once is a extra payoff for liquidators at the expense of the borrowers in order to provide bad debt clearing and deficit formation, the failure to do so is a direct loss for the borrowers from the 3.3 release. I.e. in the described circumstances nothing changes bad debt vice, it's still unrealized and require manual DAO intervention (repaying on behalf), but borrowers now lose more LB to liquidators.

### PoC

According to the contest snapshot it's about `145k` for [aToken transfer](https://github.com/sherlock-audit/2025-01-aave-v3-3/blob/main/aave-v3-origin/snapshots/AToken.transfer.json#L8) with enabling the collateral. [Supply]((https://github.com/sherlock-audit/2025-01-aave-v3-3/blob/main/aave-v3-origin/snapshots/Pool.Operations.json#L16)) looks to be more expensive, `176k`.

[AToken.transfer.json#L2](https://github.com/sherlock-audit/2025-01-aave-v3-3/blob/main/aave-v3-origin/snapshots/AToken.transfer.json#L2)

```json
  "full amount; receiver: ->enableCollateral": "144881",
```

It's about 100k per [an additional asset](https://github.com/sherlock-audit/2025-01-aave-v3-3/blob/main/aave-v3-origin/snapshots/Pool.Operations.json#L4-L5) bad debt clean-up. Whenever a bad debt bearing borrower have more than 2 debt reserves with one of them capable to take all the collateral it's profitable to transfer aToken and avoid the cleanup (`100*k > 145 if k > 1`, where `k` is number of additional debt reserves).

### Mitigation

Since user can always enable the collateral manually one way to control for the issue is to require that automatic use happens on non-dust amount addition only, e.g.:

[ValidationLogic.sol#L621-L641](https://github.com/sherlock-audit/2025-01-aave-v3-3/blob/main/aave-v3-origin/src/contracts/protocol/libraries/logic/ValidationLogic.sol#L621-L641)

```diff
  function validateAutomaticUseAsCollateral(
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    DataTypes.UserConfigurationMap storage userConfig,
    DataTypes.ReserveConfigurationMap memory reserveConfig,
+   uint256 amountInBaseCurrency,
    address aTokenAddress
  ) internal view returns (bool) {
    if (reserveConfig.getDebtCeiling() != 0) {
      // ensures only the ISOLATED_COLLATERAL_SUPPLIER_ROLE can enable collateral as side-effect of an action
      IPoolAddressesProvider addressesProvider = IncentivizedERC20(aTokenAddress)
        .POOL()
        .ADDRESSES_PROVIDER();
      if (
        !IAccessControl(addressesProvider.getACLManager()).hasRole(
          ISOLATED_COLLATERAL_SUPPLIER_ROLE,
          msg.sender
        )
      ) return false;
-   }
+   } else {
+       // ensures that amount that triggered the action is not below minimum
+       if (amountInBaseCurrency < LiquidationLogic.MIN_LEFTOVER_BASE) return false;
+   }   
    return validateUseAsCollateral(reservesData, reservesList, userConfig, reserveConfig);
  }
```

All the uses of `validateAutomaticUseAsCollateral` will need to supply the base currency equivalent amount for the action, e.g.:

[SupplyLogic.sol#L215-L230](https://github.com/sherlock-audit/2025-01-aave-v3-3/blob/main/aave-v3-origin/src/contracts/protocol/libraries/logic/SupplyLogic.sol#L215-L230)

```diff
      if (params.balanceToBefore == 0) {
        DataTypes.UserConfigurationMap storage toConfig = usersConfig[params.to];
+       uint256 assetPrice = IPriceOracleGetter(params.oracle).getAssetPrice(params.asset);
+       uint256 assetUnit = 10 ** reserve.configuration.getDecimals();
+       uint256 amountInBaseCurrency = (params.amount * assetPrice) / assetUnit;
        if (
          ValidationLogic.validateAutomaticUseAsCollateral(
            reservesData,
            reservesList,
            toConfig,
            reserve.configuration,
+           amountInBaseCurrency,
            reserve.aTokenAddress
          )
        ) {
          toConfig.setUsingAsCollateral(reserveId, true);
          emit ReserveUsedAsCollateralEnabled(params.asset, params.to);
        }
      }
```


