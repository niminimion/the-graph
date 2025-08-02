# DeFi Portfolio Tracker & Analytics Dashboard
## The Graph Protocol 竞赛项目提案

> 一个基于 The Graph Protocol 的创新型 DeFi 投资组合追踪和分析平台

---

## 🎯 项目概述

### 核心理念
构建一个**智能化的DeFi投资组合管理平台**，通过复杂的Subgraph索引和Token API集成，为用户提供：
- 🔍 **多协议统一追踪** - 跨越Uniswap、AAVE、Compound等主流DeFi协议
- 📊 **实时收益分析** - 精确计算PnL、APY、无常损失
- 🤖 **智能投资建议** - AI驱动的资产配置优化
- 📈 **深度数据洞察** - 鲸鱼地址策略分析和市场趋势

### 为什么选择这个项目？
- ✅ **高技术复杂性** - 展示Graph Node深度应用
- ✅ **真实市场需求** - DeFi用户急需的工具
- ✅ **商业化潜力** - 清晰的收费模式
- ✅ **技术创新性** - 独特的多链数据融合

---

## 🏗️ 技术架构

### 1. Subgraph 设计架构

#### 核心数据源索引
```yaml
# subgraph.yaml
specVersion: 0.0.4
schema:
  file: ./schema.graphql
dataSources:
  # Uniswap V3 流动性位置
  - kind: ethereum/contract
    name: UniswapV3PositionManager
    source:
      address: "0xC36442b4a4522E871399CD717aBDD847Ab11FE88"
      abi: NonfungiblePositionManager
    mapping:
      kind: ethereum/events
      handlers:
        - event: IncreaseLiquidity(indexed uint256,uint128,uint256,uint256)
          handler: handleIncreaseLiquidity
        - event: DecreaseLiquidity(indexed uint256,uint128,uint256,uint256)
          handler: handleDecreaseLiquidity
        - event: Collect(indexed uint256,address,uint256,uint256)
          handler: handleCollect

  # AAVE 借贷协议
  - kind: ethereum/contract
    name: AavePoolV3
    source:
      address: "0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2"
      abi: Pool
    mapping:
      handlers:
        - event: Supply(indexed address,address,indexed address,uint256,indexed uint16)
          handler: handleSupply
        - event: Withdraw(indexed address,indexed address,indexed address,uint256)
          handler: handleWithdraw
        - event: Borrow(indexed address,address,indexed address,uint256,uint8,uint256,indexed uint16)
          handler: handleBorrow

  # Compound V3
  - kind: ethereum/contract
    name: CompoundV3
    source:
      address: "0xc3d688B66703497DAA19211EEdff47f25384cdc3"
      abi: Comet
    mapping:
      handlers:
        - event: Supply(indexed address,indexed address,uint256)
          handler: handleCompoundSupply
        - event: Withdraw(indexed address,indexed address,uint256)
          handler: handleCompoundWithdraw

templates:
  # 动态LP代币检测
  - kind: ethereum/contract
    name: LPToken
    source:
      abi: ERC20
    mapping:
      handlers:
        - event: Transfer(indexed address,indexed address,uint256)
          handler: handleLPTransfer
```

#### GraphQL Schema 设计
```graphql
# schema.graphql

type User @entity {
  id: ID! # 用户钱包地址
  totalValueLocked: BigDecimal!
  totalPnL: BigDecimal!
  totalRewards: BigDecimal!
  riskScore: BigDecimal!
  
  # 关联实体
  positions: [Position!]! @derivedFrom(field: "user")
  transactions: [Transaction!]! @derivedFrom(field: "user")
  portfolioSnapshots: [PortfolioSnapshot!]! @derivedFrom(field: "user")
}

type Position @entity {
  id: ID! # protocol_user_tokenPair
  user: User!
  protocol: Protocol!
  
  # 代币信息
  token0: Token!
  token1: Token
  
  # 位置数据
  liquidity: BigInt!
  currentValue: BigDecimal!
  entryPrice: BigDecimal!
  entryTimestamp: BigInt!
  
  # 计算字段
  pnl: BigDecimal!
  impermanentLoss: BigDecimal!
  currentAPY: BigDecimal!
  
  # 历史记录
  events: [PositionEvent!]! @derivedFrom(field: "position")
}

type Protocol @entity {
  id: ID! # 协议名称
  name: String!
  type: ProtocolType!
  totalValueLocked: BigDecimal!
  totalUsers: BigInt!
  averageAPY: BigDecimal!
}

type Token @entity {
  id: ID! # 代币合约地址
  symbol: String!
  name: String!
  decimals: Int!
  
  # 从Token API获取的数据将在应用层融合
  currentPrice: BigDecimal!
  priceHistory: [PricePoint!]! @derivedFrom(field: "token")
}

type Transaction @entity {
  id: ID! # 交易哈希
  user: User!
  protocol: Protocol!
  type: TransactionType!
  hash: String!
  timestamp: BigInt!
  blockNumber: BigInt!
  
  # 交易详情
  tokenIn: Token
  tokenOut: Token
  amountIn: BigDecimal!
  amountOut: BigDecimal!
  gasUsed: BigInt!
  gasPrice: BigInt!
}

type PortfolioSnapshot @entity {
  id: ID! # user_timestamp
  user: User!
  timestamp: BigInt!
  totalValue: BigDecimal!
  totalPnL: BigDecimal!
  positionCount: Int!
}

enum ProtocolType {
  LENDING
  DEX
  YIELD_FARMING
  DERIVATIVES
}

enum TransactionType {
  SUPPLY
  WITHDRAW
  SWAP
  STAKE
  CLAIM_REWARDS
}
```

### 2. Token API 集成策略

#### 智能价格数据融合
```typescript
// src/services/TokenPriceService.ts
export class TokenPriceService {
  private tokenAPIClient: TokenAPIClient;
  private subgraphClient: SubgraphClient;

  /**
   * 智能价格获取 - Token API + Subgraph 数据融合
   */
  async getTokenPrice(address: string, timestamp?: number): Promise<TokenPrice> {
    try {
      // 1. 优先从Token API获取实时价格
      const tokenMetadata = await this.tokenAPIClient.getToken(address);
      
      if (timestamp) {
        // 2. 历史价格从Subgraph重建
        const historicalPrice = await this.reconstructHistoricalPrice(
          address, 
          timestamp, 
          tokenMetadata.currentPrice
        );
        return historicalPrice;
      }
      
      return {
        address,
        symbol: tokenMetadata.symbol,
        name: tokenMetadata.name,
        decimals: tokenMetadata.decimals,
        price: tokenMetadata.price,
        timestamp: Date.now(),
        source: 'token_api'
      };
    } catch (error) {
      // 3. 降级到Subgraph数据
      return this.getSubgraphPrice(address, timestamp);
    }
  }

  /**
   * 基于Swap事件重建历史价格
   */
  private async reconstructHistoricalPrice(
    tokenAddress: string, 
    timestamp: number, 
    currentPrice: number
  ): Promise<TokenPrice> {
    const swapQuery = `
      query GetHistoricalSwaps($token: String!, $timestamp: BigInt!) {
        swaps(
          where: { 
            and: [
              { or: [{ token0: $token }, { token1: $token }] }
              { timestamp_lte: $timestamp }
            ]
          }
          orderBy: timestamp
          orderDirection: desc
          first: 10
        ) {
          token0 { id symbol }
          token1 { id symbol }
          amount0
          amount1
          timestamp
        }
      }
    `;

    const swaps = await this.subgraphClient.query(swapQuery, {
      token: tokenAddress,
      timestamp: timestamp.toString()
    });

    // 使用最近的swap计算历史价格
    return this.calculatePriceFromSwaps(tokenAddress, swaps, currentPrice);
  }
}

// Token API客户端
export class TokenAPIClient {
  private baseURL = 'https://api.thegraph.com/token';

  async getToken(address: string): Promise<TokenMetadata> {
    const response = await fetch(`${this.baseURL}/${address}`);
    
    if (!response.ok) {
      throw new Error(`Token API failed: ${response.status}`);
    }

    return response.json();
  }

  async getBatchTokens(addresses: string[]): Promise<TokenMetadata[]> {
    // 批量获取代币信息，提高效率
    const batchRequests = addresses.map(addr => 
      this.getToken(addr).catch(err => null)
    );
    
    const results = await Promise.all(batchRequests);
    return results.filter(Boolean);
  }
}
```

### 3. 高级分析算法

#### 无常损失计算引擎
```typescript
// src/analytics/ImpermanentLossCalculator.ts
export class ImpermanentLossCalculator {
  
  /**
   * 精确计算LP位置的无常损失
   */
  calculateImpermanentLoss(position: LPPosition): ImpermanentLossResult {
    const {
      token0Amount,
      token1Amount,
      entryPrice0,
      entryPrice1,
      currentPrice0,
      currentPrice1,
      liquidityToken
    } = position;

    // 1. 计算如果持有原始代币的价值
    const holdValue = (token0Amount * currentPrice0) + (token1Amount * currentPrice1);
    
    // 2. 计算当前LP价值
    const currentLPValue = this.calculateLPValue(position);
    
    // 3. 无常损失 = LP价值 - 持有价值
    const impermanentLoss = currentLPValue - holdValue;
    const impermanentLossPercent = (impermanentLoss / holdValue) * 100;

    return {
      impermanentLoss,
      impermanentLossPercent,
      holdValue,
      currentLPValue,
      feesEarned: this.calculateFeesEarned(position)
    };
  }

  /**
   * 基于实时价格计算LP价值
   */
  private calculateLPValue(position: LPPosition): number {
    // 使用Uniswap V3的价格范围计算
    if (position.protocol === 'UNISWAP_V3') {
      return this.calculateV3LPValue(position);
    }
    
    // V2 恒定乘积计算
    return this.calculateV2LPValue(position);
  }
}
```

#### 智能收益优化引擎
```typescript
// src/ai/YieldOptimizer.ts
export class YieldOptimizer {
  
  /**
   * AI驱动的投资组合优化建议
   */
  async generateOptimizationStrategy(
    userPortfolio: Portfolio, 
    riskProfile: RiskProfile
  ): Promise<OptimizationStrategy> {
    
    // 1. 获取所有可用的收益机会
    const opportunities = await this.scanYieldOpportunities();
    
    // 2. 分析用户当前配置效率
    const currentEfficiency = this.analyzePortfolioEfficiency(userPortfolio);
    
    // 3. 基于风险偏好生成建议
    const recommendations = this.generateRecommendations(
      userPortfolio,
      opportunities,
      riskProfile,
      currentEfficiency
    );

    return {
      currentAPY: currentEfficiency.weightedAPY,
      optimizedAPY: recommendations.projectedAPY,
      potentialGain: recommendations.projectedAPY - currentEfficiency.weightedAPY,
      rebalancingSteps: recommendations.steps,
      riskScore: recommendations.riskScore,
      gasCostEstimate: recommendations.gasCost
    };
  }

  /**
   * 扫描DeFi生态系统寻找收益机会
   */
  private async scanYieldOpportunities(): Promise<YieldOpportunity[]> {
    const protocolQueries = [
      this.queryAaveRates(),
      this.queryCompoundRates(), 
      this.queryUniswapLPRates(),
      this.queryCurveRates()
    ];

    const results = await Promise.all(protocolQueries);
    return results.flat().sort((a, b) => b.apy - a.apy);
  }
}
```

---

## 🚀 核心功能特性

### 1. 多协议统一视图
- **支持协议**: Uniswap V2/V3, AAVE V2/V3, Compound V3, Curve, Balancer
- **实时同步**: 所有头寸的实时价值更新
- **历史追踪**: 完整的投资历史和收益曲线

### 2. 智能分析工具
- **无常损失监控**: 精确计算和预警
- **APY追踪**: 跨协议收益率比较
- **风险评估**: 智能合约风险和市场风险评分
- **税务报告**: 自动生成DeFi交易记录

### 3. 社交投资洞察
- **鲸鱼地址追踪**: 分析大户投资策略
- **趋势发现**: 识别新兴DeFi机会
- **策略复制**: 一键跟投成功策略

### 4. 高级用户界面
```typescript
// 前端组件示例
const PortfolioDashboard: React.FC = () => {
  const { user, loading } = useUser();
  const { portfolio } = usePortfolio(user?.address);
  const { tokenPrices } = useTokenAPI(portfolio?.tokens);

  return (
    <DashboardLayout>
      <PortfolioOverview 
        totalValue={portfolio.totalValue}
        pnl={portfolio.totalPnL}
        apy={portfolio.weightedAPY}
      />
      
      <PositionsGrid 
        positions={portfolio.positions}
        prices={tokenPrices}
        onOptimize={handleOptimize}
      />
      
      <AnalyticsCharts 
        historicalData={portfolio.snapshots}
        impermanentLoss={portfolio.impermanentLoss}
      />
      
      <RecommendationsPanel 
        suggestions={portfolio.optimizationSuggestions}
      />
    </DashboardLayout>
  );
};
```

---

## 💰 商业模式

### 分层定价策略
```typescript
const PRICING_TIERS = {
  FREE: {
    monthlyQueries: 100000, // 符合竞赛要求
    features: [
      '基础投资组合追踪',
      '简单PnL计算', 
      '3个协议支持'
    ]
  },
  
  PRO: {
    price: 29, // USD/月
    monthlyQueries: 1000000,
    features: [
      '所有协议支持',
      '高级分析工具',
      '无常损失监控',
      'API访问权限',
      '税务报告生成'
    ]
  },
  
  ENTERPRISE: {
    price: 199, // USD/月  
    unlimitedQueries: true,
    features: [
      '白标解决方案',
      '定制分析面板',
      '专属客户支持',
      '高级API限额',
      '机构级功能'
    ]
  }
};
```

### 收入预测
- **Year 1**: $50K ARR (500 付费用户)
- **Year 2**: $200K ARR (2000 付费用户)  
- **Year 3**: $500K ARR (机构客户拓展)

---

## 📅 开发计划

### Phase 1: 基础架构 (周 1-2)
- [x] Graph Node 环境搭建
- [ ] 核心 Subgraph 开发
- [ ] Token API 集成
- [ ] 基础数据模型设计

### Phase 2: 核心功能 (周 2-3)
- [ ] 多协议数据索引
- [ ] 实时价格计算引擎
- [ ] 无常损失计算
- [ ] 用户投资组合API

### Phase 3: 前端开发 (周 3-4)
- [ ] React/Next.js 仪表板
- [ ] 数据可视化组件
- [ ] 用户认证系统
- [ ] 响应式设计

### Phase 4: 高级功能 (周 4-5)
- [ ] AI收益优化
- [ ] 社交投资功能
- [ ] 实时通知系统
- [ ] 移动端适配

### Phase 5: 优化部署 (周 5-6)
- [ ] 性能优化
- [ ] 安全审计
- [ ] 产品部署
- [ ] 文档完善

---

## 🎖️ 竞争优势

### 技术优势
1. **深度Graph整合** - 充分利用现有Graph Node经验
2. **多链架构** - 支持跨链投资组合管理
3. **实时计算** - 毫秒级的价值更新
4. **AI驱动** - 智能投资建议算法

### 市场优势  
1. **真实需求** - 解决DeFi用户痛点
2. **用户体验** - 直观的数据展示
3. **功能完整** - 从追踪到优化的完整链条
4. **可扩展性** - 易于添加新协议支持

### 商业优势
1. **清晰商业模式** - 多层次收费策略
2. **高粘性用户** - 投资工具天然高粘性
3. **企业市场** - 可扩展到机构客户
4. **数据价值** - 聚合的DeFi数据具有独立价值

---

## 📊 成功指标

### 技术指标
- **Subgraph同步速度**: < 10秒延迟
- **查询性能**: < 500ms 响应时间
- **数据准确性**: 99.9% 计算精度
- **系统可用性**: 99.5% 正常运行时间

### 产品指标
- **用户留存**: 30天留存率 > 40%
- **功能使用**: 核心功能使用率 > 60%
- **用户满意度**: NPS > 50
- **付费转化**: 免费到付费转化率 > 5%

### 商业指标
- **用户增长**: 月增长率 > 20%
- **收入增长**: ARR 季度增长 > 50%
- **CAC回收**: < 6个月
- **LTV/CAC**: > 3:1

---

## 🔗 关键资源

### 开发资源
- **Graph CLI**: `npm install -g @graphprotocol/graph-cli`
- **Subgraph Studio**: https://thegraph.com/studio/
- **Token API文档**: https://thegraph.com/docs/token-api/
- **样例代码**: 本项目 `examples/` 目录

### 外部服务
- **价格数据**: CoinGecko API (备用)
- **推送通知**: Web Push API
- **用户认证**: WalletConnect
- **数据存储**: IPFS + PostgreSQL

### 监控工具
- **APM**: DataDog
- **错误追踪**: Sentry  
- **分析**: Mixpanel
- **性能**: Lighthouse CI

---

## 🚀 立即开始

### 快速启动
```bash
# 1. 克隆项目
git clone https://github.com/yourusername/defi-portfolio-tracker
cd defi-portfolio-tracker

# 2. 安装依赖
npm install

# 3. 配置环境变量
cp .env.example .env
# 编辑 .env 文件添加必要的API密钥

# 4. 启动Graph Node (使用现有配置)
cd graph-node
cargo run -p graph-node --release -- \
  --postgres-url $POSTGRES_URL \
  --ethereum-rpc mainnet:archive,traces:$ETH_RPC_URL \
  --ipfs 127.0.0.1:5001

# 5. 部署Subgraph
cd ../subgraph
graph deploy --studio defi-portfolio-tracker

# 6. 启动前端
cd ../frontend
npm run dev
```

### 下一步
1. 📝 完善Subgraph schema定义
2. 🔧 实现核心数据索引逻辑  
3. 🎨 设计用户界面原型
4. 🧪 编写单元测试和集成测试
5. 📊 配置监控和分析

---

*本项目旨在参加The Graph Protocol竞赛，展示Subgraph和Token API的创新应用，为DeFi生态系统提供真正有价值的工具。*

**项目状态**: 🚧 开发中 | **预计完成**: 3-4周 | **目标奖项**: 🥇 第一名 

## 🛑 How to Stop Docker Services

Here are the different ways to stop Docker:

### **Method 1: Stop Docker Compose Services (Recommended)**

**In PowerShell, navigate to the subgraph directory and stop:**
```powershell
cd "D:\cursor\the graph\devmatch-workshop\subgraph"
docker-compose down
```

### **Method 2: Stop All Running Containers**
```powershell
<code_block_to_apply_changes_from>
```

### **Method 3: Force Stop Everything**
```powershell
# Stop and remove all containers
docker-compose down --volumes --remove-orphans

# Or nuclear option - stop everything
docker stop $(docker ps -aq)
```

### **Method 4: Check What's Running First**
```powershell
# See what's currently running
docker ps

# See the status of your compose services
docker-compose ps
```

### **Method 5: Stop Docker Desktop Entirely**
- Click on the Docker Desktop icon in your system tray
- Right-click and select "Quit Docker Desktop"

## 🎯 Recommended Approach

**Just run these commands in PowerShell:**

1. **Navigate to your project:**
   ```powershell
   cd "D:\cursor\the graph\devmatch-workshop\subgraph"
   ```

2. **Stop the services gracefully:**
   ```powershell
   docker-compose down
   ```

3. **Verify everything is stopped:**
   ```powershell
   docker ps
   ```

This will cleanly stop the Graph Node, PostgreSQL, and IPFS containers while preserving your data.

Would you like me to help you with any specific reason for stopping Docker, or do you want to restart it with different configurations? 