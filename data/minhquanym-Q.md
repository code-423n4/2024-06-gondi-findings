 # Summary

| Risk | Title |
| --- | --- |
| L-1 | Unnecessary approve `_feeManager` to spend `_asset` |
| L-2 | No need to approve `__aavePool` to spend `__aToken` |
| L-3 | Function in BytesLib could revert with no error message |
| L-4 | `setProtocolFee()` can be called multiple times to spam event emission |
| L-5 | Repayment and liquidation could be blocked if token has a callhook to receiver |
| L-6 | Owner can set `_multiSourceLoan` to address(0) directly without `updateMultiSourceLoanAddressFirst()` |
| L-7 | Slippage of stETH swap could make `validateOffer()` revert |
| N-1 | Modifier `onlyReadyForWithdrawal` is repeatedly execute when users withdraw multiple tokens |
| N-2 | Should use defined variable in function `_checkValidators()` |
| N-3 | Wrong comment `addCallers()` is no longer 2-step process |


# L-1. No need to approve `_feeManager` to spend `_asset`

[Pool.sol#L166](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/Pool.sol#L166)

## Detail
In the constructor of Pool, `_feeManager` is approved to spend `_asset`. However, this is unnecessary since the FeeManager contract does not have any place need to use the allowance of pool.

```solidity
        _queueOutstandingValues = new OutstandingValues[](_maxTotalWithdrawalQueues + 1);
        _queueAccounting = new QueueAccounting[](_maxTotalWithdrawalQueues + 1);

        // @audit unnecessary approve feeManager
        _asset.approve(address(_feeManager), type(uint256).max); 
    }
```

---

# L-2. No need to approve `__aavePool` to spend `__aToken`

[AaveUsdcBaseInterestAllocator.sol#L61](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/AaveUsdcBaseInterestAllocator.sol#L61)

## Detail
The AavePool can burn the `aToken` directly without any allowance from burner address. So the call to approve `aToken` in the constructor of `AaveUsdcBaseInterestAllocator` contract is unnecessary.
```solidity
constructor(...) Owned(tx.origin) {
    // ... 
  
    ERC20(__usdc).approve(__aavePool, type(uint256).max);
     // @audit No need to approve aToken
    ERC20(__aToken).approve(__aavePool, type(uint256).max);
}
```
Check out the AavePool code here https://polygonscan.com/address/0x1ed647b250e5b6d71dc7b25806f44c33f5658f71#code#F1#L196

---

# L-3. Function in BytesLib could revert with no error message

[BytesLib.sol](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/utils/BytesLib.sol)

## Details
All functions in `BytesLib` have some require check for overflow. However, these checks will only work with Solidity version < 0.8 because Solidity 0.8 already has some overflow check. For example, in the function `slice()`, in case overflow actually happens in the first check, the calculation `_length + 31` is already revert by solidity 0.8 before checking the condition in the require.
```solidity
function slice(bytes memory _bytes, uint256 _start, uint256 _length) internal pure returns (bytes memory) {
require(_length + 31 >= _length, "slice_overflow");
require(_start + _length >= _start, "slice_overflow"); // @audit These calculation will revert with solidity 0.8
require(_bytes.length >= _start + _length, "slice_outOfBounds");
...
}
```

---

# L-4. `setProtocolFee()` can be called multiple times to spam event emission

[WithProtocolFee.sol#L69](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/utils/WithProtocolFee.sol#L69)

## Details
Function `setProtocolFee()` does not reset the pending value (`_pendingProtocolFee`). As the result, `setProtocolFee()` can be called infinite times. Even though the protocol fee cannot be changed, the event will be emit multiple times.
```solidity
// @audit setProtocolFee can be called again since _pendingProtocolFeeSetTime is not reset
function setProtocolFee() external virtual {
_setProtocolFee();
}

function _setProtocolFee() internal {
if (block.timestamp < _pendingProtocolFeeSetTime + FEE_UPDATE_NOTICE) {
    revert TooSoonError();
}
ProtocolFee memory protocolFee = _pendingProtocolFee;
_protocolFee = protocolFee;

emit ProtocolFeeUpdated(protocolFee);
}
```

---

# L-5. Repayment and liquidation could be blocked if token has a callhook to receiver

[AuctionLoanLiquidator.sol#L296](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/AuctionLoanLiquidator.sol#L296)

[LiquidationDistributor.sol#L114](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/LiquidationDistributor.sol#L114)

[MultiSourceLoan.sol#L922](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L922)

## Details
Some tokens like ERC777 has a callhook on receiver address. If these tokens are used in a loan, it could allow attacker to force liquidation/repayment to fail. For example, the function `settleAuction()` transfer the fee to originator. The originator could be an contract that will revert when receiving token transfer, making it impossible for auction to settle.
```solidity
asset.safeTransfer(_auction.originator, triggerFee); // @audit can revert if the token has callhook on receiver
asset.safeTransfer(msg.sender, triggerFee);
```

---

# L-6. Owner can set `_multiSourceLoan` to address(0) directly without `updateMultiSourceLoanAddressFirst()`

[PurchaseBundler.sol#L240-L249](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/callbacks/PurchaseBundler.sol#L240-L249)

## Details
The `_pendingMultiSourceLoanAddress` has default value is `address(0)`. As the result, owner could always call `finalUpdateMultiSourceLoanAddress()` to set the `_multiSourceLoan` to `address(0)` without calling `updateMultiSourceLoanAddressFirst()` first.
```solidity
function finalUpdateMultiSourceLoanAddress(address _newAddress) external onlyOwner { // @audit can set `_multiSourceLoan` to address(0) directly without `updateMultiSourceLoanAddressFirst()`
    if (_pendingMultiSourceLoanAddress != _newAddress) {
        revert InvalidAddressUpdateError();
    }

    _multiSourceLoan = MultiSourceLoan(_pendingMultiSourceLoanAddress);
    _pendingMultiSourceLoanAddress = address(0);

    emit MultiSourceLoanUpdated(_pendingMultiSourceLoanAddress);
}
```

---

# L-7. Slippage of stETH swap could make `validateOffer()` revert

[Pool.sol#L369](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/Pool.sol#L369)

## Details
The MultiSourceLoan contract calls the `validateOffer()` function to verify that the loan term has been approved by the Pool contract. This function also draws the necessary capital from the base interest allocator to ensure a sufficient balance for the loan.

As the pool's balance includes capital awaiting claim by the queues, it needs to verify that the pool has enough capital to fund the loan. If this isn't the case, and the principal exceeds the current balance, the function needs to reallocate part of it.
```solidity
if (principalAmount > undeployedAssets) {
    revert InsufficientAssetsError();
} else if (principalAmount > currentBalance) {
    IBaseInterestAllocator(getBaseInterestAllocator).reallocate(currentBalance, principalAmount, true);
}
```

However, the `reallocate()` function for the `LidoEthBaseInterestAllocator` performs a swap that could cause slippage. This could result in the contract still not having enough balance to provide the loan, even after calling `reallocate()`.

---

# N-1. Modifier `onlyReadyForWithdrawal` is repeatedly execute when users withdraw multiple tokens

[UserVault.sol#L324](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/UserVault.sol#L324)

## Detail
```solidity
modifier onlyReadyForWithdrawal(uint256 _vaultId) { // @audit Repeatedly checked when withdraw multiple tokens
if (_readyForWithdrawal[_vaultId] != msg.sender) {
    revert NotApprovedError(_vaultId);
}
_;
}
```

The `onlyReadyForWithdrawal` modifier is used to check if the vault can be withdrawn and the permitted caller is call it. However, this check is performed in internal function, making it repeatedly call for the same `_vaultId` when users withdraw more than 1 tokens

```solidity
function withdrawERC721s(uint256 _vaultId, address[] calldata _collections, uint256[] calldata _tokenIds)
external
{
...
for (uint256 i = 0; i < _collections.length;) {
    _withdrawERC721(_vaultId, _collections[i], _tokenIds[i]);
    unchecked {
        ++i;
    }
}
}

function _withdrawERC721(uint256 _vaultId, address _collection, uint256 _tokenId)
private
onlyReadyForWithdrawal(_vaultId)
{
...
}
```

---

# N-2. Should use defined variable in function `_checkValidators()`

[MultiSourceLoan.sol#L873](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L873)

```solidity
function _checkValidators(LoanOffer calldata _loanOffer, uint256 _tokenId) private {
uint256 offerTokenId = _loanOffer.nftCollateralTokenId;
// @audit Should use the cache value above
if (_loanOffer.nftCollateralTokenId != 0) {
```

---

# N-3. Wrong comment `addCallers()` is no longer 2-step process

[LoanManager.sol#L52](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/LoanManager.sol#L52)

Adding callers are no longer a 2-step process, so the Natspec comment for function `addCallers()` is not correct anymore.
```solidity
/// @notice Second step in d a caller to the accepted callers list. Can be a Loan Contract or Liquidator.
/// @dev Given repayments, we don't allow callers to be removed.
/// @param _callers The callers to add.

function addCallers(ProposedCaller[] calldata _callers) external {
```