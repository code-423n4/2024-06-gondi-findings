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

### Low-10: Unnecessary code - BytesLib methods are not used in this contract or its parent contracts.
**Instances(1)**
BytesLib methods are not used in PurchaseBundler.sol or its parent contracts.
```solidity
//src/lib/callbacks/PurchaseBundler.sol
    using BytesLib for bytes;
...
```
(https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/callbacks/PurchaseBundler.sol#L24)

Recommendations:
Consider removing this line.

### Low-11: Some proposed callers might not be confirmed
**Instances(1)**
LoanManagerParameterSetter.sol has a two-step process of adding callers. The issue is `addCallers()` doesn't check whether _callers.length == proposedCallers.length. If _callers.length < proposedCaller.length, some proposedCallers' indexes will not run in the for-loop. proposedCallers whose indexes are after callers will not be added as callers.
```solidity
//src/lib/loans/LoanManagerParameterSetter.sol
    function addCallers(ILoanManager.ProposedCaller[] calldata _callers) external onlyOwner {
        if (getProposedAcceptedCallersSetTime + UPDATE_WAITING_TIME > block.timestamp) {
            revert TooSoonError();
        }
        ILoanManager.ProposedCaller[] memory proposedCallers = getProposedAcceptedCallers;
        uint256 totalCallers = _callers.length;
|>      for (uint256 i = 0; i < totalCallers;) {
            ILoanManager.ProposedCaller calldata caller = _callers[i];
            if (
                proposedCallers[i].caller != caller.caller || proposedCallers[i].isLoanContract != caller.isLoanContract
            ) {
                revert InvalidInputError();
            }

            unchecked {
                ++i;
            }
        }
        ILoanManager(getLoanManager).addCallers(_callers);
    }
```
(https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/LoanManagerParameterSetter.sol#L110)

Recommendations:
Add check to ensure `_callers.length == proposedCallers.length`.


### Low-12: Incorrect comments
**Instances(2)**
(1) Auction Loan liquidator -> User Vault
```solidity
/// @title Auction Loan Liquidator
/// @author Florida St
/// @notice NFTs that represent bundles.
contract UserVault is ERC721, ERC721TokenReceiver, IUserVault, Owned {
...
```
(https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/UserVault.sol#L13)
(2) address(0) = ETH -> address(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE) = ETH
```solidity
    /// @notice ERC20 balances for a given vault: token => (vaultId => amount). address(0) = ETH
    mapping(address token => mapping(uint256 vaultId => uint256 amount)) _vaultERC20s;
```
(https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/UserVault.sol#L33)

Recommendations:
Correct comments

### Low-13: vaultID’s NFT/ERC20 bundle can be modified while the loan is outstanding
**Instances(1)**
UserVault.sol allows a user to bundle assets(NFT/ERC20) together in a vault to be used as a collateral NFT. 

According to [doc](https://app.gitbook.com/o/4HJV0LcOOnJ7AVJ77p8e/s/W2WSJrV6PSLWo4p8vIGq/vaults), the intended behavior is `new NFTs cannot be added to the vault unless borrower burn the vault and create a new vaultId with a new bundle of asset`.

This is not currently the case in UserVault.sol. Anyone can deposit ERC20 or ERC721 to an existing vaultID at any time. Although this doesn’t decrease assets from the vault, this may increase VaultID assets at any time during lender offer signing, loan outstanding, and loan liquidation auction process. 
```solidity
    function depositERC721(uint256 _vaultId, address _collection, uint256 _tokenId) external {
        _vaultExists(_vaultId);

        if (!_collectionManager.isWhitelisted(_collection)) {
            revert CollectionNotWhitelistedError();
        }
        _depositERC721(msg.sender, _vaultId, _collection, _tokenId);
    }

    function _depositERC721(address _depositor, uint256 _vaultId, address _collection, uint256 _tokenId) private {
        ERC721(_collection).transferFrom(_depositor, address(this), _tokenId);

        _vaultERC721s[_collection][_tokenId] = _vaultId;

        emit ERC721Deposited(_vaultId, _collection, _tokenId);
    }
```
(https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/UserVault.sol#L152-L158)

Increasing the assets of a vaultId doesn’t put a loan’s collateralization at risk. However, this may create inconsistencies in lender offers due to vaultId ‘s changing asset bundle. 

Due to the permissionless deposit process of UserVault.sol, this may also allow a malicious actor to deposit assets to a vaultID during auciton to manipulate bidding.

Recommendations:
If the intention is to disallow adding new NFTs to a vault before burning of vaultId, consider a two-step deposit and vault mint process, caller deposit assets to a new vaultId first before minting the vaultId. And disallow deposit to a vaultId after minting.

### Low-14: OraclePoolOfferHandler::validateOffer allows borrowers to game spot aprPremium movement to get lower apr
**Instances(1)**
In OraclePoolOfferHandler.sol, aprPremium is partially based on current pool utilization (totalOutstanding / totalAssets). 

In OraclePoolOfferHandler::validateOffer, aprPremium will not re-calculate unless min time interval (`getAprUpdateTolerance`) has passed. At the same time, anyone can call `setAprPremium()` to update aprPremium instantly.
```solidity
    function setAprPremium() external {
|>      uint128 aprPremium = _calculateAprPremium();
        getAprPremium = AprPremium(aprPremium, uint128(block.timestamp));

        emit AprPremiumSet(aprPremium);
    }

    function validateOffer(uint256 _baseRate, bytes calldata _offer)
        external
        view
        override
        returns (uint256, uint256)
    {
        AprPremium memory aprPremium = getAprPremium;
        uint256 aprPremiumValue =
|>          (block.timestamp - aprPremium.updatedTs > getAprUpdateTolerance) ? _calculateAprPremium() : aprPremium.value;
...
```
(https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/OraclePoolOfferHandler.sol#L193)

This allows the attack vector of a borrower game spot aprPremium change due to pool activities to get lower apr.
(1)
A borrower can back-run a large principal repay with `setAprPremium()` call before taking out a loan(`emitLoan()`). This allows the borrower get a lower apr taking advantage of a sudden drop in utilization ratio.
(2)
A borrower can also front-run a large principal borrow with `setAprPremium()` call and back-run the borrow with `emitLoan()`. This ensures the spiked utilization in the pool will not increase aprPremium at the time `emitLoan()` tx settles.

Recommendations:
Consider updating aprPremium atomically in validateOffer.

### Low-15: Current `afterCallerAdded()` hook will approve caller `type(uint256).max` regardless of whether the caller is a liquidator or a loan contract.
**Instances(1)**
In src/lib/pools/Pool.sol, accepted callers can be either a loan contract or a liquidator. Currently `afterCallerAdded()` will approve `type(uint256).max` assets to both a loan or liquidator contract. This is unnecessary since a liquidator contract doesn't pull assets from the pool.

```solidity
//src/lib/pools/Pool.sol
    function addCallers(ProposedCaller[] calldata _callers) external {
        if (msg.sender != getParameterSetter) {
            revert InvalidCallerError();
        }
        uint256 totalCallers = _callers.length;
        for (uint256 i = 0; i < totalCallers;) {
            ProposedCaller calldata caller = _callers[i];
            _acceptedCallers.add(caller.caller);
            _isLoanContract[caller.caller] = caller.isLoanContract;

|>          afterCallerAdded(caller.caller);
            unchecked {
                ++i;
            }
        }

        emit CallersAdded(_callers);
    }

    function afterCallerAdded(address _caller) internal override {
        asset.approve(_caller, type(uint256).max);
    }
```
(https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/LoanManager.sol#L66)

Recommendations:
Consider only approve type(uint256).max for loan contracts.

### Low-16: Unused library import.
**Instances(1)**
FixedPointMathLib is imported in ERC4626.sol but no longer used.
```solidity
//src/lib/pools/ERC4626.sol
|> import {FixedPointMathLib} from "@solmate/utils/FixedPointMathLib.sol";

/// @notice Fork from Solmate (https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC4626.sol)
///        to allow extra decimals.
/// @author Solmate (https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC4626.sol)
abstract contract ERC4626 is ERC20 {
    using Math for uint256;
    using SafeTransferLib for ERC20;
    using FixedPointMathLib for uint256; //@audit Low: Unused library import. Remove unused library.
...
```
(https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/ERC4626.sol#L7)

Recommendations:
Remove unused library.

### Low-17: Unused constant declaration
**Instances(1)**
`_PRINCIPAL_PRECISION` is not used in LidoEthBaseInterestAllocator.sol.
```solidity
//src/lib/pools/LidoEthBaseInterestAllocator.sol

    uint256 private constant _PRINCIPAL_PRECISION = 1e20;

```
(https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/LidoEthBaseInterestAllocator.sol#L29)

Recommendations:
Remove `_PRINCIPAL_PRECISION`.

### Low-18: In edge cases, LidoEthBaseInterestAllocator's aprBps might be updated to 0.
**Instances(1)**
In LidoEthBaseInterestAllocator.sol, `getBaseAprWithUpdate()` checks at least minimal time (getLidoUpdateTolerance) has passed before revising aprBps.

However, there is no garuantee rebasing will occur before `getLidoUpdateTolerance`.  If rebasing didn’t occur before `getLidoUpdateTolerance`, shareRate might not change. In `_updateLidoValues()`, _lidoData.aprBos will be set to 0, which is an invalid value.

```solidity
    function getBaseAprWithUpdate() external returns (uint256) {
        LidoData memory lidoData = getLidoData;
|>      if (block.timestamp - lidoData.lastTs > getLidoUpdateTolerance) {
            _updateLidoValues(lidoData);
        }
...

    function _updateLidoValues(LidoData memory _lidoData) private {
        uint256 shareRate = _currentShareRate();
|>      _lidoData.aprBps = uint16(
            _BPS * _SECONDS_PER_YEAR * (shareRate - _lidoData.shareRate) / _lidoData.shareRate
                / (block.timestamp - _lidoData.lastTs)
        );
...

```
(https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/LidoEthBaseInterestAllocator.sol#L163)

Recommendations:
In _updateLidoValues(), consider adding a check to ensure shareRate > _lidoData.shareRate before calculating the new aprBps.


