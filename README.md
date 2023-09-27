# CygnusDAO Security Review by Borosorus



## Disclaimer

A time-boxed audit doesn't guarantee the absence of security issues.

## CygnusDAO

At the core, CygnusDAO is a lending protocol allowing users to collateralize one token to borrow another one.
The biggest differences of this protocol with most well-known lending protocol lays in two facts:
    1. It provides isolated loan pools, which consists of a pair of a collateral and a borrowable.
    2. The entirety of the collateralized and idling underlying tokens can still earn yield instead of being locked.

These attributes allows the user of the protocol to open leveraged position or to trade against tokens while receiving rewards.

CygnusDAO's architecture is divided in two subfolders: core and periphery.
Core contains the protocol's critical lending and borrowing logic that are to be used via rather low level calls.
Periphery contains various entry points to ease the usage of the protocol.

### Core

The main components of core are the `CygnusBorrow` and the `CygnusCollateral` smart contracts. They are always created in pair and linked to each other via a `twinstar` variable that exists on each one of them. A pair of collateral and borrowable is called a `shuttle`.
They inherit from both `ERC20` and `CygnusTerminal` which allows to set a principal underlying at creation.
`CygnusTerminal` exposes a `deposit()` and a `redeem()` function to be able to deposit/redeem some underlying against the Cygnus ERC20 that represents either a collateral position or a lending position in the protocol.

Note that, a collateral will only allow the borrowing of its corresponding borrowable(twinstar), and a borrowable will only be borrowable by collateralizing the underlying of its twinstar.

Both `CygnusBorrow` and `CygnusCollateral` inherit from their corresponding `Void`, `Model`, and `Control` smart contracts.
    * `Void` handles the potential integration of a strategy for the underlying, meaning that this contract will change for each deployment and chain.
    * `Model` actually implements the internal logic of the collateral/borrow component.
    * `Control` allows an admin to control most of the parameters of the component.

`CygnusBorrow` and `CygnusCollateral` smart contracts consist of core's higher level functions to make use of the protocol.

The rest of the contracts: `Hangar18` and the orbiters, are used to help the deployment and management of each shuttle.

### Periphery

## Roles and privileges
* Shuttle admin ?
* Hangar admin 
* Periphery admins ?

### Privileged threats

## Scope and Versions

Scope in file scope-commits

## Issues

