# ForgeCapita: System Architecture and Technology Stack

This document describes how ForgeCapita is built: the services, the data flow, the technology choices, authentication, the gasless relayer, and how the system is run and deployed. For *what* the product does and why, read [about.md](about.md). For setup instructions, see the [README](README.md).

ForgeCapita is a TypeScript and Solidity system. Milestone-gated smart contracts hold the money and the rules; an indexer turns chain events into queryable data; an API handles everything that cannot live on-chain; and a web app ties them together.

---

## 1. System overview

```text
                    +-------------------------+
                    |   Browser + wallet      |
                    +------------+------------+
                                 |
                    +------------v------------+
                    |   Frontend (Next.js)    |
                    +-----+--------------+----+
        REST: accounts,     |              |   GraphQL / REST: on-chain data
        workflows, relay    |              |
              +-------------v--+      +----v-------------------+
              |  API (Express) |      |  Indexer (Ponder)      |
              |  + background  |      |  multi-chain           |
              |  jobs          |      +----------+-------------+
              +----+-------+---+                 |
                   |       |                     | reads events
                   |       |  shared Postgres    |
                   |       +--(Neon)-------------+
                   |          schemas: app, ponder
                   |
                   | relayer wallet (gas), explorer API, GitHub App
                   v
        +-------------------------------------------+
        |  Arbitrum Sepolia (421614), BSC Testnet (97)|
        |  ForgeCapita contracts                     |
        +-------------------------------------------+
```

| Layer | Repository | Responsibility |
| --- | --- | --- |
| Contracts | `forgecapita-contracts` | Custody, rules, fees, governance, refunds |
| Indexer | `forgecapita-indexer` | On-chain events into relational tables, read API |
| API | `forgecapita-api` | Accounts, workflows, relayer, audits, off-chain metadata |
| Frontend | `forgecapita-frontend` | Public site and role-based app |

The root repository holds the documentation and `docker-compose.yaml`.

---

## 2. Smart contract layer

* **Networks:** Arbitrum Sepolia (chain ID 421614) and BSC Testnet (chain ID 97). The same contract suite is deployed on both, and every record in the indexer and API is keyed by chain.
* **Language and tooling:** Solidity `^0.8.20`, Hardhat 3, OpenZeppelin (including Clones for EIP-1167 minimal proxies).
* **Contracts:**
  * `ForgeCapitaRegistry`: one per chain. Holds the treasury, the Guardian (cancel, emergency shutdown, milestone freeze, developer proposals) and the relayer allow-list, with two-step rotation. Also stores the gas-cost settings that bound sponsored gas.
  * `ForgeCapitaFactory`: clones a project suite for each launch and creates the P2P exchange.
  * `MilestoneEscrow`: holds raised funds and the founder's vesting tokens; runs milestone voting, execution, refunds and emergency shutdown.
  * `ProjectToken`: the ERC-20 representing fractional equity. The escrow is its only owner, so nobody can mint.
  * `DividendDistributor`: staking vault with a per-token dividend accumulator. Voting weight is the staked balance, and a vote locks the stake until the poll closes.
  * `RevenueRouter`: splits customer payments 24% creator, 1% platform, 75% stakers, with a zero-staker fallback.
  * `P2PExchange`: non-custodial order desk with a 1% fee.
* **Why clones:** deploying four full contracts per startup would be too large and too costly. Each project deploys small proxies pointing at shared master implementations.
* **Gasless actions:** claim tokens, stake (ERC-2612 permit), vote (EIP-712 signature) and execute a milestone can be submitted by an allow-listed relayer. The contract verifies the signature, its deadline and the relayer, and records the sponsored cost against a per-project budget equal to that project's 5% fee, with any overflow recovered from milestone payouts.
* **Deployments:** the addresses, start blocks and verification status are in the deployment notes in the contracts repository. Deployment is always a deliberate manual step and is never automated.

---

## 3. On-chain data indexer

Blockchains are good at writing data and poor at answering historical questions. The indexer runs a continuous extract-transform-load pipeline from the chains into Postgres.

* **Technology:** Ponder (TypeScript-native indexing framework), Viem, and Hono for custom HTTP routes.
* **Multi-chain:** it watches the factory and P2P exchange on both chains from their deployment blocks, and dynamically attaches listeners to each newly cloned project contract when `ProjectLaunched` fires.
* **Output:** relational tables for projects, milestones, per-project positions and deposits, votes, P2P listings and trades, revenue records and developer changes, all keyed by chain. It exposes `/graphql`, `/health` (answers immediately), `/ready` (only after the historical sync) and `/status` (indexed head, used by the API to refuse stale snapshots). A read-only SQL endpoint exists for local debugging only.
* **Resilience:** each chain accepts several RPC endpoints for failover. The indexer connects to Postgres through a direct (non-pooled) connection, because Ponder relies on session-level `search_path`.

---

## 4. Off-chain API

The API manages everything the chain cannot: identity, workflows, auditing and metadata.

* **Technology:** Node.js, Express 5, TypeScript, `pg`, ethers, zod for input validation, and pino-style structured logging. An OpenAPI description is served at `GET /api/v1/openapi.json` outside production.
* **Authentication:** wallet-first. A user requests a nonce, signs a message with their wallet, and receives an opaque server-side session in an HttpOnly cookie (only a hash is stored). There are no emails, passwords or second factors. Roles are `investor` (default), `founder`, `staff` and `admin`; wallets listed in `ADMIN_WALLETS` become admin. Every user completes a profile before write actions, and personal data never appears in audit rows or logs. State-changing routes check the request origin and are rate limited.
* **Founder trust pipeline:** application with resume and profile links, a KYC documents link reviewed by staff, a revenue share agreement, a ForgeCapita-created GitHub repository (provisioned through a GitHub App) and a staff-approved shadow wallet. A launch draft reconciler links the draft to the project once the indexer sees it on-chain.
* **Live minimum funding goal:** a gas-price oracle samples fees and computes the minimum goal a launch must declare. A mode switch (`live` or `test`) lets testnets skip the check.
* **Gasless relayer:** a server-held hot wallet, allow-listed in the registry, submits the sponsored transactions. A queue waits for cheap gas, enforces per-user and per-chain limits, and an optional job executes milestones automatically at the end of a vote. The relayer cannot move funds; it can only submit the gasless actions. The treasury key is never on the server.
* **Revenue audit:** an Etherscan V2 poller records native transfers into each approved shadow wallet and compares them with revenue routed through `RevenueRouter`. A compliance state machine (`OK`, `WARNED`, `ESCALATED`, `RESOLVED` or `SHUTDOWN`) notifies the founder, then staff, and lets the admin act. Stablecoin and ERC-20 allow-listing, and a "Secret Shopper" checkout crawler, are planned and not built.
* **Takeover workflow:** staff prepare a case and the API freezes a snapshot of staked balances from the indexer; the candidate accepts off-chain; investors cast signed advisory votes (quorum 20% of staked supply, 7-day window); the admin closes the vote, proposes the new developer on-chain, the candidate accepts on-chain, and repository access is handed over.
* **Background jobs:** the reconcile, auto-execution, inflow, compliance and gas-sampling jobs coordinate through transaction-scoped Postgres advisory locks, so they are safe behind a pooled connection. Transient database or network errors are survived without a crash.

---

## 5. Shared database

Both the indexer and the API use the same PostgreSQL instance, in separate schemas:

* `ponder`: written only by the indexer (on-chain ledger data).
* `app`: written only by the API through numbered, idempotent migrations (profiles, applications, wallets, sessions, drafts, relay requests, inflows, compliance cases, complaints, takeover cases, audit log).

Because both live in one database, the API can `JOIN` off-chain metadata with on-chain totals in a single query, for example a project's description with its total raised. The audit log is append-only and enforced by constraints. Amounts are handled as `bigint` and stored as `NUMERIC` or strings; floating point is never used for money.

---

## 6. Frontend

* **Technology:** Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS 4, wagmi, viem, RainbowKit, TanStack Query, and lucide icons.
* **Structure:** one application shell chooses the layout from the sign-in state: a slim top bar and footer for visitors, and a role-aware sidebar for signed-in users. Public pages (home with FAQ, projects, project detail, how it works, about, legal) are open to everyone. Investor, founder, staff and admin areas are separate route groups with a role check in their layout; the API enforces the same roles, so the UI check is only a convenience.
* **Data:** the REST API provides merged and off-chain data; the indexer's GraphQL and REST endpoints provide fast on-chain views; wagmi and viem read and write contracts directly from the user's wallet.
* **Accessibility:** status is never conveyed by an icon alone, and decorative icons are hidden from assistive technology.

---

## 7. Local development

`docker-compose.yaml` runs the indexer (port 42069), the API (port 3001) and the frontend (port 3000), with source folders mounted for hot reload. The database and the RPC endpoints are external. The indexer and API restart automatically unless stopped. A throwaway Postgres (`pg_test`, `test` profile) backs the API integration tests. See the [README](README.md) for the commands.

---

## 8. Deployment and operations (free-tier plan)

The testnet phase is designed to run for free:

* **Database: Neon Postgres.** A free tier that does not expire and scales to zero when idle. The API uses Neon's pooled endpoint; the indexer must use the direct endpoint.
* **Services: Render.** The indexer and the API run as web services. Render spins free services down after 15 minutes without inbound traffic, and a sleeping indexer misses events, so an external scheduler (`cron-job.org`) pings the indexer's `/health` endpoint every 14 minutes. `/ready` returns 503 until the historical sync completes, so it is not used for keep-alive. Set `TRUST_PROXY=1` for the API behind Render's proxy and `CORS_ORIGINS` to the deployed frontend origin.
* **Frontend:** any Next.js host.

This describes the intended hosting. The repositories are currently run locally with Docker Compose.

---

## 9. Security and engineering practices

* Server-side authorization on every route; input validated with zod; parameterised SQL only.
* Secrets only in git-ignored `.env` files; `NEXT_PUBLIC_*` variables are public by definition.
* No personal data in logs or audit rows; sessions stored as hashes.
* Treasury and Guardian transactions are signed by the admin in the browser wallet. The relayer key is the only server-held key and has no authority beyond the gasless actions.
* Contract changes are compiled and tested locally; deployment is a manual, deliberate step.
* The project is unaudited testnet software.
