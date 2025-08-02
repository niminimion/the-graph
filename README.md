# The Graph NFT Tracker

这是一个基于The Graph Protocol的NFT追踪器项目。

## 📁 项目结构

```
the-graph/
├── nft-tracker-subgraph/    # 主要的subgraph项目
├── nft-subgraph-frontend/   # 前端界面
├── graph-node/              # Graph节点
└── package.json             # 根项目配置
```

## 🚀 快速开始

### 安装依赖
```bash
npm install
```

### 构建Subgraph
```bash
cd nft-tracker-subgraph
npm install
graph codegen
graph build
```

### 部署到本地Graph节点
```bash
graph create nft-tracker --node http://localhost:8020
graph deploy nft-tracker --node http://localhost:8020 --ipfs http://localhost:5001
```

## 📋 包含的智能合约

1. **GenArt721Core** - Art Blocks生成艺术NFT合约
   - 地址: `0xa7d8d9ef8D8Ce8992Df33D8b8CF4Aebabd5bD270`
   - 开始区块: 11437151

2. **FNDNFT721** - Foundation NFT合约  
   - 地址: `0xe7C29cba93ef8017C7824DD0f25923c38d08065c`
   - 开始区块: 14087953

## 🛠️ 开发说明

所有源代码和配置文件都已包含。`node_modules`、`build`、`generated` 等目录可以通过上述命令重新生成。

## 📊 查询示例

```graphql
{
  nfts(first: 10) {
    id
    tokenId
    owner
    creator
    tokenURI
  }
}
```