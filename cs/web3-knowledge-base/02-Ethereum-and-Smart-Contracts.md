# Ethereum and Smart Contracts

Language: English | [中文](../Web3知识库/02-以太坊与智能合约.md)

> Ethereum is a public, deterministic, resource-metered state machine. Smart contract interviews are mostly about execution constraints, gas, storage, calls, security, standards, and upgrade risk.

---

## 1. Ethereum Mental Model

### Concept

Ethereum extends the blockchain ledger model into a programmable state machine. Users submit signed transactions. Transactions modify account state. Smart contracts execute inside the EVM and persist state in storage.

```text
wallet
  -> EOA signs transaction
  -> transaction enters EVM execution
  -> contract code reads/writes memory, storage, logs
  -> gas is consumed
  -> state root changes
```

### Mechanism

Ethereum has two account types:

- `EOA`: controlled by a private key, can initiate transactions;
- `contract account`: controlled by bytecode, executes only when called.

Each account can have balance, nonce, code hash, and storage root. The global state root commits to all accounts.

### Trade-off and risk

Ethereum is expressive because contracts can call each other and maintain arbitrary state. That expressiveness creates attack surface: reentrancy, unsafe external calls, storage collisions, oracle dependency, MEV, access-control mistakes, and upgrade governance failures.

### Production practice

Model every function as a state transition with permissions, gas cost, external-call risk, and event output. If a function touches assets or critical roles, it deserves tests, fuzzing, review, and monitoring.

### Interview self-check

Can you explain why Ethereum is not just "Bitcoin plus scripts" but a public programmable state machine?

---

## 2. Ethereum Evolution

### Concept

Ethereum has evolved from proof of work to proof of stake and continues to optimize around rollup-centric scaling.

### Mechanism

Important milestones:

- The Merge: replaced PoW consensus with PoS while keeping the execution layer;
- Shanghai/Capella: enabled validator withdrawals;
- Dencun: introduced EIP-4844 blobs for cheaper rollup data availability;
- later upgrades continue to improve data availability, validator UX, and protocol scalability.

### Trade-off and risk

Ethereum prioritizes decentralization and security on L1 while pushing execution scalability to Layer 2. This improves modularity but makes the user experience depend on rollups, bridges, sequencers, and data availability.

### Interview self-check

Can you explain why Ethereum scaling is now rollup-centric?

---

## 3. EVM Execution Model

### Concept

The EVM is a stack-based virtual machine that executes deterministic bytecode. Determinism is critical because every node must reach the same result.

### Mechanism

Main EVM data areas:

- `Stack`: 1024-depth stack of 256-bit words, used for computation;
- `Memory`: temporary byte-addressed memory, cleared after execution;
- `Storage`: persistent key-value storage, expensive and part of contract state;
- `Calldata`: read-only transaction or call input data;
- `Logs`: event output, cheaper than storage and useful for indexing.

Common calls:

- `CALL`: executes callee code in callee context;
- `DELEGATECALL`: executes callee code in caller storage context;
- `STATICCALL`: read-only call that forbids state changes.

### Trade-off and risk

`DELEGATECALL` powers proxy upgrades, but it is dangerous because code from another contract writes into the caller's storage layout. A layout mismatch can corrupt state.

### Production practice

For upgradeable contracts, never reorder or change existing storage variables. Use OpenZeppelin upgrade validation, storage gaps where appropriate, and explicit upgrade authorization.

### Interview self-check

Can you explain CALL vs DELEGATECALL using `msg.sender`, `address(this)`, and storage context?

---

## 4. Ethereum State and Tries

### Concept

Ethereum commits to state through trie roots stored in block headers. This allows nodes and light clients to verify state, transactions, and receipts.

### Mechanism

Important roots:

- `stateRoot`: commits to global account state;
- `transactionsRoot`: commits to transactions in the block;
- `receiptsRoot`: commits to execution receipts and logs.

Ethereum uses a Modified Merkle Patricia Trie structure to combine authenticated data with key-value lookup.

### Trade-off and risk

State growth is a long-term scalability challenge. Every storage slot added by contracts increases the burden on full nodes.

### Production practice

Minimize persistent storage. Use events for historical records, external indexes for queries, and compact data structures for frequently accessed state.

### Interview self-check

Can you explain why storage is expensive beyond "because gas says so"?

---

## 5. Transactions and Gas

### Concept

A transaction is a signed state-transition request. Gas prices the resources consumed by that request.

### Mechanism

EIP-1559 transaction fees include:

- `baseFee`: protocol-computed, burned, changes with block demand;
- `priorityFee`: tip paid to the validator;
- `maxFeePerGas`: user's maximum willingness to pay;
- `gasLimit`: maximum gas the transaction can consume.

Actual fee is approximately `gasUsed * (baseFee + priorityFee)`.

### Trade-off and risk

Gas creates user cost, denial-of-service resistance, and design pressure. Contracts can fail if gas limits are too low, or become unusable if normal operations are too expensive.

### Production practice

Use gas snapshots, storage packing, custom errors, calldata parameters, bounded loops, and careful event design. Avoid unbounded iteration over user-controlled arrays.

### Interview self-check

Can you explain why the base fee is burned and the priority fee goes to validators?

---

## 6. Solidity Foundations

### Concept

Solidity is a contract-oriented language for writing EVM programs. The language syntax is easier than the execution model.

### Mechanism

Key concepts:

- value types: `uint256`, `bool`, `address`, `bytes32`;
- reference types: arrays, structs, mappings, strings;
- visibility: `public`, `external`, `internal`, `private`;
- mutability: `view`, `pure`, `payable`;
- errors: `require`, `revert`, custom errors, `assert` for invariants.

### Trade-off and risk

`private` does not mean secret on a public chain. It only limits Solidity-level access from other contracts. State is still visible to anyone reading chain data.

### Production practice

Use current compiler versions, explicit SPDX license identifiers, audited libraries, custom errors for gas efficiency, and NatSpec comments on public APIs and critical security assumptions.

### Interview self-check

Can you explain why `private` storage is still publicly observable?

---

## 7. Storage Layout and Gas Optimization

### Concept

Storage is organized into 32-byte slots. Layout determines both gas cost and upgrade safety.

### Mechanism

- fixed-size values are stored in slots;
- small adjacent values can be packed into one slot;
- dynamic arrays store length in one slot and elements at `keccak256(slot)`;
- mappings store values at `keccak256(key, slot)`;
- inheritance order affects layout.

### Trade-off and risk

Packing can save gas, but careless layout changes in upgradeable contracts can corrupt existing state.

### Production practice

Group small variables intentionally, cache repeated storage reads, avoid writing storage inside loops, use `calldata` for external read-only arrays, and validate upgrade storage compatibility.

### Interview self-check

Can you explain why adding a state variable in the middle of an upgradeable contract is dangerous?

---

## 8. Contract Design Patterns

### Proxy Upgrade Pattern

A proxy holds state and delegates calls to an implementation contract. Upgrading changes the implementation address while preserving proxy storage.

Risks:

- storage layout collision;
- unsafe upgrade authorization;
- uninitialized implementation;
- function selector conflicts;
- admin key compromise.

### Factory Pattern

A factory deploys many similar contracts and records their addresses. It improves consistency and discoverability.

Risks:

- unsafe constructor parameters;
- missing initialization;
- predictable address assumptions;
- dependency on factory permissions.

### Access Control

Common schemes include `Ownable`, role-based access control, multisig, and timelocks.

Production rule: privileged actions should be explicit, observable through events, protected by multisig or timelock when high impact, and covered by tests.

---

## 9. Smart Contract Security

### Reentrancy

Reentrancy happens when an external call transfers control before internal state is finalized, letting the callee call back into the contract.

Protection:

- Checks-Effects-Interactions;
- `ReentrancyGuard`;
- pull-payment pattern;
- minimize external calls.

### Other common risks

| Risk | Example | Mitigation |
|------|---------|------------|
| Access control | anyone can call admin function | roles, multisig, timelock |
| Oracle manipulation | spot price used for collateral | Chainlink, TWAP, bounds |
| Front-running / MEV | transaction order extraction | commit-reveal, private RPC, slippage limits |
| Denial of service | unbounded loops | pagination, pull pattern |
| Upgrade bug | storage collision | upgrade validation and review |
| Approval risk | unlimited token allowance | permit, allowance reset, UI warnings |

### Production practice

Use layered assurance: unit tests, fuzz tests, invariant tests, static analysis, manual review, external audit, bug bounty, monitoring, and incident playbooks.

### Interview self-check

Can you explain why "we audited the code" is not enough for DeFi protocol safety?

---

## 10. ERC Standards

### ERC-20

Fungible token standard with `transfer`, `approve`, `transferFrom`, `allowance`, and events.

Common issue: approval race and unlimited allowance risk. ERC-2612 `permit` improves UX by allowing signature-based approvals.

### ERC-721

Non-fungible token standard where each token ID is unique.

Risks include unsafe metadata assumptions, marketplace approval risk, and phishing around `setApprovalForAll`.

### ERC-1155

Multi-token standard that can manage fungible and non-fungible assets in one contract, with batch operations for gas efficiency.

---

## 11. Interview Self-Check

1. What is the EVM and why must execution be deterministic?
2. What is the difference between stack, memory, storage, calldata, and logs?
3. How does EIP-1559 pricing work?
4. What is the difference between CALL and DELEGATECALL?
5. What is reentrancy and how do you prevent it?
6. Why is storage layout critical for upgradeable contracts?
7. What are the main differences among ERC-20, ERC-721, and ERC-1155?
8. What does `view` mean compared with `pure`?
9. Why are oracles and MEV part of smart contract risk?
10. What checks would you run before mainnet deployment?
