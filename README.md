# Blockchain-Based Supply Chain Provenance System

> Transparent, Tamper-Evident, and End-to-End Traceable — Arizona State University · CSE 540 Engineering Blockchain Applications

[![Solidity](https://img.shields.io/badge/Solidity-0.8.20-363636?logo=solidity)](https://soliditylang.org)
[![Hardhat](https://img.shields.io/badge/Hardhat-2.19-yellow)](https://hardhat.org)
[![Tests](https://img.shields.io/badge/Tests-60%2B_passing-brightgreen)](#testing)
[![Coverage](https://img.shields.io/badge/Coverage->95%25-brightgreen)](#testing)
[![License](https://img.shields.io/badge/License-MIT-blue)](LICENSE)

**Authors:** Hrishikesh Magadum · Aditya Sahasrabuddhe · Atharva Deshpande · Sahil Pawar — Arizona State University

---

## Table of Contents

- [Overview](#overview)
- [Why Blockchain for Supply Chain?](#why-blockchain-for-supply-chain)
- [Architecture](#architecture)
- [Smart Contract](#smart-contract)
- [Features](#features)
- [Gas Costs & Performance](#gas-costs--performance)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [API Reference](#api-reference)
- [Role Dashboards](#role-dashboards)
- [Security](#security)
- [Deployment](#deployment)

---

## Overview

This system addresses the **$1.82 trillion global counterfeiting problem** by using Ethereum smart contracts to create an immutable, decentralized ledger for end-to-end product tracking. Every ownership transfer, status update, and regulatory verification is recorded on-chain — permanently auditable by anyone.

**Key numbers:**
- `667` lines of production Solidity
- `60+` automated test cases, `>95%` code coverage
- `~185,234` gas per product registration (~$7.41 on mainnet)
- `10` RESTful API endpoints with `<500ms` response times
- `7` product lifecycle states tracked on-chain

---

## Why Blockchain for Supply Chain?

| Feature | Traditional DB | This System |
|---|---|---|
| Speed | <100ms | 2–15s |
| Cost | <$0.001 | $0.02–$7.41 |
| **Immutability** | ❌ | ✅ |
| **Transparency** | ❌ | ✅ |
| **Auditability** | Manual | Automatic |
| Consumer verification | ❌ | ✅ QR code |
| Trust model | Centralized | Trustless |

> For high-value, multi-party supply chains requiring verifiability and regulatory compliance, blockchain is the clear choice. For high-frequency, low-value operations, a hybrid approach (off-chain data + on-chain event logs) offers the best trade-offs.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     React Frontend (web/)                        │
│   Producer Dashboard · Distributor · Retailer · Regulator        │
│   Consumer QR Verification · Real-time Status Updates           │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTP/REST
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│          Node.js + Express API Backend (api/ — port 3001)        │
│   Input validation · RPC calls · Exception handling              │
│   express-validator · Morgan · CORS                             │
└──────────────────────────┬──────────────────────────────────────┘
                           │ ethers.js v6 (JSON-RPC)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│           Ethereum Node (Hardhat local — port 8545)              │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              SupplyChainProvenance.sol (667 lines)        │   │
│  │                                                          │   │
│  │  AccessControl · Pausable · ReentrancyGuard (OZ v5)      │   │
│  │                                                          │   │
│  │  registerProduct()   transferProduct()                   │   │
│  │  updateProductStatus()   addVerification()               │   │
│  │  getProduct()   getProductsByOwner()                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

> **Data flow diagram** — see `docs/images/architecture.png`

---

## Smart Contract

### Product Lifecycle States

```
Created → Dispatched → InTransit → Received → Delivered → Verified
                                                              ↓
                                                          Exception (any role)
```

The `ProductStatus` enum tracks 7 states. Every transition emits an on-chain event — building a permanent, queryable audit trail without expensive array storage.

### Key Events

```solidity
event ProductCreated(uint256 indexed productId, address indexed producer, string name, uint256 timestamp);
event OwnershipTransferred(uint256 indexed productId, address indexed from, address indexed to);
event StatusUpdated(uint256 indexed productId, ProductStatus newStatus, uint256 timestamp);
event ProductVerified(uint256 indexed productId, address indexed regulator, string certBody);
```

### Permission Matrix

| Function | Producer | Distributor | Retailer | Regulator | Public |
|---|:---:|:---:|:---:|:---:|:---:|
| `registerProduct()` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `transferProduct()` | owner | owner | owner | ❌ | ❌ |
| `updateProductStatus()` | ✅ | ✅ | ✅ | ❌ | ❌ |
| `addVerification()` | ❌ | ❌ | ❌ | ✅ | ❌ |
| `getProduct()` | ✅ | ✅ | ✅ | ✅ | ✅ |

An emergency `pause()` mechanism locks all state-changing functions during exploits or upgrades.

---

## Features

- **Immutable product registration** — on-chain records cannot be altered or deleted
- **7-state lifecycle tracking** — every status transition is a permanent on-chain event
- **5-role RBAC** — OpenZeppelin `AccessControl` with Producer, Distributor, Retailer, Regulator, Consumer
- **QR code verification** — consumers verify product authenticity without an account
- **Regulatory certification** — regulators add on-chain verifications for compliance
- **Reentrancy protection** — all state-altering functions guarded by `ReentrancyGuard`
- **Gas optimization** — event-based architecture reduced gas costs by 96% vs. array storage
- **Sepolia testnet ready** — one-command deployment via Alchemy/Infura

---

## Gas Costs & Performance

### Smart Contract Operations

| Function | Gas Used | Cost @ $2000 ETH (20 gwei) |
|---|---|---|
| `registerProduct()` | ~185,234 | ~$7.41 |
| `transferProduct()` | ~79,812 | ~$3.19 |
| `addVerification()` | ~68,923 | ~$2.76 |
| `updateProductStatus()` | ~52,441 | ~$2.10 |
| `emit event` | ~375 | ~$0.01 |
| View query | 0 | Free |

> On **Polygon L2**, costs drop ~100× (~$0.001–$0.07 per operation).

### API & Frontend Performance

| Endpoint / Action | Response Time |
|---|---|
| `GET /products/:id` | 120ms |
| `GET /products/:id/history` | 450ms |
| `POST /products` (create) | 2,100ms |
| `POST /products/:id/status` | 2,100ms |
| Initial frontend load | 1.8s |
| QR code generation | <50ms |

### Functional Test — Coffee Product Journey

| Step | Function | Gas Used | Event Emitted |
|---|---|---|---|
| 1 | Register product | 185,234 | `ProductCreated` |
| 2 | Update → Dispatched | 52,441 | `StatusUpdated` |
| 3 | Transfer to Distributor | 79,812 | `OwnershipTransferred` |
| 4 | Update → InTransit | 52,441 | `StatusUpdated` |
| 5 | Transfer to Retailer | 79,812 | `OwnershipTransferred` |
| 6 | Update → Received | 52,441 | `StatusUpdated` |
| 7 | Update → Delivered | 52,441 | `StatusUpdated` |
| 8 | Add verification | 68,923 | `ProductVerified` |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Smart contracts | Solidity 0.8.20, OpenZeppelin 5.0.0 |
| Blockchain dev | Hardhat 2.19.0 |
| Blockchain interaction | ethers.js 6.9.0 |
| Backend | Node.js 16+, Express 4.18.2 |
| Frontend | React 18.2.0, React Router DOM, Axios |
| QR codes | React QR Code |
| Testing | Mocha, Chai, Hardhat Toolbox |
| Code quality | Solhint, Prettier + prettier-plugin-solidity |
| Deployment | Alchemy/Infura (Sepolia), Vercel/Netlify (frontend) |

---

## Project Structure

```
Blockchain-Based-Supply-Chain-Provenance-System/
├── contracts/
│   └── SupplyChainProvenance.sol   # Core contract (667 lines, 24 functions)
├── scripts/
│   └── deploy.js                  # Deployment script
├── test/                          # 60+ Mocha/Chai test cases (>95% coverage)
├── api/                           # Express.js REST API (10 endpoints)
│   ├── routes/
│   └── index.js
├── web/                           # React SPA (6 pages, role-based dashboards)
│   └── src/
├── docs/
│   └── images/                    # Architecture diagrams
├── hardhat.config.js              # Network config (local + Sepolia)
├── package.json
├── env.template                   # Environment variable template
└── README.md
```

---

## Getting Started

### Prerequisites

- Node.js v16+, npm v8+
- MetaMask (for testnet)

### 1. Clone & install

```bash
git clone https://github.com/hrishikeshm12/Blockchain-Based-Supply-Chain-Provenance-System.git
cd Blockchain-Based-Supply-Chain-Provenance-System
npm run install:all
```

### 2. Configure environment

```bash
cp env.template .env
```

```env
PRIVATE_KEY=your_wallet_private_key
SEPOLIA_RPC_URL=https://sepolia.infura.io/v3/your_project_id
ETHERSCAN_API_KEY=your_key    # optional — for contract verification
```

### 3. Run locally (3 terminals)

```bash
# Terminal 1 — Start local blockchain (20 pre-funded test accounts)
npm run node

# Terminal 2 — Deploy contract + start API
npm run deploy:local
npm run api

# Terminal 3 — Start React frontend
npm run frontend
```

Open `http://localhost:3000`

---

## API Reference

| Method | Endpoint | Role | Description |
|---|---|---|---|
| `POST` | `/api/products` | Producer | Register a new product |
| `GET` | `/api/products/:id` | All | Get product details |
| `GET` | `/api/products/:id/history` | All | Full lifecycle history |
| `POST` | `/api/products/:id/status` | P/D/R | Update product status |
| `POST` | `/api/products/:id/transfer` | P/D/R | Transfer ownership |
| `POST` | `/api/products/:id/verify` | Regulator | Add certification |
| `GET` | `/api/products/owner/:address` | All | Products by owner |

---

## Role Dashboards

| Role | Capabilities |
|---|---|
| **Producer** | Register products, initiate dispatch, view QR codes |
| **Distributor** | Accept transfers, update transit status, forward products |
| **Retailer** | Mark received/delivered, track inventory |
| **Regulator** | Add certifications, verify authenticity, compliance reporting |
| **Consumer** | Scan QR code → view complete product history (no account needed) |

---

## Security

All security tests passed ✅

| Attack Vector | Protection | Result |
|---|---|---|
| Unauthorized registration | `PRODUCER_ROLE` check | Reverted |
| Unauthorized status update | Role + ownership check | Reverted |
| Reentrancy attack | `ReentrancyGuard` | Reverted |
| Role escalation | OpenZeppelin `AccessControl` | Reverted |
| Transfer to zero address | Input validation | Reverted |
| System exploit | `pause()` emergency stop | Working |

---

## Deployment

### Local (default)

```bash
npm run deploy:local   # Deploys to Hardhat node at localhost:8545
```

### Sepolia Testnet

```bash
npm run deploy:sepolia   # Requires funded wallet + RPC URL in .env
```

### NPM Scripts Reference

```bash
npm run compile          # Compile Solidity
npm run test             # Run 60+ test cases
npm run test:coverage    # Coverage report
npm run lint             # Solhint linting
npm run format           # Prettier formatting
npm run clean            # Remove build artifacts
npm run setup:demo       # Configure demo environment
```

---

## Citation

```
Sahasrabuddhe, A., Deshpande, A., Magadum, H., & Pawar, S.
"Blockchain-Based Supply Chain Provenance System: Transparent, Tamper-Evident, and End-to-End Traceable."
Arizona State University, CSE 540 Engineering Blockchain Applications, 2025.
```

---

## License

MIT
