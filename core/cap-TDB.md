## Preamble

```
CAP: To Be Assigned
Title: Transferable AMM Shares
Working Group:
    Owner: Alex Mootz <@mootz12>
    Authors: Alex Mootz <@mootz12>
    Consulted: Markus Paulson-Luna <@markuspluna>, Fsodano <plz-add-your-info>
Status: Draft
Created: 2022-06-22
Discussion: TBD
Protocol version: TBD
```

## Simple Summary
Allow the transfer of liquidity pool shares.

## Motivation
Smart contracts across the greater crypto ecosystem benefit from transferable liquidity pool (LP) shares. Notably, decentralized staking and lending protocols make effective use of LP shares. Staking contracts reward users who stake LP shares for providing liquidity to a given token pair. Lending protocols support using LP shares as collateral which increases capital efficiency and liquidity.

CAP-0038 introduced AMM's, but omitted LP share transferability due to the lack of use cases on Stellar. With the introduction of smart contract support on Stellar, this is no longer the case.

### Goals Alignment
This CAP is aligned with the following Stellar Network Goals:

  - The Stellar Network should make it easy for developers of Stellar projects to create highly usable products

## Abstract
This proposal adds transferability to liquidity pool shares. The `Payment`, `CreateClaimableBalance`, and `ClaimClaimableBalance` are modified to support transferring of pool shares. Liquidity pool shares are not able to be deposited into liquidity pools, or used to create offers. Lastly, liquidity pool shares are modified to support CAP-0054 asset interoperability on Jump Cannon.

## Specification

### XDR changes
TODO: Determine if `ChangeTrustAsset` can be removed.

This patch of XDR changes is based on the XDR files in commit (`91336c90e59c1be0584324e443bd44520a3a4ecf`) of stellar-core.

```diff mddiffcheck.base=91336c90e59c1be0584324e443bd44520a3a4ecf
diff --git a/src/protocol-curr/xdr/Stellar-ledger-entries.x b/src/protocol-curr/xdr/Stellar-ledger-entries.x
index 3eb578f1..ac76a644 100644
--- a/src/protocol-curr/xdr/Stellar-ledger-entries.x
+++ b/src/protocol-curr/xdr/Stellar-ledger-entries.x
@@ -65,6 +65,9 @@ case ASSET_TYPE_CREDIT_ALPHANUM4:
 case ASSET_TYPE_CREDIT_ALPHANUM12:
     AlphaNum12 alphaNum12;
 
+case ASSET_TYPE_POOL_SHARE:
+    PoolID liquidityPoolID;
+
     // add other asset types here in the future
 };
 ```

### Semantics

#### ChangeTrustOp
The `CHANGE_TRUST_MALFORMED` result code is thrown whenever `asset.type() == ASSET_TYPE_POOL_SHARE` while `tryIncrementPoolUseCount` is run. This modification prevents trustlines being created for liquidity pools composed of `ASSET_TYPE_POOL_SHARE` assets.

Strict enforcement on the management of pool share trustlines allows the other operations the ability to trust a pool share trustline as an OK for performing an action. Most importantly, this enables `LiquidityPoolDepositOp` to never be used to create Liquidity Pools out of pool shares, since trustline authorization fails.

#### SetTrustLineFlagsOp and AllowTrustOp
The functionality of `SetTrustlineFlagsOp` and `AllowTrustOp` are kept identical to the implementation discussed in CAP-0038.

#### PaymentOp
The `PaymentOp` operation is modified to support the transfer of liquidity pool shares. The updates to the XDR scheme for `ASSET_TYPE_POOL_SHARE` ledger entries to be standard assets allows the majority of `PaymentOp` code to function as normal. The assumption is made that `ChangeTrustOp` correctly manages the state of a pool share trustline, and `SetTrustLineFlagsOp` / `AllowTrustOp` correctly remove pool share trustlines during an asset authorization revocation. 

Based on these assumptions, consider User 1 who has trustlines to Asset A, Asset B, and Pool AB. They hold N shares of Pool AB.

User 1 transfers N Pool AB shares to User 2:
1. User 2 has no trustlines to Asset A, Asset B, or Pool AB
    - The transfer fails due to `PAYMENT_NO_TRUST`
2. User 2 only has a trustline to Pool AB
    - The transfer fails due to `PAYMENT_NO_TRUST`
3. User 2 has trustline to Pool AB, and only Asset A or Asset B, but not the other
    - The transfer fails due to `PAYMENT_NO_TRUST`
4. User 2 has all three trustlines to Asset A, Asset B, and Pool AB, but gets authorization revoked for Asset A immediately before the payment
    - The unauthorization causes the trustline to Pool AB to be deleted
    - The transfer fails due to `PAYMENT_NO_TRUST`
5. User 2 has all three trustlines
    - If User 2's pool trustline limit >= N
        - The transfer succeeds
        - User 1 has N shares deducted from their Pool AB trustline balance
        - User 2 has N shares added to their Pool AB trustline balance
    - If User 2's pool trustline limit < N
        - The transfer fails due to `PAYMENT_LINE_FULL`
6. User 2 has all three trustlines with INT_MAX limits, but User 1 gets authorization revoked for Asset A immediately before the payment
    - User 1's pool shares are redeemed and the underlying balances are placed in claimable balances
    - User 1's Pool AB trustline is deleted
    - The transfer fails due to `PAYMENT_SRC_NO_TRUST`

#### CreateClaimableBalanceOp and ClaimClaimableBalanceOp
`CreateClaimableBalanceOp` and `ClaimClaimableBalanceOp` is modified to allow a claimable balance to hold liquidity pool shares. A `CreateClaimableBalanceOp` is valid for pool shares if the source account has a trustline with enough balance to create a claimable balance with the requested amount, and all current non-asset specific criteria are met. A `ClaimClaimableBalanceOp` is valid for a claimable balance with pool shares if the claimant has a trustline with a large enough limit to support the claimable balance amount, and all current non-asset specific criteria are met.

#### ManageBuyOfferOp and ManageSellOfferOp
The `MANAGE_BUY_OFFER_MALFORMED` or `MANAGE_SELL_OFFER_MALFORMED` will be returned if the selling or buying asset of the operation is of `ASSET_TYPE_POOL_SHARE`.

## Design Rationale

### Pool Share Authorization
Authorization of pool shares is delegated to the assets that compose the pool. Currently, the pool share trustline strictly enforces the owner to have underlying asset trustlines. They can only be created if a user is authorized and has a trustline to both underlying assets, and is removed if authorization is ever revoked from the issuer for an underlying asset. Thus, each action taken against a pool share can trust the pool share trustline. This has the added benefit of being consistent with traditional payments and with the trustline mechanisms introduced in CAP-0038.

### Jump Cannon Interoperability
The expected use cases of transferable 

### Disable Liquidity Pools Nesting
Liquidity pools of liquidity pool shares are not a common presence across the current crypto ecosystem. Although possible on EVM platforms, they don't fulfill enough use cases to warrant the additional work to support them.

Nested liquidity pools also introduce technical challenges for supporting asset authorization revocations. Instead of being able to find all liquidity pools that contain an asset that was revoked, Core now needs to look for any liquidity pool shares in liquidity pools that, at some point down the tree, contain the asset that was revoked.

### Disable Creating Offers of Liquidity Pool Shares
Selling liquidity pool shares over the DEX is not supported for consistency and lack of use cases. Since liquidity pool shares are not able to be used to create new liquidity pools, they are already stopped from 1 major exchange method on Stellar. They should be either included in all forms of on chain exchange, or none, for consistencies sake.

The use case for creating an offer of a liquidity pool share is relatively weak as well. Liquidity pool shares can be deposited and redeemed from a liquidity pool at a defined rate. Thus, allowing offers to be created for liquidity pool shares will only result in arbitrage opportunities between offer prices and pool redemptions, without actually increasing a users liquidity of their pool shares.

## Protocol Upgrade Transition

### Backwards Incompatibilities
This proposal is completely backwards compatible.

### Resource Utilization
SDK developers will need to add support for the `ASSET_TYPE_POOL_SHARE` Asset type and usage within relevant operations.

## Test Cases
TODO

## Implementation
TODO