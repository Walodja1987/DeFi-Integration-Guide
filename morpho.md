# Morpho Blue

Morpho Blue's narrow focus enables trustless and efficient lending and borrowing of assets. It provides users with higher collateralization factors, improved interest rates, significantly reduced gas costs, and permissionless market creation.

## Overview

Morpho is a decentralized lending and borrowing platform with several distinguishing features:

1. Complete immutability
2. Permissionless design, allowing anyone to create a market with any collateral and loan assets
3. Multiple Interest Rate Model (IRM) options available
4. Customizable oracles for efficient liquidation management
5. Low gas costs due to the system's simplicity
6. Minimized governance: Morpho governance can only enable IRMs and LTVs, set fees and fee recipients, and has no control over funds or oracles

These features collectively contribute to a more flexible, efficient, and user-centric lending and borrowing experience in the decentralized finance ecosystem.

## Main Entry Contract
[Morpho.sol](https://github.com/morpho-org/morpho-blue/blob/main/src/Morpho.sol)

## Core Functions
1. [createMarket](https://github.com/morpho-org/morpho-blue/blob/fcb190b0e0e1355defe56b19781ac573986c3f74/src/Morpho.sol#L150)
2. [supply](https://github.com/morpho-org/morpho-blue/blob/fcb190b0e0e1355defe56b19781ac573986c3f74/src/Morpho.sol#L169)
3. [withdraw](https://github.com/morpho-org/morpho-blue/blob/fcb190b0e0e1355defe56b19781ac573986c3f74/src/Morpho.sol#L200)
4. [borrow](https://github.com/morpho-org/morpho-blue/blob/fcb190b0e0e1355defe56b19781ac573986c3f74/src/Morpho.sol#L235)
5. [repay](https://github.com/morpho-org/morpho-blue/blob/fcb190b0e0e1355defe56b19781ac573986c3f74/src/Morpho.sol#L235)
6. [supplyCollateral](https://github.com/morpho-org/morpho-blue/blob/fcb190b0e0e1355defe56b19781ac573986c3f74/src/Morpho.sol#L303)
7. [withdrawCollateral](https://github.com/morpho-org/morpho-blue/blob/fcb190b0e0e1355defe56b19781ac573986c3f74/src/Morpho.sol#L323)
8. [liquidate](https://github.com/morpho-org/morpho-blue/blob/fcb190b0e0e1355defe56b19781ac573986c3f74/src/Morpho.sol#L347)
9. [flashloan](https://github.com/morpho-org/morpho-blue/blob/fcb190b0e0e1355defe56b19781ac573986c3f74/src/Morpho.sol#L422)
10. [setAuthorization](https://github.com/morpho-org/morpho-blue/blob/fcb190b0e0e1355defe56b19781ac573986c3f74/src/Morpho.sol#L437C14-L437C30)
11. [setAuthorizationWithSig](https://github.com/morpho-org/morpho-blue/blob/fcb190b0e0e1355defe56b19781ac573986c3f74/src/Morpho.sol#L446)
12. [accrueInterest](https://github.com/morpho-org/morpho-blue/blob/fcb190b0e0e1355defe56b19781ac573986c3f74/src/Morpho.sol#L474)


## Solidity Version
```0.8.19```

## Links
- [Codebase](https://github.com/morpho-org/morpho-blue)
- [Developer Docs](https://docs.morpho.org/morpho/overview)
- [Contract Address](https://docs.morpho.org/morpho/addresses/)
- [ABI](https://docs.morpho.org/morpho/addresses/)

## Key Terminology
Key terminology in Morpho Blue:

Market
* A lending pool defined by a set of parameters (MarketParams) including loan token, collateral token, oracle, interest rate model (IRM), and loan-to-value ratio (LLTV).
* Each market has a unique identifier (Id).

Position
* Represents a user's assets in a specific market, including supply shares, borrow shares, and collateral amount.

Supply Shares
* Represents a user's share of the total supplied assets in a market.
* Used to calculate the amount of assets a user can withdraw.

Borrow Shares
* Represents a user's share of the total borrowed assets in a market.
* Used to calculate the amount of debt a user owes.

Collateral
* Assets deposited by a user to secure their borrowed position.
* Cannot be borrowed by others and doesn't earn interest.

Loan-to-Value Ratio (LLTV)
* The maximum amount a user can borrow relative to their collateral value.

Interest Rate Model (IRM)
* Determines the borrow rate for a market.
* Can be customized for each market.

Oracle
* Provides price data for collateral assets.

Authorization
* Mechanism allowing users to delegate control of their positions to other addresses.

Health Factor
* A measure of the safety of a user's position, based on the ratio of collateral value to borrowed value.

Liquidation
* Process triggered when a borrower's position becomes unhealthy.
* Allows liquidators to repay part of the debt in exchange for collateral.


## Difference in (supply, supplyCollteral) and (withdraw and withdrawCollateral)
The naming of these functions is quite similar, so let's first discuss the differences between these four similarly named functions.


1. Supply vs. Supply Collateral:

   Supply:

   - Adds assets to the lending pool

   - Increases the user's supply shares

   - Allows the user to earn interest

   - These assets can be borrowed by other users

   Supply Collateral:

   - Adds assets as collateral for borrowing

   - Increases the user's collateral balance

   - Does not earn interest

   - Cannot be borrowed by others

   - Allows the user to borrow other assets against this collateral

2. Withdraw vs. Withdraw Collateral:

   Withdraw:

   - Removes supplied assets from the lending pool

   - Decreases the user's supply shares

   - User stops earning interest on withdrawn amount

   - Requires sufficient liquidity in the pool

   Withdraw Collateral:

   - Removes collateral assets

   - Decreases the user's collateral balance

   - Doesn't affect interest earnings

   - Requires the user's position to remain healthy after withdrawal



## ```createMarket```
```solidity
    /// @notice Creates the market `marketParams`.
    /// @dev Here is the list of assumptions on the market's dependencies (tokens, IRM and oracle) that guarantees
    /// Morpho behaves as expected:
    /// - The token should be ERC-20 compliant, except that it can omit return values on `transfer` and `transferFrom`.
    /// - The token balance of Morpho should only decrease on `transfer` and `transferFrom`. In particular, tokens with
    /// burn functions are not supported.
    /// - The token should not re-enter Morpho on `transfer` nor `transferFrom`.
    /// - The token balance of the sender (resp. receiver) should decrease (resp. increase) by exactly the given amount
    /// on `transfer` and `transferFrom`. In particular, tokens with fees on transfer are not supported.
    /// - The IRM should not re-enter Morpho.
    /// - The oracle should return a price with the correct scaling.
    /// @dev Here is a list of properties on the market's dependencies that could break Morpho's liveness properties
    /// (funds could get stuck):
    /// - The token can revert on `transfer` and `transferFrom` for a reason other than an approval or balance issue.
    /// - A very high amount of assets (~1e35) supplied or borrowed can make the computation of `toSharesUp` and
    /// `toSharesDown` overflow.
    /// - The IRM can revert on `borrowRate`.
    /// - A very high borrow rate returned by the IRM can make the computation of `interest` in `_accrueInterest`
    /// overflow.
    /// - The oracle can revert on `price`. Note that this can be used to prevent `borrow`, `withdrawCollateral` and
    /// `liquidate` from being used under certain market conditions.
    /// - A very high price returned by the oracle can make the computation of `maxBorrow` in `_isHealthy` overflow, or
    /// the computation of `assetsRepaid` in `liquidate` overflow.
    /// @dev The borrow share price of a market with less than 1e4 assets borrowed can be decreased by manipulations, to
    /// the point where `totalBorrowShares` is very large and borrowing overflows.
    function createMarket(MarketParams memory marketParams) external;

    struct MarketParams {
    address loanToken;
    address collateralToken;
    address oracle;
    address irm;
    uint256 lltv;
}
```

[Source][https://github.com/morpho-org/morpho-blue/blob/main/src/Morpho.sol]

## Execution Path

```
Morpho.sol (createMarket)
   If IRM is stateful initiate it
      -> IIRM.sol (borrowRate())
```
