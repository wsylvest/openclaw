# NovaCoin: Technical Specification

> Structured for parallel development with Claude Code sub-agents.
> Each module defines its interfaces first, then implementation is parallelizable.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Client / RPC Layer                    │
│            JSON-RPC 2.0 + WebSocket Subscriptions       │
├─────────────────────────────────────────────────────────┤
│                   Transaction Pool                      │
│          Priority queue, dedup, validation               │
├──────────────┬──────────────────────┬───────────────────┤
│  Consensus   │   Execution Engine   │   State Store     │
│  (HotStuff)  │   (Native + MoveVM)  │   (Merkle Trie)  │
├──────────────┴──────────────────────┴───────────────────┤
│                   Networking (libp2p)                    │
│         Gossip, Block Sync, Peer Discovery              │
├─────────────────────────────────────────────────────────┤
│                 Cryptography Layer                       │
│     Ed25519 · BLS12-381 · SHA-256 · Blake3 · Poseidon  │
└─────────────────────────────────────────────────────────┘
```

---

## Module 1: Core Types & Cryptography (`nova-crypto`)

**Purpose:** Shared types, traits, and cryptographic primitives used by all other modules.

### Key Types

```rust
// --- Identifiers ---
pub struct AccountAddress(pub [u8; 32]);  // Ed25519 public key hash
pub struct TransactionHash(pub [u8; 32]); // Blake3 hash of signed tx
pub struct BlockHash(pub [u8; 32]);       // Blake3 hash of block header
pub struct StateRoot(pub [u8; 32]);       // Merkle trie root hash

// --- Tokens ---
pub struct Amount(pub u128);              // Atomic units (18 decimals)
pub struct TokenId(pub [u8; 32]);         // Native token = [0; 32]
pub type Nonce = u64;
pub type BlockHeight = u64;
pub type EpochId = u64;
pub type Timestamp = u64;                 // Milliseconds since Unix epoch

// --- Keys ---
pub struct PublicKey(pub [u8; 32]);       // Ed25519
pub struct Signature(pub [u8; 64]);       // Ed25519
pub struct BlsPublicKey(pub [u8; 48]);    // BLS12-381
pub struct BlsSignature(pub [u8; 96]);    // BLS12-381

// --- Transactions ---
pub struct Transaction {
    pub sender: AccountAddress,
    pub nonce: Nonce,
    pub payload: TransactionPayload,
    pub max_fee: Amount,
    pub expiry: Timestamp,               // Optional: tx expires after this
    pub chain_id: u16,                    // Replay protection
}

pub struct SignedTransaction {
    pub transaction: Transaction,
    pub signature: Signature,
    pub public_key: PublicKey,
}

pub enum TransactionPayload {
    Transfer { to: AccountAddress, amount: Amount, token: TokenId },
    CreateToken { supply: Amount, decimals: u8, metadata: TokenMetadata },
    DexPlaceOrder { pair: TradingPair, side: Side, price: Amount, qty: Amount },
    DexCancelOrder { order_id: u64 },
    EscrowCreate { recipient: AccountAddress, amount: Amount, unlock: EscrowCondition },
    EscrowFinish { escrow_id: u64 },
    EscrowCancel { escrow_id: u64 },
    MultiSigCreate { signers: Vec<AccountAddress>, threshold: u8 },
    MultiSigApprove { multisig_id: u64, tx_hash: TransactionHash },
    StakeDelegate { validator: AccountAddress, amount: Amount },
    StakeUndelegate { validator: AccountAddress, amount: Amount },
    GovernancePropose { proposal: Proposal },
    GovernanceVote { proposal_id: u64, vote: Vote },
    ContractDeploy { bytecode: Vec<u8> },
    ContractCall { contract: AccountAddress, function: String, args: Vec<u8> },
}

// --- Blocks ---
pub struct BlockHeader {
    pub height: BlockHeight,
    pub parent_hash: BlockHash,
    pub state_root: StateRoot,
    pub transactions_root: TransactionHash, // Merkle root of tx hashes
    pub timestamp: Timestamp,
    pub proposer: AccountAddress,
    pub epoch: EpochId,
    pub qc: QuorumCertificate,              // HotStuff QC from parent
}

pub struct Block {
    pub header: BlockHeader,
    pub transactions: Vec<SignedTransaction>,
}

// --- Account State ---
pub struct Account {
    pub address: AccountAddress,
    pub nonce: Nonce,
    pub balances: BTreeMap<TokenId, Amount>, // Multi-token balances
    pub code: Option<Vec<u8>>,              // Smart contract bytecode
    pub storage_root: StateRoot,            // Contract storage trie root
}
```

### Traits

```rust
pub trait Signer {
    fn sign(&self, message: &[u8]) -> Signature;
    fn public_key(&self) -> PublicKey;
}

pub trait Verifier {
    fn verify(public_key: &PublicKey, message: &[u8], signature: &Signature) -> bool;
}

pub trait Hasher {
    fn hash(data: &[u8]) -> [u8; 32];
}

pub trait MerkleTree {
    fn insert(&mut self, key: &[u8], value: &[u8]);
    fn get(&self, key: &[u8]) -> Option<Vec<u8>>;
    fn remove(&mut self, key: &[u8]);
    fn root(&self) -> StateRoot;
    fn proof(&self, key: &[u8]) -> MerkleProof;
    fn verify_proof(root: &StateRoot, key: &[u8], proof: &MerkleProof) -> bool;
}
```

### Dependencies (proven crates)

```toml
ed25519-dalek = "2"        # Ed25519 signatures
bls12_381 = "0.8"          # BLS aggregate signatures
blake3 = "1"               # Fast hashing
sha2 = "0.10"              # SHA-256 (compatibility)
borsh = "1"                # Binary serialization (faster than bincode)
serde = "1"                # JSON serialization for RPC
```

---

## Module 2: Consensus Engine (`nova-consensus`)

**Purpose:** HotStuff BFT consensus with PoS validator selection.

### Why HotStuff (not Tendermint, not PBFT)

| Property | PBFT | Tendermint | HotStuff |
|----------|------|-----------|----------|
| Message complexity | O(n²) | O(n²) | **O(n)** |
| View change | Expensive | Expensive | **Same as normal** |
| Pipelining | No | No | **Yes (chained)** |
| Responsiveness | No | No | **Optimistic** |
| Used by | Hyperledger | Cosmos chains | **Aptos, Diem** |

HotStuff achieves linear message complexity through **threshold BLS signatures** —
validators sign with BLS, and signatures are aggregated into a single QC. This means
the protocol scales to hundreds of validators without degradation.

### Core Protocol

```rust
// Chained HotStuff operates in a 3-phase pipeline:
// Each block carries a QC for its parent, forming an implicit 3-chain.
//
// Block B0 <-- Block B1 (QC for B0) <-- Block B2 (QC for B1) <-- Block B3 (QC for B2)
//              PREPARE                   PRE-COMMIT               COMMIT (B0 finalized)
//
// When B3 is certified, B0 becomes final (3-chain rule).

pub struct QuorumCertificate {
    pub block_hash: BlockHash,
    pub height: BlockHeight,
    pub aggregated_signature: BlsSignature,  // Aggregated BLS from 2f+1 validators
    pub signers_bitfield: Vec<u8>,           // Which validators signed
}

pub trait ConsensusEngine {
    /// Called when this node is the leader for the current round
    fn propose_block(&mut self, txs: Vec<SignedTransaction>) -> Block;

    /// Called when receiving a proposal from the leader
    fn on_proposal(&mut self, block: Block) -> Option<BlsSignature>;

    /// Called when enough votes are collected (2f+1)
    fn on_quorum(&mut self, qc: QuorumCertificate);

    /// Check if a block has reached finality (3-chain committed)
    fn is_finalized(&self, hash: &BlockHash) -> bool;

    /// Leader rotation: round-robin weighted by stake
    fn get_leader(&self, height: BlockHeight) -> AccountAddress;

    /// View change: triggered by timeout if leader is unresponsive
    fn on_timeout(&mut self, height: BlockHeight) -> TimeoutCertificate;
}

pub trait ValidatorSet {
    fn validators(&self) -> &[ValidatorInfo];
    fn total_stake(&self) -> Amount;
    fn is_validator(&self, addr: &AccountAddress) -> bool;
    fn voting_power(&self, addr: &AccountAddress) -> u64; // Proportional to stake
}

pub struct ValidatorInfo {
    pub address: AccountAddress,
    pub bls_public_key: BlsPublicKey,
    pub stake: Amount,
    pub commission_rate: u16,  // Basis points (e.g., 500 = 5%)
}
```

### Parameters

```
Block time target:        1-2 seconds
Validators per epoch:     100-300 (weighted by stake)
Epoch length:             10,000 blocks (~3-6 hours)
Finality:                 3 blocks (~3-6 seconds)
Timeout base:             5 seconds (exponential backoff)
Min stake:                governance-configurable
Unbonding period:         14 days (604,800 seconds)
Slashing (double-sign):   5% of stake
Slashing (downtime):      0.1% per missed epoch
```

---

## Module 3: State Storage (`nova-store`)

**Purpose:** Merkle Patricia Trie over RocksDB for authenticated state.

### Interface

```rust
pub trait StateDB {
    /// Get account state
    fn get_account(&self, addr: &AccountAddress) -> Option<Account>;

    /// Apply a batch of state changes atomically
    fn apply_changeset(&mut self, changes: StateChangeset) -> StateRoot;

    /// Get the current state root
    fn state_root(&self) -> StateRoot;

    /// Generate a merkle proof for a key
    fn prove(&self, key: &[u8]) -> MerkleProof;

    /// Snapshot for read-only access at a specific block
    fn snapshot_at(&self, height: BlockHeight) -> Box<dyn StateDB>;

    /// Prune old state (keep last N blocks for light client proofs)
    fn prune(&mut self, keep_blocks: u64);
}

pub struct StateChangeset {
    pub account_updates: Vec<(AccountAddress, AccountUpdate)>,
    pub storage_updates: Vec<(AccountAddress, Vec<(Vec<u8>, Vec<u8>)>)>,
}

pub enum AccountUpdate {
    Set(Account),
    UpdateBalance { token: TokenId, new_balance: Amount },
    IncrementNonce,
    Delete,
}
```

### Design Decisions

- **Merkle Patricia Trie** (not plain Merkle) — efficient sparse key storage
- **RocksDB** backend — proven at scale (used by Ethereum, Solana, CockroachDB)
- **Versioned state** — keep state roots per block for light client proofs
- **Lazy pruning** — archive nodes keep everything; full nodes prune after N blocks

### Dependencies

```toml
rocksdb = "0.22"           # Storage backend
```

---

## Module 4: Execution Engine (`nova-exec`)

**Purpose:** Execute transactions, update state, collect fees.

### Interface

```rust
pub struct ExecutionResult {
    pub changeset: StateChangeset,
    pub receipt: TransactionReceipt,
    pub events: Vec<Event>,
}

pub struct TransactionReceipt {
    pub tx_hash: TransactionHash,
    pub status: ExecutionStatus,
    pub fee_charged: Amount,
    pub gas_used: u64,              // For contract calls
    pub events: Vec<Event>,
}

pub enum ExecutionStatus {
    Success,
    Failed(String),                 // Reason; fee still charged
}

pub struct Event {
    pub emitter: AccountAddress,
    pub event_type: String,
    pub data: Vec<u8>,
}

pub trait Executor {
    /// Execute a single transaction against current state
    fn execute(
        &self,
        state: &dyn StateDB,
        tx: &SignedTransaction,
    ) -> ExecutionResult;

    /// Execute a full block (ordered transactions)
    fn execute_block(
        &self,
        state: &dyn StateDB,
        block: &Block,
    ) -> (StateRoot, Vec<TransactionReceipt>);
}
```

### Native Transaction Handlers

Each `TransactionPayload` variant maps to a native handler:

| Payload | Handler | Logic |
|---------|---------|-------|
| `Transfer` | `transfer_handler` | Debit sender, credit recipient, support multi-token |
| `CreateToken` | `token_handler` | Mint supply to creator, register token metadata |
| `DexPlaceOrder` | `dex_handler` | Match against order book, settle fills, queue remainder |
| `DexCancelOrder` | `dex_handler` | Return locked funds to owner |
| `EscrowCreate` | `escrow_handler` | Lock funds with unlock conditions |
| `EscrowFinish` | `escrow_handler` | Release funds to recipient if conditions met |
| `StakeDelegate` | `staking_handler` | Lock tokens, update validator's delegated stake |
| `GovernancePropose` | `gov_handler` | Create proposal with voting period |
| `GovernanceVote` | `gov_handler` | Record vote, execute if threshold met |
| `ContractDeploy` | `vm_handler` | Verify bytecode, store in account |
| `ContractCall` | `vm_handler` | Load contract, execute in Move VM sandbox |

### Fee Model

```
base_fee = dynamic (EIP-1559 style, adjusts per block based on utilization)
execution_fee = gas_used * gas_price (for contract calls only)
total_fee = base_fee + execution_fee

Distribution:
  50% burned (deflationary)
  50% to block proposer
```

---

## Module 5: Networking (`nova-net`)

**Purpose:** P2P networking using libp2p for gossip, sync, and discovery.

### Interface

```rust
pub trait Network {
    /// Broadcast a signed transaction to peers
    fn broadcast_transaction(&self, tx: SignedTransaction);

    /// Broadcast a block proposal (leader only)
    fn broadcast_block(&self, block: Block);

    /// Send a consensus vote to the next leader
    fn send_vote(&self, leader: AccountAddress, vote: ConsensusVote);

    /// Subscribe to incoming messages
    fn subscribe(&self) -> Receiver<NetworkMessage>;

    /// Request blocks from peers for sync
    fn request_blocks(&self, from: BlockHeight, to: BlockHeight) -> Vec<Block>;

    /// Get connected peer count
    fn peer_count(&self) -> usize;
}

pub enum NetworkMessage {
    Transaction(SignedTransaction),
    BlockProposal(Block),
    ConsensusVote(ConsensusVote),
    SyncRequest { from: BlockHeight, to: BlockHeight },
    SyncResponse { blocks: Vec<Block> },
}
```

### Protocols

| Protocol | Purpose | libp2p Component |
|----------|---------|------------------|
| Gossip | Tx & block propagation | `gossipsub` |
| Sync | Block download for new/behind nodes | `request-response` |
| Discovery | Find peers | `kademlia` + `mdns` (local) |
| Identity | Node authentication | `noise` (encrypted transport) |
| Multiplexing | Multiple streams per connection | `yamux` |

### Dependencies

```toml
libp2p = { version = "0.54", features = [
    "gossipsub", "kademlia", "mdns", "noise",
    "yamux", "tcp", "quic", "dns", "identify",
    "request-response"
]}
tokio = { version = "1", features = ["full"] }
```

---

## Module 6: RPC & Client API (`nova-rpc`)

**Purpose:** JSON-RPC 2.0 API for wallets, explorers, and SDKs.

### Endpoints

```
// --- Account ---
nova_getAccount(address) -> Account
nova_getBalance(address, token_id?) -> Amount
nova_getNonce(address) -> Nonce

// --- Transactions ---
nova_submitTransaction(signed_tx) -> TransactionHash
nova_getTransaction(hash) -> SignedTransaction + Receipt
nova_estimateFee(tx) -> Amount
nova_simulateTransaction(tx) -> ExecutionResult  // Dry run

// --- Blocks ---
nova_getBlock(height | "latest") -> Block
nova_getBlockHeader(height | "latest") -> BlockHeader
nova_getLatestHeight() -> BlockHeight

// --- DEX ---
nova_getOrderBook(pair) -> OrderBook
nova_getTradeHistory(pair, limit?) -> Vec<Trade>
nova_getAmmPool(pair) -> PoolState

// --- Staking ---
nova_getValidators() -> Vec<ValidatorInfo>
nova_getDelegations(address) -> Vec<Delegation>
nova_getStakingRewards(address) -> Amount

// --- Governance ---
nova_getProposals(status?) -> Vec<Proposal>
nova_getProposal(id) -> Proposal + VoteTally

// --- Subscriptions (WebSocket) ---
nova_subscribe("newBlocks") -> Stream<Block>
nova_subscribe("newTransactions") -> Stream<SignedTransaction>
nova_subscribe("account", address) -> Stream<AccountEvent>
nova_subscribe("dex", pair) -> Stream<DexEvent>
```

### Dependencies

```toml
jsonrpsee = "0.24"         # JSON-RPC server (HTTP + WebSocket)
axum = "0.8"               # REST API (for explorer/health endpoints)
tower = "0.5"              # Middleware (rate limiting, auth)
```

---

## Module 7: Native DEX (`nova-dex`)

**Purpose:** Built-in decentralized exchange with central limit order book + AMM.

### Design

```rust
pub struct OrderBook {
    pub pair: TradingPair,
    pub bids: BTreeMap<Price, VecDeque<Order>>,  // Sorted descending
    pub asks: BTreeMap<Price, VecDeque<Order>>,  // Sorted ascending
}

pub struct TradingPair {
    pub base: TokenId,
    pub quote: TokenId,
}

pub struct Order {
    pub id: u64,
    pub owner: AccountAddress,
    pub side: Side,
    pub price: Amount,         // In quote token units
    pub quantity: Amount,      // In base token units
    pub filled: Amount,
    pub timestamp: Timestamp,
    pub order_type: OrderType,
}

pub enum Side { Buy, Sell }

pub enum OrderType {
    Limit,                     // Rests on book until filled or cancelled
    ImmediateOrCancel,         // Fill what you can, cancel remainder
    FillOrKill,                // Fill entirely or reject
}

pub struct AmmPool {
    pub pair: TradingPair,
    pub reserve_base: Amount,
    pub reserve_quote: Amount,
    pub lp_token: TokenId,     // Liquidity provider shares
    pub fee_bps: u16,          // e.g., 30 = 0.3%
    pub total_lp_supply: Amount,
}

pub trait DexEngine {
    /// Place a limit order — matches against book, remainder rests
    fn place_order(&mut self, order: Order) -> Vec<Fill>;

    /// Cancel an open order — return locked funds
    fn cancel_order(&mut self, order_id: u64, owner: &AccountAddress) -> Result<()>;

    /// AMM swap — constant product (x * y = k)
    fn amm_swap(&mut self, pool: &TradingPair, input: Amount, side: Side) -> Amount;

    /// Add liquidity to AMM pool
    fn add_liquidity(&mut self, pool: &TradingPair, base: Amount, quote: Amount) -> Amount;

    /// Remove liquidity from AMM pool
    fn remove_liquidity(&mut self, pool: &TradingPair, lp_amount: Amount) -> (Amount, Amount);
}
```

### Why Both CLOB and AMM

- **CLOB** (like XRP): Better for high-value, precise trading. Institutional traders need limit orders.
- **AMM** (like Uniswap): Better for long-tail pairs with lower liquidity. Permissionless pool creation.
- **Hybrid routing**: Smart order routing checks both venues for best execution.

---

## Module 8: Escrow & Payment Channels (`nova-escrow`)

### Escrow (inspired by XRP)

```rust
pub struct Escrow {
    pub id: u64,
    pub sender: AccountAddress,
    pub recipient: AccountAddress,
    pub amount: Amount,
    pub token: TokenId,
    pub condition: EscrowCondition,
    pub created_at: Timestamp,
}

pub enum EscrowCondition {
    /// Release after timestamp
    TimeLock { unlock_time: Timestamp },

    /// Release when cryptographic condition is met (hash preimage)
    CryptoCondition { hash: [u8; 32] },

    /// Release after time OR crypto condition (whichever first)
    Combined {
        unlock_time: Timestamp,
        hash: [u8; 32],
    },

    /// Multi-party: release when M of N parties approve
    MultiParty {
        approvers: Vec<AccountAddress>,
        threshold: u8,
    },

    /// Cancellable after expiry if not claimed
    Cancellable {
        unlock_time: Timestamp,
        cancel_after: Timestamp,
    },
}
```

### Payment Channels (for micropayments)

```rust
pub struct PaymentChannel {
    pub id: u64,
    pub sender: AccountAddress,
    pub recipient: AccountAddress,
    pub deposit: Amount,
    pub balance_sender: Amount,    // Remaining sender balance
    pub balance_recipient: Amount, // Accumulated recipient balance
    pub sequence: u64,             // Monotonically increasing
    pub expiry: Timestamp,         // Channel close deadline
}

// Off-chain: sender signs incremental payment updates
// On-chain: only open + close (or dispute) touch the ledger
pub struct ChannelUpdate {
    pub channel_id: u64,
    pub sequence: u64,
    pub balance_sender: Amount,
    pub balance_recipient: Amount,
    pub signature: Signature,
}
```

---

## Module 9: Governance (`nova-gov`)

```rust
pub struct Proposal {
    pub id: u64,
    pub proposer: AccountAddress,
    pub title: String,
    pub description: String,
    pub proposal_type: ProposalType,
    pub voting_start: Timestamp,
    pub voting_end: Timestamp,
    pub status: ProposalStatus,
}

pub enum ProposalType {
    /// Change a runtime parameter (fees, block size, etc.)
    ParameterChange { key: String, value: Vec<u8> },

    /// Upgrade runtime Wasm blob
    RuntimeUpgrade { wasm_hash: [u8; 32], wasm: Vec<u8> },

    /// Treasury spend
    TreasurySpend { recipient: AccountAddress, amount: Amount },

    /// Free-form text proposal
    TextProposal,
}

pub enum Vote { Yes, No, Abstain }

pub struct VoteTally {
    pub yes: Amount,        // Stake-weighted
    pub no: Amount,
    pub abstain: Amount,
    pub turnout: Amount,    // Total voting power that participated
}

// Thresholds
// Parameter changes:  >50% yes, >33% turnout
// Runtime upgrades:   >66% yes, >50% turnout
// Treasury spends:    >60% yes, >40% turnout
// Text proposals:     >50% yes, >25% turnout
```

---

## Agent Work Streams

These modules have clean boundaries. Each can be developed in parallel once
the shared types in Module 1 are defined.

```
Agent 1: nova-crypto (Module 1)      ← START HERE (shared types, ~1 session)
   │
   ├─► Agent 2: nova-consensus (Module 2)    — consensus engine
   ├─► Agent 3: nova-store (Module 3)        — state storage
   ├─► Agent 4: nova-exec (Module 4)         — execution engine
   ├─► Agent 5: nova-net (Module 5)          — P2P networking
   ├─► Agent 6: nova-rpc (Module 6)          — client API
   ├─► Agent 7: nova-dex (Module 7)          — DEX engine
   ├─► Agent 8: nova-escrow (Module 8)       — escrow & channels
   └─► Agent 9: nova-gov (Module 9)          — governance

Integration: wire modules together in `nova-node` binary
Testing: each module has unit tests; integration tests at `nova-node` level
```

### Dependency Graph

```
nova-crypto ─────────────────────────────────────── (no deps, build first)
     │
     ├── nova-store ──────────────────────────────── (depends on crypto)
     │       │
     ├── nova-exec ───────────────────────────────── (depends on crypto, store)
     │       │
     ├── nova-consensus ──────────────────────────── (depends on crypto, store)
     │       │
     ├── nova-net ────────────────────────────────── (depends on crypto)
     │       │
     ├── nova-dex ────────────────────────────────── (depends on crypto, store)
     │       │
     ├── nova-escrow ─────────────────────────────── (depends on crypto, store)
     │       │
     ├── nova-gov ────────────────────────────────── (depends on crypto, store)
     │       │
     └── nova-rpc ────────────────────────────────── (depends on all above)

nova-node (binary) ───────────────────────────────── (wires everything together)
```

---

## Getting Started: Repository Bootstrap

```bash
# Create the Rust workspace
cargo init --name nova-node
mkdir -p crates/{nova-crypto,nova-consensus,nova-store,nova-exec,nova-net,nova-rpc,nova-dex,nova-escrow,nova-gov}

# Workspace Cargo.toml
[workspace]
members = [
    "crates/nova-crypto",
    "crates/nova-consensus",
    "crates/nova-store",
    "crates/nova-exec",
    "crates/nova-net",
    "crates/nova-rpc",
    "crates/nova-dex",
    "crates/nova-escrow",
    "crates/nova-gov",
]
resolver = "2"

[workspace.dependencies]
ed25519-dalek = "2"
bls12_381 = "0.8"
blake3 = "1"
sha2 = "0.10"
borsh = "1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
thiserror = "2"
tracing = "0.1"
```
