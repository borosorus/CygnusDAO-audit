# CygnusDAO Security Review by Borosorus

### Disclaimer

A time-boxed audit doesn't guarantee the absence of security issues.

### Scope and Versions

    Revision 0
    ---------------------------------------------------
    periphery: db395115c845165c8b774a308b5d8ef89d1cc900
    core: 9a77e651e64b5392ac8ca831ea30f8d6e85696b4

    Revision 1 (Current)
    ---------------------------------------------------
    periphery: c35c413616001e6a76f63a8f96e2bbab8115e2a1
    core: d82dc261a0cd3bd19ad04622be0b0865f90a8639

## CygnusDAO

CygnusDAO is a lending protocol which provides isolated loan pools that consist of a pair of a collateral and a borrowable. The entirety of the collateralized and borrowable underlying tokens can earn yield while idling.

These attributes allows the users of the protocol to access the wide range of uses of a lending protocol, without missing on potential yield opportunities.

CygnusDAO's architecture is divided in two subfolders: core and periphery.
Core contains the protocol's critical lending and borrowing logic that are to be used via rather low level calls.
Periphery contains various entry points to ease the usage of the protocol.

### Core

The main components of core are the `CygnusBorrow` and the `CygnusCollateral` smart contracts. They are always created in pair and linked to each other via a `twinstar` variable that exists in each one of them. A pair of collateral and borrowable is called a `shuttle`.
They inherit from both `ERC20` and `CygnusTerminal` which allows to set a principal underlying at creation.

`CygnusTerminal` exposes a `deposit()` and a `redeem()` function to be able to deposit/redeem some underlying against the Cygnus ERC20 that represents either a collateral position or a lending position in the protocol.

Note that, a collateral will only allow the borrowing of its corresponding borrowable (twinstar), and a borrowable will only be borrowable by collateralizing the underlying of its twinstar.

Both `CygnusBorrow` and `CygnusCollateral` inherit from their corresponding `Void`, `Model`, and `Control` smart contracts.
* `Void` handles the potential integration of a strategy for the underlying: this component changes for each deployment and chain.
* `Model` actually implements the internal logic of the collateral/borrow component.
* `Control` allows an admin to control most of the parameters of the component.

`CygnusBorrow` and `CygnusCollateral` smart contracts consist of core's higher level functions to make use of the protocol.

The rest of the contracts (`Hangar18` and the orbiters) are used to help the deployment and management of each shuttle.

### Periphery

The base router contract `CygnusAltair` will be the main entry point for users to interact with the protocol.
It eases data construction and it handles leverage and liquidation functionalities.

Shuttles also have an extension contract associated to their addresses in the router, these contract will be delegated to from the router when needed. This way, the management part of leveraging/delevaraging transactions is located in those extensions, which handles stablecoin swapping to collateral through different decentralised exchanges.

`XHypervisor` is the only extension in scope.

## Roles and privileges

#### *Cygnus Admin*
* Abilities:
    - Set router extensions
    - Deploy orbiters and board shuttles
    - Kill Orbiters
    - Manage DAO Reserves address
    - Set Cygnus Pillars
    - Set Cygnus Altair (Router)
    - Set Cygnus X1 Vault
    - Sweep tokens of most contracts (except for underlyings)
    - Set parameters of each shuttle
    - Set harvester of each shuttle

**A malicious Cygnus admin could have access to most protocol funds and user funds allowed to the router.**

## Findings

### Severity Classification

| Severity               | Impact: High | Impact: Medium | Impact: Low |
| ---------------------- | ------------ | -------------- | ----------- |
| **Likelihood: High**   | Critical     | High           | Medium      |
| **Likelihood: Medium** | High         | Medium         | Low         |
| **Likelihood: Low**    | Medium       | Low            | Low         |

**Impact** - the technical, economic and reputation damage of a successful attack

**Likelihood** - the chance that a particular vulnerability gets discovered and exploited

**Severity** - the overall criticality of the risk

### Findings table

| ID   | Title                                                                           | Severity | Status |
| ---- | ------------------------------------------------------------------------------- | -------- | ------ |
| [01] | Inflation attack protection can be bypassed                                     | High     | Fixed  |
| [02] | Reserve shares are not properly computed                                        | Medium   | Fixed  |
| [03] | Collateral value computation may lead to some rounding errors                   | Medium   | Ack.   |
| [04] | Collateral transfer operation doesn't take the latest borrow balance in account | Medium   | Fixed  |
| [05] | Liquidation fee shares are not properly computed                                | Medium   | Fixed  |
| [06] | Extension isn't storage aligned                                                 | Low      | Fixed  |
| [07] | `else` case not handled                                                         | Low      | Fixed  |

### [01] Inflation attack protection can be bypassed [HIGH] [Fixed]

The protection against inflation attacks forces the first user to mint more than 1000 shares. 
    Those first 1000 shares are then sent to the zero address, so that no one controls them:

```
// Avoid inflation attack on the vault - This is only for the first pool depositor as after there will always
// be 1000 shares locked in zero address
if (totalSupply() == 0) {
    // Update shares for first depositor
    shares -= 1000;

    // Lock initial tokens
    _mint(address(0), 1000);
}
```

Note however, that there exist one special way to mint shares in the Borrowing contract: the `mintReservesPrivate()` function.

An attacker could:

        1. Send a small amount of the underlying strategy assets to the contract
        2. Borrow an amount of funds to make it so that accruing the interests will mint 1 wei of shares to the reserve
        3. Repay the debt
        4. The total supply is now equal to 1 wei, the protection can be bypassed
    
The inflation attack allows the attacker to steal money from the first depositors, and might also lead to some unforeseen cases of rounding errors because of an extreme exchange rate value.

Note that in thise case, the attacker takes more financial risk than for the usual inflation attack, as there already exists 1 wei of shares that isn't in its possession.

**[Fixed in revision 1]**

`mintReservesPrivate()` doesn't use `_convertToShares()` anymore to compute shares to mint, but always use `totalSupply()` instead.

### [02] Reserve shares are not properly computed [MEDIUM] [Fixed]
    
When accumulating borrow interests, a portion of them are collected as reserves for each borrowable.
The responsible function uses `_convertToShares()` to compute a share amount based on the interests amount:

```
function mintReservesPrivate(uint256 interestAccumulated) private returns (uint256 newReserves) {
    // Calculate new reserves to mint based on the reserve factor and latest exchange rate
    newReserves = _convertToShares(interestAccumulated.mulWad(reserveFactor));

    // Check to mint new reserves
    if (newReserves > 0) {
        // Get the DAO Reserves current address
        address daoReserves = hangar18.daoReserves();
    
        // Mint to Hangar18's latest `daoReserves`
        _mint(daoReserves, newReserves);
    }
}
```

The issue lies in the fact that `_convertToShares()` uses `totalAssets()` which has just been increased by the total interests amount before this function was called.

Computing shares to mint for an amount that has already been included in `totalAssets()` will result in a smaller amount of shares than expected.

**[Fixed in revision 1]**

`mintReservesPrivate()` now properly computes the shares to mint by integrating users' interests into the division.

### [03] Collateral value computation may lead to some rounding errors [MEDIUM] [Acknowledged]

To compute the value of a collateral amount in the underlying, the collateral contract uses the oracle, which returns a price denoted in the borrowable underlying's decimals.

```
uint256 collateralInUsd = amountCollateral.mulWad(getLPTokenPrice());
```

Note the use of `mulWad()` from the `FixedPointMathLib`, that returns `div(mul(x, y), WAD)`, with x being `amountCollateral` and y the lp token price.

The use of such a fonction on non-WAD denominated amounts leads to some significant rounding issues.

Let's consider an example:

If an LP has 8 decimals, and the borrowing underlying has 6 decimals, `mul(x, y)` would in this case result in an amount denominated with 14 decimals, rounding down to zero all amounts under 4 digits, and rounding down by a large amount values that are greater.

This computation is also present in some view functions: `getBorrowerPosition()` ; `collateralTvlUsd()`.

**[Acknowledged]**

CygnusDAO acknowledged the issue and answered:
    
    We use different Deployer contracts for each type of collateral. If we deploy collaterals with less than 18 decimals, we will rely on another library.

### [04] Collateral transfer operation doesn't take the latest borrow balance in account [MEDIUM] [Fixed]

When transfering a collateral amount, the collateral contract checks whether the source is allowed to move this collateral with regards to its debt position. 

This logic happens in the `_beforeTokenTransfer()` hook, and calls the function `canRedeem()` to fetch the borrow balance and compute the needed collateral:

```
function _beforeTokenTransfer(address from, address, uint256 amount) internal view override(ERC20) {
    // Escape in case of `flashRedeemAltair()` and `mint()`
    // 1. This contract should never have CygLP outside of flash redeeming. If a user is flash redeeming it requires them
    // to `transfer()` or `transferFrom()` to this address first, and it will check `canRedeem` before transfer.
    if (from == address(this)) return;

    /// @custom:error InsufficientLiquidity Avoid transfers or burns if there's shortfall
    if (!canRedeem(from, amount)) revert CygnusCollateral__InsufficientLiquidity();
}
```

However, the borrow balance fetched in `canRedeem()` isn't up to date, as interests aren't accrued in the borrowing contract.

**[Fixed in revision 1]**

Interests are now accrued on the borrowable before calling `canRedeem()`.

### [05] Liquidation fee shares are not properly computed [MEDIUM] [Fixed]

In the `seizeCygLP()` function, a collateral shares amount is computed out of the debt amount that will be repaid:

```
        ...

        // Get the equivalent of the repaid amount + liquidation bonus, in the underlying LP
        uint256 seizedLPs = repayAmount.fullMulDiv(liquidationIncentive, lpTokenPrice);

        // Convert the LP amount seized to CygLP shares to seize this amount
        cygLPAmount = _convertToShares(seizedLPs);

        // Transfer the repaid amount + liq. incentive to the liquidator, escapes canRedeem
        _transfer(borrower, liquidator, cygLPAmount);

        // Initialize and check if liquidation fee is set
        uint256 daoFee;

        // Check for protocol fee
        if (liquidationFee > 0) {
            // Get the liquidation fee amount that is kept by the protocol
            daoFee = cygLPAmount.mulWad(liquidationFee);
        
        ...

```

Here, the total amount of seized lp should be equal to `repaidValue * (liquidationIncentive + liquidationFee)`.

However note that the `daoFee` is actually computed from `cygLPAmount`, which already contains the liquidation incentive so we rather end up with `(repaidValue * liquidationIncentive) * (1 + liquidationFee)`, consequently seizing more collateral than needed.

**[Fixed in revision 1]**

The dao fee is now computed on the proper amount.

### [06] Extension isn't storage aligned [LOW] [Fixed]

`CygnusAltairX` isn't aligned in storage with `CygnusAltair`, and has a function to modify a storage variable: `setName()`.

Delegating to the extension to modify the router's name could have unforeseen consequences on storage variables.

**[Fixed in revision 1]**

The `setName()` function was removed, and name variables made constant.

### [07] `else` case not handled [LOW] [Fixed]

In the `CygnusAltairX` smart contract, the function `_swapTokensAggregator()` redirects to the right swap function depending on the parameter `dexAggregator`, however, if this parameter doesn't correspond to one of the pre-defined aggregator, the function will return as if it executed successfully.
Note that `dexAggregator` is a user-defined parameter.

**[Fixed in revision 1]**

The else case now reverts.
    
## Notes

| ID    | Title                                                                           
| ----- | -------------------------------------------------------------------------------                          
| [N01] | Limited front-end provided data validation
| [N02] | Extensions will always exist in router
| [N03] | `_previewTotalBalance()` should always return the current balance
| [N04] | Debt Ratio tuning
| [N05] | Reentrancy and external integrations

### [N01] Limited front-end provided data validation

Most user provided data in the router isn't validated.
A hijacked front-end could make a user interact with the trusted router but with malicious data that could lead to some funds loss.

### [N02] Extensions will always exist in router

When adding an extension to the router, the extension's address is saved as a valid extension in the mapping `isExtension`.
However, there is no way to remove an extension from this mapping.

### [N03] `_previewTotalBalance()` should always return the current balance

The view function `_previewTotalBalance()` from `CygnusTerminal` should always return the current balance from the underlying's strategy. 

In the case of compound's V3 Comet USDC, the fetched balance is updated with the latest index and works as expected.

However, when dealing with Compound V2 for example, fetching the current balance from a view function doesn't return the current balance but the latest balance. Fetching an outdated balance could potentially break the accounting of core.

### [N04] Debt Ratio tuning

The `debtRatio` parameter must be kept far under 100% for liquidations to be profitable and therefore for the protocol's liquidation mechanism to work properly.

### [N05] Reentrancy and external integrations

The core logic of the protocol allows for some reentrant behavior. Consequently, an external actor (protocol/smart contract) looking at the state of the protocol at any point in time might be tricked.
External protocols integrating with CygnusDAO should take extra care in securing their implementations against this kind of read-only reentrancies.