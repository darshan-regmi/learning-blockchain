# Stage 5 — Full-Stack dApp Development

`Stage 5 of 6` · Estimated 4–6 weeks

## Stage Overview

### What This Stage Builds

This stage assembles all previous knowledge into the ability to build, deploy, and operate production-grade decentralized applications. You will learn wallet integration, Web3 libraries, on-chain/off-chain interaction patterns, indexing and querying blockchain data, production deployment infrastructure, and the architectural decisions that separate amateur dApps from production systems. You will understand the full vertical stack — from a user clicking a button in a browser, to a transaction landing on-chain, to indexed data being served back to the UI.

### Why This Stage Is Critical

Knowledge of smart contracts alone does not produce a usable product. The largest fraction of dApp security and UX failures happen at the application layer: improper wallet handling, missing transaction error recovery, inadequate state synchronization, centralized dependencies that become single points of failure. This stage bridges the gap between protocol knowledge and deployed products.

### Competence at the End of This Stage

You can build a complete dApp from scratch including: wallet connection, contract interaction, real-time state updates, transaction lifecycle management, indexed data queries, and production deployment on real infrastructure. You can reason about the centralization risks in your stack and mitigate them. You can design a system that handles network failures, transaction failures, and chain reorganizations gracefully.

---

## Conceptual Map

```text
Wallet Architecture
├── Accounts, private keys, and seedphrases
├── EIP-1193 Provider API
├── WalletConnect and multi-wallet support
└── ERC-4337 Account Abstraction
        │
        ▼
Web3 Libraries and RPC
├── ethers.js / viem
├── wagmi + React integration
├── JSON-RPC API surface
└── Provider management (failover, load balancing)
        │
        ▼
Contract Interaction Patterns
├── Encoding calls and decoding results
├── Event listening and log parsing
├── Multicall batching
├── Transaction lifecycle (submission → confirmation → finality)
├── Error handling and reverts
└── Simulation (eth_call before eth_sendTransaction)
        │
        ▼
Frontend Architecture for dApps
├── State management for blockchain state
├── Optimistic UI updates
├── Handling chain reorganizations
├── Multi-chain support
└── Gas estimation and user experience
        │
        ▼
Backend + Blockchain Architecture
├── When to use a backend
├── Off-chain computation, on-chain settlement
├── Signature-based authorization
└── Relayer patterns (meta-transactions, ERC-2771)
        │
        ▼
Indexing and Querying Blockchain Data
├── Why raw RPC is insufficient
├── The Graph Protocol (subgraphs)
├── Custom indexers (Ponder, Envio)
├── Event-driven architecture
└── Historical data queries
        │
        ▼
Production Infrastructure
├── RPC provider strategy (Infura, Alchemy, self-hosted)
├── IPFS and decentralized storage (Pinata, web3.storage, Filecoin)
├── Smart contract deployment strategies
├── Monitoring and alerting (on-chain and off-chain)
└── Security hardening of the application stack
```

---

## Topic 1: Wallet Architecture and Integration

### Explanation

**What a Wallet Is**

A crypto wallet does not store currency — it stores private keys (or the seed phrase that derives them). The actual assets exist as state on the blockchain; the wallet is the tool that generates, stores, and uses private keys to produce valid signatures.

- **EOA (Externally Owned Account)**: Controlled by a private key. Any transaction signed by the private key is valid.
- **Smart Contract Wallet**: An ERC-4337 or Gnosis Safe wallet where the account is a contract. Validation logic (multi-sig, social recovery, spending limits) is programmable.

**HD Wallets and BIP-32/39/44**

Modern wallets use hierarchical deterministic (HD) key derivation:

- **BIP-39**: A 12 or 24-word mnemonic encodes 128–256 bits of entropy. Security: 2^128 to 2^256 possible seeds.
- **BIP-32**: Defines a tree of key derivation from the root seed using HMAC-SHA512. Any child key can be derived deterministically from the parent + derivation path.
- **BIP-44**: Standard derivation paths: `m/44'/60'/0'/0/index` for Ethereum addresses. The path encodes: purpose (44), coin type (60 = ETH), account, change, and address index.

**EIP-1193: The Browser Provider Standard**

Browser wallets (MetaMask) inject a provider object at `window.ethereum` implementing EIP-1193:

```javascript
// Request accounts
const accounts = await window.ethereum.request({ method: 'eth_requestAccounts' });

// Listen for account changes
window.ethereum.on('accountsChanged', (accounts) => { ... });

// Listen for chain changes
window.ethereum.on('chainChanged', (chainId) => { ... });

// Send a transaction
const txHash = await window.ethereum.request({
  method: 'eth_sendTransaction',
  params: [{ from, to, value, data, gas }]
});
```

**WalletConnect**

WalletConnect (v2) is a protocol for connecting mobile wallets to dApps. It uses a relay server for QR code pairing and encrypted message passing. The relay can be self-hosted. WalletConnect enables: mobile wallet connections, hardware wallet connections via bridge apps, and cross-device session persistence.

**ERC-4337: Account Abstraction**

ERC-4337 introduces a new transaction type — UserOperations — that allows smart contract wallets to be first-class accounts without protocol changes:

1. `EntryPoint` contract: A global singleton that validates and executes UserOperations.
2. `UserOperation`: A signed intent (not a direct transaction). Contains: `sender`, `callData`, `signature`, `paymasterAndData`.
3. `Bundlers`: Nodes that aggregate UserOperations into a single L1 transaction (like block proposers for UserOps).
4. `Paymasters`: Contracts that can sponsor gas fees for users, or accept ERC-20 tokens for gas instead of ETH.

Account abstraction enables: gasless transactions (paymaster sponsors gas), session keys (temporary limited signing keys), social recovery (approve transactions via multiple signers), and batch transactions.

### Why It Matters

Wallet UX is the largest bottleneck to blockchain adoption. ERC-4337 eliminates the requirement for users to hold ETH before they can do anything, enables one-click transactions (session keys), and makes recovery possible without a seed phrase. As an engineer, understanding wallet architecture lets you design better onboarding (gasless, social login) and avoid wallet integration bugs that lock users out.

### Real-World Context

- MetaMask has 30M+ monthly active users. Its EIP-1193 implementation has subtle bugs that require working around in production code (e.g., chain ID returned as hex string in some versions, number in others).
- Safe (Gnosis Safe) is the dominant multi-sig wallet, holding >$100B in assets. It is a smart contract wallet — predecessor to ERC-4337.
- Privy, Dynamic, and Turnkey provide embedded wallet services for apps that want to abstract away wallet complexity from non-crypto-native users.
- ZeroDev, Biconomy, and Stackup are infrastructure providers for ERC-4337 bundlers and paymasters.

### Tools / Technologies

| Tool                | Purpose                                   |
| ------------------- | ----------------------------------------- |
| MetaMask SDK        | Browser extension and mobile wallet       |
| WalletConnect v2    | Multi-wallet connection protocol          |
| `wagmi`             | React hooks library for wallet connection |
| `viem`              | TypeScript-native Ethereum library        |
| `permissionless.js` | ERC-4337 (smart contract wallet) SDK      |
| Safe SDK            | Multi-sig wallet integration              |

### Deep Mastery Questions

1. Derive the Ethereum address from a BIP-44 derivation path `m/44'/60'/0'/0/0` starting from a known seed phrase. What is each level of the path for?
2. What is the difference between `eth_sign` and `personal_sign` (EIP-191)? Why is `eth_sign` dangerous?
3. How does WalletConnect v2 work without requiring the dApp to know the user's IP address? What does the relay server see?
4. In ERC-4337, who pays the gas fee if there is no paymaster? What happens if neither the wallet nor the paymaster has ETH?
5. What is a "session key" in the context of account abstraction, and what security constraints must it have to be safe?
6. How does social recovery work in a smart contract wallet? What is the minimum number of guardians for this to be secure?
7. Why is losing a BIP-39 seed phrase an irreversible loss of funds? What does social recovery in ERC-4337 wallets change about this?

### Hands-On Exercises

1. **HD wallet generation**: Using `ethers.js`, generate an HD wallet from a mnemonic. Derive the first 5 addresses at the standard BIP-44 path. Verify the addresses match MetaMask for the same seed.
2. **EIP-1193 integration**: Build a minimal HTML/JS dApp that: (a) detects whether MetaMask is installed, (b) requests account access, (c) displays account address and ETH balance, (d) handles chain switching to Sepolia testnet.
3. **ERC-4337 UserOp**: Using ZeroDev SDK, deploy an ERC-4337 smart account on testnet. Send a gasless UserOperation (use their testnet paymaster). Verify the transaction on the block explorer.
4. **Multi-sig setup**: Deploy a Gnosis Safe on testnet with 2-of-3 multi-sig. Submit a transaction from one owner, confirm from a second, and execute.

### Mini Build Task

**Build a Wallet Connection Component.** A React component that: supports MetaMask, WalletConnect v2, and Coinbase Wallet simultaneously; handles chain switching (display an error if the user is on wrong chain, offer to switch); shows connected address, ENS name resolved, and ETH balance; handles disconnection and re-connection gracefully; and detects and handles the case where the user changes accounts or chains in MetaMask mid-session. This seemingly simple component requires handling a dozen edge cases.

---

## Topic 2: Web3 Libraries and the JSON-RPC API

### Explanation

**The JSON-RPC API**

Every Ethereum node exposes a JSON-RPC API — a stateless request-response protocol over HTTP or WebSocket. Key methods:

| Method                            | Purpose                                       |
| --------------------------------- | --------------------------------------------- |
| `eth_blockNumber`                 | Latest block number                           |
| `eth_getBalance(addr, block)`     | ETH balance at block                          |
| `eth_call(tx, block)`             | Execute a call without creating a transaction |
| `eth_estimateGas(tx)`             | Estimate gas for a transaction                |
| `eth_sendRawTransaction(rawTx)`   | Submit a signed transaction                   |
| `eth_getTransactionReceipt(hash)` | Get receipt (includes logs, status)           |
| `eth_getLogs(filter)`             | Get event logs matching a filter              |
| `eth_subscribe(event)`            | WebSocket subscription for new blocks/logs    |
| `debug_traceTransaction(hash)`    | Full execution trace (node-specific)          |

**ethers.js v6**

The most widely used JavaScript library for Ethereum:

```typescript
import { ethers } from "ethers";

// Connect to provider
const provider = new ethers.JsonRpcProvider(process.env.ETH_RPC_URL);

// Connect wallet
const wallet = new ethers.Wallet(privateKey, provider);

// Interact with contract
const contract = new ethers.Contract(address, abi, wallet);
const result = await contract.balanceOf(userAddress);

// Send transaction
const tx = await contract.transfer(recipient, amount);
const receipt = await tx.wait(); // Wait for 1 confirmation
```

**viem**

A newer, TypeScript-native library with a focus on type safety and tree-shakability:

```typescript
import { createPublicClient, createWalletClient, http, parseEther } from "viem";
import { mainnet } from "viem/chains";

const publicClient = createPublicClient({ chain: mainnet, transport: http() });
const balance = await publicClient.getBalance({ address: "0x..." });

const walletClient = createWalletClient({ chain: mainnet, transport: http() });
const hash = await walletClient.sendTransaction({
  to: "0x...",
  value: parseEther("1"),
});
```

**wagmi (React hooks)**

wagmi wraps viem with React hooks for state management, caching, and wallet connection:

```typescript
import { useReadContract, useWriteContract, useAccount } from 'wagmi'

function TokenBalance() {
  const { address } = useAccount()
  const { data: balance } = useReadContract({
    address: TOKEN_ADDRESS,
    abi: erc20Abi,
    functionName: 'balanceOf',
    args: [address],
  })
  return <div>{balance?.toString()}</div>
}
```

**Multicall**

Batch multiple `eth_call`s into a single RPC call using the Multicall3 contract (deployed at `0xcA11bde05977b3631167028862bE2a173976CA11` on most EVM chains):

```typescript
const multicall = await publicClient.multicall({
  contracts: [
    {
      address: TOKEN_A,
      abi: erc20Abi,
      functionName: "balanceOf",
      args: [user],
    },
    {
      address: TOKEN_B,
      abi: erc20Abi,
      functionName: "balanceOf",
      args: [user],
    },
    {
      address: TOKEN_C,
      abi: erc20Abi,
      functionName: "balanceOf",
      args: [user],
    },
  ],
});
// Returns: [balanceA, balanceB, balanceC] in one RPC request
```

### Why It Matters

Understanding the raw JSON-RPC layer lets you debug library issues, build custom tooling, and reason about performance. Choosing the right library (ethers vs viem) and integration pattern (wagmi vs custom hooks) has significant impact on bundle size, type safety, and developer experience. Multicall is essential for performant dApps — a dApp that makes 50 individual RPC calls for a portfolio view will be slow and rate-limited.

### Real-World Context

- Infura and Alchemy impose rate limits (e.g., 300 requests/second on paid plans). A dApp making individual RPC calls per user will hit limits quickly — multicall reduces RPC usage by 10–50x.
- viem is now the primary library for new Ethereum projects (replacing ethers.js) due to its full TypeScript support and smaller bundle.
- wagmi v2 powers wallet interactions for Uniswap, Coinbase, and many other major dApps.

### Tools / Technologies

| Tool                    | Purpose                                   |
| ----------------------- | ----------------------------------------- |
| `ethers.js` v6          | Comprehensive Ethereum JS library         |
| `viem`                  | Modern TypeScript-native Ethereum library |
| `wagmi` v2              | React hooks for Ethereum                  |
| `@tanstack/react-query` | Data fetching/caching (used by wagmi)     |
| Multicall3              | Batch RPC calls                           |
| `eth-sdk`               | Type-safe contract bindings generation    |

### Deep Mastery Questions

1. What is the difference between `eth_call` and `eth_sendTransaction`? Can `eth_call` mutate state?
2. Why is `eth_estimateGas` not a reliable upper bound for gas usage? What conditions could cause a transaction to consume more gas than estimated?
3. What is the difference between polling `eth_getTransactionReceipt` and using `eth_subscribe` to listen for new blocks? What are the trade-offs?
4. How does the Multicall3 contract work? What failure mode should you handle when one of the batched calls reverts?
5. How would you implement a rate-limited, failover-capable RPC provider that falls back from Alchemy to Infura to a self-hosted node?
6. Why does `tx.wait(1)` (wait for one confirmation) not give you finality guarantee on Ethereum PoS? What does `tx.wait(64)` not guarantee?

### Hands-On Exercises

1. **Raw RPC**: Make all major JSON-RPC calls using raw `curl`. Fetch a block, get a balance, estimate gas, and submit a transaction. Parse the responses manually.
2. **Multicall portfolio**: Given 10 token addresses and 5 user addresses, fetch all 50 balances in one multicall. Measure round-trip time vs. 50 individual calls.
3. **Event subscription**: Build a script that subscribes to all ERC-20 Transfer events on a specific token contract via WebSocket. Print sender, receiver, and amount in real time.
4. **Provider failover**: Implement a provider that tries 3 different RPC endpoints in order, falling back on error. Test by intentionally making the first two fail.

### Mini Build Task

**Build a DeFi Portfolio Tracker.** Given a wallet address, fetch (via multicall): ETH balance, ERC-20 token balances for the top 20 tokens, and Uniswap V3 LP positions. Display balances with USD values (from CoinGecko API or an on-chain oracle). Handle: rate limits, missing tokens, failed calls in multicall gracefully. Target: all data fetched in <2 RPC calls.

---

## Topic 3: Contract Interaction and Transaction Lifecycle

### Explanation

**Transaction Lifecycle (Full Detail)**

1. **Construction**: Assemble transaction parameters. For contract calls: ABI-encode `(selector + args)` as `data`.
2. **Simulation**: Call `eth_call` with the same parameters. If it reverts, surface the error before spending gas.
3. **Gas estimation**: Call `eth_estimateGas`. Apply a buffer (1.2–1.5x) for non-deterministic functions.
4. **Fee determination**: Fetch current `baseFee` (from block header). Query user for `maxPriorityFeePerGas` preference.
5. **Signing**: Sign the typed transaction (EIP-1559 Type 2) with the user's private key.
6. **Submission**: Send to mempool via `eth_sendRawTransaction`. Receive `txHash` immediately.
7. **Pending**: Transaction is in the mempool. May be accelerated (replace with higher tip, same nonce), cancelled (send 0-value to self with same nonce, higher tip), or stuck (insufficient tip during congestion).
8. **Inclusion**: Transaction included in a block. `eth_getTransactionReceipt` returns `blockNumber`, `status` (1=success, 0=revert), `logs`.
9. **Finality**: PoS: after 2 epochs (~13 minutes), the block is finalized. Application policy determines acceptable confirmation depth.
10. **Reorg handling**: Monitor for block reorganizations. If the block containing the transaction is orphaned, the transaction may need resubmission.

**Decoding Revert Reasons**

When a transaction reverts, the receipt status is 0. The revert reason is encoded in the return data:

- `Error(string)`: Standard revert — ABI-decode as `(string)` after the `0x08c379a0` selector.
- Custom errors: `MyCustomError(args)` — decode using the contract ABI.
- OOG (out-of-gas): No return data; receipt gas used ≈ gas limit.

**Event Decoding**

Events are stored in transaction receipts as `logs`. Each log contains:

- `address`: The contract that emitted the event.
- `topics[0]`: Keccak-256 of the event signature (e.g., `Transfer(address,address,uint256)`).
- `topics[1..n]`: ABI-encoded indexed parameters.
- `data`: ABI-encoded non-indexed parameters.

**Transaction Replacement and Cancellation**

A transaction in the mempool can be replaced by sending a new transaction with the same nonce but higher `maxPriorityFeePerGas` (typically +10%+). Most mempools enforce a minimum fee bump to replace. Cancel by sending to yourself with 0 value at the same nonce.

**The Sign-Then-Submit Pattern**

For operations where the user should not be exposed to blockchain latency in the UI interaction:

1. Prompt the user to sign a typed message (no gas cost).
2. Backend verifies the signature and performs the on-chain action.
3. Used for: access control gates, off-chain orders (0x, OpenSea), meta-transactions.

### Why It Matters

Transaction management is where most dApp UX fails. A dApp that doesn't handle stuck transactions, doesn't decode revert reasons clearly, or loses track of pending transactions creates a confusing and frustrating user experience. For financial applications, transaction lifecycle correctness is a safety property — a missed reorg or a double-submission could result in financial loss.

### Real-World Context

- The 2021 NFT minting "gas wars" (e.g., Bored Ape Yacht Club) required minting bots to manage transaction replacement in real-time to win the block inclusion race.
- Uniswap's frontend always simulates transactions before submitting them, showing the user an error message if the swap would revert, rather than wasting gas.
- The Flashbots RPC endpoint (`flashbots_sendBundle`) allows submitting bundles of transactions to specific blocks without the mempool, preventing front-running.

### Tools / Technologies

| Tool                             | Purpose                                       |
| -------------------------------- | --------------------------------------------- |
| Tenderly                         | Transaction simulation, debugging, monitoring |
| Flashbots Protect RPC            | MEV-protected transaction submission          |
| `cast send` (Foundry)            | CLI transaction submission                    |
| `ethers.js` transaction handling | `tx.wait()`, `tx.replaceTransaction()`        |

### Deep Mastery Questions

1. What is the exact sequence of events when a transaction "fails" (reverts)? Does gas get spent? What appears on the blockchain?
2. How would you implement automatic transaction "speed-up" in your dApp? What are the edge cases (transaction already mined before replacement is submitted)?
3. Why can't you rely solely on `eth_getTransactionReceipt` returning a result to conclude that a transaction is confirmed? What else must you check?
4. What happens to a pending transaction if it becomes stuck (below base fee) during a period of high congestion? Will it be included eventually?
5. Explain the mechanics of a "sandwich attack" from the attacker's perspective. What transactions are submitted, in what order, and why?
6. What is the "nonce gap" problem? If a user accidentally sends transaction with nonce 5 to two different nodes, what happens?

### Hands-On Exercises

1. **Transaction replacement**: Write a script that sends a transaction with a low tip, detects that it's stuck (not mined within 30 seconds), and replaces it with a higher tip. Repeat until mined.
2. **Revert decoding**: Write a function that takes a transaction hash of a failed transaction and decodes the full revert reason (handling both `Error(string)` and custom errors). Test it against known reverting transactions.
3. **Event-driven indexing**: Write a script that fetches all `Transfer` events from USDC since block 18,000,000. Decode each event and aggregate total volume.
4. **Simulation before submission**: Before submitting any transaction to a sample contract, run `eth_call` with the same parameters. If it fails, display the reason. Only proceed if simulation succeeds.

### Mini Build Task

**Build a Transaction Manager Library.** A TypeScript class that: submits transactions, monitors their status (polling, with exponential backoff), detects stuck transactions and offers replacement, handles reorgs (monitors for block reorganizations and resubmits affected transactions), and decodes revert reasons. Test it under adverse conditions: simulate network failures, OOG reverts, and nonce conflicts.

---

## Topic 4: Blockchain Data Indexing with The Graph

### Explanation

**Why Raw RPC is Insufficient**

`eth_getLogs` with a filter works for recent data, but:

- Ethereum limits log query ranges (~10,000 blocks or 2,000 results per request).
- No aggregation, joins, or computed fields — you get raw log bytes.
- No relationship tracking across events.
- Historical queries require iterating through potentially millions of blocks.

For a dApp displaying "all swap events for a user's address in the last year," raw RPC is completely impractical.

**The Graph Protocol**

The Graph is a decentralized indexing protocol. Developers write **subgraphs** — definitions of which events to capture and how to store them — and Graph nodes index the data.

```yaml
# subgraph.yaml
dataSources:
  - kind: ethereum/contract
    name: UniswapV3Pool
    network: mainnet
    source:
      address: "0x..."
      abi: UniswapV3Pool
      startBlock: 12369621
    mapping:
      kind: ethereum/events
      eventHandlers:
        - event: Swap(indexed address,indexed address,int256,int256,uint160,uint128,int24)
          handler: handleSwap
```

```typescript
// mapping.ts (AssemblyScript, compiled to WASM)
export function handleSwap(event: SwapEvent): void {
  let swap = new Swap(event.transaction.hash.toHex());
  swap.sender = event.params.sender;
  swap.amount0 = event.params.amount0;
  swap.timestamp = event.block.timestamp;
  swap.save();
}
```

Subgraphs expose a **GraphQL API** for querying indexed data:

```graphql
query {
  swaps(where: { sender: "0x..." }, orderBy: timestamp, orderDirection: desc) {
    id
    amount0
    amount1
    timestamp
  }
}
```

**Custom Indexers: Ponder and Envio**

**Ponder** (Node.js) and **Envio** (Rust) are alternatives to The Graph for self-hosted indexing:

- Written in TypeScript/JavaScript (Ponder) — familiar developer experience.
- Faster sync times, local development with hot-reloading.
- Full SQL queries on indexed data.
- Self-hosted — no decentralization, but no indexer centralization risk.

**When to Use What**

| Approach               | When                                                                  |
| ---------------------- | --------------------------------------------------------------------- |
| `eth_getLogs`          | Simple, one-time queries over small block ranges                      |
| The Graph (Hosted)     | Cross-team, external data sharing, GraphQL API                        |
| Ponder (self-hosted)   | Custom indexing with TypeScript, internal use                         |
| Dune Analytics         | Ad-hoc analytics with SQL, cross-protocol queries                     |
| Alchemy/Quicknode APIs | Enhanced APIs for specific data types (NFT metadata, token transfers) |

### Why It Matters

Virtually every production dApp with non-trivial data requirements uses an indexer. Without indexing, leaderboards, history, analytics, and aggregate views are impossible. The Graph's hosted service has experienced outages that took major DeFi protocols offline — understanding its architecture and failure modes helps you design resilient systems with fallback indexers.

### Tools / Technologies

| Tool           | Purpose                                   |
| -------------- | ----------------------------------------- |
| `graph-cli`    | The Graph subgraph CLI                    |
| Graph Studio   | Hosted subgraph deployment                |
| `ponder`       | TypeScript-native Ethereum indexer        |
| `envio`        | Rust-based hypersync indexer              |
| Dune Analytics | SQL-based blockchain analytics            |
| Goldsky        | Managed subgraph hosting with mirror sync |

### Deep Mastery Questions

1. Why can't you use `eth_getLogs` directly to serve a production dApp's "transaction history" page? What specific limits does this approach hit?
2. In The Graph's architecture, what happens if a subgraph handler throws an unhandled error? What data is lost?
3. What is the "determinism requirement" for subgraph handlers? Why can't a handler make HTTP calls to external APIs?
4. How does The Graph handle chain reorganizations in its indexed data? What happens to entities created from orphaned blocks?
5. Compare the trust model of using The Graph's decentralized network vs. hosting your own indexer. What are the security implications of each?
6. What is "time travel" querying in The Graph, and why is it important for analytics dashboards?

### Hands-On Exercises

1. **Write a subgraph**: Create a subgraph for a simple ERC-20 token contract. Index Transfer events. Expose: per-address total received, per-address total sent, and a global transfer history.
2. **GraphQL querying**: Query Uniswap V3's official subgraph for: the 10 highest-volume pools in the last 24 hours, all swaps by a specific address, and the price history for ETH/USDC.
3. **Ponder indexer**: Build a local Ponder indexer for the same ERC-20 token. Compare development experience, sync speed, and query flexibility vs. The Graph.
4. **Reorg handling test**: In your local Graph node, artificially introduce a block reorg. Observe how the indexed data is rolled back and re-indexed.

### Mini Build Task

**Build a DEX Analytics Dashboard.** Create a subgraph for a Uniswap V2 fork (or use the official Uniswap subgraph). Display in real-time: top pools by volume (24h), price charts for selected pairs, and per-wallet trade history. The application must: handle loading states, error states, and stale data gracefully; refetch data every 30 seconds; and display data within 2 seconds of page load (using caching).

---

## Topic 5: Backend and Off-Chain Architecture

### Explanation

**When You Need a Backend**

Blockchains are expensive for computation and storage. Some logic belongs off-chain:

- **Computation-heavy operations**: Compute off-chain, verify on-chain (using commitments or ZK proofs).
- **User authentication and session management**: Rely on EVM signature verification (Sign-In With Ethereum, EIP-4361).
- **Notification systems**: Webhooks for on-chain events → email/push notifications.
- **Order books**: Off-chain order matching with on-chain settlement (0x Protocol, Seaport).
- **Analytics and reporting**: Heavy data aggregation.
- **Content delivery**: Media files, metadata hosted on IPFS but served via a CDN gateway.

**Sign-In With Ethereum (SIWE, EIP-4361)**

Authentication without a password or centralized provider:

1. Backend sends a nonce and terms to the client.
2. Client signs the message with their Ethereum account (EIP-191 personal sign).
3. Backend verifies the signature and associates the session with the Ethereum address.
4. Session is a JWT or server-side session tied to the address.

```typescript
// SIWE message construction
import { SiweMessage } from "siwe";
const message = new SiweMessage({
  domain: "app.example.com",
  address: userAddress,
  statement: "Sign in to Example App",
  uri: "https://app.example.com",
  version: "1",
  chainId: 1,
  nonce: generateNonce(),
  issuedAt: new Date().toISOString(),
});
const prepared = message.prepareMessage();
const signature = await wallet.signMessage(prepared);
// Backend verifies: message.verify({ signature })
```

**Meta-Transactions (ERC-2771)**

Users sign a message off-chain; a trusted "relayer" submits the transaction on-chain and pays gas. The target contract uses `_msgSender()` which returns the original signer (extracted from the forwarded call data) rather than the relayer's address.

This enables gasless UX — users never hold ETH, but still interact with contracts as if they are the sender.

**Off-Chain Order Books with On-Chain Settlement**

The 0x Protocol pattern:

1. Market maker signs an off-chain order (EIP-712 typed data): `{makerToken, takerToken, makerAmount, takerAmount, expiry}`.
2. Order is stored in an off-chain order book (0x API).
3. Taker submits the signed order to the exchange contract on-chain, which validates the signature and executes the swap atomically.

Benefits: No gas cost for order placement/cancellation. On-chain cost only for executed trades.

**Webhook Architecture for dApps**

Alchemy Notify, Moralis, and The Graph can trigger webhooks when specific on-chain events occur (e.g., NFT transferred to an address, a large trade executed). This powers: notification systems, backend state updates, fraud detection.

### Real-World Context

- OpenSea uses Seaport Protocol for on-chain order execution, with an off-chain order book managed by their API. Orders are signed off-chain (no gas); settlement is on-chain.
- Stripe's fiat-to-crypto onramp integrates with SIWE for proving wallet ownership.
- Gas Station Network (GSN) is an open-source relayer network implementing meta-transactions.

### Tools / Technologies

| Tool               | Purpose                                  |
| ------------------ | ---------------------------------------- |
| `siwe` library     | Sign-In With Ethereum implementation     |
| ERC-2771 / OpenGSN | Meta-transaction / relayer framework     |
| Alchemy Notify     | On-chain event webhooks                  |
| 0x Protocol        | Off-chain orderbook, on-chain settlement |
| Seaport (OpenSea)  | NFT marketplace protocol                 |

### Deep Mastery Questions

1. In a SIWE flow, what prevents a phishing attack where a malicious website presents the same nonce to the user and captures their signature?
2. In ERC-2771 (meta-transactions), what prevents a malicious relayer from forging the original sender's address in the forwarded call?
3. Why does off-chain order book + on-chain settlement create a better UX and economic model than a fully on-chain order book?
4. What is the trust model for an app that uses webhooks from Alchemy for critical operations (e.g., crediting a user's in-app balance when they deposit on-chain)? What attack is possible?
5. How would you design a system that provides notifications within 1 second of an on-chain event while remaining trustless?

### Hands-On Exercises

1. **SIWE authentication**: Build a Next.js app with SIWE authentication. Users log in with MetaMask. The backend issues a JWT tied to their Ethereum address. Restrict certain API routes to authenticated addresses.
2. **Meta-transaction**: Deploy an ERC-2771-compatible ERC-20 token. Deploy a forwarder contract. Build a relayer service that accepts signed meta-transactions and submits them. Allow a user with no ETH to transfer tokens.
3. **Webhook integration**: Set up an Alchemy webhook that triggers when any address sends ETH to a specified vault address. Log each deposit to a database within 2 seconds.

### Mini Build Task

**Build a Gasless NFT Minting dApp.** Users sign a minting request off-chain. Your relayer service (a simple Express.js server) validates the request, checks eligibility (from a Merkle allowlist), and submits the mint transaction, paying gas. The NFT contract uses ERC-2771 to record the user as the minter, not the relayer. Deploy on Sepolia testnet.

---

## Topic 6: Production Deployment and Infrastructure

### Explanation

**RPC Provider Strategy**

Never rely on a single RPC endpoint:

- **Single provider failure**: Alchemy, Infura, and QuickNode have had outages. A resilient dApp must fall back gracefully.
- **Rate limiting**: High-traffic dApps regularly hit rate limits.
- **Geographic latency**: Serve users from the nearest provider endpoint.

Pattern: **Primary provider** (Alchemy/QuickNode) → **Fallback provider** (Infura) → **Self-hosted node** (Geth/Erigon, for critical operations).

**IPFS and Decentralized Storage**

- **Pinata**: Managed IPFS pinning service. Upload files, get IPFS CIDs, serve via their gateway.
- **web3.storage** (now Storacha): Filecoin-backed storage with IPFS CIDs.
- **Self-hosted IPFS**: Full control, but requires operational overhead.
- **Filecoin**: Blockchain-based storage with cryptographic proofs of storage (PoSt, PoRep). Decentralized but complex for direct use.

For NFT metadata: Pin to multiple services. Never use a centralized URL as the `tokenURI`.

**Smart Contract Deployment Strategy**

1. **Local development**: Foundry Anvil or Hardhat Network — instant, no cost.
2. **Testnet**: Sepolia (Ethereum), Amoy (Polygon), Arbitrum Sepolia. Test full deployment flow.
3. **Mainnet**: Use a deployment script (Foundry `forge script` or `hardhat-deploy`). Verify contracts on Etherscan immediately after deployment.
4. **Contract verification**: Post ABI and source to Etherscan/Blockscout for public auditability.
5. **Multi-sig ownership**: Transfer contract ownership to a Gnosis Safe immediately after deployment. Remove single private key control.

**Monitoring and Alerting**

Essential monitoring for production dApps:

- **Transaction monitoring**: Detect failed transactions, gas spikes, unusual activity.
- **Oracle deviation alerts**: Alert if a price feed deviates by >5% in one block.
- **TVL/balance monitoring**: Alert if a vault's balance changes dramatically.
- **Event monitoring**: Custom alerts for specific emitted events (governance proposals, large transfers).

Tools: Forta Network (decentralized bot monitoring), OpenZeppelin Defender, Tenderly Alerts, custom CloudWatch/Grafana dashboards.

**Security Hardening of the App Stack**

| Layer          | Threat                                                         | Mitigation                                                                   |
| -------------- | -------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| Frontend       | Malicious code injection (supply chain)                        | Content Security Policy, Subresource Integrity                               |
| DNS            | DNS hijacking                                                  | DNSSEC, IPFS hosting (ENS)                                                   |
| RPC            | Man-in-the-middle                                              | HTTPS only, never trust unsigned RPC responses for critical operations       |
| Backend        | SSRF (reading blockchain or calling contracts through backend) | Validate all inputs, use allowlists                                          |
| Smart contract | Already deployed bugs                                          | Immutable contracts + bug bounty; upgradeable contracts + monitoring + pause |

**IPFS Frontend Hosting**

For maximum censorship resistance, host the frontend on IPFS and serve via ENS:

1. Build frontend to static files.
2. Upload to IPFS (get CID).
3. Update ENS `contenthash` to the CID.
4. Users with ENS-aware browsers can access `app.uniswap.eth` pointing to your IPFS-hosted frontend.

### Why It Matters

Production failures in dApps typically aren't smart contract bugs — they're infrastructure failures: RPC downtime, IPFS gateway failures, event handling bugs, insufficient monitoring. The 2017 CryptoKitties launch exposed Ethereum's scalability limits. The 2022 Wintermute hack started with a vulnerable vanity address generator — a tooling security issue. Complex systems fail at the boundaries. This stage teaches you to design and monitor those boundaries.

### Real-World Context

- OpenZeppelin Defender monitors large DeFi protocols including Compound and Aave. It detected and automatically paused a compromised protocol in 2023.
- Uniswap's frontend is deployed to both Vercel (centralized) and IPFS/ENS (decentralized). Users can access both.
- Forta Network has deployed 700+ detection bots that monitor for unusual on-chain activity in real-time.

### Tools / Technologies

| Tool                   | Purpose                                         |
| ---------------------- | ----------------------------------------------- |
| OpenZeppelin Defender  | Smart contract monitoring, automation           |
| Tenderly               | Transaction simulation, alerting, observability |
| Forta Network          | Decentralized threat detection                  |
| Pinata                 | IPFS pinning service                            |
| Foundry `forge script` | Deployment scripting                            |
| Blockscout             | Open-source block explorer (self-hostable)      |

### Deep Mastery Questions

1. What is the difference between deploying a smart contract with a private key (EOA) vs. through a Gnosis Safe? Why should all production contracts be owned by a multi-sig?
2. If your dApp's frontend is hijacked (DNS attack swapping the legitimate site with a malicious one), what on-chain harm can attackers cause? What can they NOT cause?
3. How does the `contenthash` ENS record work for IPFS-hosted frontends? What are the trust assumptions?
4. Design a monitoring system that detects: (a) an unusual spike in liquidations at a lending protocol, (b) a price oracle deviation of >10%, and (c) a large unauthorized withdrawal from a treasury. What data sources and tooling would you use?
5. What is Subresource Integrity (SRI) and how does it protect against supply chain attacks on dApp frontends?

### Hands-On Exercises

1. **Multi-provider setup**: Configure a viem transport with 3 RPC endpoints using fallback logic. Simulate the primary going offline and verify fallback activates.
2. **Deployment script**: Write a Foundry `forge script` that: deploys a contract, verifies it on Etherscan, and transfers ownership to a specified Gnosis Safe.
3. **IPFS upload**: Build, upload a React dApp to IPFS via Pinata, and verify the CID is accessible via multiple public gateways.
4. **Monitoring bot**: Write a Forta bot that alerts when a specific ERC-20 token has a single transfer > $1M equivalent. Deploy to Forta testnet.

### Mini Build Task

**Deploy a Production-Ready dApp.** Take your AMM from Stage 4 or your lending protocol from Stage 3. Build a full production deployment: (a) deploy contracts to Sepolia with Foundry scripts; (b) verify on Etherscan; (c) transfer ownership to a testnet Gnosis Safe; (d) build a React frontend with wagmi; (e) deploy frontend to IPFS; (f) set up Tenderly monitoring for 3 critical events; (g) write a post-deployment runbook (how to pause, upgrade, and monitor the system).

---

## Mini Projects

### Project 1: Full-Stack NFT Mint Site

End-to-end: ERC-721 contract with Merkle-proof allowlist minting, metadata on IPFS, a React frontend with wagmi wallet connection, reveal mechanism (pre-reveal placeholder → post-reveal real metadata via Chainlink VRF), and Etherscan-verified deployment on testnet.

### Project 2: On-Chain Governance Dashboard

Frontend for a DAO governor contract: display all active proposals (via subgraph), show vote counts, allow token holders to cast votes, and show the timelock queue. Handle: wallet connection, token delegation, and proposal execution transactions.

### Project 3: Cross-Chain Bridge Monitor

Build a monitoring tool that tracks the state of assets bridged between Ethereum mainnet and an OP Stack L2. Show: pending deposits (not yet relayed), completed withdrawals, and the 7-day challenge window status for optimistic rollup withdrawals.

---

## Capstone Project: Full-Stack DeFi Application — DEX with Analytics

Build a complete, production-grade DEX (decentralized exchange) frontend and backend for your AMM from Stage 4:

1. **Frontend**: React/Next.js with wagmi. Swap interface with: token selection, price impact display (calculated from contract state), slippage tolerance setting, real-time price updates (via event subscription), and transaction status tracking.
2. **Subgraph**: Index all swap and liquidity events. Expose GraphQL API for: pool statistics, price history, per-user trade history.
3. **Analytics**: Display candlestick chart for the token pair (from indexed swap events), TVL over time, volume bars (24h buckets).
4. **Backend**: Node.js server: SIWE authentication, user trade history, email notifications for large trades (webhook-triggered).
5. **Infrastructure**: Deploy contracts on Sepolia (Foundry script, Etherscan-verified). Host frontend on Vercel + IPFS. Self-host a Graph node with your subgraph.
6. **Monitoring**: Tenderly alerts for OOG transaction spikes and anomalous trade sizes.

---

## Common Mistakes & Misconceptions

| Mistake                                     | Reality                                                                                                                                      |
| ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| "The RPC will always respond"               | Build retry and failover logic. Treat RPC endpoints as unreliable service dependencies.                                                      |
| "Gas estimation is exact"                   | It's a simulation result. Add a buffer. Gas usage can differ at execution time due to state changes between estimate and inclusion.          |
| "Transaction confirmed = transaction final" | Confirmed ≠ finalized. On PoS Ethereum, PoS finality takes ~13 minutes. For high-value transactions, wait for finality or PoS justification. |
| "The Graph never lies"                      | Subgraphs can lag, miss events, or have handler bugs. Always validate critical on-chain data via direct RPC as a secondary check.            |
| "IPFS is permanent"                         | IPFS data persists only as long as someone pins it. Without multiple pinners, content can disappear.                                         |
| "ENS resolves for all users"                | ENS resolution requires a compatible browser or dApp. Most users access dApps via `app.example.com` DNS, not ENS.                            |

---

## Security Considerations

- **Never store private keys in environment variables in production.** Use hardware security modules (HSMs), AWS KMS, or Vault for key management.
- **Validate all on-chain data before trusting it.** RPC providers can serve stale or incorrect data. For financial decisions, verify against multiple sources.
- **Frontend supply chain attacks are real.** Compromised npm packages have injected address-replacing code in DeFi frontends. Use Subresource Integrity, pin dependency versions, and audit packages.
- **Unsigned API responses are malleable.** An attacker between your dApp and your backend can modify API responses. Sign critical data with a known key or derive it directly from on-chain state.

---

## Readings & Resources

**Documentation**

- wagmi v2 documentation: wagmi.sh
- viem documentation: viem.sh
- The Graph documentation: thegraph.com/docs
- EIP-4361 (SIWE): eips.ethereum.org/EIPS/eip-4361
- ERC-4337 specification: eips.ethereum.org/EIPS/eip-4337
- Foundry book (forge script): book.getfoundry.sh/forge/deploying

**Technical Blogs**

- Rainbow wallet engineering blog (wallet UX patterns)
- OpenZeppelin blog (production deployment practices)
- Alchemy blog (RPC and node performance)
- Paradigm engineering blog (dApp architecture)

**Tools Reference**

- `evm.codes` — Opcode gas costs
- `ethernodes.org` — Node client diversity
- `l2beat.com` — L2 security analysis
- `defillama.com` — TVL and protocol analytics

---

<div style="display:flex;align-items:center;justify-content:space-between;padding:14px 0;margin-top:48px;">
  <a href="./04_Ecosystem_Scaling_and_Crypto_Economics.md" style="display:flex;align-items:center;gap:8px;text-decoration:none;color:#8b949e;font-size:0.875rem;padding:6px 10px;border-radius:6px;">
    <svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor"><path d="M9.78 12.78a.75.75 0 0 1-1.06 0L4.47 8.53a.75.75 0 0 1 0-1.06l4.25-4.25a.75.75 0 0 1 1.06 1.06L6.06 8l3.72 3.72a.75.75 0 0 1 0 1.06z"/></svg>
    <span style="display:flex;flex-direction:column;line-height:1.3;">
      <span style="font-size:0.7rem;opacity:0.55;text-transform:uppercase;letter-spacing:0.06em;">Previous</span>
      <span style="font-weight:500;">04 · Ecosystem & Scaling</span>
    </span>
  </a>
  <span style="font-size:0.7rem;color:#8b949e;opacity:0.5;letter-spacing:0.08em;text-transform:uppercase;">Stage 5 of 6 · Full-Stack dApp</span>
  <a href="./06_Advanced_Protocol_Engineering.md" style="display:flex;align-items:center;gap:8px;text-decoration:none;color:#8b949e;font-size:0.875rem;padding:6px 10px;border-radius:6px;">
    <span style="display:flex;flex-direction:column;line-height:1.3;text-align:right;">
      <span style="font-size:0.7rem;opacity:0.55;text-transform:uppercase;letter-spacing:0.06em;">Next</span>
      <span style="font-weight:500;">06 · Advanced Protocol</span>
    </span>
    <svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor"><path d="M6.22 3.22a.75.75 0 0 1 1.06 0l4.25 4.25a.75.75 0 0 1 0 1.06l-4.25 4.25a.75.75 0 0 1-1.06-1.06L9.94 8 6.22 4.28a.75.75 0 0 1 0-1.06z"/></svg>
  </a>
</div>
