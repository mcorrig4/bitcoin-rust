# Bitcoin Core Detailed Flow Diagrams

This document contains specialized diagrams showing detailed flows, state machines, and architectural patterns in Bitcoin Core.

## Table of Contents
1. [Block Validation State Machine](#1-block-validation-state-machine)
2. [Transaction Lifecycle](#2-transaction-lifecycle)
3. [Chain Reorganization Process](#3-chain-reorganization-process)
4. [UTXO Cache Hierarchy](#4-utxo-cache-hierarchy)
5. [Network Message Flow](#5-network-message-flow)
6. [Fee Estimation Algorithm](#6-fee-estimation-algorithm)
7. [Script Validation Pipeline](#7-script-validation-pipeline)
8. [Wallet Key Derivation](#8-wallet-key-derivation)
9. [Database Transaction Flow](#9-database-transaction-flow)
10. [Initial Block Download (IBD)](#10-initial-block-download-ibd)
11. [Memory Pool Package Handling](#11-memory-pool-package-handling)
12. [Consensus Fork Resolution](#12-consensus-fork-resolution)

---

## 1. Block Validation State Machine

```mermaid
stateDiagram-v2
    [*] --> BLOCK_RECEIVED: New Block
    
    BLOCK_RECEIVED --> HEADER_VALID: CheckBlockHeader()
    BLOCK_RECEIVED --> REJECTED: Invalid Header
    
    HEADER_VALID --> TRANSACTIONS_VALID: CheckBlock()
    HEADER_VALID --> REJECTED: Invalid Structure
    
    TRANSACTIONS_VALID --> CONTEXT_VALID: ContextualCheckBlock()
    TRANSACTIONS_VALID --> REJECTED: Invalid TX
    
    CONTEXT_VALID --> CONNECTING: ConnectBlock()
    CONTEXT_VALID --> REJECTED: Bad Context
    
    CONNECTING --> SCRIPTS_VALID: Script Validation
    CONNECTING --> DISCONNECTED: Script Failure
    
    SCRIPTS_VALID --> CONNECTED: Update UTXO
    SCRIPTS_VALID --> DISCONNECTED: UTXO Error
    
    CONNECTED --> ACTIVE_CHAIN: ActivateBestChain()
    CONNECTED --> VALID_FORK: Lower Work
    
    ACTIVE_CHAIN --> [*]: Success
    VALID_FORK --> [*]: Stored
    REJECTED --> [*]: Discarded
    DISCONNECTED --> [*]: Rolled Back
    
    note right of SCRIPTS_VALID
        Parallel validation
        across N CPU cores
    end note
    
    note left of CONTEXT_VALID
        Check timestamp,
        difficulty, size
    end note
```

---

## 2. Transaction Lifecycle

```mermaid
journey
    title Transaction Lifecycle in Bitcoin Core
    
    section Creation
      User creates TX: 5: User, Wallet
      Select coins: 3: Wallet
      Sign inputs: 4: Wallet
      
    section Propagation  
      Submit to mempool: 5: Wallet, Mempool
      Validate TX: 3: Mempool
      Relay to peers: 4: Network
      
    section Mining
      Select for block: 5: Miner
      Include in template: 4: Miner
      Mine block: 2: Miner
      
    section Confirmation
      Block propagated: 5: Network
      Block validated: 4: Node
      TX confirmed: 5: Wallet, User
      
    section Finalization
      6 confirmations: 5: Blockchain
      Spend outputs: 5: User
```

---

## 3. Chain Reorganization Process

```mermaid
flowchart TB
    subgraph "Detection Phase"
        NEW_TIP[New Competing Tip]
        COMPARE_WORK[Compare Chain Work]
        MORE_WORK{More Work?}
    end
    
    subgraph "Disconnection Phase"
        FIND_FORK[Find Fork Point]
        DISCONNECT_BLOCKS[Disconnect Blocks]
        UNDO_UTXO[Restore UTXO State]
        RETURN_TO_MEMPOOL[Return TXs to Mempool]
    end
    
    subgraph "Connection Phase"
        CONNECT_NEW[Connect New Blocks]
        VALIDATE_SCRIPTS[Validate All Scripts]
        UPDATE_UTXO[Update UTXO Set]
        REMOVE_FROM_MEMPOOL[Remove Confirmed TXs]
    end
    
    subgraph "Finalization"
        UPDATE_TIP[Update Chain Tip]
        NOTIFY[Notify Subscribers]
        UPDATE_INDEXES[Update Indexes]
        FLUSH_STATE[Flush to Disk]
    end
    
    NEW_TIP --> COMPARE_WORK
    COMPARE_WORK --> MORE_WORK
    MORE_WORK -->|No| KEEP_CURRENT[Keep Current Chain]
    MORE_WORK -->|Yes| FIND_FORK
    
    FIND_FORK --> DISCONNECT_BLOCKS
    DISCONNECT_BLOCKS --> UNDO_UTXO
    UNDO_UTXO --> RETURN_TO_MEMPOOL
    
    RETURN_TO_MEMPOOL --> CONNECT_NEW
    CONNECT_NEW --> VALIDATE_SCRIPTS
    VALIDATE_SCRIPTS --> UPDATE_UTXO
    UPDATE_UTXO --> REMOVE_FROM_MEMPOOL
    
    REMOVE_FROM_MEMPOOL --> UPDATE_TIP
    UPDATE_TIP --> NOTIFY
    NOTIFY --> UPDATE_INDEXES
    UPDATE_INDEXES --> FLUSH_STATE
    
    style MORE_WORK fill:#ff9,stroke:#333,stroke-width:4px
    style VALIDATE_SCRIPTS fill:#f99,stroke:#333,stroke-width:2px
```

---

## 4. UTXO Cache Hierarchy

```mermaid
graph TD
    subgraph "Memory Layer"
        L1[L1: CCoinsViewCache<br/>~300MB<br/>Most Recent]
        L2[L2: CCoinsViewCache<br/>~1GB<br/>Backing Cache]
    end
    
    subgraph "Database Layer"
        L3[L3: CCoinsViewDB<br/>LevelDB Cache<br/>~450MB]
        DISK[(Disk Storage<br/>~5GB UTXO Set)]
    end
    
    subgraph "Access Patterns"
        READ[Read Request]
        WRITE[Write Request]
        FLUSH[Periodic Flush]
    end
    
    READ --> L1
    L1 -->|Miss| L2
    L2 -->|Miss| L3
    L3 -->|Miss| DISK
    
    WRITE --> L1
    L1 -->|Full| FLUSH
    FLUSH --> L2
    L2 -->|Full| L3
    L3 -->|Batch| DISK
    
    style L1 fill:#9ff,stroke:#333,stroke-width:4px
    style DISK fill:#ff9,stroke:#333,stroke-width:2px
```

---

## 5. Network Message Flow

```mermaid
sequenceDiagram
    participant Peer
    participant SocketHandler
    participant MessageHandler
    participant PeerManager
    participant Validation
    participant Mempool
    
    Peer->>SocketHandler: TCP Data
    SocketHandler->>SocketHandler: Read into Buffer
    SocketHandler->>MessageHandler: Complete Message
    
    MessageHandler->>MessageHandler: Deserialize Header
    MessageHandler->>MessageHandler: Verify Checksum
    MessageHandler->>PeerManager: Process Message
    
    alt Block Message
        PeerManager->>Validation: ProcessNewBlock()
        Validation->>Validation: CheckBlock()
        Validation->>Validation: ConnectBlock()
        Validation-->>PeerManager: Result
    else Transaction Message
        PeerManager->>Mempool: AcceptToMemoryPool()
        Mempool->>Mempool: CheckTransaction()
        Mempool-->>PeerManager: Result
    else Addr Message
        PeerManager->>PeerManager: UpdateAddrMan()
    end
    
    PeerManager-->>Peer: Response/Relay
    
    Note over SocketHandler: Async I/O Thread
    Note over MessageHandler: Message Thread
    Note over Validation: Main Thread
```

---

## 6. Fee Estimation Algorithm

```mermaid
flowchart LR
    subgraph "Data Collection"
        NEW_TX[New Transaction]
        FEE_RATE[Calculate Fee Rate]
        BUCKET[Assign to Bucket]
        TRACK[Track Entry Height]
    end
    
    subgraph "Confirmation Tracking"
        CONFIRMED[TX Confirmed]
        BLOCKS_WAITED[Blocks Waited]
        UPDATE_STATS[Update Statistics]
    end
    
    subgraph "Estimation Process"
        REQUEST[Estimate Request]
        TARGET_BLOCKS[Target Blocks]
        CONFIDENCE[Confidence Level]
        HISTORICAL[Historical Data]
        CALCULATE[Calculate Rate]
    end
    
    subgraph "Fee Buckets"
        B1[1-2 sat/vB]
        B2[2-3 sat/vB]
        B3[3-5 sat/vB]
        B4[5-10 sat/vB]
        B5[10+ sat/vB]
    end
    
    NEW_TX --> FEE_RATE
    FEE_RATE --> BUCKET
    BUCKET --> TRACK
    
    TRACK --> B1
    TRACK --> B2
    TRACK --> B3
    TRACK --> B4
    TRACK --> B5
    
    CONFIRMED --> BLOCKS_WAITED
    BLOCKS_WAITED --> UPDATE_STATS
    UPDATE_STATS --> HISTORICAL
    
    REQUEST --> TARGET_BLOCKS
    TARGET_BLOCKS --> CONFIDENCE
    CONFIDENCE --> HISTORICAL
    HISTORICAL --> CALCULATE
    
    style CALCULATE fill:#9ff,stroke:#333,stroke-width:4px
```

---

## 7. Script Validation Pipeline

```mermaid
flowchart TD
    subgraph "Input Preparation"
        TX_INPUT[Transaction Input]
        PREV_OUT[Fetch Previous Output]
        SCRIPT_PAIR[ScriptSig + ScriptPubKey]
    end
    
    subgraph "Execution Stack"
        SCRIPT_SIG_EXEC[Execute ScriptSig]
        STACK_COPY[Copy Stack]
        SCRIPT_PUBKEY_EXEC[Execute ScriptPubKey]
        CHECK_RESULT[Check Stack Result]
    end
    
    subgraph "Witness Validation"
        IS_WITNESS{Is Witness?}
        WITNESS_PROG[Execute Witness Program]
        WITNESS_STACK[Witness Stack Execution]
    end
    
    subgraph "Special Cases"
        P2SH{Is P2SH?}
        REDEEM_SCRIPT[Execute Redeem Script]
        TAPROOT{Is Taproot?}
        TAPSCRIPT[Execute Tapscript]
    end
    
    TX_INPUT --> PREV_OUT
    PREV_OUT --> SCRIPT_PAIR
    
    SCRIPT_PAIR --> SCRIPT_SIG_EXEC
    SCRIPT_SIG_EXEC --> STACK_COPY
    STACK_COPY --> SCRIPT_PUBKEY_EXEC
    SCRIPT_PUBKEY_EXEC --> CHECK_RESULT
    
    CHECK_RESULT --> IS_WITNESS
    IS_WITNESS -->|Yes| WITNESS_PROG
    WITNESS_PROG --> WITNESS_STACK
    
    CHECK_RESULT --> P2SH
    P2SH -->|Yes| REDEEM_SCRIPT
    
    WITNESS_STACK --> TAPROOT
    TAPROOT -->|Yes| TAPSCRIPT
    
    style CHECK_RESULT fill:#ff9,stroke:#333,stroke-width:4px
    style WITNESS_STACK fill:#9ff,stroke:#333,stroke-width:2px
```

---

## 8. Wallet Key Derivation

```mermaid
graph TD
    subgraph "HD Wallet Structure"
        SEED[Master Seed<br/>256 bits]
        MASTER[Master Key<br/>m]
        PURPOSE[Purpose<br/>m/84']
        COIN[Coin Type<br/>m/84'/0']
        ACCOUNT[Account<br/>m/84'/0'/0']
        CHANGE[Change<br/>m/84'/0'/0'/0]
        ADDRESS[Address<br/>m/84'/0'/0'/0/0]
    end
    
    subgraph "Key Types"
        EXTERNAL[External Chain<br/>Receiving]
        INTERNAL[Internal Chain<br/>Change]
    end
    
    subgraph "Address Generation"
        PUBKEY[Public Key]
        WITNESS[Witness Program]
        BECH32[Bech32 Address]
    end
    
    SEED -->|HMAC-SHA512| MASTER
    MASTER -->|Hardened| PURPOSE
    PURPOSE -->|Hardened| COIN
    COIN -->|Hardened| ACCOUNT
    
    ACCOUNT --> EXTERNAL
    ACCOUNT --> INTERNAL
    
    EXTERNAL --> CHANGE
    INTERNAL --> CHANGE
    
    CHANGE -->|Normal| ADDRESS
    ADDRESS --> PUBKEY
    PUBKEY --> WITNESS
    WITNESS --> BECH32
    
    style SEED fill:#f99,stroke:#333,stroke-width:4px
    style BECH32 fill:#9f9,stroke:#333,stroke-width:2px
```

---

## 9. Database Transaction Flow

```mermaid
flowchart LR
    subgraph "Write Operation"
        BEGIN[Begin Batch]
        WRITE1[Write Op 1]
        WRITE2[Write Op 2]
        WRITEN[Write Op N]
        COMMIT[Commit Batch]
    end
    
    subgraph "LevelDB Layer"
        MEMTABLE[MemTable<br/>In-Memory]
        IMMUTABLE[Immutable<br/>MemTable]
        L0[Level 0<br/>SST Files]
        L1[Level 1<br/>SST Files]
        LN[Level N<br/>SST Files]
    end
    
    subgraph "Compaction"
        COMPACT[Background<br/>Compaction]
        MERGE[Merge SSTables]
        DELETE[Delete Old Files]
    end
    
    BEGIN --> WRITE1
    WRITE1 --> WRITE2
    WRITE2 --> WRITEN
    WRITEN --> COMMIT
    
    COMMIT --> MEMTABLE
    MEMTABLE -->|Full| IMMUTABLE
    IMMUTABLE -->|Flush| L0
    
    L0 --> COMPACT
    L1 --> COMPACT
    COMPACT --> MERGE
    MERGE --> LN
    MERGE --> DELETE
    
    style COMMIT fill:#ff9,stroke:#333,stroke-width:4px
    style COMPACT fill:#9ff,stroke:#333,stroke-width:2px
```

---

## 10. Initial Block Download (IBD)

```mermaid
stateDiagram-v2
    [*] --> FIND_PEERS: Start Node
    
    FIND_PEERS --> CONNECT_PEERS: DNS Seeds
    CONNECT_PEERS --> GET_HEADERS: Version Handshake
    
    GET_HEADERS --> VALIDATE_HEADERS: Receive Headers
    VALIDATE_HEADERS --> REQUEST_BLOCKS: Valid Chain
    
    REQUEST_BLOCKS --> DOWNLOAD_WINDOW: Parallel Requests
    DOWNLOAD_WINDOW --> RECEIVE_BLOCKS: Block Data
    
    RECEIVE_BLOCKS --> VALIDATE_BLOCKS: Full Validation
    VALIDATE_BLOCKS --> CONNECT_BLOCKS: Valid
    VALIDATE_BLOCKS --> REQUEST_BLOCKS: Invalid/Timeout
    
    CONNECT_BLOCKS --> UPDATE_INDEXES: Update State
    UPDATE_INDEXES --> CHECK_SYNC: Check Progress
    
    CHECK_SYNC --> REQUEST_BLOCKS: Not Synced
    CHECK_SYNC --> NORMAL_OPERATION: Fully Synced
    
    NORMAL_OPERATION --> [*]: IBD Complete
    
    note right of DOWNLOAD_WINDOW
        1024 blocks in flight
        from multiple peers
    end note
    
    note left of VALIDATE_BLOCKS
        Full consensus validation
        including all scripts
    end note
```

---

## 11. Memory Pool Package Handling

```mermaid
flowchart TD
    subgraph "Package Reception"
        PKG_RECEIVE[Receive Package]
        PKG_SORT[Topological Sort]
        PKG_VALIDATE[Validate Package]
    end
    
    subgraph "Package Validation"
        CHECK_SIZE[Check Package Size]
        CHECK_ANCESTORS[Check Ancestors]
        CHECK_DEPS[Verify Dependencies]
        CHECK_FEES[Calculate Package Fees]
    end
    
    subgraph "Individual Validation"
        FOREACH_TX[For Each TX]
        VALIDATE_TX[Validate Transaction]
        CHECK_CONFLICTS[Check Conflicts]
        POLICY_CHECK[Policy Checks]
    end
    
    subgraph "Package Acceptance"
        ACCEPT_PKG[Accept Package]
        UPDATE_MEMPOOL[Update Mempool]
        UPDATE_FEES[Update Fee Stats]
        RELAY_PKG[Relay Package]
    end
    
    PKG_RECEIVE --> PKG_SORT
    PKG_SORT --> PKG_VALIDATE
    
    PKG_VALIDATE --> CHECK_SIZE
    CHECK_SIZE -->|Pass| CHECK_ANCESTORS
    CHECK_SIZE -->|Fail| REJECT_PKG[Reject Package]
    
    CHECK_ANCESTORS --> CHECK_DEPS
    CHECK_DEPS --> CHECK_FEES
    
    CHECK_FEES --> FOREACH_TX
    FOREACH_TX --> VALIDATE_TX
    VALIDATE_TX --> CHECK_CONFLICTS
    CHECK_CONFLICTS --> POLICY_CHECK
    
    POLICY_CHECK -->|All Pass| ACCEPT_PKG
    POLICY_CHECK -->|Any Fail| REJECT_PKG
    
    ACCEPT_PKG --> UPDATE_MEMPOOL
    UPDATE_MEMPOOL --> UPDATE_FEES
    UPDATE_FEES --> RELAY_PKG
    
    style CHECK_FEES fill:#9ff,stroke:#333,stroke-width:4px
    style ACCEPT_PKG fill:#9f9,stroke:#333,stroke-width:2px
```

---

## 12. Consensus Fork Resolution

```mermaid
flowchart TB
    subgraph "Fork Detection"
        BLOCK_A[Block at Height N<br/>Chain A]
        BLOCK_B[Block at Height N<br/>Chain B]
        FORK_DETECTED[Fork Detected]
    end
    
    subgraph "Chain Evaluation"
        WORK_A[Calculate Work A]
        WORK_B[Calculate Work B]
        COMPARE[Compare Total Work]
        CHECK_VALID[Validate Blocks]
    end
    
    subgraph "Resolution Process"
        SELECT_CHAIN{Select Chain}
        REORG_TO_A[Reorganize to A]
        REORG_TO_B[Reorganize to B]
        STAY_CURRENT[Keep Current]
    end
    
    subgraph "Network Consensus"
        MONITOR_PEERS[Monitor Peer Tips]
        MAJORITY{Majority on Same?}
        CONVERGED[Network Converged]
        ISOLATED[Node Isolated]
    end
    
    BLOCK_A --> FORK_DETECTED
    BLOCK_B --> FORK_DETECTED
    
    FORK_DETECTED --> WORK_A
    FORK_DETECTED --> WORK_B
    
    WORK_A --> COMPARE
    WORK_B --> COMPARE
    COMPARE --> CHECK_VALID
    
    CHECK_VALID --> SELECT_CHAIN
    SELECT_CHAIN -->|A Wins| REORG_TO_A
    SELECT_CHAIN -->|B Wins| REORG_TO_B
    SELECT_CHAIN -->|Equal| STAY_CURRENT
    
    REORG_TO_A --> MONITOR_PEERS
    REORG_TO_B --> MONITOR_PEERS
    STAY_CURRENT --> MONITOR_PEERS
    
    MONITOR_PEERS --> MAJORITY
    MAJORITY -->|Yes| CONVERGED
    MAJORITY -->|No| ISOLATED
    
    style COMPARE fill:#ff9,stroke:#333,stroke-width:4px
    style CONVERGED fill:#9f9,stroke:#333,stroke-width:2px
    style ISOLATED fill:#f99,stroke:#333,stroke-width:2px
```

---

## Integration Points and Data Flow Summary

```mermaid
graph TB
    subgraph "External Interfaces"
        NET_IN[Network Input]
        USER_IN[User Input]
        DISK_IN[Disk Storage]
    end
    
    subgraph "Core Processing"
        VALIDATE[Validation Core]
        MEMPOOL[Mempool Manager]
        CHAIN[Chain Manager]
        WALLET[Wallet Manager]
    end
    
    subgraph "Internal State"
        UTXO_SET[UTXO Set]
        CHAIN_STATE[Chain State]
        PEER_STATE[Peer State]
        WALLET_STATE[Wallet State]
    end
    
    subgraph "Output Systems"
        NET_OUT[Network Output]
        DISK_OUT[Disk Writes]
        NOTIFY_OUT[Notifications]
        RPC_OUT[RPC Responses]
    end
    
    %% Input flows
    NET_IN --> VALIDATE
    NET_IN --> MEMPOOL
    USER_IN --> WALLET
    USER_IN --> VALIDATE
    DISK_IN --> CHAIN
    
    %% Core interactions
    VALIDATE <--> MEMPOOL
    VALIDATE <--> CHAIN
    MEMPOOL <--> WALLET
    CHAIN <--> WALLET
    
    %% State updates
    VALIDATE --> UTXO_SET
    CHAIN --> CHAIN_STATE
    NET_IN --> PEER_STATE
    WALLET --> WALLET_STATE
    
    %% Output flows
    MEMPOOL --> NET_OUT
    CHAIN --> DISK_OUT
    VALIDATE --> NOTIFY_OUT
    WALLET --> RPC_OUT
    
    style VALIDATE fill:#ff9,stroke:#333,stroke-width:4px
    style UTXO_SET fill:#9ff,stroke:#333,stroke-width:4px
```

---

## Performance Bottlenecks and Optimization Points

```mermaid
graph LR
    subgraph "CPU Intensive"
        SCRIPT_VAL[Script Validation<br/>70% CPU]
        SIG_VERIFY[Signature Verification<br/>60% CPU]
        HASH_CALC[Hash Calculations<br/>20% CPU]
    end
    
    subgraph "Memory Intensive"
        UTXO_CACHE[UTXO Cache<br/>2-4 GB]
        MEMPOOL_MEM[Mempool<br/>300 MB]
        PEER_BUFFERS[Peer Buffers<br/>100 MB/peer]
    end
    
    subgraph "I/O Intensive"
        BLOCK_READ[Block Reading<br/>100 MB/s]
        LEVELDB_WRITE[Database Writes<br/>50 MB/s]
        NETWORK_IO[Network I/O<br/>10 MB/s]
    end
    
    subgraph "Optimizations"
        PARALLEL_VAL[Parallel Validation]
        CACHE_OPT[Cache Optimization]
        BATCH_IO[Batch I/O Operations]
    end
    
    SCRIPT_VAL --> PARALLEL_VAL
    SIG_VERIFY --> PARALLEL_VAL
    
    UTXO_CACHE --> CACHE_OPT
    MEMPOOL_MEM --> CACHE_OPT
    
    BLOCK_READ --> BATCH_IO
    LEVELDB_WRITE --> BATCH_IO
    
    style SCRIPT_VAL fill:#f00,stroke:#333,stroke-width:4px
    style UTXO_CACHE fill:#f90,stroke:#333,stroke-width:4px
```

These detailed diagrams provide deep insights into:

1. **State machines** - How components transition between states
2. **Sequence flows** - Step-by-step message and data flows
3. **Decision trees** - How the system makes choices
4. **Data structures** - Hierarchical organization of data
5. **Algorithms** - Visual representation of key algorithms
6. **Integration patterns** - How components work together
7. **Performance analysis** - Where bottlenecks occur

Each diagram focuses on a specific aspect of Bitcoin Core's architecture, providing the detail needed to understand both the individual components and how they integrate into the complete system.