# Bitcoin Core System Integration Diagrams

This document provides comprehensive system-level integration diagrams showing how all components work together as a cohesive system.

## Table of Contents
1. [Complete System Data Flow](#1-complete-system-data-flow)
2. [Component Dependency Graph](#2-component-dependency-graph)
3. [Event Propagation System](#3-event-propagation-system)
4. [Layered Architecture View](#4-layered-architecture-view)
5. [Cross-Component Transaction Flow](#5-cross-component-transaction-flow)
6. [System Initialization Sequence](#6-system-initialization-sequence)
7. [Shutdown and Cleanup Flow](#7-shutdown-and-cleanup-flow)
8. [Full Node Operation Lifecycle](#8-full-node-operation-lifecycle)

---

## 1. Complete System Data Flow

```mermaid
graph TB
    subgraph "Input Sources"
        P2P[P2P Network<br/>Blocks & TXs]
        RPC_IN[RPC Commands<br/>User Requests]
        WALLET_IN[Wallet Operations<br/>Send/Receive]
        MINER_IN[Mining Software<br/>getblocktemplate]
    end
    
    subgraph "Primary Processing"
        VALIDATION[Validation Engine<br/>Consensus Rules]
        MEMPOOL[Memory Pool<br/>Pending TXs]
        NETWORKING[Network Manager<br/>Peer Connections]
        WALLET_PROC[Wallet Processor<br/>Key & TX Mgmt]
    end
    
    subgraph "State Management"
        CHAINSTATE[Chain State<br/>Active Chain]
        UTXO_SET[UTXO Set<br/>Unspent Outputs]
        ADDRMAN[Address Manager<br/>Peer Addresses]
        FEE_EST[Fee Estimator<br/>Confirmation Stats]
    end
    
    subgraph "Storage Layer"
        BLOCKS_DB[(Block Storage<br/>blk*.dat)]
        STATE_DB[(State Database<br/>LevelDB)]
        WALLET_DB[(Wallet Database<br/>SQLite)]
        INDEX_DB[(Optional Indexes<br/>LevelDB)]
    end
    
    subgraph "Output Interfaces"
        P2P_OUT[P2P Relay<br/>Block/TX Propagation]
        RPC_OUT[RPC Responses<br/>JSON Results]
        ZMQ_OUT[ZMQ Notifications<br/>Real-time Events]
        GUI_OUT[GUI Updates<br/>User Interface]
    end
    
    %% Input flows
    P2P --> NETWORKING
    P2P --> VALIDATION
    RPC_IN --> WALLET_PROC
    RPC_IN --> MEMPOOL
    RPC_IN --> VALIDATION
    WALLET_IN --> WALLET_PROC
    MINER_IN --> MEMPOOL
    
    %% Core processing
    NETWORKING <--> VALIDATION
    VALIDATION <--> MEMPOOL
    MEMPOOL <--> WALLET_PROC
    VALIDATION <--> CHAINSTATE
    
    %% State updates
    VALIDATION --> UTXO_SET
    NETWORKING --> ADDRMAN
    MEMPOOL --> FEE_EST
    CHAINSTATE --> UTXO_SET
    
    %% Storage operations
    CHAINSTATE --> BLOCKS_DB
    CHAINSTATE --> STATE_DB
    UTXO_SET --> STATE_DB
    WALLET_PROC --> WALLET_DB
    VALIDATION --> INDEX_DB
    
    %% Output flows
    MEMPOOL --> P2P_OUT
    VALIDATION --> P2P_OUT
    WALLET_PROC --> RPC_OUT
    CHAINSTATE --> RPC_OUT
    VALIDATION --> ZMQ_OUT
    WALLET_PROC --> GUI_OUT
    
    style VALIDATION fill:#ff9,stroke:#333,stroke-width:4px
    style MEMPOOL fill:#9f9,stroke:#333,stroke-width:3px
    style UTXO_SET fill:#9ff,stroke:#333,stroke-width:3px
    style STATE_DB fill:#f9f,stroke:#333,stroke-width:2px
```

---

## 2. Component Dependency Graph

```mermaid
graph BT
    subgraph "Core Dependencies"
        CRYPTO[Crypto Libraries<br/>secp256k1, SHA256]
        SERIALIZE[Serialization<br/>Network Format]
        LOGGING[Logging System<br/>Debug Output]
        CONFIG[Configuration<br/>Settings & Args]
    end
    
    subgraph "Base Components"
        PRIMITIVES[Primitives<br/>Block, Transaction]
        SCRIPT[Script System<br/>Script Interpreter]
        CONSENSUS[Consensus Rules<br/>Validation Logic]
        UTIL[Utilities<br/>Time, Random, etc]
    end
    
    subgraph "Node Components"
        CHAIN_COMP[Chain Management<br/>CChain, CBlockIndex]
        NET_COMP[Network Layer<br/>CConnman, CNode]
        MEMPOOL_COMP[Mempool<br/>CTxMemPool]
        STORAGE_COMP[Storage<br/>BlockStorage, UTXO DB]
    end
    
    subgraph "High-Level Services"
        VALIDATION_SVC[Validation Service<br/>Block/TX Processing]
        RPC_SVC[RPC Service<br/>API Interface]
        WALLET_SVC[Wallet Service<br/>Key Management]
        INDEX_SVC[Index Service<br/>Optional Indexes]
    end
    
    subgraph "User Interfaces"
        GUI_APP[GUI Application<br/>bitcoin-qt]
        CLI_APP[CLI Tools<br/>bitcoin-cli]
        DAEMON[Daemon<br/>bitcoind]
    end
    
    %% Base dependencies
    CRYPTO --> PRIMITIVES
    SERIALIZE --> PRIMITIVES
    LOGGING --> UTIL
    CONFIG --> UTIL
    
    %% Component dependencies
    PRIMITIVES --> SCRIPT
    PRIMITIVES --> CONSENSUS
    SCRIPT --> CONSENSUS
    UTIL --> CHAIN_COMP
    UTIL --> NET_COMP
    
    %% Service dependencies
    CONSENSUS --> VALIDATION_SVC
    CHAIN_COMP --> VALIDATION_SVC
    NET_COMP --> VALIDATION_SVC
    MEMPOOL_COMP --> VALIDATION_SVC
    STORAGE_COMP --> VALIDATION_SVC
    
    VALIDATION_SVC --> RPC_SVC
    VALIDATION_SVC --> WALLET_SVC
    VALIDATION_SVC --> INDEX_SVC
    
    %% UI dependencies
    RPC_SVC --> GUI_APP
    RPC_SVC --> CLI_APP
    VALIDATION_SVC --> DAEMON
    WALLET_SVC --> GUI_APP
    
    style VALIDATION_SVC fill:#ff9,stroke:#333,stroke-width:4px
    style CONSENSUS fill:#f99,stroke:#333,stroke-width:3px
    style CRYPTO fill:#9ff,stroke:#333,stroke-width:2px
```

---

## 3. Event Propagation System

```mermaid
sequenceDiagram
    participant Network
    participant Validation
    participant MainSignals
    participant Wallet
    participant Indexes
    participant GUI
    participant Mempool
    
    Note over MainSignals: CMainSignals - Event Hub
    
    Network->>Validation: New Block Received
    Validation->>Validation: Validate Block
    
    alt Block Valid
        Validation->>MainSignals: BlockConnected Signal
        MainSignals->>Wallet: UpdatedBlockTip()
        MainSignals->>Indexes: BlockConnected()
        MainSignals->>GUI: NotifyBlockTip()
        MainSignals->>Mempool: RemoveForBlock()
        
        Wallet->>Wallet: Update Balances
        Indexes->>Indexes: Update Index
        GUI->>GUI: Update Display
        Mempool->>Mempool: Remove Confirmed TXs
    else Block Invalid
        Validation->>MainSignals: BlockDisconnected Signal
        MainSignals->>Wallet: BlockDisconnected()
        MainSignals->>Indexes: BlockDisconnected()
        MainSignals->>Mempool: AddBackToMempool()
    end
    
    Network->>Mempool: New Transaction
    Mempool->>MainSignals: TransactionAddedToMempool
    MainSignals->>Wallet: TransactionAddedToMempool()
    MainSignals->>GUI: NotifyTransactionChanged()
    
    Note over MainSignals: Async signal delivery via boost::signals2
```

---

## 4. Layered Architecture View

```mermaid
graph TB
    subgraph "Layer 6: User Interface"
        GUI[Qt GUI]
        CLI[CLI Tools]
        API[External APIs]
    end
    
    subgraph "Layer 5: Application Services"
        RPC_LAYER[RPC Server]
        REST_LAYER[REST API]
        ZMQ_LAYER[ZMQ Publisher]
    end
    
    subgraph "Layer 4: Business Logic"
        WALLET_LOGIC[Wallet Logic]
        MINING_LOGIC[Mining Logic]
        FEE_LOGIC[Fee Estimation]
        INDEX_LOGIC[Index Management]
    end
    
    subgraph "Layer 3: Core Services"
        VALIDATION_CORE[Validation Core]
        MEMPOOL_CORE[Mempool Management]
        P2P_CORE[P2P Protocol]
        SCRIPT_CORE[Script Engine]
    end
    
    subgraph "Layer 2: Data Management"
        CHAIN_DATA[Chain State]
        UTXO_DATA[UTXO Management]
        PEER_DATA[Peer Management]
        TX_DATA[Transaction Store]
    end
    
    subgraph "Layer 1: Storage & Infrastructure"
        LEVELDB_STORE[LevelDB]
        FILE_STORE[File System]
        NETWORK_STACK[Network Stack]
        CRYPTO_LIB[Crypto Libraries]
    end
    
    %% Layer connections
    GUI --> RPC_LAYER
    CLI --> RPC_LAYER
    API --> REST_LAYER
    
    RPC_LAYER --> WALLET_LOGIC
    RPC_LAYER --> MINING_LOGIC
    REST_LAYER --> VALIDATION_CORE
    ZMQ_LAYER --> VALIDATION_CORE
    
    WALLET_LOGIC --> VALIDATION_CORE
    MINING_LOGIC --> MEMPOOL_CORE
    FEE_LOGIC --> MEMPOOL_CORE
    INDEX_LOGIC --> VALIDATION_CORE
    
    VALIDATION_CORE --> CHAIN_DATA
    MEMPOOL_CORE --> TX_DATA
    P2P_CORE --> PEER_DATA
    SCRIPT_CORE --> VALIDATION_CORE
    
    CHAIN_DATA --> LEVELDB_STORE
    UTXO_DATA --> LEVELDB_STORE
    TX_DATA --> FILE_STORE
    P2P_CORE --> NETWORK_STACK
    SCRIPT_CORE --> CRYPTO_LIB
    
    style VALIDATION_CORE fill:#ff9,stroke:#333,stroke-width:4px
    style MEMPOOL_CORE fill:#9f9,stroke:#333,stroke-width:3px
    style LEVELDB_STORE fill:#9ff,stroke:#333,stroke-width:2px
```

---

## 5. Cross-Component Transaction Flow

```mermaid
graph LR
    subgraph "Transaction Creation"
        USER[User Request]
        WALLET_CREATE[Wallet<br/>Create TX]
        SIGN[Sign TX]
    end
    
    subgraph "Local Validation"
        MEMPOOL_ACCEPT[Mempool<br/>Accept]
        POLICY_CHECK[Policy<br/>Checks]
        SCRIPT_CHECK[Script<br/>Validation]
    end
    
    subgraph "Network Propagation"
        INV_SEND[Send INV]
        PEER_REQUEST[Peer<br/>GETDATA]
        TX_SEND[Send TX]
    end
    
    subgraph "Remote Processing"
        PEER_VALIDATE[Peer<br/>Validates]
        PEER_MEMPOOL[Peer<br/>Mempool]
        PEER_RELAY[Peer<br/>Relays]
    end
    
    subgraph "Mining Inclusion"
        MINER_SELECT[Miner<br/>Selects TX]
        BLOCK_CREATE[Create<br/>Block]
        BLOCK_MINE[Mine<br/>Block]
    end
    
    subgraph "Confirmation"
        BLOCK_PROP[Block<br/>Propagated]
        BLOCK_VALID[Block<br/>Validated]
        TX_CONFIRM[TX<br/>Confirmed]
    end
    
    USER --> WALLET_CREATE
    WALLET_CREATE --> SIGN
    SIGN --> MEMPOOL_ACCEPT
    
    MEMPOOL_ACCEPT --> POLICY_CHECK
    POLICY_CHECK --> SCRIPT_CHECK
    SCRIPT_CHECK --> INV_SEND
    
    INV_SEND --> PEER_REQUEST
    PEER_REQUEST --> TX_SEND
    TX_SEND --> PEER_VALIDATE
    
    PEER_VALIDATE --> PEER_MEMPOOL
    PEER_MEMPOOL --> PEER_RELAY
    PEER_MEMPOOL --> MINER_SELECT
    
    MINER_SELECT --> BLOCK_CREATE
    BLOCK_CREATE --> BLOCK_MINE
    BLOCK_MINE --> BLOCK_PROP
    
    BLOCK_PROP --> BLOCK_VALID
    BLOCK_VALID --> TX_CONFIRM
    
    style MEMPOOL_ACCEPT fill:#9f9,stroke:#333,stroke-width:3px
    style BLOCK_VALID fill:#ff9,stroke:#333,stroke-width:3px
    style TX_CONFIRM fill:#9ff,stroke:#333,stroke-width:3px
```

---

## 6. System Initialization Sequence

```mermaid
sequenceDiagram
    participant Main
    participant Config
    participant Network
    participant Storage
    participant Validation
    participant Mempool
    participant Wallet
    participant RPC
    participant Scheduler
    
    Main->>Config: Parse Arguments
    Config->>Config: Load Config File
    Config->>Main: Settings
    
    Main->>Storage: Initialize Databases
    Storage->>Storage: Open LevelDB
    Storage->>Storage: Load Block Index
    Storage->>Main: DB Ready
    
    Main->>Validation: Initialize Chainstate
    Validation->>Validation: Load UTXO Set
    Validation->>Validation: Verify Best Block
    Validation->>Main: Chain Ready
    
    Main->>Network: Initialize Network
    Network->>Network: Bind Sockets
    Network->>Network: Start Threads
    Network->>Main: Network Ready
    
    Main->>Mempool: Load Mempool
    Mempool->>Mempool: Read mempool.dat
    Mempool->>Mempool: Validate TXs
    Mempool->>Main: Mempool Ready
    
    Main->>Wallet: Load Wallets
    Wallet->>Wallet: Open Wallet DBs
    Wallet->>Wallet: Scan Chain
    Wallet->>Main: Wallets Ready
    
    Main->>RPC: Start RPC Server
    RPC->>RPC: Bind HTTP Port
    RPC->>RPC: Register Commands
    RPC->>Main: RPC Ready
    
    Main->>Scheduler: Start Scheduler
    Scheduler->>Scheduler: Schedule Tasks
    
    Note over Main: System Ready for Operation
```

---

## 7. Shutdown and Cleanup Flow

```mermaid
flowchart TD
    subgraph "Shutdown Trigger"
        SIGNAL[Shutdown Signal]
        RPC_STOP[RPC Stop Command]
        GUI_CLOSE[GUI Close]
        ERROR[Fatal Error]
    end
    
    subgraph "Stop Services"
        STOP_RPC[Stop RPC Server]
        STOP_P2P[Stop P2P Network]
        STOP_MINING[Stop Mining]
        STOP_SCHEDULER[Stop Scheduler]
    end
    
    subgraph "Flush Data"
        FLUSH_MEMPOOL[Save Mempool]
        FLUSH_WALLET[Flush Wallets]
        FLUSH_CHAIN[Flush Chain State]
        FLUSH_INDEXES[Flush Indexes]
    end
    
    subgraph "Cleanup Resources"
        CLOSE_DB[Close Databases]
        FREE_MEM[Free Memory]
        CLOSE_FILES[Close Files]
        JOIN_THREADS[Join Threads]
    end
    
    SIGNAL --> STOP_RPC
    RPC_STOP --> STOP_RPC
    GUI_CLOSE --> STOP_RPC
    ERROR --> STOP_RPC
    
    STOP_RPC --> STOP_P2P
    STOP_P2P --> STOP_MINING
    STOP_MINING --> STOP_SCHEDULER
    
    STOP_SCHEDULER --> FLUSH_MEMPOOL
    FLUSH_MEMPOOL --> FLUSH_WALLET
    FLUSH_WALLET --> FLUSH_CHAIN
    FLUSH_CHAIN --> FLUSH_INDEXES
    
    FLUSH_INDEXES --> CLOSE_DB
    CLOSE_DB --> FREE_MEM
    FREE_MEM --> CLOSE_FILES
    CLOSE_FILES --> JOIN_THREADS
    
    style STOP_RPC fill:#f99,stroke:#333,stroke-width:3px
    style FLUSH_CHAIN fill:#ff9,stroke:#333,stroke-width:3px
    style CLOSE_DB fill:#9ff,stroke:#333,stroke-width:2px
```

---

## 8. Full Node Operation Lifecycle

```mermaid
stateDiagram-v2
    [*] --> STARTING: Launch
    
    STARTING --> INITIALIZING: Load Config
    INITIALIZING --> LOADING: Open Databases
    LOADING --> VERIFYING: Load Blockchain
    VERIFYING --> SYNCING: Verify State
    
    SYNCING --> IBD: Behind Tip
    SYNCING --> NORMAL: Synced
    
    IBD --> NORMAL: Catch Up
    NORMAL --> REINDEX: Corruption
    REINDEX --> VERIFYING: Rebuild
    
    NORMAL --> MINING: Generate
    MINING --> NORMAL: Stop Mining
    
    NORMAL --> SHUTTING_DOWN: Shutdown
    IBD --> SHUTTING_DOWN: Shutdown
    MINING --> SHUTTING_DOWN: Shutdown
    
    SHUTTING_DOWN --> FLUSHING: Save State
    FLUSHING --> CLEANUP: Close Resources
    CLEANUP --> [*]: Exit
    
    note right of IBD
        Initial Block Download
        Fetch and validate
        historical blocks
    end note
    
    note right of NORMAL
        Normal operation:
        - Validate new blocks
        - Relay transactions
        - Serve RPC requests
    end note
    
    note left of REINDEX
        Rebuild block index
        and chain state from
        block files
    end note
```

---

## Integration Summary

### Key Integration Patterns

1. **Event-Driven Architecture**
   - CMainSignals for loose coupling between components
   - Asynchronous notification of state changes
   - Observer pattern for extensibility

2. **Layered Dependencies**
   - Clear separation between layers
   - Dependencies flow downward only
   - Interfaces abstract implementation details

3. **Shared State Management**
   - Critical sections protected by mutexes
   - Lock ordering to prevent deadlocks
   - Atomic operations for simple state

4. **Resource Lifecycle**
   - RAII for automatic cleanup
   - Explicit initialization order
   - Graceful shutdown sequence

5. **Cross-Component Communication**
   - Direct function calls within layers
   - Event signals across layers
   - Message passing for network operations

### Performance Considerations

```mermaid
graph LR
    subgraph "Bottlenecks"
        LOCK[Lock Contention<br/>cs_main]
        IO[Disk I/O<br/>Block Storage]
        CPU[CPU Usage<br/>Script Validation]
        NET[Network Latency<br/>Block Propagation]
    end
    
    subgraph "Optimizations"
        PARALLEL[Parallelization]
        CACHE[Caching Strategy]
        ASYNC[Async Operations]
        BATCH[Batch Processing]
    end
    
    LOCK --> PARALLEL
    IO --> CACHE
    CPU --> PARALLEL
    NET --> ASYNC
    IO --> BATCH
    
    style LOCK fill:#f99,stroke:#333,stroke-width:3px
    style PARALLEL fill:#9f9,stroke:#333,stroke-width:3px
```

### Critical Data Flows

1. **Block Processing**: Network → Validation → Storage → Notifications
2. **Transaction Flow**: Wallet → Mempool → Network → Mining → Confirmation
3. **State Updates**: Validation → UTXO Set → Database → Cache
4. **Query Path**: RPC → Validation/Wallet → Response

These integration diagrams demonstrate how Bitcoin Core's components work together as a cohesive system, with clear boundaries, well-defined interfaces, and robust error handling throughout the architecture.