0x007

medium

# Use pool.supply instead of pool.deposit

## Summary
AAVE V2 uses deposit function to supply tokens. However, V3 uses supply function. deposit function is not mentioned in the [AAVE docs](https://docs.aave.com/developers/core-contracts/pool). It is included **at the bottom** of the [code](https://github.com/aave/aave-v3-core/blob/94e571f3a7465201881a59555314cd550ccfda57/contracts/protocol/pool/Pool.sol#L754-L774) just for backward compatibility. The IPool interface also has a [message for devs](https://github.com/aave/aave-v3-core/blob/94e571f3a7465201881a59555314cd550ccfda57/contracts/interfaces/IPool.sol#L740) to use supply instead.

## Vulnerability Detail
AAVEV3 library uses deprecated deposit function

## Impact
It won't be great using such a deprecated function, especially considering the fact that the pool contract could be upgraded and AAVEV3 library should be resilient

## Code Snippet
https://github.com/sherlock-audit/2023-05-Index/blob/main/index-protocol/contracts/protocol/integration/lib/AaveV3.sol#L64
https://github.com/sherlock-audit/2023-05-Index/blob/main/index-protocol/contracts/protocol/integration/lib/AaveV3.sol#L52-L101

## Tool used

Manual Review

## Recommendation
Use the supply function rather than deposit function for AAVE V3 library.
