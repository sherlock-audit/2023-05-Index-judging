ShadowForce

medium

# use of tx.origin will cause problems with optimism

## Summary
the use of tx.origin is incorrect
## Vulnerability Detail
For `onlyEOA` , `tx.origin` is used to ensure that the caller is from an EOA and not a smart contract.

```solidity
  /**
     * Throws if caller is a contract, can be used to stop flash loan and sandwich attacks
     */
    modifier onlyEOA() {
        require(msg.sender == tx.origin, "Caller must be EOA Address");
        _;
    }
```

However, according to [EIP 3074](https://eips.ethereum.org/EIPS/eip-3074#abstract),

This EIP introduces two EVM instructions AUTH and AUTHCALL. The first sets a context variable authorized based on an ECDSA signature. The second sends a call as the authorized account. This essentially delegates control of the externally owned account (EOA) to a smart contract.

Therefore, using tx.origin to ensure msg.sender is an EOA will not hold true in the event EIP 3074 goes through.
## Impact
Using modifier `onlyEOA` to ensure calls are made only from EOA will not hold true in the event EIP 3074 goes through.
## Code Snippet
https://github.com/IndexCoop/index-coop-smart-contracts/blob/317dfb677e9738fc990cf69d198358065e8cb595/contracts/lib/BaseExtension.sol#L56-L62
## Tool used

Manual Review

## Recommendation
Recommend using OpenZepellin's isContract function (https://docs.openzeppelin.com/contracts/2.x/api/utils#Address-isContract-address-). Note that there are edge cases like contract in constructor that can bypass this and hence caution is required when using this.
```solidity
  /**
     * Throws if caller is a contract, can be used to stop flash loan and sandwich attacks
     */
    modifier onlyEOA() {
        require(isContract(msg.sender)) 
        _;
    }
```
