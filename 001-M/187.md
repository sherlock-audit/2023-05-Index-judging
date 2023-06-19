Diana

medium

# Use of deprecated chainlink function: latestAnswer()

## Summary
According to Chainlink’s documentation ([API Reference](https://docs.chain.link/data-feeds/price-feeds/api-reference#latestanswer)), the `latestAnswer()` function is deprecated. This function does not throw an error if no answer has been reached, but instead returns 0, possibly causing an incorrect price to be fed to the different price feeds or even a Denial of Service by a division by zero.

## Vulnerability Detail
According to Chainlink’s documentation ([API Reference](https://docs.chain.link/data-feeds/price-feeds/api-reference#latestanswer)), the `latestAnswer()` function is deprecated. This function does not throw an error if no answer has been reached, but instead returns 0,  causing an incorrect price to be fed to the different price feeds.

## Impact
The `latestAnswer()` function is deprecated. This could lead to stale prices. 
This function does not throw an error if no answer has been reached, but instead returns 0,  causing an incorrect price to be fed to the different price feeds.

## Code Snippet
https://github.com/sherlock-audit/2023-05-Index/blob/main/index-coop-smart-contracts/contracts/adapters/AaveLeverageStrategyExtension.sol#L895

```solidity
        int256 rawCollateralPrice = strategy.collateralPriceOracle.latestAnswer();
```

https://github.com/sherlock-audit/2023-05-Index/blob/main/index-coop-smart-contracts/contracts/adapters/AaveLeverageStrategyExtension.sol#L897

```solidity
        int256 rawBorrowPrice = strategy.borrowPriceOracle.latestAnswer();
```

## Tool used

Manual Review

## Recommendation
It is recommended to use Chainlink’s `latestRoundData()` function to get the price instead. It is also recommended to add checks on the return data with proper revert messages if the price is stale or the round is incomplete.