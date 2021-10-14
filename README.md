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

# Contest setup

## ⭐️ Sponsor: Provide contest details

Under "SPONSORS ADD INFO HERE" heading below, include the following:

- [ ] Name of each contract and:
  - [ ] lines of code in each
  - [ ] external contracts called in each
  - [ ] libraries used in each
- [ ] Describe any novel or unique curve logic or mathematical models implemented in the contracts
- [ ] Does the token conform to the ERC-20 standard? In what specific ways does it differ?
- [ ] Describe anything else that adds any special logic that makes your approach unique
- [ ] Identify any areas of specific concern in reviewing the code
- [ ] Add all of the code to this repo that you want reviewed
- [ ] Create a PR to this repo with the above changes.

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

All the contracts in `contracts/`, namely `Identity.sol`, `libs/SignatureValidatorV2.sol`, `IdentityFactory.sol`, `wallet/QuickAccManager.sol`, `wallet/Zapper.sol` - that's a total of 704 lines.

The same contracts can be found in this repo: https://github.com/AmbireTech/adex-protocol-eth/tree/identity-v5.2, specifically the identity-v5.2 branch. In the repo, there are also tests that can be ran, namely `test/TestIdentity.js`.

## Architecture

## Smart contract summary

### Identity.sol

### IdentityFactory.sol

### wallet/QuickAccManager.sol

### wallet/Zapper.sol

## Known tradeoffs

**NOTE**: "bundle"/"user bundle" in this context means array of Identity-level transactions (`Identity.Transaction[]`)

* **QuickAccManager security model**: QuickAccManager allows users to control their wallets through a 2/2 multisig (see [security model](https://gist.github.com/Ivshti/fe86f13c3adff3404a1f5ce1e364304c)), with one of the keys in their own custody and the other key on the Ambire Relayer, with a possibility of the user backing it up. Timelocked transactions can be sent or cancelled by only 1/2 keys. This means that if the Ambire key is compromised AND lost, the attacker can cause grief by cancelling every attempt of the user to recover their funds. This can be avoided if the user backs up their key, which we recommend anyway for guaranteed full custody.
* **Storing additional data in `privileges`:** instead of boolean values, we use `bytes32` for the `privileges` mapping and treat any nonzero value as `true`. This is because we utilize the storage space for periphery contracts such as `QuickAccManager` or a planned `MultiSigManager` in the future. Utilizing a storage slot has the same gas costs no matter if `true` or hash is stored.
* **ERC20 fees taken through the transaction batch:** there's no special mechanism for reimbursing the relayer for the gas fee. Instead, the relayer looks at the bundle (`Transactions[]`) and sees if one or more of those transactions are ERC20 `transfer`s that send tokens to it. The relayer is responsible for checking whether the fee token and amount is acceptable for it, as well as checking it the transaction will execute before broadcasting it to the mempool. This is also a tradeoff cause the internal transactions may fail, in which case the whole bundle reverts and the fee is not paid, but the relayer will pay for gas. This is worked around on the Relayer end by utilizing Flashbots and Eden to avoid mining failing transactions, and by simulating the transactions right before trying to mine them. The reason we don't try/catch the errors int he `Identity` is because we want user bundles to succeed/fail as a whole (atomically), and the transaction to show as failing on Etherscan.
* **Signature spoof mode:** the `SignatureValidatorV2.sol` contract has a mode which allows signatures to be spoofed. The purpose of this is to allow easier simulation through `eth_call` and `eth_estimateGas` before having a signature from the user, since without this we would have a cyclical dependency that takes two steps to resolve (fee is unknown, user signs once to estimate the fee, then user signs a second time cause the bundle changed). This spoofing should not be allowed when calling through anywhere else other than `Identity.execute`, and it only works if `tx.origin == address(1)`.
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
