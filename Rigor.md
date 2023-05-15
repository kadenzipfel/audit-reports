# Golom Codearena Report

## Metadata

- Repo: https://github.com/code-423n4/2022-08-rigor
- Contest page: https://code4rena.com/contests/2022-08-rigor-protocol-contest

## Security Findings

#### Signature replay griefing possible

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2022-08-rigor/blob/5ab7ea84a1516cb726421ef690af5bc41029f88f/contracts/Project.sol#L219
- https://github.com/code-423n4/2022-08-rigor/blob/5ab7ea84a1516cb726421ef690af5bc41029f88f/contracts/Project.sol#L386
- https://github.com/code-423n4/2022-08-rigor/blob/5ab7ea84a1516cb726421ef690af5bc41029f88f/contracts/Community.sol#L509

##### Description:

Methods requiring signed data which do not require that the signature has not already been used, e.g. `Project.addTasks`, `Project.changeOrder`, and `Community.escrow`; are vulnerable to signature replay attacks. For example, a malicious actor may take a past transaction on one of these methods and re-execute it by passing the same `_data` and `_signature`, causing unexpected effects.

##### Remediation:

It's recommended to require a nonce be passed with all signed data and that a check is performed to ensure that every individual data packet can only be executed once.

#### Rebasing and fee-on-transfer tokens will break accounting logic

##### Severity: Low

##### Context:

- https://github.com/code-423n4/2022-08-rigor/blob/5ab7ea84a1516cb726421ef690af5bc41029f88f/contracts/Community.sol#L757
- https://github.com/code-423n4/2022-08-rigor/blob/5ab7ea84a1516cb726421ef690af5bc41029f88f/contracts/Community.sol#L824

##### Description:

The accounting logic used throughout the protocol assumes that the balance of tokens will not change during the lifecycle. As such, it is important that tokens used in the protocol do not contain any rebasing, fee-on-transfer, or similar balance changing logic. Failure to use static balance tokens will result in unexpected effects including locked funds and failed transactions.

##### Remediation

It's recommended to take care in ensuring that tokens used in the protocol do not contain rebasing, fee-on-transfer, or similar balance changing logic.

## Gas Optimizations

#### Avoid decoding unused values

##### Severity: Gas Optimization

##### Context:

- https://github.com/code-423n4/2022-08-rigor/blob/5ab7ea84a1516cb726421ef690af5bc41029f88f/contracts/Project.sol#L505

##### Description:

In `Project.raiseDispute`, all values are decoded from `_data`, but only the first two values are used. The additional logic required to decode the last three values costs significant gas without any benefit.

##### Remediation:

Don't decode the final three values as they're not being used to reduce gas usage from 217449 to 216464.

#### Use `uint256` over smaller `uint` types when possible

##### Severity: Gas Optimization

##### Context:

- https://github.com/code-423n4/2022-08-rigor/blob/5ab7ea84a1516cb726421ef690af5bc41029f88f/contracts/Disputes.sol#L100
- https://github.com/code-423n4/2022-08-rigor/blob/5ab7ea84a1516cb726421ef690af5bc41029f88f/contracts/Disputes.sol#L103

##### Description:

In `Disputes.raiseDispute`, `_actionType` is decoded from `_data` and casted to a `uint8` type. This is unnecessary and inefficient as the necessary logic can be performed with a `uint256`, and when compiled, `_actionType` gets casted back to `uint256` costing additional gas in the process.

##### Remediation:

Decode `_actionType` as a `uint256` in `Disputes.raiseDispute` to reduce gas usage from 217449 to 217412.

#### Save storage variables locally when reused

##### Severity: Gas Optimization

##### Context:

- https://github.com/code-423n4/2022-08-rigor/blob/5ab7ea84a1516cb726421ef690af5bc41029f88f/contracts/Community.sol#L385
- https://github.com/code-423n4/2022-08-rigor/blob/5ab7ea84a1516cb726421ef690af5bc41029f88f/contracts/Project.sol#L190
- https://github.com/code-423n4/2022-08-rigor/blob/5ab7ea84a1516cb726421ef690af5bc41029f88f/contracts/Project.sol#L350
- https://github.com/code-423n4/2022-08-rigor/blob/5ab7ea84a1516cb726421ef690af5bc41029f88f/contracts/Project.sol#L409

##### Description:

Throughout `Community.sol` and `Project.sol`, there are storage variables which are reused within the same method. This should be generally avoided as it is much more efficient to save a variable locally and retrieve from the stack or memory instead of storage.

##### Remediation:

- In `Community.lendToProject`, save `_communities[_communityId]` to a local variable to reduce gas usage from 190993 to 190425.
- In `Project.lendToProject`, save `builder` to a local variable to reduce gas usage from 108069 to 107986
- In `Project.setComplete`, save `tasks[_taskId]` to a local variable to reduce gas usage from 93713 to 93635
- In `Project.changeOrder`, save `tasks[_taskId]` to a local variable to reduce gas usage from 75640 to 75459

#### Use optimal comparison operator

##### Severity: Gas Optimization

##### Context:

- Throughout the contracts

##### Description:

Throughout the contracts, suboptimal comparison operators are used.

##### Remediation:

Optimal comparison operators should be used as follows:

- When checking that uints are greater than 0, use `value != 0` instead of `value > 0` to save 44 gas
- When checking that a uint is less/greater than or equal to another, use, e.g. `value > otherValue - 1` instead of `value >= otherValue` to save 38 gas
