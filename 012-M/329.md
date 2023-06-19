ShadowForce

high

# Wrongly assume the token decimals is 18

## Summary

Wrongly assume the token decimals is 18

## Vulnerability Detail

very important

```solidity
struct ActionInfo {
	uint256 collateralBalance;                      // Balance of underlying held in Aave in base units (e.g. USDC 10e6)
	uint256 borrowBalance;                          // Balance of underlying borrowed from Aave in base units
	uint256 collateralValue;                        // Valuation in USD adjusted for decimals in precise units (10e18)
	uint256 borrowValue;                            // Valuation in USD adjusted for decimals in precise units (10e18)
	uint256 collateralPrice;                        // Price of collateral in precise units (10e18) from Chainlink
	uint256 borrowPrice;                            // Price of borrow asset in precise units (10e18) from Chainlink
	uint256 setTotalSupply;                         // Total supply of SetToken
}
```

and

```solidity
struct ContractSettings {
	ISetToken setToken;                             // Instance of leverage token
	ILeverageModule leverageModule;                 // Instance of Aave leverage module
	IProtocolDataProvider aaveProtocolDataProvider; // Instance of Aave protocol data provider
	IChainlinkAggregatorV3 collateralPriceOracle;   // Chainlink oracle feed that returns prices in 8 decimals for collateral asset
	IChainlinkAggregatorV3 borrowPriceOracle;       // Chainlink oracle feed that returns prices in 8 decimals for borrow asset
	IERC20 targetCollateralAToken;                  // Instance of target collateral aToken asset
	IERC20 targetBorrowDebtToken;                   // Instance of target borrow variable debt token asset
	address collateralAsset;                        // Address of underlying collateral
	address borrowAsset;                            // Address of underlying borrow asset
	uint256 collateralDecimalAdjustment;            // Decimal adjustment for chainlink oracle of the collateral asset. Equal to 28 - collateralDecimals (10^18 * 10^18 / 10^decimals / 10^8)
	uint256 borrowDecimalAdjustment;                // Decimal adjustment for chainlink oracle of the borrowing asset. Equal to 28 - borrowDecimals (10^18 * 10^18 / 10^decimals / 10^8)
}
```

we know 

uint256 collateralValue and uint256 borrowValue

are both in 10e18

and we have 

https://github.com/sherlock-audit/2023-05-Index/blob/3190057afd3085143a31746d65045a0d1bacc78c/index-coop-smart-contracts/contracts/adapters/AaveLeverageStrategyExtension.sol#L889

```solidity
uint256 collateralDecimalAdjustment;            // Decimal adjustment for chainlink oracle of the collateral asset. Equal to 28 - collateralDecimals (10^18 * 10^18 / 10^decimals / 10^8)

	uint256 borrowDecimalAdjustment;                // Decimal adjustment for chainlink oracle of the borrowing asset. Equal to 28 - borrowDecimals (10^18 * 10^18 / 10^decimals / 10^8)
```

the decimal adjustment is (28 - decimal of token)

ok now let us proceed

this is important

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

        rebalanceInfo.collateralBalance = strategy.targetCollateralAToken.balanceOf(address(strategy.setToken));
        rebalanceInfo.borrowBalance = strategy.targetBorrowDebtToken.balanceOf(address(strategy.setToken));
        rebalanceInfo.collateralValue = rebalanceInfo.collateralPrice.preciseMul(rebalanceInfo.collateralBalance);
        rebalanceInfo.borrowValue = rebalanceInfo.borrowPrice.preciseMul(rebalanceInfo.borrowBalance);
        rebalanceInfo.setTotalSupply = strategy.setToken.totalSupply();

        return rebalanceInfo;
    }
```

let us use the focus on the collateral price sacling

```solidity
  int256 rawCollateralPrice = strategy.collateralPriceOracle.latestAnswer();
        rebalanceInfo.collateralPrice = rawCollateralPrice.toUint256().mul(10 ** strategy.collateralDecimalAdjustment);
```

and the comment:

```solidity
// Calculate prices from chainlink. Chainlink returns prices with 8 decimal places, but we need 36 - underlyingDecimals decimal places.
```

so assume the chainlink return 8 decimals

rawCollateral price is 8 decimals

the collateral decimal adjustment is (28 - token decimals), and the token decimal is regular 18

the collateral decimals adjustment is 10

we are using 8 decimals ** 10 ** 10, which is 18 decimals, for collateral value, this is fine, the math checkout

However, token decimal does not have to be 18,

if the token decimal is 6, the math does not work

for example, if the token decimal is 6

28 - 6 = 22, the decimal adjustment is 22

and 8 decimals * 10 ** 22 = 10e32, 

## Impact

if the collateral token decimal is not 18, the code does not work because of the incorrect scaling of the oracle price

```solidity
   /**
     * Derive the min repay units from collateral units for delever. Units are calculated as target collateral rebalance units multiplied by slippage tolerance
     * and pair price (collateral oracle price / borrow oracle price). Output is measured in borrow unit decimals.
     *
     * return uint256           Min position units to repay in borrow asset
     */
    function _calculateMinRepayUnits(uint256 _collateralRebalanceUnits, uint256 _slippageTolerance, ActionInfo memory _actionInfo) internal pure returns (uint256) {
        // @audit
        // division before manipulation?
        return _collateralRebalanceUnits
            .preciseMul(_actionInfo.collateralPrice)
            .preciseDiv(_actionInfo.borrowPrice)
            .preciseMul(PreciseUnitMath.preciseUnit().sub(_slippageTolerance));
    }
```

there are also code such as 

```solidity
  function _calculateCurrentLeverageRatio(
        uint256 _collateralValue,
        uint256 _borrowValue
    )
        internal
        pure
        returns(uint256)
    {
        // @audit
        return _collateralValue.preciseDiv(_collateralValue.sub(_borrowValue));
    }
```

if token decimals is not 18, which result in collateral value or borrow value not in 10e18 decimals,

```solidity
_collateralValue.sub(_borrowValue)
```

would not work

if collateral value is 10e32, and borrow value is 10e18 decimals,

```solidity
_collateralValue.sub(_borrowValue)
```

is too small because 10e18 is neglible amotun comparing to 10e32

## Code Snippet

https://github.com/sherlock-audit/2023-05-Index/blob/3190057afd3085143a31746d65045a0d1bacc78c/index-coop-smart-contracts/contracts/adapters/AaveLeverageStrategyExtension.sol#L889

## Tool used

Manual Review

## Recommendation

We recommend the protocol does not use 28 - token decimals for decimal adjustment
