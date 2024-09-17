# Aave V3

>👷 This document is a work in progress. Please feel free to contribute to it by opening a pull request (ensure to follow our [CONTRIBUTION guidelines](CONTRIBUTING.md)).

The [Pool contract](https://github.com/aave/aave-v3-core/blob/master/contracts/protocol/pool/Pool.sol) is the main entry point for interacting with the Aave V3 protocol. It implements the core lending and borrowing functionality, including:
* [supply](#supply)
* borrow (TODO)
* repay (TODO)
* withdraw (TODO)
* liquidationCall (TODO)
* flashLoan (TODO)

## Links
* Github: https://github.com/aave/aave-v3-core
* Docs: https://docs.aave.com/developers


## `supply`

### Execution path

```
Pool.sol (supply)
  -> SupplyLogic.sol (executeSupply)
    -> ValidationLogic.sol (validateSupply)
    -> ERC20.sol of supplied asset (safeTransferFrom)
    -> AToken.sol (mint)
      -> ScaledBalanceTokenBase.sol (_mintScaled)
    -> ValidationLogic.sol (validateAutomaticUseAsCollateral) if the asset is supplied for the first time
```


### Execution steps

* Get reserves data.
* Confirm that the `amount` parameter provided as part of the `supply` function is not 0.
* Confirm that the reserve is active (meaning that the asset to be supplied is supported by the protocol).
* Confirm that the reserve is not paused.
* Confirm that the reserve is not frozen.
* Confirm that the supply cap is not exceeded.
* Transfer the supplied asset from the user to the Pool contract.
* Mint the corresponding amount of aTokens to the user.
* If the asset is supplied for the first time, confirm that it can be enabled as collateral (isolated assets are not enabled as collateral automatically).
* Update the reserve data.

>**Frozen reserve:**
> * Does not allow new supply, borrow, or rate switch (variable/stable) operations.
> * Does allow repay, withdraw, liquidations, and interest accrual (stable rate rebalances).

>**Paused reserve:**
> * Does not allow any protocol interaction for the asset being supplied, including supply, borrow, repay, withdraw, rate switch (variable/stable), liquidation, and aToken transfer.


### Revert conditions

`ValidationLogic.validateSupply`:
* If the amount is 0 (Errors.INVALID_AMOUNT)
* If the reserve is not active (Errors.RESERVE_INACTIVE)
* If the reserve is paused (Errors.RESERVE_PAUSED)
* If the reserve is frozen (Errors.RESERVE_FROZEN)
* If the supply cap is exceeded (Errors.SUPPLY_CAP_EXCEEDED)

> **Note:** All error codes and messages are defined in [Errors.sol](https://github.com/aave/aave-v3-core/blob/master/contracts/protocol/libraries/helpers/Errors.sol)

`IERC20(params.asset).safeTransferFrom`:
* If the user has not approved the Pool contract to spend their tokens
* If the user doesn't have enough balance of the asset to be supplied


### Emitted events

#### `Supply`

Triggered in: `SupplyLogic.sol (executeSupply)`

```solidity
event Supply(
    address indexed reserve,
    address user,
    address indexed onBehalfOf,
    uint256 amount,
    uint16 indexed referralCode
);
```


#### `Transfer`

Triggered in: `ERC20.sol of supplied asset (safeTransferFrom)`

```solidity
event Transfer(
    address indexed from,
    address indexed to,
    uint256 value
);
```


#### `Mint`

Triggered in: `ScaledBalanceTokenBase.sol (_mintScaled)`

```solidity
event Mint(
    address indexed caller,
    address indexed onBehalfOf,
    uint256 amount,
    uint256 index
);
```


#### `Transfer`

Triggered in: `ScaledBalanceTokenBase.sol (_mintScaled)`

```solidity
event Transfer(
    address indexed from,
    address indexed to,
    uint256 value
);
```


#### `ReserveUsedAsCollateralEnabled` (only on first supply)

Triggered in: `ValidationLogic.sol (validateUseAsCollateral)`

```solidity
event ReserveUsedAsCollateralEnabled(
    address indexed reserve,
    address indexed user
);
```


#### `ReserveDataUpdated`

Triggered in: `ReserveLogic.sol (updateInterestRates)`

```solidity
event ReserveDataUpdated(
    address indexed reserve,
    uint256 liquidityRate,
    uint256 stableBorrowRate,
    uint256 variableBorrowRate,
    uint256 liquidityIndex,
    uint256 variableBorrowIndex
);
```	


[Example transaction](https://polygonscan.com/tx/0xc968c68100094e7c5e579ca0c83afdc737aa021717cc456529e1170ffc77acbe#eventlog)