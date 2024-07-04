### Low-01: Delegated and RevokeDelegate event should have `bytes32 _rights`
**Instances(2)**
When a borrower delegate or revokeDelegate, custom delegation rights `bytes32 _rights` will not be included in event Delegated and event RevokeDelegate.

In DelegateRegistry, custom rights `bytes32 _rights` are hashed into delegation key when storing delegation data. `bytes32 _rights` is crucial in tracking delegations off-chain.

```solidity
//src/lib/loans/MultiSourceLoan.sol
    function delegate(
        uint256 _loanId,
        Loan calldata loan,
        address _delegate,
        bytes32 _rights,
        bool _value
    ) external {
...
        IDelegateRegistry(getDelegateRegistry).delegateERC721(
            _delegate,
            loan.nftCollateralAddress,
            loan.nftCollateralTokenId,
            _rights,
            _value
        );
|>      emit Delegated(_loanId, _delegate, _value);
```
(https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L487)
```solidity
//src/lib/loans/MultiSourceLoan.sol
    function revokeDelegate(address _delegate, address _collection, uint256 _tokenId) external {
...
|>        emit RevokeDelegate(_delegate, _collection, _tokenId);
```
(https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L498)

Recommendations:
Consider adding `bytes32 _rights` field in Delegated and RevokeDelegate event.

### Low-02: `_checkStrictlyBetter` might underflow revert without triggering the custom error.
**Instances(1)**
In MutliSourceLoan::refinanceFull(), if lender initiated the refinance, `_checkStrictlyBetter()` will run to ensure if the lender provided more principal, the annual interest with the new principal is still better than existing loan.

If it's not strictly better, the tx should throw with a custom error `NotStrictlyImprovedError()`.

However, the custom error might not be triggered due to an earlier underflow error.

If `_offerPrincipalAmount` < `_loanPrincipalAmount`, or if `_loanAprBps *_loanPrincipalAmount` < `_offerAprBps * _offerPrincipalAmount`, the tx will throw without triggering the custom error.
```solidity
//src/lib/loans/MultiSourceLoan.sol
    function _checkStrictlyBetter(
        uint256 _offerPrincipalAmount,
        uint256 _loanPrincipalAmount,
        uint256 _offerEndTime,
        uint256 _loanEndTime,
        uint256 _offerAprBps,
        uint256 _loanAprBps,
        uint256 _offerFee
    ) internal view {
...
        if (
|>           ((_offerPrincipalAmount - _loanPrincipalAmount != 0) &&
|>                ((_loanAprBps *
                    _loanPrincipalAmount -
                    _offerAprBps *
                    _offerPrincipalAmount).mulDivDown(
                        _PRECISION,
                        _loanAprBps * _loanPrincipalAmount
                    ) < minImprovementApr)) ||
            (_offerFee != 0) ||
            (_offerEndTime < _loanEndTime)
        ) {
            revert NotStrictlyImprovedError();
        }
...
```
(https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L1112-L1116)

Recommendations: 
check for underflow and revert with custom Error early.

### Low-03: Unnecessary math operation when `_remainingNewLender` is set to type(uint256).max in the refinance flow.
**Instances(1)**
When a lender initiates `refinanceFull()` or `refinancePartial()`, `_remainingNewLender` [will be set to type(uint256).max](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L280), which indicates the lender will repay existing lenders.

However, even when `_remainingNewLender` is set to `type(uint256).max`, in `_processOldTranche()`, `_remainingNewLender -= oldLenderDebt` will still run. This is not meaningful.
```solidity
//src/lib/loans/MultiSourceLoan.sol
    function _processOldTranche(
        address _lender,
        address _borrower,
        address _principalAddress,
        Tranche memory _tranche,
        uint256 _endTime,
        uint256 _protocolFeeFraction,
        uint256 _remainingNewLender
    ) private returns (uint256 accruedInterest, uint256 thisProtocolFee, uint256 remainingNewLender) {
...
        if (oldLenderDebt > 0) {
            if (_lender != _tranche.lender) {
                asset.safeTransferFrom(_lender, _tranche.lender, oldLenderDebt);
            }
            unchecked {
|>              _remainingNewLender -= oldLenderDebt;
            }
        }
|>       remainingNewLender = _remainingNewLender;
```
(https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L665)

Recommendations: 
In `_processOldTranche()`, add a check and only perform the subtraction when _remainingNewLender != type(uint256).max.

### Low-04: Incorrect spelling
**Instances(1)**
There's one instance of incorrect spelling. renegotiationIf should be renegotiationId.
```solidity
//src/lib/loans/BaseLoan.sol
    mapping(address user => mapping(uint256 renegotiationIf => bool notActive))
        public isRenegotiationOfferCancelled;
```
(https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/BaseLoan.sol#L57)

Recommendations:
Change renegotiationIf into renegotiationId.

### Low-05: Pool contract can miss Loan offer origination fees if the borrower submits `emitLoan()`
**Instances(1)**
A lender can specify an optional loan offer origination fee, which will be [charged at loan initiation (emitLoan)](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L1012). The origination fee specified in struct LoanOffer will be deducted from the borrow principal amount.

If the lender is a pool contract, this origination fee can be skipped by the borrower initiating `emitLoan()` with OfferExecution.offer.fee as 0. This fee will not be verified in [the current pool.validateOffer flow](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/Pool.sol#L358-L359).

Note that if the protocol submitted the `emitLoan` (e.g. the borrower placed loan request through UI), the protocol can still enforce the origination fee by supplying non-zero OfferExecution.offer.fee.

This allows a borrower to avoid paying the origination fee for any pool contract lender.

Recommendations: 
Either in the pool’s `validateOffer()` or `PoolOfferHandler.validateOffer()`, add a check to ensure the origination fee is satisfied.








