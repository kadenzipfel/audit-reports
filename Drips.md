Drips Codearena Report

## Metadata

- Repo: https://github.com/code-423n4/2023-01-drips

## Security

#### Hacked or malicious owner can steal all tokens

##### Severity: Medium

##### Context: 

- https://github.com/code-423n4/2023-01-drips/blob/9fd776b50f4be23ca038b1d0426e63a69c7a511d/src/Managed.sol#L157

##### Description:

Tokens for all active drips are stored in the DripsHub contract. Since DripsHub is an upgradeable ERC1967Proxy, a malicious or hacked owner can simply upgrade the contract to include e.g., the following function:

```
function stealTokens(IERC20 token, address to, uint256 amt) external {
	require(msg.sender == <hacker_address>);
	token.safeTransfer(to, amt);
}
```

and immediately steal all tokens in the contract. Additionally, all token approvals to the contract can be exploited. 

Should implement a timelock to give users time to withdraw tokens and revoke approvals or remove upgradeability altogether.


#### `Caller.callBatched` doesn't enforce `msg.value` is equal to sum of call values

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2023-01-drips/blob/9fd776b50f4be23ca038b1d0426e63a69c7a511d/src/Caller.sol#L198

##### Description:

For each call in `callBatched`, we pass a value to be sent along with the call:

```
for (uint256 i = 0; i < calls.length; i++) {
	Call memory call = calls[i];
	returnData[i] = _call(sender, call.to, call.data, call.value);
}
```

The sum of the values of the calls should be equal to `msg.value`. If `msg.value` is insufficient, the batch will revert. If `msg.value` is greater than the sum of the values of the calls, then excess ETH will be left in the contract, which anyone can take by using `callBatched` with a call which sends the ETH to their address with `msg.value == 0`.


#### DoS with block gas limit in `squeezeDrips`

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2023-01-drips/blob/9fd776b50f4be23ca038b1d0426e63a69c7a511d/src/Drips.sol#L450

##### Description:

Squeezing drips requires verifying the entire history of that drip. This means iterating over an unbounded loop of the size of the history

```
for (uint256 i = 0; i < dripsHistory.length; i++) {
	DripsHistory memory drips = dripsHistory[i];
	bytes32 dripsHash = drips.dripsHash;
	if (drips.receivers.length != 0) {
		require(dripsHash == 0, "Drips history entry with hash and receivers");
		dripsHash = _hashDrips(drips.receivers);
	}
	historyHashes[i] = historyHash;
	historyHash = _hashDripsHistory(historyHash, dripsHash, drips.updateTime, drips.maxEnd);
}
```

As a result of having to iterate over the entire history, if the history exceeds a certain size, the total gas cost of squeezing that drip will exceed the block gas limit, making it impossible to ever squeeze from that drip. This means that all tokens from that drip would be locked in the contract until the drips are complete.


#### Signature replay attacks possible if deployed on multiple chains

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2023-01-drips/blob/9fd776b50f4be23ca038b1d0426e63a69c7a511d/src/Caller.sol#L164

##### Description:

`Caller.callSigned` operates using an EIP-712 signature which verifies the signed data to be used in a call on behalf of the signer. The problem with this method lies in the fact that it doesn't specify the chain ID, and thus if the contract is ever deployed to multiple chains, it will be possible to replay a signature on a different chain.


#### Gas Optimizations

##### Redundant require statement
```js
require(prevUserId != userId, "Duplicate splits receivers");
require(prevUserId < userId, "Splits receivers not sorted by user ID");
```

https://github.com/code-423n4/2023-01-drips/blob/9fd776b50f4be23ca038b1d0426e63a69c7a511d/src/Splits.sol#L238

##### Short circuit no receivable drips
In the case that there are no drips to be received in a call to receiveDrips, the function fully executes with no state changes, including an event emitted with an amount of 0. This logic is wasteful and redundant and instead it may make more sense to revert or at least immediately return 0 in `_receiveDrips` if a revert is unwanted.

https://github.com/code-423n4/2023-01-drips/blob/9fd776b50f4be23ca038b1d0426e63a69c7a511d/src/DripsHub.sol#L238

##### Use external function visibility over public when relevant
Often functions are marked as public when they are not being called from within the contract. Explicitly labeling external functions forces the function parameter storage location to be set as calldata, which saves gas each time the function is executed.

##### Tokens mistakenly sent will be permanently locked
The drips contracts use internal acounting rather than checking the actual token balance of the contracts, as a result, if the token balance exceeds the internal accounting balance, all excess tokens are unrecoverable