# Issue H-1: eMode implementation is completely broken 

Source: https://github.com/sherlock-audit/2023-05-Index-judging/issues/251 

## Found by 
0x52, 0xGoodess, 0xStalin, BugBusters, Cryptor, hildingr, volodya
## Summary

Enabling eMode allows assets of the same class to be borrowed at much higher a much higher LTV. The issue is that the current implementation makes the incorrect calls to the Aave V3 pool making so that the pool can never take advantage of this higher LTV.

## Vulnerability Detail

[AaveLeverageStrategyExtension.sol#L1095-L1109](https://github.com/sherlock-audit/2023-05-Index/blob/main/index-coop-smart-contracts/contracts/adapters/AaveLeverageStrategyExtension.sol#L1095-L1109)

    function _calculateMaxBorrowCollateral(ActionInfo memory _actionInfo, bool _isLever) internal view returns(uint256) {
        
        // Retrieve collateral factor and liquidation threshold for the collateral asset in precise units (1e16 = 1%)
        ( , uint256 maxLtvRaw, uint256 liquidationThresholdRaw, , , , , , ,) = strategy.aaveProtocolDataProvider.getReserveConfigurationData(address(strategy.collateralAsset));

        // Normalize LTV and liquidation threshold to precise units. LTV is measured in 4 decimals in Aave which is why we must multiply by 1e14
        // for example ETH has an LTV value of 8000 which represents 80%
        if (_isLever) {
            uint256 netBorrowLimit = _actionInfo.collateralValue
                .preciseMul(maxLtvRaw.mul(10 ** 14))
                .preciseMul(PreciseUnitMath.preciseUnit().sub(execution.unutilizedLeveragePercentage));

            return netBorrowLimit
                .sub(_actionInfo.borrowValue)
                .preciseDiv(_actionInfo.collateralPrice);

When calculating the max borrow/repay allowed, the contract uses the getReserveConfigurationData subcall to the pool. 

[AaveProtocolDataProvider.sol#L77-L100](https://github.com/aave/aave-v3-core/blob/29ff9b9f89af7cd8255231bc5faf26c3ce0fb7ce/contracts/misc/AaveProtocolDataProvider.sol#L77-L100)

    function getReserveConfigurationData(
      address asset
    )
      external
      view
      override
      returns (
          ...
      )
    {
      DataTypes.ReserveConfigurationMap memory configuration = IPool(ADDRESSES_PROVIDER.getPool())
        .getConfiguration(asset);
  
      (ltv, liquidationThreshold, liquidationBonus, decimals, reserveFactor, ) = configuration
        .getParams();

The issue with using getReserveConfigurationData is that it always returns the default settings of the pool. It never returns the adjusted eMode settings. This means that no matter the eMode status of the set token, it will never be able to borrow to that limit due to calling the incorrect function.

It is also worth considering that the set token as well as other integrated modules configurations/settings would assume this higher LTV. Due to this mismatch, the set token would almost guaranteed be misconfigured which would lead to highly dangerous/erratic behavior from both the set and it's integrated modules. Due to this I believe that a high severity is appropriate.

## Impact

Usage of eMode, a core function of the contracts, is completely unusable causing erratic/dangerous behavior 

## Code Snippet

[AaveLeverageStrategyExtension.sol#L1095-L1109](https://github.com/sherlock-audit/2023-05-Index/blob/main/index-coop-smart-contracts/contracts/adapters/AaveLeverageStrategyExtension.sol#L1095-L1109)

## Tool used

Manual Review

## Recommendation

Pull the adjusted eMode settings rather than the base pool settings



## Discussion

**ckoopmann**

Yep, this is correct, and will be addressed. 

Btw: I think I saw a bunch of duplicates of this issue when looking through the unfiltered list.

**0xffff11**

Agree with sponsor, valid high. Added missing duplicates

# Issue H-2: _calculateMaxBorrowCollateral calculates repay incorrectly and can lead to set token liquidation 

Source: https://github.com/sherlock-audit/2023-05-Index-judging/issues/254 

## Found by 
0x52
## Summary

When calculating the amount to repay, `_calculateMaxBorrowCollateral` incorrectly applies `unutilizedLeveragePercentage` when calculating `netRepayLimit`. The result is that if the `borrowValue` ever exceeds `liquidationThreshold * (1 - unutilizedLeveragPercentage)` then all attempts to repay will revert. 

## Vulnerability Detail

[AaveLeverageStrategyExtension.sol#L1110-L1118](https://github.com/sherlock-audit/2023-05-Index/blob/main/index-coop-smart-contracts/contracts/adapters/AaveLeverageStrategyExtension.sol#L1110-L1118)

        } else {
            uint256 netRepayLimit = _actionInfo.collateralValue
                .preciseMul(liquidationThresholdRaw.mul(10 ** 14))
                .preciseMul(PreciseUnitMath.preciseUnit().sub(execution.unutilizedLeveragePercentage));

            return _actionInfo.collateralBalance
                .preciseMul(netRepayLimit.sub(_actionInfo.borrowValue))
                .preciseDiv(netRepayLimit);
        }

When calculating `netRepayLimit`, `_calculateMaxBorrowCollateral` uses the `liquidationThreshold` adjusted by `unutilizedLeveragePercentage`. It then subtracts the borrow value from this limit. This is problematic because if the current `borrowValue` of the set token exceeds `liquidationThreshold * (1 - unutilizedLeveragPercentage)` then this line will revert making it impossible to make any kind of repayment. Once no repayment is possible the set token can't rebalance and will be liquidated.

## Impact

Once the leverage exceeds a certain point the set token can no longer rebalance

## Code Snippet

[AaveLeverageStrategyExtension.sol#L1110-L1118](https://github.com/sherlock-audit/2023-05-Index/blob/main/index-coop-smart-contracts/contracts/adapters/AaveLeverageStrategyExtension.sol#L1110-L1118)

## Tool used

Manual Review

## Recommendation

Don't adjust the max value by `unutilizedLeveragPercentage`



## Discussion

**pblivin0x**

The outlined issue and fix LGTM. We need to loosen the performed `netRepayLimit` check to avoid the case where we have high leverage and can't submit repayment (`borrowValue > liquidationThreshold * (1 - unutilizedLeveragPercentage)`)

# Issue M-1: setIncentiveSettings would be halt during a rebalance operation that gets stuck due to supply cap is reached at Aave 

Source: https://github.com/sherlock-audit/2023-05-Index-judging/issues/10 

## Found by 
0xGoodess
## Summary
setIncentiveSettings would be halt during a rebalance operation that gets stuck due to supply cap is reached at Aave

## Vulnerability Detail
rebalance implement a cap of tradeSize and if the need to rebalance require taking more assets than the maxTradeSize, then `twapLeverageRatio` would be set to the targeted leverage.
`twapLeverageRatio` == 0 is required during rebalance.

Consider:

lever is needed during rebalance, the strategy require to borrow more ETH and sell to wstETH
during the 1st call of rebalance the protocol cache the new twapLeverageRatio
However wstETH market in Aave reach supply cap. rebalance/iterateRebalance comes to a halt.
`twapLeverageRatio` remains caching the targeted leverage

setIncentiveSettings requires a condition in which no rebalance is in progress. With the above case, setIncentiveSettings can be halted for an extended period of time until the wstETH market falls under supply cap.

Worth-noting, at the time of writing this issue, [the wstETH market at Aave has been at supply cap](https://app.aave.com/reserve-overview/?underlyingAsset=0x7f39c581f595b53c5cb19bd0b3f8da6c935e2ca0&marketName=proto_mainnet_v3)

In this case, malicious actor who already has a position in wstETH can do the following:

- deposit into the setToken, trigger a rebalance.

- malicious trader withdraw his/her position in Aave wstETH market so there opens up vacancy for supply again.

- protocol owner see supply vacancy, call rebalance in order to lever as required. Now twapLeverageRatio is set to new value since multiple trades are needed

- malicious trader now re-supply the wstETH market at Aave so it reaches supply cap again.

- the protocol gets stuck with a non-zero twapLeverageRatio, `setIncentiveSettings` can not be called.

```solidity
    function setIncentiveSettings(IncentiveSettings memory _newIncentiveSettings) external onlyOperator noRebalanceInProgress {
        incentive = _newIncentiveSettings;

        _validateNonExchangeSettings(methodology, execution, incentive);

        emit IncentiveSettingsUpdated(
            incentive.etherReward,
            incentive.incentivizedLeverageRatio,
            incentive.incentivizedSlippageTolerance,
            incentive.incentivizedTwapCooldownPeriod
        );
    }
```
## Impact
setIncentiveSettings would be halt.

## Code Snippet
https://github.com/sherlock-audit/2023-05-Index/blob/main/index-coop-smart-contracts/contracts/adapters/AaveLeverageStrategyExtension.sol#L484-L495
## Tool used

Manual Review

## Recommendation
Add some checks on whether the supply cap of an Aave market is reached during a rebalance. If so, allows a re-set of twapLeverageRatio



## Discussion

**ckoopmann**

This is another scenario, that we will investigate in more detail.

**pblivin0x**

In the listed vulnerability, it is proposed that a

`malicious actor who already has a position in wstETH can deposit into the setToken, trigger a rebalance.`

But I don't believe this is the case. Unpermissioned actors can _mint_ the SetToken with exact replication via the DebtIssuanceModuleV2. In this case the leverage ratio would remain the same as before the mint and not trigger a rebalance.

I believe the current plan for avoiding any Aave supply cap issues is by imposing a SetToken supply cap.



**0xffff11**

As discussed with sponsor, valid medium

# Issue M-2: Protocol doesn't completely protect itself from `LTV = 0` tokens 

Source: https://github.com/sherlock-audit/2023-05-Index-judging/issues/159 

## Found by 
Angry\_Mustache\_Man, Bauchibred, BugBusters, hildingr, volodya

## Summary

The AaveLeverageStrategyExtension does not completely protect against tokens with a Loan-to-Value (LTV) of 0. Tokens with an LTV of 0 in Aave V3 pose significant risks, as they cannot be used as collateral to borrow upon a breaking withdraw. Moreover, LTVs of assets could be set to 0, even though they currently aren't, it could create substantial problems with potential disruption of multiple functionalities. This bug could cause a Denial-of-Service (DoS) situation in some cases, and has a potential to impact the borrowing logic in the protocol, leading to an unintentionally large perceived borrowing limit.

## Vulnerability Detail

When an AToken has LTV = 0, Aave restricts the usage of certain operations. Specifically, if a user owns at least one AToken as collateral with an LTV = 0, certain operations could revert:

1. **Withdraw**: If the asset being withdrawn is collateral and the user is borrowing something, the operation will revert if the withdrawn collateral is an AToken with LTV > 0.
2. **Transfer**: If the asset being transferred is an AToken with LTV > 0 and the sender is using the asset as collateral and is borrowing something, the operation will revert.
3. **Set the reserve of an AToken as non-collateral**: If the AToken being set as non-collateral is an AToken with LTV > 0, the operation will revert.

Take a look at [AaveLeverageStrategyExtension.sol#L1050-L1119](https://github.com/sherlock-audit/2023-05-Index/blob/3190057afd3085143a31746d65045a0d1bacc78c/index-coop-smart-contracts/contracts/adapters/AaveLeverageStrategyExtension.sol#L1050-L1119)

```solidity
    /**
     * Calculate total notional rebalance quantity and chunked rebalance quantity in collateral units.
     *
     * return uint256          Chunked rebalance notional in collateral units
     * return uint256          Total rebalance notional in collateral units
     */
    function _calculateChunkRebalanceNotional(
        LeverageInfo memory _leverageInfo,
        uint256 _newLeverageRatio,
        bool _isLever
    )
        internal
        view
        returns (uint256, uint256)
    {
        // Calculate absolute value of difference between new and current leverage ratio
        uint256 leverageRatioDifference = _isLever ? _newLeverageRatio.sub(_leverageInfo.currentLeverageRatio) : _leverageInfo.currentLeverageRatio.sub(_newLeverageRatio);

        uint256 totalRebalanceNotional = leverageRatioDifference.preciseDiv(_leverageInfo.currentLeverageRatio).preciseMul(_leverageInfo.action.collateralBalance);

        uint256 maxBorrow = _calculateMaxBorrowCollateral(_leverageInfo.action, _isLever);

        uint256 chunkRebalanceNotional = Math.min(Math.min(maxBorrow, totalRebalanceNotional), _leverageInfo.twapMaxTradeSize);

        return (chunkRebalanceNotional, totalRebalanceNotional);
    }

    /**
     * Calculate the max borrow / repay amount allowed in base units for lever / delever. This is due to overcollateralization requirements on
     * assets deposited in lending protocols for borrowing.
     *
     * For lever, max borrow is calculated as:
     * (Net borrow limit in USD - existing borrow value in USD) / collateral asset price adjusted for decimals
     *
     * For delever, max repay is calculated as:
     * Collateral balance in base units * (net borrow limit in USD - existing borrow value in USD) / net borrow limit in USD
     *
     * Net borrow limit for levering is calculated as:
     * The collateral value in USD * Aave collateral factor * (1 - unutilized leverage %)
     *
     * Net repay limit for delevering is calculated as:
     * The collateral value in USD * Aave liquiditon threshold * (1 - unutilized leverage %)
     *
     * return uint256          Max borrow notional denominated in collateral asset
     */
    function _calculateMaxBorrowCollateral(ActionInfo memory _actionInfo, bool _isLever) internal view returns(uint256) {

        // Retrieve collateral factor and liquidation threshold for the collateral asset in precise units (1e16 = 1%)
        ( , uint256 maxLtvRaw, uint256 liquidationThresholdRaw, , , , , , ,) = strategy.aaveProtocolDataProvider.getReserveConfigurationData(address(strategy.collateralAsset));

        // Normalize LTV and liquidation threshold to precise units. LTV is measured in 4 decimals in Aave which is why we must multiply by 1e14
        // for example ETH has an LTV value of 8000 which represents 80%
        if (_isLever) {
            uint256 netBorrowLimit = _actionInfo.collateralValue
                .preciseMul(maxLtvRaw.mul(10 ** 14))
                .preciseMul(PreciseUnitMath.preciseUnit().sub(execution.unutilizedLeveragePercentage));

            return netBorrowLimit
                .sub(_actionInfo.borrowValue)
                .preciseDiv(_actionInfo.collateralPrice);
        } else {
            uint256 netRepayLimit = _actionInfo.collateralValue
                .preciseMul(liquidationThresholdRaw.mul(10 ** 14))
                .preciseMul(PreciseUnitMath.preciseUnit().sub(execution.unutilizedLeveragePercentage));

            return _actionInfo.collateralBalance
                .preciseMul(netRepayLimit.sub(_actionInfo.borrowValue))
                .preciseDiv(netRepayLimit);
        }
    }
```

Apart from the aforementioned issue with `LTV = 0` tokens, there's another issue with the `_calculateMaxBorrowCollateral()` function. When LTV = 0, `maxLtvRaw` also equals 0, leading to a `netBorrowLimit` of 0. When the borrowing value is subtracted from this, it results in an underflow, causing the borrowing limit to appear incredibly large. This essentially breaks the borrowing logic of the protocol.

## Impact

This bug could potentially disrupt the entire borrowing logic within the protocol by inflating the perceived borrowing limit. This could lead to users borrowing an unlimited amount of assets due to the underflow error. In extreme cases, this could lead to a potential loss of user funds or even a complete protocol shutdown, thus impacting user trust and the overall functionality of the protocol.

## Code Snippet

[AaveLeverageStrategyExtension.sol#L1050-L1119](https://github.com/sherlock-audit/2023-05-Index/blob/3190057afd3085143a31746d65045a0d1bacc78c/index-coop-smart-contracts/contracts/adapters/AaveLeverageStrategyExtension.sol#L1050-L1119)

## Tool used

Manual Review

## Recommendation

The protocol should consider implementing additional protections against tokens with an LTV of 0.



## Discussion

**ckoopmann**

This raises a good point that wasn't fully considered during the design. 
We will investigate this. 

Generally I don't think the "Impact" section is accurate though as users cannot borrow assets on behalf of the token. However if the previous information is accurate it could have an affect on issuance / redemption if the aToken transfers are blocked.

**ckoopmann**

I set this to confirmed as the issue raises valid questions / scenarios that weren't fully considered during the design. 
Unfortunately the recommendation is pretty vague and not very actionable.

After digging into [this audit report](https://www.certora.com/wp-content/uploads/2023/05/Aave_v3_Core_PR_820_Review.pdf) it seems that at least there are protections in place on aave side that no malicious user could produce such a situation by sending us aTokens with ltv=0 or something like that.
So the only scenario where this situation could arise would be if aave governance sets the LTV of the collateral token we are using to 0.

One potential change that could allow us to delever the token in such a situation could be to add flashloan based delevering as also suggested in this issue:
https://github.com/sherlock-audit/2023-05-Index-judging/issues/255

**ckoopmann**

Also see this issue on the morpho spearbit audit for reference. (You will have to create an account solodit to access it) - note that the "Vulernability Detail" section seems to be copy / pasted from that report:
https://solodit.xyz/issues/16216

**ckoopmann**

After rereading this more carefully it seems the above listed limitations only affect a second aToken with LTV > 0. So for this scenario to come into effect we would have to have two aTokens as components in the set, one of which would have `LTV = 0` and the other one (which would not be able to be transfered anymore) would have `LTV > 0`. 

While the issue seems to be valid, if the above understanding is correct we might keep the logic as is and list this as an explicit limitation. Because the strategy extension is designed to work with only one aToken anyway. (The Set Token could have another aToken as a component that is not managed as part of the leverage strategy but this should be avoided and is certainly not part of the expected use)

**0xffff11**

`While the issue seems to be valid, if the above understanding is correct we might keep the logic as is and list this as an explicit limitation`
Keeping the issue as a med because there is still a small possibility for this to happen even though it is not the intended behavior: 
`he Set Token could have another aToken as a component that is not managed as part of the leverage strategy but this should be avoided and is certainly not part of the expected use`

# Issue M-3: no validation to ensure the arbitrum sequencer is down 

Source: https://github.com/sherlock-audit/2023-05-Index-judging/issues/262 

## Found by 
0x007, Bauer, BugBusters, MohammedRizwan, Phantasmagoria, Saeedalipoor01988, ShadowForce, hildingr, jasonxiale, kutugu, rvierdiiev, sashik\_eth
## Summary
There is no validation to ensure sequencer is down
## Vulnerability Detail
```solidity
 int256 rawCollateralPrice = strategy.collateralPriceOracle.latestAnswer();
        rebalanceInfo.collateralPrice = rawCollateralPrice.toUint256().mul(10 ** strategy.collateralDecimalAdjustment);
        int256 rawBorrowPrice = strategy.borrowPriceOracle.latestAnswer();
        rebalanceInfo.borrowPrice = rawBorrowPrice.toUint256().mul(10 ** strategy.borrowDecimalAdjustment);
```

Using Chainlink in L2 chains such as Arbitrum requires to check if the sequencer is down to avoid prices from looking like they are fresh although they are not.

The bug could be leveraged by malicious actors to take advantage of the sequencer downtime.
## Impact
when sequencer is down, stale price is used for oracle and the borrow value and collateral value is calculated and the protocol can be forced to rebalance in a loss position
## Code Snippet
https://github.com/IndexCoop/index-coop-smart-contracts/blob/317dfb677e9738fc990cf69d198358065e8cb595/contracts/adapters/AaveLeverageStrategyExtension.sol#L889-L907
## Tool used

Manual Review

## Recommendation
recommend to add checks to ensure the sequencer is not down.



## Discussion

**ckoopmann**

Seems to be correct however I'm not sure regarding validity / severity since this is specific to L2 and not relevant for our current deployment strategy on Ethereum.

# Issue M-4: Relying solely on oracle base slippage parameters can cause significant loss due to sandwich attacks 

Source: https://github.com/sherlock-audit/2023-05-Index-judging/issues/285 

## Found by 
0x52
## Summary

AaveLeverageStrategyExtension relies solely on oracle price data when determining the slippage parameter during a rebalance. This is problematic as chainlink oracles, especially mainnet, have upwards of 2% threshold before triggering a price update. If swapping between volatile assets, the errors will compound causing even bigger variation. These variations can be exploited via sandwich attacks. 

## Vulnerability Detail

[AaveLeverageStrategyExtension.sol#L1147-L1152](https://github.com/sherlock-audit/2023-05-Index/blob/main/index-coop-smart-contracts/contracts/adapters/AaveLeverageStrategyExtension.sol#L1147-L1152)

    function _calculateMinRepayUnits(uint256 _collateralRebalanceUnits, uint256 _slippageTolerance, ActionInfo memory _actionInfo) internal pure returns (uint256) {
        return _collateralRebalanceUnits
            .preciseMul(_actionInfo.collateralPrice)
            .preciseDiv(_actionInfo.borrowPrice)
            .preciseMul(PreciseUnitMath.preciseUnit().sub(_slippageTolerance));
    }

When determining the minimum return from the swap, _calculateMinRepayUnits directly uses oracle data to determine the final output. The differences between the true value and the oracle value can be systematically exploited via sandwich attacks. Given the leverage nature of the module, these losses can cause significant loss to the pool.

## Impact

Purely oracle derived slippage parameters will lead to significant and unnecessary losses 

## Code Snippet

[AaveLeverageStrategyExtension.sol#L1147-L1152](https://github.com/sherlock-audit/2023-05-Index/blob/main/index-coop-smart-contracts/contracts/adapters/AaveLeverageStrategyExtension.sol#L1147-L1152)

## Tool used

Manual Review

## Recommendation

The solution to this is straight forward. Allow keepers to specify their own slippage value. Instead of using an oracle slippage parameter, validate that the specified slippage value is within a margin of the oracle. This gives the best of both world. It allows for tighter and more reactive slippage controls while still preventing outright abuse in the event that the trusted keeper is compromised.



## Discussion

**ckoopmann**

While raising a valid point regarding Chainlink price change trigger, this still seems the best way to set slippage. 

Letting keepers set their own min/max amount is definitely not safe as they are not necessarily trusted. (Especially in the case of the "ripcord" function which can be called by anyone)

**pblivin0x**

The proposed solution to `allow keepers to specify their own slippage value` and `validate that the specified slippage value is within a margin of the oracle` looks good to me

**ckoopmann**

Ah yes, I missed the `validate that the specified slippage value is within a margin of the oracle` part of the suggestion, which does make sense and would probably slightly decrease slippage risk. However I'm not sure if it is worth the resulting changes in our keeper infrastructure and having a different keeper interface from other leverage modules. 

Overall changing this to "confirmed" but unsure yet of wether we will actually fix it. 

**0xffff11**

Great catch and fix seems reasonable. Valid medium

# Issue M-5: Chainlink price feed is `deprecated`, not sufficiently validated and can return `stale` prices. 

Source: https://github.com/sherlock-audit/2023-05-Index-judging/issues/296 

## Found by 
0x007, 0x8chars, 0xGoodess, 0xStalin, Bauchibred, Bauer, Brenzee, BugBusters, Cryptor, Diana, Madalad, MohammedRizwan, Ocean\_Sky, Oxsadeeq, Phantasmagoria, Saeedalipoor01988, ShadowForce, erictee, jasonxiale, kn0t, kutugu, lil.eth, oxchryston, rvierdiiev, saidam017, sashik\_eth, shogoki, volodya, warRoom, whitehat
## Summary
The function `_createActionInfo()` uses Chainlink's deprecated latestAnswer function, this function also does not guarantee that the price returned by the Chainlink price feed is not stale and there is no additional checks to ensure that the return values are valid.

## Vulnerability Detail

The internal function `_createActionInfo()` uses calls `strategy.collateralPriceOracle.latestAnswer()` and `strategy.borrowPriceOracle.latestAnswer()` that uses Chainlink's deprecated latestAnswer() to get the latest price. However, there is no check for if the return value is a stale data.
```solidity

function _createActionInfo() internal view returns(ActionInfo memory) {
        ActionInfo memory rebalanceInfo;

        // Calculate prices from chainlink. Chainlink returns prices with 8 decimal places, but we need 36 - underlyingDecimals decimal places.
        // This is so that when the underlying amount is multiplied by the received price, the collateral valuation is normalized to 36 decimals.
        // To perform this adjustment, we multiply by 10^(36 - 8 - underlyingDecimals)
        int256 rawCollateralPrice = strategy.collateralPriceOracle.latestAnswer();
        rebalanceInfo.collateralPrice = rawCollateralPrice.toUint256().mul(10 ** strategy.collateralDecimalAdjustment);
        int256 rawBorrowPrice = strategy.borrowPriceOracle.latestAnswer();
        rebalanceInfo.borrowPrice = rawBorrowPrice.toUint256().mul(10 ** strategy.borrowDecimalAdjustment);
// More Code....
}
   
```

## Impact
The function `_createActionInfo()` is used to return important values used throughout the contract, the staleness of the chainlinklink return values will lead to wrong calculation of the collateral and borrow prices and other unexpected behavior.

## Code Snippet
https://github.com/IndexCoop/index-coop-smart-contracts/blob/317dfb677e9738fc990cf69d198358065e8cb595/contracts/adapters/AaveLeverageStrategyExtension.sol#L889

## Tool used

Manual Review

## Recommendation
The `latestRoundData` function should be used instead of the deprecated `latestAnswer` function and add sufficient checks to ensure that the pricefeed is not stale.

```solidity
(uint80 roundId, int256 assetChainlinkPriceInt, , uint256 updatedAt, uint80 answeredInRound) = IPrice(_chainlinkFeed).latestRoundData();
            require(answeredInRound >= roundId, "price is stale");
            require(updatedAt > 0, "round is incomplete");
 ```          





## Discussion

**0xffff11**

Sponsor comments:
```
Good point to switch away from using the deprecated method, which we will look into.
However from this issue it is not clear how / if there is any actual vulnerability resulting from the use of this method.
--
Agree with @ckoopmann , the proposed fix of using latestRoundData() looks reasonable to me
--
I switched to confirmed / disagree with severity as this issue is factually correct and will result in us changing the code, but does not seem to have any real adverse consequences.
```

**0xffff11**

I do believe that this should remain as a medium. Not just for the impact stated by the watson, but also because Chainlink might simply not support it anymore in the future.

# Issue M-6: The protocol does not compatible with token such as USDT because of the Approval Face Protection 

Source: https://github.com/sherlock-audit/2023-05-Index-judging/issues/314 

## Found by 
0x007, 0xStalin, Bauchibred, Ruhum, ShadowForce, bitsurfer, jasonxiale, jprod15, kutugu, n33k, rvierdiiev, shogoki, volodya
## Summary

The protocol does not compatible with token such as USDT because of the Approval Face Protection

## Vulnerability Detail

the protocol is intended to interact with any ERC20 token and USDT is a common one

> Q: Which ERC20 tokens do you expect will interact with the smart contracts?
The protocol expects to interact with any ERC20.

> Individual SetToken's should only interact with ERC20 chosen by the SetToken manager.

when doing the deleverage

https://github.com/IndexCoop/index-protocol/blob/86be7ee76d9a7e4f7e93acfc533216ebef791c89/contracts/protocol/modules/v1/AaveV3LeverageModule.sol#L313

first, we construct the deleverInfo

```solidity
ActionInfo memory deleverInfo = _createAndValidateActionInfo(
		_setToken,
		_collateralAsset,
		_repayAsset,
		_redeemQuantityUnits,
		_minRepayQuantityUnits,
		_tradeAdapterName,
		false
	);
```

then we withdraw from the lending pool, execute trade and repay the borrow token

```solidity
_withdraw(deleverInfo.setToken, deleverInfo.lendingPool, _collateralAsset, deleverInfo.notionalSendQuantity);

        uint256 postTradeReceiveQuantity = _executeTrade(deleverInfo, _collateralAsset, _repayAsset, _tradeData);

        uint256 protocolFee = _accrueProtocolFee(_setToken, _repayAsset, postTradeReceiveQuantity);

        uint256 repayQuantity = postTradeReceiveQuantity.sub(protocolFee);

        _repayBorrow(deleverInfo.setToken, deleverInfo.lendingPool, _repayAsset, repayQuantity);
```

this is calling _repayBorrow

```solidity
/**
 * @dev Invoke repay from SetToken using AaveV2 library. Burns DebtTokens for SetToken.
 */
function _repayBorrow(ISetToken _setToken, ILendingPool _lendingPool, IERC20 _asset, uint256 _notionalQuantity) internal {
	_setToken.invokeApprove(address(_asset), address(_lendingPool), _notionalQuantity);
	_setToken.invokeRepay(_lendingPool, address(_asset), _notionalQuantity, BORROW_RATE_MODE);
}
```

the trade received (quantity - the protocol fee) is used to repay the debt

but the required debt to be required is the (borrowed amount + the interest rate)

suppose the only debt that needs to be repayed is 1000 USDT

trade received (quantity - the protocol) fee is 20000 USDT

only 1000 USDT is used to repay the debt

because when repaying, the paybackAmount is only the debt amount

https://github.com/aave/aave-v3-core/blob/29ff9b9f89af7cd8255231bc5faf26c3ce0fb7ce/contracts/protocol/libraries/logic/BorrowLogic.sol#L204

```solidity
uint256 paybackAmount = params.interestRateMode == DataTypes.InterestRateMode.STABLE
  ? stableDebt
  : variableDebt;
```

then when burning the variable debt token

https://github.com/aave/aave-v3-core/blob/29ff9b9f89af7cd8255231bc5faf26c3ce0fb7ce/contracts/protocol/libraries/logic/BorrowLogic.sol#L224

```solidity
reserveCache.nextScaledVariableDebt = IVariableDebtToken(
	reserveCache.variableDebtTokenAddress
  ).burn(params.onBehalfOf, paybackAmount, reserveCache.nextVariableBorrowIndex);
```

only the "payback amount", which is 1000 USDT is transferred to pay the debt,

the excessive leftover amount is (20000 USDT - 1000 USDT) = 19000 USDT

but if we lookback into the repayBack function

```solidity
/**
 * @dev Invoke repay from SetToken using AaveV2 library. Burns DebtTokens for SetToken.
 */
function _repayBorrow(ISetToken _setToken, ILendingPool _lendingPool, IERC20 _asset, uint256 _notionalQuantity) internal {
	_setToken.invokeApprove(address(_asset), address(_lendingPool), _notionalQuantity);
	_setToken.invokeRepay(_lendingPool, address(_asset), _notionalQuantity, BORROW_RATE_MODE);
}
```

the approved amount is 20000 USDT, but only 1000 USDT approval limit is used, we have 19000 USDT approval limit left

according to

https://github.com/d-xo/weird-erc20#approval-race-protections

> Some tokens (e.g. OpenZeppelin) will revert if trying to approve the zero address to spend tokens (i.e. a call to approve(address(0), amt)).

> Integrators may need to add special cases to handle this logic if working with such a token.

USDT is such token that subject to approval race condition, without approving 0 first, the second approve after first repay will revert

## Impact

second and following repay borrow will revert if the ERC20 token is subject to approval race condition

## Code Snippet

https://github.com/IndexCoop/index-protocol/blob/86be7ee76d9a7e4f7e93acfc533216ebef791c89/contracts/protocol/modules/v1/AaveV3LeverageModule.sol#L313

## Tool used

Manual Review

## Recommendation

Approval 0 first



## Discussion

**pblivin0x**

Looks reasonable to add a preceding zero approval here
`_setToken.invokeApprove(address(_asset), address(_lendingPool), 0);`

**0xffff11**

Valid medium

**ckoopmann**

Still having some final discussion internally over wether to fix this or explicitly list USDT as an incompatible token.

