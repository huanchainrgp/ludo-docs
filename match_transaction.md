# Match Transactions System - Architecture Diagrams

## System Architecture Overview

```mermaid
graph TB
    subgraph "Interface Layer"
        HTTP[HTTP Handler]
        WS[WebSocket Handler]
    end
    
    subgraph "Application Layer"
        MoveUC[MovePieceUseCase]
        LootSvc[LootService]
        RecordUC[RecordMatchTransactionUseCase]
        CalcScoreUC[CalculateFinalScoresUseCase]
        SettleUC[SettleMatchBalancesUseCase]
    end
    
    subgraph "Domain Layer"
        MatchTx[MatchTransaction Entity]
        RepoInterface[MatchTransactionRepository Interface]
    end
    
    subgraph "Infrastructure Layer"
        RepoImpl[PostgresMatchTransactionRepository]
        DB[(PostgreSQL Database)]
    end
    
    HTTP --> MoveUC
    WS --> MoveUC
    MoveUC --> LootSvc
    MoveUC --> RecordUC
    LootSvc --> RecordUC
    MoveUC --> CalcScoreUC
    CalcScoreUC --> SettleUC
    RecordUC --> RepoInterface
    CalcScoreUC --> RepoInterface
    SettleUC --> RepoInterface
    RepoInterface --> RepoImpl
    RepoImpl --> DB
    MatchTx --> RepoInterface
```

## Transaction Flow During Gameplay

```mermaid
sequenceDiagram
    participant Player
    participant MoveUC as MovePieceUseCase
    participant LootSvc as LootService
    participant RecordUC as RecordMatchTransactionUseCase
    participant Repo as MatchTransactionRepository
    participant DB as Database
    
    Player->>MoveUC: Move Piece
    MoveUC->>MoveUC: Calculate Position & Distance
    MoveUC->>RecordUC: Record Piece Position Transaction
    RecordUC->>Repo: Create Transaction
    Repo->>DB: INSERT piece_position
    
    alt Piece Knocks Enemy
        MoveUC->>LootSvc: ProcessLoot(attacker, victim, turn)
        LootSvc->>LootSvc: Calculate Loot Amount
        LootSvc->>LootSvc: Update Balances
        LootSvc-->>MoveUC: Return LootResult
        MoveUC->>RecordUC: Record Loot Transaction
        RecordUC->>Repo: Create Transaction
        Repo->>DB: INSERT loot
    end
```

## Match Completion Flow

```mermaid
sequenceDiagram
    participant Handler
    participant CalcUC as CalculateFinalScoresUseCase
    participant SettleUC as SettleMatchBalancesUseCase
    participant RecordUC as RecordMatchTransactionUseCase
    participant WalletRepo as WalletRepository
    participant MatchTxRepo as MatchTransactionRepository
    participant DB as Database
    
    Handler->>CalcUC: Calculate Final Scores
    CalcUC->>CalcUC: Get Player Balances
    CalcUC->>CalcUC: Get Piece States
    CalcUC->>CalcUC: Get Position Transactions
    CalcUC->>CalcUC: Calculate Scores
    loop For Each Player
        CalcUC->>RecordUC: Record Final Score Transaction
        RecordUC->>MatchTxRepo: Create Transaction
        MatchTxRepo->>DB: INSERT final_score
    end
    CalcUC-->>Handler: Return Final Scores
    
    Handler->>SettleUC: Settle Match Balances
    SettleUC->>SettleUC: Get Final Score Transactions
    loop For Each Player
        SettleUC->>WalletRepo: Get User Balance
        SettleUC->>WalletRepo: Credit Balance
        SettleUC->>WalletRepo: Create Wallet Transaction
        SettleUC->>RecordUC: Record Balance Update Transaction
        RecordUC->>MatchTxRepo: Create Transaction
        MatchTxRepo->>DB: INSERT balance_update
    end
    SettleUC-->>Handler: Return Settlement Results
```

## Transaction Types and Data Model

```mermaid
erDiagram
    MATCH_TRANSACTIONS {
        uuid id PK
        uuid match_id FK
        uuid user_id FK
        varchar transaction_type
        bigint amount
        uuid looted_from_user_id FK
        uuid piece_id FK
        integer from_position
        integer to_position
        integer distance
        bigint final_score
        bigint balance_before
        bigint balance_after
        integer turn
        jsonb metadata
        timestamp created_at
        timestamp updated_at
    }
    
    MATCHES {
        uuid id PK
        uuid room_id FK
        varchar status
    }
    
    USERS {
        uuid id PK
        varchar username
    }
    
    PIECE_STATES {
        uuid id PK
        uuid match_id FK
        uuid user_id FK
    }
    
    PLAYER_BALANCES {
        uuid id PK
        uuid match_id FK
        uuid user_id FK
        bigint current_balance
        bigint looted_amount
    }
    
    WALLET_TRANSACTIONS {
        uuid id PK
        uuid user_id FK
        varchar transaction_type
        bigint amount
    }
    
    MATCH_TRANSACTIONS ||--o{ MATCHES : "belongs to"
    MATCH_TRANSACTIONS ||--o{ USERS : "involves"
    MATCH_TRANSACTIONS ||--o{ PIECE_STATES : "references"
    MATCH_TRANSACTIONS ||--o{ PLAYER_BALANCES : "tracks"
    MATCH_TRANSACTIONS ||--o{ WALLET_TRANSACTIONS : "links to"
```

## Transaction Type Flow

```mermaid
stateDiagram-v2
    [*] --> Gameplay
    
    Gameplay --> LootTransaction: Player knocks enemy
    Gameplay --> PositionTransaction: Piece moves
    
    LootTransaction --> Recorded: Save to DB
    PositionTransaction --> Recorded: Save to DB
    
    Gameplay --> MatchEnd: Game finishes
    
    MatchEnd --> CalculateScores: Process all players
    CalculateScores --> FinalScoreTransaction: For each player
    FinalScoreTransaction --> Recorded: Save to DB
    
    CalculateScores --> SettleBalances: Update wallets
    SettleBalances --> BalanceUpdateTransaction: For each player
    BalanceUpdateTransaction --> WalletTransaction: Create wallet tx
    BalanceUpdateTransaction --> Recorded: Save to DB
    
    Recorded --> [*]
```

## Component Dependencies

```mermaid
graph LR
    subgraph "Use Cases"
        RecordUC[RecordMatchTransactionUseCase]
        CalcUC[CalculateFinalScoresUseCase]
        SettleUC[SettleMatchBalancesUseCase]
    end
    
    subgraph "Services"
        LootSvc[LootService]
        MoveUC[MovePieceUseCase]
    end
    
    subgraph "Repositories"
        MatchTxRepo[MatchTransactionRepository]
        PlayerBalanceRepo[PlayerBalanceRepository]
        PieceStateRepo[PieceStateRepository]
        WalletRepo[WalletRepository]
    end
    
    RecordUC --> MatchTxRepo
    CalcUC --> MatchTxRepo
    CalcUC --> PlayerBalanceRepo
    CalcUC --> PieceStateRepo
    CalcUC --> RecordUC
    SettleUC --> MatchTxRepo
    SettleUC --> PlayerBalanceRepo
    SettleUC --> WalletRepo
    SettleUC --> RecordUC
    MoveUC --> LootSvc
    MoveUC --> RecordUC
    MoveUC --> PlayerBalanceRepo
```

## Final Score Calculation Logic

```mermaid
flowchart TD
    Start([Match Ends]) --> GetBalances[Get All Player Balances]
    GetBalances --> GetPieces[Get All Piece States]
    GetPieces --> GetPositions[Get Position Transactions]
    
    GetPositions --> LoopStart{For Each Player}
    LoopStart --> CalcBase[Base Score = Current Balance]
    CalcBase --> CountFinished[Count Finished Pieces]
    CountFinished --> CalcFinishedBonus[Finished Bonus = Count × 1000]
    
    CalcFinishedBonus --> SumDistance[Sum Total Distance]
    SumDistance --> CalcDistanceBonus[Distance Bonus = Distance × 10]
    
    CalcDistanceBonus --> CalcFinal[Final Score = Base + Finished Bonus + Distance Bonus]
    CalcFinal --> RecordScore[Record Final Score Transaction]
    RecordScore --> NextPlayer{More Players?}
    
    NextPlayer -->|Yes| LoopStart
    NextPlayer -->|No| ReturnScores[Return All Final Scores]
    ReturnScores --> End([Complete])
```

## Data Flow: Loot Transaction

```mermaid
flowchart LR
    A[Player A Moves] --> B[Knocks Player B's Piece]
    B --> C[LootService.ProcessLoot]
    C --> D{Attacker Bankrupt?}
    D -->|Yes| E[No Money Transfer]
    D -->|No| F{Victim Bankrupt?}
    F -->|Yes| E
    F -->|No| G[Calculate Loot Amount]
    G --> H[Update Attacker Balance]
    H --> I[Update Victim Balance]
    I --> J[Record Loot Transaction]
    J --> K[(Database)]
    E --> L[Record Piece Reset Only]
```

## Integration Points

```mermaid
graph TB
    subgraph "Gameplay Events"
        Move[Move Piece]
        Loot[Loot Event]
        End[Match End]
    end
    
    subgraph "Transaction Recording"
        RecordPos[Record Position]
        RecordLoot[Record Loot]
        RecordScore[Record Final Score]
        RecordBalance[Record Balance Update]
    end
    
    subgraph "Business Logic"
        CalcScore[Calculate Scores]
        Settle[Settle Balances]
    end
    
    subgraph "External Systems"
        Wallet[Wallet System]
        Stats[Statistics System - Future]
    end
    
    Move --> RecordPos
    Loot --> RecordLoot
    End --> CalcScore
    CalcScore --> RecordScore
    CalcScore --> Settle
    Settle --> RecordBalance
    Settle --> Wallet
    RecordPos --> Stats
    RecordLoot --> Stats
    RecordScore --> Stats
    RecordBalance --> Stats
```

