---
sponsor: "Gondi"
slug: "2024-06-gondi"
date: "2024-07-25"
title: "Gondi Invitational"
findings: "https://github.com/code-423n4/2024-06-gondi-findings/issues"
contest: 394
---

# Overview

## About C4

Code4rena (C4) is an open organization consisting of security researchers, auditors, developers, and individuals with domain expertise in smart contracts.

A C4 audit is an event in which community participants, referred to as Wardens, review, audit, or analyze smart contract logic in exchange for a bounty provided by sponsoring projects.

During the audit outlined in this document, C4 conducted an analysis of the Gondi smart contract system written in Solidity. The audit took place between June 14 — July 5, 2024.

## Wardens

In Code4rena's Invitational audits, the competition is limited to a small group of wardens; for this audit, 2 wardens contributed reports:

  1. [oakcobalt](https://code4rena.com/@oakcobalt)
  2. [minhquanym](https://code4rena.com/@minhquanym)

This audit was judged by [0xsomeone](https://code4rena.com/@0xsomeone).

Final report assembled by [thebrittfactor](https://twitter.com/brittfactorC4).

# Summary

The C4 analysis yielded an aggregated total of 6 unique vulnerabilities. Of these vulnerabilities, 3 received a risk rating in the category of HIGH severity and 3 received a risk rating in the category of MEDIUM severity.

Additionally, C4 analysis included 2 reports detailing issues with a risk rating of LOW severity or non-critical.

All of the issues presented here are linked back to their original finding.

# Scope

The code under review can be found within the [C4 Gondi repository](https://github.com/code-423n4/2024-06-gondi), and is composed of 29 smart contracts written in the Solidity programming language and includes 4117 lines of Solidity code.

# Severity Criteria

C4 assesses the severity of disclosed vulnerabilities based on three primary risk categories: high, medium, and low/non-critical.

High-level considerations for vulnerabilities span the following key areas when conducting assessments:

- Malicious Input Handling
- Escalation of privileges
- Arithmetic
- Gas use

For more information regarding the severity criteria referenced throughout the submission review process, please refer to the documentation provided on [the C4 website](https://code4rena.com), specifically our section on [Severity Categorization](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization).

# High Risk Findings (3)
## [[H-01] Lido's apr can be maliciously updated to 0 value due to missing `getLidoUpdateTolerance` check, DOS pool lending](https://github.com/code-423n4/2024-06-gondi-findings/issues/9)
*Submitted by [oakcobalt](https://github.com/code-423n4/2024-06-gondi-findings/issues/9)*

Function `updateLidoValues()` is permissionless and missing the [`getLidoupdateTolerance` check](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/LidoEthBaseInterestAllocator.sol#L95). Lido’s stETH [rebases daily](https://docs.lido.fi/guides/lido-tokens-integration-guide#accounting-oracle). A malicious actor can call `updateLidoValues()` between rebasing to directly set `lidoData.aprBps` to 0. Once set, this DOS pool lending, because calling `getBaseAprWithUpdate()` will always revert with `InvalidAprError`.

```solidity
//src/lib/pools/LidoEthBaseInterestAllocator.sol
    //@audit anyone can call updateLidoValues at any time, risks of _lidoData.aprBps being set to 0.
    function updateLidoValues() external {
        _updateLidoValues(getLidoData);
    }

    function _updateLidoValues(LidoData memory _lidoData) private {
        uint256 shareRate = _currentShareRate();
        _lidoData.aprBps = uint16(
            (_BPS * _SECONDS_PER_YEAR * (shareRate - _lidoData.shareRate)) /
                _lidoData.shareRate /
                (block.timestamp - _lidoData.lastTs)
        );
        _lidoData.shareRate = uint144(shareRate);
        _lidoData.lastTs = uint96(block.timestamp);
        getLidoData = _lidoData;
        emit LidoValuesUpdated(_lidoData);
    }
```

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/LidoEthBaseInterestAllocator.sol#L105-L106

See added unit test:

```solidity
//test/pools/LidoEthBaseInterestAllocator.t.sol
...
    function testMaliciousUpdateLidoValues() public {
        assertEq(_baseAllocator.getBaseAprWithUpdate(), 1000);
        (uint96 lastTs, , ) = _baseAllocator.getLidoData();
        vm.warp(uint256(lastTs) + 12);
        _baseAllocator.updateLidoValues();
        (, , uint16 newAprBps) = _baseAllocator.getLidoData();
        assertEq(newAprBps, 0);
        vm.expectRevert(abi.encodeWithSignature("InvalidAprError()"));
        _baseAllocator.getBaseAprWithUpdate();
    }
...
```

```
Ran 1 test for test/pools/LidoEthBaseInterestAllocator.t.sol:LidoEthBaseInterestAllocatorTest
[PASS] testMaliciousUpdateLidoValues() (gas: 30835)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.93ms (486.17µs CPU time)
```

### Recommended Mitigation Steps

In `updateLidoValues()`, consider adding check to only run `_updateLidoValues()` when `(block.timestamp - lidoData.lastTs > getLidoUpdateTolerance)`.

**[0xend (Gondi) confirmed](https://github.com/code-423n4/2024-06-gondi-findings/issues/9#event-13414822150)**

**[0xsomeone (judge) increased severity to High and commented](https://github.com/code-423n4/2024-06-gondi-findings/issues/9#issuecomment-2220116640):**
 > The Warden outlines how the lack of a timestamp-related validation check when updating Lido's APR permits the APR measurements from the `LidoEthBaseInterestAllocator` to revert due to the relevant metric becoming `0`.
> 
> A malicious user can repeatedly perform this attack at will by following any valid APR measurement by a re-measurement causing it to always be `0`. In turn, this would lead to the pool not being able to validate new offers and effectively result in a permanent Denial-of-Service with significant impact.
> 
> As a result of the above analysis, I believe a high-risk rating is appropriate for this submission given that an important feature of the protocol can be repetitively denied for an indeterminate amount of time.

***

## [[H-02] OraclePoolOfferHandler's `_getFactors` allows exact `tokenId` offer terms to be used for collection offers. A borrower can take on a loan with incorrect terms](https://github.com/code-423n4/2024-06-gondi-findings/issues/7)
*Submitted by [oakcobalt](https://github.com/code-423n4/2024-06-gondi-findings/issues/7)*

In MultiSourceLoan.sol, a loan `offerExectuion` can be either a collection offer or an exact collateral token Id match offer. In a collection offer, the loanOffer's `collateralTokenId` doesn't have to match the borrower-supplied token id.

**1. `MutlisSourceLoan::_checkValidators`:**

The condition checks for a collection offer is [one empty validator address](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L881-L882) in struct LoanOffer. (`(totalValidators == 1) && _loanOffer.validators[0].validator == address(0)`).

The condition check for exact Id match is either [`loanOffer.nftCollateralTokenId != 0`](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L873-L875) or [`loanOffer.nftCollateralTokenId == 0 && totalValidators.length == 0`](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L879).

**2. `OraclePoolOfferHandler::_getFactors`:**

The condition checks for a collection offer is [`totalValidators.length == 0`](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/OraclePoolOfferHandler.sol#L351-L352). This is the opposite of number 1.

The condition checks for an exact Id offer is [one empty validator address](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/OraclePoolOfferHandler.sol#L353). (`_validators.length == 1 && _isZeroAddress(_validators[0].validator)`) Also the opposite of number 1.

`MutlisSourceLoan::_checkValidators` and `OraclePoolOfferHandler::_getFactors` are invoked in the same `_validateOfferExecution()` call: 

( `MultiSourceLoan::emitLoan()->_processOffersFromExecutionData()-> _validateOfferExecution() -> Pool.validateOffer() / _checkValidators()`)

Impacts: A borrower's exact token id match OfferExecution will be validated against a collection offer term. Or vice versa, a borrower's collection `offerExecution` will be validated against any specific token-based term.

Suppose:
- `_hashKey(wrappedPunk, 30 days, tokenId 0)`'s principal factor is (50%, 25%).
- `_hashKey(wrappedPunk, 30 days, tokened 1)`'s principal factor is (33%, 16.67%).
- `_hashKey(wrappedPunk, 30 days, "")`'s principal factor is (25%, 12.5%).

`wrappedPunk`'s `currentFloor.value = historicalFloor.value = 1000` usd.

`MaxPrincipal`:
- `tokenId` 0: 250 usd
- `tokenId` 1: 166.67 usd
- collection offer: 125 usd

Alice can borrow using a cheap `tokenId` 1 as collateral, but with a LoanOffer based on `tokenId` 0 with a higher `MaxPrincipal`.

Alice's `ExecutionData`:
- `ExecutionData.tokenId` = 1,
- `ExecutionData.OfferExecution\[0].LoanOffer.nftCollateralTokenId` = 0,
- `ExecutionData.OfferExecution\[0].LoanOffer.validators.length` == 1,
- `ExecutionData.OfferExecution\[0].LoanOffer.validators\[0]` = `address(0)`,
- `ExecutionData.OfferExecution\[0].LoanOffer.validators.arguments.code` = 3, (individual)
- `ExecutionData.OfferExecution\[0].LoanOffer.validators.arguments.data` = uint(0),(`tokenId` 0)
- `ExecutionData.OfferExecution\[0].amount` = 250 usd

Alice's emitLoan tx succeeds. She transferred a cheaper token 1 but got 250 usd principal.

Bob uses a more expansive `tokenId` 0 as collateral for higher principal. Bob needs an exact `tokenId` offer term match so his `validators.length == 0`. 

Bob's `ExeuctionData`:
- `ExecutionData.tokenId` = 0,
- `ExecutionData.OfferExecution\[0].LoanOffer.nftCollateralTokenId` = 0,
- `ExecutionData.OfferExecution\[0].LoanOffer.validators.length` == 0,
- `ExecutionData.OfferExecution\[0].amount` = 250 usd

Bob's emitLoan tx reverted. Even though he has the expensive token 0, his offer is validated as a collection offer with a max 125 usd allowance.

### Recommended Mitigation Steps

In `_getFactors()`, consider swaping the if condition `_validator.length==0` with `_validators.length == 1 && _isZeroAddress(_validators[0].validator)`.

### Assessed type

Error

**[0xend (Gondi) confirmed](https://github.com/code-423n4/2024-06-gondi-findings/issues/7#event-13414606817)**

**[0xsomeone (judge) commented](https://github.com/code-423n4/2024-06-gondi-findings/issues/7#issuecomment-2220143528):**
 > The Warden has demonstrated how the offer type conditionals across the codebase are inconsistent permitting a token ID-specific offer type to be used for a collection-level offer and vice versa.
> 
> I believe a high-risk severity rating is appropriate given that the vulnerability demonstrated can be utilized to cause unfair loans to be taken out at the expense of lenders, and a significant feature of the protocol is impacted. 

*Note: For full discussion, see [here](https://github.com/code-423n4/2024-06-gondi-findings/issues/7).*

***

## [[H-03] `OraclePoolOfferHandler` has an invalid check on `collateralTokenId`, incorrect loan offers can pass validation](https://github.com/code-423n4/2024-06-gondi-findings/issues/6)
*Submitted by [oakcobalt](https://github.com/code-423n4/2024-06-gondi-findings/issues/6)*

In `OraclePoolOfferHandler::validateOffer`, loan offer terms can be validated in [three ways](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/OraclePoolOfferHandler.sol#L19-L24):
1. Range (collateral tokens ids in a certain range will have a corresponding [`PrincipalFactor` used to calculate `maxPrincipal` allowed](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/OraclePoolOfferHandler.sol#L315)).
2. Merkle root.
3. Individual token id.

In the case of a range validation (`validationData.code == 1`), `_getFactors()` has an incorrect check implementation. The check will not revert when the `collateralTokenId` is out of range.

```solidity
//src/lib/pools/OraclePoolOfferHandler.sol
    function _getFactors(
        address _collateralAddress,
        uint256 _collateralTokenId,
        uint256 _duration,
        IBaseLoan.OfferValidator[] memory _validators
    ) private view returns (PrincipalFactors memory) {
        bytes32 key;
        if (_validators.length == 0) {
...
        } else if (
            _validators.length == 1 && _isZeroAddress(_validators[0].validator)
        ) {
            PrincipalFactorsValidationData memory validationData = abi.decode(
                _validators[0].arguments,
                (PrincipalFactorsValidationData)
            );
            if (validationData.code == 1) {
                 // Range
                (uint256 min, uint256 max) = abi.decode(validationData.data, (uint256, uint256));
                //@audit invalid check condition. Should be `_collateralTokenId < min || _collateralTokenId > max`
|>              if (_collateralTokenId < min && _collateralTokenId > max) {
                    revert InvalidInputError();
                }
                key = _hashKey(_collateralAddress, uint96(_duration), validationData.data);
...
        return _principalFactors[key];
```

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/OraclePoolOfferHandler.sol#L359-L360

As seen, an out-of-range `collateralTokenId` can pass the check, resulting in invalid `principalFactors` used in loan validation.

```solidity
//src/lib/pools/OraclePoolOfferHandler.sol
    function validateOffer(uint256 _baseRate, bytes calldata _offer)
        external
        view
        override
        returns (uint256, uint256)
    {
...
        PrincipalFactors memory factors = _getFactors(
            offerExecution.offer.nftCollateralAddress,
            offerExecution.offer.nftCollateralTokenId,
            duration,
            offerExecution.offer.validators
        );

        uint128 maxPrincipalFromCurrentFloor = uint128(uint256(currentFloor.value).mulDivDown(factors.floor, PRECISION));
        uint128 maxPrincipalFromHistoricalFloor =
            uint128(uint256(historicalFloor.value).mulDivDown(factors.historicalFloor, PRECISION));
        uint256 maxPrincipal = maxPrincipalFromCurrentFloor > maxPrincipalFromHistoricalFloor
            ? maxPrincipalFromHistoricalFloor
            : maxPrincipalFromCurrentFloor;
        //@audit-info maxPrincipal is based on returned principalFactors from _getFactors. Invalid range check in _getFactors() compromises validateOffer
|>      if (offerExecution.amount > maxPrincipal) {
            revert InvalidPrincipalAmountError();
        }
...
```

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/OraclePoolOfferHandler.sol#L329-L330

### Recommended Mitigation Steps

Change the check condition into `if(_collateralTokenId < min || _collateralTokenId > max) {//revert`.

### Assessed type

Error

**[0xsomeone (judge) commented](https://github.com/code-423n4/2024-06-gondi-findings/issues/6#issuecomment-2220161790):**
 > This submission was initially mistakenly set as a duplicate of [#7](https://github.com/code-423n4/2024-06-gondi-findings/issues/7); however, upon closer inspection, it is a distinct vulnerability.
> 
> Specifically, the `min` and `max` range checks performed by the `OraclePoolOfferHandler::_getFactors` function appear to be invalid. This will permit a collection-level offer that has a specific range of token IDs to have its restriction bypassed.
> 
> I had requested the Sponsor's feedback for this submission on exhibit #7 and have queried for it again to ensure a fully informed judgment is issued for this submission. However, I am inclined to keep it as a valid high-risk vulnerability at this point based on similar rationale as the one laid out in #7; significant functionality of the protocol is impacted and the vulnerability can be exploited for profit at the expense of users by supplying less lucrative token IDs.

**[0xend (Gondi) confirmed](https://github.com/code-423n4/2024-06-gondi-findings/issues/6#event-13459919778)**

***
 
# Medium Risk Findings (3)
## [[M-01] Anyone can DOS `OraclePoolOfferHandler` setting `collectionFactors`](https://github.com/code-423n4/2024-06-gondi-findings/issues/4)
*Submitted by [oakcobalt](https://github.com/code-423n4/2024-06-gondi-findings/issues/4)*

In OraclePoolOfferHandler.sol, `collecitonFactors` are set in a two-step process. The owner will propose collection factors through `setCollectionFactors()`, which writes to [`getProposedCollectionFactors`](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/OraclePoolOfferHandler.sol#L237) mapping. After a minimal delay, anyone can call `confirmCollectionFactors` with collection data to confirm proposed values.

The problem is, `confirmCollectionFactors` has insufficient checks on caller-supplied collection data. Currently, only the array lengths submitted are verified. There is no check on whether each array element is duplicated, non-zero, or an already confirmed value.

```solidity
//src/lib/pools/OraclePoolOfferHandler.sol
    function confirmCollectionFactors(
        address[] calldata _collection,
        uint96[] calldata _duration,
        bytes[] calldata _extra,
        PrincipalFactors[] calldata _factor
    ) external {
...
        uint256 updates = _collection.length;
        //@audit-info note: only array length is checked
        if (
|>          getTotalUpdatesPending != updates ||
            updates != _duration.length ||
            updates != _factor.length ||
            updates != _extra.length
        ) {
            revert InvalidInputLengthError();
        }

        for (uint256 i; i < updates; ) {
            bytes32 key = _hashKey(_collection[i], _duration[i], _extra[i]);
            PrincipalFactors
                memory proposedFactor = getProposedCollectionFactors[key];
            //@audit getProposeCollectionFactors is a mapping that is never reset, a caller can pass arrays with 0 value elements, which will pass the check. Caller can also pass already confirmed keys and values.
            if (
|>              proposedFactor.floor != _factor[i].floor ||
                proposedFactor.historicalFloor != _factor[i].historicalFloor
            ) {
                revert InvalidInputError();
            }
            _principalFactors[key] = proposedFactor;
            unchecked {
                ++i;
            }
        }
        //@audit After setting 0 values, or already confirmed values, the TS reset. The owner has to propose again. 
|>      getProposedCollectionFactorsSetTs = type(uint256).max;
        getTotalUpdatesPending = 0;
...
```

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/OraclePoolOfferHandler.sol#L265-L266

Because `getProposedCollectionFactors` mapping is never cleared, there are multiple ways to exploit:
1. The caller can call `confirmCollectionFactors()` with 0 values array.
2. The caller can call with already confirmed key and value pairs.
3. The caller can set duplicated values; After the call `getProposedCollectionFactorsSetTs` is reset, and the owner has to propose factors again.

### Recommended Mitigation Steps

Consider storing proposed keys in a transient storage array. `ConfirmCollectionsFactors()` only need to iterate over the storage key array.


**[0xend (Gondi) confirmed](https://github.com/code-423n4/2024-06-gondi-findings/issues/4#event-13414642037)**

**[0xsomeone (judge) commented](https://github.com/code-423n4/2024-06-gondi-findings/issues/4#issuecomment-2220188840):**
 > The Warden has outlined how the proposal and confirmation functions can become disconnected given that there is insufficient input validation when confirming a collection's factors. 
> 
> I believe a medium-risk rating is appropriate for this submission given that no special privileges are necessary to exploit it, and an important function of the system albeit not critical is impacted and can be denied repetitively.

***

## [[M-02] `AprPremium` calculation might be incorrect due to loss of precision](https://github.com/code-423n4/2024-06-gondi-findings/issues/3)
*Submitted by [oakcobalt](https://github.com/code-423n4/2024-06-gondi-findings/issues/3)*

In OraclePoolOfferHandler.sol, `AprPremium` is calculated partially based on the ratio of total outstanding assets over total assets. This issue is the ratio is not scaled properly to BPS decimals before adding to `aprFactors.minPremium`.

Based on code comments, `utilizationFactor` is in `PRECISION` (`1e27`), and `minPremium` is in `BPS(1e4)`.

```solidity
    /// @notice UtilizationFactor Expressed in `PRECISION`. minPremium in BPS
    struct AprFactors {
        uint128 minPremium;
        uint128 utilizationFactor;
    }
```

In `_calculateAprPremium()`, `(totalOutstanding.mulDivUp(aprFactors.utilizationFactor, totalAssets * PRECISION)` is not scaled to BPS before adding `aprFactors.minPremium`.

```solidity
    function _calculateAprPremium() private view returns (uint128) {
        /// @dev cached
        Pool pool = Pool(getPool);
        AprFactors memory aprFactors = getAprFactors;
        uint256 totalAssets = pool.totalAssets();
        uint256 totalOutstanding = totalAssets - pool.getUndeployedAssets();
        return uint128(
|>          totalOutstanding.mulDivUp(aprFactors.utilizationFactor, totalAssets * PRECISION) + aprFactors.minPremium
        );
    }
```

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/OraclePoolOfferHandler.sol#L396

**Examples**

1. `deployedAssets = 1e18`, `totalAssets = 4e18`, `aprFactors.utilizationFactor = 0.01e27`, `aprFactor.minPremium = 500` (5%), `PRECISION = 1e27`.
    - Calculated apr premium = 1 + 500 = 501
    - Expected apr premium: 25 + 500 = 525

2. `deployedAssets = 3.9e18`, `totalAssets = 4e18`, `aprFactors.utilizationFactor = 0.01e27`, `aprFactors.minPremium = 500` (5%), `PRECISION = 1e27`.
    - Calculated apr premium = 1 + 500 = 501
    - Expected apr premium: 98 + 500 = 598

### Recommended Mitigation Steps

Scale up `totalOutstanding` by 1e4 before performing division and addition.

**[0xend (Gondi) confirmed](https://github.com/code-423n4/2024-06-gondi-findings/issues/3#event-13414586503)**

**[0xsomeone (judge) commented](https://github.com/code-423n4/2024-06-gondi-findings/issues/3#issuecomment-2220202556):**
 > The Warden outlines how inconsistent units are employed in the APR premium calculations of the `OraclePoolOfferHandler`, resulting in a lower-than-expected APR in all circumstances.
> 
> I believe a medium-risk severity rating is appropriate given that the miscalculation would permit offers with a higher APR than expected to be validated and thus higher interest loans to be processed by the system as valid.

***

## [[M-03] Delegations cannot be removed in some cases due to vulnerable `revokeDelegate()` implementation](https://github.com/code-423n4/2024-06-gondi-findings/issues/2)
*Submitted by [oakcobalt](https://github.com/code-423n4/2024-06-gondi-findings/issues/2)*

An old borrower can use an old delegation to claim on behalf of a new borrower.

### Proof of Concept

A borrower can delegate locked collateral NFT through `delegateRegistry` to prove token ownership and claim airdrops, event ticketing, etc.

[`delegateRegistry`](https://etherscan.io/address/0x00000000000000447e69651d841bD8D104Bed493#code) by Delegate.Cash protocol allows custom rights to be configured to a delegatee.

Currently, `MultiSourceLoan::delegate` allows a borrower to configure `bytes32 _rights` to `delegateERC721`. In `DelegateRegistry::delegateERC721`, `bytes32 rights` will be [hashed as part of the key](https://github.com/delegatexyz/delegate-registry/blob/ce89e65f9364db21fc621e247a829d9c08374b4e/src/DelegateRegistry.sol#L83) to store delegation data.

```solidity
//src/lib/loans/MultiSourceLoan.sol
    function delegate(uint256 _loanId, Loan calldata loan, address _delegate, bytes32 _rights, bool _value) external {
        if (loan.hash() != _loans[_loanId]) {
            revert InvalidLoanError(_loanId);
        }
        if (msg.sender != loan.borrower) {
            revert InvalidCallerError();
        }
        //@audit-info a borrower can pass custom rights to delegateERC721
|>      IDelegateRegistry(getDelegateRegistry).delegateERC721(
            _delegate, loan.nftCollateralAddress, loan.nftCollateralTokenId, _rights, _value
        );

        emit Delegated(_loanId, _delegate, _value);
    }
```

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L484

The problem is, in `MultiSourceLoan::revokeDelegate`, empty rights will always be passed to `delegateERC721`. This means when a borrower configures custom rights in `delegate()`, they cannot remove the delegation. In `DelegateRegistry::delegateERC721`, empty rights will be hashed into a different key from the borrower's actual delegation. [Incorrect delegation data will be read](https://github.com/delegatexyz/delegate-registry/blob/ce89e65f9364db21fc621e247a829d9c08374b4e/src/DelegateRegistry.sol#L83-L85) and `delegateERC721` call will return with no change.

```solidity
//src/lib/loans/MultiSourceLoan.sol
    function revokeDelegate(address _delegate, address _collection, uint256 _tokenId) external {
        if (ERC721(_collection).ownerOf(_tokenId) == address(this)) {
            revert InvalidMethodError();
        }
        //@audit revokeDelegate will always pass empty rights.
|>      IDelegateRegistry(getDelegateRegistry).delegateERC721(_delegate, _collection, _tokenId, "", false);

        emit RevokeDelegate(_delegate, _collection, _tokenId);
    }
```

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L496

**POC:**

1. The original borrower set custom rights and delegated their collateral NFT to a custom contract.
2. The original borrower's loan ended and NFT is transferred to a new borrower.
3. The protocol or the new borrower calls `revokeDelegate()` to remove previous delegations of the NFT.
4. The new borrower takes out a loan with the NFT and calls `delegate()`, delegating the NFT to a hot wallet.
5. The original borrower's old delegation is not cleared from `delegateRegistry` and still claims an event ticket using the old delegation. The old borrower claims the new borrower's ticket.

### Recommended Mitigation Steps

In `revokeDelegate()`, allow passing `bytes32 _rights` into `delegateERC721()` to correctly revoke existing delegations with custom rights.

### Assessed type

Error

**[0xend (Gondi) confirmed](https://github.com/code-423n4/2024-06-gondi-findings/issues/2#event-13414538986)**

**[0xsomeone (judge) commented](https://github.com/code-423n4/2024-06-gondi-findings/issues/2#issuecomment-2220217025):**
 > The Warden has outlined how the protocol will incorrectly integrate with the `DelegateRegistry` system, attempting to revoke a previous delegation via an empty payload which is a futile attempt as proper revocation would require the same `_rights` to be passed in with a `false` value for the `_enable` flag.
> 
> I am slightly mixed in relation to this submission as the `MultiSourceLoan::delegate` function can be utilized with a correct payload to remove delegation from the previous user correctly. I believe that users, protocols, etc., will attempt to use the `MultiSourceLoan::revokeDelegate` function to revoke their delegation, and thus, a medium-risk severity rating is appropriate even though a circumvention already exists in the code.
> 
> To note, the code also goes against its `interface` [specification](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/interfaces/loans/IMultiSourceLoan.sol#L257-L261) further re-inforcing a medium-risk rating level.

***

# Low Risk and Non-Critical Issues

For this audit, 2 reports were submitted by wardens detailing low risk and non-critical issues. The [report highlighted below](https://github.com/code-423n4/2024-06-gondi-findings/issues/10) by **oakcobalt** received the top score from the judge.

*The following wardens also submitted reports: [minhquanym](https://github.com/code-423n4/2024-06-gondi-findings/issues/13).*

## [L-01] `Delegated` and `RevokeDelegate` event should have `bytes32 _rights`

**Instances (2)**

When a borrower `delegate` or `revokeDelegate`, custom delegation rights `bytes32 _rights` will not be included in event `Delegated` and event `RevokeDelegate`.

In `DelegateRegistry`, custom rights `bytes32 _rights` are hashed into delegation key when storing delegation data. `bytes32 _rights` is crucial in tracking delegations off-chain.

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

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L487

```solidity
//src/lib/loans/MultiSourceLoan.sol
    function revokeDelegate(address _delegate, address _collection, uint256 _tokenId) external {
...
|>        emit RevokeDelegate(_delegate, _collection, _tokenId);
```

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L498

### Recommendations

Consider adding `bytes32 _rights` field in `Delegated` and `RevokeDelegate` event.

## [L-02] `_checkStrictlyBetter` might underflow revert without triggering the custom error

In `MutliSourceLoan::refinanceFull()`, if lender initiated the refinance, `_checkStrictlyBetter()` will run to ensure if the lender provided more principal; the annual interest with the new principal is still better than existing loan. If it's not strictly better, the tx should throw with a custom error `NotStrictlyImprovedError()`. However, the custom error might not be triggered due to an earlier underflow error.

If `_offerPrincipalAmount < _loanPrincipalAmount`, or if `_loanAprBps * _loanPrincipalAmount < _offerAprBps * _offerPrincipalAmount`, the tx will throw without triggering the custom error.

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

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L1112-L1116

### Recommendations

Check for underflow and revert with custom error early.

## [L-03] Unnecessary math operation when `_remainingNewLender` is set to type(uint256).max in the refinance flow

When a lender initiates `refinanceFull()` or `refinancePartial()`, `_remainingNewLender` [will be set to `type(uint256).max`](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L280), which indicates the lender will repay existing lenders.

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

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L665

### Recommendations

In `_processOldTranche()`, add a check and only perform the subtraction when `_remainingNewLender != type(uint256).max`.

## [L-04] Incorrect spelling

There's one instance of incorrect spelling: `renegotiationIf` should be `renegotiationId`.

```solidity
//src/lib/loans/BaseLoan.sol
    mapping(address user => mapping(uint256 renegotiationIf => bool notActive))
        public isRenegotiationOfferCancelled;
```

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/BaseLoan.sol#L57

### Recommendation

Change `renegotiationIf` into `renegotiationId`.

## [L-05] Pool contract can miss Loan offer origination fees if the borrower submits `emitLoan()`

A lender can specify an optional loan offer origination fee, which will be [charged at loan initiation (`emitLoan`)](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L1012). The origination fee specified in struct `LoanOffer` will be deducted from the borrow principal amount.

If the lender is a pool contract, this origination fee can be skipped by the borrower initiating `emitLoan()` with `OfferExecution.offer.fee` as 0. This fee will not be verified in [the current `pool.validateOffer` flow](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/Pool.sol#L358-L359).

Note that if the protocol submitted the `emitLoan` (e.g., the borrower placed loan request through UI), the protocol can still enforce the origination fee by supplying non-zero `OfferExecution.offer.fee`. This allows a borrower to avoid paying the origination fee for any pool contract lender.

### Recommendation

Either in the pool’s `validateOffer()` or `PoolOfferHandler.validateOffer()`, add a check to ensure the origination fee is satisfied.

## [L-06] Borrower can use an arbitrary `offerId` for a pool contract's loan offer, which might lead to incorrect off-chain accounting.

`offerId` is a field in struct [`LoanOffer`](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/interfaces/loans/IMultiSourceLoan.sol#L26) and `LoanOffer` is typically signed by the lender. Currently, `offerId` is generated off-chain and its correctness is verified through [`_checkSignature(lender, offer.hash()`, `_lenderOfferSignature`)](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L780) in the emitLoan flow.

But when the lender is a pool contract, `offerId` will not be verified due to `_checkSignature()` being [bypassed in `_validateOfferExectuion()`](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L777). 

A borrower can supply any offerIds for a pool lender offer in `emitLoan()` or `refinanceFromLoanExeuctionData()` flow. As a result, `emit LoanEmitted(loanId, offerIds, loan, totalFee)` or `emit LoanRefinancedFromNewOffers(_loanId, newLoanId, loan, offerIds, totalFee)` will contain arbitrary `offerIds`. This may create conflicts in off-chain offerId accounting.

### Recommendation

If `offerId` for a pool lender is relevant, consider allowing a pool contract to increment and store its next offerId on-chain.

## [L-07] Borrowers might use a lender's `addNewTranche` renegotiation offer to `refinanceFull` in some cases

A borrower can use a lender’s renegotiation offer signature for `addNewTranch()` in `refinanceFull()`, as long as the loan only has one tranche.

This is because `refinanceFull()` only checks [whether `_renegotiationOffer.trancheIndex.length == _loan.tranche.length`](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L168). When there's only one tranche in the loan, [`addNewTranche()`'s `renegoationOffer`'s check condition](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L379) will also satisfy.

`refinanceFull()` will also ensure the refinanced tranche [gets a better apr](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L591). So the borrower gets better apr for the existing tranche instead of taking out additional principal in `addNewTranche()`.

In addition, if the lender signed a renegotiation offer intended for `refinanceFull()`, the same offer can also be used for `addNewTranche()` if the condition `_renegotiationOffer.trancheIndex[0] == _loan.tranche.length` is satisfied. Because `_renegotiationOffer.trancheIndex[0]` is never checked in `refinanceFull()` flow, the lender might supply any values. In this case, the lender is forced to open a more junior tranche which can be risky for lenders.

It's better to prevent the same renegotiation offer from being used interchangeably in different methods with different behaviors.

### Recommendation

In `refinanceFull()`, add a check to ensure `_renegotiationOffer.trancheIndex[0]==0`.

## [L-08] Consider adding a cap for `minLockPeriod`

`_minLockPeriod` is used to compute the lock time for a tranche or a loan. If the ratio is set too high (e.g., 10000), tranches or loans cannot be refinanced due to [failing `_loanLocked()` checks](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L181).

```solidity
//src/lib/loans/MultiSourceLoan.sol
    function setMinLockPeriod(uint256 __minLockPeriod) external onlyOwner {

        _minLockPeriod = __minLockPeriod;

        emit MinLockPeriodUpdated(__minLockPeriod);
    }
```

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L508

### Recommendation

Considering adding a cap value (e.g., 10%).
 
## [L-09] `updateLiquidationContract()` might lock collaterals and funds in the current liquidator contract

In `LiquidationHandler::updateLiquidationContract()`, the loan liquidator contract can be updated. 

```solidity
    function updateLiquidationContract(address __loanLiquidator) external override onlyOwner {
        __loanLiquidator.checkNotZero();
        _loanLiquidator = __loanLiquidator;
        emit LiquidationContractUpdated(__loanLiquidator);
    }
```

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/LiquidationHandler.sol#L74

There are two vulnerable conditions: `updateLiquidationContract()` is called when there are ongoing/unsettled auctions in the current liquidator or there might be a pending `liquidateLoan()` tx.

1. If MultisourceLoan's liquidator contract is updated. None of the exiting auctions originated from the MultisourceLoan can be settled because `AuctionLoanLiquidator::settleAuction` will call [`IMultiSourceLoan(_auction.loanAddress).loanLiquidated(...`](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/AuctionLoanLiquidator.sol#L300). This will cause the [`onlyLiquidator` modifier to revert](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L464). MultiSourceLoan contract no longer recognizes the old liquidator contract. The collateral and bid funds will be locked in the old liquidator contract.

2. If there is a pending [`MultiSourceLoan::liquidateLoan`](https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/MultiSourceLoan.sol#L445) tx before `updateLiquidationContract()` call. The auction of the loan will be created right before `updateLiquidationContract()` settles. Similar to number 1, the collateral will be locked in the old liquidator contract. In addition, since `MultiSourceLoan::liquidateLoan()` is permissionless, an attacker can front-run `updateLiquidationContract` tx to cause the loan liquidated to an old liquidator contract.

### Recommendation

1. In `UpdateLiquidationContract()`, consider adding a check that the existing liquidator’s token balance is 0, with no outstanding auction.
2. Only update `liquidationContract` when there are no liquidatable loans.

## [L-10] Unnecessary code - BytesLib methods are not used in this contract or its parent contracts

BytesLib methods are not used in PurchaseBundler.sol or its parent contracts.

```solidity
//src/lib/callbacks/PurchaseBundler.sol
    using BytesLib for bytes;
...
```

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/callbacks/PurchaseBundler.sol#L24

### Recommendation

Consider removing this line.

## [L-11] Some proposed callers might not be confirmed

LoanManagerParameterSetter.sol has a two-step process of adding callers. The issue is `addCallers()` doesn't check whether `_callers.length == proposedCallers.length`. If `_callers.length < proposedCaller.length`, some `proposedCallers`' indexes will not run in the for-loop. `proposedCallers` whose indexes are after callers will not be added as callers.

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

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/LoanManagerParameterSetter.sol#L110

### Recommendation

Add check to ensure `_callers.length == proposedCallers.length`.

## [L-12] Incorrect comments

**Instances (2)**

1. Auction Loan liquidator -> User Vault

```solidity
/// @title Auction Loan Liquidator
/// @author Florida St
/// @notice NFTs that represent bundles.
contract UserVault is ERC721, ERC721TokenReceiver, IUserVault, Owned {
...
```

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/UserVault.sol#L13

2. `address(0)` = ETH -> address(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE) = ETH

```solidity
    /// @notice ERC20 balances for a given vault: token => (vaultId => amount). address(0) = ETH
    mapping(address token => mapping(uint256 vaultId => uint256 amount)) _vaultERC20s;
```

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/UserVault.sol#L33

### Recommendation

Correct comments.

## [L-13] `vaultID`’s NFT/ERC20 bundle can be modified while the loan is outstanding

UserVault.sol allows a user to bundle assets (NFT/ERC20) together in a vault to be used as a collateral NFT. 

According to [doc](https://app.gitbook.com/o/4HJV0LcOOnJ7AVJ77p8e/s/W2WSJrV6PSLWo4p8vIGq/vaults), the intended behavior is `new NFTs cannot be added to the vault unless borrower burn the vault and create a new vaultId with a new bundle of asset`.

This is not currently the case in UserVault.sol. Anyone can deposit ERC20 or ERC721 to an existing `vaultID` at any time. Although this doesn’t decrease assets from the vault, this may increase VaultID assets at any time during lender offer signing, loan outstanding, and loan liquidation auction process.

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

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/UserVault.sol#L152-L158

Increasing the assets of a `vaultId` doesn’t put a loan’s collateralization at risk. However, this may create inconsistencies in lender offers due to `vaultId`‘s changing asset bundle. 

Due to the permissionless deposit process of UserVault.sol, this may also allow a malicious actor to deposit assets to a `vaultID` during auciton to manipulate bidding.

### Recommendation

If the intention is to disallow adding new NFTs to a vault before burning of `vaultId`, consider a two-step deposit and vault mint process: caller deposit assets to a new `vaultId` first before minting the `vaultId` and disallow deposit to a `vaultId` after minting.

## [L-14] `OraclePoolOfferHandler::validateOffer` allows borrowers to game spot `aprPremium` movement to get lower apr

In OraclePoolOfferHandler.sol, `aprPremium` is partially based on current pool utilization (`totalOutstanding`/`totalAssets`). 

In `OraclePoolOfferHandler::validateOffer`, `aprPremium` will not re-calculate unless min time interval (`getAprUpdateTolerance`) has passed. At the same time, anyone can call `setAprPremium()` to update `aprPremium` instantly.

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

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/OraclePoolOfferHandler.sol#L193

This allows the attack vector of a borrower game spot `aprPremium` change due to pool activities to get lower apr.
1. A borrower can back-run a large principal repay with `setAprPremium()` call before taking out a loan (`emitLoan()`). This allows the borrower get a lower apr taking advantage of a sudden drop in utilization ratio.
2. A borrower can also front-run a large principal borrow with `setAprPremium()` call and back-run the borrow with `emitLoan()`. This ensures the spiked utilization in the pool will not increase `aprPremium` at the time `emitLoan()` tx settles.

### Recommendation

Consider updating `aprPremium` atomically in `validateOffer`.

## [L-15] Current `afterCallerAdded()` hook will approve caller `type(uint256).max` regardless of whether the caller is a liquidator or a loan contract.

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

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/loans/LoanManager.sol#L66

### Recommendation

Consider only approve `type(uint256).max` for loan contracts.

## [L-16] Unused library import

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

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/ERC4626.sol#L7

### Recommendation

Remove unused library.

## [L-17] Unused constant declaration

`_PRINCIPAL_PRECISION` is not used in LidoEthBaseInterestAllocator.sol.

```solidity
//src/lib/pools/LidoEthBaseInterestAllocator.sol

    uint256 private constant _PRINCIPAL_PRECISION = 1e20;
```

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/LidoEthBaseInterestAllocator.sol#L29

### Recommendation

Remove `_PRINCIPAL_PRECISION`.

## [L-18] In edge cases, LidoEthBaseInterestAllocator's `aprBps` might be updated to 0

In LidoEthBaseInterestAllocator.sol, `getBaseAprWithUpdate()` checks at least minimal time (`getLidoUpdateTolerance`) has passed before revising `aprBps`.

However, there is no guarantee rebasing will occur before `getLidoUpdateTolerance`. If rebasing didn’t occur before `getLidoUpdateTolerance`, `shareRate` might not change. In `_updateLidoValues()`, `_lidoData.aprBos` will be set to 0, which is an invalid value.

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

https://github.com/code-423n4/2024-06-gondi/blob/ab1411814ca9323c5d427739f5771d3907dbea31/src/lib/pools/LidoEthBaseInterestAllocator.sol#L163

### Recommendation

In `_updateLidoValues()`, consider adding a check to ensure `shareRate > _lidoData.shareRate` before calculating the new `aprBps`.

***

# Disclosures

C4 is an open organization governed by participants in the community.

C4 audits incentivize the discovery of exploits, vulnerabilities, and bugs in smart contracts. Security researchers are rewarded at an increasing rate for finding higher-risk issues. Audit submissions are judged by a knowledgeable security researcher and solidity developer and disclosed to sponsoring developers. C4 does not conduct formal verification regarding the provided code but instead provides final verification.

C4 does not provide any guarantee or warranty regarding the security of this project. All smart contract software should be used at the sole risk and responsibility of users.
