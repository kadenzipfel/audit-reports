Polynomial Codearena Report

## Metadata
- Repo: https://github.com/code-423n4/2023-03-polynomial

## Security

#### Fees will be permanently locked in LiquidityPool

##### Severity: High

##### Context: 

- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L494-L521

##### Description

###### Impact

Fees received by the `LiquidityPool` contract when a short position is opened are unaccounted for and end up permanently locked in the contract, incurring a loss to liquidity providers.

###### Proof of Concept

When opening and closing short positions, the `LiquidityPool` requires either a fee to be transferred in from the `user` account or withheld in the case that sUSD is being transferred from the contract to the user. Consider for example how fees must be transferred from the `user` in `openLong`:

```
// openLong() - LiquidityPool.sol:L441-444
uint256 fees = orderFee(int256(amount));
totalCost = tradeCost + fees;

SUSD.safeTransferFrom(user, address(this), totalCost);
```

Since the contract is taking a fee, it's necessary to update the corresponding `totalFunds` state variable to reflect the collected fee:

```
// openLong() - LiquidityPool.sol:L453
totalFunds += feesCollected - externalFee;
```

In `openShort`, however, we deduct a fee from the amount being transferred to the user as expected:

```
// openShort() - LiquidityPool.sol:L505-508
uint256 fees = orderFee(-int256(amount));
totalCost = tradeCost - fees;

SUSD.safeTransfer(user, totalCost);
```

But there is no corresponding update to `totalFunds`.

Since we can't withdraw beyond `totalFunds` (the following will underflow and revert):

```
// withdraw() - LiquidityPool.sol:255
totalFunds -= susdToReturn;
```

All fees collected in `openShort` are not accounted for and permanently locked in the contract. Since these fees would go to the liquidity providers, this is considered a loss of funds for users.

###### Recommended Mitigation

Include the necessary update to `totalFunds` in `openShort`:

```
totalFunds += feesCollected - externalFee;
```


#### Admin can steal all underlying tokens from Kangaroo Vault

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L543-L550

##### Description

###### Impact

`KangarooVault.saveToken` is intended to allow the contract admin to retrieve tokens lost in the contract, e.g. by being accidentally transferred. Contrary to indended ability, the admin can withdraw any and all underlying tokens from the vault.

###### Proof of Concept

Documentation for `saveToken` states that it can "Save ERC20 token from the vault (not SUSD or UNDERLYING)". However, there exists only a check that the token to withdraw is not sUSD, without an accompanying check that it's also not the underlying token:

```
// saveToken() - KangarooVault.sol:L548
require(token != address(SUSD));
```

Therefore, contrary to documentation, it's entirely possible for the admin to withdraw any and all underlying tokens from the vault.

###### Recommended Mitigation

Simply include a check that the token to be withdrawn is also not the underlying token.


#### `KangarooVault` `maxDepositAmount` not enforced

##### Severity: Medium

##### Context: 

- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/KangarooVault.sol#L88

##### Description

In `KangarooVault`, there exists a `maxDepositAmount` state variable which exists obviously to enforce a maximum deposit amount. `maxDepositAmount` is, however, not actually used to validate the deposit amount as it clearly is intended, allowing users to deposit without any upper bound and disabling any ability for an admin to later enforce a maximum deposit amount.


#### LiquidityPool depositFee/withdrawalFee not applied on queued deposits/withdrawals

##### Severity: Medium

##### Context: 

- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L93
- https://github.com/code-423n4/2023-03-polynomial/blob/aeecafc8aaceab1ebeb94117459946032ccdff1e/src/LiquidityPool.sol#L96

##### Description

Deposit and withdrawal fees in `LiquidityPool` are enforced on instant deposits and withdrawals:

```
// deposit() - LiquidityPool.sol:L186-192
uint256 fees = amount.mulWadDown(depositFee);
uint256 amountForTokens = amount - fees;
...
SUSD.safeTransferFrom(msg.sender, feeReceipient, fees);
SUSD.safeTransferFrom(msg.sender, address(this), amountForTokens);
```

```
// withdraw() - LiquidityPool.sol:251-254
uint256 susdToReturn = tokens.mulWadDown(tokenPrice);
uint256 fees = susdToReturn.mulWadDown(withdrawalFee);
SUSD.safeTransfer(feeReceipient, fees);
SUSD.transfer(user, susdToReturn - fees);
```

In queued deposits and withdrawals, however, fees are ignored entirely.