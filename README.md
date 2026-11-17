# forgeCapita

forgeCapita is a trustless Web3 crowdfunding and yield platform for developers. Built with Solidity and React, it uses decentralized milestone escrows to protect investor capital and automated smart contracts to route SaaS revenue directly to token holders. No banks, no trust required—just verifiable code.

## 🏗 Architecture & Repositories
forgeCapita is built as a highly decoupled, production-ready microservice ecosystem. Because this project utilizes a Polyrepo structure, the code for each service is maintained in its own dedicated repository:

* **[forgeCapita-contracts](https://github.com/anupasiriwardana/forgeCapita-contracts):** Hardhat environment for Solidity smart contract compilation, testing, and deployment.
* **[forgeCapita-frontend](https://github.com/anupasiriwardana/forgeCapita-frontend):** Vite + React + TypeScript web application utilizing ethers.js and Chakra UI.
* **[forgeCapita-indexer](https://github.com/anupasiriwardana/forgeCapita-indexer):** Node.js background worker using WebSockets to listen for on-chain events and index them to the database.
* **[forgeCapita-api](https://github.com/anupasiriwardana/forgeCapita-api):** Express REST API to serve blockchain data efficiently to the frontend without gas fees.

## 📋 Prerequisites
Before running this project, ensure you have the following installed:
* **Git** (To clone the microservices)
* **Docker & Docker Compose** (Crucial for running the isolated Linux environments on Windows/Mac)
* **MetaMask** Browser Extension (Connected to a Localhost network)
* A **PostgreSQL** database (You can easily spin up a free serverless database using [Neon.tech](https://neon.tech/))

---

## 📥 Step 1: Clone the Microservices
Because this root directory acts purely as the Docker orchestrator, you must clone the individual codebases into this folder before proceeding. 

Open a terminal at the root of this `forgeCapita` folder and run the following commands to pull in the microservices:

```bash
git clone [https://github.com/anupasiriwardana/forgeCapita-api.git](https://github.com/anupasiriwardana/forgeCapita-api.git)
git clone [https://github.com/anupasiriwardana/forgeCapita-contracts.git](https://github.com/anupasiriwardana/forgeCapita-contracts.git)
git clone [https://github.com/anupasiriwardana/forgeCapita-frontend.git](https://github.com/anupasiriwardana/forgeCapita-frontend.git)
git clone [https://github.com/anupasiriwardana/forgeCapita-indexer.git](https://github.com/anupasiriwardana/forgeCapita-indexer.git)
```
### 📂 Expected Directory Structure
After running the commands above, your root folder must look exactly like this so Docker Compose can map the paths correctly:

```text
forgeCapita/ (This Root Repository)
│
├── docker-compose.yaml
├── README.md
├── forgeCapita-api/        (Cloned repository)
├── forgeCapita-contracts/  (Cloned repository)
├── forgeCapita-frontend/   (Cloned repository)
└── forgeCapita-indexer/    (Cloned repository)
```

## ⚙️ Step 2: Environment Setup
Before booting the containers, you need to configure the database connections for the API and Indexer.

1. Navigate to the newly cloned `forgeCapita-indexer` folder and create a `.env` file:
   ```env
   DATABASE_URL="postgresql://<username>:<password>@<your-db-url>/neondb?sslmode=require"
   WS_URL="ws://hardhat:8545"
   CONTRACT_ADDRESS="0x5FbDB2315678afecb367f032d93F642f64180aa3"
   ```

2. Navigate to the newly cloned `forgeCapita-api` folder and create a `.env` file:
   ```env
   DATABASE_URL="postgresql://<username>:<password>@<your-db-url>/neondb?sslmode=require"
   ```

*(Note: The `CONTRACT_ADDRESS` is the deterministic address Hardhat generates on a fresh local node for the first deployed contract).*

## 🚀 Step 3: Running the Project

### Boot the Docker Network
Ensure you do not have any local `node_modules` folders inside the cloned directories causing symlink conflicts with the Docker Linux environment. From the root directory of the project, run:

```bash
docker-compose up -d --build
```
*This command will build all four images, install clean Node dependencies internally, generate the Prisma database clients, and start all services.*

### Deploy the Smart Contract
Because the local Hardhat node resets to Block 0 every time it starts, you must deploy the smart contract to the fresh blockchain. Execute the deployment command inside the running Hardhat container:

```bash
docker-compose exec hardhat npx hardhat ignition deploy ./ignition/modules/RevenueRouter.ts --network localhost
```

### Configure MetaMask
To interact with the local dApp, configure MetaMask to connect to the Hardhat node:
1. Open MetaMask and navigate to **Add Network -> Add manually**.
2. **Network Name:** Hardhat Local
3. **RPC URL:** `http://127.0.0.1:8545`
4. **Chain ID:** `31337`
5. **Currency Symbol:** `ETH`

Finally, to cover gas fees, import one of the Hardhat test accounts into MetaMask using the private keys printed in the Hardhat container logs.
*(**Important:** Whenever you restart Docker, go to MetaMask Settings > Advanced > "Clear activity tab data" to reset your transaction nonce).*

## 🌐 Services Overview
Once up and running, your microservices are mapped to the following local ports:
* **Frontend Application:** http://localhost:5173
* **Backend REST API:** http://localhost:3001
* **Local Blockchain Node:** http://127.0.0.1:8545
* **Indexer Service:** Runs silently in the background (`docker-compose logs -f indexer` to view live event captures).

## 🛠 Useful Commands
* **View Logs for all services:** `docker-compose logs -f`
* **Restart a specific service:** `docker-compose restart api`
* **Update the Database Schema:** `docker-compose exec indexer npx prisma db push`
* **Install a new npm package (Example: API):**
  `docker-compose exec api npm install <package-name>`
  `docker-compose up -d --build api`

## 💻 Frontend Development (React + TypeScript + Vite)

This project's frontend is initialized with Vite. The template provides a minimal setup to get React working in Vite with HMR and some ESLint rules.

Currently, two official plugins are available:
- [@vitejs/plugin-react](https://github.com/vitejs/plugin-react/blob/main/packages/plugin-react) uses [Oxc](https://oxc.rs)
- [@vitejs/plugin-react-swc](https://github.com/vitejs/plugin-react-swc/blob/main/packages/plugin-react-swc) uses [SWC](https://swc.rs/)

