# 🔥 Ambire Wallet 🔥

_The first DeFi wallet that combines power, security and ease of use, while also being open-source and non-custodial._

![Ambire Wallet](/marketing-assets/ambire.png)

# ✨ So you want to sponsor a contest

This `README.md` contains a set of checklists for our contest collaboration.

Your contest will use two repos: 
- **a _contest_ repo** (this one), which is used for scoping your contest and for providing information to contestants (wardens)
- **a _findings_ repo**, where issues are submitted. 

Ultimately, when we launch the contest, this contest repo will be made public and will contain the smart contracts to be reviewed and all the information needed for contest participants. The findings repo will be made public after the contest is over and your team has mitigated the identified issues.

Some of the checklists in this doc are for **C4 (🐺)** and some of them are for **you as the contest sponsor (⭐️)**.

---

# ⭐️ Sponsor: Provide marketing details

- [ ] Your logo (URL or add file to this repo - SVG or other vector format preferred)
- [ ] Your primary Twitter handle
- [ ] Any other Twitter handles we can/should tag in (e.g. organizers' personal accounts, etc.)
- [ ] Your Discord URI
- [ ] Your website
- [ ] Optional: Do you have any quirks, recurring themes, iconic tweets, community "secret handshake" stuff we could work in? How do your people recognize each other, for example? 
- [ ] Optional: your logo in Discord emoji format

---

# Contest prep

## ⭐️ Sponsor: Contest prep
- [ ] Make sure your code is thoroughly commented using the [NatSpec format](https://docs.soliditylang.org/en/v0.5.10/natspec-format.html#natspec-format).
- [ ] Modify the bottom of this `README.md` file to describe how your code is supposed to work with links to any relevent documentation and any other criteria/details that the C4 Wardens should keep in mind when reviewing. ([Here's a well-constructed example.](https://github.com/code-423n4/2021-06-gro/blob/main/README.md))
- [ ] Please have final versions of contracts and documentation added/updated in this repo **no less than 8 hours prior to contest start time.**
- [ ] Ensure that you have access to the _findings_ repo where issues will be submitted.
- [ ] Promote the contest on Twitter (optional: tag in relevant protocols, etc.)
- [ ] Share it with your own communities (blog, Discord, Telegram, email newsletters, etc.)
- [ ] Optional: pre-record a high-level overview of your protocol (not just specific smart contract functions). This saves wardens a lot of time wading through documentation.
- [ ] Designate someone (or a team of people) to monitor DMs & questions in the C4 Discord (**#questions** channel) daily (Note: please *don't* discuss issues submitted by wardens in an open channel, as this could give hints to other wardens.)
- [ ] Delete this checklist and all text above the line below when you're ready.

---

## Contest scope

All the contracts in `contracts/`, namely `Identity.sol`, `libs/SignatureValidatorV2.sol`, `libs/BytesLib.sol`, `IdentityFactory.sol`, `wallet/QuickAccManager.sol`, `wallet/Zapper.sol` - that's a total of 754 lines.

## Architecture

Ambire is a smart wallet. Each user is represented by a smart contract, which is a minimal proxy (EIP 1167) for `Identity.sol` ([example](https://polygonscan.com/address/0x7ce38c302924f4b84a2c3a158df7ca9a5b7d1e1e#code)) - we call "account". Many addresses can control each account - we call this "privileges" in the contract and "authorities" in the UI.

The main contract everything is centered around is `Identity.sol`, which is the actual smart wallet.

Accounts can execute multiple calls in the same on-chain transaction. We call the array of user transactions a "user bundle" - the user signs the hash of this array along with anti-replay data such as nonce, chainID and others. Once it's signed, anyone can execute it by calling `Identity(account).execute`

The addresses that control an account (privileges) can be EOAs but they can also be smart contracts themselves, thanks to the `SmartWallet` signature mode in `SignatureValidatorV2` which enables EIP 1271 signatures to be used.

To allow more sophisticated authentication schemes without upgradability, we use a very simple relationship: a periphery contract that only deals with the specific authentication scheme can be added to `privileges`. For example, if a user wants to convert their account to a multisig, they can remove all other privileges and only authorize a single one: a multisig manager contract, that will verify N/M signatures and call `Identity(account).executeBySender` upon successful verification. This also works for EIP 1271 signatures since `Identity.isValidSignature` uses `SignatureValidatorV2`, which supports EIP 1271 itself, so it will propagate the call down to the multisig manager contract.

This very system is used by `QuickAccManager`, which is a simple 2/2 multisig, that also allows 1/2 transactions but with a timelock. This is used to allow for simple email/password login that can be upgraded by either backing up the second key or by moving to a hardware wallet. For more info on this authentication scheme please read the [security model Gist](https://gist.github.com/Ivshti/fe86f13c3adff3404a1f5ce1e364304c).

There are two ways for a user bundle to get executed:
* Directly, when a user's EOA pays for gas
* Through a Relayer that takes the signed message that authorizes a user bundle, and broadcasts it itself, paying for gas. The user bundle will have to contain an ERC20 transaction that pays the Relayer to reimburse it for gas. Currently we have a proprietary relayer that does all of this.

Because user bundles are authorized as signed messages, there's no need for hardware wallets to support EIP 1559 directly.

Similar products include Argent, Gnosis Safe and Authereum. The most notable differences is that the Ambire contracts are designed to be as simple as possible, and prefer composability to upgradability and built-in modularity.

### Testing and JS libs

The contracts in scope can also be found in this repo: https://github.com/AmbireTech/adex-protocol-eth/tree/identity-v5.2, specifically the identity-v5.2 branch (NOTE: we only care about the contracts in scope!).

In the repo, there are also tests that can be ran, namely `test/TestIdentity.js`. Other pieces of code you need to know about are `js/Bundle.js`, responsible for preparing and signing user bundles, and `js/IdentityProxyDeploy.js`, responsible for deploying the minimal proxies.


**NOTE**: The UI is currently in private beta, but you can use the [factory contract](https://polygonscan.com/address/0x447f228e6af15c2df147235ecb9abe53bd1f46ad#code) and [Identity.sol](https://polygonscan.com/address/0xa2e9e41ee85ae792a8213736c7f9398a933f8184) to experiment on Polygon mainnet.

### Design decisions
The contracts are free of inheritance and external dependencies.

There is no code upgradability and no ownership (`onlyOwner`) or pausability.

Storage usage is cut down to the minimum: when bigger data structures need to be saved, we take advantage of the cheap calldata and always pass them in, verifying the hash against a storage slot in the process, for example `QuickAccManager` uses this for quick accounts.

## Smart contract summary

### Identity.sol
The core of the Ambire smart wallet. Each user is a minimal proxy with this contract as a base. It contains very few methods, with the most notable being:
* `execute`: executes a signed user bundle
* `executeBySender`: executes a bundle as long as `msg.sender` is authorized

It's only dependency is an internal one, `SignatureValidatorV2`.

### SignatureValidatorV2.sol
Validates signatures in a few modes: EIP 712, EthSign, SmartWallet and Spoof. The first two verify signed messages using `ecrecover`, the only difference being that EthSign expects the "Ethereum signed message:" prefix. SmartWallet is for ERC 1271 signatures (smart contract signatures), and Spoof is for spoofed signatures that only work when `tx.origin == address(1)`.

### IdentityFactory.sol
A simple CREATE2 factory contract designed to deploy minimal proxies for users.

### wallet/QuickAccManager.sol

### wallet/Zapper.sol

## Known tradeoffs

**NOTE**: "bundle"/"user bundle" in this context means array of Identity-level transactions (`Identity.Transaction[]`)

* **QuickAccManager security model**: QuickAccManager allows users to control their wallets through a 2/2 multisig (see [security model](https://gist.github.com/Ivshti/fe86f13c3adff3404a1f5ce1e364304c)), with one of the keys in their own custody and the other key on the Ambire Relayer, with a possibility of the user backing it up. Timelocked transactions can be sent or cancelled by only 1/2 keys. This means that if the Ambire key is compromised AND lost, the attacker can cause grief by cancelling every attempt of the user to recover their funds. This can be avoided if the user backs up their key, which we recommend anyway for guaranteed full custody.
* **Storing additional data in `privileges`:** instead of boolean values, we use `bytes32` for the `privileges` mapping and treat any nonzero value as `true`. This is because we utilize the storage space for periphery contracts such as `QuickAccManager` or a planned `MultiSigManager` in the future. Utilizing a storage slot has the same gas costs no matter if `true` or hash is stored.
* **ERC20 fees taken through the transaction batch:** there's no special mechanism for reimbursing the relayer for the gas fee. Instead, the relayer looks at the bundle (`Transactions[]`) and sees if one or more of those transactions are ERC20 `transfer`s that send tokens to it. The relayer is responsible for checking whether the fee token and amount is acceptable for it, as well as checking it the transaction will execute before broadcasting it to the mempool. This is also a tradeoff cause the internal transactions may fail, in which case the whole bundle reverts and the fee is not paid, but the relayer will pay for gas. This is worked around on the Relayer end by utilizing Flashbots and Eden to avoid mining failing transactions, and by simulating the transactions right before trying to mine them. The reason we don't try/catch the errors int he `Identity` is because we want user bundles to succeed/fail as a whole (atomically), and the transaction to show as failing on Etherscan.
* **Signature spoof mode:** the `SignatureValidatorV2.sol` contract has a mode which allows signatures to be spoofed. The purpose of this is to allow easier simulation through `eth_call` and `eth_estimateGas` before having a signature from the user, since without this we would have a cyclical dependency that takes two steps to resolve (fee is unknown, user signs once to estimate the fee, then user signs a second time cause the bundle changed). This spoofing should not be allowed when calling through anywhere else other than `Identity(account).execute`, and it only works if `tx.origin == address(1)`.
* **Zapper approvals:** The `Zapper` contract does not require any ERC20 approvals, because we utilize the fact that users can batch transactions to transfer tokens to it and then do the trades in oen transaction, atomically. This saves some gas. This also means there are no `transferFrom` calls in the `Zapper`, as it just assumes it should have the tokens sent to it beforehand.

# Ambire contest details
- $23,750 USDC main award pot
- $1,250 USDC gas optimization award pot
- Join [C4 Discord](https://discord.gg/EY5dvm3evD) to register
- Submit findings [using the C4 form](https://code423n4.com/2021-10-Ambire-contest/submit)
- [Read our guidelines for more details](https://docs.code4rena.com/roles/wardens)
- Starts October 15, 2021 00:00 UTC
- Ends October 17, 2021 23:59 UTC

This repo will be made public before the start of the contest. (C4 delete this line when made public)

[ ⭐️ SPONSORS ADD INFO HERE ]
