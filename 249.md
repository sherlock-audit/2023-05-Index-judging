MohammedRizwan

medium

# Unhandled chainlink revert would lock price oracle access

## Summary
Chainlink's latestAnswer() is used which could potentially revert and make it impossible to query any prices. This could lead to permanent denial of service.

## Vulnerability Detail
Following smart contract function uses latestAnswer(),

In AaveLeverageStrategyExtension.sol contract, _createActionInfo() function is given by,

```solidity
File: contracts/adapters/AaveLeverageStrategyExtension.sol

895        int256 rawCollateralPrice = strategy.collateralPriceOracle.latestAnswer();

897        int256 rawBorrowPrice = strategy.borrowPriceOracle.latestAnswer();

```
[Link to code](https://github.com/IndexCoop/index-coop-smart-contracts/blob/317dfb677e9738fc990cf69d198358065e8cb595/contracts/adapters/AaveLeverageStrategyExtension.sol#L895-L897)

As seen above, _createActionInfo() function makes use of Chainlink's latestAnswer() to get the latest price. However, there is no fallback logic to be executed when the access to the Chainlink data feed is denied by Chainlink's multisigs. Chainlink's multisigs can immediately block access to price feeds at will. Therefore, to prevent denial of service scenarios, it is recommended to query Chainlink price feeds using a defensive approach with Solidity’s try/catch structure. In this way, if the call to the price feed fails, the caller contract is still in control and can handle any errors safely and explicitly.

## Impact
Call to latestAnswer could potentially revert and make it impossible to query any prices. This could lead to permanent denial of service.

## Code Snippet
https://github.com/IndexCoop/index-coop-smart-contracts/blob/317dfb677e9738fc990cf69d198358065e8cb595/contracts/adapters/AaveLeverageStrategyExtension.sol#L895-L897

Openzeppelin reference-

Refer to https://blog.openzeppelin.com/secure-smart-contract-guidelines-the-dangers-of-price-oracles/ for more information regarding potential risks to account for when relying on external price feed providers.

## Tool used
Manual Review

## Recommendation
Surround the call to latestAnswer() with try/catch instead of calling it directly. In a scenario where the call reverts, the catch block can be used to call a fallback oracle or handle the error in any other suitable way.

NOTE:
latestAnswer() is deprecated and it is already recommended to use latestRoundData() instead of latestAnswer(). However, Please note that this issue is also applicable to  latestAnswer() as well as latestRoundData().
