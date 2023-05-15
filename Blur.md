# Blur Codearena Report

## Metadata

- Repo: https://github.com/code-423n4/2022-10-blur
- Contest page: https://code4rena.com/contests/2022-10-blur-exchange-contest

## Security Findings

#### ETH mistakenly sent with WETH buy orders will be permanently lost

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/BlurExchange.sol#L130

##### Description:

`BlurExchange.execute` is marked as payable, and orders can either be processed with ETH or WETH. The likelihood that a user picks an order denominated in WETH and transfers the ETH amount along with the order is non-negligible. In the case that a user does make this mistake, the ETH will become permanently locked in the contract.

##### Remediation:

It is recommended that in `BlurExchange._executeFundsTransfer`, a `require` is included to enforce that `msg.value == 0` in the case that the order is not denominated in ETH.

#### Malicious owner can set a new `ExecutionDelegate` contract and steal ETH from incoming buy orders without transferring any ERC721/ERC1155 tokens

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/BlurExchange.sol#L215

##### Description:

`ExecutionDelegate` is responsible for handling transfer of tokens via the `transferERC721/1155/20` methods. Since the contract owner can set the `executionDelegate` address, it is possible that they create a new contract that includes the same methods but simply executes no code, allowing the execution to succeed, transferring ETH to the recipient without any ERC721/1155 tokens being transferred.

Consider for example a scenario where the owner lists an expensive NFT, waits for a buy order, then frontruns the buy order by updating the executionDelegate contract. The trade is executed, with ETH being transferred from the buyer to the owner without transferring the listed NFT.

#### `BlurExchange.initialize` can be frontrun

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/BlurExchange.sol#L95

##### Description:

`BlurExchange.initialize` lacks access control, and can thus be executed by anyone, taking ownership of the protocol.

##### Remediation:

It is recommended that the `initialize` method includes access control such that it can only be executed by intended parties.

#### WETH address lacks `address(0)` check

##### Severity: Low

##### Context:

- https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/BlurExchange.sol#L113

##### Description:

In `BlurExchange.initialize`, the `weth` address variable is set. This variable is effectively immutable since there are no methods to update it. Since there are no checks to ensure that the address isn't being set to `address(0)`, user error could lead to it being incorrectly set, requiring a costly amount of gas to be spent to redeploy the contract or potentially causing unexpected effects if gone unnoticed.

##### Remediation:

It is recommended that a `require` is added to the `initialize` method to ensure that `_weth != address(0)`.
