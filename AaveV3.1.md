# Aave V3.1

>👷 This document is a work in progress. Please feel free to contribute to it by opening a pull request (ensure to follow our [CONTRIBUTION guidelines](CONTRIBUTING.md)).

The [Pool contract](https://github.com/aave-dao/aave-v3-origin/blob/main/src/core/contracts/protocol/pool/Pool.sol) is the main entry point for interacting with the Aave V3.1 protocol. It implements the core lending and borrowing functionality, including:
* [supply](#supply)
* borrow (TODO)
* repay (TODO)
* withdraw (TODO)
* liquidationCall (TODO)
* flashLoan (TODO)

## Links
* Github: https://github.com/aave-dao/aave-v3-origin
* Docs: https://docs.aave.com/developers


## `supply`

### Function definition

```
/**
* @notice Supplies an `amount` of underlying asset into the reserve, receiving in return overlying aTokens.
* - E.g. User supplies 100 USDC and gets in return 100 aUSDC
* @param asset The address of the underlying asset to supply
* @param amount The amount to be supplied
* @param onBehalfOf The address that will receive the aTokens, same as msg.sender if the user
* wants to receive them on his own wallet, or a different address if the beneficiary of aTokens
* is a different wallet
* @param referralCode Code used to register the integrator originating the operation, for potential rewards.
* 0 if the action is executed directly by the user, without any middle-man
*/
function supply(
    address asset,
    uint256 amount,
    address onBehalfOf,
    uint16 referralCode
) external;
```
[Source](https://github.com/aave-dao/aave-v3-origin/blob/3aad8ca184159732e4b3d8c82cd56a8707a106a2/src/core/contracts/interfaces/IPool.sol#L248)

### Execution path

```
Pool.sol (supply)
    -> SupplyLogic.sol (executeSupply)
        -> ReserveLogic.sol (updateState)
        -> ValidationLogic.sol (validateSupply)
        -> ReserveLogic.sol (updateInterestRatesAndVirtualBalance)
        -> ERC20.sol of supplied asset (transferFrom called through the Gnosis wrapper contract GPv2SafeERC20.sol's safeTransferFrom function)
        -> AToken.sol (mint)
            -> ScaledBalanceTokenBase.sol (_mintScaled)
        If the asset is supplied for the first time:
        -> ValidationLogic.sol (validateAutomaticUseAsCollateral)
            If validateAutomaticUseAsCollateral function returns true:
            -> UserConfiguration.sol (setUsingAsCollateral)
```


### Execution steps

* Get reserves data associated with the supplied asset (`params.asset`).
* Update the reserve data, including liquidity cumulative index and the variable borrow index, and accrue interest to treasury. Skipped if it was already updated within the same block (e.g., if the `supply` function was called twice within the same block).
* Confirm that the `amount` parameter provided as part of the `supply` function is not 0.
* Confirm that the reserve is active (meaning that the asset to be supplied is supported by the protocol).
* Confirm that the reserve is not paused.
* Confirm that the reserve is not frozen.
* Confirm that the aToken recipient (`onBehalfOf` argument) is not the reserve's aToken address.
* Confirm that the reserve's supply cap is not exceeded.
* Transfer the supplied asset from the user to the Pool contract.
* Mint the corresponding amount of aTokens to the user.
* If the asset is supplied for the first time, confirm that it can be automatically enabled as collateral (isolated assets are not enabled as collateral automatically). If yes, reflect it in the user's bitmap configuration accordingly.

>**Frozen reserve:**
> * Does not allow new supply, borrow, or rate switch (variable/stable) operations.
> * Does allow repay, withdraw, liquidations, and interest accrual (stable rate rebalances).

>**Paused reserve:**
> * Does not allow any protocol interaction for the asset being supplied, including supply, borrow, repay, withdraw, rate switch (variable/stable), liquidation, and aToken transfer.


### Revert conditions

`ValidationLogic.validateSupply`:
* If the amount is 0 (reverts with Errors.INVALID_AMOUNT -> error code 26)
* If the reserve is not active (reverts with Errors.RESERVE_INACTIVE -> error code 27)
* If the reserve is paused (reverts with Errors.RESERVE_PAUSED -> error code 29)
* If the reserve is frozen (reverts with Errors.RESERVE_FROZEN -> error code 28)
* If the aToken recipient (`onBehalfOf` argument) equals the reserve's aToken address (reverts with Errors.SUPPLY_TO_ATOKEN -> error code 94)
* If the reserve's supply cap is exceeded (reverts with Errors.SUPPLY_CAP_EXCEEDED -> error code 51)


> **Note:** All error codes and messages are defined in [Errors.sol](https://github.com/aave-dao/aave-v3-origin/blob/main/src/core/contracts/protocol/libraries/helpers/Errors.sol)

`IERC20(params.asset).safeTransferFrom`:
* If the user has not approved the Pool contract to spend their tokens.
* If the user doesn't have enough balance of the asset to be supplied.


### Emitted events

The following events are emitted, listed in the order of occurrence:

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


#### `ReserveUsedAsCollateralEnabled`

Trigger condition: only on first supply and if the supplied asset can be enabled as collateral (i.e. it's not an isolated asset).

Triggered in: `SupplyLogic.sol (executeSupply)`

```solidity
event ReserveUsedAsCollateralEnabled(
    address indexed reserve,
    address indexed user
);
```


#### `ReserveDataUpdated`

Triggered in: `ReserveLogic.sol (updateInterestRatesAndVirtualBalance)`

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

### Network-specific considerations

None



## withdraw

### Function definition

```
/**
 * @notice Withdraws an `amount` of underlying asset from the reserve, burning the equivalent aTokens owned
 * E.g. User has 100 aUSDC, calls withdraw() and receives 100 USDC, burning the 100 aUSDC
 * @param asset The address of the underlying asset to withdraw
 * @param amount The underlying amount to be withdrawn
 *   - Send the value type(uint256).max in order to withdraw the whole aToken balance
 * @param to The address that will receive the underlying, same as msg.sender if the user
 *   wants to receive it on his own wallet, or a different address if the beneficiary is a
 *   different wallet
 * @return The final amount withdrawn
 */
function withdraw(
    address asset,
    uint256 amount,
    address to
) external returns (uint256);
```
[Source](https://github.com/aave-dao/aave-v3-origin/blob/3aad8ca184159732e4b3d8c82cd56a8707a106a2/src/core/contracts/interfaces/IPool.sol#L287)

### Execution path

```
Pool.sol (withdraw)
    -> SupplyLogic.sol (executeWithdraw)
        -> ReserveLogic.sol (updateState)
        -> ValidationLogic.sol (validateWithdraw)
        -> ReserveLogic.sol (updateInterestRatesAndVirtualBalance)
        -> AToken.sol (burn)
            -> ScaledBalanceTokenBase.sol (_burnScaled)
        If the user withdraws all their balance and the asset was used as collateral:
        -> UserConfiguration.sol (setUsingAsCollateral)
        If the user has any borrows:
        -> ValidationLogic.sol (validateHFAndLtv)
```

### Execution steps

* Get reserves data associated with the asset to be withdrawn (params.asset).
* Update the reserve data, including liquidity cumulative index and the variable borrow index, and accrue interest to treasury. Skipped if it was already updated within the same block.
* If the amount parameter is set to type(uint256).max, set amountToWithdraw to the user's entire aToken balance.
* Confirm that the amountToWithdraw is not 0.
* Confirm that the reserve is active.
* Confirm that the reserve is not paused.
* Confirm that the user has enough balance to withdraw the requested amount.
* Update the reserve's interest rates based on the new utilization after withdrawal.
* Burn the corresponding amount of aTokens from the user.
* If the user withdraws their entire balance and the asset was used as collateral, update the user's configuration to reflect that the asset is no longer used as collateral.
* If the user has any outstanding borrows, validate that the health factor remains above 1 after the withdrawal.
* Transfer the underlying asset from the aToken contract to the recipient (to address).

### Revert conditions

`ValidationLogic.validateWithdraw`:
* If the amount is 0 (reverts with Errors.INVALID_AMOUNT -> error code 26)
* If the user doesn't have enough balance (reverts with Errors.NOT_ENOUGH_AVAILABLE_USER_BALANCE -> error code 30)
* If the reserve is not active (reverts with Errors.RESERVE_INACTIVE -> error code 27)
* If the reserve is paused (reverts with Errors.RESERVE_PAUSED -> error code 29)

`SupplyLogic.executeWithdraw`:
* If the recipient (to address) is the same as the aToken address (reverts with Errors.WITHDRAW_TO_ATOKEN -> error code 93)

`ValidationLogic.validateHFAndLtv`:
* If the withdrawal would cause the user's health factor to drop below 1 (reverts with Errors.HEALTH_FACTOR_LOWER_THAN_LIQUIDATION_THRESHOLD -> error code 42)

### Emitted events

The following events are emitted, listed in the order of occurrence:

#### `ReserveUsedAsCollateralDisabled`

Trigger condition: only if the user withdraws their entire balance and the asset was used as collateral.

Triggered in: `SupplyLogic.sol (executeWithdraw)`

```solidity
event ReserveUsedAsCollateralDisabled(
    address indexed reserve,
    address indexed user
);
```

#### `Burn`

Triggered in: `ScaledBalanceTokenBase.sol (_burnScaled)`

```solidity
event Burn(
    address indexed caller,
    address indexed onBehalfOf,
    uint256 amount,
    uint256 index
);
```

#### `Transfer`

Triggered in: `ScaledBalanceTokenBase.sol (_burnScaled)`

```solidity
event Transfer(
    address indexed from,
    address indexed to,
    uint256 value
);
```

#### `Withdraw`

Triggered in: `SupplyLogic.sol (executeWithdraw)`

```solidity
event Withdraw(
    address indexed reserve,
    address indexed user,
    address indexed to,
    uint256 amount,
    uint256 index
);
```

#### `ReserveDataUpdated`

Triggered in: `ReserveLogic.sol (updateInterestRatesAndVirtualBalance)`

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

#### `Example transaction`

[Example transaction](https://polygonscan.com/tx/0x746d50fcfc5d80549f09dd0267eb1ad3076f28c06abb78c4ac7bebf03dae6918#eventlog)

#### `Network-specific considerations`

None