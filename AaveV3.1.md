# Aave V3.1

>👷 This document is a work in progress. Please feel free to contribute to it by opening a pull request (ensure to follow our [CONTRIBUTION guidelines](CONTRIBUTING.md)).

## Overview

Aave V3.1 is a decentralized lending protocol that allows users to:

1. Supply digital assets to earn interest
2. Post these assets as collateral
3. Borrow other assets against their collateral

The protocol automatically manages interest rates based on supply and demand. It includes safety features like liquidations to ensure the system remains solvent.

## Main entry contract

* [Pool contract](https://github.com/aave-dao/aave-v3-origin/blob/main/src/core/contracts/protocol/pool/Pool.sol)

## Core functions

* [supply](#supply)
* [withdraw](#withdraw)
* borrow (TODO)
* repay (TODO)
* liquidationCall (TODO)
* flashLoan (TODO)

## Solidity version
`^0.8.10`

## Links
* Codebase (V3.1): https://github.com/aave-dao/aave-v3-origin
* Developer docs: https://docs.aave.com/developers
* Contract addresses: https://docs.aave.com/developers/deployed-contracts/deployed-contracts
* ABIs: https://docs.aave.com/developers/deployed-contracts/deployed-contracts

## Key terminology

### Frozen reserve
- Does not allow new supply, borrow, or rate switch (variable/stable) operations.
- Does allow repay, withdraw, liquidations, and interest accrual (stable rate rebalances).

### Paused reserve
- Does not allow any protocol interaction for the asset being supplied, including supply, borrow, repay, withdraw, rate switch (variable/stable), liquidation, and aToken transfer.

### Inactive reserve
- The asset to be supplied is not supported by the protocol.

### Enabled as collateral
- Enabling a supplied asset as collateral means that the asset can be used as collateral for borrowing.

### Isolated asset
An isolated asset in the Aave protocol refers to an asset that has been configured with special restrictions to limit its potential risk to the overall system. The decision to make an asset isolated, as well as setting its specific parameters (like debt ceiling), is controlled by Aave governance. Here are the key characteristics of an isolated asset:

- **Limited borrowing**: Users can only borrow stablecoins that have been specifically configured by Aave governance to be borrowable when using an isolated asset as collateral.
- **Debt ceiling**: There is a maximum amount (debt ceiling) that can be borrowed against isolated collateral, expressed in USD.
- **Single collateral usage**: When a user supplies an isolated asset as collateral, they cannot use any other assets as collateral simultaneously. The isolated asset must be the only collateral for that user's position.
- **Restricted as collateral**: Other non-isolated assets cannot be used as collateral if a user has an isolated asset enabled as collateral.
- **Isolation mode**: When a user enables an isolated asset as collateral, they enter "isolation mode", which applies these special restrictions to their account.


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

#### `Transfer`

Triggered in: `ERC20.sol of supplied asset (safeTransferFrom)`

```solidity
event Transfer(
    address indexed from,
    address indexed to,
    uint256 value
);
```

#### `Approval`

Triggered in: `ERC20.sol of supplied asset (safeTransferFrom)`

Note that for some tokens such as USDT on Ethereum, the `Approval` event may not be emitted.

```solidity
event Approval(
    address indexed owner,
    address indexed spender,
    uint256 value
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

#### `Mint`

Triggered in: `ScaledBalanceTokenBase.sol (_mintScaled)`

```solidity
event Mint(
    address indexed caller,
    address indexed onBehalfOf,
    uint256 value,
    uint256 balanceIncrease,
    uint256 index
);
```

#### `ReserveUsedAsCollateralEnabled`

Trigger condition: only on first supply and if the supplied asset can be enabled as collateral.

Triggered in: `SupplyLogic.sol (executeSupply)`

```solidity
event ReserveUsedAsCollateralEnabled(
    address indexed reserve,
    address indexed user
);
```

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

* [Example transaction](https://polygonscan.com/tx/0xc968c68100094e7c5e579ca0c83afdc737aa021717cc456529e1170ffc77acbe#eventlog) including `ReserveUsedAsCollateralEnabled` event.

### Network-specific considerations

None

### Security considerations

None

## `withdraw`

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
        -> UserConfiguration.sol (isUsingAsCollateral)
        If isUsingAsCollateral is true and user withdraws their entire balance:
        -> UserConfiguration.sol (setUsingAsCollateral)
        -> AToken.sol (burn)
            -> ScaledBalanceTokenBase.sol (_burnScaled)
        If the user has any borrows:
        -> ValidationLogic.sol (validateHFAndLtv)
            -> GenericLogic.sol (calculateUserAccountData)
```

### Execution steps

* Get reserves data associated with the asset to be withdrawn (`params.asset`).
* Confirm that the recipient (`to` argument) is not the same as the aToken address.
* Update the reserve data, including liquidity cumulative index and the variable borrow index, and accrue interest to treasury. Skipped if it was already updated within the same block.
* If the amount parameter is set to `type(uint256).max`, set amount to withdraw to the user's entire aToken balance.
* Confirm that the amount to withdraw is not 0.
* Confirm that the user has enough aToken balance to withdraw the requested amount.
* Confirm that the reserve is active.
* Confirm that the reserve is not paused.
* Update the reserve's interest rates based on the new utilization after withdrawal.
* If the user withdraws their entire balance and the asset was used as collateral, update the user's configuration to reflect that the asset is no longer used as collateral.
* Burn the corresponding amount of aTokens from the `msg.sender` and transfer the underlying asset from the aToken contract to the recipient (`to` argument), if the recipient does not equal the aToken address.
* If the user has any outstanding borrows, validate that the health factor remains above 1 after the withdrawal.


### Revert conditions

`SupplyLogic.executeWithdraw`:
* If the recipient (`to` argument) is the same as the aToken address (reverts with Errors.WITHDRAW_TO_ATOKEN -> error code 93)
* If `asset` is the zero address, the function fails silently when trying to call `scaledBalanceOf` on a non-existent aToken contract.

`ValidationLogic.validateWithdraw`:
* If the amount is 0 (reverts with Errors.INVALID_AMOUNT -> error code 26)
* If the user doesn't have enough balance (reverts with Errors.NOT_ENOUGH_AVAILABLE_USER_BALANCE -> error code 32)
* If the reserve is not active (reverts with Errors.RESERVE_INACTIVE -> error code 27)
* If the reserve is paused (reverts with Errors.RESERVE_PAUSED -> error code 29)

`ValidationLogic.validateHFAndLtv`:
* If the withdrawal would cause the user's health factor to drop below 1 (reverts with Errors.HEALTH_FACTOR_LOWER_THAN_LIQUIDATION_THRESHOLD -> error code 35)


### Emitted events

The following events are emitted, listed in the order of occurrence:

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

#### `ReserveUsedAsCollateralDisabled`

Trigger condition: only if the user withdraws their entire balance and the asset was used as collateral.

Triggered in: `SupplyLogic.sol (executeWithdraw)`

```solidity
event ReserveUsedAsCollateralDisabled(
    address indexed reserve,
    address indexed user
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

#### `Mint`

Trigger condition: In some instances, a burn transaction will emit a `Mint` event if the amount to burn is less than the interest that the user accrued (see comment in `ScaledBalanceTokenBase.sol (_burnScaled)`).

Triggered in: `ScaledBalanceTokenBase.sol (_burnScaled)`

```solidity
event Mint(
    address indexed caller,
    address indexed onBehalfOf,
    uint256 value,
    uint256 balanceIncrease,
    uint256 index
);
```

#### `Burn`

Triggered in: `ScaledBalanceTokenBase.sol (_burnScaled)`

```solidity
event Burn(
    address indexed from,
    address indexed target,
    uint256 value,
    uint256 balanceIncrease,
    uint256 index
);
```

#### `Withdraw`

Triggered in: `SupplyLogic.sol (executeWithdraw)`

```solidity
event Withdraw(
    address indexed reserve,
    address indexed user,
    address indexed to,
    uint256 amount
);
```

* [Example transaction](https://polygonscan.com/tx/0x746d50fcfc5d80549f09dd0267eb1ad3076f28c06abb78c4ac7bebf03dae6918#eventlog) excluding `ReserveUsedAsCollateralDisabled` and `Mint` events.
* [Example transaction](https://polygonscan.com/tx/0x080ed836e06c75a375a9bf64d521a6572e423657a414e58ec1ac5694fab73eb8#eventlog) including `ReserveUsedAsCollateralDisabled` event, excluding `Mint` event.

### Network-specific considerations

None

### Security considerations

None