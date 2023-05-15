Aragon Codearena Report

## Metadata

- Repo: https://github.com/code-423n4/2023-03-aragon

## Security

#### `execute` calls to non-existent contracts will fail silently, allowing execution to continue

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2023-03-aragon/blob/4db573870aa4e1f40a3381cdd4ec006222e471fe/packages/contracts/src/core/dao/DAO.sol#L186-L199

##### Description

###### Impact

Calls may fail silently during `DAO.execute` as low-level calls to non-existent contracts will return `success`.

###### Proof of Concept

We can see that in `DAO.sol`, we perform low-level calls over an array of `_actions` and revert according to an `_allowFailureMap`.

```
for (uint256 i = 0; i < _actions.length; ) {
	address to = _actions[i].to;
	(bool success, bytes memory response) = to.call{value: _actions[i].value}(
		_actions[i].data
	);

	if (!success) {
		// If the call failed and wasn't allowed in allowFailureMap, revert.
		if (!hasBit(_allowFailureMap, uint8(i))) {
			revert ActionFailed(i);
		}

		// If the call failed, but was allowed in allowFailureMap, store that
		// this specific action has actually failed.
		failureMap = flipBit(failureMap, uint8(i));
	}

	execResults[i] = response;

	unchecked {
		++i;
	}
}
```

The problem with this logic is that it falsely assumes that the call is successful if it returned `success`. This is however not necessarily the case with low-level calls.

As stated in the [solidity docs](https://docs.soliditylang.org/en/v0.8.15/control-structures.html?highlight=low%20level%20calls#external-function-calls):

"Due to the fact that the EVM considers a call to a non-existing contract to always succeed, Solidity uses the `extcodesize` opcode to check that the contract that is about to be called actually exists (it contains code) and causes an exception if it does not. This check is skipped if the return data will be decoded after the call and thus the ABI decoder will catch the case of a non-existing contract.

Note that this check is not performed in case of [low-level calls](https://docs.soliditylang.org/en/v0.8.15/units-and-global-variables.html#address-related) which operate on addresses rather than contract instances."

Therefore, the logic of catching failing calls with the `_allowFailureMap` won't necessarily catch all failing calls, leading to subsequent calls still being executed, potentially leading to an unwanted state. Consider for example the following scenario:

- The DAO wants to add collateral to their lending position and take out more debt so they create the parameters to perform this transaction
- They create the following `_actions` array:
	- Approve WETH to Aave
	- Deposit WETH on Aave
	- Withdraw USDC against WETH position on Aave
- They set the `_allowFailureMap` such that if either of the first two `_actions` fails, the whole execution should revert
- Under the false assumption that their `_allowFailureMap` will protect them in case if anything goes wrong, they aren't concerned with simulating the execution before calling `_execute`
- The address for the second transaction turns out to be incorrect, causing it to silently fail, allowing the third transaction to be executed
- Their debt position becomes insufficiently collateralized and is liquidated

###### Recommended Mitigation

To solve this issue, include a bitmap similar to `_allowFailureMap` which enforces which actions should first verify that the calling address is a contract.


#### Lack of `address(0)` check

##### Severity: Low

##### Context:

- https://github.com/code-423n4/2023-03-aragon/blob/4db573870aa4e1f40a3381cdd4ec006222e471fe/packages/contracts/src/core/dao/DAO.sol#L118

##### Description

On deployment of a DAO contract, there exist no checks to ensure that the address being set as `_initalOwner`, which is granted `ROOT_PERMISSION_ID`, is not `address(0)`. As a result, it's possible to initially grant `ROOT_PERMISSION_ID` only to `address(0)`, thereby effectively bricking the entire DAO.


#### `merkleRoot` immutability can cause tokens to be permanently locked in `MerkleDistributor`

##### Severity: Low

##### Context: 

- https://github.com/code-423n4/2023-03-aragon/blob/4db573870aa4e1f40a3381cdd4ec006222e471fe/packages/contracts/src/plugins/token/MerkleMinter.sol#L90

##### Description

`MerkleDistributor` uses a `merkleRoot` to allow users to claim appropriate amounts of tokens. This value can only be set in the `initialize` method which can only be executed once. 

`MerkleMinter.merkleMint` creates a `MerkleDistributor` contract and immediately mints tokens to the new address. There exist no checks that the `_merkleRoot` being passed to the `initialize` method is valid, i.e. not `bytes32(0)`. As a result, it's possible that the tokens being sent to the distributor could be permanently locked in the contract.


#### Lack of ENS subdomain validation

##### Severity: Low

##### Context:

- https://github.com/code-423n4/2023-03-aragon/blob/4db573870aa4e1f40a3381cdd4ec006222e471fe/packages/contracts/src/framework/utils/RegistryUtils.sol#L13

##### Description

Before registering a DAO or plugin's subdomain, `isSubdomainValid` is called to validate the provided string. This method fails to validate the length of the subdomain being provided, allowing creation of subdomains with 0 characters and >255 characters, with the latter breaking DNS and [NameWrapper](https://github.com/ensdomains/ens-contracts/tree/master/contracts/wrapper) compatibility.