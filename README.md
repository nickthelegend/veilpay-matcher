# Veilpay Matcher

**Off-chain matching engine for a privacy-preserving dark pool DEX, backed by zero-knowledge order commitments.**

> Users submit encrypted, committed orders; the matcher crosses them off-chain and settles matches on-chain — order details (price, size, side) stay hidden throughout.

## Overview

Veilpay Matcher (internally *DarkBook*) is the relayer/matcher tier of a confidential decentralized exchange. Traders publish an on-chain **commitment** to an order along with a ZK proof that the order is well-formed and funded, then relay the encrypted order body to the matcher over an ECDH-secured channel. The matcher runs an in-memory limit orderbook, finds crossing orders using price-time priority, executes at the **midpoint** of the two limit prices, and submits batched settlement transactions to the chain.

The repository contains two parts: the **TypeScript matcher service** (orderbook, encryption, prover integration, settler, indexer) and the **Noir ZK circuits** that back order validity, match validity, and balance updates. Proof generation in the service is currently scaffolded with placeholder proofs (MVP) while the Noir circuits define the full constraint system.

## Features

- **In-memory limit orderbook** — per-pair buy/sell books with price-time priority, binary-search insertion, continuous matching, and partial fills (`src/orderbook.ts`).
- **Midpoint settlement** — crossing orders execute at `(bid + ask) / 2` rather than at either side's limit.
- **Encrypted order relay** — ECDH key exchange (secp256k1) + HKDF-SHA256 + AES-256-GCM so order contents are only visible to the matcher (`src/encryption.ts`).
- **ZK circuits in Noir** — three circuits: `order_commitment` (commitment correctness, nullifier, balance sufficiency via Merkle proof), `match_proof` (opposite sides, price crossing, valid fill, midpoint pricing), and `balance_update` (post-settlement balances, no underflow, new Merkle root).
- **Pedersen commitments & nullifiers** — domain-separated hashing to bind order parameters and prevent double-submission; Merkle balance tree of depth 20 (~1M leaves).
- **Batched on-chain settlement** — a periodic batch loop submits `settleMatch` transactions via viem to an EVM engine contract (`src/settler.ts`).
- **Event indexer + WebSocket API** — polls `OrderSubmitted` / `OrderCancelled` / `MatchSettled` events, maintains shadow state, and serves snapshots and live updates to a frontend (`src/indexer.ts`).
- **Test coverage** — Vitest suite for orderbook sorting, matching, partial fills, and cancellation (`tests/orderbook.test.ts`).

## Tech Stack

- **Language / runtime:** TypeScript (ES2022, ESM), Node.js
- **Chain client:** [viem](https://viem.sh) — configured for Conflux eSpace Testnet (chain id 71)
- **ZK:** [Noir](https://noir-lang.org) circuits (`@noir-lang/noir_js`) with the [`@aztec/bb.js`](https://www.npmjs.com/package/@aztec/bb.js) UltraHonk backend
- **Crypto:** `@noble/curves`, `@noble/ciphers`, `@noble/hashes` (secp256k1 ECDH, AES-GCM, HKDF)
- **Networking:** `ws` (WebSocket server for the indexer API)
- **Config:** `dotenv`
- **Tooling:** `tsx`, `vitest`, `typescript`

## Getting Started

### Prerequisites
- Node.js 18+
- [Nargo](https://noir-lang.org) (only if you want to compile the Noir circuits)

### Install & run

```bash
# clone
git clone https://github.com/nickthelegend/veilpay-matcher.git
cd veilpay-matcher

# install dependencies
npm install

# configure environment
cp .env.example .env
# then edit .env: RPC URL, matcher private key, engine/vault addresses, ECDH key

# run in watch mode (dev)
npm run dev

# or build and run
npm run build
npm run start

# run tests
npm test          # watch mode
npm run test:run  # single run
```

### Compile the circuits (optional)

```bash
cd circuits
nargo compile
```

## Project Structure

```
veilpay-matcher/
├── src/
│   ├── index.ts        # Service entry point; wires components + batch loop
│   ├── orderbook.ts    # In-memory price-time-priority matching engine
│   ├── encryption.ts   # ECDH + AES-256-GCM encrypted order relay
│   ├── prover.ts       # ZK proof-generation service (Noir.js + bb.js)
│   ├── settler.ts      # Batched on-chain settlement via viem
│   ├── indexer.ts      # Event indexer + WebSocket API
│   └── types.ts        # Shared type definitions
├── circuits/
│   ├── src/
│   │   ├── main.nr             # Circuit entry point
│   │   ├── order_commitment.nr # Order validity + balance proof
│   │   ├── match_proof.nr      # Valid-match proof
│   │   ├── balance_update.nr   # Post-settlement balance proof
│   │   └── lib/                # merkle, pedersen, utils helpers
│   ├── Nargo.toml
│   └── Prover.toml
├── tests/
│   └── orderbook.test.ts
├── .env.example
├── package.json
└── tsconfig.json
```

---

Built by [**nickthelegend**](https://github.com/nickthelegend) · [nickthelegend.tech](https://nickthelegend.tech)
