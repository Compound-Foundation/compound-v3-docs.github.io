---
layout: docs-content
title: Compound III Docs | ERC-4626 Wrapper
permalink: /erc4626-wrapper/
docs_namespace: v3

## Element ID: In-page Heading
sidebar_nav_data:
  erc-4626-wrapper: ERC-4626 Wrapper
  deployed-contracts: Deployed Contracts
  depositing-the-base-token: Depositing the Base Token
  depositing-a-comet-position: Depositing a Comet Position
  resolving-a-wrapper: Resolving a Wrapper
  integration-notes: Integration Notes
---

# ERC-4626 Wrapper

A Compound III supply position is a rebasing balance: `comet.balanceOf(account)` grows every second as interest accrues, with no transfer and no event. Most DeFi integrations (AMMs, lending markets, vault accounting) cannot hold a rebasing balance without leaking or mis-recording the yield.

The Wrapped Comet ERC-4626 vault converts a base-asset supply position into a fixed number of non-rebasing shares. The share count never changes; interest accrues as an increase in the amount of the underlying each share redeems for, like any other ERC-4626 vault.

* The vault's `asset()` is the Comet market token (for example cUSDCv3), and `totalAssets()` is the vault's live `comet.balanceOf(address(this))`.
* Wrappers are immutable. They have no owner, admin, pauser, sweep function, or upgrade path.
* Supported markets are Ethereum mainnet cUSDCv3 and cUSDTv3.
* Wrapper shares have 18 decimals on every market.

The wrapper is an extension that sits on top of Compound III. It does not change any Comet market, and Comet markets do not depend on it.

### Deployed Contracts

Ethereum mainnet:

| Contract | Symbol | Address | Asset |
| --- | --- | --- | --- |
| Wrapped Compound USDC | wcUSDCv3 | [0x89dd54aB898944BB4cb8a2C403B7D511F88f9E73](https://etherscan.io/address/0x89dd54aB898944BB4cb8a2C403B7D511F88f9E73){:target="_blank"} | cUSDCv3 `0xc3d688B66703497DAA19211EEdff47f25384cdc3` |
| Wrapped Compound USDT | wcUSDTv3 | [0xD7c9F42a35C8d13b71D4c0B024AE9eb0e77b9cFf](https://etherscan.io/address/0xD7c9F42a35C8d13b71D4c0B024AE9eb0e77b9cFf){:target="_blank"} | cUSDTv3 `0x3Afdc9BCA9213A35503b077a6072F3D0d5AB0840` |
| WrappedCometERC4626Factory | | [0xf77f675a383eB43fa032ea3c6d1dd4dC56B69BF6](https://etherscan.io/address/0xf77f675a383eB43fa032ea3c6d1dd4dC56B69BF6){:target="_blank"} | |

### Depositing the Base Token

Accounts holding USDC or USDT can enter and exit in one transaction with the router functions. These never require `comet.allow`: the wrapper supplies to Comet on its own account and withdraws directly to the caller.

#### Wrapped Comet ERC-4626

```solidity
function supplyAndWrap(uint256 amount) external returns (uint256 shares);
function supplyAndWrapWithPermit(uint256 amount, uint256 deadline, uint8 v, bytes32 r, bytes32 s) external returns (uint256 shares);
function supplyAndWrapWithPermit2(uint256 amount) external returns (uint256 shares);
function unwrapAndWithdraw(uint256 shares) external returns (uint256 amount);
```

* `amount`: For the `supplyAndWrap` functions, the amount of the base token to supply, in base token units (6 decimals for USDC and USDT).
* `shares`: For `unwrapAndWithdraw`, the number of wrapper shares to burn (18 decimals).
* `RETURNS`: `supplyAndWrap*` returns the shares minted to the caller. `unwrapAndWithdraw` returns the amount of base token sent to the caller.

Authorization required before calling:

* `supplyAndWrap`: `baseToken.approve(wrapper, amount)`.
* `supplyAndWrapWithPermit`: an EIP-2612 signature over `amount`. Mainnet USDT does not implement `permit`, so do not use this function on wcUSDTv3.
* `supplyAndWrapWithPermit2`: an ERC-20 approval from the caller to [Permit2](https://etherscan.io/address/0x000000000022D473030F116dDEE9F6B43aC78BA3){:target="_blank"}, then a Permit2 allowance for the wrapper of at least `amount`.

All router functions act only for `msg.sender`; none take a receiver or owner argument. To exit a whole position, call `unwrapAndWithdraw(wrapper.balanceOf(msg.sender))`.

#### Solidity

```solidity
IERC20(usdc).approve(address(wrapper), 1000e6);
uint256 shares = wrapper.supplyAndWrap(1000e6);

uint256 usdcOut = wrapper.unwrapAndWithdraw(shares);
```

#### Ethers.js v5.x

```js
const wrapper = new ethers.Contract(wrapperAddress, abiJson, signer);
await usdc.approve(wrapperAddress, 1000e6);
const tx = await wrapper.supplyAndWrap(1000e6);
```

### Depositing a Comet Position

Accounts that already hold a Compound III supply position can use the standard ERC-4626 functions, denominated in the Comet market token (for example cUSDCv3, 6 decimals).

```solidity
function deposit(uint256 assets, address receiver) external returns (uint256 shares);
function depositAll(uint256 minShares, address receiver) external returns (uint256 shares);
function mint(uint256 shares, address receiver) external returns (uint256 assets);
function withdraw(uint256 assets, address receiver, address owner) external returns (uint256 shares);
function redeem(uint256 shares, address receiver, address owner) external returns (uint256 assets);
```

* `depositAll`: Not part of ERC-4626. Deposits the caller's entire Comet balance, read at execution, and reverts if fewer than `minShares` are minted. A reasonable `minShares` is `previewDeposit(comet.balanceOf(caller))` minus a small margin.

This path requires the caller to first call `comet.allow(wrapper, true)` on the Comet market. Comet's manager permission is account-wide: it also covers the caller's collateral and the ability to borrow against it. The wrapper never calls `transferAssetFrom` or `withdrawFrom`, and always pulls from `msg.sender`, so it cannot use that permission against a depositor. Callers should still revoke it with `comet.allow(wrapper, false)` once they have exited. Use the router functions above to avoid the grant entirely.

`deposit` and `mint` revert with `InsufficientPositiveCometBalance` rather than open a borrow when the requested amount exceeds the caller's Comet balance.

### Resolving a Wrapper

The factory registers at most one canonical wrapper per Comet market. Integrators should resolve wrappers from the factory rather than hard-coding or recomputing addresses.

#### WrappedCometERC4626Factory

```solidity
function getWrapper(address comet) external view returns (address);
function allWrappers(uint256 index) external view returns (address);
function allWrappersLength() external view returns (uint256);
```

* `comet`: The address of the Compound III market.
* `RETURNS`: The canonical wrapper for the market, or the zero address if none exists.

#### Solidity

```solidity
WrappedCometERC4626Factory factory = WrappedCometERC4626Factory(0xf77f675a383eB43fa032ea3c6d1dd4dC56B69BF6);
address wrapper = factory.getWrapper(0xc3d688B66703497DAA19211EEdff47f25384cdc3);
```

### Integration Notes

**Rounding costs a few base units.** Comet floors present value to principal on both sides of every transfer, roughly one base unit per transfer at current indexes. The wrapper prices every mint and burn off the measured change in its own Comet balance, so rounding always falls in the vault's favor and a round trip can never profit. A same-block `deposit` then `redeem` can cost up to 9 base units (0.000009 USDC). A `withdraw(assets)` can deliver up to 2 base units less than `assets`; measure the received amount rather than asserting an exact figure.

**Previews are bounds, not exact amounts.** `previewMint` may overstate what `mint` charges and `previewRedeem` may understate what `redeem` pays, each within the direction ERC-4626 requires. `mint` returns the caller's measured debit and `redeem` returns the receiver's measured credit. `Deposit` and `Withdraw` events carry the vault's measured balance change, so the return value and event amount can differ by up to 2 base units.

**Use `depositAll` and `redeem` for whole positions.** Anyone can call `comet.transfer(account, 0)`, which reduces the account's Comet balance by about one base unit through rounding. A `deposit` sized to a balance read off-chain, or a `withdraw` sized to `maxWithdraw`, can be front-run this way and revert. `depositAll` reads the balance at execution and `redeem(maxRedeem(owner), ...)` burns exact shares, so neither is affected. `withdraw(maxWithdraw(owner), ...)` can also leave a few base units of unburnable dust shares behind.

**Shares and assets use different decimals.** Shares have 18 decimals; cUSDCv3, cUSDTv3, USDC, and USDT have 6. Convert with `convertToAssets` and `convertToShares` rather than hard-coded scaling.

**Wrapped positions do not earn COMP.** Protocol rewards accrue to the wrapper's address, and the wrapper has no function to claim or distribute them. Reward speeds on cUSDCv3 and cUSDTv3 are currently zero. If governance enables rewards on these markets, rewards accruing to wrapped supply would be permanently locked in the wrapper.

**Pauses.** While Comet's transfer pause is active, `maxDeposit`, `maxMint`, `maxWithdraw`, and `maxRedeem` return 0 and the ERC-4626 functions revert. The router functions are governed by Comet's supply and withdraw pauses instead and revert inside Comet when those are active.

**Dust deposits revert.** Any call that would move zero assets or mint or burn zero shares reverts with a typed error such as `ZeroAssetsCredited` or `ZeroSharesMinted`, rather than succeeding as a no-op.

**Tokens sent directly to a wrapper are not recoverable.** A direct Comet transfer to a wrapper raises the share price for existing holders. Collateral or other tokens sent to a wrapper are stranded permanently.
