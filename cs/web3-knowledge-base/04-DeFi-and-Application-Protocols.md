# DeFi and Application Protocols

Language: English | [中文](../Web3知识库/04-DeFi与应用协议.md)

> DeFi interviews are not about naming protocols. They are about explaining how assets flow, how prices are formed, how risk is transferred, and how incentives can fail.

---

## 1. DeFi Asset-Flow Mental Model

### Concept

DeFi replaces parts of financial infrastructure with smart contracts and on-chain settlement. The core objects are assets, pools, users, collateral, debt, prices, and governance.

```text
user funds
  -> wallet
  -> protocol contract
  -> pool, vault, lending market, stablecoin system, or derivative venue
  -> price source from pool math or oracle
  -> swaps, interest, liquidation, arbitrage, or settlement
```

### Mechanism

Protocol rules are executed by contracts. Users interact through wallets. Prices come from AMM curves, order books, oracles, or a combination. Arbitrageurs and liquidators keep systems closer to intended states.

### Trade-off and risk

DeFi is composable and transparent, but the risk surface includes smart contract bugs, economic design flaws, oracle manipulation, liquidity shocks, MEV, governance capture, and bridge dependency.

### Production practice

Before integrating with a protocol, analyze asset flow, admin controls, oracle design, liquidation logic, pause mechanisms, upgradeability, historical incidents, audits, and dependency graph.

### Interview self-check

Can you trace where money enters, where it can leave, who can change rules, and where price assumptions come from?

---

## 2. AMMs and DEXs

### Concept

An automated market maker lets users trade against a liquidity pool instead of a centralized order book.

### Mechanism

Uniswap V2 uses the constant product formula:

```text
x * y = k
```

If a user adds token X to the pool, token Y must be removed so the product remains approximately constant after fees. The pool price changes with reserves, which creates slippage.

Uniswap V3 adds concentrated liquidity. LPs choose price ranges where their capital is active, improving capital efficiency.

### Trade-off and risk

- Constant product AMMs are simple and permissionless but capital-inefficient across all prices.
- Concentrated liquidity improves capital efficiency but requires active LP management.
- Large trades cause slippage.
- Low-liquidity pools are easier to manipulate.
- AMM spot prices are unsafe as direct oracles.

### Production practice

Use slippage limits, TWAPs, route splitting, liquidity checks, and MEV-aware transaction submission for large trades.

### Interview self-check

Can you explain why `x*y=k` creates automatic pricing and why it also creates slippage?

---

## 3. Impermanent Loss

### Concept

Impermanent loss is the difference between the value of holding LP shares and simply holding the original assets when relative prices move.

### Mechanism

In a constant product pool, arbitrageurs rebalance reserves to match external prices. LPs end up holding more of the underperforming asset and less of the outperforming asset.

The simplified formula is:

```text
IL = 2 * sqrt(r) / (1 + r) - 1
```

where `r` is the price ratio after the move.

### Trade-off and risk

LPs earn fees but take price divergence risk. Fee income may or may not compensate for impermanent loss.

### Production practice

Evaluate pool volatility, fee tier, trading volume, range selection, token correlation, and incentive emissions. Stablecoin pools have lower divergence risk but still carry depeg and smart contract risk.

### Interview self-check

Can you explain why LPs lose relative value when one asset doubles against the other?

---

## 4. Lending Protocols

### Concept

Most DeFi lending is overcollateralized. Borrowers deposit collateral and borrow assets below the collateral value.

### Mechanism

Key parameters:

- `LTV`: maximum borrowing power;
- `liquidation threshold`: point where the position becomes liquidatable;
- `health factor`: collateral value adjusted by threshold divided by debt value;
- `liquidation bonus`: incentive paid to liquidators;
- `utilization`: borrowed amount divided by supplied liquidity.

Interest rates usually rise with utilization to encourage deposits and repayments when liquidity becomes scarce.

### Trade-off and risk

Lending safety depends on collateral liquidity, oracle correctness, liquidation speed, market volatility, and parameter governance. Bad parameters can make a protocol insolvent even if the code works.

### Production practice

Monitor health factor distribution, oracle staleness, liquidation throughput, bad debt, utilization spikes, governance changes, and collateral concentration.

### Interview self-check

Can you explain how a lending protocol remains solvent during a fast collateral price drop?

---

## 5. Stablecoins

### Concept

A stablecoin attempts to maintain a price peg, often to USD.

### Mechanism

Major categories:

- fiat-backed: reserves held by an issuer, e.g. USDC or USDT;
- crypto-collateralized: overcollateralized on-chain vaults, e.g. DAI;
- algorithmic or partially collateralized: relies on incentives, mint/burn mechanics, or endogenous collateral.

### Trade-off and risk

- Fiat-backed coins have centralized issuer and regulatory risk.
- Crypto-collateralized coins have liquidation and collateral volatility risk.
- Algorithmic designs can enter death spirals when confidence breaks.

### Production practice

Check reserve transparency, redemption path, collateral type, oracle dependency, governance powers, blacklisting controls, and historical peg behavior.

### Interview self-check

Can you explain why stablecoin risk depends on the backing mechanism?

---

## 6. Oracles

### Concept

A blockchain cannot directly read external data. Oracles bring off-chain data such as prices on-chain.

### Mechanism

Oracle designs include:

- centralized feeds;
- decentralized node networks;
- exchange-weighted prices;
- TWAP from AMMs;
- cross-chain state proofs.

TWAP reduces single-block manipulation by averaging price over time, but introduces lag.

### Trade-off and risk

Oracles are often the weakest dependency in DeFi. A protocol can be exploited if it uses a manipulable spot price, stale feed, wrong decimals, wrong asset pair, or insufficient deviation checks.

### Production practice

Use robust feeds, staleness checks, circuit breakers, sanity bounds, fallback rules, and monitoring. Never use a low-liquidity AMM spot price as the only collateral oracle.

### Interview self-check

Can you explain the oracle problem and why TWAP helps against flash-loan manipulation?

---

## 7. Flash Loans

### Concept

A flash loan is an uncollateralized loan that must be borrowed and repaid within one transaction.

### Mechanism

Transaction atomicity guarantees that if the borrower fails to repay principal plus fee, the entire transaction reverts.

Use cases:

- arbitrage;
- liquidation;
- collateral swap;
- refinancing;
- attacks involving price manipulation or governance voting.

### Trade-off and risk

Flash loans are neutral primitives. They make capital-efficient strategies possible, but they also let attackers temporarily access enormous capital to stress protocol assumptions.

### Production practice

Design protocols as if attackers have unlimited temporary capital inside one transaction. Use robust oracles, time delays, and cross-block assumptions only when carefully enforced.

### Interview self-check

Can you explain why flash loans are dangerous only when another protocol has a weak assumption?

---

## 8. MEV

### Concept

MEV, or maximal extractable value, is value captured by ordering, inserting, censoring, or reordering transactions.

### Mechanism

Common MEV forms:

- DEX arbitrage;
- liquidations;
- sandwich attacks;
- backrunning;
- time-bandit attacks;
- cross-domain MEV.

A sandwich attack places one transaction before and one after a user's trade to profit from the user's price impact.

### Trade-off and risk

MEV can improve market efficiency through arbitrage and liquidation, but it can harm users through worse execution and can create consensus centralization pressure.

### Production practice

Use slippage limits, private transaction routes, batch auctions, intent-based execution, commit-reveal when suitable, and MEV-aware monitoring.

### Interview self-check

Can you explain the difference between benign arbitrage MEV and harmful sandwich MEV?

---

## 9. DeFi Security

### Risk categories

| Category | Failure mode | Example mitigation |
|----------|--------------|--------------------|
| Code risk | reentrancy, access control bug | audit, tests, guards |
| Oracle risk | manipulated or stale price | robust feeds, TWAP, bounds |
| Economic risk | bad liquidation or incentive design | simulations, stress tests |
| Governance risk | malicious parameter change | multisig, timelock, monitoring |
| Composability risk | dependency failure cascades | limits, isolation, dependency review |
| Bridge risk | validator or message failure | minimize bridge dependency |

### Production practice

Run scenario analysis: oracle failure, liquidity collapse, extreme volatility, mass liquidation, governance compromise, sequencer downtime, bridge delay, and RPC outage.

### Interview self-check

Can you explain why DeFi security is not just smart contract security?

---

## 10. Interview Self-Check

1. What does `x*y=k` mean in an AMM?
2. Why does slippage increase with trade size?
3. What is impermanent loss?
4. How does a lending protocol decide when to liquidate?
5. What are the main stablecoin models and risks?
6. What is the oracle problem?
7. Why do flash loans amplify weak assumptions?
8. What is MEV, and how does a sandwich attack work?
9. How would you design oracle protection for a lending market?
10. What should you review before integrating with an external DeFi protocol?
