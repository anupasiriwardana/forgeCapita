# ForgeCapita

**Milestone-governed crowdfunding and automated revenue sharing for SaaS startups, on-chain.**

ForgeCapita lets founders raise funds from investors through programmatic escrow milestones, and then routes the startup's SaaS revenue back to the people who funded it. Investors vote on each milestone with their staked tokens, and revenue is split by smart contract: no bank, no fund manager, and no promise to trust. Everything that moves money is verifiable code, and everything that cannot be put on-chain (identity checks, repositories, revenue auditing) is handled by a transparent off-chain service.

> **Status: testnet.** The protocol is deployed on Arbitrum Sepolia and BSC Testnet. Tokens have no monetary value, the legal agreement text is only a draft, and nothing here is investment advice. Do not use real funds.

---

## Read this first: documentation map

| Document | What it answers |
| --- | --- |
| **[about.md](about.md)** | The full product and protocol specification: business model, tokenomics, fees, governance, refunds, emergency shutdown, takeover, revenue auditing and the trust pipeline. Start here to understand *what* ForgeCapita does and *why*. |
| **[system-architecture-and-tech-stack.md](system-architecture-and-tech-stack.md)** | How the system is built: the four services, how data flows between them, the database layout, authentication, the gasless relayer, deployment and operations. Start here to understand *how* it works. |

---

## How it works

1. **Founders apply and get vetted.** A founder signs in with a wallet, completes a profile, and applies with a resume and links. Staff review the application and a KYC documents link. An approved founder accepts a revenue share agreement, gets a ForgeCapita-created GitHub repository and registers a public "shadow wallet" where their payment processor settles.
2. **Founders launch a project.** The founder sets a token, a funding goal, a deadline and up to five milestones. A factory contract deploys a lightweight proxy suite for the project: token, milestone escrow, staking and dividend vault, and revenue router. A live minimum funding goal (based on current gas prices) protects the project from launching with a goal that cannot cover its own costs.
3. **Investors fund the project.** If the goal is met, backers receive 80% of the token supply pro rata, and the founder's 20% stays locked in escrow. A 5% platform fee is taken when funding closes. If the goal is missed, everyone is refunded in full.
4. **Milestones release funds.** The escrow releases money milestone by milestone. Investors stake their tokens and vote; voting power is the staked balance, and a vote locks those tokens until the poll closes. Founder tokens vest only when the final milestone is executed.
5. **Revenue is shared automatically.** Customer payments go through the project's revenue router: 24% to the creator, 1% to the platform and 75% to staked investors, who claim ETH yield at any time.
6. **Investors can trade.** A native peer-to-peer exchange lets holders trade project tokens for ETH with a 1% fee, with no AMM slippage.
7. **Investors are protected.** If a founder stalls, the Guardian can cancel the project so investors refund themselves; an emergency shutdown, a milestone freeze and a staked-weight takeover vote followed by a developer handover cover the other failure modes. Off-chain, a revenue audit compares each founder's shadow wallet inflow with the revenue routed on-chain and escalates the gap.

Four actions are gasless for users (claim tokens, stake, vote, execute a milestone): a ForgeCapita relayer pays the gas out of the platform fee, so a plain MetaMask-style wallet is enough.

---

## Architecture and repositories

The system is a polyrepo. This root repository holds the documentation and the Docker Compose orchestration; each service lives in its own repository:

| Repository | Role | Stack |
| --- | --- | --- |
| **[forgeCapita-contracts](https://github.com/anupasiriwardana/forgeCapita-contracts)** | Smart contracts: registry, factory, escrow, token, dividend distributor, revenue router, P2P exchange. Deployed on both testnets, with a gasless relayer, a gas ledger, milestone freeze and a role split. | Solidity, Hardhat 3, OpenZeppelin |
| **[forgeCapita-indexer](https://github.com/anupasiriwardana/forgeCapita-indexer)** | Multi-chain event indexer. Turns on-chain events into relational tables and serves a read API. | Ponder, Viem, Hono, PostgreSQL |
| **[forgeCapita-api](https://github.com/anupasiriwardana/forgeCapita-api)** | Off-chain service: wallet sign-in and roles, founder applications, KYC link review, agreement, repository provisioning, gasless relayer, revenue audit, complaints, takeover workflow, admin endpoints. Never holds the treasury key. | Node.js, Express 5, TypeScript, pg, ethers, zod |
| **[forgeCapita-frontend](https://github.com/anupasiriwardana/forgeCapita-frontend)** | The web app, with a public site and role-based areas for investors, founders, staff and admins. | Next.js 16, React 19, TypeScript, Tailwind CSS 4, wagmi, viem, RainbowKit |

```text
                 +----------------------+
                 |   Browser + wallet   |
                 +----------+-----------+
                            |
                 +----------v-----------+
                 |  Frontend (Next.js)  |
                 +----+-------------+---+
       REST (accounts,  |             |  GraphQL / REST (on-chain data)
       workflows, relay)|             |
            +-----------v---+     +---v------------------+
            |  API (Express)|     |  Indexer (Ponder)    |
            +------+--------+     +----------+-----------+
                   |   shared Postgres (Neon): schemas app + ponder
                   +------------+------------+
                                |
                +---------------v----------------+
                |  Arbitrum Sepolia and BSC      |
                |  Testnet: ForgeCapita contracts|
                +--------------------------------+
```

The API and the indexer use the same PostgreSQL database in separate schemas (`app` for off-chain data, `ponder` for indexed chain data), so the API can join the two. See **[system-architecture-and-tech-stack.md](system-architecture-and-tech-stack.md)** for the details.

### Testnet deployment

| Chain | Chain ID | Registry | Factory |
| --- | --- | --- | --- |
| Arbitrum Sepolia | 421614 | `0x2e6A22fd249930b9F2905D1bD66443CbE366AacB` | `0x246Ffa3Bf583a15e92e17FAa32d3663C17204d52` |
| BSC Testnet | 97 | `0x2e6A22fd249930b9F2905D1bD66443CbE366AacB` | `0x246Ffa3Bf583a15e92e17FAa32d3663C17204d52` |

The full address list, start blocks and verification status are in the deployment notes inside the contracts repository.

---

## The web app

| Area | Who | Pages |
| --- | --- | --- |
| **Public site** | Everyone | Home (with FAQ), Projects marketplace, project detail, How it works, About, Legal |
| **Investor** | Every signed-in user (default role) | Portfolio (summary and positions), P2P market, milestone voting, become a founder, profile |
| **Founder** | Approved founders | My projects, Launch project, My launches, Revenue, Repository, Shadow wallets, Takeover requests, Agreement |
| **Staff / Admin** | Operations | Applications, Shadow wallets, Users (admin only), Launch drafts, Takeover, Compliance, Platform fees, Relay, Audit log, Complaints |

Sign-in is wallet-first: sign a message, no email or password. Every user completes a profile before taking write actions.

---

## Quick start (local development)

The services run in Docker; the blockchain and the database are external (a testnet RPC and a Neon Postgres database).

### Prerequisites

* **Git**
* **Docker** with Docker Compose
* A **PostgreSQL** database, for example a free one from [Neon](https://neon.tech/). Use the *direct* (non-pooled) connection string for the indexer.
* **RPC URLs** for Arbitrum Sepolia and BSC Testnet (the public defaults work for light use)
* **MetaMask** or another wallet, set to one of the two testnets
* Optional: an Etherscan V2 API key (revenue audit), a GitHub App (repository provisioning) and a relayer wallet (gasless actions). The app runs without them, with those features disabled.

### 1. Clone the services into this folder

The Compose file expects the four services next to it, with these exact folder names:

```bash
git clone https://github.com/anupasiriwardana/forgeCapita-api.git forgecapita-api
git clone https://github.com/anupasiriwardana/forgeCapita-contracts.git forgecapita-contracts
git clone https://github.com/anupasiriwardana/forgeCapita-frontend.git forgecapita-frontend
git clone https://github.com/anupasiriwardana/forgeCapita-indexer.git forgecapita-indexer
```

```text
forgeCapita/                 (this repository)
|-- docker-compose.yaml
|-- README.md, about.md, system-architecture-and-tech-stack.md
|-- forgecapita-api/
|-- forgecapita-contracts/
|-- forgecapita-frontend/
`-- forgecapita-indexer/
```

### 2. Configure the environment

Each service documents every variable in its `.env.example`. Copy it and fill in your own values. Never commit real values; `.env` files are git-ignored.

| Service | File to create | Required values |
| --- | --- | --- |
| `forgecapita-api` | `.env` | `DATABASE_URL`; optionally `ADMIN_WALLETS`, `RELAYER_*`, `ETHERSCAN_API_KEY`, `GITHUB_APP_*`, `INDEXER_URL` |
| `forgecapita-indexer` | `.env.local` | `DATABASE_URL` (direct connection), `PONDER_RPC_URL_421614`, `PONDER_RPC_URL_97`, contract addresses |
| `forgecapita-frontend` | `.env.local` | Optional: `NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID`, RPC overrides |

For local testing of the funding flow you can set `MINIMUM_GOAL_MODE=test` in the API environment, which disables the live minimum funding goal check. Leave it at the default (`live`) elsewhere.

### 3. Start everything

```bash
docker compose up -d --build
```

Apply the database migrations (always before restarting the API after a schema change):

```bash
docker compose exec -T api npm run migrate
```

| Service | URL |
| --- | --- |
| Frontend | http://localhost:3000 |
| API | http://localhost:3001 |
| Indexer (GraphQL, health) | http://localhost:42069 |

The indexer needs a while to sync from the contracts' deployment blocks on first start. Its `/health` endpoint answers immediately; `/ready` only once the historical sync has finished.

### 4. Make yourself an admin (testnet)

Set `ADMIN_WALLETS` in the API environment to your wallet address, restart the API, then sign in. From the admin area you can promote other wallets to staff.

### Useful commands

```bash
docker compose logs -f api            # follow one service's logs
docker compose restart api            # restart one service
docker compose exec -T api npm test   # API unit tests
docker compose exec -T frontend npm install   # install a new frontend dependency
```

API integration tests run against a throwaway Postgres in the `test` profile:

```bash
docker compose --profile test up -d --wait pg_test
docker compose exec -T api npm run test:integration
```

Contracts are compiled and tested in their own repository (`npx hardhat compile`, `npx hardhat test`). Deployment is a deliberate manual step documented in that repository.

---

## Security notes

* The platform never holds investor funds off-chain, and the API never holds the treasury key. Shutdown, freeze and takeover transactions are signed by the admin in the browser wallet.
* Secrets belong in `.env` files only (git-ignored). The frontend exposes only `NEXT_PUBLIC_*` values, which must never contain secrets.
* Personal data collected for profiles and KYC is access-controlled, and audit entries never contain the values themselves.
* This is unaudited testnet software. Do not use it with real funds.
