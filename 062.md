0xStalin

medium

# All functions that uses the transferFrom() of the ExplicitERC20 library will fail if the token to be transfered is a fee-on-transfer token

## Summary
All erc20 tokens that charge fees on transfer will revert because the strictly equal comparisson to validate if the exact amount of specified quantity was transferred without considering the fees

## Vulnerability Detail
The implementation of the `transferFrom()` in the `ExplicitERC20` library uses a [strict equal check operator to validate that the `post-transferFrom balance is exactly equal to the pre-transferFrom balance + the amount of tokens that were requested to be transferred`](https://github.com/sherlock-audit/2023-05-Index/blob/main/index-protocol/contracts/lib/ExplicitERC20.sol#L65-L68), the issue is if the token charges fees on the transfer, the actual received amount will be slightly less than what was requested, thus, [the strict equal check will fail, and the whole tx will be reverted](https://github.com/sherlock-audit/2023-05-Index/blob/main/index-protocol/contracts/lib/ExplicitERC20.sol#L65-L68)

- [Checkout this doc for further reference about fee-on-transfer tokens](https://github.com/d-xo/weird-erc20#fee-on-transfer)

## Impact
All erc20 tokens that charge fees on transfer will revert because the strictly equal comparisson to validate if the exact amount of specified quantity was transferred without considering the fees

## Code Snippet
https://github.com/sherlock-audit/2023-05-Index/blob/main/index-protocol/contracts/lib/ExplicitERC20.sol#L43-L70

## Tool used
Manual Review

## Recommendation
Instead of using the current check, update it to validate that the post-transferFrom balance is greater than the pre-transferFrom balance, in this way, if the erc20 charged fees, the tx won't revert
```solidity
function transferFrom(
    IERC20 _token,
    address _from,
    address _to,
    uint256 _quantity
)
    internal
{
    
    if (_quantity > 0) {
        uint256 existingBalance = _token.balanceOf(_to);

        SafeERC20.safeTransferFrom(
            _token,
            _from,
            _to,
            _quantity
        );

        uint256 newBalance = _token.balanceOf(_to);

        // Verify transfer quantity is reflected in balance
        require(
-           newBalance == existingBalance.add(_quantity),
+           newBalance > existingBalance,
            "Invalid post transfer balance"
        );
    }
}
);
```