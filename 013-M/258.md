0x52

medium

# Optimism and Arbitrum use an L2 optimized version of Pool.sol making AaveV3.sol completely incompatible

## Summary

The OP and ARB implementations of AaveV3 use an [L2 optimized pool](https://docs.aave.com/developers/getting-started/l2-optimization/l2pool) contract that has completely different signatures for a majority of user facing functions such as repay, withdraw and borrow

## Vulnerability Detail

[L2Pool.sol#L50-L65](https://github.com/aave/aave-v3-core/blob/29ff9b9f89af7cd8255231bc5faf26c3ce0fb7ce/contracts/protocol/pool/L2Pool.sol#L50-L65)

    function borrow(bytes32 args) external override {
      (address asset, uint256 amount, uint256 interestRateMode, uint16 referralCode) = CalldataLogic
        .decodeBorrowParams(_reservesList, args);
  
      borrow(asset, amount, interestRateMode, referralCode, msg.sender);
    }
  
    function repay(bytes32 args) external override returns (uint256) {
      (address asset, uint256 amount, uint256 interestRateMode) = CalldataLogic.decodeRepayParams(
        _reservesList,
        args
      );
  
      return repay(asset, amount, interestRateMode, msg.sender);
    }

Above we see the repay and borrow functions of the L2Pool.sol contract. These selectors are different than their L1 counterparts.

[AaveV3.sol#L269-L289](https://github.com/sherlock-audit/2023-05-Index/blob/main/index-protocol/contracts/protocol/integration/lib/AaveV3.sol#L269-L289)

    function getRepayCalldata(
        IPool _lendingPool,
        address _asset, 
        uint256 _amountNotional,
        uint256 _interestRateMode,        
        address _onBehalfOf
    )
        public
        pure
        returns (address, uint256, bytes memory)
    {
        bytes memory callData = abi.encodeWithSignature(
            "repay(address,uint256,uint256,address)", 
            _asset, 
            _amountNotional, 
            _interestRateMode,            
            _onBehalfOf
        );
        
        return (address(_lendingPool), 0, callData);
    }

This completely breaks all compatibility with OP and ARB due to calling a selector that is incorrect.

## Impact

All AaveV3 integrations are completely broken on L2s

## Code Snippet

[AaveV3.sol#L269-L289](https://github.com/sherlock-audit/2023-05-Index/blob/main/index-protocol/contracts/protocol/integration/lib/AaveV3.sol#L269-L289)

## Tool used

Manual Review

## Recommendation

To work with L2s like ARB and OP, AaveV3 be modified to pack data similar to [L2Encoder](https://docs.aave.com/developers/getting-started/l2-optimization/l2encoder) provided.