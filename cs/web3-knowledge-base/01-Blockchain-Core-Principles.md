# Blockchain Core Principles

Language: English | [中文](../Web3知识库/01-区块链核心原理.md)

> Read this document as a system story: signed transactions modify a shared ledger, blocks organize history, cryptography makes data verifiable, and consensus decides which history is canonical.

---

## 1. Start With One Transaction

### Concept

A blockchain transaction is a signed instruction to modify shared state. It is not simply a database row. It carries authorization, replay protection, fee information, and state-transition intent.

### Mechanism

```text
user creates transaction
  -> signs it with a private key
  -> broadcasts it to the P2P network
  -> nodes validate signature, nonce, balance, and rules
  -> a block producer includes it in a block
  -> consensus accepts the block
  -> all nodes apply the same state transition
```

Different mechanisms protect different parts of this path:

- digital signatures prove authorization;
- hashes bind data to a tamper-evident identifier;
- Merkle trees allow efficient inclusion proofs;
- consensus decides which candidate history becomes canonical;
- finality defines when reversal becomes infeasible or impossible under the protocol assumptions.

### Trade-off and risk

The system is robust because many parties verify the same transition. It is slower because many parties must verify the same transition.

### Production practice

For production systems, always reason about transaction lifecycle: pending state, mempool visibility, nonce conflicts, reorgs, confirmations, finality, and replay protection.

### Interview self-check

Can you explain what each component protects: signature, hash chain, Merkle proof, consensus, and finality?

---

## 2. Blocks, Hash Chains, and Immutability

### Concept

A block groups transactions and links to the previous block through a cryptographic hash. The chain of hashes makes historical tampering visible.

### Mechanism

A simplified block contains:

- block header;
- previous block hash;
- Merkle root of transactions;
- timestamp and consensus-specific fields;
- transaction list.

If an attacker modifies an old transaction, the transaction hash changes, the Merkle root changes, the block hash changes, and every later block reference becomes invalid.

### Trade-off and risk

"Immutable" does not mean mathematically impossible to rewrite. It means rewriting becomes economically, computationally, or protocol-wise infeasible under the consensus assumptions. In PoW, rewriting requires enormous hash power. In PoS, attacks can require slashable stake and coordination.

### Production practice

Applications should wait for an appropriate number of confirmations or finality checkpoints before treating a transaction as settled. The required threshold depends on asset value and chain security.

### Interview self-check

Can you explain why changing one historical transaction invalidates later blocks?

---

## 3. UTXO vs Account Model

### Concept

Blockchains represent state in different ways. Bitcoin uses UTXOs, while Ethereum uses accounts.

### Mechanism

In the `UTXO model`, coins are unspent outputs. A transaction consumes old UTXOs and creates new UTXOs.

In the `account model`, the chain tracks account state such as balance, nonce, code hash, and storage root. Transactions update these account states.

### Trade-off and risk

| Dimension | UTXO | Account |
|----------|------|---------|
| State representation | Set of unspent outputs | Global account state |
| Parallelism | Natural because inputs are independent | Harder because nonces and storage can conflict |
| Privacy | Better if addresses are not reused | Weaker due to account reuse |
| Smart contracts | Limited in Bitcoin-style scripts | More flexible and expressive |
| Developer model | More explicit asset flow | More familiar stateful programming |

### Production practice

When designing wallets, indexers, or bridges, the transaction model matters. UTXO systems require coin selection and change outputs. Account systems require nonce management, gas estimation, and storage-aware execution.

### Interview self-check

Can you explain why UTXO is easier to parallelize but less convenient for rich smart contracts?

---

## 4. P2P Networking and Propagation

### Concept

A blockchain network is a peer-to-peer system. Nodes discover peers, exchange transactions and blocks, and maintain enough connectivity for fast propagation.

### Mechanism

- `Kademlia-style DHT`: helps nodes discover peers using XOR distance and routing tables.
- `Gossip protocol`: spreads transactions and blocks by forwarding messages to peers.
- `Compact blocks`: reduce bandwidth by sending block headers and short transaction IDs when peers already have many transactions in the mempool.

### Trade-off and risk

Faster propagation reduces stale blocks and improves consensus stability. But open P2P networks face eclipse attacks, spam, bandwidth limits, and peer churn.

### Production practice

Node operators should monitor peer count, propagation delay, client diversity, disk usage, and network health. Critical services should avoid relying on a single RPC provider.

### Interview self-check

Can you explain why gossip is a good fit for decentralized networks?

---

## 5. Consensus Mechanisms

### Concept

Consensus answers four questions:

1. Who can propose the next block?
2. How are Sybil attacks resisted?
3. How are forks resolved?
4. When is a block considered final?

### Proof of Work

PoW makes block production costly by requiring miners to find a hash below a difficulty target. Security comes from the cost of acquiring and operating hash power.

Trade-offs:

- strong and battle-tested;
- high energy cost;
- probabilistic finality;
- vulnerable in theory to majority hash power attacks or selfish mining.

### Proof of Stake

PoS selects validators based on stake and punishes misbehavior through slashing. Ethereum uses slots, epochs, attestations, fork choice, and checkpoint finality.

Trade-offs:

- lower energy usage;
- economic finality;
- requires careful slashing, validator incentives, and client diversity;
- introduces stake concentration and governance concerns.

### BFT-style consensus

PBFT, Tendermint, and HotStuff-style protocols use voting rounds and quorums. They can provide fast or immediate finality but usually need a bounded validator set.

### Production practice

When evaluating a chain, ask:

- what resource prevents Sybil attacks;
- how many entities control block production;
- how finality works;
- what happens under network partitions;
- what client diversity and validator distribution look like.

### Interview self-check

Can you compare PoW, PoS, and BFT using Sybil resistance, finality, scalability, and decentralization?

---

## 6. Cryptographic Hashes and Digital Signatures

### Hash Functions

A cryptographic hash maps arbitrary input to a fixed-size output. It should be deterministic, efficient, collision-resistant, preimage-resistant, and avalanche-like.

Common examples:

- Bitcoin uses SHA-256 heavily;
- Ethereum uses Keccak-256, which differs slightly from standardized SHA-3.

### Digital Signatures

Digital signatures prove that the holder of a private key authorized a message. Ethereum and Bitcoin historically use ECDSA over secp256k1.

Signature safety depends on never reusing the signing nonce `k`. If `k` is reused for two signatures, the private key can be recovered.

### Trade-off and risk

Cryptography gives strong primitives, but operational mistakes still break systems: weak randomness, leaked keys, replay attacks, phishing, unsafe signing prompts, or compromised multisig signers.

### Production practice

Use audited signing libraries, hardware-backed keys for high-value roles, chain ID protection, EIP-712 typed data for readable signing, and multisig with clear operational controls.

### Interview self-check

Can you explain why signature nonce reuse can leak a private key?

---

## 7. Merkle Trees and Light Verification

### Concept

A Merkle tree is a binary hash tree that commits to many pieces of data with one root hash.

### Mechanism

To prove that a transaction is included in a block, a prover provides the transaction and the sibling hashes along the path to the Merkle root. The verifier recomputes the root and compares it with the block header.

Proof size is `O(log N)`, much smaller than downloading the whole block.

### Trade-off and risk

A Merkle proof proves inclusion, not necessarily semantic validity unless the verifier also trusts the block header and consensus assumptions.

### Production practice

Merkle proofs are used in light clients, bridge verification, airdrops, allowlists, rollup state proofs, and storage proofs.

### Interview self-check

Can you explain what a Merkle proof proves and what it does not prove?

---

## 8. Zero-Knowledge Proofs

### Concept

A zero-knowledge proof lets a prover convince a verifier that a statement is true without revealing extra information.

Core properties:

- completeness: true statements can be proven;
- soundness: false statements cannot be proven except with negligible probability;
- zero-knowledge: the verifier learns nothing beyond the truth of the statement.

### Mechanism

`zk-SNARKs` are small and fast to verify, but often require trusted setup and rely on elliptic-curve assumptions.

`zk-STARKs` avoid trusted setup and are hash-based, but proofs are larger.

In ZK rollups, the prover executes many transactions off-chain and generates a validity proof. The L1 verifier contract checks the proof and accepts the new state root.

### Trade-off and risk

ZK systems reduce on-chain verification cost but add prover complexity, circuit engineering, trusted setup considerations, hardware requirements, and audit difficulty.

### Production practice

For production ZK systems, monitor prover latency, proof generation failures, circuit soundness, upgrade governance, verifier contract correctness, and data availability.

### Interview self-check

Can you explain why a ZK rollup can withdraw faster than an optimistic rollup?

---

## 9. Forks and Finality

### Concept

A fork is a divergence in blockchain history. Finality defines when a block should no longer be reversed under normal security assumptions.

### Mechanism

Fork types:

- temporary fork: caused by network delay or simultaneous block proposals;
- soft fork: backward-compatible rule tightening;
- hard fork: incompatible rule change that requires coordinated upgrade.

Finality types:

- probabilistic finality: reversal probability decreases with confirmations;
- economic finality: reversal requires slashable economic loss;
- absolute or deterministic finality: once committed by a BFT quorum, the block is final under assumptions.

### Production practice

Use different confirmation depths for low-value UI updates, exchange deposits, bridge transfers, and high-value settlement.

### Interview self-check

Can you explain the difference between fork choice and finality?

---

## 10. Interview Self-Check

1. Why is blockchain history hard to tamper with?
2. What is the difference between UTXO and account models?
3. How does a Merkle proof support light clients?
4. How do PoW and PoS resist Sybil attacks?
5. What problem does slashing solve in PoS?
6. What are the three core properties of zero-knowledge proofs?
7. Why do optimistic rollups need a challenge window while ZK rollups do not?
8. What is the difference between soft forks and hard forks?
9. What is finality, and why does it matter for bridges and exchanges?
10. What should a production system do when chain reorganizations happen?
