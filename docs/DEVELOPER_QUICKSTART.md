# XanePay — Developer Quickstart & Onboarding Guide

Welcome to the XanePay engineering repository! This guide will walk you through setting up your local environment, running the services with Docker, applying database migrations, and understanding the architecture.

---

## 1. System Architecture & Component Inventory

XanePay is structured as a high-performance, modular fintech and crypto orchestration platform comprising three core backend services and a mobile/web frontend:

```
                            ┌─────────────────────────────────────────┐
                            │        XanePay Client Applications      │
                            │   (Mobile Expo App / Web Frontend)      │
                            └────────────────────┬────────────────────┘
                                                 │
                                                 ▼
                            ┌─────────────────────────────────────────┐
                            │    1. XaneApp Main App (Port 3000)      │
                            │  Identity, Passkeys, Wallets, Dex Swaps,│
                            │  Bridge Routing, Gasless Contract Calls │
                            └────────────┬──────────────┬─────────────┘
                                         │              │
                   (Internal HMAC / RPC) │              │ (Internal API)
                                         ▼              ▼
┌─────────────────────────────────────────┐    ┌─────────────────────────────────────────┐
│   2. Treasury Engine (Port 3001)        │    │   3. Conversion Backend (Port 8080)     │
│ Double-Entry Ledger (ACID), Bank Rails  │    │ Fiat-Crypto Ledger Service, Identity &  │
│ (Fincra NGN, OnSwitch), Payout Worker   │    │ Reconciliation Scheduler                │
└────────────────────┬────────────────────┘    └────────────────────┬────────────────────┘
                     │                                              │
                     └──────────────────────┬───────────────────────┘
                                            │
                                            ▼
                     ┌──────────────────────────────────────────────┐
                     │          Shared Infrastructure Layer         │
                     │  ● PostgreSQL 16 (Port 5433:5432) — ACID DB  │
                     │  ● Redis 7 (Port 6379) — Jobs, Caches, Locks │
                     └──────────────────────────────────────────────┘
```

### Component Breakdown

| Directory | Name | Port | Database | Primary Responsibility |
|---|---|---|---|---|
| [`XaneApp/`](file:///c:/Projects/XanePay/XaneApp) | **Main App** | `3000` | PostgreSQL / Firebase (Switchable) | User auth, passkeys, Web3Auth/MPC, wallets, DEX swaps (Uniswap/SaucerSwap), cross-chain bridges (Wormhole/Hashport), contract interactions |
| [`XaneApp/engine/`](file:///c:/Projects/XanePay/XaneApp/engine) | **Treasury Engine** | `3001` | PostgreSQL 16 (`xanepay_engine`) | Fiat↔crypto conversion pipeline, locked quotes, double-entry immutable ledger, BullMQ payout queue, bank rails (Fincra, OnSwitch) |
| [`XaneApp/backend/`](file:///c:/Projects/XanePay/XaneApp/backend) | **Conversion Backend** | `8080` | PostgreSQL 16 (`xanepay_backend`) | Identity reconciliation, fiat ledger sync, HMAC service bridge |
| [`XaneApp/frontend/`](file:///c:/Projects/XanePay/XaneApp/frontend) | **Client App** | `8081` | Static / Expo Client | React Native Expo application for mobile and web |

---

## 2. Prerequisites

Ensure your development machine has the following tools installed:

1. **Node.js**: Version `20.x` or higher (`node -v`)
2. **Docker Desktop**: Version `24.x+` with Docker Compose enabled (`docker --version`)
3. **Git**: Version `2.x+`
4. **npm** / **pnpm**: Node package managers

---

## 3. Step-by-Step Local Setup

### Step 1: Clone and Navigate to the Repository

```bash
git clone https://github.com/xaneApp/XaneApp.git
cd XanePay
```

---

### Step 2: Spin Up Infrastructure (PostgreSQL & Redis)

We provide Docker Compose configurations for instant zero-config local development.

From the `XaneApp/` directory:

```bash
cd XaneApp

# Start PostgreSQL 16 and Redis 7 in the background
npm run db:up
```

> [!NOTE]
> PostgreSQL is mapped to host port **`5433`** (`5433:5432`) to avoid conflicts if you already have a local PostgreSQL instance running on default port `5432`.
> Redis is mapped to port **`6379`**.

To verify containers are healthy:
```bash
docker compose ps
```

---

### Step 3: Configure Environment Variables

Create your local `.env` file in `XaneApp/`:

```bash
# In XaneApp/
cp .env.example .env
```

Key environment configurations:

```env
# Server Configuration
NODE_ENV=development
PORT=3000
APP_NAME=XaneWallet-Backend

# Database Configuration (PostgreSQL Mode)
DATABASE_PROVIDER=postgres
DATABASE_URL=postgresql://xaneapp:xaneapp_dev@localhost:5433/xaneapp
DATABASE_SSL=false
DB_POOL_MIN=2
DB_POOL_MAX=10

# Redis Cache
REDIS_HOST=localhost
REDIS_PORT=6379
ENABLE_REDIS_CACHING=false

# Security
JWT_SECRET=your_super_secret_local_dev_jwt_key_at_least_32_chars
```

---

### Step 4: Run Database Migrations

Apply the Knex schema migrations to create all database tables and indexes:

```bash
# In XaneApp/
npm run migrate
```

This creates the following tables:
- `users`: User profiles and hashed credentials
- `wallets`: User balance balances and currency records
- `transactions`: Blockchain transaction records and confirmations
- `contract_operations`: State machine for smart contract executions
- `abi_registry`: Contract ABI caching and verification
- `metrics`: Operation latency and performance logging
- `alerts`: Anomaly alerts and operational notifications

To check migration status at any time:
```bash
npm run migrate:status
```

---

### Step 5: Start the Services

#### Option A: Running XaneApp Main Service (Port 3000)

```bash
# In XaneApp/
npm run dev
```
The server will boot and log:
```
[SERVER] XaneApp Backend started successfully
[SERVER] Listening on http://localhost:3000
[SERVER] Database Provider: POSTGRES
```

Interactive Swagger API Documentation is available at:
👉 **`http://localhost:3000/api-docs`**

#### Option B: Running Treasury Engine (Port 3001)

Open a separate terminal window:
```bash
cd XaneApp/engine
cp .env.example .env
npm install
npm run migrate
npm run dev
```
The Engine will start on **`http://localhost:3001`**.

---

## 4. Switching Database Backends

XaneApp includes a switchable multi-database provider. You can switch backends by changing a single line in `XaneApp/.env`:

### Switching to PostgreSQL:
```env
DATABASE_PROVIDER=postgres
DATABASE_URL=postgresql://xaneapp:xaneapp_dev@localhost:5433/xaneapp
```

### Switching to Firebase Firestore:
```env
DATABASE_PROVIDER=firebase
```
*(Firebase Admin will use `src/config/serviceAccountKey.json` for credentials)*

---

## 5. Running Tests & Verifications

Run the automated test suites:

```bash
# Run unit tests
npm run test:unit

# Run full type check across all TypeScript source files
npx tsc --noEmit

# Compile production bundle to dist/
npm run build
```

---

## 6. Useful npm Scripts Reference

| Script | Command | Purpose |
|---|---|---|
| `npm run dev` | `ts-node-dev src/index.ts` | Start development server with live reload |
| `npm run build` | `tsc` | Compile TypeScript into `dist/` |
| `npm run start` | `node dist/index.js` | Start compiled production server |
| `npm run db:up` | `docker compose up -d` | Start PostgreSQL & Redis containers |
| `npm run db:down` | `docker compose down` | Stop containers |
| `npm run migrate` | `knex migrate:latest` | Run latest database migrations |
| `npm run migrate:rollback` | `knex migrate:rollback` | Roll back the latest migration batch |
| `npm run migrate:status` | `knex migrate:status` | View status of database migrations |
| `npm test` | `jest` | Run full test suite |

---

## 7. Troubleshooting & FAQ

### Q: Docker reports `port 5433 already in use`
Check if another service or background container is bound to 5433:
```bash
# On Windows (PowerShell)
Get-NetTCPConnection -LocalPort 5433
# On Linux/macOS
lsof -i :5433
```
Either stop the conflicting process or change the left-side port in `docker-compose.yml` (`5434:5432`) and update `DATABASE_URL` in `.env`.

### Q: Knex migrations fail to connect
Make sure the Docker container is running and healthy:
```bash
docker compose ps
docker compose logs postgres
```

### Q: Swagger UI returns 404
Swagger is mounted at `http://localhost:3000/api-docs`. Make sure you are navigating to `/api-docs` (with hyphen) and that `npm run dev` is running.
