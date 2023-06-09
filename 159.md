Bauchibred

high

# Protocol doesn't completely protect itself from `LTV = 0` tokens


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
