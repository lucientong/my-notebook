# DeFi 与应用协议

---

## 📑 目录

### DEX 与交易
1. [DeFi 概述](#1-defi-概述)
2. [AMM 自动做市商](#2-amm-自动做市商)
3. [无常损失](#3-无常损失)

### 借贷与稳定币
4. [借贷协议](#4-借贷协议)
5. [稳定币机制](#5-稳定币机制)

### 基础设施
6. [预言机](#6-预言机)
7. [闪电贷](#7-闪电贷)
8. [MEV](#8-mev)

### 实践
9. [DeFi 安全](#9-defi-安全)
10. [面试题自查](#10-面试题自查)

---

## 1. DeFi 概述

### 1.1 什么是 DeFi

**DeFi（Decentralized Finance）**：去中心化金融，基于区块链的开放式金融服务。

```
传统金融 vs DeFi：

┌─────────────────────────────────────────────────────────────┐
│         传统金融 (CeFi)          │        DeFi             │
├──────────────────────────────────┼──────────────────────────┤
│ 中心化机构控制                   │ 智能合约自动执行         │
│ KYC/AML 门槛                     │ 无许可，开放接入         │
│ 营业时间限制                     │ 24/7 全天候              │
│ 不透明                           │ 链上透明可验证           │
│ 跨境汇款慢                       │ 全球即时结算             │
│ 托管风险                         │ 自托管 + 合约风险        │
└──────────────────────────────────┴──────────────────────────┘
```

### 1.2 DeFi 生态

```
DeFi 协议栈：

┌─────────────────────────────────────────────────────────────┐
│                    聚合器层                                 │
│  1inch, Paraswap, Yearn Finance                            │
├─────────────────────────────────────────────────────────────┤
│                    应用层                                   │
│  DEX: Uniswap, Curve, Balancer                             │
│  借贷: Aave, Compound, MakerDAO                            │
│  衍生品: dYdX, GMX, Synthetix                              │
├─────────────────────────────────────────────────────────────┤
│                    基础设施层                               │
│  预言机: Chainlink, Band Protocol                          │
│  桥: Wormhole, LayerZero                                   │
│  稳定币: USDC, USDT, DAI                                   │
├─────────────────────────────────────────────────────────────┤
│                    结算层                                   │
│  Ethereum, Arbitrum, Optimism, Polygon                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. AMM 自动做市商

### 2.1 恒定乘积公式

```
Uniswap V2 核心公式：

x × y = k

其中：
- x：代币 A 的储备量
- y：代币 B 的储备量
- k：常数（每次交易后保持不变）

示例：
池子初始状态：100 ETH × 200,000 USDC = 20,000,000 (k)

用户用 10 ETH 买 USDC：
- 新的 ETH 储备：100 + 10 = 110
- 新的 USDC 储备：20,000,000 / 110 = 181,818
- 用户获得 USDC：200,000 - 181,818 = 18,182

价格影响（滑点）：
- 理论价格：10 ETH × 2000 = 20,000 USDC
- 实际获得：18,182 USDC
- 滑点：9.1%
```

### 2.2 Uniswap V2 合约

```solidity
// 简化的 swap 逻辑
function swap(uint amount0Out, uint amount1Out, address to) external {
    require(amount0Out > 0 || amount1Out > 0, 'INSUFFICIENT_OUTPUT_AMOUNT');
    
    (uint112 reserve0, uint112 reserve1,) = getReserves();
    require(amount0Out < reserve0 && amount1Out < reserve1, 'INSUFFICIENT_LIQUIDITY');
    
    // 转出代币
    if (amount0Out > 0) _safeTransfer(token0, to, amount0Out);
    if (amount1Out > 0) _safeTransfer(token1, to, amount1Out);
    
    // 获取新余额
    uint balance0 = IERC20(token0).balanceOf(address(this));
    uint balance1 = IERC20(token1).balanceOf(address(this));
    
    // 计算输入量
    uint amount0In = balance0 > reserve0 - amount0Out ? balance0 - (reserve0 - amount0Out) : 0;
    uint amount1In = balance1 > reserve1 - amount1Out ? balance1 - (reserve1 - amount1Out) : 0;
    
    // 验证 k 值（扣除 0.3% 手续费后）
    uint balance0Adjusted = balance0 * 1000 - amount0In * 3;
    uint balance1Adjusted = balance1 * 1000 - amount1In * 3;
    require(balance0Adjusted * balance1Adjusted >= uint(reserve0) * uint(reserve1) * 1000**2, 'K');
    
    _update(balance0, balance1, reserve0, reserve1);
}
```

### 2.3 Uniswap V3 集中流动性

```
V3 核心创新：集中流动性

V2：流动性均匀分布在 (0, ∞) 价格范围
V3：LP 可以指定流动性的价格区间 [Pa, Pb]

┌─────────────────────────────────────────────────────────────┐
│  V2 流动性分布                                              │
│                                                             │
│  |                                                          │
│  |  ████████████████████████████████████████████████████   │
│  |  █ 流动性均匀分布在所有价格 █                           │
│  |  ████████████████████████████████████████████████████   │
│  └──────────────────────────────────────────────────────    │
│    0                      价格                       ∞      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  V3 集中流动性                                              │
│                                                             │
│  |                    █████████                             │
│  |                  ███████████████                         │
│  |                ███████████████████                       │
│  |              █████████████████████████                   │
│  └──────────────────────────────────────────────────────    │
│    0           Pa    当前价格    Pb                  ∞      │
│                  ↑ 流动性集中 ↑                            │
└─────────────────────────────────────────────────────────────┘

优势：
- 资本效率提升数百倍
- 相同 TVL 可提供更深的流动性
- LP 可以根据策略选择价格区间
```

### 2.4 其他 AMM 模型

| 协议 | 模型 | 特点 |
|------|------|------|
| Uniswap V2 | x×y=k | 通用，简单 |
| Uniswap V3 | 集中流动性 | 资本效率高 |
| Curve | StableSwap | 稳定币优化，低滑点 |
| Balancer | 加权池 | 多资产，自定义权重 |

---

## 3. 无常损失

### 3.1 无常损失原理

```
无常损失（Impermanent Loss）：

LP 提供流动性后，如果价格变化，持有 LP Token 的价值
可能低于单纯持有原资产。

示例：
初始：存入 1 ETH + 2000 USDC（ETH 价格 2000）
价格翻倍：ETH 价格变为 4000

按 x × y = k：
  原始 k = 1 × 2000 = 2000
  新价格下保持 k：
    x × y = 2000
    y/x = 4000 (新价格比)
    解得：x = √0.5 ≈ 0.707 ETH
          y = √8000 ≈ 2828 USDC

LP 持仓价值：0.707 × 4000 + 2828 = 5656 USDC
单纯持有价值：1 × 4000 + 2000 = 6000 USDC
无常损失：(6000 - 5656) / 6000 = 5.7%
```

### 3.2 无常损失公式

```
无常损失 IL = 2√r / (1+r) - 1

其中 r 是价格变化比例（新价格/原价格）

| 价格变化 | 无常损失 |
|----------|----------|
| 1.25x    | 0.6%     |
| 1.5x     | 2.0%     |
| 2x       | 5.7%     |
| 3x       | 13.4%    |
| 4x       | 20.0%    |
| 5x       | 25.5%    |

注意：价格涨跌的损失是对称的（涨 2x 和跌 0.5x 损失相同）
```

---

## 4. 借贷协议

### 4.1 超额抵押借贷

```
借贷协议核心机制（Aave/Compound）：

┌─────────────────────────────────────────────────────────────┐
│                      借贷流程                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  存款人 ──存入 USDC──> [借贷池] <──存入 ETH── 借款人       │
│     ↑                    │                      │          │
│     │                    │                      ↓          │
│   利息收益              浮动利率              借出 USDC     │
│                                                             │
└─────────────────────────────────────────────────────────────┘

关键参数：
- LTV (Loan-to-Value)：最大借款比例，如 80%
- 清算阈值：如 85%，超过触发清算
- 清算惩罚：如 5%，清算时额外扣除

示例：
存入 10 ETH（价值 $20,000），LTV=80%
最大可借：$20,000 × 80% = $16,000 USDC

如果 ETH 下跌，抵押率 > 85% 时被清算
```

### 4.2 利率模型

```
利率曲线（Aave 模型）：

利率
  │
  │                          ╱
  │                        ╱
  │                      ╱
  │                    ╱
  │             _____╱
  │       _____╱
  │______╱
  └──────────────────────────── 利用率
  0%     Uoptimal              100%

分段函数：
当 U < Uoptimal：
  利率 = R0 + U × (Rslope1 / Uoptimal)
  
当 U ≥ Uoptimal：
  利率 = R0 + Rslope1 + (U - Uoptimal) × (Rslope2 / (1 - Uoptimal))

目的：激励在高利用率时归还借款
```

### 4.3 清算机制

```solidity
// 简化的清算逻辑
function liquidate(address borrower, address collateral, uint debtToCover) external {
    // 检查是否可清算
    uint healthFactor = calculateHealthFactor(borrower);
    require(healthFactor < 1e18, "Position is healthy");
    
    // 计算可获得的抵押品
    uint collateralToReceive = debtToCover * getPrice(debtAsset) / getPrice(collateral);
    uint bonus = collateralToReceive * LIQUIDATION_BONUS / 100;  // 如 5%
    uint totalCollateral = collateralToReceive + bonus;
    
    // 执行清算
    transferFrom(msg.sender, address(this), debtToCover);  // 还债
    transfer(msg.sender, collateral, totalCollateral);      // 获得抵押品 + 奖励
    
    // 更新借款人状态
    decreaseDebt(borrower, debtToCover);
    decreaseCollateral(borrower, totalCollateral);
}

// 健康因子
// healthFactor = (抵押品价值 × 清算阈值) / 债务价值
// healthFactor < 1 时可被清算
```

---

## 5. 稳定币机制

### 5.1 稳定币分类

| 类型 | 代表 | 机制 | 风险 |
|------|------|------|------|
| 法币抵押 | USDC, USDT | 1:1 银行存款 | 中心化、监管 |
| 加密抵押 | DAI | 超额抵押 CDP | 清算风险、抵押物波动 |
| 算法稳定币 | FRAX | 部分抵押+算法 | 死亡螺旋 |

### 5.2 MakerDAO CDP

```
MakerDAO 抵押债仓（CDP/Vault）：

┌─────────────────────────────────────────────────────────────┐
│  用户存入 ETH                                               │
│       │                                                     │
│       ▼                                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Vault (抵押债仓)                        │   │
│  │  ┌─────────────┐        ┌─────────────┐            │   │
│  │  │ 抵押品 ETH  │        │ 债务 DAI    │            │   │
│  │  │   10 ETH    │        │  5000 DAI   │            │   │
│  │  └─────────────┘        └─────────────┘            │   │
│  │                                                     │   │
│  │  抵押率 = ETH价值 / DAI债务 = 300%                  │   │
│  │  最低抵押率 = 150%                                  │   │
│  └─────────────────────────────────────────────────────┘   │
│       │                                                     │
│       ▼                                                     │
│  用户获得 DAI                                               │
└─────────────────────────────────────────────────────────────┘

清算：抵押率 < 150% 时，清算人可以用 DAI 偿还债务并获得抵押品+奖励
稳定费：借出 DAI 需支付年化利息（稳定费率）
```

---

## 6. 预言机

### 6.1 预言机问题

```
预言机问题（Oracle Problem）：

区块链是确定性的封闭系统，无法直接获取外部数据（如价格）。
预言机是连接链上和链下数据的桥梁。

风险：
- 单点故障：单一数据源可能被操纵
- 延迟：链下价格变化到链上更新有延迟
- 闪电贷攻击：单交易内操纵价格
```

### 6.2 Chainlink

```
Chainlink 去中心化预言机网络：

┌─────────────────────────────────────────────────────────────┐
│                    数据源层                                 │
│  Binance, Coinbase, Kraken, 其他交易所...                  │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                  Chainlink 节点网络                         │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐           │
│  │ Node 1 │  │ Node 2 │  │ Node 3 │  │ Node N │           │
│  │$2000.01│  │$2000.05│  │$2000.03│  │$1999.98│           │
│  └────┬───┘  └────┬───┘  └────┬───┘  └────┬───┘           │
│       └───────────┼───────────┼───────────┘                │
│                   ▼                                         │
│             聚合合约（取中位数）                             │
│                   │                                         │
└───────────────────┼─────────────────────────────────────────┘
                    ▼
┌───────────────────────────────────────────────────────────┐
│              Price Feed 合约                               │
│           latestAnswer() → $2000.03                        │
└───────────────────────────────────────────────────────────┘
```

### 6.3 TWAP 时间加权平均价格

```solidity
// Uniswap V2 TWAP
contract TWAPOracle {
    uint public price0CumulativeLast;
    uint public price1CumulativeLast;
    uint32 public blockTimestampLast;
    
    // 更新累计价格
    function update() external {
        (uint price0Cumulative, uint price1Cumulative, uint32 blockTimestamp) = 
            UniswapV2OracleLibrary.currentCumulativePrices(pair);
        
        uint32 timeElapsed = blockTimestamp - blockTimestampLast;
        require(timeElapsed >= PERIOD, 'PERIOD_NOT_ELAPSED');
        
        // 计算 TWAP
        price0Average = (price0Cumulative - price0CumulativeLast) / timeElapsed;
        price1Average = (price1Cumulative - price1CumulativeLast) / timeElapsed;
        
        price0CumulativeLast = price0Cumulative;
        price1CumulativeLast = price1Cumulative;
        blockTimestampLast = blockTimestamp;
    }
}

// TWAP 优势：单区块内无法操纵，因为需要时间积累
// 劣势：价格有延迟，不适合高频场景
```

---

## 7. 闪电贷

### 7.1 闪电贷原理

```
闪电贷（Flash Loan）：同一交易内借款并归还的无抵押贷款

交易原子性保证：
- 要么全部成功（借款 + 使用 + 归还）
- 要么全部失败（状态回滚）

┌─────────────────────────────────────────────────────────────┐
│                    单笔交易内                               │
│                                                             │
│  1. 借款：从池子借出 1,000,000 USDC                        │
│         ↓                                                   │
│  2. 使用：套利、清算、自清算、抵押品置换...                │
│         ↓                                                   │
│  3. 归还：归还 1,000,000 + 手续费 USDC                     │
│         ↓                                                   │
│  4. 验证：池子检查余额 ≥ 原始 + 手续费                     │
│                                                             │
│  如果第4步验证失败 → 整个交易回滚，借款从未发生            │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 闪电贷实现

```solidity
// Aave V3 闪电贷
interface IFlashLoanReceiver {
    function executeOperation(
        address[] calldata assets,
        uint256[] calldata amounts,
        uint256[] calldata premiums,  // 手续费
        address initiator,
        bytes calldata params
    ) external returns (bool);
}

contract MyFlashLoan is IFlashLoanReceiver {
    IPool constant POOL = IPool(AAVE_POOL_ADDRESS);
    
    function executeFlashLoan(uint256 amount) external {
        address[] memory assets = new address[](1);
        assets[0] = USDC;
        
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = amount;
        
        // 发起闪电贷
        POOL.flashLoan(
            address(this),  // receiver
            assets,
            amounts,
            0,              // interestRateMode
            bytes(""),      // params
            0               // referralCode
        );
    }
    
    // 回调函数
    function executeOperation(
        address[] calldata assets,
        uint256[] calldata amounts,
        uint256[] calldata premiums,
        address initiator,
        bytes calldata params
    ) external override returns (bool) {
        // 使用借来的资金（套利、清算等）
        doArbitrage(amounts[0]);
        
        // 归还：本金 + 手续费
        uint256 amountOwed = amounts[0] + premiums[0];
        IERC20(assets[0]).approve(address(POOL), amountOwed);
        
        return true;
    }
}
```

### 7.3 闪电贷应用

| 应用 | 说明 |
|------|------|
| 套利 | 跨 DEX 价差套利 |
| 清算 | 无资金清算他人仓位 |
| 自清算 | 闪电贷还债避免被清算 |
| 抵押品置换 | 无需平仓更换抵押品 |
| 攻击 | 操纵价格预言机 |

---

## 8. MEV

### 8.1 MEV 概述

```
MEV（Maximal Extractable Value）：

区块生产者通过排序、插入、审查交易可提取的最大价值。

类型：
1. 套利：DEX 价差套利
2. 清算：借贷协议清算
3. 三明治攻击：前后夹击用户交易
4. 时间强盗：重组区块获取历史 MEV

参与者：
- Searcher：寻找 MEV 机会
- Builder：构建包含 MEV 交易的区块
- Proposer：提议区块（PoS 验证者）
```

### 8.2 三明治攻击

```
三明治攻击（Sandwich Attack）：

用户广播交易：用 10 ETH 买 TOKEN，滑点容忍 2%

攻击者检测到 mempool 中的交易：

┌─────────────────────────────────────────────────────────────┐
│  原区块交易顺序：                                           │
│  [其他交易] [用户: 10 ETH → TOKEN] [其他交易]              │
│                                                             │
│  攻击者重排后：                                             │
│  [攻击者: 买 TOKEN]   ← 前置交易：推高价格                  │
│  [用户: 10 ETH → TOKEN] ← 用户以更高价格成交               │
│  [攻击者: 卖 TOKEN]   ← 后置交易：卖出获利                  │
└─────────────────────────────────────────────────────────────┘

用户损失 = 滑点内的价差
攻击者利润 = 价差 - Gas 费用
```

### 8.3 MEV 保护

```
MEV 保护方案：

1. Flashbots Protect
   - 交易直接发给 Flashbots，不进入公共 mempool
   - 防止前置运行和三明治攻击

2. MEV-Share
   - 用户分享部分 MEV 收益
   - Searcher 为获取优先权付费给用户

3. 私有内存池
   - 使用私有 RPC 提交交易

4. 协议层保护
   - Commit-Reveal 方案
   - 批量拍卖（如 CoW Protocol）

5. 用户侧保护
   - 设置严格滑点
   - 使用限价单
   - 拆分大额交易
```

---

## 9. DeFi 安全

### 9.1 常见攻击

| 攻击类型 | 说明 | 案例 |
|---------|------|------|
| 闪电贷攻击 | 操纵价格预言机 | bZx, Harvest |
| 重入攻击 | 递归调用提取资金 | The DAO |
| 预言机操纵 | 单区块价格操纵 | 多个协议 |
| 治理攻击 | 闪电贷借票通过恶意提案 | Beanstalk |
| 桥接攻击 | 跨链桥漏洞 | Wormhole, Ronin |

### 9.2 安全最佳实践

```
DeFi 协议安全清单：

1. 预言机
   - 使用 Chainlink 或 TWAP
   - 多数据源聚合
   - 价格边界检查

2. 闪电贷防护
   - 关键操作跨区块（存款→操作）
   - 使用区块延迟

3. 重入防护
   - 检查-生效-交互模式
   - ReentrancyGuard

4. 权限控制
   - 时间锁延迟
   - 多签

5. 审计
   - 多家审计公司
   - 漏洞赏金计划
```

---

## 10. 面试题自查

**Q1: Uniswap V2 的 x×y=k 公式是什么意思？**

恒定乘积做市商，池中两种代币储备量的乘积恒定。交易时一种代币增加，另一种减少，保持 k 不变。这决定了价格（y/x）随交易量自动调整。

**Q2: 什么是无常损失？何时发生？**

LP 提供流动性后，如果价格变化，持有 LP Token 的价值可能低于单纯持有原资产。价格变化越大，无常损失越大。价格恢复到初始时，损失消失（"无常"）。

**Q3: Uniswap V3 相比 V2 的核心改进是什么？**

集中流动性：LP 可以指定提供流动性的价格区间，而非全范围。这使资本效率提升数百倍，同样 TVL 可提供更深的流动性。

**Q4: 借贷协议的清算机制是如何工作的？**

当借款人的健康因子（抵押品价值×清算阈值/债务价值）低于 1 时，清算人可以代还部分债务，获得等值抵押品加清算奖励（如 5%）。激励清算人维护协议健康。

**Q5: 什么是闪电贷？有什么用途？**

同一交易内借款并归还的无抵押贷款，利用交易原子性保证。用途：套利、清算、自清算、抵押品置换。风险：可被用于操纵价格攻击。

**Q6: 预言机的 TWAP 是什么？为什么能防止价格操纵？**

Time-Weighted Average Price，时间加权平均价格。通过累计价格除以时间间隔计算。由于需要时间积累，攻击者无法在单个区块内操纵 TWAP，防止闪电贷攻击。

**Q7: 什么是 MEV？三明治攻击是如何进行的？**

MEV 是区块生产者通过交易排序可提取的最大价值。三明治攻击：攻击者在用户交易前后各插入一笔交易，前置交易推高价格，后置交易卖出获利，用户以更差价格成交。

**Q8: MakerDAO 的 DAI 是如何保持锚定的？**

用户超额抵押（如 150%）铸造 DAI，抵押率不足时被清算。稳定费（借贷利率）和 DAI 存款利率调节供需。价格偏离时套利者会买卖 DAI 恢复锚定。

**Q9: Curve 的 StableSwap 和 Uniswap 的恒定乘积有什么区别？**

StableSwap 使用修改后的曲线，在稳定币等价附近非常平坦（极低滑点），偏离时趋向恒定乘积。适合稳定币交易，相同流动性下滑点比 Uniswap 低得多。

**Q10: 如何防止 DeFi 协议被闪电贷攻击？**

使用 Chainlink 等抗操纵预言机、TWAP 时间加权价格、关键操作跨区块（存款后下一区块才能操作）、价格边界检查、使用多数据源聚合。

**Q11: 借贷协议的利率模型是如何设计的？**

利率随资金利用率上升。低利用率时利率低（激励借款），高利用率时利率陡峭上升（激励还款、保证流动性）。分段函数，Uoptimal 是拐点。

**Q12: 什么是 Flashbots？如何保护用户免受 MEV？**

Flashbots 提供私有交易通道，交易直接发给区块构建者，不进入公共 mempool，防止被前置运行和三明治攻击。MEV-Share 还允许用户分享部分 MEV 收益。

**Q13: 算法稳定币的"死亡螺旋"是什么？**

当算法稳定币价格下跌，用户恐慌抛售，协议机制（如铸造治理代币）加剧卖压，治理代币暴跌，进一步失去信心，最终脱锚归零。如 UST/LUNA。

**Q14: LP 提供流动性能获得什么收益？风险是什么？**

收益：交易手续费（如 0.3%）、流动性挖矿代币奖励。风险：无常损失、智能合约漏洞、极端行情下抵押品大幅贬值、协议被攻击。

**Q15: 什么是 DeFi 可组合性（Composability）？有什么风险？**

DeFi 协议可以像乐高一样组合使用（如闪电贷+清算+DEX 套利）。风险：一个协议漏洞可能级联影响依赖它的所有协议，系统性风险增加。
