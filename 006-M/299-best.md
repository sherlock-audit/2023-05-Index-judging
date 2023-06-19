ShadowForce

high

# division before multiplication may result in truncation of result

## Summary
division before multiplication may result in truncation of result
## Vulnerability Detail
```solidity
 function _calculateMinRepayUnits(uint256 _collateralRebalanceUnits, uint256 _slippageTolerance, ActionInfo memory _actionInfo) internal pure returns (uint256) {
        // @audit
        // division before manipulation?
        return _collateralRebalanceUnits
            .preciseMul(_actionInfo.collateralPrice)
            .preciseDiv(_actionInfo.borrowPrice)
            .preciseMul(PreciseUnitMath.preciseUnit().sub(_slippageTolerance));
    }
```
in the snippet above we can see there is division before multiplication. As we all know doing this can result in truncation.
funds may be lost (0) due to division before multiplication precision issues

for example:
if _actionInfo.borrowPrice is less than 1, it will be truncated to 0. any further multiplication with this number will also result in 0.
we observe multiplication after this point in this specific place.
```solidity
 .preciseMul(_actionInfo.collateralPrice)
            .preciseDiv(_actionInfo.borrowPrice)
            .preciseMul(PreciseUnitMath.preciseUnit().sub(_slippageTolerance));
```
all multiplacation must first be done before we decide to divide.
## Impact
funds may be lost (0) due to division before multiplication precision issues
## Code Snippet
https://github.com/IndexCoop/index-coop-smart-contracts/blob/317dfb677e9738fc990cf69d198358065e8cb595/contracts/adapters/AaveLeverageStrategyExtension.sol#L1147-L1152
## Tool used

Manual Review

## Recommendation
multiply first then divide to avoid precision loss