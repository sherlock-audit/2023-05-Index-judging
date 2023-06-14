ShadowForce

high

# last trade timestamp is not updated

## Summary
last trade timestamp is never updated
## Vulnerability Detail
in AaveLeverageStrategyExtension.sol there is a function
```solidity
  function disengage(string memory _exchangeName) external onlyOperator {
        LeverageInfo memory leverageInfo = _getAndValidateLeveragedInfo(
            execution.slippageTolerance,
            exchangeSettings[_exchangeName].twapMaxTradeSize,
            _exchangeName
        );

        uint256 newLeverageRatio = PreciseUnitMath.preciseUnit();

        (
            uint256 chunkRebalanceNotional,
            uint256 totalRebalanceNotional
        ) = _calculateChunkRebalanceNotional(leverageInfo, newLeverageRatio, false);

        if (totalRebalanceNotional > chunkRebalanceNotional) {
            _delever(leverageInfo, chunkRebalanceNotional);
        } else {
            _deleverToZeroBorrowBalance(leverageInfo, totalRebalanceNotional);
        }

        emit Disengaged(
            leverageInfo.currentLeverageRatio,
            newLeverageRatio,
            chunkRebalanceNotional,
            totalRebalanceNotional
        );
    }
```
this function never updates the last trade timestamp, additionally the cooldown period check is ineffective.
```solditiy
 _updateLastTradeTimestamp(_exchangeName)
```
the snippet above should be included in the function
## Impact
because the timestamp is not updated correctly, this allows rebalance to be called more frequently than it should in a cooldown period, this breaks an invariant of the protocol.
## Code Snippet
https://github.com/IndexCoop/index-coop-smart-contracts/blob/317dfb677e9738fc990cf69d198358065e8cb595/contracts/adapters/AaveLeverageStrategyExtension.sol#L412-L438
## Tool used

Manual Review

## Recommendation
we recommend to update last trade timestamp in the disengage function.