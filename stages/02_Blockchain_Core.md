# Stage 2 — Blockchain Core: The Protocol Layer

`Stage 2 of 6` · Estimated 4–6 weeks

## Stage Overview

### What This Stage Builds

This stage takes the foundational primitives from Stage 1 and assembles them into a working understanding of blockchain protocols. You will learn how transactions are created, validated, and ordered into blocks; how nodes communicate to form a coherent network; how consensus mechanisms (Proof of Work, Proof of Stake) achieve agreement among untrusted participants; and how forks, reorganizations, and chain selection rules maintain a single coherent history.

### Why This Stage Is Critical

This is where theory becomes protocol. Every blockchain — Bitcoin, Ethereum, Solana, Cosmos — is a specific instantiation of the primitives you learned in Stage 1. Understanding the protocol layer means understanding _why_ design decisions were made, not just _what_ they are. Engineers who skip this stage build dApps without understanding the substrate they run on — they are surprised by reorgs, confused by finality, and vulnerable to consensus-level attacks.

### Competence at the End of This Stage

You can trace a transaction from creation to finality across the entire protocol stack. You can explain the security model of PoW and PoS from first principles. You can reason about fork choice rules, chain reorganizations, and finality delays. You can analyze a blockchain's data model and understand why it constrains or enables certain application patterns.

---

## Conceptual Map

```text
Blockchain as State Machine
├── State, transitions, determinism
└── Accounts vs UTXO models
        │
        ▼
Transactions
├── Structure and encoding
├── Signature validation
├── Nonce management
├── Fee mechanisms
└── Transaction lifecycle
        │
        ▼
Blocks
├── Block structure and headers
├── Block production
├── Block validation rules
└── Block propagation
        │
        ▼
Nodes & Networking
├── Full nodes, light nodes, archive nodes
├── Mempool dynamics
├── Block gossip
└── Sync strategies (full, fast, snap)
        │
        ▼
Consensus Mechanisms
├── Proof of Work
│   ├── Mining difficulty
│   ├── Hash rate and security
│   └── Energy and hardware
├── Proof of Stake
│   ├── Validator selection
│   ├── Slashing
│   └── Finality (Casper FFG)
└── Fork choice rules
        │
        ▼
Forks & Reorganizations
├── Soft forks vs hard forks
├── Chain reorganizations
├── Finality models (probabilistic vs deterministic)
└── Governance and upgrade mechanisms
        │
        ▼
Token Standards & Data Models
├── Native tokens vs smart contract tokens
├── ERC-20, ERC-721, ERC-1155
├── UTXO vs Account model (deep comparison)
└── State growth and pruning
```

---

## Topic 1: Blockchain as a State Machine

### Explanation

A blockchain is a **replicated, deterministic state machine**. This is the most precise and useful mental model.

- **State**: The complete snapshot of all relevant data at a given point in time. In Ethereum, this is the set of all accounts (each with balance, nonce, code, and storage). In Bitcoin, this is the set of all unspent transaction outputs (UTXOs).

- **State transition function**: A deterministic function that takes the current state and a transaction, and produces a new state:

  ```text
  S' = STF(S, tx)
  ```

  "Deterministic" means: given the same state and the same transaction, every node in the network must compute exactly the same new state.

- **Block**: A batch of transactions applied sequentially to transition the state:

  ```text
  S_{n+1} = apply(S_n, Block_n)
  ```

  where `apply` processes each transaction in the block in order.

- **Blockchain**: A sequence of blocks, each referencing the previous block, forming a chain of state transitions from genesis (initial state) to the current state.

**The UTXO Model (Bitcoin)**

State = set of unspent transaction outputs. Each UTXO is `(txid, output_index, value, locking_script)`. A transaction _consumes_ UTXOs (inputs) and _creates_ new UTXOs (outputs). The sum of input values must equal the sum of output values plus fees.

Key property: UTXOs are **atomic** — they are either fully spent or fully unspent. There is no concept of a "balance" at the protocol level; a wallet's balance is the sum of its UTXOs.

**The Account Model (Ethereum)**

State = mapping from addresses to account objects. Each account has: `{nonce, balance, codeHash, storageRoot}`. Externally Owned Accounts (EOAs) have no code; Contract Accounts have code that executes when called.

Key property: State is **mutable in place** — a transaction updates the sender's and receiver's balance directly.

### Why It Matters

The state machine model is the foundation for reasoning about correctness, determinism, and consensus. If the state transition function is not perfectly deterministic, nodes diverge and the chain forks. Understanding which model (UTXO vs account) a chain uses shapes everything: transaction construction, parallel execution, privacy, and smart contract capability.

### Real-World Context

- Bitcoin's UTXO model enables straightforward parallel validation (UTXOs are independent) but makes complex state logic (e.g., multi-step DeFi interactions) awkward.
- Ethereum's account model enables rich stateful smart contracts but makes state growth a serious scalability concern (~1.5 TB for a full archive node as of 2024).
- Cardano uses an extended UTXO model (eUTXO) that adds datums and scripts to UTXOs, attempting to combine UTXO benefits with smart contract capability.
- Solana uses an account model but with a unique parallel execution model (Sealevel) enabled by explicit state access declarations.

### Tools / Technologies

| Tool                      | Purpose                                         |
| ------------------------- | ----------------------------------------------- |
| Bitcoin Core (`bitcoind`) | Reference Bitcoin UTXO implementation           |
| Geth (`go-ethereum`)      | Reference Ethereum account-model implementation |
| `btcd` (Go)               | Bitcoin full node in Go, well-documented        |
| `ethereumjs-vm`           | JavaScript EVM implementation for study         |

### Deep Mastery Questions

1. In the UTXO model, what does a "double spend" look like mechanically? How is it detected and prevented?
2. In the account model, what is the purpose of the nonce? What happens if two transactions from the same account have the same nonce?
3. Why does Bitcoin's UTXO model make certain forms of parallelism easier than Ethereum's account model?
4. If the state transition function produced different results on two nodes (a non-determinism bug), what would happen to the network?
5. Why does Ethereum need a concept of "gas" while Bitcoin does not? How is this related to the expressiveness of their respective state transition functions?
6. In the UTXO model, how does a wallet determine "its balance"? What are the implications for privacy?
7. What are the trade-offs between Cardano's eUTXO model and Ethereum's account model for DeFi applications?

### Hands-On Exercises

1. **UTXO chain**: Implement a minimal UTXO-based blockchain in Python. Create UTXOs, construct transactions that spend UTXOs and create new ones, and validate that inputs equal outputs plus fees.
2. **Account-based chain**: Implement a minimal account-based blockchain. Maintain a global state dictionary mapping addresses to balances. Apply transactions that debit the sender and credit the receiver.
3. **Determinism test**: Intentionally introduce non-determinism (e.g., use `time.time()` in a state transition). Show how two nodes processing the same block arrive at different states.

### Mini Build Task

**Implement both UTXO and Account models side by side.** Process the same set of logical transfers (Alice sends Bob 5 coins, Bob sends Charlie 3 coins, etc.) through both models. Compare: state representation size, transaction size, ease of implementing multisig, and parallel validation capability. Write a comparison report.

---

## Topic 2: Transactions and Blocks

### Explanation

**Transaction Structure**

A transaction is a signed instruction to transition the state. In Ethereum, a transaction (Type 2, EIP-1559) contains:

| Field                  | Description                                                    |
| ---------------------- | -------------------------------------------------------------- |
| `chainId`              | Network identifier (prevents replay across chains)             |
| `nonce`                | Sequential counter per sender account                          |
| `maxPriorityFeePerGas` | Tip to the validator (EIP-1559)                                |
| `maxFeePerGas`         | Maximum total fee per gas                                      |
| `gasLimit`             | Maximum gas units for execution                                |
| `to`                   | Recipient address (or empty for contract creation)             |
| `value`                | Amount of ETH to transfer (in wei)                             |
| `data`                 | Calldata (function selector + arguments, or contract bytecode) |
| `v, r, s`              | ECDSA signature components                                     |

The transaction is RLP-encoded, hashed with Keccak-256, and signed with the sender's private key. The sender's address is _recovered_ from the signature — it is not explicitly included in the transaction.

**Transaction Lifecycle**

1. **Creation**: User constructs and signs the transaction.
2. **Propagation**: Transaction is broadcast to the network via gossip. Nodes store it in their **mempool** (memory pool of pending transactions).
3. **Inclusion**: A block producer selects transactions from the mempool and includes them in a block.
4. **Execution**: When the block is processed, the transaction's state transition is applied.
5. **Finality**: After sufficient confirmations (PoW) or a finality vote (PoS), the transaction is considered irreversible.

**Block Structure**

A block consists of:

- **Header**: Metadata including parent hash, state root, transactions root, receipts root, timestamp, gas limit, gas used, and consensus-specific data (e.g., PoW nonce, PoS attestations).
- **Body**: The ordered list of transactions.

The block header is the most critical component for consensus — nodes can verify headers without processing the full block body.

**Fee Mechanisms**

- **Bitcoin**: Simple fee market. Transactions include a fee as the difference between total inputs and total outputs. Miners prioritize transactions by fee rate (satoshis per byte).
- **Ethereum (EIP-1559)**: Two-component fee. A **base fee** (protocol-set, burned) adjusts dynamically based on block utilization. A **priority fee** (user-set) goes to the validator. This creates more predictable pricing and reduces overpayment.

### Why It Matters

Understanding transaction structure enables you to construct, sign, and decode transactions at the byte level. Understanding the block lifecycle reveals the incentive dynamics that drive block production, transaction ordering, and MEV extraction. Understanding fee mechanisms is essential for building applications where transaction cost predictability matters.

### Real-World Context

- Ethereum's EIP-1559 (London upgrade, August 2021) fundamentally changed the fee market. Prior to EIP-1559, users overpaid during congestion (bidding wars). After EIP-1559, base fee burns removed ~3.5M ETH in its first two years.
- Bitcoin's SegWit upgrade (2017) separated signature data (witness) from the transaction body, effectively increasing block capacity and enabling the Lightning Network.
- MEV bots monitor the mempool for profitable reordering opportunities (front-running, sandwich attacks, arbitrage), making the mempool a competitive arena.

### Tools / Technologies

| Tool                    | Purpose                                         |
| ----------------------- | ----------------------------------------------- |
| `ethers.js` / `web3.js` | Construct, sign, and send Ethereum transactions |
| `cast` (Foundry)        | CLI tool for decoding and sending transactions  |
| `bitcoin-cli`           | Bitcoin Core RPC for transaction operations     |
| Etherscan / Blockchair  | Block explorers for transaction inspection      |
| Flashbots Protect       | Private transaction submission (MEV protection) |

### Deep Mastery Questions

1. Why is the sender address not explicitly included in an Ethereum transaction? How is it recovered, and what are the implications?
2. What happens if a user submits two Ethereum transactions with nonces 5 and 7, but not 6? What state do these transactions enter?
3. How does EIP-1559's base fee adjustment algorithm work? What is the target block utilization, and how does the base fee respond to deviations?
4. In Bitcoin, what is the "fee rate" and why is it measured in satoshis per virtual byte (sat/vB) rather than total fee?
5. What is transaction malleability, and how did SegWit fix it in Bitcoin?
6. How does a block producer (miner/validator) decide which transactions to include and in what order? What incentives drive this decision?
7. What is an "uncle block" in Ethereum (pre-merge), and why were uncle rewards paid?
8. Explain how a "stuck" Ethereum transaction (too low gas price) can be unstuck by sending a replacement transaction with the same nonce but higher fee.

### Hands-On Exercises

1. **Raw transaction construction**: Using `ethers.js`, construct an Ethereum transaction from scratch (set all fields manually), sign it, serialize it, and decode the raw bytes. Verify the signature and recover the sender address.
2. **Block anatomy**: Fetch a real Bitcoin block and a real Ethereum block using RPC calls. Parse the header fields and verify: (a) the parent hash matches the previous block, (b) the Merkle root matches the transactions.
3. **Fee analysis**: For the last 100 Ethereum blocks, plot the base fee over time. Identify periods of congestion and observe how the base fee responds.
4. **Mempool observation**: Connect to a Bitcoin or Ethereum node and observe the mempool. Track how long transactions wait before inclusion and correlate with fee level.

### Mini Build Task

**Build a transaction explorer.** Given a block number, fetch the block, decode all transactions, and display: sender, recipient, value, gas used, and effective fee. For each transaction, verify the signature and confirm the sender matches. Implement in JavaScript using `ethers.js`.

---

## Topic 3: Nodes and Networking

### Explanation

**Node Types**

- **Full Node**: Downloads and validates every block and transaction. Maintains the current state. Does not necessarily store all historical state.
  - Example: Geth in "full sync" mode, Bitcoin Core.
  - Trust model: Full nodes trust nothing — they verify everything from the genesis block.

- **Archive Node**: A full node that additionally stores all historical state at every block. Enables queries like "What was this account's balance at block 5,000,000?"
  - Storage requirement: ~13 TB for Ethereum (2024).
  - Used by: block explorers, analytics platforms, indexing services.

- **Light Node (Light Client)**: Downloads only block headers and uses Merkle proofs to verify specific data on demand.
  - Trust model: Trusts that the longest chain (by proof of work or stake weight) is valid, but verifies individual data items via proofs.
  - Examples: Ethereum light clients (Helios), Bitcoin SPV wallets.

- **Validator/Miner Node**: A full node that additionally participates in consensus — proposing blocks (PoW mining or PoS proposing) and voting (PoS attesting).

**Mempool Dynamics**

The mempool is each node's local pool of unconfirmed transactions. Key dynamics:

- **Admission**: Nodes validate transactions before admitting them (valid signature, sufficient balance, acceptable gas price).
- **Eviction**: When the mempool is full, lowest-fee transactions are evicted.
- **Heterogeneity**: Different nodes may have different mempools (different admission policies, propagation delays). There is no single global mempool.
- **Privacy**: Transactions in the mempool are visible to all nodes. This enables front-running and MEV extraction.

**Sync Strategies**

- **Full Sync**: Download and execute every block from genesis. Slowest, most thorough. Validates every historical state transition.
- **Fast Sync (Ethereum)**: Download all block headers, verify PoW/PoS, then download the state trie at a recent block. Validates headers but not historical state transitions.
- **Snap Sync (Ethereum)**: Downloads the state trie in a flat format (not the full trie structure), then reconstructs. Fastest sync method. Default in Geth since v1.10.
- **Headers-First (Bitcoin)**: Download all headers first, then download blocks in parallel from multiple peers.

**Peer Management**

Nodes maintain peer connections with scoring systems:

- Peers that provide invalid data are penalized or disconnected.
- Peers that provide timely, useful data are scored higher.
- Ethereum's devp2p maintains a peer reputation system.

### Why It Matters

Node architecture determines who can participate in the network, what they can verify, and how much resource commitment is required. The shift from PoW to PoS changed validator economics but didn't change the fundamental role of full nodes. Light clients are critical for mobile wallets and embedded devices but come with weaker trust assumptions. Understanding sync strategies matters when running infrastructure.

### Real-World Context

- Running an Ethereum validator requires a full node (execution client + consensus client). As of 2024, this requires ~2 TB SSD, 16 GB RAM, and a stable internet connection.
- Infura, Alchemy, and QuickNode operate centralized RPC services that provide full/archive node access. Most dApps rely on these — creating a tension between decentralization ideals and practical infrastructure.
- The "node count" of a network (often cited as a decentralization metric) is less meaningful than the _diversity_ of node operators (geographic, hosting provider, client implementation).

### Tools / Technologies

| Tool                                 | Purpose                             |
| ------------------------------------ | ----------------------------------- |
| Geth                                 | Ethereum execution client (Go)      |
| Nethermind                           | Ethereum execution client (C#)      |
| Prysm / Lighthouse / Teku / Lodestar | Ethereum consensus clients          |
| Bitcoin Core                         | Bitcoin full node                   |
| `nodewatch.io`                       | Ethereum node distribution tracker  |
| Ethernodes                           | Ethereum client diversity dashboard |

### Deep Mastery Questions

1. What is the minimum hardware to run an Ethereum full node? An archive node? Why is the difference important?
2. Why does Ethereum require _two_ clients (execution + consensus) post-Merge? What does each one do?
3. If 70% of Ethereum nodes run Geth (one client implementation), what risk does this create? What has the community done to address it?
4. Explain snap sync's trust model. What assumptions does a snap-synced node make compared to a full-synced node?
5. How does Bitcoin's headers-first sync mitigate the "long-range attack" that a full genesis sync might face?
6. What is the significance of the mempool being _local_ to each node rather than a global shared structure? How does this affect MEV?
7. If you run a light client, what attack can a dishonest majority execute against you that a full node would detect?

### Hands-On Exercises

1. **Run a node**: Set up and sync a Geth node on Ethereum Sepolia testnet. Monitor sync progress, disk usage, and peer count.
2. **RPC exploration**: Use `curl` to make JSON-RPC calls to your node. Fetch the latest block, get an account balance, and trace a transaction.
3. **Mempool analysis**: Subscribe to pending transactions on a testnet node. Log transaction hashes, gas prices, and time-to-inclusion.
4. **Client diversity**: Survey the current client distribution of Ethereum mainnet using ethernodes.org. Analyze the risk implications.

### Mini Build Task

**Build a block header sync tool.** Connect to an Ethereum node via RPC and download the last 1,000 block headers. Verify the chain: each header's parentHash must match the hash of the previous header. Report any gaps or inconsistencies. Measure download time and compare header-only sync vs. full block download.

---

## Topic 4: Proof of Work

### Explanation

Proof of Work (PoW) is a consensus mechanism where block producers (miners) compete to find a value (`nonce`) such that:

```text
H(block_header || nonce) < target
```

where `H` is a cryptographic hash function and `target` is a number that determines the difficulty. A smaller target means more hashes must be tried on average — higher difficulty.

**Mining Process**

1. Miner constructs a candidate block (selects transactions from mempool, builds Merkle root).
2. Miner iterates over nonce values, computing `H(header || nonce)` for each.
3. If the hash is below the target, the block is valid. The miner broadcasts it.
4. Other nodes verify: (a) the hash is below target, (b) all transactions are valid, (c) the block follows all protocol rules.

**Difficulty Adjustment**

- **Bitcoin**: Difficulty adjusts every 2,016 blocks (~2 weeks) to maintain a 10-minute average block time. If blocks came faster, difficulty increases; if slower, it decreases.
- **Ethereum (pre-Merge)**: Used a per-block difficulty adjustment (Ethash), including the "difficulty bomb" that exponentially increased difficulty to incentivize the Merge.

**Hash Rate and Security**

The network's total hash rate determines the cost of a 51% attack. An attacker needs >50% of the hash rate to (probabilistically) produce a longer chain and rewrite history. The cost of this attack is the capital and electricity cost of the necessary mining hardware.

**Mining Pools**

Individual miners have high variance in rewards (may mine for months without finding a block). Pools aggregate hash power and distribute rewards proportionally, reducing variance. Pool protocols: Stratum v1, Stratum v2.

**ASIC Resistance**

Some PoW algorithms (Ethash, RandomX) attempt to resist Application-Specific Integrated Circuits (ASICs) by requiring large memory access patterns that are inefficient on specialized hardware. This is a contentious design choice — ASIC resistance promotes GPU mining (wider participation) but may reduce total hash rate (weaker security).

### Why It Matters

PoW was the original blockchain consensus mechanism and remains the security model for Bitcoin (the largest blockchain by market cap). Understanding PoW is essential for reasoning about mining economics, 51% attack costs, and why the industry moved toward PoS (energy efficiency, different security model).

### Real-World Context

- Bitcoin's hash rate reached ~600 EH/s (exahashes per second) in 2024, making a 51% attack astronomically expensive (~$10B+ in hardware alone).
- Ethereum transitioned from PoW (Ethash) to PoS in "The Merge" (September 2022), reducing energy consumption by ~99.95%.
- Bitcoin Cash and Bitcoin SV, with much lower hash rates, have experienced 51% attacks and deep reorganizations.
- The Bitcoin halving (every 210,000 blocks, ~4 years) cuts the block reward in half, shifting miner revenue toward transaction fees.

### Tools / Technologies

| Tool                      | Purpose                                                     |
| ------------------------- | ----------------------------------------------------------- |
| `cpuminer`                | Simple CPU miner for educational use                        |
| `hashcat`                 | High-speed hashing (demonstrates SHA-256 throughput)        |
| Bitcoin mining calculator | Estimate profitability given hash rate and electricity cost |
| Stratum V2                | Modern mining pool protocol                                 |

### Deep Mastery Questions

1. Why is finding a valid PoW nonce computationally hard but verifying it easy? What cryptographic property provides this asymmetry?
2. What is the exact mechanism of a 51% attack? Can the attacker steal funds from arbitrary accounts? What can and can't they do?
3. Why does Bitcoin use a 10-minute block time? What would happen if it were 10 seconds? What if it were 1 hour?
4. How does the difficulty adjustment algorithm prevent an attacker from mining many blocks quickly by temporarily adding massive hash power?
5. Why is "selfish mining" a rational strategy when a miner controls ~33% of the hash rate? How does it break the assumption that mining on the longest chain is always optimal?
6. What is the long-term security concern for Bitcoin as block rewards approach zero? How must the fee market evolve?
7. Why does Monero use the RandomX algorithm? What specific hardware property does it target?

### Hands-On Exercises

1. **Manual mining**: Implement a simplified PoW mining loop in Python. Given a block header, find a nonce such that `SHA256(header + nonce)` starts with a specified number of zero bits. Measure how nonce attempts scale with difficulty.
2. **Difficulty analysis**: For Bitcoin, fetch the difficulty adjustments over the last year. Plot difficulty vs. hash rate and verify their correlation.
3. **51% attack simulation**: Simulate two miners (honest majority 60%, attacker 40%). Run 10,000 rounds. Count how often the attacker produces a longer chain over a 6-block window. Repeat with 50%/50% and 40%/60%.
4. **Mining economics**: Calculate the break-even electricity cost for mining 1 BTC given current difficulty, block reward, and hash rate of a modern ASIC miner.

### Mini Build Task

**Build a complete PoW blockchain.** Extend your Stage 1 capstone (data layer) with:

- A mining loop that finds valid nonces.
- Dynamic difficulty adjustment (target a 5-second block time with 10-block adjustment windows).
- Reward transactions (coinbase transactions) that credit the miner.
- Chain selection: always follow the longest valid chain.
- Run it with two simulated miners and observe forks, difficulty changes, and reward distribution.

---

## Topic 5: Proof of Stake

### Explanation

Proof of Stake (PoS) replaces computational work with economic stake as the Sybil resistance mechanism. Instead of mining, validators lock (stake) cryptocurrency as collateral. The right to propose and validate blocks is allocated based on stake weight.

**Validator Selection**

Different systems use different selection mechanisms:

- **Random selection weighted by stake**: Used by Ethereum. A pseudo-random function selects the block proposer for each slot, weighted by stake.
- **Round-robin with stake weighting**: Used by Tendermint/Cosmos. Validators take turns proposing, with frequency proportional to stake.
- **VRF-based selection**: Algorand uses Verifiable Random Functions for private, non-interactive leader election.

**Ethereum's PoS (Gasper = Casper FFG + LMD-GHOST)**

Ethereum's PoS combines two mechanisms:

1. **LMD-GHOST** (Latest Message Driven Greediest Heaviest Observed SubTree): Fork choice rule. Instead of following the longest chain, nodes follow the fork with the most recent attestation weight. This provides fast chain growth.

2. **Casper FFG** (Friendly Finality Gadget): Finality mechanism. Every 32 slots (~6.4 minutes, one "epoch"), validators vote on checkpoint blocks. A block becomes "justified" when 2/3+ of validators vote for it. When two consecutive checkpoints are justified, the first becomes "finalized" — irreversible unless 1/3+ of the total stake is slashed.

**Slashing**

Validators who violate protocol rules lose a portion of their stake:

- **Double voting**: Signing two different blocks for the same slot.
- **Surround voting**: Casting an attestation that "surrounds" a previously finalized checkpoint (attempting to revert finality).
- Slashing penalty is proportional to the number of validators slashed simultaneously (correlation penalty), discouraging coordinated attacks.

**Delegation and Liquid Staking**

- **Delegated PoS (DPoS)**: Token holders delegate their stake to validators (e.g., Cosmos, Polkadot). Reduces active validator set but enables broader participation.
- **Liquid Staking**: Protocols like Lido issue derivative tokens (stETH) representing staked ETH. This allows stakers to maintain liquidity but introduces systemic risk (concentration of staked assets in a single protocol).

### Why It Matters

PoS is now the dominant consensus mechanism for smart contract platforms (Ethereum, Cosmos, Solana, Polkadot, Cardano). Understanding its security model — what attacks it defends against, what economic assumptions it requires, and where it differs from PoW — is essential for any blockchain engineer.

### Real-World Context

- Ethereum's Merge (September 2022) was the largest PoS transition in blockchain history, involving ~$40B of staked ETH and 400,000+ validators.
- Lido's dominance (~30% of all staked ETH) raised centralization concerns and prompted governance discussions about concentration limits.
- Solana's PoS uses a unique combination of Proof of History (PoH) for ordering and Tower BFT for consensus, enabling ~400ms block times.
- Cosmos chains (running Tendermint BFT) provide instant finality but halt if >1/3 of validators go offline.

### Tools / Technologies

| Tool                       | Purpose                                  |
| -------------------------- | ---------------------------------------- |
| Ethereum Staking Launchpad | Guide for becoming an Ethereum validator |
| Lido                       | Liquid staking protocol                  |
| `ethdo`                    | CLI for Ethereum staking operations      |
| Beaconcha.in               | Ethereum beacon chain explorer           |
| Rated.network              | Validator performance analytics          |

### Deep Mastery Questions

1. Why does PoS require `3f + 1` stake (at least 2/3 honest by stake) for safety, matching BFT bounds?
2. What is the "nothing at stake" problem, and how does Ethereum's slashing mechanism address it?
3. Explain the difference between "justified" and "finalized" in Casper FFG. Why are two rounds of justification needed for finality?
4. What is a "long-range attack" in PoS, and why doesn't it exist in PoW? How does Ethereum defend against it?
5. If 1/3+ of Ethereum validators go offline simultaneously, what happens to the chain? Does it halt? Does it continue with weaker finality?
6. Why is correlation penalty important? What behavior does it discourage that a fixed penalty would not?
7. What systemic risk does liquid staking (e.g., Lido holding 30% of staked ETH) create for Ethereum? How is this different from mining pool concentration in PoW?
8. Compare the finality guarantees of Ethereum (Gasper), Cosmos (Tendermint), and Bitcoin (Nakamoto consensus). Explain the trade-offs.

### Hands-On Exercises

1. **Validator economics**: Calculate the expected annual return for an Ethereum validator with 32 ETH. Account for: attestation rewards, proposal rewards, sync committee duties, and the impact of correlation penalties.
2. **Casper FFG simulation**: Implement a simplified Casper FFG finality gadget with 10 validators. Simulate voting across 5 epochs and show how justification and finalization progress. Introduce 3 Byzantine validators and show that finality is maintained.
3. **Fork choice**: Implement LMD-GHOST fork choice for a simple tree of blocks. Given attestation weights, determine the canonical head.
4. **Slashing trace**: Examine a real slashing event on the Ethereum beacon chain (use Beaconcha.in). Identify: what offense was committed, how much stake was slashed, and the correlation factor.

### Mini Build Task

**Build a simplified PoS consensus simulator.** 16 validators with varying stake weights. For each slot: (a) select a proposer weighted by stake, (b) have all validators attest to the proposed block, (c) implement a simple fork choice (heaviest chain). Run for 100 epochs. Track: proposer distribution, attestation agreement rates, and simulated rewards. Then introduce 4 Byzantine validators that double-attest—show that the slashing mechanism detects them.

---

## Topic 6: Forks and Chain Reorganization

### Explanation

**Forks**

A fork occurs when two valid blocks reference the same parent — the chain "splits" into two competing branches.

- **Natural forks**: In PoW, two miners may find valid blocks nearly simultaneously. Nodes that receive different blocks first follow different chains. This resolves when the next block extends one of the forks (the other becomes an "orphan" or "stale" block).

- **Soft forks**: A backward-compatible protocol change. Old nodes still accept new blocks (they don't enforce the new rules). New nodes reject blocks that violate the new rules. Soft forks tighten the rules.
  - Example: Bitcoin's SegWit (BIP 141) — old nodes see SegWit transactions as valid (anyone-can-spend), new nodes enforce the witness rules.

- **Hard forks**: A backward-incompatible protocol change. Old nodes reject new blocks that follow the new rules. The network splits unless all nodes upgrade.
  - Example: Ethereum's DAO fork (2016) — the chain split into Ethereum (forked to return DAO funds) and Ethereum Classic (unforked).

**Chain Reorganization**

A reorganization (reorg) occurs when a node discovers a heavier/longer chain that diverges from its current chain at some past block. The node abandons its current chain tip and switches to the new fork. All transactions that were in the abandoned blocks but not in the new fork are returned to the mempool.

Reorg depth matters:

- 1-block reorgs are common and benign in PoW (natural forks).
- Deep reorgs (e.g., 6+ blocks in Bitcoin) are extremely rare under honest majority assumptions and indicate either an attack or a severe network partition.
- In Ethereum PoS, finalized blocks cannot be reorged (without 1/3+ stake being slashed).

**Finality Models**

- **Probabilistic finality (Bitcoin/PoW)**: A transaction becomes "more final" with each confirmation. 6 confirmations (~60 minutes) is the standard threshold, but there is always a nonzero probability of reorg.
- **Deterministic finality (Ethereum PoS / Tendermint)**: Once a block is finalized, it cannot be reverted without provably malicious behavior by 1/3+ of the validator set. Finality is binary, not probabilistic.
- **Economic finality**: The cost of reverting a finalized block is quantifiable (the slashed stake). This gives a dollar value to finality guarantees.

### Why It Matters

Forks and reorgs are not theoretical concerns — they happen regularly and have real consequences. Applications must handle reorgs gracefully: a payment confirmed in block N may be reversed if a reorg replaces that block. Exchange deposit policies (requiring N confirmations) are directly informed by reorg risk. Protocol upgrades (soft and hard forks) are the mechanism for blockchain evolution.

### Real-World Context

- Bitcoin's longest unintentional reorganization was a 1-block reorg (common, happens several times per year).
- The Ethereum Classic 51% attacks (2020) caused deep reorganizations (38 blocks, then 78 blocks), enabling double-spends worth millions.
- Ethereum's transition from PoW to PoS was a hard fork (the Merge), but uniquely, all nodes upgraded simultaneously — there was no persistent chain split.
- Polygon experienced a 157-block reorganization in 2023, highlighting the risks of lower-security PoS chains.

### Tools / Technologies

| Tool                           | Purpose                          |
| ------------------------------ | -------------------------------- |
| Fork Monitor (bitcoin.ninja)   | Real-time Bitcoin fork detection |
| Reorg tracker (blockchair.com) | Multi-chain reorg monitoring     |
| `eth_getBlockByNumber`         | RPC call to track block changes  |
| Bitcoin Core `getchaintips`    | Detect alternative chain tips    |

### Deep Mastery Questions

1. Why is 6 confirmations the standard for Bitcoin transaction finality? What security assumption does this encode?
2. In Ethereum PoS, what must happen for a finalized block to be reverted? Quantify the economic cost.
3. What is the difference between a soft fork and a hard fork in terms of backward compatibility? Give an example of each.
4. How should a dApp handle a chain reorganization? What data structures and event handling patterns are needed?
5. What is a "time bandit" attack in PoW, and how does it relate to MEV and reorgs?
6. Why are deep reorgs more dangerous on lower-hash-rate chains? What is the relationship between hash rate and reorg cost?
7. In a hard fork where the community is split (e.g., DAO fork), what determines which chain "wins"?

### Hands-On Exercises

1. **Reorg simulation**: Extend your PoW blockchain from Topic 4 to support forks. Create a scenario where two miners find blocks simultaneously. Then one miner extends their fork — show the other fork being abandoned and transactions returning to the mempool.
2. **Confirmation analysis**: For a Bitcoin block explorer, trace 100 transactions and record the number of confirmations at which exchanges credited the deposit. Analyze the distribution.
3. **Soft fork exercise**: In your simulated blockchain, implement a soft fork that adds a new validation rule (e.g., maximum transaction value). Show that old nodes still accept the new blocks but new nodes reject blocks violating the rule.

### Mini Build Task

**Build a fork visualizer.** Given a blockchain with forks (use your simulator from Topic 4 or fetch real data), render a tree visualization showing the main chain and orphaned forks. For each fork, show: depth, number of transactions affected, and time to resolution. Highlight any transactions that appeared in the abandoned fork but not in the canonical chain.

---

## Topic 7: Token Standards and Blockchain Data Models

### Explanation

**Native Tokens vs. Smart Contract Tokens**

Every blockchain has a native token (BTC, ETH, SOL) used for transaction fees and consensus incentives. This token exists at the protocol level — its transfer is a primitive operation of the state transition function.

Smart contract tokens are implemented as state within a smart contract. The "token" is just a mapping `(address → balance)` maintained by contract code. Token transfers are contract calls, not protocol-level operations.

**Ethereum Token Standards**

- **ERC-20 (Fungible Tokens)**: A standard interface for fungible tokens. Functions: `transfer`, `transferFrom`, `approve`, `balanceOf`, `totalSupply`, `allowance`. Every ERC-20 token has identical behavior from the protocol's perspective — the VM doesn't know or care about the "meaning" of the token.

- **ERC-721 (Non-Fungible Tokens)**: Each token has a unique `tokenId`. Functions: `ownerOf`, `transferFrom`, `approve`, `safeTransferFrom`. Ownership is a mapping `(tokenId → address)`.

- **ERC-1155 (Multi-Token Standard)**: Supports both fungible and non-fungible tokens in a single contract. Enables batch transfers. More gas-efficient for games and applications with many token types.

**UTXO vs. Account Model — Deep Comparison**

| Dimension               | UTXO (Bitcoin)                    | Account (Ethereum)               |
| ----------------------- | --------------------------------- | -------------------------------- |
| State representation    | Set of unspent outputs            | Map of addresses to accounts     |
| Transaction model       | Consumes and creates UTXOs        | Debits sender, credits receiver  |
| Parallelism             | High (UTXOs are independent)      | Lower (shared state per account) |
| Privacy                 | Better (new UTXO per transaction) | Worse (persistent address)       |
| Smart contracts         | Limited (Script)                  | Rich (EVM)                       |
| State growth            | Bounded by UTXO set               | Unbounded (accounts persist)     |
| Double-spend prevention | UTXO can only be spent once       | Nonce ordering per account       |

**State Growth and Pruning**

As the blockchain processes more transactions, the state grows. In Ethereum, every new account and every new storage slot increases the state trie. This creates a "state bloat" problem:

- The Ethereum state is ~100 GB (as of 2024).
- Full state access requires keeping the entire trie in memory or on fast storage.
- **Pruning** removes old state that is no longer needed for current validation. Archive nodes retain everything; pruned nodes keep only the recent state.
- **State expiry** (proposed, not yet implemented in Ethereum) would require inactive accounts to be "refreshed" periodically or they become inaccessible (but not deleted).

### Why It Matters

Token standards define the interface layer for the entire DeFi and NFT ecosystem. Understanding why ERC-20's `approve/transferFrom` pattern creates a specific security surface (infinite approvals, front-running) enables better dApp design. Understanding state growth is critical for long-term protocol sustainability — a blockchain that grows without bound eventually becomes impractical to operate.

### Real-World Context

- The ERC-20 standard enabled the ICO boom of 2017 and remains the foundation for DeFi tokens (USDC, DAI, UNI, LINK).
- ERC-721 enabled the NFT market (CryptoPunks, Bored Apes, etc.). The standard's `safeTransferFrom` function checks whether the recipient can handle NFTs, preventing accidental locks.
- Ethereum's state growth has been a persistent concern. The "Berlin" and "Shanghai" upgrades adjusted gas costs for state operations to slow growth.
- Bitcoin's UTXO set (~80M UTXOs, ~5 GB as of 2024) is much smaller than Ethereum's state, partly because Bitcoin lacks general smart contracts.

### Tools / Technologies

| Tool                           | Purpose                                           |
| ------------------------------ | ------------------------------------------------- |
| OpenZeppelin Contracts         | Audited ERC-20, ERC-721, ERC-1155 implementations |
| `erc20-watcher`                | Monitor ERC-20 token events                       |
| Dune Analytics                 | On-chain analytics and token data queries         |
| Ethereum State Size dashboards | Monitor state growth                              |

### Deep Mastery Questions

1. Why does ERC-20 use the `approve/transferFrom` pattern instead of a simpler `transferTo` function? What problem does it solve, and what new problem does it create?
2. How does an ERC-721 contract differentiate between two distinct NFTs? What is stored on-chain vs. off-chain?
3. Why is ERC-1155 more gas-efficient for games with many token types than deploying separate ERC-20 and ERC-721 contracts?
4. In the UTXO model, how would you implement a "token" (e.g., colored coins)? What limitations does this approach have compared to ERC-20?
5. What is the "state rent" or "state expiry" proposal for Ethereum? What problem does it solve, and why is it controversial?
6. Why is Ethereum's state larger than Bitcoin's UTXO set, even though Bitcoin has been running longer? What architectural difference causes this?

### Hands-On Exercises

1. **ERC-20 dissection**: Read the OpenZeppelin ERC-20 implementation line by line. Trace a `transfer` call and a `approve` + `transferFrom` call through the code. Identify where balances change and where events are emitted.
2. **Token analytics**: Using Dune Analytics (or direct RPC), find: the total supply of USDC, the number of unique holders, and the top 10 holders by balance.
3. **UTXO set analysis**: Using Bitcoin's RPC, query the UTXO set size (`gettxoutsetinfo`). Compare the UTXO set memory footprint to Ethereum's state.

### Mini Build Task

**Build a multi-token system.** Implement a simplified token registry (in Python, not Solidity—this is about the data model, not the smart contract platform) that supports: (a) creating new fungible token types, (b) minting tokens, (c) transferring tokens between addresses, (d) querying balances. Then extend it to support non-fungible tokens with unique IDs. Compare the data structures required for each type.

---

## Mini Projects

### Project 1: Full PoW Blockchain with Networking

Extend your PoW blockchain to include basic P2P networking. Two nodes communicate over TCP. Node A mines a block and gossips it to Node B. Node B validates and adds it. Implement basic fork resolution (longest chain wins).

### Project 2: Ethereum Block Analyzer

Build a tool that connects to an Ethereum RPC endpoint, fetches a range of blocks, and produces a report: average gas used, transaction count distribution, fee distribution (base fee vs. priority fee), and ERC-20 transfer events decoded from logs.

### Project 3: Consensus Mechanism Comparison Dashboard

Build a dashboard (web or CLI) that compares PoW and PoS across dimensions: energy consumption (estimated), finality time, throughput (TPS), hardware requirements, and minimum stake/investment to attack. Use real data from Bitcoin and Ethereum.

### Project 4: Transaction Mempool Simulator

Simulate a mempool with 1,000 pending transactions of varying gas prices. Implement a block producer that greedily fills a block (30M gas limit). Compare strategies: highest gas price first, highest total fee first, and include MEV bundles. Measure producer revenue under each strategy.

---

## Capstone Project: Complete Blockchain Node (Simplified)

Build a **simplified but fully functional blockchain node** that integrates everything from this stage:

1. **State machine**: Account-based model with balances and nonces.
2. **Transactions**: Signed, with nonce ordering and balance validation.
3. **Blocks**: Contain transaction lists, Merkle roots, and parent hashes.
4. **Mining**: PoW with dynamic difficulty targeting 5-second block times.
5. **Networking**: Two or more nodes communicate over TCP, gossiping blocks and transactions.
6. **Consensus**: Longest-chain fork choice. Handle natural forks and reorgs.
7. **Mempool**: Accept and validate incoming transactions. Evict low-fee transactions when full.
8. **API**: Simple RPC interface for querying balances, submitting transactions, and getting block info.

Deliverable: Two nodes running on different ports, independently mining, gossiping blocks, and converging on the same chain. A client can submit a transaction to either node and see it reflected in both.

---

## Common Mistakes & Misconceptions

| Mistake                                      | Reality                                                                                                                                                                     |
| -------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "Transactions are instant on the blockchain" | Transactions are pending until included in a block. Even after inclusion, they are not final until sufficient confirmations (PoW) or finalization (PoS).                    |
| "Miners/validators can steal your funds"     | Miners can reorder, delay, or censor transactions. They cannot forge signatures or spend funds they don't control.                                                          |
| "PoS is less secure than PoW"                | Different security model, not weaker. PoS has economic finality (slashing); PoW has thermodynamic security (energy cost).                                                   |
| "More TPS = better blockchain"               | TPS is meaningless without context: what is the decentralization? What is the finality model? What are the hardware requirements to run a node?                             |
| "The mempool is a single place"              | Every node has its own mempool. There is no canonical global mempool.                                                                                                       |
| "Hard forks always create two chains"        | Hard forks only create persistent splits when a significant minority refuses to upgrade (e.g., ETH/ETC). Most hard forks are coordinated upgrades where the old chain dies. |

---

## Security Considerations

At this stage, internalize:

- **Confirmation requirements**: Never treat an unconfirmed or recently confirmed transaction as final. The number of required confirmations should be proportional to the transaction value and the chain's security.
- **Reorg awareness**: Any application that reads blockchain state must handle the possibility that recent blocks may be reverted.
- **Eclipse attack awareness**: A node that is isolated from the honest network (all connections controlled by an attacker) can be fed a false chain. Defense: peer diversity, hard-coded bootstrap nodes, and multiple data sources.
- **51% attack economics**: Always reason about the cost of attacking a chain. For low-hash-rate PoW chains, this cost may be surprisingly low (attackable with rented hash power from NiceHash).

---

## Readings & Resources

**Books**

- _Mastering Bitcoin_ by Andreas Antonopoulos — Chapters 5–10 (transactions, blocks, mining, network)
- _Mastering Ethereum_ by Andreas Antonopoulos & Gavin Wood — Chapters 1–7
- _Bitcoin and Cryptocurrency Technologies_ by Narayanan et al. — Comprehensive academic textbook

**Papers**

- Nakamoto — _"Bitcoin: A Peer-to-Peer Electronic Cash System"_ (2008) — Read again now, with full understanding.
- Eyal & Sirer — _"Majority is Not Enough: Bitcoin Mining is Vulnerable"_ (2013) — Selfish mining attack.
- Buterin & Griffith — _"Casper the Friendly Finality Gadget"_ (2017)
- Sompolinsky & Zohar — _"PHANTOM/GHOSTDAG"_ (2018) — DAG-based consensus.
- EIP-1559 specification — ethereum.github.io/EIPs/EIPS/eip-1559

**Documentation**

- Ethereum Yellow Paper (Gavin Wood) — Formal specification of Ethereum.
- Bitcoin Developer Reference — https://developer.bitcoin.org/reference/
- Ethereum Consensus Specification — https://github.com/ethereum/consensus-specs

**Technical Blogs**

- Vitalik Buterin's blog: https://vitalik.eth.limo/
- BitMEX Research (mining and PoW analysis)
- Flashbots blog (MEV and transaction ordering)
- Paradigm blog (protocol research)

---

<div style="display:flex;align-items:center;justify-content:space-between;padding:14px 0;margin-top:48px;">
  <a href="./01_Foundations.md" style="display:flex;align-items:center;gap:8px;text-decoration:none;color:#8b949e;font-size:0.875rem;padding:6px 10px;border-radius:6px;">
    <svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor"><path d="M9.78 12.78a.75.75 0 0 1-1.06 0L4.47 8.53a.75.75 0 0 1 0-1.06l4.25-4.25a.75.75 0 0 1 1.06 1.06L6.06 8l3.72 3.72a.75.75 0 0 1 0 1.06z"/></svg>
    <span style="display:flex;flex-direction:column;line-height:1.3;">
      <span style="font-size:0.7rem;opacity:0.55;text-transform:uppercase;letter-spacing:0.06em;">Previous</span>
      <span style="font-weight:500;">01 · Foundations</span>
    </span>
  </a>
  <span style="font-size:0.7rem;color:#8b949e;opacity:0.5;letter-spacing:0.08em;text-transform:uppercase;">Stage 2 of 6 · Blockchain Core</span>
  <a href="./03_Smart_Contracts_and_EVM.md" style="display:flex;align-items:center;gap:8px;text-decoration:none;color:#8b949e;font-size:0.875rem;padding:6px 10px;border-radius:6px;">
    <span style="display:flex;flex-direction:column;line-height:1.3;text-align:right;">
      <span style="font-size:0.7rem;opacity:0.55;text-transform:uppercase;letter-spacing:0.06em;">Next</span>
      <span style="font-weight:500;">03 · Smart Contracts & EVM</span>
    </span>
    <svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor"><path d="M6.22 3.22a.75.75 0 0 1 1.06 0l4.25 4.25a.75.75 0 0 1 0 1.06l-4.25 4.25a.75.75 0 0 1-1.06-1.06L9.94 8 6.22 4.28a.75.75 0 0 1 0-1.06z"/></svg>
  </a>
</div>
