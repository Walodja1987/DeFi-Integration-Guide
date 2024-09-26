# [--Protocol Name--]

## Overview

<!-- Add a brief overview of the protocol -->

<!-- Example:
Aave V3.1 is a decentralized lending protocol that allows users to:

1. Supply digital assets to earn interest
2. Post these assets as collateral
3. Borrow other assets against their collateral

The protocol automatically manages interest rates based on supply and demand. It includes safety features like liquidations to ensure the system remains solvent.
-->

## Main entry contract

<!-- Add the main entry contract here -->

<!-- Example:
* [Pool contract](https://github.com/aave-dao/aave-v3-origin/blob/main/src/core/contracts/protocol/pool/Pool.sol)
-->

## Core functions

<!-- Add the core functions here -->

<!-- Example:
* [function1](#function1)
* [function2](#function2)
-->

## Solidity version

<!-- Add the Solidity version used in the protocol -->

<!-- Example:
`^0.8.10`
-->

## Links
* Contracts: [--Link to the Github repo--]
* Developer docs: [--Link to the docs--]
* Contract addresses: [--Link to the addresses--]
* ABIs: [--Link to the ABIs--]
* Audit reports: [--Link to the audit reports--]

## Key terminology

<!-- Add the key terminology specific to the protocol to help developers understand terms used in the following sections -->

<!-- Example:
### Frozen reserve
- Does not allow new supply, borrow, or rate switch (variable/stable) operations.
- Does allow repay, withdraw, liquidations, and interest accrual (stable rate rebalances).
-->

## `function1`

### Function definition

<!-- Add the function definition here. Could be a simple copy paste from the smart contract code -->

<!-- Example:
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
-->

### Execution path

<!-- Add the execution path here to highlight which contracts are entered through which functions -->

<!-- Example:
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
-->

### Execution steps

<!-- Add the execution steps here to highlight the order of operations. Also highlight the return values if any. -->

<!-- Example:
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
-->

### Revert conditions

<!-- Add the revert conditions here. Group the revert conditions by the contract and function that contains them. Also highlight the error codes that are returned if applicable. Provide links if deemed necessary. -->

<!-- Example:
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
-->

### Emitted events

The following events are emitted, listed in the order of occurrence:

<!-- Add the emitted events here. Indicate in which contract and function the event is emitted. Highlight any conditions under which the event is triggered. Include an example transaction link to showcase the events emitted. -->

<!-- Example:
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

* [Example transaction](https://polygonscan.com/tx/0xc968c68100094e7c5e579ca0c83afdc737aa021717cc456529e1170ffc77acbe#eventlog) including `ReserveUsedAsCollateralEnabled` event.
-->

### Network-specific considerations

<!-- Add the network-specific considerations here. Examples may include some network-specific behavior such as approving USDT on Ethereum requires to reset the allowance to 0 first. Link any libraries that may help to handle these nuances. -->

### Security considerations

<!-- Add security considerations here. Provide links to relevant audit findings to highlight common mistakes made by developers. -->


## `function2`

### Function definition

### Execution path

### Execution steps

### Revert conditions

### Emitted events

### Network-specific considerations

### Security considerations
