# Web3 Knowledge Base

Language: English | [中文](../Web3知识库/README.md)

> This knowledge base is the English interview-preparation version of `cs/Web3知识库`. It keeps the same document numbering and topic coverage, but reorganizes the material around how senior Web3 engineers explain systems in English interviews: concept -> mechanism -> trade-off/security risk -> production practice -> interview self-check.

---

## Purpose

Use this folder to prepare for English technical interviews at global Web3 companies, protocol teams, exchange infrastructure teams, security firms, or blockchain-focused backend/platform teams.

The goal is not to memorize protocol names. The goal is to explain:

- why blockchains trade performance for verifiability and trust minimization;
- how transactions, accounts, consensus, execution, and data availability connect;
- how Ethereum smart contracts behave differently from ordinary backend services;
- why DeFi risk is a mix of code risk, mechanism risk, oracle risk, governance risk, and economic incentives;
- how Layer 2 systems and bridges inherit or modify trust assumptions;
- how production DApps are built, tested, deployed, indexed, upgraded, and monitored.

---

## Recommended Reading Paths

### Path A: Build the full Web3 mental model

1. [00-Web3 Overview and Mental Model](./00-Web3-Overview-and-Mental-Model.md)
2. [01-Blockchain Core Principles](./01-Blockchain-Core-Principles.md)
3. [02-Ethereum and Smart Contracts](./02-Ethereum-and-Smart-Contracts.md)
4. [06-Web3 Engineering Practices](./06-Web3-Engineering-Practices.md)
5. [04-DeFi and Application Protocols](./04-DeFi-and-Application-Protocols.md)
6. [05-Layer2 and Cross-Chain Technology](./05-Layer2-and-Cross-Chain-Technology.md)
7. [03-Rust Systems Programming](./03-Rust-Systems-Programming.md)

### Path B: Smart contract engineer track

1. [00-Web3 Overview and Mental Model](./00-Web3-Overview-and-Mental-Model.md)
2. [01-Blockchain Core Principles](./01-Blockchain-Core-Principles.md)
3. [02-Ethereum and Smart Contracts](./02-Ethereum-and-Smart-Contracts.md)
4. [06-Web3 Engineering Practices](./06-Web3-Engineering-Practices.md)
5. [04-DeFi and Application Protocols](./04-DeFi-and-Application-Protocols.md)

### Path C: Protocol research or infrastructure track

1. [00-Web3 Overview and Mental Model](./00-Web3-Overview-and-Mental-Model.md)
2. [01-Blockchain Core Principles](./01-Blockchain-Core-Principles.md)
3. [02-Ethereum and Smart Contracts](./02-Ethereum-and-Smart-Contracts.md)
4. [04-DeFi and Application Protocols](./04-DeFi-and-Application-Protocols.md)
5. [05-Layer2 and Cross-Chain Technology](./05-Layer2-and-Cross-Chain-Technology.md)
6. [03-Rust Systems Programming](./03-Rust-Systems-Programming.md)

---

## Difficulty Levels

### L1: Beginner mental model

You should be able to explain what a ledger, transaction, wallet, account, signature, gas, and smart contract are. Start with documents 00 and 01.

### L2: Execution and engineering

You should be able to explain EVM execution, storage vs memory, gas pricing, contract calls, upgrade patterns, test strategies, wallets, RPC, indexing, and deployment. Focus on documents 02 and 06.

### L3: Protocol design and security

You should be able to reason about AMMs, lending, liquidation, stablecoins, oracles, MEV, rollups, data availability, bridges, and Rust-based infrastructure. Focus on documents 03, 04, and 05.

---

## Document Map

| File | Chinese Source | Main Interview Value | Difficulty |
|------|----------------|----------------------|------------|
| [00-Web3-Overview-and-Mental-Model.md](./00-Web3-Overview-and-Mental-Model.md) | [00-Web3入门与全景认知.md](../Web3知识库/00-Web3入门与全景认知.md) | Build the minimum object graph: ledger, wallet, account, gas, contracts, protocols | L1 |
| [01-Blockchain-Core-Principles.md](./01-Blockchain-Core-Principles.md) | [01-区块链核心原理.md](../Web3知识库/01-区块链核心原理.md) | Explain blocks, transactions, consensus, cryptography, Merkle proofs, ZK basics, finality | L1-L2 |
| [02-Ethereum-and-Smart-Contracts.md](./02-Ethereum-and-Smart-Contracts.md) | [02-以太坊与智能合约.md](../Web3知识库/02-以太坊与智能合约.md) | Explain EVM, gas, storage, Solidity, ERC standards, calls, upgrades, security | L2 |
| [03-Rust-Systems-Programming.md](./03-Rust-Systems-Programming.md) | [03-Rust系统编程.md](../Web3知识库/03-Rust系统编程.md) | Explain ownership, lifetimes, traits, concurrency, async, errors for blockchain infrastructure roles | L2-L3 |
| [04-DeFi-and-Application-Protocols.md](./04-DeFi-and-Application-Protocols.md) | [04-DeFi与应用协议.md](../Web3知识库/04-DeFi与应用协议.md) | Explain AMMs, LP risk, lending, stablecoins, oracles, flash loans, MEV, DeFi security | L2-L3 |
| [05-Layer2-and-Cross-Chain-Technology.md](./05-Layer2-and-Cross-Chain-Technology.md) | [05-Layer2与跨链技术.md](../Web3知识库/05-Layer2与跨链技术.md) | Explain rollups, fraud proofs, validity proofs, DA, blobs, modular chains, bridges | L3 |
| [06-Web3-Engineering-Practices.md](./06-Web3-Engineering-Practices.md) | [06-Web3工程实践.md](../Web3知识库/06-Web3工程实践.md) | Explain DApp delivery: Hardhat, Foundry, wallets, viem, indexing, IPFS, testing, deployment | L2 |

---

## How To Answer Web3 Interview Questions In English

Use a layered answer instead of jumping to a definition.

1. Start with the concept: "A rollup is an execution environment that processes transactions off L1 while using L1 for settlement and data availability."
2. Explain the mechanism: "The sequencer orders transactions, executes them, posts batches to L1, and the L1 contract accepts state updates through fraud proofs or validity proofs."
3. State the trade-off or risk: "The main risks are sequencer centralization, data availability, bridge security, and proof system complexity."
4. Add production practice: "In production I would monitor batch posting, bridge liquidity, forced withdrawals, sequencer downtime, and contract upgrade governance."
5. Close with a self-check: "The key question is what trust assumption changed compared with L1."

Useful sentence patterns:

- "The key trade-off is ..."
- "The security assumption is ..."
- "The failure mode is ..."
- "In production, I would validate this by ..."
- "Compared with a traditional backend, the main difference is ..."
- "The protocol is safe only if ..."

---

## Maintenance Notes

- Keep file numbering aligned with `cs/Web3知识库`.
- Add the language link at the top of every English document.
- Prefer timeless mechanism explanations over volatile market metrics such as TVL or daily TPS.
- When protocol facts change, update both the Chinese source and the English version if the source is affected.
- Keep global indexes out of this folder-level update unless the parent task explicitly coordinates them.
