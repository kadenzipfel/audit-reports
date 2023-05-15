# Golom Codearena Report

## Metadata

- Repo: https://github.com/code-423n4/2022-07-golom
- Contest page: https://code4rena.com/contests/2022-07-golom-contest

## Security Findings

#### Protocol fees will be locked in the `RewardDistributor` contract once reward token `totalSupply` exceeds 1 billion

##### Severity: CRITICAL

##### Context:

- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/rewards/RewardDistributor.sol#L100
- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/rewards/RewardDistributor.sol#L48

##### Description:

`RewardDistributor.sol#L100-103` indicates that once the `totalSupply` of the reward token exceeds 1 billion, we immediately return out of the `addFee` method, presumably to prevent further emissions of the reward token. However, this method is also responsible for the accounting logic related to ETH-denominated protocol fees. Preventing this method from executing prevents trader, exchange, and staker rewards from being accounted and thus they become unclaimable. Furthermore there is no other method of withdrawing ETH from the contract, and as such all ETH accrued as protocol fees will become permanently locked in the contract.

Given an initial supply of 212,500,000 reward tokens (150,000,000 airdrop + 62,500,000 genesis reward) and a constant daily emission of 600,000 reward tokens. We estimate that it will take ~1313 days for the `totalSupply` to exceed 1 billion and trigger the contract to permanently lock funds.

##### Remediation:

We recommend Golom to separate the accounting logic for token emissions and ETH-denominated protocol fees such that they may cutoff token emissions without affecting protocol fee accounting.

#### Incorrect accounting logic in `GolomTrader._settleBalances` leads to incorrect amount being paid to seller, permanently locking funds in the contract

##### Severity: CRITICAL

##### Context:

- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/core/GolomTrader.sol#L389
- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/core/GolomTrader.sol#L396

#### Description:

In `GolomTrader._settleBalances` lines 389-394 and 396-399, accounting logic is performed to calculate the amount of Ether to be sent to the seller. This calculation in both places incorrectly includes a value for the protocol fee (`protocolfee`) which is already multiplied by the amount of orders being filled, then multiplies that value again by the amount of orders being filled. This results in the seller receiving less ether than intended for all filled bids with an amount greater than 1.

Furthermore, since `GolomTrader` does not provide any logic to withdraw Ether from the contract, the ETH will be permanently locked in the contract.

#### Remediation:

We recommend that Golom carefully considers and tests all logic in the `_settleBalances` function to remediate and further prevent accounting errors.

#### `GolomTrader.fillAsk` enforces a greater protocol fee than intended

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/core/GolomTrader.sol#L211
- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/core/GolomTrader.sol#L262
- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/core/GolomTrader.sol#L396

##### Description:

At `GolomTrader.sol#L211`, as described by the comment on line 210, this require statement intends to enforce that the order total amount (`o.totalAmt`) is >= the sum of all payments as well as a 50 basis point fee using the following logic: `o.totalAmt >= o.exchange.paymentAmt + o.prePayment.paymentAmt + o.refererrAmt + (o.totalAmt * 50) / 10000`. This logic contains an incorrect fee calculation by referencing the total amount (including the fee) in calculating what the fee should be. This leads to the fee being 50 basis points higher than intended.

##### Remediation:

We recommend Golom to properly calculate the total amount using the following logic as an example:

```
uint256 paymentTotal = o.exchange.paymentAmt + o.prePayment.paymentAmt + o.refererrAmt;
require(
    o.totalAmt >= paymentTotal + (paymentTotal * 50) / 10000,
    'amt not matching'
);
```

#### `GolomTrader.fillBid` and `GolomTrader.fillCriteriaBid` enforce incorrect total order amounts

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/core/GolomTrader.sol#L285
- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/core/GolomTrader.sol#L342

##### Description:

`GolomTrader.fillBid` and `GolomTrader.fillCriteriaBid` both enforce incorrect total order amounts (`o.totalAmt`). The following errors have been noted:

- Both functions exclude the protocol fee in their calculations
- `fillBid` enforces that `o.totalAmt` is strictly greater than what should be the minimum amount. Instead should be >= assuming that the calculated minimum amount is correct
- `fillCriteriaBid` doesn't include the optional `p.paymentAmt` sent in `_settleBalances`

##### Remediation:

We recommend Golom thoroughly considers and enforces the proper total order amount (`o.totalAmt`) in both `GolomTrader.fillBid` and `GolomTrader.fillCriteriaBid`.

#### `GolomTrader.fillAsk` allows for greater than required amount to be paid

##### Severity: Low

##### Context:

- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/core/GolomTrader.sol#L217

##### Description:

In `GolomTrader.sol#L217`, it's enforced that the `msg.value` is greater than or equal to the expected amount. Allowing for a greater amount than expected can result in loss of funds for buyers in the case of a front-end bug or user error.

#### Remediation:

We recommend that Golom instead enforces the `msg.value` to be strictly equal to the expected amount to prevent loss of funds from a front-end/user error.

#### `RewardDistributor` 'rewards' view functions' contain unbounded loop which can lead to block gas limit DoS if used improperly

##### Severity: Low

##### Context:

- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/rewards/RewardDistributor.sol#L226
- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/rewards/RewardDistributor.sol#L258
- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/rewards/RewardDistributor.sol#L273

##### Description:

`RewardDistributor`'s `stakerRewards`, `traderRewards` and `exchangeRewards` methods all contain an unbounded loop, which if called by a state changing function could lead to a block gas limit DoS.

##### Remediation:

We recommend that Golom tracks this data using an off-chain indexer to prevent possible misuse.

## Gas Optimizations

#### Make variable immutable

##### Severity: Gas Optimization

##### Context:

- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/vote-escrow/VoteEscrowCore.sol#L300

##### Description:

The `token` variable at line 300 should be made immutable.

#### Remove unused storage variable

##### Severity: Gas Optimization

##### Context:

- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/rewards/RewardDistributor.sol#L63

##### Description:

The `rewardLP` mapping at `RewardDistributor.sol#L63` is never used and should be removed to decrease deployment costs.

#### Combine corresponding storage variables

##### Severity: Gas Optimization

##### Context:

- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/rewards/RewardDistributor.sol#L61
- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/rewards/RewardDistributor.sol#L62
- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/rewards/RewardDistributor.sol#L120
- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/rewards/RewardDistributor.sol#L121

##### Description:

We can see in `RewardDistributor.addFee#L120-121` that `rewardTrader` and `rewardExchange` are simply percentages of `tokenToEmit - stakerReward`, 67% and 33% respectively. As such we can reduce them to one mapping containing the full value of `tokenToEmit - stakerReward` and perform the percentage logic elsewhere to save gas by reducing unnecessary expensive SLOAD and SSTORE operations.

#### Reward tokens can be minted directly to recipient instead of being minted to contract and later transferred to recipient

##### Severity: Gas Optimization

##### Context:

- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/rewards/RewardDistributor.sol#L122
- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/rewards/RewardDistributor.sol#L151
- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/rewards/RewardDistributor.sol#L165
- https://github.com/code-423n4/2022-07-golom/blob/e5efa8f9d6dda92a90b8b2c4902320acf0c26816/contracts/rewards/RewardDistributor.sol#L208

##### Description:

In `RewardDistributor.addFee` we mint the total emitted tokens from the current epoch to the distributor contract. These tokens are later transferred to the corresponding recipient on lines 151, 165 and 208. Instead of performing a mint transaction then later performing transfers, we can simply mint the tokens directly to the recipients on lines 151, 165, and 208.

#### Avoid repeating logic

##### Severity: Gas Optimization

##### Context:

- Throughout the contracts

##### Description:

It is noted that throughout the contracts, logic is often being repeated. Instead, repeated logic should be saved to a variable for cheaper reuse.

#### Use optimal comparison operator

##### Severity: Gas Optimization

##### Context:

- Throughout the contracts

##### Description:

It is noted that throughout the contracts, suboptimal comparison operators are used, and should be modified as follows:

- When checking that uints are greater than 0, use `value != 0` instead of `value > 0` to save 44 gas
- When checking that a uint is less/greater than or equal to another, use, e.g. `value > otherValue - 1` instead of `value >= otherValue` to save 38 gas

#### Use optimal loop incrementor

##### Severity: Gas Optimization

##### Context:

- Throughout the contracts

##### Description:

It is noted that throughout the contracts, suboptimal loop incrementors are used. Instead, it should be optimized as follows:

Instead of:

```
for (uint256 i = 0; i < array.length; i++) { ... }
```

Use the optimal loop incrementor:

```
for (uint256 i = 0; i < array.length;) {
    ...
    unchecked {
        ++i;
    }
}
```
