# Web3 Overview and Mental Model

Language: English | [中文](../Web3知识库/00-Web3入门与全景认知.md)

> This document builds the minimum mental model for Web3 before diving into smart contracts, DeFi, Layer 2, or tooling. The core relationship is: ledger -> transaction -> account -> gas -> contract -> protocol.

---

## 1. What Web3 Is Really About

### Concept

Web3 is not only wallets, tokens, exchanges, or Solidity templates. Technically, it is a stack of systems that lets participants who do not fully trust each other coordinate around shared state.

A useful four-layer model is:

- `ledger layer`: blocks, transactions, nodes, signatures, consensus, and state;
- `execution layer`: accounts, EVM, gas, smart contracts, and state transitions;
- `protocol layer`: DEXs, lending, stablecoins, oracles, governance, MEV, and bridges;
- `engineering layer`: frameworks, wallets, RPC, indexing, storage, deployment, monitoring, and upgrades.

### Mechanism

A user signs a transaction with a private key. The transaction is broadcast to a peer-to-peer network. Nodes validate it, block producers include it in a block, consensus decides the canonical history, and all honest nodes apply the same state transition rules.

```text
wallet signs transaction
  -> transaction enters the network
  -> nodes validate signature, nonce, balance, and rules
  -> block producer includes it
  -> consensus accepts a block
  -> account or contract state changes
```

### Trade-off and risk

Blockchains are slower and more expensive than centralized databases because they add public verification, replicated execution, adversarial consensus, and tamper-resistant history. This is not a bug in the normal sense. It is the cost of reducing reliance on a single trusted operator.

### Production practice

When designing a Web3 system, ask where each piece of state lives:

- on-chain state should be small, security-critical, and consensus-relevant;
- off-chain state should be used for indexing, UI, analytics, and non-critical metadata;
- signatures and events should create an auditable link between the two.

### Interview self-check

Can you explain why Web3 sacrifices throughput and operational convenience for verifiability, censorship resistance, and stronger ownership guarantees?

---

## 2. Why A Blockchain Is Not A Normal Database

### Concept

A centralized backend normally has an application server, a database, an administrator, and an operational team that can patch, migrate, or roll back data. A public blockchain replaces this trusted operational center with public rules, cryptography, economic incentives, and consensus.

### Mechanism

In a traditional system:

- one platform controls writes;
- users trust the operator to enforce rules;
- database state can be modified by privileged operators;
- incident recovery can rely on backups and manual intervention.

In a blockchain system:

- many nodes store or verify the ledger;
- users submit signed transactions;
- validators or miners propose blocks;
- consensus chooses the valid chain;
- state changes are deterministic and publicly verifiable.

### Trade-off and risk

The main trade-off is performance versus trust minimization. The system gains verifiability and stronger auditability, but loses cheap writes, easy rollbacks, and simple upgrades.

### Production practice

Do not put ordinary mutable application data on-chain just because it can be stored there. Use the chain for settlement, ownership, permissions, and critical protocol state; use off-chain systems for search, cache, analytics, notifications, and user experience.

### Interview self-check

If asked "why not just use PostgreSQL?", can you answer in terms of trust assumptions rather than hype?

---

## 3. Wallets, Private Keys, Addresses, and Accounts

### Concept

A wallet is not where assets physically live. Assets are state recorded on the blockchain. The wallet manages private keys and helps users sign transactions that modify chain state.

### Mechanism

- `Private key`: the secret that controls signing authority.
- `Address`: a public identifier derived from the public key.
- `Wallet`: software or hardware that manages keys and signs messages or transactions.
- `Account`: a chain-level entity with balance, nonce, and possibly code or storage.

In Ethereum there are two main account types:

- `EOA`: externally owned account controlled by a private key;
- `Contract account`: account controlled by deployed code.

Only EOAs can initiate transactions directly. Contract accounts execute when called.

### Trade-off and risk

Self-custody gives users stronger control, but key loss or key theft is catastrophic. Wallet UX is therefore part of protocol security, not only frontend convenience.

### Production practice

Production systems should support hardware wallets, clear signing messages, chain ID checks, nonce handling, transaction simulation, phishing-resistant UI patterns, and emergency recovery or multisig flows for admin roles.

### Interview self-check

Can you explain why "the wallet contains my coins" is technically inaccurate?

---

## 4. Gas As Resource Pricing

### Concept

Gas is not merely a transaction fee. It is the resource-metering system for blockchain execution.

### Mechanism

Every EVM operation has a gas cost. Users specify limits and fees. Validators are compensated for including transactions, while gas limits prevent infinite computation or denial-of-service execution.

Gas prices make contract design different from normal backend design:

- storage writes are expensive;
- loops over unbounded arrays are dangerous;
- repeated external calls can fail or become too costly;
- user-facing operations must remain economically reasonable.

### Trade-off and risk

Gas introduces a direct cost to every state transition. A contract can be logically correct but practically unusable if common operations are too expensive.

### Production practice

Use events for off-chain indexing, compact storage layouts, pagination, calldata where possible, custom errors, caching of storage reads, and careful upgrade storage layout checks.

### Interview self-check

Can you explain why gas is both a fee mechanism and a denial-of-service protection mechanism?

---

## 5. What Smart Contracts Are

### Concept

A smart contract is code deployed to a blockchain and executed by the network according to deterministic rules. It is closer to a public state transition function than to a private backend service.

### Mechanism

Users send transactions to contract functions. The EVM executes bytecode, reads and writes storage, emits logs, and may call other contracts. If execution reverts, state changes are rolled back.

### Trade-off and risk

Smart contracts are public, hard to patch, and often directly control assets. Security risk is not limited to code bugs. It also includes authorization mistakes, upgrade governance, oracle dependency, economic design, liquidity conditions, MEV, and composability with other protocols.

### Production practice

Use audited libraries, explicit permissions, tests and fuzzing, invariant checks, event design, multisig and timelocks for sensitive operations, emergency pause mechanisms only when justified, and independent reviews before mainnet deployment.

### Interview self-check

Can you explain why smart contract development is more about secure state transitions than just implementing features?

---

## 6. Common Web3 Applications

### Tokens

Tokens are standard representations of assets or rights. ERC-20 models fungible assets, ERC-721 models unique NFTs, and ERC-1155 supports multiple token types in one contract.

### DEXs

A decentralized exchange lets users swap assets through on-chain rules. AMMs use liquidity pools and formulas instead of centralized order books.

### Lending protocols

Lending protocols use collateral, debt accounting, interest models, liquidation thresholds, and oracle prices to maintain solvency.

### Stablecoins

Stablecoins try to maintain a price peg through fiat reserves, crypto collateral, algorithmic mechanisms, or a hybrid design.

### Layer 2

Layer 2 systems move execution away from Ethereum L1 while using L1 for settlement, proof verification, and often data availability.

### Bridges

Bridges move assets or messages across independent state machines. They are high-risk because they introduce extra verification and custody assumptions.

---

## 7. Why Web3 Security Is Hard

### Concept

Web3 security combines software security, distributed systems, cryptography, game theory, operational security, and financial risk.

### Mechanism

A protocol can fail through:

- code bugs such as reentrancy or access control mistakes;
- mechanism failures such as weak liquidation logic or unstable incentives;
- oracle manipulation;
- governance capture;
- bridge validator compromise;
- MEV extraction and transaction ordering attacks;
- frontend or wallet phishing.

### Production practice

Security work should include threat modeling, code review, static analysis, fuzzing, invariant testing, audits, bug bounties, monitoring, key management, and incident response.

### Interview self-check

Can you distinguish a contract bug from a protocol design failure and from an oracle or governance failure?

---

## 8. Beginner Pitfalls

- Starting with Solidity templates before understanding transactions, accounts, gas, and storage.
- Treating smart contracts like normal backend services.
- Ignoring private key and wallet security.
- Learning bridges or Layer 2 before understanding L1 consensus and data availability.
- Evaluating DeFi protocols only by TVL or APY instead of mechanism risk.
- Assuming "decentralized" means there are no centralized components.

---

## 9. Interview Self-Check

1. What problem does a blockchain solve that a normal database does not solve?
2. What happens from the moment a user signs a transaction until state changes?
3. What is the difference between a private key, address, wallet, EOA, and contract account?
4. Why does gas exist?
5. Why is smart contract security broader than source-code review?
6. Where should data live in a production DApp: on-chain, indexed off-chain, or decentralized storage?
