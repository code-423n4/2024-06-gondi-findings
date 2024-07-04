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

### Low-06: Borrower can use an arbitrary `offerId` for a pool contract's loan offer, which might lead to incorrect off-chain accounting.
**Instances(1)**
`offerId` is a field in struct [LoanOffer](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/interfaces/loans/IMultiSourceLoan.sol#L26) and LoanOffer is typically signed by the lender. Currently, `offerId` is generated off-chain and its correctness is verified through [_checkSignature(lender, offer.hash(), _lenderOfferSignature)](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L780) in the emitLoan flow.

But when the lender is a pool contract, `offerId` will not be verified due to _checkSignature() being [bypassed in _validateOfferExectuion()](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L777). 

A borrower can supply any offerIds for a pool lender offer in `emitLoan()` or `refinanceFromLoanExeuctionData()` flow. As a result, `emit LoanEmitted(loanId, offerIds, loan, totalFee)` or `emit LoanRefinancedFromNewOffers(_loanId, newLoanId, loan, offerIds, totalFee)` will contain arbitrary offerIds. This may create conflicts in off-chain offerId accounting.

Recommendations:
If offerId for a pool lender is relevant, consider allowing a pool contract to increment and store its next offerId on-chain.

### Low-07: Borrowers might use a lender's `addNewTranche` renegotiation offer to `refinanceFull` in some cases
**Instances(1)**
A borrower can use a lender’s renegotiation offer signature for `addNewTranch()` in `refinanceFull()`, as long as the loan only has one tranche.

This is because `refinanceFull()` only checks [whether _renegotiationOffer.trancheIndex.length == _loan.tranche.length](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L168). When there's only one tranche in the loan, [addNewTranche()'s renegoationOffer's check condition](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L379) will also satisfy.

`refinanceFull()` will also ensure the refinanced tranche [gets a better apr](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L591). So the borrower gets better apr for the existing tranche instead of taking out additional principal in `addNewTranche()`.

In addition, if the lender signed a renegotiation offer intended for `refinanceFull()`, the same offer can also be used for `addNewTranche()` if the condition `_renegotiationOffer.trancheIndex[0] == _loan.tranche.length` is satisfied. Because `_renegotiationOffer.trancheIndex[0]` is never checked in `refinanceFull()` flow, the lender might supply any values. In this case, the lender is forced to open a more junior tranche which can be risky for lenders.

It's better to prevent the same renegotiation offer from being used interchangeably in different methods with different behaviors.

Recommendations:
In refinanceFull(), add a check to ensure `_renegotiationOffer.trancheIndex[0]==0`.

### Low-08: Consider adding a cap for minLockPeriod
**Instances(1)**
`_minLockPeriod` is used to compute the lock time for a tranche or a loan. If the ratio is set too high (e.g.10000) Tranches or loans cannot be refinanced due to [failing _loanLocked() checks](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L181).
```solidity
//src/lib/loans/MultiSourceLoan.sol
    function setMinLockPeriod(uint256 __minLockPeriod) external onlyOwner {

        _minLockPeriod = __minLockPeriod;

        emit MinLockPeriodUpdated(__minLockPeriod);
    }
```
(https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L508)

Recommendations:
Considering adding a cap value (e.g. 10%)

 
### Low-09: `updateLiquidationContract()` might lock collaterals and funds in the current liquidator contract
**Instances(1)**
In LiquidationHandler::updateLiquidationContract(), the loan liquidator contract can be updated. 
```solidity
    function updateLiquidationContract(address __loanLiquidator) external override onlyOwner {
        __loanLiquidator.checkNotZero();
        _loanLiquidator = __loanLiquidator;
        emit LiquidationContractUpdated(__loanLiquidator);
    }
```
(https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/LiquidationHandler.sol#L74)

There are two vulnerable conditions:`updateLiquidationContract()` is called when (1) there are on-going  / unsettled auctions in the current liquidator, (2) or there might be a pending `liquidateLoan()` tx.

1. If MultisourceLoan's liquidator contract is updated. None of the exiting auctions originated from the MultisourceLoan can be settled because AuctionLoanLiquidator::settleAuction will call [IMultiSourceLoan(_auction.loanAddress).loanLiquidated(...](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/AuctionLoanLiquidator.sol#L300) This will cause [onlyLiquidator modifier revert](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L464). MultiSourceLoan contract no longer recognizes the old liquidator contract.

The collateral and bid funds will be locked in the old liquidator contract.

2. If there is a pending [MultiSourceLoan::liquidateLoan](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L445) tx before updateLiquidationContract() call. The auction of the loan will be created right before updateLiquidationContract() settles. Similar to (1), the collateral will be locked in the old liquidator contract.

In addition, since MultiSourceLoan::liquidateLoan() is permissionless, an attacker can front-run updateLiquidationContract tx to cause the loan liquidated to an old liquidator contract.

Recommendations:
1. In UpdateLiquidationContract(), consider adding a check that the existing liquidator’s token balance is 0, with no outstanding auction.
2. Only update liquidationContract when there are no liquidatable loans.











