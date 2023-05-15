# Trader Joe V2 Codearena Report

## Metadata

- Repo: https://github.com/code-423n4/2022-10-traderjoe

## Security Findings

#### Low-level call token transfer may lead to false positive

##### Severity: HIGH

##### Context:

- https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/libraries/TokenHelper.sol#L40
- https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/libraries/TokenHelper.sol#L74

##### Description:

`TokenHelper.safeTransfer/safeTransferFrom` are used in `LBPair` to execute token transfers. `safeTransfer/safeTransferFrom` perform a low-level call to the token contract, executing the transfer method. However, low-level calls in solidity will always return `success` if the calling account is non-existent. As a result, calls using these methods may falsely succeed and continue execution of the method with a failed token transfer in the case the token contract has been self destructed or simply does not exist.

##### Remediation:

It is recommended that either:

- A check is added to ensure the contract being called exists, or
- A high level transfer call is used in place of the low-level call.

#### `LBToken.safeTransferFrom/safeBatchTransferFrom` do not check that contract recipients implement `ERC1155TokenReceiver`

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBToken.sol#L131
- https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBToken.sol#L149

##### Description:

`LBToken` is a token contract similar to that of the ERC-1155 token standard. ERC-1155 tokens enforce a check that recipient contracts implement `ERC1155TokenReceiver` to prevent funds from getting locked in contracts that do not have logic to manage the tokens. Since `LBToken` implements a similar interface to ERC-1155, including the `safeTransferFrom/safeBatchTransferFrom` method names, it is expected that the token will enforce an `ERC1155TokenReceiver` check. As a result, tokens may end up locked in contracts that do not have the logic to manage the tokens.

##### Remediation:

It is recommended that the ERC-1155 `ERC1155TokenReceiver` check is implemented in `safeTransferFrom/safeBatchTransferFrom`, with careful reentrancy prevention throughout the codebase. Otherwise, the `safe` prefix should be removed from the method names so as not to give developers a false sense of security when writing contracts to interact with Liquidity Book.

#### Unprotected `address(this)` checks allow attacker to delegatecall from another contract to spoof values such as the token balances of `LBPair` instances

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/libraries/TokenHelper.sol#L68

##### Description:

`LBPair` token balance checks are intended to exclusively read the token balances of the `LBPair` instance. However, it is possible for an attacker to make a delegatecall into one of the methods reading the token balance, overriding the `address(this)` check to be read in the context of the calling contract, resulting in unexpected effects.

##### Remediation:

It is recommended that a check is included to confirm that `address(this)` is the correct address, e.g.:

```
address private immutable original;

constructor() {
    original = address(this);
}

function checkNotDelegateCall() private view {
    require(address(this) == original);
}

modifier noDelegateCall() {
    checkNotDelegateCall();
    _;
}
```

#### DoS possible when swapping through too many bins

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L326

##### Description:

The minimum bin step, `MIN_BIN_STEP` is 1 basis point, thus the maximum number of bins is 10,000. When making a swap, the trade is routed through as many bins as necessary or intended. The logic of swapping through bins is relatively expensive, making each bin swapped through cost ~15.5k gas. With a current block gas limit of 30 million, it takes only ~1950 bins to be swapped through to exceed the block gas limit and prevent the transaction from executing.

An attacker could add 1 wei of liquidity to every bin in every pool, forcing trades to be made through each bin, preventing users from using the system, or at the very least making many transactions more expensive than necessary.

##### Remediation:

It is recommended that either the minimum bin step is increased significantly and/or a minimum liquidity per bin restriction is added.

#### Earned interest to tokens in pool can be stolen

##### Severity: Low

##### Context:

- https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L312

##### Description:

In `LBPair.swap`, the contract token balance is checked to confirm that the user executing the swap sent in a sufficient amount of tokens. It's possible that in the case of a non-standard ERC-20 token, an attacker may call a method on the token to accrue interest to the pool contract, such that the balance increases sufficiently for them to execute their swap without actually transferring any tokens.

##### Remediation:

It is recommended that the possible effects of non-standard ERC-20 tokens are well documented to minimize loss of user funds.

#### Avoid emitting events inside loops

##### Severity: Gas optimization

##### Context:

- https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L342
- https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L581
- https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L665

##### Description:

In `swap`, `mint` and `burn`, a loop is run to execute logic on each bin involved in the transaction. Within this loop, events are emitted. It is recommended that instead of emitting these duplicate events many times per transaction, the events are removed from the loop and modified to emit a total of all actions completed within the loop.

#### Don't attempt fee transfer if fees to collect in `collectFees` are 0

##### Severity: Gas optimization

##### Context:

- https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L716
- https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L723

##### Description:

In `LBPair.collectFees`, checks are performed to see if `amountX` and `amountY` are greater than 0. However, outside of these checks, a `safeTransfer` is attempted which will simply spend a small amount of gas if the amount to transfer is zero. This can easily be avoided by placing the `safeTransfer` methods within the checks that `amountX` and `amountY` are greater than 0.

#### Set `_mintInfo.totalDistributionX/Y` as the value of `_mintInfo.distributionX/Y` instead of incrementing from 0

##### Severity: Gas optimization

##### Context:

- https://github.com/code-423n4/2022-10-traderjoe/blob/79f25d48b907f9d0379dd803fc2abc9c5f57db93/src/LBPair.sol#L505

##### Description:

In `LBPair.mint`, `_mintInfo.totalDistributionX/Y` are initialized as 0, then subsequently incremented to the value `_mintInfo.distributionX/Y`. Since the value of `_mintInfo.totalDistributionX/Y` will always 0 at this stage, instead of incrementing we can simply do `_mintInfo.totalDistributionX/Y = _mintInfo.distributionX/Y`.
