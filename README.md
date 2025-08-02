# The Graph NFT Tracker

A NFT tracker project built on The Graph Protocol.

## 📁 Project Structure

```
the-graph/
├── nft-tracker-subgraph/    # Main subgraph project
├── nft-subgraph-frontend/   # Frontend interface
├── graph-node/              # Graph node
└── package.json             # Root project configuration
```

## 🚀 Quick Start

### Install Dependencies
```bash
npm install
```

### Build Subgraph
```bash
cd nft-tracker-subgraph
npm install
graph codegen
graph build
```

### Deploy to Local Graph Node
```bash
graph create nft-tracker --node http://localhost:8020
graph deploy nft-tracker --node http://localhost:8020 --ipfs http://localhost:5001
```

## 📋 Included Smart Contracts

1. **GenArt721Core** - Art Blocks generative art NFT contract
   - Address: `0xa7d8d9ef8D8Ce8992Df33D8b8CF4Aebabd5bD270`
   - Start Block: 11437151

2. **FNDNFT721** - Foundation NFT contract  
   - Address: `0xe7C29cba93ef8017C7824DD0f25923c38d08065c`
   - Start Block: 14087953

## 🛠️ Development Notes

All source code and configuration files are included. The `node_modules`, `build`, `generated` directories can be regenerated using the commands above.

## 📊 Query Example

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