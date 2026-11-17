# ForgeCapita Ecosystem Architecture & Business Specification

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





### C. Gas Fee Handling: Gasless Relayer (contracts v2) and the retired v1 Paymaster Signature

ForgeCapita sponsors gas for four actions: **claim tokens, stake, vote and execute a milestone**. Every other action (invest, claim yield, refund, create project, send revenue, unstake, P2P trading) is paid by the user. Contracts v1 could not do this; **contracts v2 adds it** and is deployed on Arbitrum Sepolia and BSC Testnet (addresses in `forgecapita-contracts/docs/deployments-v2.md`; v1 is archived by the `v1-testnet` git tag). Users need only a plain MetaMask-style wallet, not a smart-contract wallet.

**v2 model (as built):**

* **Signed actions:** voting uses a free EIP-712 signature and staking uses a free ERC-2612 permit signature (no ETH needed). Claiming tokens needs no signature at all, because the tokens can only go to the depositor. A ForgeCapita **relayer** wallet submits the transaction and pays the gas. The contract verifies the signature (plain wallet or ERC-1271 contract wallet), its deadline and that the caller is an allow-listed relayer. A vote counts once per milestone, and a failed permit is tolerated only for the owner or an allow-listed relayer.
* **Gas ledger:** the treasury keeps the whole 5% platform fee and pays gas from it. Each project's contract records the sponsored cost (measured at the end of the call plus a fixed overhead per action, with a per-action cap `maxCostPerAction` and an Arbitrum data-fee `l1Surcharge` set in the registry) against a budget equal to that project's 5% fee. Any overflow is recovered from milestone payouts, at most 15% of each milestone release; what is still unpaid carries to the next milestone and is written off after the final one. Above the ceiling (budget plus 15% of net funds) the relayer is refused and the user simply pays their own gas. The registry settings belong to the Guardian and change without a redeploy.
* **Queueing:** the web app shows a gas tracker (current fee versus the 24-hour average). When fees are high, a signed request waits and is sent when they fall back to the average, for at most 6 hours (less when a vote is about to close).
* **Wallet roles:** one shared `ForgeCapitaRegistry` per chain holds the treasury, the Guardian (cancel, emergency shutdown, milestone freeze, developer proposals) and the relayer allow-list, with two-step rotation. At launch the treasury and the Guardian are the same wallet. The relayer is a separate server-held hot wallet that can only submit the gasless actions.
* **Removed in v2:** the treasury signature on `executeMilestone`. Once a milestone is approved anyone can trigger it and the payout always goes to the developer. Withholding payouts is replaced by an explicit Guardian freeze: 45 days per freeze, 90 days in total, blocking only the payout and extending any open vote by the time frozen.

**v1 behaviour (retired, archived by the `v1-testnet` tag; kept for history):**

* **Stateless Verification (v1):** Accumulated excess gas debt is tracked off-chain by the ForgeCapita API.
* **Cryptographic Settlement:** When a developer calls `executeMilestone(uint256 gasDebt, bytes calldata signature)`, the `MilestoneEscrow` contract reconstructs the payload hash using `keccak256(abi.encodePacked(address(this), currentMilestoneIndex, block.chainid, gasDebt))` and recovers the ECDSA signer.


* **Automated Deduction:** The contract enforces `recoveredSigner == forgeCapitaTreasury` and verifies that the milestone's ETH payout covers the debt (`ethToRelease >= gasDebt`). The `gasDebt` is transferred directly to `forgeCapitaTreasury` first, and the remaining `developerNetPayout` is sent to the developer.



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

* Founders submit an off-chain application to the API (`status = 'PENDING'`).
* Staff or the admin review each application (approve, reject or request changes). Approval requires manual KYC, an accepted Revenue Share Agreement, a ForgeCapita-created GitHub or GitLab repository and an approved shadow wallet.
* The Next.js frontend disables the contract deployment workflow until the profile is approved (`status = 'APPROVED'`).
* **Rogue Deployment Defense:** If a malicious user interacts directly with the `ForgeCapitaFactory` contract via an RPC node to bypass the UI, the Ponder indexer captures the `ProjectLaunched` event. The backend API cross-references the indexed `project` table against `startup_profiles`, flags the unapproved deployment, and allows the Guardian to neutralize it: cancel the project (refunds open), then run `emergencyShutdown()` once the refund window has passed.



### C. Shadow Wallet Auditing & Etherscan API Polling ("Secret Shopper")

A core vulnerability in Web3 revenue-sharing protocols is off-chain revenue evasion—founders collecting Web2 fiat payments (e.g., via Stripe) and failing to route them through `RevenueRouter.routePayment()`.

* **Shadow Wallet Registration:** Founders must complete KYC (`kyc_verified`) and register a dedicated off-chain `shadow_wallet` address where their fiat-to-crypto payment processor settles funds. The approved wallet is shown publicly on the project page and changing it needs staff or admin approval.
* **Automated Cron Reconciliation:** The Node.js API runs a scheduled `node-cron` job (`auditService`) that polls the Etherscan and BscScan APIs to inspect inbound token/ETH transfers to each project's `shadow_wallet`.
* **Allow-listed tokens:** the tally covers the native coin and an admin-managed allow-list of ERC-20 tokens per chain (stablecoins counted at $1, others priced through a price API), so paying the founder in stablecoins cannot evade the audit. The router accepts native coin only, so founders convert before depositing and the comparison is in USD.
* **Enforcement:** the auditor compares the 30-day shadow wallet inflow against the on-chain `PaymentReceived` deposits indexed from `RevenueRouter`. Only deposits **below** observed inflow by more than the tolerance (default about 4%, admin-adjustable) are flagged. A case moves `OK → WARNED` (founder notified) `→ ESCALATED` after 14 days unresolved (admin notified). The admin may freeze milestone payouts and, as a final decision, trigger **Emergency Shutdown** or a **Hostile Takeover**. A weekly **Secret Shopper** checkout crawl verifies that the live checkout pays the registered shadow wallet.
* **Limit:** the tally only sees what arrives in the shadow wallet, not the founder's true customer revenue.



---

### D. Accounts, Roles & Operations

* **Accounts:** email + password (verification, reset, lockout, server-side sessions). A wallet is linked to an account by signing a message; one wallet per account.
* **Roles:** `investor`, `founder`, `staff`, `admin`. There is one admin, whose wallet is the on-chain treasury. Two-factor (TOTP) is mandatory for staff and admin.
* **Staff** handle non-technical operations: application review, manual KYC, repo provisioning, shadow-wallet approvals and investor complaints. **The admin** makes the final decisions: emergency shutdown, milestone freeze, takeover execution, repo lockout, staff management and system settings. Shutdown, freeze and takeover transactions are signed by the admin in the browser wallet; the server never holds the treasury key.

### E. Founder Escape Protection (End-of-Runway, Legal Recourse, Takeover)

* **Code ownership:** ForgeCapita creates and owns the project repository (GitHub or GitLab, the founder's choice) and grants the founder write access. Access can be revoked in a takeover.
* **Identity and legal mapping:** manual KYC plus an "I agree" Revenue Share Agreement (draft text, to be reviewed by a lawyer) that ties the founder's identity to the project wallet, the shadow wallet and the repository rules.
* **Investor complaints:** investors can file complaints; staff triage and the admin decides on escalations.
* **Community takeover:** an off-chain signalling vote weighted by staked balance (snapshot from the indexer) shows investor support; the admin then executes `proposeNewDeveloper`, the new developer accepts, and repository access is handed over. Contracts alone cannot elect a developer.
* **Limit:** after the final milestone the founder's tokens are vested, and an emergency shutdown would destroy investors' token value. Remedies at that stage are repository lockout, takeover and legal recourse.

---

## 4. Repository Architecture & Responsibilities

The ForgeCapita monorepo/workspace is decoupled into four specialized services:

| Repository | Tech Stack | Primary Purpose & Responsibilities |
| --- | --- | --- |
| **`contracts`** | Solidity `^0.8.20`, Hardhat, OpenZeppelin (Clones & Upgradeable), Ethers.js | Houses the core smart contracts (`ForgeCapitaFactory`, `MilestoneEscrow`, `DividendDistributor`, `RevenueRouter`, `ProjectToken`, `P2PExchange`) and interfaces. Manages compilation, unit testing, deployment scripts, and block explorer verification across **Arbitrum Sepolia** (Chain ID `421614`) and **BSC Testnet** (Chain ID `97`). v1 is deployed and archived (`v1-testnet` tag); v2 (gasless relayer, gas ledger, freeze, role split) is deployed on both testnets (`forgecapita-contracts/docs/deployments-v2.md`). |
| **`indexer`** | Ponder, Viem, Hono, GraphQL, PostgreSQL | Multi-chain blockchain indexer that monitors `ForgeCapitaFactory` and `P2PExchange` from their deployment `startBlock`s on Arbitrum Sepolia and BSC Testnet. Dynamically attaches listeners to newly cloned child contracts (`MilestoneEscrow`, `DividendDistributor`, `ProjectToken`, `RevenueRouter`) upon `ProjectLaunched` events. Transforms raw EVM logs into relational tables (`project`, `milestone`, per-project positions and deposits, votes, `p2pListing`, trades and revenue records, all keyed by chain) in Neon PostgreSQL and exposes a real-time `/graphql` and `/sql/*` API. |
| **`api`** | Node.js, Express.js, TypeScript, `pg` (PostgreSQL Pool), Ethers.js, `node-cron`, Multer | Off-chain microservice running on port `3001`. Manages the `startup_profiles` table and executes `LEFT JOIN` queries combining Ponder's on-chain `project` rows with off-chain metadata (bios, pitch decks, uploaded logos, shadow wallets, KYC status). Houses accounts and role-based access (investor, founder, staff, admin), the founder application / manual KYC / agreement / repo-provisioning workflow, the **gasless relayer** (gas oracle, request queue, relayer wallet), the **revenue audit** (Etherscan reconciliation incl. allow-listed ERC-20, compliance cases, Secret Shopper worker), complaints and takeover workflow, and the dashboard endpoints. It never holds the treasury key. |
| **`frontend`** | Next.js (App Router), TypeScript, Tailwind CSS, Wagmi/Viem | Client-side web application strictly separated from backend business logic. Dark, mobile-first design system with a gas tracker and sign-a-message flows for the gasless actions. Provides four role-specific interfaces: **Founder Dashboard** (application wizard, gated contract launch, milestone management, revenue reporting), **Investor Dashboard** (crowdfunding deposits, token claiming, staking, voting, yield, P2P trading, complaints), **Staff Dashboard** (application and KYC review, repo and shadow-wallet approvals, complaints) and **Admin Dashboard** (compliance cases, rogue contract monitoring, freeze, emergency shutdown, takeover, relayer health, settings). |