# Bitcoin Core Architecture Diagrams

This document contains comprehensive visual representations of Bitcoin Core's architecture, data flows, and component interactions.

## Table of Contents
1. [High-Level System Architecture](#1-high-level-system-architecture)
2. [Component Interaction Overview](#2-component-interaction-overview)
3. [Consensus and Validation Flow](#3-consensus-and-validation-flow)
4. [Networking and P2P Architecture](#4-networking-and-p2p-architecture)
5. [Mempool Transaction Flow](#5-mempool-transaction-flow)
6. [Storage Layer Architecture](#6-storage-layer-architecture)
7. [RPC/API Request Flow](#7-rpcapi-request-flow)
8. [Wallet Transaction Creation](#8-wallet-transaction-creation)
9. [Mining and Block Creation](#9-mining-and-block-creation)
10. [Thread Model and Concurrency](#10-thread-model-and-concurrency)
11. [Data Flow Integration](#11-data-flow-integration)
12. [Error Handling and Recovery](#12-error-handling-and-recovery)

---

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph "User Interfaces"
        GUI[Qt GUI Application]
        CLI[bitcoin-cli]
        RPC[RPC Clients]
        REST[REST API Clients]
    end
    
    subgraph "API Layer"
        RPCSERVER[RPC Server]
        HTTPSERVER[HTTP Server]
        RESTAPI[REST Interface]
    end
    
    subgraph "Node Core"
        INTERFACES[Node Interfaces]
        VALIDATION[Validation Engine]
        NETWORK[Network Manager]
        MEMPOOL[Transaction Pool]
        MINER[Block Assembly]
        WALLET[Wallet System]
    end
    
    subgraph "Data Layer"
        CHAINSTATE[(Chain State DB)]
        BLOCKS[(Block Files)]
        UTXO[(UTXO Set)]
        WALLETDB[(Wallet DB)]
        INDEXES[(Optional Indexes)]
    end
    
    GUI --> INTERFACES
    CLI --> RPCSERVER
    RPC --> RPCSERVER
    REST --> RESTAPI
    
    RPCSERVER --> HTTPSERVER
    RESTAPI --> HTTPSERVER
    HTTPSERVER --> INTERFACES
    
    INTERFACES --> VALIDATION
    INTERFACES --> NETWORK
    INTERFACES --> WALLET
    INTERFACES --> MINER
    
    VALIDATION <--> MEMPOOL
    VALIDATION <--> CHAINSTATE
    VALIDATION <--> BLOCKS
    VALIDATION <--> UTXO
    
    NETWORK <--> MEMPOOL
    WALLET <--> WALLETDB
    MINER <--> MEMPOOL
    
    style GUI fill:#f9f,stroke:#333,stroke-width:2px
    style VALIDATION fill:#ff9,stroke:#333,stroke-width:4px
    style UTXO fill:#9ff,stroke:#333,stroke-width:2px
```

---

## 2. Component Interaction Overview

```mermaid
graph LR
    subgraph "External Connections"
        PEERS[P2P Network Peers]
        USERS[User Applications]
    end
    
    subgraph "Bitcoin Core Components"
        NET[CConnman<br/>Network Layer]
        PROC[PeerManager<br/>Message Processing]
        VAL[ChainstateManager<br/>Validation]
        MEM[CTxMemPool<br/>Mempool]
        WAL[CWallet<br/>Wallet]
        RPC[RPC Server]
        INDEX[Indexers]
    end
    
    PEERS <--> NET
    NET <--> PROC
    PROC <--> VAL
    PROC <--> MEM
    VAL <--> MEM
    
    USERS <--> RPC
    RPC --> VAL
    RPC --> MEM
    RPC --> WAL
    RPC --> NET
    
    WAL --> MEM
    VAL --> INDEX
    
    MEM -.-> WAL
    
    style VAL fill:#ff9,stroke:#333,stroke-width:4px
    style MEM fill:#9f9,stroke:#333,stroke-width:2px
```

---

## 3. Consensus and Validation Flow

```mermaid
flowchart TD
    subgraph "Block Reception"
        RECEIVE[Receive Block<br/>from Network]
        HEADER[Check Block Header]
        CONTEXT[Check Context]
    end
    
    subgraph "Block Validation"
        DESERIALIZE[Deserialize Block]
        CHECKTX[Check Transactions]
        MERKLE[Verify Merkle Root]
        SIZE[Check Size/Weight]
    end
    
    subgraph "Chain Integration"
        CONNECT[ConnectBlock]
        SCRIPTS[Validate Scripts]
        UTXO_UPDATE[Update UTXO Set]
        CHAIN_UPDATE[Update Chain State]
    end
    
    subgraph "Post-Processing"
        SIGNALS[Send Signals]
        MEMPOOL_UPDATE[Update Mempool]
        RELAY[Relay to Peers]
    end
    
    RECEIVE --> HEADER
    HEADER -->|Valid| CONTEXT
    HEADER -->|Invalid| REJECT1[Reject Block]
    
    CONTEXT -->|Valid| DESERIALIZE
    CONTEXT -->|Invalid| REJECT2[Reject Block]
    
    DESERIALIZE --> CHECKTX
    CHECKTX --> MERKLE
    MERKLE --> SIZE
    
    SIZE -->|Valid| CONNECT
    SIZE -->|Invalid| REJECT3[Reject Block]
    
    CONNECT --> SCRIPTS
    SCRIPTS -->|Valid| UTXO_UPDATE
    SCRIPTS -->|Invalid| REJECT4[Reject & Ban Peer]
    
    UTXO_UPDATE --> CHAIN_UPDATE
    CHAIN_UPDATE --> SIGNALS
    SIGNALS --> MEMPOOL_UPDATE
    SIGNALS --> RELAY
    
    style CONNECT fill:#ff9,stroke:#333,stroke-width:4px
    style SCRIPTS fill:#f99,stroke:#333,stroke-width:2px
    style REJECT4 fill:#f00,stroke:#333,stroke-width:2px
```

---

## 4. Networking and P2P Architecture

```mermaid
graph TB
    subgraph "Network Layer"
        SOCKET[Socket Handler]
        CONNMAN[CConnman]
        NODES[CNode Objects]
    end
    
    subgraph "Message Processing"
        PEERMAN[PeerManager]
        MSGPROC[ProcessMessages]
        SEND[SendMessages]
    end
    
    subgraph "Protocol Messages"
        VERSION[VERSION/VERACK]
        BLOCK_MSG[BLOCK/HEADERS]
        TX_MSG[TX/INV]
        ADDR_MSG[ADDR/GETADDR]
        PING[PING/PONG]
    end
    
    subgraph "Network Services"
        DNS[DNS Seeds]
        UPNP[UPnP/NAT-PMP]
        TOR[Tor Integration]
        I2P[I2P Support]
    end
    
    SOCKET <--> CONNMAN
    CONNMAN <--> NODES
    NODES <--> PEERMAN
    
    PEERMAN --> MSGPROC
    PEERMAN --> SEND
    
    MSGPROC --> VERSION
    MSGPROC --> BLOCK_MSG
    MSGPROC --> TX_MSG
    MSGPROC --> ADDR_MSG
    MSGPROC --> PING
    
    DNS --> CONNMAN
    UPNP --> CONNMAN
    TOR --> SOCKET
    I2P --> SOCKET
    
    style PEERMAN fill:#9ff,stroke:#333,stroke-width:4px
    style CONNMAN fill:#ff9,stroke:#333,stroke-width:2px
```

---

## 5. Mempool Transaction Flow

```mermaid
flowchart LR
    subgraph "Transaction Entry"
        RECV[Receive TX]
        VALIDATE[ValidateTransaction]
        POLICY[PolicyChecks]
    end
    
    subgraph "Mempool Structure"
        TXMAP[TX Map by TXID]
        FEEMAP[TX by Fee Rate]
        TIMEMAP[TX by Entry Time]
        ANCESTORMAP[TX by Ancestors]
    end
    
    subgraph "Transaction Selection"
        MINING[For Mining]
        RELAY[For Relay]
        EVICTION[For Eviction]
    end
    
    subgraph "Exit Conditions"
        CONFIRMED[Block Confirmation]
        EXPIRED[Expiration]
        EVICTED[Fee Eviction]
        REPLACED[RBF Replacement]
    end
    
    RECV --> VALIDATE
    VALIDATE -->|Valid| POLICY
    VALIDATE -->|Invalid| REJECT[Reject]
    
    POLICY -->|Pass| TXMAP
    POLICY -->|Fail| REJECT
    
    TXMAP --> FEEMAP
    TXMAP --> TIMEMAP
    TXMAP --> ANCESTORMAP
    
    FEEMAP --> MINING
    TIMEMAP --> RELAY
    FEEMAP --> EVICTION
    
    TXMAP --> CONFIRMED
    TXMAP --> EXPIRED
    TXMAP --> EVICTED
    TXMAP --> REPLACED
    
    style TXMAP fill:#9f9,stroke:#333,stroke-width:4px
    style VALIDATE fill:#ff9,stroke:#333,stroke-width:2px
```

---

## 6. Storage Layer Architecture

```mermaid
graph TB
    subgraph "Application Layer"
        CHAIN[Chain State]
        TXINDEX[Transaction Index]
        BLOCKFILTER[Block Filter Index]
        COINSTATS[Coin Stats Index]
    end
    
    subgraph "Cache Layer"
        COINSCACHE[CCoinsViewCache<br/>UTXO Cache]
        DBCACHE[Database Cache]
    end
    
    subgraph "Database Abstraction"
        WRAPPER[CDBWrapper]
        BATCH[CDBBatch]
        ITERATOR[CDBIterator]
    end
    
    subgraph "Storage Engines"
        LEVELDB[(LevelDB)]
        SQLITE[(SQLite)]
    end
    
    subgraph "File Storage"
        BLOCKFILES[blk*.dat<br/>Block Data]
        UNDOFILES[rev*.dat<br/>Undo Data]
    end
    
    CHAIN --> COINSCACHE
    TXINDEX --> WRAPPER
    BLOCKFILTER --> WRAPPER
    COINSTATS --> WRAPPER
    
    COINSCACHE --> WRAPPER
    WRAPPER --> BATCH
    WRAPPER --> ITERATOR
    WRAPPER --> DBCACHE
    
    BATCH --> LEVELDB
    ITERATOR --> LEVELDB
    DBCACHE --> LEVELDB
    
    CHAIN --> BLOCKFILES
    CHAIN --> UNDOFILES
    
    style COINSCACHE fill:#9ff,stroke:#333,stroke-width:4px
    style LEVELDB fill:#ff9,stroke:#333,stroke-width:2px
```

---

## 7. RPC/API Request Flow

```mermaid
sequenceDiagram
    participant Client
    participant HTTPServer
    participant RPCServer
    participant Dispatcher
    participant Handler
    participant Node
    participant Response
    
    Client->>HTTPServer: HTTP POST /
    HTTPServer->>HTTPServer: Authentication
    HTTPServer->>RPCServer: JSON-RPC Request
    RPCServer->>RPCServer: Parse JSON
    RPCServer->>Dispatcher: Method + Params
    Dispatcher->>Dispatcher: Lookup Handler
    Dispatcher->>Handler: Execute Method
    Handler->>Node: Core Operations
    Node-->>Handler: Result
    Handler-->>Dispatcher: Return Value
    Dispatcher-->>RPCServer: Format Result
    RPCServer-->>HTTPServer: JSON Response
    HTTPServer-->>Client: HTTP Response
    
    Note over HTTPServer: Thread Pool Workers
    Note over Handler,Node: May acquire cs_main lock
```

---

## 8. Wallet Transaction Creation

```mermaid
flowchart TD
    subgraph "Transaction Request"
        USER[User Request]
        PARAMS[Parse Parameters]
        VALIDATE_ADDR[Validate Addresses]
    end
    
    subgraph "Coin Selection"
        AVAILABLE[Get Available Coins]
        SELECT[Select Coins Algorithm]
        CHANGE[Calculate Change]
    end
    
    subgraph "Transaction Building"
        CREATE_TX[Create Transaction]
        ADD_INPUTS[Add Inputs]
        ADD_OUTPUTS[Add Outputs]
        ADD_CHANGE[Add Change Output]
    end
    
    subgraph "Signing"
        FETCH_KEYS[Fetch Private Keys]
        SIGN_INPUTS[Sign Each Input]
        VERIFY_SIGS[Verify Signatures]
    end
    
    subgraph "Finalization"
        CALC_FEE[Calculate Final Fee]
        BROADCAST[Broadcast to Mempool]
        SAVE[Save to Wallet DB]
    end
    
    USER --> PARAMS
    PARAMS --> VALIDATE_ADDR
    
    VALIDATE_ADDR --> AVAILABLE
    AVAILABLE --> SELECT
    SELECT --> CHANGE
    
    CHANGE --> CREATE_TX
    CREATE_TX --> ADD_INPUTS
    ADD_INPUTS --> ADD_OUTPUTS
    ADD_OUTPUTS --> ADD_CHANGE
    
    ADD_CHANGE --> FETCH_KEYS
    FETCH_KEYS --> SIGN_INPUTS
    SIGN_INPUTS --> VERIFY_SIGS
    
    VERIFY_SIGS --> CALC_FEE
    CALC_FEE --> BROADCAST
    BROADCAST --> SAVE
    
    style SELECT fill:#9ff,stroke:#333,stroke-width:4px
    style SIGN_INPUTS fill:#ff9,stroke:#333,stroke-width:2px
```

---

## 9. Mining and Block Creation

```mermaid
flowchart LR
    subgraph "Block Template"
        GETTEMPLATE[GetBlockTemplate]
        HEIGHT[Next Height]
        PREVBLOCK[Previous Block]
    end
    
    subgraph "Transaction Selection"
        MEMPOOL_SNAPSHOT[Mempool Snapshot]
        SORT_BY_FEE[Sort by Fee Rate]
        PACKAGE_EVAL[Package Evaluation]
        ADD_TXS[Add Transactions]
    end
    
    subgraph "Block Assembly"
        COINBASE[Create Coinbase]
        MERKLE_ROOT[Calculate Merkle Root]
        HEADER[Build Block Header]
        VALIDATE_WEIGHT[Check Weight/Size]
    end
    
    subgraph "Finalization"
        NONCE[Nonce/ExtraNonce]
        WITNESS[Witness Commitment]
        TEMPLATE[Complete Template]
    end
    
    GETTEMPLATE --> HEIGHT
    HEIGHT --> PREVBLOCK
    
    PREVBLOCK --> MEMPOOL_SNAPSHOT
    MEMPOOL_SNAPSHOT --> SORT_BY_FEE
    SORT_BY_FEE --> PACKAGE_EVAL
    PACKAGE_EVAL --> ADD_TXS
    
    ADD_TXS --> COINBASE
    COINBASE --> MERKLE_ROOT
    MERKLE_ROOT --> HEADER
    HEADER --> VALIDATE_WEIGHT
    
    VALIDATE_WEIGHT --> NONCE
    NONCE --> WITNESS
    WITNESS --> TEMPLATE
    
    style PACKAGE_EVAL fill:#9ff,stroke:#333,stroke-width:4px
    style MERKLE_ROOT fill:#ff9,stroke:#333,stroke-width:2px
```

---

## 10. Thread Model and Concurrency

```mermaid
graph TB
    subgraph "Main Thread"
        INIT[Initialization]
        SHUTDOWN[Shutdown Handler]
    end
    
    subgraph "Network Threads"
        MSGHAND[Message Handler Thread]
        SOCKET[Socket Thread]
        DNSADD[DNS Address Thread]
        UPNP_THR[UPnP Thread]
    end
    
    subgraph "Validation Threads"
        MAIN_VAL[Main Validation]
        SCRIPT1[Script Check Worker 1]
        SCRIPT2[Script Check Worker 2]
        SCRIPTN[Script Check Worker N]
    end
    
    subgraph "RPC Threads"
        HTTP1[HTTP Worker 1]
        HTTP2[HTTP Worker 2]
        HTTPN[HTTP Worker N]
    end
    
    subgraph "Background Tasks"
        SCHEDULER[CScheduler Thread]
        INDEXER[Index Build Thread]
        WALLET_FLUSH[Wallet Flush Thread]
    end
    
    subgraph "Shared Resources"
        CS_MAIN{{cs_main mutex}}
        CS_MEMPOOL{{mempool.cs mutex}}
        CS_WALLET{{wallet.cs mutex}}
    end
    
    MAIN_VAL -.-> CS_MAIN
    MSGHAND -.-> CS_MAIN
    HTTP1 -.-> CS_MAIN
    
    MSGHAND -.-> CS_MEMPOOL
    MAIN_VAL -.-> CS_MEMPOOL
    
    WALLET_FLUSH -.-> CS_WALLET
    HTTP1 -.-> CS_WALLET
    
    style CS_MAIN fill:#f99,stroke:#333,stroke-width:4px
    style MAIN_VAL fill:#ff9,stroke:#333,stroke-width:2px
```

---

## 11. Data Flow Integration

```mermaid
graph TB
    subgraph "Data Sources"
        P2P[P2P Network]
        USER_RPC[User RPC/CLI]
        MINER_REQ[Miner Requests]
        WALLET_OP[Wallet Operations]
    end
    
    subgraph "Processing Pipeline"
        RECEIVE[Receive & Validate]
        PROCESS[Process Data]
        UPDATE[Update State]
        PERSIST[Persist Changes]
        NOTIFY[Notify Subscribers]
    end
    
    subgraph "State Management"
        BLOCKCHAIN[Blockchain State]
        MEMPOOL_STATE[Mempool State]
        WALLET_STATE[Wallet State]
        NETWORK_STATE[Network State]
    end
    
    subgraph "Data Sinks"
        DATABASE[(Persistent Storage)]
        PEERS_OUT[Peer Broadcast]
        NOTIFICATIONS[Event Notifications]
        CACHE[Memory Caches]
    end
    
    P2P --> RECEIVE
    USER_RPC --> RECEIVE
    MINER_REQ --> RECEIVE
    WALLET_OP --> RECEIVE
    
    RECEIVE --> PROCESS
    PROCESS --> UPDATE
    
    UPDATE --> BLOCKCHAIN
    UPDATE --> MEMPOOL_STATE
    UPDATE --> WALLET_STATE
    UPDATE --> NETWORK_STATE
    
    UPDATE --> PERSIST
    PERSIST --> DATABASE
    
    UPDATE --> NOTIFY
    NOTIFY --> PEERS_OUT
    NOTIFY --> NOTIFICATIONS
    
    UPDATE --> CACHE
    
    style PROCESS fill:#9ff,stroke:#333,stroke-width:4px
    style UPDATE fill:#ff9,stroke:#333,stroke-width:4px
```

---

## 12. Error Handling and Recovery

```mermaid
flowchart TD
    subgraph "Error Detection"
        OPERATION[Normal Operation]
        ERROR[Error Detected]
        CLASSIFY[Classify Error]
    end
    
    subgraph "Error Types"
        NETWORK_ERR[Network Error]
        VALIDATION_ERR[Validation Error]
        STORAGE_ERR[Storage Error]
        CONSENSUS_ERR[Consensus Error]
    end
    
    subgraph "Recovery Actions"
        RETRY[Retry Operation]
        ROLLBACK[Rollback State]
        DISCONNECT[Disconnect Peer]
        REINDEX[Reindex Blockchain]
        FATAL[Fatal Shutdown]
    end
    
    subgraph "Logging & Monitoring"
        LOG[Log Error]
        METRICS[Update Metrics]
        ALERT[Send Alerts]
    end
    
    OPERATION --> ERROR
    ERROR --> CLASSIFY
    
    CLASSIFY --> NETWORK_ERR
    CLASSIFY --> VALIDATION_ERR
    CLASSIFY --> STORAGE_ERR
    CLASSIFY --> CONSENSUS_ERR
    
    NETWORK_ERR --> RETRY
    NETWORK_ERR --> DISCONNECT
    
    VALIDATION_ERR --> ROLLBACK
    VALIDATION_ERR --> DISCONNECT
    
    STORAGE_ERR --> REINDEX
    STORAGE_ERR --> FATAL
    
    CONSENSUS_ERR --> FATAL
    
    ERROR --> LOG
    LOG --> METRICS
    METRICS --> ALERT
    
    style CONSENSUS_ERR fill:#f00,stroke:#333,stroke-width:4px
    style FATAL fill:#f00,stroke:#333,stroke-width:2px
```

---

## Component Interaction Matrix

| Component | Validation | Network | Mempool | Storage | Wallet | RPC | Mining |
|-----------|------------|---------|---------|---------|--------|-----|--------|
| **Validation** | - | Receives blocks | Updates after block | Reads/Writes | Notifies | Status queries | Validates templates |
| **Network** | Sends blocks | - | Relays txs | - | - | Peer info | - |
| **Mempool** | Removed txs | Receives txs | - | - | Broadcasts | TX queries | Provides txs |
| **Storage** | Chain state | - | - | - | Wallet DB | - | - |
| **Wallet** | Balance updates | - | Sends txs | Reads/Writes | - | Wallet RPCs | - |
| **RPC** | Chain queries | Peer control | TX queries | - | Wallet ops | - | Mining RPCs |
| **Mining** | New blocks | - | Selects txs | - | - | Templates | - |

---

## Key Design Patterns

### 1. Observer Pattern
- **ValidationInterface**: Notifies subscribers of blockchain events
- **CMainSignals**: Manages signal delivery to multiple observers
- Used for wallet updates, index building, and P2P relay

### 2. Command Pattern
- **RPC Commands**: Each RPC method is a command object
- Enables dynamic registration and dispatch
- Supports permission-based access control

### 3. Strategy Pattern
- **Coin Selection**: Multiple algorithms for transaction input selection
- **Fee Estimation**: Different models for predicting confirmation times
- **Script Validation**: Configurable validation flags

### 4. Chain of Responsibility
- **Message Processing**: Network messages pass through handler chain
- **Error Handling**: Errors bubble up through component layers
- **Validation Pipeline**: Sequential validation steps

### 5. Template Method
- **Block Template Creation**: Customizable mining algorithm
- **Transaction Validation**: Standard flow with override points
- **Index Building**: Base class with derived implementations

---

## Performance Critical Paths

```mermaid
graph LR
    subgraph "Hot Paths"
        SCRIPT_VAL[Script Validation<br/>15k ops/block]
        UTXO_LOOKUP[UTXO Lookups<br/>100k/sec]
        SIG_VERIFY[Signature Verification<br/>50k/block]
        MERKLE_CALC[Merkle Calculation<br/>1k txs/block]
    end
    
    subgraph "Optimizations"
        PARALLEL[Parallel Validation]
        CACHE[Multi-level Caches]
        SIMD[SIMD Instructions]
        PRECOMPUTE[Precomputed Tables]
    end
    
    SCRIPT_VAL --> PARALLEL
    UTXO_LOOKUP --> CACHE
    SIG_VERIFY --> SIMD
    MERKLE_CALC --> PRECOMPUTE
    
    style SCRIPT_VAL fill:#f00,stroke:#333,stroke-width:4px
    style UTXO_LOOKUP fill:#f00,stroke:#333,stroke-width:4px
```

---

## Security Boundaries

```mermaid
graph TB
    subgraph "Untrusted Input"
        NETWORK_MSG[Network Messages]
        RPC_INPUT[RPC Parameters]
        USER_INPUT[User Input]
    end
    
    subgraph "Validation Layer"
        DESERIALIZE[Safe Deserialization]
        BOUNDS_CHECK[Bounds Checking]
        SANITIZE[Input Sanitization]
    end
    
    subgraph "Trusted Core"
        CONSENSUS[Consensus Rules]
        STATE[System State]
        KEYS[Private Keys]
    end
    
    NETWORK_MSG --> DESERIALIZE
    RPC_INPUT --> BOUNDS_CHECK
    USER_INPUT --> SANITIZE
    
    DESERIALIZE --> CONSENSUS
    BOUNDS_CHECK --> STATE
    SANITIZE --> KEYS
    
    style DESERIALIZE fill:#f99,stroke:#333,stroke-width:4px
    style CONSENSUS fill:#9f9,stroke:#333,stroke-width:4px
```

This comprehensive set of diagrams provides a complete visual understanding of Bitcoin Core's architecture, showing:

1. **System structure** - How components are organized
2. **Data flows** - How information moves through the system
3. **Processing pipelines** - Step-by-step operations
4. **Concurrency model** - Thread interactions and synchronization
5. **Error handling** - Recovery and resilience mechanisms
6. **Performance paths** - Critical optimization points
7. **Security boundaries** - Trust zones and validation layers

Each diagram can be rendered using any Mermaid-compatible viewer or documentation system.