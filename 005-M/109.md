PRAISE

medium

# missing deadline and deadline checker in lever() and delever() function in AaveV3LeverageModule.sol contract.

## Summary
Lever() and delever() transactions could be delayed in mempool due to Network Congestion

## Vulnerability Detail
Network congestion in the mempool refers to a situation where the number of pending transactions waiting to be included in a blockchain exceeds the available capacity of the network to process them promptly.  

So a deadline and deadline checker(usually a require statement that takes account of duration) is needed to revert the transactions. 

This is expedient because:
1. The transactions can be pending for a very long time due to network congestion.

2. price slippage can happen as the 2 functions handle DEX trades. There could be a change in price between the collateral per borrow asset/ repay asset affecting the minimum quantities specified in lever() and delever() functions

A very vivid example is ETH/USDT price.  As of the time of writing this report 1 ETH = 1,883 USDT and it's constantly changing.

## Impact
transactions can be stuck in the mempool for a long period of time due to missing deadline and deadline checker

Also there could be price slippage between the collateral Asset and borrow Assets/repay Assets.

## Code Snippet
https://github.com/sherlock-audit/2023-05-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AaveV3LeverageModule.sol#L252

https://github.com/sherlock-audit/2023-05-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AaveV3LeverageModule.sol#L313
## Tool used

Manual Review

## Recommendation
implement a deadline and deadline checker in the lever() and delever() functions like uniswap