# Stage 3 — Smart Contracts and the EVM

`Stage 3 of 6` · Estimated 6–8 weeks

## Stage Overview

### What This Stage Builds

This stage transforms you from a blockchain observer into a blockchain programmer. You will understand the Ethereum Virtual Machine at the instruction level, master Solidity as an engineering language (not a scripting tool), design and test production-grade smart contracts, and develop the security mindset essential for writing code that holds financial value. By the end, you will be capable of reading and auditing arbitrary Solidity code, reasoning about gas costs, and identifying vulnerabilities before they are exploited.

### Why This Stage Is Critical

Smart contracts are the programmable layer of blockchain infrastructure. They govern trillions of dollars of assets across DeFi, NFTs, DAOs, and bridges. A bug in a smart contract is permanent and often exploitable — there are no patches once deployed to the canonical chain. This is the stage where engineering discipline becomes non-negotiable.

### Competence at the End of This Stage

You can write, test, and deploy production-quality Solidity contracts. You understand what the EVM executes at the opcode level. You can estimate gas costs analytically. You can identify all major vulnerability classes in a given contract. You can write a full test suite with unit tests, fuzz tests, and invariant tests. You can reason about contract upgradeability and access control.

---

## Conceptual Map

```text
EVM Architecture
├── Stack, memory, storage, calldata
├── Opcodes and execution
├── Contract creation (initcode vs runtime code)
└── The call stack (CALL, DELEGATECALL, STATICCALL, CREATE2)
        │
        ▼
Gas Model
├── Opcode costs
├── Storage costs (cold vs warm)
├── Refunds and EIP-3529
├── Gas optimization patterns
└── Gas griefing attacks
        │
        ▼
Solidity Engineering
├── Types and encoding (ABI)
├── Control flow and functions
├── Contract structure and inheritance
├── Modifiers and events
├── Interfaces and abstract contracts
├── Libraries (internal vs external)
├── Error handling (require, revert, custom errors)
└── Assembly (Yul/inline assembly)
        │
        ▼
Contract Architecture Patterns
├── Ownable, Access Control
├── Proxy patterns (Transparent, UUPS, Beacon)
├── Factory pattern
├── Pull-over-push payments
└── Checks-Effects-Interactions
        │
        ▼
Testing and Debugging
├── Foundry (forge test, fuzz, invariant)
├── Hardhat
├── Unit testing patterns
├── Fuzz testing
├── Invariant (property-based) testing
└── On-chain debugging with traces
        │
        ▼
Security Vulnerabilities
├── Reentrancy (classic, cross-function, cross-contract, read-only)
├── Integer overflow/underflow
├── Access control failures
├── Front-running and MEV
├── Oracle manipulation
├── Flash loan attacks
├── Delegatecall and storage collisions
├── Signature replay
├── Denial of service
└── Phishing (tx.origin)
```

---

## Topic 1: EVM Architecture

### Explanation

The Ethereum Virtual Machine is a quasi-Turing-complete, stack-based virtual machine that executes bytecode. "Quasi" because execution is bounded by gas — preventing infinite loops.

**Data Locations**

The EVM has four distinct data locations, each with different cost and persistence characteristics:

| Location     | Persistence                   | Access                  | Cost                                                                          |
| ------------ | ----------------------------- | ----------------------- | ----------------------------------------------------------------------------- |
| **Storage**  | Permanent (on-chain)          | Keyed (32-byte slots)   | Highest (SLOAD: 2100 gas cold, 100 gas warm; SSTORE: 20,000 for zero→nonzero) |
| **Memory**   | Transaction-scoped            | Linear (byte-addressed) | Quadratic above 724 bytes                                                     |
| **Stack**    | Expression-scoped             | LIFO, max 1024 elements | Cheapest                                                                      |
| **Calldata** | Read-only, transaction-scoped | Byte-indexed            | Cheaper than memory (used for function arguments)                             |

**The Stack Machine**

The EVM executes instructions that push and pop 32-byte words onto a stack. Most opcodes consume their operands from the stack and push results. Example sequence:

```text
PUSH1 0x03   // Stack: [3]
PUSH1 0x05   // Stack: [5, 3]
ADD          // Stack: [8]
```

**Key Opcodes**

| Category    | Opcodes                                               | Description                                                                |
| ----------- | ----------------------------------------------------- | -------------------------------------------------------------------------- |
| Arithmetic  | ADD, SUB, MUL, DIV, MOD, EXP                          | 256-bit arithmetic. Overflow wraps silently (Solidity <0.8 had this risk). |
| Comparison  | LT, GT, EQ, ISZERO                                    | Push 1 (true) or 0 (false).                                                |
| Bitwise     | AND, OR, XOR, SHL, SHR                                | Bitwise ops.                                                               |
| Storage     | SLOAD, SSTORE                                         | Read/write persistent storage.                                             |
| Memory      | MLOAD, MSTORE, MSTORE8                                | Read/write memory.                                                         |
| Control     | JUMP, JUMPI, JUMPDEST                                 | Unconditional/conditional jumps (must land on JUMPDEST).                   |
| Call        | CALL, DELEGATECALL, STATICCALL, CALLCODE              | Inter-contract calls.                                                      |
| Create      | CREATE, CREATE2                                       | Deploy new contracts.                                                      |
| Context     | CALLER, CALLVALUE, CALLDATALOAD, TIMESTAMP, BLOCKHASH | Environmental information.                                                 |
| Log         | LOG0, LOG1, LOG2, LOG3, LOG4                          | Emit events.                                                               |
| Termination | RETURN, REVERT, SELFDESTRUCT, STOP                    | End execution.                                                             |

**Contract Lifecycle**

1. **Deployment**: A transaction with empty `to` field and `data` containing **initcode** is sent.
2. **Constructor execution**: EVM runs initcode as a regular transaction. Constructor arguments are ABI-encoded after the bytecode.
3. **Runtime code storage**: The constructor returns the **runtime bytecode** (via the RETURN opcode), which is stored at the contract address.
4. **Calls**: Future transactions to the contract address execute the runtime bytecode.

**CALL vs DELEGATECALL**

- `CALL(target, gas, value, args)`: Executes target's code in target's **context** (target's storage, `address(this)` is the target). Most common form of inter-contract call.
- `DELEGATECALL(target, gas, args)`: Executes target's code in the **caller's context** (caller's storage, `address(this)` is the caller, `msg.sender` is preserved). Used in proxy patterns. Dangerous if target address is not trusted.
- `STATICCALL(target, gas, args)`: Like CALL but disallows state modification (SSTORE, LOG, etc.). Used for read-only calls and Solidity's `view` functions.

**CREATE vs CREATE2**

- `CREATE`: Contract address = `keccak256(deployer_address, nonce)`.
- `CREATE2`: Contract address = `keccak256(0xff, deployer_address, salt, keccak256(initcode))`. **Deterministic** — address is known before deployment. Enables counterfactual instantiation (open channels, safe factory patterns).

### Why It Matters

Every Solidity construct compiles to EVM opcodes. Understanding the EVM means you can reason about gas costs precisely, understand why certain patterns are dangerous, and read decompiled bytecode. Proxy patterns (DELEGATECALL) power the upgradeable contract ecosystem — and power its most insidious bugs. CREATE2 enables counterfactual deployment patterns used by state channels and account abstraction.

### Real-World Context

- The Parity multisig hack (2017, $150M) resulted from an incorrect DELEGATECALL that allowed anyone to call the library's `initWallet` function and become owner.
- CREATE2 is used by Uniswap V2 to deploy pools with deterministic addresses (computed from token pairs), enabling off-chain address calculation.
- The Ethereum Yellow Paper formally defines EVM semantics. Client implementations (Geth, Nethermind, Besu) must all agree on the same execution results.
- EVM object format (EOF, EIP-3540+) is an ongoing Ethereum upgrade to add better structure to bytecode.

### Tools / Technologies

| Tool                       | Purpose                                                          |
| -------------------------- | ---------------------------------------------------------------- |
| `evm.codes`                | Interactive EVM opcode reference with gas costs and stack traces |
| `ethervm.io`               | Online decompiler for EVM bytecode                               |
| `foundry cast disassemble` | Disassemble bytecode to opcodes                                  |
| `hevm`                     | Haskell EVM implementation, used for symbolic execution          |
| `pyrometer`                | Rust-based Ethereum bytecode analysis                            |

### Deep Mastery Questions

1. Why does the EVM use 32-byte (256-bit) words as its native data type? What does this enable cryptographically?
2. Trace the execution of `uint256 x = 5 + 3;` through the EVM opcode level. What does the stack look like after each opcode?
3. Why is SLOAD so expensive (2100 gas cold)? What does "cold" vs "warm" mean in EIP-2929?
4. Explain exactly what happens when Contract A does `DELEGATECALL` to Contract B. Whose storage is modified? Who is `msg.sender` inside B's code?
5. Why can you send ETH to a contract via `SELFDESTRUCT` even if the contract has no receive/fallback function?
6. What is the difference between the `initcode` and the `runtime code`? What happens if the constructor reverts?
7. How does CREATE2 enable counterfactual instantiation? Give a concrete example (state channels).
8. Why are `TIMESTAMP` and `BLOCKHASH` considered unreliable sources of randomness?

### Hands-On Exercises

1. **Opcode tracing**: Write a minimal Solidity contract (just `return a + b`). Compile it with `solc --opcodes`. Read the opcode output and explain what each instruction does.
2. **DELEGATECALL experiment**: Write two contracts — a proxy and a logic contract. Call the logic contract's function via delegatecall from the proxy. Verify that the proxy's storage is modified, not the logic contract's.
3. **CREATE2 address prediction**: Given deployer address, salt, and initcode hash, compute the CREATE2 address off-chain using keccak256. Deploy and verify the address matches.
4. **Bytecode analysis**: Take any deployed ERC-20 contract from Etherscan. Fetch its deployed bytecode. Disassemble it with `cast disassemble`. Identify: the function dispatcher, the SLOAD/SSTORE calls.

### Mini Build Task

**Build an EVM Instruction Tracer.** Using `ethereumjs-vm` (JavaScript) or `py-evm` (Python), execute simple bytecode manually. For each opcode executed, log: current opcode, gas remaining, stack state, and memory state. This gives you a window into the exact execution of any Ethereum computation.

---

## Topic 2: Gas and the Execution Model

### Explanation

Gas is the metering system that prevents the halting problem — infinite loops — from being exploitable. Every EVM opcode has a defined gas cost. Transaction execution consumes gas; when gas is exhausted, the transaction reverts (all state changes rolled back) but the gas fee is still paid to the block producer.

**Gas Cost Structure**

- **Base fee**: 21,000 gas for any transaction.
- **Calldata**: 4 gas per zero byte, 16 gas per non-zero byte (post EIP-2028). This is why ABI encoding matters for gas.
- **Opcodes**: Each opcode has a fixed cost. Computationally intensive operations (EXP, SHA3, ECRECOVER) cost more.
- **Storage**: Most expensive operations. Writing to a previously-zero storage slot costs 20,000 gas. Writing to a non-zero slot costs 2,900 gas (warm) or 5,000 gas (cold after EIP-2929). Clearing a slot (setting nonzero to zero) previously gave a refund; refunds were reduced in EIP-3529 (London, 2021).

**Gas Optimization Patterns**

| Pattern                                                                          | Savings                                             |
| -------------------------------------------------------------------------------- | --------------------------------------------------- |
| Pack multiple `uint`s into one storage slot                                      | Up to 15,000 gas per eliminated SSTORE              |
| Use `unchecked` arithmetic in Solidity 0.8+ when overflow is provably impossible | ~200 gas per arithmetic op                          |
| Use `calldata` instead of `memory` for read-only function parameters             | 3–5x cheaper                                        |
| Cache storage reads in memory variables                                          | 2100 gas saved per additional read beyond the first |
| Use custom errors instead of `require` with strings                              | Variable savings (~50+ gas per revert)              |
| Short-circuit conditionals (cheapest check first)                                | Variable                                            |
| Use `immutable` instead of `constant` for non-compile-time values                | ~200 gas per read vs SLOAD                          |

**Gas Limit and the 63/64 Rule**

When Contract A calls Contract B, A can specify the gas passed. EIP-150 introduced the 63/64 rule: Ethereum withholds 1/64 of the remaining gas for the calling frame even if the caller specifies "pass all gas." This ensures the caller has enough gas to handle the return and any cleanup.

**Gas Griefing**

An attack where a malicious callee intentionally uses up all forwarded gas to prevent the caller from completing. Defense: use Pull-over-Push payment patterns and avoid arbitrary external calls in critical paths.

### Why It Matters

Gas is not just a fee mechanism — it is the entire security model preventing computational abuse. Gas optimization directly impacts user experience (lower fees) and protocol competitiveness. Gas griefing is a real attack vector in multi-contract systems. Understanding the 63/64 rule is essential for safe inter-contract call design.

### Real-World Context

- Uniswap V2 swap functions were heavily optimized (~130,000 gas per swap). Uniswap V3 improved to ~100,000 gas with more complex math.
- The Istanbul upgrade (2019) increased SLOAD cost from 200 to 800 gas, breaking some existing contracts that relied on the old costs.
- The 2016 Shanghai DDoS attacks exploited cheap EXTCODESIZE and BALANCE opcodes to spam the network with computationally expensive transactions.

### Tools / Technologies

| Tool                   | Purpose                                           |
| ---------------------- | ------------------------------------------------- |
| `foundry` gas reports  | Automatic gas benchmarking per function           |
| `hardhat-gas-reporter` | Gas analysis plugin for Hardhat                   |
| `evmdiff.com`          | Compare opcodes and gas costs across EVM versions |
| `evm.codes`            | Per-opcode gas cost breakdown                     |

### Deep Mastery Questions

1. Why does storing a `uint128` and a `uint128` in separate storage slots cost twice as much as packing them into one `uint256` slot? What exactly happens at the EVM level?
2. Explain the 63/64 rule. In a deep call chain (A → B → C → D → E), how much of A's original gas reaches E?
3. Why did EIP-3529 remove gas refunds for clearing storage? What was the attack vector enabled by the old refund system?
4. A function reads from a storage variable 5 times in a loop. How much total gas is spent on SLOAD? How would you optimize it?
5. Why are zero bytes in calldata cheaper than non-zero bytes? What does this imply for transaction encoding optimization?
6. What is a "gas bomb" attack, and how does the `gasleft()` check defend against it in a contract that processes deposits?

### Hands-On Exercises

1. **Gas comparison**: Write a function that reads a storage variable in a loop 10 times — once without caching, once with a memory cache. Measure gas difference using Foundry.
2. **Slot packing**: Write a contract with four `uint64` variables. First, declare them individually (four separate slots). Then, declare them in one `uint64[4]` stored in a struct. Measure the gas cost difference for reading and writing all four.
3. **Custom error optimization**: Write a function with `require(condition, "Long error string")`. Convert to a custom error. Measure the gas savings on successful calls and on reverts.
4. **Calldata vs memory**: Write a function that takes a large array parameter, once as `memory` and once as `calldata`. Measure gas on the call.

### Mini Build Task

**Gas Benchmark Suite.** Write 10 pairs of "naive" and "optimized" implementations of common operations (e.g., array traversal, accumulation, token transfer). For each pair, write a Foundry test that compares the gas cost. Produce a report showing the gas savings of each optimization and explain the EVM-level reason.

---

## Topic 3: Solidity Engineering

### Explanation

Solidity is a statically-typed, compiled language with Rust-like ownership semantics for data locations and EVM-specific behaviors that make it unlike any other language.

**Type System**

| Type                         | Notes                                                                                                             |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `uint256`, `int256`          | Default integer type. Arithmetic is checked (overflow reverts) in >=0.8.                                          |
| `address`, `address payable` | 20-byte Ethereum address. Only `address payable` can receive ETH via `transfer`/`send`.                           |
| `bytes32`, `bytes`           | Fixed and dynamic byte arrays. `bytes32` is value type (stored in one slot).                                      |
| `string`                     | Dynamic, UTF-8 encoded. Expensive to manipulate on-chain.                                                         |
| `bool`                       | ABI-encoded as a 32-byte word (not 1 bit) — no packing.                                                           |
| `struct`                     | Composite value type. Packing applies within structs.                                                             |
| `mapping(K => V)`            | Hash table. Keys are hashed with slot index to find storage location. `mapping` values are NOT iterable natively. |
| `enum`                       | Represented as `uint8` internally.                                                                                |

**ABI Encoding**

The Application Binary Interface (ABI) is the standard encoding for function calls:

- Function selector: first 4 bytes of `keccak256("functionName(paramTypes)")`.
- Arguments: ABI-encoded after the selector. Static types are inlined; dynamic types (arrays, strings) use offsets.

Understanding ABI encoding is required for: manual transaction construction, low-level call composition, and debugging call failures.

**Function Visibility and State Mutability**

| Modifier   | Meaning                                                                                   |
| ---------- | ----------------------------------------------------------------------------------------- |
| `public`   | Callable externally and internally; generates getter for state variables                  |
| `external` | Only callable from outside the contract                                                   |
| `internal` | Only callable from this contract and inheriting contracts                                 |
| `private`  | Only callable from this exact contract                                                    |
| `view`     | Reads state but does not modify it. Compiled to STATICCALL internally for external calls. |
| `pure`     | Neither reads nor modifies state.                                                         |
| `payable`  | Can receive ETH. Functions without this modifier revert on ETH receipt.                   |

**Critical: `private` does not mean "secret."** All contract storage is publicly readable from the blockchain. `private` only restricts _which code_ can call the function or read the variable.

**Inheritance and Linearization**

Solidity uses C3 linearization for multiple inheritance. The order of inheritance matters significantly — functions are resolved in reverse order of the inheritance list.

**Libraries**

- **Internal libraries**: Functions are inlined at compile time. No deployment cost.
- **External libraries**: Deployed to a separate address. Called via DELEGATECALL. Must be linked at deployment time.

**Error Handling**

- `require(condition, "error")`: Reverts on false. Gas refunded for remaining gas.
- `revert CustomError()`: Reverts with a typed error (Solidity 0.8+). More gas-efficient than string errors.
- `assert(condition)`: Reverts on false with all gas consumed. Used for invariant violations.
- Try/catch: Handles reverts from external calls.

**Yul / Inline Assembly**

Solidity allows inline assembly via `assembly { ... }` blocks using Yul, EVM's intermediate language. Used for:

- Gas optimization (avoid bounds checking, direct slot access).
- Low-level operations (extcodesize, keccak256 on dynamic data).
- Proxy contracts (custom dispatch in fallback via DELEGATECALL).

### Why It Matters

Solidity has subtle traps that are not obvious from its syntax: integer division truncates, address type conversion requires explicit casting, `delete` on a mapping doesn't clear it, and `msg.value` persists through internal calls. Mastering Solidity means internalizing these traps and writing defensively around them.

### Real-World Context

- The USDC stablecoin contract (Circle) is a feature-rich ERC-20 with blacklisting, pause, and upgrade capabilities. Reading it teaches production-grade Solidity patterns.
- OpenZeppelin Contracts is the canonical reference library. Virtually every DeFi protocol extends OZ's `Ownable`, `AccessControl`, `ERC20`, or `ReentrancyGuard`.
- Solidity's `override` and `virtual` keywords enforce explicit override acknowledgment — a lesson learned from security bugs caused by accidental function overrides.

### Tools / Technologies

| Tool                   | Purpose                                        |
| ---------------------- | ---------------------------------------------- |
| `solc`                 | Solidity compiler                              |
| Foundry (`forge`)      | All-in-one testing and build tool              |
| Hardhat                | Node.js-based Ethereum development environment |
| OpenZeppelin Contracts | Audited contract library                       |
| `slither`              | Static analysis for Solidity                   |
| Remix IDE              | Browser-based IDE for quick iteration          |

### Deep Mastery Questions

1. What is the difference between `memory` and `storage` reference types? What happens to memory when a function returns?
2. Why does Solidity hash mapping keys with the storage slot? What are the implications for detecting mapping key collisions?
3. What is the order of inheritance in `Contract C is A, B` if A and B both define the same function? How is this resolved?
4. Why does `delete myMapping[key]` not clear the mapping's content? What does it actually do?
5. Explain ABI encoding for `callFoo(uint256, bool, bytes)`. What does the calldata look like byte-by-byte?
6. What is the "short-circuit evaluation" behavior of `&&` and `||` in Solidity? How can you exploit this for gas optimization?
7. Why is `msg.value` the same across reentrant calls in a single transaction? What attack vector does this create if not handled?
8. What is the `receive()` function vs the `fallback()` function? Under what conditions is each triggered?

### Hands-On Exercises

1. **ABI encoder**: Manually construct the calldata for calling `transfer(address to, uint256 amount)` with hardcoded values. Verify by comparing to `abi.encodeWithSignature`.
2. **Inheritance exercise**: Create a diamond inheritance scenario (A, B extend Base; C extends A and B). Implement a function in Base that is overridden in B. Verify C's resolution order via C3 linearization.
3. **Assembly exercise**: Implement `memcpy(dest, src, len)` in Yul inline assembly. Compare gas to the Solidity equivalent.
4. **Storage layout**: Write a contract with multiple state variables of different types. Use Foundry's `cast storage` to read each slot directly. Verify the packing of small types.

### Mini Build Task

**Build a feature-complete ERC-20 token from scratch.** Without inheriting OpenZeppelin, implement: `transfer`, `transferFrom`, `approve`, `mint`, `burn`, `Ownable` (only owner can mint/burn), `Pausable` (owner can pause all transfers), and `permit` (EIP-2612 signature-based approval). Write natspec documentation for all functions. This exercise forces mastery of Solidity fundamentals without shortcuts.

---

## Topic 4: Smart Contract Architecture Patterns

### Explanation

**Checks-Effects-Interactions (CEI)**

The most important pattern for reentrancy prevention. Structure every state-modifying function in three phases:

1. **Checks**: Validate all conditions.
2. **Effects**: Update all state variables.
3. **Interactions**: Call external contracts.

Violation: performing an external call before updating state allows the callee to reenter and observe stale state.

**Pull-over-Push Payments**

Never send ETH via push (calling the recipient from your contract). Instead, track owed balances in a mapping and let recipients `withdraw()` themselves. This isolates the risk of the recipient's address being a malicious contract.

**Proxy Patterns for Upgradeability**

Upgradeability in an immutable system is an architectural challenge. Proxy patterns solve this via DELEGATECALL:

- **Transparent Proxy (EIP-1967)**: A proxy contract holds state and delegates all calls to a logic contract. The logic contract defines the functions. To upgrade, point the proxy to a new logic contract. Complexity: admin must be distinguished from regular users to avoid selector clashes.
- **UUPS Proxy (EIP-1822)**: The upgrade function lives in the logic contract itself (not the proxy). More gas-efficient proxy, but self-destructing a bad logic contract can brick the proxy.
- **Beacon Proxy**: Multiple proxies share a single "beacon" contract that points to the logic contract. One upgrade at the beacon upgrades all proxy instances.

**Storage Layout in Proxies**

DELEGATECALL executes code in the caller's storage context. If the proxy and logic contract have different storage layouts, variables overwrite each other — a storage collision. **EIP-1967** defines standard storage slots for proxy admin and implementation addresses using pseudo-random slots within the implementation's storage space.

**Factory Pattern**

A factory contract deploys instances of a child contract. Enables: batch deployment, tracking deployed instances, and centralized initialization. Often combined with CREATE2 for deterministic child addresses.

**Access Control**

- `Ownable`: Single owner (usually deployer) controls privileged functions.
- `AccessControl` (OpenZeppelin): Role-based access with granular permissions. Multiple roles (e.g., MINTER_ROLE, PAUSER_ROLE). Roles can be granted/revoked per address.
- `Timelock`: Privileged operations must be queued and executed after a delay, giving users time to exit before potentially harmful changes take effect.

### Why It Matters

Production smart contracts are not isolated functions — they are systems of interacting contracts that manage thousands of participants and billions of dollars. Architecture patterns provide the structure for multi-contract systems, upgrade paths, and privilege management. The wrong architecture creates systemic risk; the right architecture creates a system that evolves safely.

### Real-World Context

- Compound protocol's governor uses a Timelock: all governance proposals have a 2-day delay before execution.
- Uniswap V3 uses a factory pattern (UniswapV3Factory) to deploy pair contracts with CREATE2 for deterministic pool addresses.
- The Audius hack (2022) exploited a DELEGATECALL proxy storage collision bug to steal $6M.
- OpenZeppelin's TransparentUpgradeableProxy is deployed by thousands of DeFi protocols.

### Tools / Technologies

| Tool                           | Purpose                                      |
| ------------------------------ | -------------------------------------------- |
| OpenZeppelin Upgrades Plugins  | Safe proxy deployment and upgrade validation |
| `hardhat-deploy`               | Deployment scripts with proxy management     |
| `foundry` upgradeability tests | Test proxy pattern correctness               |

### Deep Mastery Questions

1. Trace the exact DELEGATECALL path in a Transparent Proxy. What is the storage slot used for the implementation address (EIP-1967)? Why was that specific slot chosen?
2. What is a "storage collision" in a proxy contract? Give a concrete example of two variables in the proxy and logic contract that would collide.
3. Why is the timelock pattern valuable even if you trust the protocol admin? What scenario does it protect against?
4. In UUPS, why is it dangerous to call `upgradeTo` from within the logic contract's `initialize` function?
5. What is the difference between `constructor` and `initialize` in upgradeable contracts, and why can't you use a constructor with a proxy?
6. Why does OpenZeppelin's Upgrades plugin warn you if you add a new state variable at the beginning of a contract instead of at the end?

### Hands-On Exercises

1. **Proxy implementation**: Deploy a UUPS proxy and logic contract. Call a function. Upgrade the logic to a new version with an additional function. Verify both old state is preserved and new function works.
2. **Storage layout validation**: Use OpenZeppelin's `storageLayout` plugin to validate that an upgrade doesn't break the storage layout.
3. **Factory + CREATE2**: Write a factory that deploys "vaults" with CREATE2. Compute the vault address off-chain before deployment. Verify it matches after deployment.
4. **Access control system**: Implement a three-tiered access control system (admin, operator, user) for a contract that manages a simple token vault.

### Mini Build Task

**Build an Upgradeable Token Vault.** A UUPS-upgradeable contract where users can deposit ERC-20 tokens. Version 1: basic deposit and withdraw with per-user balance tracking. Version 2: add a withdrawal fee (without breaking existing balances). Demonstrate: deploying V1, making deposits, upgrading to V2, verifying balances are preserved, verifying fee is now charged on withdrawal.

---

## Topic 5: Testing and Debugging Smart Contracts

### Explanation

**Testing Hierarchy**

| Test Type           | What It Checks                               | Tools                       |
| ------------------- | -------------------------------------------- | --------------------------- |
| Unit tests          | Individual function behavior                 | Foundry, Hardhat            |
| Integration tests   | Multi-contract interactions                  | Foundry with forks, Hardhat |
| Fuzz tests          | Function behavior over random inputs         | Foundry `forge fuzz`        |
| Invariant tests     | System-wide properties that must always hold | Foundry `forge invariant`   |
| Formal verification | Mathematical proof of correctness            | Certora, Halmos             |

**Foundry Testing**

Foundry is the preferred tool for serious smart contract testing. Key features:

```solidity
contract TokenTest is Test {
    Token token;

    function setUp() public {
        token = new Token(1_000_000e18);
    }

    function test_transferReducesSenderBalance() public {
        token.transfer(alice, 100e18);
        assertEq(token.balanceOf(alice), 100e18);
    }

    // Fuzz test: runs 256 times with random inputs
    function test_fuzz_transferNeverOverflows(uint256 amount) public {
        amount = bound(amount, 0, token.balanceOf(address(this)));
        token.transfer(alice, amount);
        assertLe(token.balanceOf(alice), token.totalSupply());
    }

    // Invariant test: Foundry calls handler functions in random order,
    // then checks invariants after each call sequence
    function invariant_totalSupplyEqualsAllBalances() public {
        assertEq(token.totalSupply(), sumOfAllBalances());
    }
}
```

**Mainnet Forking**

Foundry and Hardhat support forking mainnet state at a specific block. This enables testing your contract against real deployed contracts (real USDC, real Uniswap pools) without touching mainnet.

```bash
forge test --fork-url $ETH_RPC_URL --fork-block-number 18000000
```

**Trace Debugging**

Foundry's `forge test -vvvv` outputs a full execution trace with every call, every storage access, and revert reasons. `cast run <txhash> --rpc-url` traces a mainnet transaction.

**Formal Verification with Halmos**

Halmos performs symbolic execution — instead of running with concrete input values, it explores all possible inputs using constraint solving (SMT). It can prove that a property holds for _all_ inputs or find a counterexample.

### Why It Matters

Testing in smart contract development is not optional hygiene — it is mandatory engineering. The cost of a bug in production is irreversible fund loss. The DeFi ecosystem has lost billions to exploits that would have been caught by invariant tests. Coverage metrics are insufficient; what matters is whether you have tested the _right_ properties.

### Real-World Context

- The Euler Finance hack ($197M, 2023) would have been prevented by an invariant test checking `totalBorrow <= totalDeposits` — a property that was violated by the exploit.
- The Mango Markets exploit (2022) manipulated oracle prices. A fuzz test varying oracle prices would have revealed the issue.
- Paradigm's research team discovered several Uniswap V3 edge cases via fuzz testing before the protocol launched.
- OpenZeppelin's test suite for their contracts covers 100% of lines and branches.

### Tools / Technologies

| Tool              | Purpose                                              |
| ----------------- | ---------------------------------------------------- |
| Foundry (`forge`) | Testing, fuzzing, invariant testing, deployment      |
| Hardhat           | JS-based testing with Mocha/Chai                     |
| `forge coverage`  | Measure code coverage                                |
| Halmos            | Symbolic execution for Solidity                      |
| Certora Prover    | Formal verification via specification language (CVL) |
| `slither`         | Static analysis — detects 90+ vulnerability patterns |
| `mythril`         | Symbolic execution-based security scanner            |

### Deep Mastery Questions

1. What is the difference between a fuzz test and an invariant test? When would you use each?
2. What invariants would you define for a lending protocol? Give 5 specific invariants that would, if violated, indicate an exploit.
3. Why is line coverage an insufficient metric for smart contract test quality?
4. How does mainnet forking enable you to test real protocol interactions? What are its limitations?
5. What are the soundness trade-offs of symbolic execution (Halmos) vs. random fuzzing? Which finds more bugs, and under what conditions?
6. How would you test a reentrancy guard? What does the test need to simulate?
7. In Foundry's invariant testing, what is a "handler" contract and why is it needed?

### Hands-On Exercises

1. **Invariant testing**: Write 5 invariant tests for the ERC-20 token from Topic 3. Ensure total supply is always conserved, balances never go negative, and approval accounting is correct.
2. **Fuzz a bug**: Write a function with a subtle bug (e.g., rounding error that can drain funds). Write a fuzz test that discovers the bug.
3. **Fork test**: Fork Ethereum mainnet. Write a test that uses real USDC to test your deposit contract.
4. **Static analysis run**: Run `slither` on your ERC-20 token. Analyze each finding — is it a true positive or false positive? Fix the true positives.

### Mini Build Task

**Test Coverage Engineering.** Take an existing open-source DeFi contract (e.g., a Compound fork or Uniswap V2 pair). Write a comprehensive Foundry test suite including: unit tests for every function, fuzz tests for all arithmetic operations, and 10+ invariant tests. Achieve 100% line coverage. Produce a security report identifying any gaps between coverage and actual security assurance.

---

## Topic 6: Smart Contract Security Vulnerabilities

### Explanation

**Reentrancy**

The most infamous smart contract vulnerability class. A malicious external call executes before the caller's state is updated.

```solidity
// VULNERABLE: Effects after Interactions
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount);
    (bool success, ) = msg.sender.call{value: amount}(""); // External call!
    require(success);
    balances[msg.sender] -= amount; // State updated AFTER call
}
```

Attack: the recipient's `receive()` function re-calls `withdraw()` before `balances` is updated. Fix: Update state BEFORE the external call (CEI pattern) or use a ReentrancyGuard mutex.

Variants:

- **Cross-function reentrancy**: Reenters via a _different_ function than the one called.
- **Read-only reentrancy**: A view function is called mid-execution and returns stale state, which another protocol trusts as correct.

**Integer Overflow/Underflow (Pre-0.8)**

Before Solidity 0.8, arithmetic silently wrapped around. `uint8(255) + 1 == 0`. The Proof-of-Concept exploit of the BatchOverflow bug on BEC token (2018) minted $900M worth of tokens from overflow.

Fix: Use Solidity 0.8+ (checked arithmetic by default) or OpenZeppelin's `SafeMath` for 0.7 and earlier.

**Access Control Failures**

Missing or incorrect permission checks. Functions that should be restricted are public or reachable via an unexpected call path.

Example: The Parity multisig wallet bug left the `initWallet` function public, allowing anyone to become the owner.

**Front-Running and Sandwich Attacks**

MEV bots monitor the mempool and inject transactions before and after a victim's transaction:

- **Front-run**: Bot submits the same trade ahead of the victim (higher gas price), taking the favorable price.
- **Sandwich**: Bot buys before and sells after the victim's large swap, profiting from their price impact.

Defenses: commit-reveal schemes, private mempools (Flashbots Protect), slippage limits.

**Oracle Manipulation**

On-chain oracles (like Uniswap V2 spot price) can be manipulated within a single transaction using flash loans. Protocol that reads a spot price (not a time-weighted average) is vulnerable.

Example: The Harvest Finance exploit (2020, $34M) manipulated USDC/USDT prices in Curve's pool via flash loans to drain the vault.

Fix: Use time-weighted average prices (TWAP), multiple oracle sources, and Chainlink decentralized oracles.

**Flash Loan Attacks**

Flash loans provide unsecured, immediate liquidity that must be repaid in the same transaction. They amplify the attacker's effective capital for oracle manipulation, governance attacks, and arbitrage exploits.

**DELEGATECALL Storage Collision**

When a proxy uses DELEGATECALL, the logic contract's storage layout must exactly match the proxy's. If the proxy has state variables in slots 0–2 and the logic contract defines different variables in those slots, writes to the logic variables corrupt the proxy's admin address.

**Signature Replay**

A valid signature for one transaction can be replayed in another context if the message doesn't include a unique identifier (nonce, chain ID, contract address).

Example: A signed "approve this withdrawal" message replayed on a fork chain or against a different contract instance.

Fix: EIP-712 typed structured data signing includes domain separator (chain ID + contract address + version).

**tx.origin Phishing**

`tx.origin` returns the original EOA that initiated the transaction. A malicious contract tricking a user into calling it can then call the victim contract and pass the `require(tx.origin == owner)` check.

Fix: Never use `tx.origin` for authorization. Use `msg.sender`.

**Denial of Service via Gas Exhaustion**

A contract that loops over an unbounded array can be griefed by an attacker who makes the array large enough that the loop runs out of gas. Example: a contract that distributes ETH to all holders in a loop — anyone can add many zero-value holder addresses.

### Why It Matters

Every vulnerability class here has caused real, significant financial losses. The combined DeFi hacks from 2020–2024 total over $10 billion. Cryptocurrency transactions are irreversible. Smart contract developers must internalize every vulnerability class and audit their code against each one before deployment.

### Real-World Context

| Exploit         | Year | Loss  | Vulnerability                                |
| --------------- | ---- | ----- | -------------------------------------------- |
| The DAO         | 2016 | $60M  | Reentrancy                                   |
| Parity Multisig | 2017 | $150M | Missing access control                       |
| bZx             | 2020 | $8M   | Oracle manipulation (flash loan)             |
| Harvest Finance | 2020 | $34M  | Oracle manipulation                          |
| Cream Finance   | 2021 | $130M | Reentrancy (cross-function)                  |
| Nomad Bridge    | 2022 | $190M | Access control / initialization              |
| Euler Finance   | 2023 | $197M | Missing health check (unexpected state path) |

### Tools / Technologies

| Tool      | Purpose                                        |
| --------- | ---------------------------------------------- |
| `slither` | Static analysis for 90+ vulnerability patterns |
| `mythril` | Symbolic execution vulnerability scanner       |
| `echidna` | Property-based fuzzer for security testing     |
| Foundry   | Exploit PoC scripting                          |
| Tenderly  | Transaction simulation and trace analysis      |

### Deep Mastery Questions

1. Explain the exact reentrant call sequence in The DAO exploit. What state was read, what was not yet updated, and what was the financial impact of each reentrant call?
2. How does the cross-function reentrancy attack work? Why does a ReentrancyGuard mutex prevent it?
3. What is the read-only reentrancy attack, and why is a ReentrancyGuard insufficient to prevent it?
4. A protocol relies on Uniswap V2 spot price for collateral valuation. Walk through a flash loan attack that exploits this. How much profit is possible?
5. Why does EIP-712 structured signing prevent signature replay attacks? What specific fields make each signature unique?
6. You have a smart contract that distributes rewards to all users in a `for` loop. Identify the DoS vector and propose a fix.
7. Explain the storage collision bug in a proxy contract where the proxy stores `address admin` in slot 0 and the logic contract stores `address owner` in slot 0. What happens when the logic contract writes to `owner`?
8. Why is `tx.origin` considered dangerous for authorization? Construct an exact attack scenario.

### Hands-On Exercises

1. **Reentrancy CTF**: Implement the vulnerable `withdraw` function, implement an attacking contract, and run the exploit in a Foundry test. Then fix the vulnerability with CEI and ReentrancyGuard, and verify the attack no longer works.
2. **Oracle manipulation PoC**: Fork mainnet with a Uniswap V2 pool. Write an attack contract that uses a flash loan to manipulate the spot price, exploits a vulnerable lending protocol, and repays the flash loan. Run as a Foundry test against a fork.
3. **Access control audit**: Take a given Solidity contract (intentionally containing missing access control). Use slither to detect the issue. Write a proof-of-concept exploit.
4. **Signature replay**: Implement a vulnerable signature-based withdrawal system. Build an exploit that replays a valid signature. Fix with EIP-712.
5. **Storage collision**: Write a proxy and a logic contract with a known storage collision. Demonstrate that upgrading corrupts the proxy's admin address. Fix by using EIP-1967 storage slots.

### Mini Build Task

**Audit a Protocol.** Take an open-source, publicly known vulnerable contract (e.g., from Damn Vulnerable DeFi or EtherNaut). For each of the 12+ vulnerability classes covered in this topic: (a) describe whether the contract is vulnerable, (b) if vulnerable, write a working exploit as a Foundry test, (c) propose a fix. Write a formal security report with: executive summary, findings ranked by severity, recommended remediations.

---

## Mini Projects

### Project 1: Minimal DeFi Lending Protocol

Build a basic lending protocol: deposit ERC-20 collateral, borrow against it, repay loans, and liquidate under-collateralized positions. Implement proper access control, reentrancy guards, and a simulated price oracle. Avoid all known vulnerability classes.

### Project 2: Upgradeable NFT Marketplace

A UUPS-upgradeable contract where sellers can list ERC-721 NFTs at a price, and buyers can purchase them (ERC-20 payment). Version 1: fixed-price sales. Version 2: add auction functionality. Demonstrate upgrade without losing seller listings.

### Project 3: Governance System

Implement on-chain governance: token holders vote on proposals, proposals have a time delay before execution, and approved proposals call arbitrary functions via a timelock. Model after Compound Governor Bravo.

---

## Capstone Project: Decentralized Stablecoin System

Build a minimal over-collateralized stablecoin (inspired by MakerDAO):

1. **Vault contract**: Users deposit ETH (or a mock ERC-20) as collateral and mint a stablecoin ("DSC") up to 150% collateralization ratio.
2. **Stablecoin (DSC)**: ERC-20 token, mintable only by the vault contract.
3. **Price oracle**: Use a mock oracle (updatable by admin) to simulate ETH price feeds.
4. **Liquidation**: If a vault falls below 150% collateralization ratio, liquidators can repay the stablecoin debt and claim the collateral at a discount.
5. **Stability mechanism**: Burning DSC increases the collateralization ratio.
6. **Test suite**: 100% line coverage. Invariant tests: total DSC supply ≤ total collateral value, no undercollateralized vaults can exist after each action.
7. **Security audit**: Run slither and mythril. Document and fix all findings.

---

## Common Mistakes & Misconceptions

| Mistake                                   | Reality                                                                                                                                                        |
| ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "`private` means secret"                  | Private state variables are fully readable by anyone who reads the blockchain. Use encryption off-chain for actual secrets.                                    |
| "Solidity 0.8 prevents all overflow"      | `unchecked` blocks disable overflow protection. Libraries and assembly bypass it entirely.                                                                     |
| "ReentrancyGuard prevents all reentrancy" | Read-only reentrancy bypasses the guard because `view` functions don't trigger it.                                                                             |
| "Audit = complete security"               | Audits find bugs based on auditor skill and scope. Formal verification provides stronger guarantees. Novel attack vectors are discovered post-audit regularly. |
| "Gas fees are the user's problem"         | Contracts that make users pay excessive gas will not be used. Gas optimization is a UX requirement.                                                            |
| "Hard-coded addresses are fine"           | Dependency on hard-coded deployed contract addresses creates fragility across networks and protocol upgrades.                                                  |

---

## Security Considerations

At this stage, internalize these disciplines:

1. **CEI is not optional.** Structure every state-modifying function with Checks-Effects-Interactions. No exceptions.
2. **Threat model before code.** Before writing a contract, ask: who can call this? What can they do with it? What is the worst-case outcome of each function being called by a malicious actor?
3. **Principle of least privilege.** Give each function only the permissions it needs. Separate admin functions with timelocks.
4. **Assume hostile callees.** Any external contract you call can be malicious, can reenter, and can revert to cause denial of service.
5. **Oracle paranoia.** Never trust a single on-chain price source for financial decisions without manipulation resistance (TWAP, multiple sources, circuit breakers).
6. **Invariant-first design.** Before implementing a protocol, enumerate its invariants. Then write invariant tests before writing implementation.

---

## Readings & Resources

**Books**

- _Smart Contract Security_ (Trail of Bits, free) — Comprehensive security focused guide
- _Mastering Ethereum_ — Chapters 7–13 (Solidity, EVM, smart contracts in depth)

**Papers**

- Luu et al. — _"Making Smart Contracts Smarter"_ (2016) — Formal analysis of EVM vulnerabilities
- Hildenbrandt et al. — _"KEVM: A Complete Semantics of the EVM"_ (2018)
- EIP-1967: Standard Proxy Storage Slots
- EIP-712: Typed Structured Data Signing
- EIP-3529: Reduction in refunds

**Documentation**

- Solidity documentation: docs.soliditylang.org
- Foundry book: book.getfoundry.sh
- OpenZeppelin Contracts: docs.openzeppelin.com/contracts

**Security Resources**

- SWC Registry (Smart Contract Weakness Classification): swcregistry.io
- Damn Vulnerable DeFi: damnvulnerabledefi.xyz
- EtherNaut: ethernaut.openzeppelin.com
- Trail of Bits public audits: github.com/trailofbits/publications
- Rekt News (post-mortems of exploits): rekt.news
- Immunefi blog (bug bounty post-mortems)

---

<div style="display:flex;align-items:center;justify-content:space-between;padding:14px 0;margin-top:48px;">
  <a href="./02_Blockchain_Core.md" style="display:flex;align-items:center;gap:8px;text-decoration:none;color:#8b949e;font-size:0.875rem;padding:6px 10px;border-radius:6px;">
    <svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor"><path d="M9.78 12.78a.75.75 0 0 1-1.06 0L4.47 8.53a.75.75 0 0 1 0-1.06l4.25-4.25a.75.75 0 0 1 1.06 1.06L6.06 8l3.72 3.72a.75.75 0 0 1 0 1.06z"/></svg>
    <span style="display:flex;flex-direction:column;line-height:1.3;">
      <span style="font-size:0.7rem;opacity:0.55;text-transform:uppercase;letter-spacing:0.06em;">Previous</span>
      <span style="font-weight:500;">02 · Blockchain Core</span>
    </span>
  </a>
  <span style="font-size:0.7rem;color:#8b949e;opacity:0.5;letter-spacing:0.08em;text-transform:uppercase;">Stage 3 of 6 · Smart Contracts & EVM</span>
  <a href="./04_Ecosystem_Scaling_and_Crypto_Economics.md" style="display:flex;align-items:center;gap:8px;text-decoration:none;color:#8b949e;font-size:0.875rem;padding:6px 10px;border-radius:6px;">
    <span style="display:flex;flex-direction:column;line-height:1.3;text-align:right;">
      <span style="font-size:0.7rem;opacity:0.55;text-transform:uppercase;letter-spacing:0.06em;">Next</span>
      <span style="font-weight:500;">04 · Ecosystem & Scaling</span>
    </span>
    <svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor"><path d="M6.22 3.22a.75.75 0 0 1 1.06 0l4.25 4.25a.75.75 0 0 1 0 1.06l-4.25 4.25a.75.75 0 0 1-1.06-1.06L9.94 8 6.22 4.28a.75.75 0 0 1 0-1.06z"/></svg>
  </a>
</div>
