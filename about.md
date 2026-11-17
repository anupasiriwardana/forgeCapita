# ForgeCapita Ecosystem Architecture & Business Specification

> Part of the [ForgeCapita](README.md) documentation. This file specifies *what* the product does and why (business model, protocol rules, trust pipeline). For *how* it is built and run, see [system-architecture-and-tech-stack.md](system-architecture-and-tech-stack.md). This is a testnet build: tokens have no monetary value and nothing here is investment advice.

ForgeCapita is a decentralized, milestone-governed crowdfunding and automated revenue-sharing protocol designed for SaaS applications and Web2/Web3 startups. By combining **EIP-1167 Minimal Proxy smart contracts**, **multi-chain event indexing**, and an **off-chain compliance and gasless-relayer microservice**, ForgeCapita eliminates traditional venture capital trust bottlenecks. Investors fund startups through programmatic escrow milestones and receive automated, real-time ETH dividends derived directly from the startup's gross revenue.

---

## 1. Business Model & Tokenomics

ForgeCapita aligns the financial incentives of startup founders, retail/institutional backers, and the platform itself through three distinct on-chain revenue and equity streams:

### A. Crowdfunding & Equity Allocation (`MilestoneEscrow` & `ProjectToken`)

* **Capital Formation:** Founders launch a project by defining a token name, symbol, total supply, funding goal, funding duration (in days), and up to 5 sequential milestones whose unlock percentages must sum to exactly 100%.


* **80/20 Token Split:** Upon initialization, the smart contract reserves 20% of the total token supply for the founder (`developerSupply`) and allocates the remaining 80% (`publicSupply`) to public investors.


* **Proportional Claims:** Once the funding goal is met and the round closes, backers claim their proportional share of the 80% public supply based on their ETH contribution relative to the `totalRaised`.


* **Founder Vesting:** The founder's 20% token allocation remains locked inside the `MilestoneEscrow` contract and only vests to the developer's wallet when the final milestone is successfully approved and executed.


* **Crowdfunding Platform Fee:** When the founder closes a successful funding round via `closeFundingAndStart()`, a **5% platform fee** (`PLATFORM_FEE_PERCENT`) is deducted from `totalRaised` and transferred immediately to the `forgeCapitaTreasury` wallet, leaving the remaining 95% as `netFunds` for milestone releases.



### B. Automated SaaS Revenue Routing (`RevenueRouter` & `DividendDistributor`)

Every launched project is wired to a dedicated `RevenueRouter` contract that processes inbound customer payments (`routePayment()`) and splits gross revenue deterministically:

* **1% Platform Fee:** Credited to `treasuryPendingFunds` for withdrawal by the `forgeCapitaTreasury`.


* **24% Creator Cut:** Credited to `creatorPendingFunds` for withdrawal by the active project developer.


* **75% Investor Dividend Cut:** Automatically forwarded to the project's `DividendDistributor` contract if investors have staked tokens (`totalStaked > 0`). Using a precision magnifier ($10^{18}$), the distributor increments `dividendPerToken`, allowing staked investors to claim real-time ETH yield via `claimYield()`. If zero tokens are staked at the time of payment, the 75% investor cut rolls over into the developer's pending funds.



### C. Secondary Peer-to-Peer Market (`P2PExchange`)

* **Liquid Equity:** Investors can trade their `ProjectToken` holdings on the standalone `P2PExchange` orderbook by creating ETH-denominated listings (`createListing`), filling active orders (`fillListing`), or cancelling open orders (`cancelListing`).


* **Secondary Trading Fee:** The exchange deducts a **1% protocol fee** (`PROTOCOL_FEE_BPS = 100` basis points) from every filled listing and routes it directly to the `forgeCapitaTreasury` before paying the seller.



---

## 2. Governance, Security & Lifecycle Mechanics

### A. Project Ownership & Token Control

When `ForgeCapitaFactory.launchProject()` deploys a startup's contract suite, no human wallet is granted ownership of the ERC-20 `ProjectToken`.

* The Factory temporarily initializes the token, mints the entire scaled supply to the newly cloned `MilestoneEscrow` contract, and immediately executes `ProjectToken(tokenClone).transferOwnership(escrowClone)`.


* Consequently, the `MilestoneEscrow` smart contract is the sole `owner` of the `ProjectToken`. Neither the founder nor the ForgeCapita admin can arbitrarily mint new tokens, and privileged token state changes (such as global token invalidation) can only be triggered through the Escrow contract's strict state machine.



### B. Staking-Weight Governance & Vote Locking

* **Voting Power:** To prevent Sybil attacks and flash-loan governance manipulation, voting power in `MilestoneEscrow.vote()` is determined strictly by querying the investor's staked balance inside the `DividendDistributor` contract (`dividendDistributor.stakedBalance(msg.sender)`). The project developer is explicitly barred from voting.


* **Vote Locking:** Once an investor casts a vote on an active milestone, `isUserVoteLocked(account)` returns `true`. When the user attempts to call `withdraw()` on the `DividendDistributor`, the distributor queries the Escrow contract and reverts with `"Tokens locked in active milestone vote"`, preventing users from voting and immediately dumping their tokens.


* **Approval Thresholds:** A milestone can be executed via two paths:


1. **Fast-Track Approval:** `yesVotes` strictly exceed 50% of the entire `publicSupply`, bypassing the time lock.


2. **Standard Quorum:** The 28-day `VOTING_PERIOD` elapses, total participation (`yesVotes + noVotes`) reaches at least the 20% `QUORUM_PERCENT` of `publicSupply`, and `yesVotes > noVotes`.





### C. Gas Fee Handling: Gasless Relayer

ForgeCapita sponsors gas for four actions: **claim tokens, stake, vote and execute a milestone**. Every other action (invest, claim yield, refund, create project, send revenue, unstake, P2P trading) is paid by the user. The contracts are deployed on Arbitrum Sepolia and BSC Testnet. Users need only a plain MetaMask-style wallet, not a smart-contract wallet.

* **Signed actions:** voting uses a free EIP-712 signature and staking uses a free ERC-2612 permit signature (no ETH needed). Claiming tokens needs no signature at all, because the tokens can only go to the depositor. A ForgeCapita **relayer** wallet submits the transaction and pays the gas. The contract verifies the signature (plain wallet or ERC-1271 contract wallet), its deadline and that the caller is an allow-listed relayer. A vote counts once per milestone, and a failed permit is tolerated only for the owner or an allow-listed relayer.
* **Gas ledger:** the treasury keeps the whole 5% platform fee and pays gas from it. Each project's contract records the sponsored cost (measured at the end of the call plus a fixed overhead per action, with a per-action cap `maxCostPerAction` and an Arbitrum data-fee `l1Surcharge` set in the registry) against a budget equal to that project's 5% fee. Any overflow is recovered from milestone payouts, at most 15% of each milestone release; what is still unpaid carries to the next milestone and is written off after the final one. Above the ceiling (budget plus 15% of net funds) the relayer is refused and the user simply pays their own gas. The registry settings belong to the Guardian and change without a redeploy.
* **Queueing:** the web app shows a gas tracker (current fee versus the 24-hour average). When fees are high, a signed request waits and is sent when they fall back to the average, for at most 6 hours (less when a vote is about to close).
* **Wallet roles:** one shared `ForgeCapitaRegistry` per chain holds the treasury, the Guardian (cancel, emergency shutdown, milestone freeze, developer proposals) and the relayer allow-list, with two-step rotation. At launch the treasury and the Guardian are the same wallet. The relayer is a separate server-held hot wallet that can only submit the gasless actions.
* **Milestone execution:** once a milestone is approved anyone can trigger it, and the payout always goes to the developer. There is no treasury signature on `executeMilestone`. Withholding payouts is handled by an explicit Guardian freeze: 45 days per freeze, 90 days in total, blocking only the payout and extending any open vote by the time frozen.



### D. Hostile Takeover (Developer Handover Protocol)

If a founder abandons the project, loses their private keys, or violates off-chain legal agreements, ForgeCapita can replace the developer without migrating liquidity or redeploying contracts:

* **Two-Step Handshake:** Role transfers require calling `proposeNewDeveloper(address payable _newDeveloper)` followed by `acceptDeveloperRole()`.


* **Guardian Override:** `proposeNewDeveloper` permits execution by either the current `developer` (for voluntary team handovers) or the registry's Guardian (`require(msg.sender == developer || msg.sender == registry.guardian(), "Only active dev or Guardian can propose")`).


* **Dynamic Revenue Re-routing:** Once the nominated `pendingDeveloper` executes `acceptDeveloperRole()`, the `developer` address in `MilestoneEscrow` is updated. Because `RevenueRouter.claimCreatorFunds()` dynamically queries `IMilestoneEscrow(milestoneEscrow).developer()` on every withdrawal, all future milestone payouts and 24% SaaS revenue cuts automatically route to the new developer.



### E. End-of-Runway Refunds & Emergency Shutdown

ForgeCapita protects investor capital across three failure modes:

1. **Failed or Abandoned Crowdfund (`refundFailedCrowdfund`):** If `fundingDeadline` passes without reaching `fundingGoal`, investors can pull back 100% of their deposited ETH. If the goal was reached but the founder never calls `closeFundingAndStart()`, the founder has a 14-day close window (`CLOSE_GRACE`) after the deadline. When it ends, closing is no longer possible and investors can pull back 100% of their deposits (no fee is taken).


2. **Cancelled Project or Failed Vote (`refundDeadProject`):** A project is cancelled when the developer or the Guardian calls `cancelProject()`, or when a milestone vote ends (28 days, extended by any frozen time) without approval: that failure is settled inside `executeMilestone()`, which emits `MilestoneFailed` and opens refunds without reverting. There is no inactivity deadline; a stalled developer is handled by the Guardian (cancel, or `proposeNewDeveloper`). Investors unstake their tokens and call `refundDeadProject()`, which transfers their `ProjectToken` balance back to the escrow, decrements `remainingPublicTokens`, and returns their proportional share of the remaining ETH. The developer is barred from this refund, and refunds are always paid by the investor (never sponsored).


3. **Emergency Circuit Breaker (`emergencyShutdown`):** Restricted to the registry's Guardian. It works on a cancelled project once every public token has been refunded or 90 days (`REFUND_WINDOW`) have passed since the cancel (otherwise it reverts "Refunds still open"), and on a completed project. On a project that is neither, it reverts "Cancel project first". If a malicious founder performs an exit scam or deploys an unapproved rogue project, the Guardian first cancels the project so investors can refund themselves, and then, after the window, `emergencyShutdown()`:


* Closes funding (a completed project is also flagged `projectCancelled = true`; clients must check `completed` first).


* Calls `projectToken.emergencyInvalidate()`, flipping `isInvalidated = true`. Due to the `checkValidity` modifier on `ProjectToken`, all `transfer`, `transferFrom`, and `approve` functions permanently revert with `"Token is dead (Emergency Shutdown)"`, stopping secondary market victims from buying dead tokens on DEXs or the P2P Exchange.


* Burns all unvested tokens held inside the `MilestoneEscrow` contract.


* Sweeps the ETH still in the Escrow (what holders did not refund within the window, plus rounding dust) into the treasury. There is no off-chain payout of the swept amount.





---

## 3. Key Technical Decisions & Off-Chain Enforcement

### A. EIP-1167 Minimal Proxy (Clone) Pattern

Deploying full bytecode for four separate contracts (`ProjectToken`, `MilestoneEscrow`, `DividendDistributor`, `RevenueRouter`) per startup would exceed EVM contract size limits in the Factory and cost prohibitive gas fees. By deploying single master implementation contracts with `Initializable` guards and using OpenZeppelin's `Clones.clone()` inside `ForgeCapitaFactory`, each project launch deploys lightweight 45-byte proxy delegates.

### B. Pre-Launch UI Gating vs. On-Chain Signature Gating

To keep `ForgeCapitaFactory.launchProject()` gas-efficient and avoid redeploying immutable smart contracts, project vetting is enforced via off-chain reconciliation rather than on-chain signature checks:

* Founders submit an off-chain application to the API (`status = 'PENDING'`) with a resume link, a short description and at least one of LinkedIn, GitHub or portfolio, plus a link to their KYC documents (hosted in the founder's own storage, never uploaded to ForgeCapita).
* Staff or the admin review each application (approve or reject with a note). Approval is recorded together with a manual KYC check of the documents link. Launching then also requires an accepted Revenue Share Agreement (draft text; on testnet it is treated as reviewed), a ForgeCapita-created GitHub repository (one per founder, created through a GitHub App) and an approved shadow wallet.
* The Next.js frontend disables the contract deployment workflow until all launch checks pass, and the API refuses a launch draft otherwise. The draft must also declare a funding goal at or above a **live minimum goal** computed from current gas prices (a floor of 0.5 ETH on Arbitrum Sepolia and 1 BNB on BSC Testnet), so a project cannot launch with a goal that cannot cover its own costs. Testnets can switch the check off with `MINIMUM_GOAL_MODE=test`.
* **Rogue Deployment Defense:** If a malicious user interacts directly with the `ForgeCapitaFactory` contract via an RPC node to bypass the UI, the Ponder indexer captures the `ProjectLaunched` event. The backend API cross-references the indexed `project` table against `startup_profiles`, flags the unapproved deployment, and allows the Guardian to neutralize it: cancel the project (refunds open), then run `emergencyShutdown()` once the refund window has passed.



### C. Shadow Wallet Auditing & Etherscan API Polling ("Secret Shopper")

A core vulnerability in Web3 revenue-sharing protocols is off-chain revenue evasion—founders collecting Web2 fiat payments (e.g., via Stripe) and failing to route them through `RevenueRouter.routePayment()`.

* **Shadow Wallet Registration:** Founders must complete KYC (`kyc_verified`) and register a dedicated off-chain `shadow_wallet` address where their fiat-to-crypto payment processor settles funds. The approved wallet is shown publicly on the project page and changing it needs staff or admin approval.
* **Automated Reconciliation:** background jobs in the Node.js API poll the Etherscan V2 explorer API to record inbound transfers to each approved shadow wallet, and compare them with the revenue routed on-chain.
* **What is counted today:** the testnet build compares **native coin only** (ETH on Arbitrum Sepolia, BNB on BSC Testnet). An admin-managed allow-list of ERC-20 tokens (stablecoins counted at $1, others priced through a price API) is planned, so that paying the founder in stablecoins cannot evade the audit once it ships. The router accepts native coin only, so founders convert before depositing.
* **Enforcement:** the auditor compares the 30-day shadow wallet inflow against the on-chain `PaymentReceived` deposits indexed from `RevenueRouter`. Only deposits **below** observed inflow by more than the tolerance (default 4%, adjustable) are flagged. A case moves `OK → WARNED` (founder notified) `→ ESCALATED` after 14 days unresolved (admin notified) and ends as `RESOLVED` or `SHUTDOWN`. The admin may freeze milestone payouts and, as a final decision, trigger **Emergency Shutdown** or a **Hostile Takeover**. Founders see the plain-language compliance state of their own projects, and the numbers shown are a stored snapshot refreshed on a schedule.
* **Planned:** a weekly **Secret Shopper** checkout crawl that verifies the live checkout pays the registered shadow wallet. It is not built yet.
* **Limit:** the tally only sees what arrives in the shadow wallet, not the founder's true customer revenue.



---

### D. Accounts, Roles & Operations

* **Accounts:** wallet-first. A user signs in by signing a one-time message with their wallet (no email, password or second factor) and receives a server-side session. Every user must complete a profile (name, contact details and similar) before taking write actions; that data is access-controlled and never copied into audit entries.
* **Roles:** `investor` (the default for every new wallet), `founder`, `staff`, `admin`. There is one admin, whose wallet is also the on-chain treasury and Guardian at launch.
* **Staff** handle non-technical operations: application review, KYC checks, shadow-wallet approvals, takeover preparation and investor complaints. **The admin** makes the final decisions: emergency shutdown, milestone freeze, takeover execution, repository lockout, staff management and system settings. Shutdown, freeze and takeover transactions are signed by the admin in the browser wallet; the server never holds the treasury key.

### E. Founder Escape Protection (End-of-Runway, Legal Recourse, Takeover)

* **Code ownership:** ForgeCapita creates and owns the project repository on GitHub (through a GitHub App, one repository per founder) and grants the founder write access. Access can be revoked in a takeover.
* **Identity and legal mapping:** a manual KYC check plus an "I agree" Revenue Share Agreement (draft text, to be reviewed by a lawyer before any real-money use) that ties the founder's identity to the project wallet, the shadow wallet and the repository rules.
* **Investor complaints:** investors can file complaints; staff triage and the admin decides on escalations.
* **Community takeover:** an off-chain, advisory signalling vote weighted by staked balance. When staff open a vote, the API freezes a snapshot of staked balances (taken from the indexer, and refused if the indexer is lagging). The candidate first accepts off-chain; investors then sign their votes. A vote passes when at least 20% of the snapshot's total staked balance takes part and yes beats no (a tie fails), within a 7-day window; a failed or no-quorum case triggers a 30-day cooldown. The admin closes the vote and executes `proposeNewDeveloper` on-chain, the candidate accepts with `acceptDeveloperRole`, and repository access is handed over (the new developer is added, then the old one is revoked). Contracts alone cannot elect a developer.
* **Limit:** after the final milestone the founder's tokens are vested, and an emergency shutdown would destroy investors' token value. Remedies at that stage are repository lockout, takeover and legal recourse.

---

## 4. Repository Architecture & Responsibilities

The ForgeCapita monorepo/workspace is decoupled into four specialized services:

| Repository | Tech Stack | Primary Purpose & Responsibilities |
| --- | --- | --- |
| **`contracts`** | Solidity `^0.8.20`, Hardhat, OpenZeppelin (Clones & Upgradeable), Ethers.js | Houses the core smart contracts (`ForgeCapitaFactory`, `MilestoneEscrow`, `DividendDistributor`, `RevenueRouter`, `ProjectToken`, `P2PExchange`) and interfaces. Manages compilation, unit testing, deployment scripts, and block explorer verification across **Arbitrum Sepolia** (Chain ID `421614`) and **BSC Testnet** (Chain ID `97`). The contracts (gasless relayer, gas ledger, milestone freeze, role split) are deployed on both testnets. |
| **`indexer`** | Ponder, Viem, Hono, GraphQL, PostgreSQL | Multi-chain blockchain indexer that monitors `ForgeCapitaFactory` and `P2PExchange` from their deployment `startBlock`s on Arbitrum Sepolia and BSC Testnet. Dynamically attaches listeners to newly cloned child contracts (`MilestoneEscrow`, `DividendDistributor`, `ProjectToken`, `RevenueRouter`) upon `ProjectLaunched` events. Transforms raw EVM logs into relational tables (`project`, `milestone`, per-project positions and deposits, votes, `p2pListing`, trades and revenue records, all keyed by chain) in Neon PostgreSQL and exposes a real-time `/graphql` and `/sql/*` API. |
| **`api`** | Node.js, Express 5, TypeScript, `pg` (PostgreSQL Pool), Ethers.js, zod | Off-chain microservice running on port `3001`. Stores off-chain metadata in the `app` schema and executes joins against Ponder's on-chain `project` rows (descriptions, links, shadow wallets, KYC status; no file uploads, logos and documents are plain https links). Houses wallet sign-in and role-based access (investor, founder, staff, admin), the founder application / KYC link review / agreement / GitHub repository provisioning workflow, the live minimum funding goal, the **gasless relayer** (gas oracle, request queue, relayer wallet), the **revenue audit** (Etherscan reconciliation of native transfers, compliance cases), the complaints and takeover workflow, and the admin endpoints. It never holds the treasury key. |
| **`frontend`** | Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS 4, wagmi / viem / RainbowKit | Client-side web application strictly separated from backend business logic. Dark, mobile-first design with a public site (home with FAQ, projects, how it works, about, legal) and a role-aware sidebar once signed in. Role areas: **Investor** (portfolio with summary, funding, token claiming, staking, voting, yield, P2P trading, complaints), **Founder** (my projects, gated project launch, my launches, revenue and claimable funds, repository, shadow wallets, takeover requests, agreement) and **Staff / Admin** (applications, shadow wallets, users, launch drafts, takeover, compliance, platform fees, relayer health, audit log, complaints). Gasless actions use sign-a-message flows. |

---

## 5. Current Build Status

ForgeCapita is a working testnet build. Delivered: the contracts on both testnets, the multi-chain indexer, the API (accounts, founder pipeline, relayer, revenue audit, complaints, takeover) and the web app (public site and the investor, founder, staff and admin areas). Not built yet, by design for the testnet phase: ERC-20 / stablecoin revenue auditing, the Secret Shopper checkout crawler, file uploads, a GitHub webhook receiver, a lawyer-reviewed agreement, a security audit and mainnet deployment.