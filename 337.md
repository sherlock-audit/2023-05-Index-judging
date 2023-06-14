MohammedRizwan

medium

# Wrong use of access modifier on BaseManagerV2.transferTokens() function

## Summary
Wrong use of access modifier on BaseManagerV2.transferTokens() function

## Vulnerability Detail
## Impact
In BaseManagerV2.sol contract, transferTokens() function is given as below,

```solidity
File: contracts/manager/BaseManagerV2.sol

309    /**
310     * OPERATOR ONLY: Transfers _tokens held by the manager to _destination. Can be used to
311     * recover anything sent here accidentally. In BaseManagerV2, extensions should
312     * be the only contracts designated as `feeRecipient` in fee modules.
313     *
314     * @param _token           ERC20 token to send
315     * @param _destination     Address receiving the tokens
316     * @param _amount          Quantity of tokens to send
317     */
318    function transferTokens(address _token, address _destination, uint256 _amount) external onlyExtension {
319        IERC20(_token).safeTransfer(_destination, _amount);
320    }
321
322    /**
```
As seen above at L-318, transferTokens() function has used onlyExtension  modifier which is wrong use as per NatSpec comment.

at L-310, the function NatSpec comments says, OPERATOR ONLY modifier to be used on transferTokens(),

onlyOperator modifier is given as below,

```solidity
File: contracts/manager/BaseManagerV2.sol

    /**
     * Throws if the sender is not the SetToken operator
     */
    modifier onlyOperator() {
        require(msg.sender == operator, "Must be operator");
        _;
    }
```

## Code Snippet
Code Link-
https://github.com/IndexCoop/index-coop-smart-contracts/blob/317dfb677e9738fc990cf69d198358065e8cb595/contracts/manager/BaseManagerV2.sol#L309-L322

## Tool used
Manual Review

## Recommendation
Use onlyOperator modifier on transferTokens() instead of onlyExtension modifier.


```solidity
File: contracts/manager/BaseManagerV2.sol

-    function transferTokens(address _token, address _destination, uint256 _amount) external onlyExtension {
+    function transferTokens(address _token, address _destination, uint256 _amount) external onlyOperator {
        IERC20(_token).safeTransfer(_destination, _amount);
    }
```
