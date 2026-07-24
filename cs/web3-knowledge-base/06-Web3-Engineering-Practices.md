# Web3 Engineering Practices

Language: English | [中文](../Web3知识库/06-Web3工程实践.md)

> A production DApp is not just a contract plus a wallet button. It is a chain of contracts, deployment scripts, tests, wallets, RPC providers, indexers, storage, monitoring, upgrades, and incident response.

---

## 1. Full DApp Delivery Flow

### Concept

Web3 engineering connects on-chain and off-chain components.

```text
product requirement
  -> Solidity or Rust contract design
  -> local tests, fuzzing, invariant tests, static analysis
  -> deployment scripts and verification
  -> frontend wallet connection
  -> contract reads and writes through RPC
  -> event indexing through The Graph or custom indexer
  -> metadata or assets on IPFS or similar storage
  -> monitoring, upgrades, incident response
```

### Mechanism

Each layer has a different failure mode:

- contract bug can lose funds;
- deployment bug can initialize wrong parameters;
- wallet bug can sign wrong chain or wrong payload;
- RPC issue can show stale data;
- indexer lag can confuse users;
- IPFS gateway issue can break metadata display;
- upgrade governance can become a centralization risk.

### Production practice

Design the system with explicit boundaries: contracts for settlement and authority, frontend for UX, indexer for queries, storage for metadata, monitoring for operations, and governance for controlled change.

### Interview self-check

Can you explain why a DApp still needs serious off-chain engineering?

---

## 2. Hardhat

### Concept

Hardhat is a JavaScript/TypeScript-based Ethereum development environment for compiling, testing, scripting, deploying, and verifying contracts.

### Mechanism

Typical project structure:

```text
contracts/
scripts/
test/
hardhat.config.ts
package.json
```

Common tasks:

- compile Solidity contracts;
- run unit and integration tests;
- fork mainnet for realistic testing;
- deploy to testnet or mainnet;
- verify contracts on block explorers.

### Trade-off and risk

Hardhat has excellent ecosystem integration, but JavaScript tests can drift from Solidity-level invariants if not designed carefully.

### Production practice

Use deterministic deployment scripts, environment variable validation, fork tests for protocol integrations, gas reports, coverage reports, and explicit network configuration.

### Interview self-check

Can you explain when you would choose Hardhat over Foundry?

---

## 3. Foundry

### Concept

Foundry is a fast Solidity-native development toolkit. Tests and deployment scripts can be written in Solidity.

### Mechanism

Main tools:

- `forge`: build, test, fuzz, script;
- `cast`: command-line Ethereum RPC interactions;
- `anvil`: local Ethereum node.

Foundry supports fuzz testing and invariant testing natively.

### Trade-off and risk

Foundry is extremely fast and strong for contract-heavy teams, but teams with large TypeScript integration flows may still prefer Hardhat or a hybrid setup.

### Production practice

Use fuzz tests for input ranges, invariant tests for protocol-level properties, fork tests for DeFi integrations, and gas snapshots for regressions.

### Interview self-check

Can you explain the difference between unit tests, fuzz tests, and invariant tests?

---

## 4. ethers.js and viem

### Concept

Frontend and backend services interact with Ethereum through RPC clients and contract ABIs.

### Mechanism

`ethers.js` provides providers, signers, contract objects, utilities, and transaction handling.

`viem` provides TypeScript-first, function-oriented clients with strong typing and good integration with wagmi.

Common operations:

- read ETH or token balance;
- call read-only contract methods;
- estimate gas;
- send transactions;
- wait for receipts;
- decode logs;
- handle revert errors.

### Trade-off and risk

Reads are usually simple RPC calls. Writes require wallet approval, gas estimation, chain ID correctness, transaction monitoring, and failure handling.

### Production practice

Use typed ABIs, simulate transactions before sending, handle chain/account changes, parse custom errors, and expose clear pending/confirmed/failed states to users.

### Interview self-check

Can you explain the difference between a provider/public client and a signer/wallet client?

---

## 5. Wallet Integration

### Concept

Wallet integration lets users authenticate with keys and authorize transactions or messages.

### Mechanism

A frontend commonly uses injected wallets, WalletConnect, Web3Modal, wagmi, and viem.

Important events:

- account changed;
- chain changed;
- disconnect;
- user rejected request;
- transaction pending, confirmed, or reverted.

### Trade-off and risk

Wallet UX is security-sensitive. Users can sign on the wrong chain, approve malicious spenders, or misunderstand typed data.

### Production practice

Show chain, address, contract, method, amount, spender, and risk clearly. Never ask users to paste private keys. Use EIP-712 typed data for readable signatures when possible.

### Interview self-check

Can you explain how you would safely handle account and chain changes in a React DApp?

---

## 6. Indexing With The Graph or Custom Indexers

### Concept

Raw RPC is not a database query engine. Indexers transform chain events and state into queryable application data.

### Mechanism

A subgraph defines:

- data sources and contract ABIs;
- event handlers;
- entity schema;
- mapping logic;
- GraphQL query interface.

Custom indexers may use event streams, archive nodes, Kafka, Postgres, ClickHouse, or other backend infrastructure.

### Trade-off and risk

Indexers can lag, reorg, fail, or compute derived state incorrectly. They improve UX but should not become hidden sources of truth for settlement-critical decisions.

### Production practice

Design reliable events, handle reorgs, make indexer lag visible, backfill safely, and keep critical authorization decisions on-chain.

### Interview self-check

Can you explain why an event-driven indexer is often better than querying contract state directly for UI lists?

---

## 7. IPFS and Decentralized Storage

### Concept

IPFS uses content addressing. The content identifier is derived from the content hash, so changing content changes the address.

### Mechanism

NFT metadata and images are often stored off-chain and referenced by `ipfs://` URIs. Pinning services keep content available.

### Trade-off and risk

IPFS gives content integrity but not automatic persistence. If nobody pins the content, it may become unavailable. Gateways can also fail or censor.

### Production practice

Pin critical content with multiple providers, store metadata immutably after reveal when required, use content hashes, and avoid relying on a single HTTP gateway.

### Interview self-check

Can you explain the difference between content integrity and content availability?

---

## 8. Testing and Auditing

### Concept

Smart contract testing must cover more than happy-path unit tests. Protocols need properties and invariants.

### Mechanism

Testing layers:

- unit tests: function-level behavior;
- integration tests: multi-contract flows;
- fork tests: realistic mainnet dependencies;
- fuzz tests: random inputs;
- invariant tests: properties that should always hold;
- static analysis: pattern detection;
- symbolic execution: path exploration;
- manual audit: human threat modeling.

### Trade-off and risk

No tool proves complete safety for arbitrary complex contracts. Each method catches different classes of bugs.

### Production practice

Use Slither for static analysis, Foundry/Echidna for fuzzing and invariants, fork tests for integrations, manual review for business logic, and external audits for high-value systems.

### Interview self-check

Can you name a bug class that unit tests might miss but invariant testing can catch?

---

## 9. Deployment and Upgrade Strategy

### Concept

Deployment is a security-critical operation. Upgradeability is an architectural decision, not a convenience switch.

### Mechanism

Deployment concerns:

- correct network and chain ID;
- deployer balance and nonce;
- constructor or initializer arguments;
- ownership transfer;
- contract verification;
- address registry updates;
- post-deploy smoke tests.

Upgrade patterns:

- Transparent Proxy;
- UUPS Proxy;
- Beacon Proxy;
- immutable contracts with migration.

### Trade-off and risk

Upgradeable contracts reduce bug permanence but introduce admin trust, storage layout risk, and governance complexity. Immutable contracts reduce admin risk but make bugs harder to fix.

### Production practice

Use multisig, timelock, upgrade validation, dry runs, deployment checklists, runbooks, event monitoring, and clearly documented rollback or mitigation options.

### Interview self-check

Can you explain how you would prevent storage layout corruption during an upgrade?

---

## 10. Web3 Project Architecture

### Concept

A typical DApp architecture includes frontend, wallet layer, RPC provider, smart contracts, indexer, decentralized storage, optional backend services, and monitoring.

### Mechanism

```text
frontend
  -> wagmi / viem / wallet connector
  -> RPC provider
  -> contracts and events
  -> indexer / subgraph
  -> database or cache for derived data
  -> IPFS gateway for metadata
  -> observability and alerting
```

### Trade-off and risk

Even decentralized applications often rely on centralized frontends, RPC providers, indexers, DNS, gateways, and admin keys. These components must be acknowledged instead of hidden.

### Production practice

Use multiple RPC providers, health checks, cache invalidation, reorg handling, circuit breakers, monitoring tools such as Tenderly or Forta where appropriate, and incident response procedures.

### Interview self-check

Can you list which parts of your DApp are decentralized and which are still centralized operational dependencies?

---

## 11. Interview Self-Check

1. What are the main steps from contract development to mainnet deployment?
2. How do Hardhat and Foundry differ?
3. What is fuzz testing, and why is it useful for contracts?
4. What is the difference between ethers.js and viem?
5. How should a frontend handle wallet account and chain changes?
6. Why is The Graph or an indexer needed?
7. What does IPFS guarantee, and what does it not guarantee?
8. What is the difference between UUPS and Transparent Proxy?
9. What checks belong in a deployment checklist?
10. How would you monitor a production DeFi or NFT protocol after deployment?
