# Layer2 and Cross-Chain Technology

Language: English | [中文](../Web3知识库/05-Layer2与跨链技术.md)

> Layer 2 and bridge interviews are trust-assumption interviews. Always ask: where is execution done, where is data published, where is settlement, who orders transactions, who verifies messages, and how can users exit safely?

---

## 1. Ethereum Scalability Problem

### Concept

Ethereum L1 prioritizes decentralization and security, which limits throughput and keeps execution expensive. Layer 2 systems improve scalability by moving execution away from L1 while preserving some L1 guarantees.

### Mechanism

L1 bottlenecks include:

- every full node executes every transaction;
- global state keeps growing;
- block gas limits bound execution throughput;
- P2P propagation and validator requirements limit block size.

Scaling categories:

- L1 scaling: protocol-level improvements;
- state channels and Plasma: older off-chain designs;
- rollups: the dominant L2 approach;
- validiums and volitions: lower-cost designs with different data availability assumptions.

### Trade-off and risk

Scaling systems usually change at least one assumption: execution location, data availability, finality, sequencing, withdrawal path, or bridge trust.

### Production practice

When choosing an L2, evaluate security stage, sequencer decentralization, proof system, upgrade controls, bridge design, data availability, ecosystem tooling, and incident history.

### Interview self-check

Can you explain what security property a rollup inherits from Ethereum and what it does not inherit automatically?

---

## 2. Rollup Core Architecture

### Concept

A rollup executes transactions off-chain and posts enough information to L1 so that state updates can be verified or challenged.

### Mechanism

```text
users submit L2 transactions
  -> sequencer orders and executes them
  -> batch is created
  -> data is posted to L1 calldata or blobs
  -> L1 contract accepts state root through fraud proof or validity proof
  -> users can bridge or withdraw according to protocol rules
```

Security depends on:

- data availability: can users reconstruct the state?
- execution correctness: can invalid state roots be rejected?
- exit guarantees: can users withdraw if the sequencer fails or censors?

### Trade-off and risk

Rollups reduce cost by batching many transactions, but they introduce sequencer dependency, bridge contracts, proof infrastructure, and sometimes centralized upgrade keys.

### Production practice

Monitor batch posting, L1 data costs, sequencer uptime, withdrawal queues, proof submission, bridge liquidity, and contract upgrades.

### Interview self-check

Can you explain why data availability is essential for a rollup but not the same thing as execution correctness?

---

## 3. Optimistic Rollups

### Concept

Optimistic rollups assume state updates are valid unless challenged during a challenge window.

### Mechanism

- Sequencer executes transactions and posts batches.
- Validators recompute state.
- If a state transition is invalid, a challenger submits a fraud proof.
- After the challenge window, the state is accepted as final for withdrawal.

Interactive fraud proofs reduce L1 computation by using a dispute game to locate the exact invalid step.

### Trade-off and risk

Advantages:

- strong EVM compatibility;
- simpler execution than ZK systems;
- mature ecosystem.

Risks:

- withdrawal delay;
- honest challenger assumption;
- sequencer centralization;
- fraud-proof implementation maturity;
- bridge and upgrade governance risk.

### Production practice

For applications, account for withdrawal delays and bridge liquidity. For protocol analysis, verify whether fraud proofs are live, who can challenge, and how forced transactions work.

### Interview self-check

Can you explain why optimistic rollups require a challenge period?

---

## 4. ZK Rollups

### Concept

A ZK rollup posts a validity proof that proves the correctness of a batch's state transition.

### Mechanism

- L2 executes many transactions.
- A prover generates a proof for the state transition.
- The L1 verifier contract checks the proof.
- If valid, the new state root is accepted.

ZK systems use SNARKs, STARKs, PLONK-like systems, recursion, and specialized prover infrastructure.

### Trade-off and risk

Advantages:

- faster finality after proof verification;
- no fraud challenge window;
- strong correctness guarantee if the circuit and verifier are sound.

Risks:

- prover complexity and cost;
- circuit bugs;
- verifier contract bugs;
- EVM equivalence challenges;
- hardware and latency constraints.

### Production practice

Evaluate prover decentralization, proof latency, circuit audit history, EVM compatibility level, and upgrade controls.

### Interview self-check

Can you explain why a validity proof changes the withdrawal experience compared with fraud proofs?

---

## 5. Data Availability

### Concept

Data availability means transaction data is actually accessible to the network. Without data, users cannot reconstruct state or prove what they own.

### Mechanism

Rollups publish transaction data to Ethereum L1, historically through calldata and now often through blob data introduced by EIP-4844.

Alternative DA designs use external DA layers or data availability committees.

### Trade-off and risk

| Design | Data location | Cost | Security assumption |
|--------|---------------|------|---------------------|
| Rollup | Ethereum L1 | higher | Ethereum DA |
| Validium | off-chain committee | lower | committee honesty |
| External DA layer | separate DA network | lower or medium | DA layer security |

If DA fails, funds may become frozen or impossible to exit safely even if the execution proof is valid.

### Production practice

For high-value assets, prefer stronger DA assumptions. Monitor blob posting, DA layer status, and forced-exit paths.

### Interview self-check

Can you explain how a malicious sequencer could cause problems by withholding data?

---

## 6. EIP-4844, Blobs, and Danksharding Direction

### Concept

EIP-4844 introduced blobs: temporary, cheaper data containers optimized for rollup data availability.

### Mechanism

Blob data:

- is attached to special transactions;
- is committed using KZG commitments;
- is not directly accessible by EVM execution;
- is retained by consensus nodes for a limited period;
- has a fee market separate from normal execution gas.

### Trade-off and risk

Blobs reduce L2 costs, but they do not by themselves solve all scaling limits. They also make rollup operations depend on blob fee markets and data retention assumptions.

### Production practice

Rollup teams should track blob fee volatility, batch compression, fallback behavior, proof and data posting reliability, and archival requirements for indexers.

### Interview self-check

Can you explain the difference between calldata and blob data?

---

## 7. Modular Blockchains and DA Layers

### Concept

A modular blockchain separates execution, settlement, consensus, and data availability into different layers.

### Mechanism

Example stack:

- execution: rollup VM or appchain;
- settlement: Ethereum or another settlement layer;
- data availability: Ethereum blobs, Celestia, EigenDA, Avail, or another DA layer;
- consensus: ordering for the DA or settlement layer.

Data availability sampling lets light nodes probabilistically check that data was published without downloading all data.

### Trade-off and risk

Modularity improves specialization and scalability but adds cross-layer complexity and new trust assumptions.

### Production practice

Map all trust boundaries. If execution, DA, and settlement are separate, failure in any layer can affect user safety or liveness.

### Interview self-check

Can you explain why modularity helps scaling but complicates security analysis?

---

## 8. Cross-Chain Bridges

### Concept

Different chains are independent state machines. A bridge moves assets or messages between them by proving or attesting that something happened on the source chain.

### Mechanism

Bridge patterns:

- `lock-mint`: lock asset on source chain, mint wrapped asset on destination;
- `burn-mint`: burn on source chain, mint on destination;
- `liquidity pool`: user deposits on one chain and receives liquidity on another;
- `native rollup bridge`: relies on L1 and rollup contracts.

Verification models:

- multisig or MPC external validators;
- optimistic verification with challenge period;
- light-client verification;
- ZK proof verification;
- native verification through settlement layer.

### Trade-off and risk

Bridges are high-risk because they combine smart contract risk, validator risk, key-management risk, message replay risk, liquidity risk, and chain-reorg risk.

### Production practice

Prefer canonical bridges for high-value transfers when possible. For third-party bridges, analyze validator set, threshold, upgrade keys, limits, pause controls, monitoring, audits, and incident history.

### Interview self-check

Can you explain why bridges are often described as one of the weakest points in crypto infrastructure?

---

## 9. Cross-Chain Messaging

### Concept

Cross-chain messaging sends arbitrary instructions or state proofs across chains, not just tokens.

### Mechanism

A message usually contains:

- source chain ID;
- destination chain ID;
- sender;
- receiver;
- nonce or message ID;
- payload;
- proof or validator attestation.

Protocols such as IBC use light clients so each chain can verify the other chain's state transitions under the counterparty consensus assumptions.

### Trade-off and risk

Arbitrary messaging increases composability but also increases blast radius. A forged message can trigger actions in lending, governance, minting, or bridging contracts.

### Production practice

Always include chain IDs, nonces, replay protection, source address verification, payload validation, and clear failure handling for delayed or failed messages.

### Interview self-check

Can you explain how nonce and chain ID prevent replay attacks?

---

## 10. Bridge Security Case Studies

### Ronin-style validator compromise

Root causes:

- concentrated validator control;
- compromised private keys;
- stale permissions;
- weak monitoring.

Lesson: threshold signatures do not help if the threshold is operationally easy to compromise.

### Nomad-style message validation failure

Root causes:

- invalid initialization or trusted root handling;
- insufficient message verification;
- replayable attack pattern.

Lesson: bridge verification logic must be treated as consensus-critical code.

### Production security controls

- decentralized validators or stronger verification;
- rate limits and daily limits;
- large-transfer delays;
- multisig and timelock governance;
- emergency pause with clear policy;
- real-time monitoring;
- independent audits and bug bounty;
- replay protection and domain separation.

---

## 11. Interview Self-Check

1. What is the blockchain trilemma?
2. What makes rollups different from sidechains?
3. How does an optimistic rollup use fraud proofs?
4. How does a ZK rollup use validity proofs?
5. What is data availability, and why is it essential?
6. What is the difference between calldata and blob data?
7. What is a KZG commitment used for in blob data?
8. What is the difference between rollup, validium, and external DA designs?
9. How does lock-mint bridging work?
10. What are the main bridge verification models?
11. How do replay attacks happen in cross-chain messaging?
12. How would you design safety controls for a high-value bridge?
