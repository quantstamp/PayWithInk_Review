# Overview

This smart contract audit was prepared by [Quantstamp](https://www.quantstamp.com/), the protocol for securing smart contracts.

This security audit report follows a generic template. Future Quantstamp reports will follow a similar template and they will be fully generated by automated tools.

## Specification

Our understanding of the specification was based on the following documentation:
* [Whitepaper](https://paywithink.com/wp-content/uploads/2017/11/Ink_Protocol_Whitepaper_V4_Listia_Inc.pdf) dated November 28, 2017,
* [Functional Specification](https://paywithink.com/wp-content/uploads/2017/11/Ink-Functional-Spec.pdf), and
* [Anatomy of an Ink Protocol Transaction](https://medium.com/@PayWithInk/anatomy-of-an-ink-protocol-transaction-24fec7e316b8).

We also reviewed the instructions provided in the [README.md](https://github.com/listia/ink/blob/2c3e8ea6fe2027b6ef0013a5a93c292d030c611d/README.md) file in the github repository at the time of the audit.

## Methodology

The review was conducted during 2017-Dec-07 thru 2017-Dec-13 by the Quantstamp team, which included senior engineers Alex Murashkin, Kacper Bak, and Steven Stewart.

Their procedure can be summarized as follows:

1. Code review
    * Review of the specification
    * Manual review of code
    * Comparison to specification
2. Testing and automated analysis
    * Test coverage analysis
    * Symbolic execution (automated code path evaluation)
3. Best-practices review
4. Itemize recommendations

## Source Code

The following source code was reviewed during the audit.

| Repository                            | Commit                                                                                   |
|---------------------------------------|------------------------------------------------------------------------------------------|
| [ink](https://github.com/listia/ink)  | [2c3e8ea](https://github.com/listia/ink/commit/2c3e8ea6fe2027b6ef0013a5a93c292d030c611d) |

# Security Audit

Quantstamp's objective was to evaluate the Listia Pay with Ink code for security-related issues, code quality, and adherence to best-practices.

Possible issues include (but are not limited to):

* Transaction-ordering dependence
* Timestamp dependence
* Mishandled exceptions and call stack limits
* Unsafe external calls
* Integer overflow / underflow
* Number rounding errors
* Reentrancy and cross-function vulnerabilities
* Denial of service / logical oversights

# Test coverage

We evaluated the test coverage using truffle and solidity-coverage. The below notes outline the setup and steps that were performed.

## Setup

Testing setup:
* Truffle v4.0.1
* solidity-coverage v0.4.3
* a modified version of Oyente tool based on the [commit]( https://github.com/melonproject/oyente/commits/ece2c241517ff3e32060b53c596a7540b985282c).

## Steps

Steps taken to run the full test suite:

* Installed dependencies as described in the file [README.md](https://github.com/listia/ink/blob/2c3e8ea6fe2027b6ef0013a5a93c292d030c611d/README.md).
* Ran the entire test suite using `yarn run truffle test`. Encountered some test failures, which was consistent with the Pay with Ink authors' notes. In order to collect accurate test coverage information, we introduced the following fixes:
  * To fix the "out of gas" exception, we turned each module from the "test/Ink/" subfolder into runnable test fixtures by replacing the lines like `module.exports = (accounts) => {` with `contract("Ink", async (accounts) => {` and appending `);` to the end of each file. Removed the file `test/Ink.js`. Possible reasons for this change include Truffle being unable to handle large test fixtures.
  * To fix the error `Error: VM Exception while processing transaction: invalid opcode` when running the test `Agent #createAccount() creates a user contract`, commented out `onlyOperator` in `contracts/Agent.sol`. See the section *Recommendations* for details.
* Since the coverage tool does not support contracts that have references to interfaces, we turned `InkMediator.sol` and `InkPolicy.sol` into contracts by replacing the keyword `interface` with `contract`.
* Installed the `solidity-coverage` tool: `npm install --save-dev solidity-coverage`.
* Patched `yarn` with `yarn-bin-fix` to work around the yarn-specific issue with transitive dependencies.
* Added a configuration file `.solcover.js` to the project root:
  ```
  module.exports = {
    copyNodeModules: true
  }
  ```

* Ran the coverage tool: `./node_modules/.bin/solidity-coverage`
* To workaround limitations of the Oyente tool, line 3 of `Ink.sol` was prepended by `./` (`zeppelin-solidity` -> `./zeppelin-solidity`), and Zeppelin files were moved accordingly.

## Evaluation

The coverage result of the `Ink.sol` file:
```
95.88% Statements 186/194
83.75% Branches 134/160
94.12% Functions 48/51
95.72% Lines 179/187
```

We evaluated the coverage report and identified three classes of missing test coverage:
1. Most `require()` calls, such as, `require(_transaction.state == TransactionState.Initiated);`, do not have any tests covering cases when the underlying expression evaluates to `false`. We recommend adding tests to cover these edge cases.
2. The linking logic (lines `203-210`, `671-678`) is not covered. However, since the contract does not make any queries to created links, test coverage of these lines is not necessary.
3. The `revert()` calls (lines `375`, `399`) are not covered, however, we recommend adding tests for these edge cases as well.

We did not evaluate the coverage of `Agent.sol`, `Account.sol`, and `ThreeOwnable.sol` as they are outside the scope of the current audit request.

Symbolic execution (the Oyente tool) did not detect any vulnerabilities of types Parity Multisig Bug 2, Callstack Depth Attack, Transaction-Ordering Dependence (TOD), Timestamp Dependency, and Re-Entrancy Vulnerability. However, EVM code coverage was reported as `68.3%`, so the tool did not explore all the possible paths.

# Recommendations

## Emitting Events After Other Operations Succeeded

Events are used for logging information from contract execution. Often, external program listen to events to do off-chain computations, e.g., to present the status of the transaction in a buyer-facing application. We noticed that the function `_createTransaction()` emits the event `TransactionInitiated()` before calling the function `_transferFrom()`. The latter, however, may fail if the buyer attempts to transfer more tokens than they actually possess. Consequently, external tools may get confused. They will see that a transaction had been initiated, whereas it got reverted after emitting the event. We recommend emitting the event only after the token transfer succeeds. If external tools can deal with events emitted by reverted transactions, we recommend clearly documenting this assumption in the contract.

## Require Mediator to Differ from Buyer and Seller

According to the Whitepaper, mediator is a well known third party that helps to dispute a transaction between the buyer and the seller. Nowhere in the contract have we found a statement requiring that the mediator must be different from the buyer and the seller. If one of the parties is careless, another party may take advantage by establishing themselves as the mediator. This is a potential security vulnerability. Although the contract cannot ensure that the mediator is always legitimate, it can rule out these obvious cases.

## Respecting the Modifier `onlyOperator`

The test `Agent.js` was failing due to the call `createAccount()`. Similarly to other methods of the contract `Agent.sol`, `createAccount()` has the modifier `onlyOperator` (inherited from `ThreeOwnable`). The modifier requires that the caller belongs to the map `operators`. None of the constructors of `AgentMock`, `Agent`, nor `ThreeOwnable` updates the map `operators`. Consequently, the call throws an exception because the required statement in `onlyOperator` fails. In the current setup, call to any other method protected by `onlyOperator` will fail as well. We recommend updating the contract constructor(s) to correctly modify the map `operators`.

## Code Documentation

We noted that majority of the functions were self-explanatory. Although standard documentation tags (such as `@dev`, `@param`, and `@returns`) were missing, inline comments provided sufficient information to clarify the code. Assumptions and behavior of some functions, however, was not apparent from the code, and, in fact, contradicted the specification.

According to the Functional Specification users may link accounts to aggregate ratings. Linking must be mutual, i.e., accepted by the two linked accounts. Futhermore, the contract is supposed to store information about which accounts are linked. We found out that the code does not adhere to this specification, which could have security implications. In the code, account linking functions (`linkWith()`, `link()` and `_link()`) only validate parameters and emit events. First, they do not assure mutual linking. Second, they do not store information about which accounts are linked. Consequently, anyone could call `linkWith()` to merge their reputation with another account. The Pay with Ink team explained that account linking will be handled outside the contract.

We recommend clearly documenting implicit assumptions and possible interactions with external components of the system.

# Appendix

## File Signatures

Below are SHA256 file signatures of the relevant files reviewed in the audit.

```
$ shasum -a 256 ./contracts/* ./contracts/*/* ./contracts/*/*/*
f9b39269211312cc450f2a663148cc19215bab0a44189951fa03675c76baeb65  ./contracts/Account.sol
7a833e9906ce1f774f819c240b361605ef3259d756f6611b4bb1491c0d8df03c  ./contracts/Agent.sol
6673113f132352513c070c90b91567e2aeb0f109b579166c5454a82a6bbde9b6  ./contracts/Ink.sol
583a67642be6d2c52956d2f4e281740f2e0b9f8e5659c01285665401a2ae7c4e  ./contracts/InkMediator.sol
107717e544bc8fecf9898f247480cf211fa4652562a4f52ddf3972f1a729f5fc  ./contracts/InkPolicy.sol
5a91ab9832553d50a57f7e1637f00699a081141459d784952e0f254ee821e8fa  ./contracts/Migrations.sol
597874d23315b7262c0b7ab0f98c5c6c44860aef96854fa824bb2935d9901011  ./contracts/ThreeOwnable.sol
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  ./contracts/examples
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  ./contracts/mocks
4a229feca81bbfb5d4a01cd6f55fbae6d76cdb0b8fdd1e18bc06ac6159365935  ./contracts/examples/ExampleMediator.sol
201b7d52c48ea5eb62edfd9260cd2e01a41d2ed5b66472176b9d075cf37f94f8  ./contracts/examples/ExamplePolicy.sol
f7c288613317ca6cfb416fe0c49fcbe8b7124cf3b7860794979a417c415a85ab  ./contracts/mocks/AgentMock.sol
3f3d12ab82c0d5d42ff851f4417a6e3ed7dd57adf68f70b750b92632df720aee  ./contracts/mocks/InkMock.sol
c8d577af3878955e4fb85cfe34cf40aa0f599f9b32b2441c09676eb533059e84  ./contracts/mocks/MediatorFeeMock.sol
7b22d353b41dae13205e021c0ec33d89a0af16de5b70f9dddc3424d5d008a234  ./contracts/mocks/MediatorMock.sol
9faeb30c86724eed299faf28d0006b574a4f087cd62070653bd082e93f6d05df  ./contracts/mocks/PolicyMock.sol
73b12a81535ad3aa29424121a9bb0133048bfa4b47e69131b88adcfbe6127bc5  ./contracts/mocks/ThreeOwnableMock.sol

$ shasum -a 256 ./test/* ./test/*/*
4375b5417f17a1088efdde9685edc16b327020ef687998b202868269e533c82e  ./test/Agent.js
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  ./test/Ink
f6877756dc0a90adf05237a55e7b0f12d144e4787ad52b11a54e777eda010b53  ./test/Ink.js
7c8e176782f4b56ffa4c487865e304931d96481df081032c5aa0aecec0179779  ./test/ThreeOwnable.js
72c970292841cf68f0719ca10ee734b4313919759da0bbdd58a4b8575b47d5b9  ./test/Ink/acceptTransaction.js
e478a8c6fa9171429495a8a6afdfe79b21ee35248a5a5f04da1479f510edda86  ./test/Ink/authorize.js
085c39e79ebdb72013a4e94a6d08f3df34d4c8f6af6f1e32d3c955c0890e0bbe  ./test/Ink/authorizedBy.js
131729d9e624b778277d106cc7aee2999fb72a1c1dd44253d6496988acaa1e94  ./test/Ink/confirmTransaction.js
52be77309c06a896b7203758d3df98588c7683b6e1867fb212c12aeca4e99c7a  ./test/Ink/confirmTransactionAfterExpiry.js
807ae32e6fe1666a093bbdb31f835e97608b67cc6901dff9ce421bc47b2e1283  ./test/Ink/confirmTransactionByMediator.js
a81d7a07de1c101c371cb7c6e0fed4152b278931e2e0990ef066a09cbc697c76  ./test/Ink/createTransaction.js
1191da7df21fc4cd0e70367045a69f0392c93cc46c4dec18e29bb1903da73ae0  ./test/Ink/createTransactionForBuyer.js
f330cb1d19a0ee2788c8855346172ab63f3fbc52e372b2ddabb363b7a4528986  ./test/Ink/createTransactionForBuyerAndSeller.js
19aeed668c9b9623d42deb4be1c9fd7b79043923a198ae83ba5311e62684c335  ./test/Ink/deauthorize.js
04545ab0e4fd2cba753bde87748f28e28c5986798af3bf715f44d843587f31d8  ./test/Ink/disputeTransaction.js
903906921c5460f35d20dcd1697291b81b2d0b47fc83ac43a609e04614b9c302  ./test/Ink/escalateDisputeToMediator.js
d44c794d80c862a1e55a93aa2dff3a29bdf6ec84816e2df3c7791e498b27a80b  ./test/Ink/gasAnalysis.js
265791fb9fe044623d590494f57bd348671a57685070a9d914371581b1911c13  ./test/Ink/provideTransactionFeedback.js
562e1199186509d165ebdbf46416234fd6eb18daaf50bab8f93da3bfe1f7c78f  ./test/Ink/refundTransaction.js
f386d89bc3e5215000371f4c1a0104fe271fa12e0d69615660052bbd1fb1f72d  ./test/Ink/refundTransactionAfterExpiry.js
57490ecb5164bfe91d3df4b22b181547906b22df079c54015acd75a88b798e77  ./test/Ink/refundTransactionByMediator.js
99eaf53c0eb8ced939c37aafecb506eab972ba4ba65dfc133138540e91fc9fd2  ./test/Ink/revokeTransaction.js
d42c5347729235c694b0fa51ee89689bddfc93600f7b97a430b1b0c51930ca9d  ./test/Ink/settleTransaction.js
7a59ecf2b66f6c57b11192067bf299b7dc9cf5daf4d5310756902104a25234b7  ./test/Ink/settleTransactionByMediator.js
654ff0cd192c5eb35782b517fcdf5f38b655012ea4a052c0a2c353b4ecd5c862  ./test/Ink/utils.js

$ shasum -a 256 ./migrations/*
42c21b4229b39fd1cad164ed6d4c24168620e2f04a66521b2b2f2945e23b867d  ./migrations/1_initial_migration.js
02262647f9e4ae93fd8cde457d47e7349af9147d370f7eabc89557691994f61b  ./migrations/2_deploy_contracts.js
71392eff99a6cbcd3696fbdc20e9728c274361663c5dd1a1e45554f1106a8219  ./migrations/3_deploy_agent.js

```

# Disclosure

## Purpose of report

The scope of our review is limited to a review of Solidity code and only the source code we note as being within the scope of our review within this report. Cryptographic tokens are emergent technologies and carry with them high levels of technical risk and uncertainty. The Solidity language itself remains under development and is subject to unknown risks and flaws. The review does not extend to the compiler layer, or any other areas beyond Solidity that could present security risks.

The report is not an endorsement or indictment of any particular project or team, and the report does not guarantee the security of any particular project. This report does not consider, and should not be interpreted as considering or having any bearing on, the potential economics of a token, token sale or any other product, service or other asset.

No third party should rely on the reports in any way, including for the purpose of making any decisions to buy or sell any token, product, service or other asset. Specifically, for the avoidance of doubt, this report does not constitute investment advice, is not intended to be relied upon as investment advice, is not an endorsement of this project or team, and it is not a guarantee as to the absolute security of the project.

## Links to other websites
You may, through hypertext or other computer links, gain access to web sites operated by persons other than Quantstamp Technologies Inc. (QTI). Such hyperlinks are provided for your reference and convenience only, and are the exclusive responsibility of such web sites' owners. You agree that QTI are not responsible for the content or operation of such web sites, and that QTI shall have no liability to you or any other person or entity for the use of third-party web sites. Except as described below, a hyperlink from this web site to another web site does not imply or mean that QTI endorses the content on that web site or the operator or operations of that site. You are solely responsible for determining the extent to which you may use any content at any other web sites to which you link from the report. QTI assumes no responsibility for the use of third-party software on the website and shall have no liability whatsoever to any person or entity for the accuracy or completeness of any outcome generated by such software.

## Timeliness of content
The content contained in the report is current as of the date appearing on the report and is subject to change without notice, unless indicated otherwise by QTI; however, QTI does not guarantee or warrant the accuracy, timeliness, or completeness of any report you access using the internet or other means, and assumes no obligation to update any information following publication.
