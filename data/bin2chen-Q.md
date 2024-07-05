## Findings Summary

| Label | Description |
| - | - |
| [L-01] | ProtocolFee.fraction must be limited to >0|


## [L-01] ProtocolFee.fraction must be limited to >0
Currently the protocol does not restrict `ProtocolFee` from being 0
If the protocol fee is 0, it can cause a lot of problems

Take `addNewTranche()` as an example, suppose `ProtocolFee==0`

Then `borrower` can maliciously construct a very large `_renegotiationOffer.principalAmount == renegotiationOffer.fee == type(uint256).max - loan.principalAmount `
(new lender is also himself)

This way: 
1. borrower pay : `_renegotiationOffer.principalAmount - renegotiationOffer.fee = 0`
2. need pay `protocolFee = renegotiationOffer.fee * protocolFee = 0`

`addNewTranche()` works. A large `principalAmount` can lead to an overflow during liquidation.

suggest:
check `ProtocolFee>0`