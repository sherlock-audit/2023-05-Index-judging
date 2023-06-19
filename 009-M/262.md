ShadowForce

medium

# no validation to ensure the arbitrum sequencer is down

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