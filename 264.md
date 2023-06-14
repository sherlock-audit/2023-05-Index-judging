ShadowForce

medium

# Chainlink price can return a negative number

## Summary
Chainlink price can return a negative number
## Vulnerability Detail
```solidity
int256 rawCollateralPrice = strategy.collateralPriceOracle.latestAnswer();
        rebalanceInfo.collateralPrice = rawCollateralPrice.toUint256().mul(10 ** strategy.collateralDecimalAdjustment);
        int256 rawBorrowPrice = strategy.borrowPriceOracle.latestAnswer();
        rebalanceInfo.borrowPrice = rawBorrowPrice.toUint256().mul(10 ** strategy.borrowDecimalAdjustment);
```
in the snippet above there is a call for a chainlink price feed. The problem here is that there is no check to ensure the number returned is negative or not. Therefore a negative price can be returned by the chainlink price feed. A negative price feed is indicative of a malfunctioning price feed. Therefore the tx should revert. Because there is no checks for negative price, the tx will not revert and this is a problem for rebalancing and will cause a loss of funds for many users.
## Impact
 if the chainlink oracle returns a negative number, that means the oracle is malfunctioning and the transaction should revert, in the current implementation the negative price can be cast to positive integer and used as a price oracle
therefore wrong price is used for oracle and the borrow value and collateral value is calculated and the protocol can be forced to rebalance in a loss position.
## Code Snippet
https://github.com/IndexCoop/index-coop-smart-contracts/blob/317dfb677e9738fc990cf69d198358065e8cb595/contracts/adapters/AaveLeverageStrategyExtension.sol#L889-L907
## Tool used

Manual Review

## Recommendation
ensure tx reverts when negative price is provided.