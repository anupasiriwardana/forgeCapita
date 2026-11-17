# forgeCapita-v2: System Architecture & Technology Stack

This document outlines the complete technical architecture, technology choices, and deployment strategy for the forgeCapita-v2 ecosystem. The platform is designed as a unified TypeScript/Web3 stack, combining milestone-gated smart contracts with a highly performant off-chain data layer.

---

## 1. Blockchain & Smart Contract Layer

The foundational layer of forgeCapita-v2 consists of immutable smart contracts handling all capital formation, governance, secondary trading, and revenue distribution.

*   **Network:** Ethereum Sepolia Testnet (Public, free-tier execution).
*   **Language:** Solidity (^0.8.20).
*   **Core Contracts:**
    *   `ForgeCapitaFactory`: Master deployment engine that spins up ecosystem contracts for each startup.
    *   `MilestoneEscrow`: Holds crowdfunded ETH and developer token equity. Enforces Cliff Vesting (founder tokens unlock only at 100% completion) and manages the 120-day deadlock liquidation refund.
    *   `ProjectToken`: A lightweight ERC-20 token representing fractional SaaS equity.
    *   `DividendDistributor`: The staking vault. Implements a High-Water Mark algorithm to distribute ETH yield accurately to staked investors.
    *   `RevenueRouter`: Payment splitter for SaaS subscriptions (24% Developer OpEx, 1% Treasury, 75% Staking Yield). Includes a zero-staker fallback to prevent payment reverts.
    *   `P2PExchange`: Native, non-custodial OTC order desk charging a 1% platform fee, completely bypassing the extreme slippage of AMM liquidity pools.
*   **Governance Security (Staked-Governance):** To bypass the EVM 24KB contract size limit and prevent Sybil "Flash-Vote" exploits without relying on heavy OpenZeppelin `ERC20Votes` cryptography, the platform uses a **Dynamic Vote-Lock**. Voting weight is strictly mapped to tokens deposited in the `DividendDistributor`. If an investor votes on an active milestone, their tokens are cryptographically locked in the staking contract until the poll closes.

---

## 2. On-Chain Data Indexer

Blockchains are append-only ledgers; they are exceptional at writing data but terrible at answering complex historical queries. To provide a fast, Web2-like dashboard, we use an indexer to run a continuous Extract, Transform, Load (ETL) pipeline.

*   **Technology:** Ponder (Open-source, TypeScript-native indexing framework).
*   **Why Ponder?** Unlike The Graph (which requires AssemblyScript and heavy local infrastructure) or Envio (which deletes free cloud deployments after 30 days), Ponder is a lightweight Node.js package. It features instant hot-reloading and generates a production-ready GraphQL API automatically.
*   **Functionality:** Ponder listens to the Sepolia RPC node for specific contract events (`ProjectLaunched`, `MilestoneExecuted`, `YieldClaimed`), decodes the hex data using the contract ABIs, and writes it directly to a relational database.

---

## 3. Off-Chain Backend API

While Ponder handles everything that happens *on-chain*, the platform still requires a backend to manage traditional *off-chain* metadata (startup logos, founder bios, pitch decks, and SaaS application URLs).

*   **Technology:** Express.js (Node.js / TypeScript).
*   **Why not Java/Spring Boot?** Introducing Java would fragment the tech stack. By keeping the backend in Node.js/Express, we maintain a unified TypeScript ecosystem. This allows the Frontend, Indexer, and Backend API to seamlessly share data models, types, and utility functions, drastically reducing development friction.
*   **Functionality:** Exposes REST API endpoints for the frontend to fetch and update rich metadata that is too expensive to store natively on the Ethereum blockchain.

---

## 4. The Shared Database Architecture

The most powerful feature of this off-chain architecture is the unified database layer. 

*   **Technology:** PostgreSQL.
*   **The Architecture:** Both Ponder and the Express API connect to the *exact same* PostgreSQL database instance.
    *   Ponder strictly manages and writes to the schema representing on-chain ledger events.
    *   Express manages and writes to the schema representing off-chain metadata.
*   **The Superpower:** Because the data coexists in one relational database, the Express API can execute SQL `JOIN` operations across both datasets. It can query an off-chain "Pitch Deck URL" and instantly join it with Ponder's on-chain "Total ETH Raised" table, serving the frontend a perfectly merged data object in milliseconds.

---

## 5. Frontend Application

The user-facing dashboard where investors fund projects, trade on the P2P exchange, and claim yields.

*   **Technology:** Next.js (React / TypeScript).
*   **Web3 Integration:** `wagmi` / `viem` (or `ethers.js`) for connecting MetaMask, reading instantaneous contract states, and prompting users to sign transactions.
*   **Data Fetching:** Consumes the merged data from the Express REST API and queries Ponder's GraphQL endpoint directly for high-speed, real-time on-chain metrics.

---

## 6. Deployment & DevOps (The Free-Tier Strategy)

To host the entire off-chain infrastructure perpetually for free during the testnet phase, we utilize specialized cloud providers to bypass standard 30-day trial limitations.

*   **Database Hosting: Neon Postgres**
    *   *Why Neon?* Neon offers a generous 1GB free tier that never expires. Unlike Supabase (which forcibly pauses databases after 7 days of inactivity, taking your app offline), Neon features automatic "scale-to-zero" compute. It sleeps when idle to save quota but wakes up in milliseconds the exact moment Ponder or Express sends a query.
*   **Backend & Indexer Hosting: Render**
    *   *Deployment:* Both Ponder and the Express API are deployed as individual "Web Services" on Render's free tier.
    *   *The Anti-Sleep Hack:* Render automatically spins down free Web Services after 15 minutes of inbound inactivity. If Ponder sleeps, it stops listening to the blockchain and misses events. To bypass this, we use `cron-job.org` to ping the Ponder `/health` endpoint every 14 minutes (`/ready` is 503 until the historical sync completes, so it is not used for keep-alive). This tricks Render into staying awake indefinitely, ensuring 100% indexing uptime at zero cost.