# Gameplay Diagrams - Visual Workflow

This document contains visual diagrams for the gameplay workflow using Mermaid syntax.

## Table of Contents

1. [Game Lifecycle Diagram](#game-lifecycle-diagram)
2. [Initialization Flow](#initialization-flow)
3. [Turn Flow Diagram](#turn-flow-diagram)
4. [State Machine Diagrams](#state-machine-diagrams)
5. [Player Actions Flow](#player-actions-flow)
6. [Piece Movement Logic](#piece-movement-logic)
7. [Interaction Resolution](#interaction-resolution)
8. [Sequence Diagrams](#sequence-diagrams)

---

## Game Lifecycle Diagram

### Complete Game Flow

```mermaid
graph TB
    Start([Game Start]) --> CreateRoom[Create Room]
    CreateRoom --> CreateMatch[Create Match]
    CreateMatch --> JoinPlayers[Players Join Match]
    JoinPlayers --> CheckPlayers{4 Players?}
    CheckPlayers -->|No| WaitMore[Wait for More Players]
    WaitMore --> JoinPlayers
    CheckPlayers -->|Yes| StartMatch[Start Match]
    StartMatch --> InitGame[Initialize Game]
    
    InitGame --> RandomizeOrder[Randomize Player Order]
    RandomizeOrder --> CreateBoard[Create Board State]
    CreateBoard --> CreatePieces[Create Pieces]
    CreatePieces --> CreateBalances[Create Player Balances]
    CreateBalances --> CreateDicePool[Create Dice Pool]
    CreateDicePool --> GameReady[Game Ready]
    
    GameReady --> GameLoop{Game Loop}
    
    GameLoop --> TurnStart[Turn Start]
    TurnStart --> RollDice[Roll Dice]
    RollDice --> CheckDouble{Is Double?}
    CheckDouble -->|Yes| CheckDoubleCount{DoubleCount < 3?}
    CheckDoubleCount -->|Yes| MovePiece[Move Piece]
    CheckDoubleCount -->|No| EndTurn[End Turn]
    CheckDouble -->|No| MovePiece
    
    MovePiece --> CalculatePosition[Calculate Position]
    CalculatePosition --> HandleSpecialCases[Handle Special Cases]
    HandleSpecialCases --> ResolveInteractions[Resolve Interactions]
    ResolveInteractions --> UpdatePiece[Update Piece]
    UpdatePiece --> RecordHistory[Record History]
    RecordHistory --> CheckDoubleTurn{Double < 3?}
    
    CheckDoubleTurn -->|Yes| ExtraTurn[Extra Turn]
    ExtraTurn --> RollDice
    CheckDoubleTurn -->|No| EndTurn
    
    EndTurn --> CheckEndGame{End Game?}
    CheckEndGame -->|No| NextPlayer[Next Player]
    NextPlayer --> TurnStart
    CheckEndGame -->|Yes| GameFinished[Game Finished]
    
    GameFinished --> End([End])
    
    style Start fill:#4CAF50,stroke:#2E7D32,color:#FFFFFF
    style End fill:#E53935,stroke:#B71C1C,color:#FFFFFF
    style GameReady fill:#1976D2,stroke:#0D47A1,color:#FFFFFF
    style GameFinished fill:#FFB300,stroke:#F57F17,color:#111111
```

---

## Initialization Flow

### Game Initialization Process

```mermaid
flowchart TD
    Start([Initialize Request]) --> ValidateMatch{Validate Match}
    ValidateMatch -->|Invalid| Error1[Return Error]
    ValidateMatch -->|Valid| CheckStatus{Status = STARTED?}
    CheckStatus -->|No| Error2[Match Not Started]
    CheckStatus -->|Yes| GetPlayers[Get Match Players]
    
    GetPlayers --> CheckCount{4 Players?}
    CheckCount -->|No| Error3[Requires 4 Players]
    CheckCount -->|Yes| ShufflePlayers[Randomize Player Order]
    
    ShufflePlayers --> CreateBoard[Create Board State]
    CreateBoard --> InitGameState[Create Game State]
    InitGameState --> InitPieces[Initialize Pieces]
    InitPieces --> InitBalances[Initialize Balances]
    
    InitBalances --> SaveGameState[Save Game State]
    SaveGameState --> SavePieces[Save Pieces]
    SavePieces --> SaveBalances[Save Balances]
    
    SaveBalances --> CreateDicePool[Create Dice Pool in Redis]
    CreateDicePool --> Success([Game Initialized])
    
    style Start fill:#4CAF50,stroke:#2E7D32,color:#FFFFFF
    style Success fill:#43A047,stroke:#1B5E20,color:#FFFFFF
    style Error1 fill:#E53935,stroke:#B71C1C,color:#FFFFFF
    style Error2 fill:#E53935,stroke:#B71C1C,color:#FFFFFF
    style Error3 fill:#E53935,stroke:#B71C1C,color:#FFFFFF
```

---

## Turn Flow Diagram

### Complete Turn Cycle

```mermaid
stateDiagram-v2
    [*] --> TurnStart: Game Ready
    
    TurnStart: Turn Start
    TurnStart: Timer = 300s
    TurnStart: CurrentPlayer = Seat Index
    
    TurnStart --> RollDice: Player Action
    
    RollDice: Roll Dice
    RollDice: Validate Turn
    RollDice: Calculate Weighted Roll
    RollDice: Check Double
    
    RollDice --> CheckDouble: Dice Rolled
    
    CheckDouble: Check Double
    CheckDouble --> IncrementDouble: Is Double
    CheckDouble --> ResetDouble: Not Double
    CheckDouble --> MovePiece: Roll Complete
    
    IncrementDouble: Increment DoubleCount
    IncrementDouble --> CheckDoubleLimit: Count Updated
    
    CheckDoubleLimit: Check Limit
    CheckDoubleLimit --> MovePiece: Count < 3
    CheckDoubleLimit --> EndTurn: Count >= 3
    
    ResetDouble: Reset DoubleCount
    ResetDouble --> MovePiece: Count = 0
    
    MovePiece: Move Piece
    MovePiece: Find Active Piece
    MovePiece: Calculate Position
    MovePiece: Handle Special Cases
    MovePiece: Resolve Interactions
    
    MovePiece --> UpdateState: Move Complete
    
    UpdateState: Update State
    UpdateState: Update Piece Position
    UpdateState: Record History
    UpdateState: Update Match State
    
    UpdateState --> CheckExtraTurn: State Updated
    
    CheckExtraTurn: Check Extra Turn
    CheckExtraTurn --> RollDice: Double && Count < 3
    CheckExtraTurn --> NextPlayer: Normal Turn
    
    NextPlayer: Next Player
    NextPlayer: Advance Seat Index
    NextPlayer: Increment Turn if needed
    NextPlayer: Reset Flags
    
    NextPlayer --> CheckEndGame: Turn Complete
    
    CheckEndGame: Check End Game
    CheckEndGame --> TurnStart: Continue
    CheckEndGame --> GameFinished: End Conditions Met
    
    GameFinished: Game Finished
    GameFinished --> [*]
```

---

## State Machine Diagrams

### Game Status State Machine

```mermaid
stateDiagram-v2
    [*] --> WAITING: Game Created
    
    WAITING --> STARTING: Initialization Started
    WAITING --> CANCELLED: Cancel Match
    
    STARTING --> IN_PROGRESS: Initialization Complete
    STARTING --> CANCELLED: Initialization Failed
    
    IN_PROGRESS --> IN_PROGRESS: Continue Playing
    IN_PROGRESS --> FINISHED: End Conditions Met
    IN_PROGRESS --> CANCELLED: Game Cancelled
    
    FINISHED --> [*]: Game Complete
    CANCELLED --> [*]: Game Cancelled
    
    note right of WAITING
        Waiting for initialization
    end note
    
    note right of IN_PROGRESS
        Active gameplay
        Turns continue
    end note
    
    note right of FINISHED
        Winner determined
        Rewards calculated
    end note
```

### Match Status State Machine

```mermaid
stateDiagram-v2
    [*] --> PENDING: Match Created
    
    PENDING --> PENDING: Player Joining
    PENDING --> STARTED: 4 Players Joined
    PENDING --> CANCELLED: Match Cancelled
    
    STARTED --> IN_PROGRESS: Game Initialized
    STARTED --> FINISHED: Match Ended
    
    IN_PROGRESS --> FINISHED: Game Completed
    
    FINISHED --> [*]: Match Complete
    CANCELLED --> [*]: Match Cancelled
```

### Turn State Machine

```mermaid
stateDiagram-v2
    [*] --> TurnReady: Turn Starts
    
    TurnReady: Ready to Roll
    TurnReady: HasRolled = false
    TurnReady: HasMoved = false
    
    TurnReady --> DiceRolled: Player Rolls Dice
    
    DiceRolled: Dice Rolled
    DiceRolled: HasRolled = true
    DiceRolled: Roll Result Stored
    
    DiceRolled --> PieceMoved: Player Moves Piece
    
    PieceMoved: Piece Moved
    PieceMoved: HasMoved = true
    PieceMoved: Position Updated
    
    PieceMoved --> CheckConditions: Move Complete
    
    CheckConditions: Check Conditions
    CheckConditions --> ExtraTurn: Double && Count < 3
    CheckConditions --> TurnEnd: Normal Turn
    
    ExtraTurn: Extra Turn
    ExtraTurn --> TurnReady: Continue Same Player
    
    TurnEnd: Turn Ends
    TurnEnd: Reset Flags
    TurnEnd: Next Player
    
    TurnEnd --> [*]: Turn Complete
```

---

## Player Actions Flow

### Roll Dice Flow

```mermaid
flowchart TD
    Start([Roll Dice Request]) --> Auth{Authenticated?}
    Auth -->|No| AuthError[Unauthorized]
    Auth -->|Yes| GetMatch[Get Match]
    
    GetMatch --> CheckMatchStatus{Match Started?}
    CheckMatchStatus -->|No| MatchError[Match Not Started]
    CheckMatchStatus -->|Yes| GetGameState[Get Game State]
    
    GetGameState --> CheckGameStatus{Game In Progress?}
    CheckGameStatus -->|No| GameError[Game Not In Progress]
    CheckGameStatus -->|Yes| CheckTurn{Player's Turn?}
    
    CheckTurn -->|No| TurnError[Not Your Turn]
    CheckTurn -->|Yes| CheckExisting{Already Rolled?}
    
    CheckExisting -->|Yes| ReturnExisting[Return Existing Roll]
    CheckExisting -->|No| CalculateRoll[Calculate Weighted Roll]
    
    CalculateRoll --> CheckDouble{Is Double?}
    CheckDouble -->|Yes| IncrementDouble[Increment DoubleCount]
    CheckDouble -->|No| ResetDouble[Reset DoubleCount]
    
    IncrementDouble --> SaveRoll[Save Roll History]
    ResetDouble --> SaveRoll
    
    SaveRoll --> UpdateMatch[Update Match State]
    UpdateMatch --> UpdateGameState[Update Game State]
    UpdateGameState --> ReturnResult([Return Roll Result])
    
    style Start fill:#4CAF50,stroke:#2E7D32,color:#FFFFFF
    style ReturnResult fill:#43A047,stroke:#1B5E20,color:#FFFFFF
    style AuthError fill:#E53935,stroke:#B71C1C,color:#FFFFFF
    style MatchError fill:#E53935,stroke:#B71C1C,color:#FFFFFF
    style GameError fill:#E53935,stroke:#B71C1C,color:#FFFFFF
    style TurnError fill:#E53935,stroke:#B71C1C,color:#FFFFFF
```

### Move Piece Flow

```mermaid
flowchart TD
    Start([Move Piece Request]) --> ValidateMatch{Match Started?}
    ValidateMatch -->|No| Error1[Match Not Started]
    ValidateMatch -->|Yes| CheckRolled{Dice Rolled?}
    
    CheckRolled -->|No| Error2[Dice Not Rolled]
    CheckRolled -->|Yes| CheckMoved{Already Moved?}
    
    CheckMoved -->|Yes| Error3[Already Moved]
    CheckMoved -->|No| GetRollHistory[Get Roll History]
    
    GetRollHistory --> ValidateRoll{Valid Roll?}
    ValidateRoll -->|No| Error4[Invalid Roll]
    ValidateRoll -->|Yes| GetPieces[Get All Pieces]
    
    GetPieces --> FindPiece[Find Active Piece]
    FindPiece --> CheckPiece{Piece Found?}
    CheckPiece -->|No| Error5[No Active Piece]
    CheckPiece -->|Yes| CalculatePos[Calculate Position]
    
    CalculatePos --> CheckHome{Piece In Home?}
    CheckHome -->|Yes| CheckRoll6{Roll = 6?}
    CheckHome -->|No| CheckFinish{Exceeds Finish?}
    
    CheckRoll6 -->|No| Error6[Need Roll of 6]
    CheckRoll6 -->|Yes| MoveToTrack[Move to Track]
    
    CheckFinish -->|Yes| SetFinish[Set to Finish Index]
    CheckFinish -->|No| BuildPath[Build Path Indices]
    
    MoveToTrack --> BuildPath
    SetFinish --> BuildPath
    
    BuildPath --> ResolvePass[Resolve Pass-Through]
    ResolvePass --> ResolveLanding[Resolve Landing]
    
    ResolveLanding --> UpdatePiece[Update Piece Position]
    UpdatePiece --> SaveRaceHistory[Save Race History]
    SaveRaceHistory --> SaveKickHistory[Save Kick History]
    SaveKickHistory --> UpdateMatch[Update Match State]
    
    UpdateMatch --> Success([Move Success])
    
    style Start fill:#4CAF50,stroke:#2E7D32,color:#FFFFFF
    style Success fill:#43A047,stroke:#1B5E20,color:#FFFFFF
    style Error1 fill:#E53935,stroke:#B71C1C,color:#FFFFFF
    style Error2 fill:#E53935,stroke:#B71C1C,color:#FFFFFF
    style Error3 fill:#E53935,stroke:#B71C1C,color:#FFFFFF
    style Error4 fill:#E53935,stroke:#B71C1C,color:#FFFFFF
    style Error5 fill:#E53935,stroke:#B71C1C,color:#FFFFFF
    style Error6 fill:#E53935,stroke:#B71C1C,color:#FFFFFF
```

---

## Piece Movement Logic

### Position Calculation Flow

```mermaid
flowchart LR
    Start([Move Piece]) --> GetCurrent[Get Current Position]
    GetCurrent --> GetRoll[Get Roll Result]
    
    GetRoll --> CheckStatus{Piece Status}
    
    CheckStatus -->|IN_HOME| CheckRoll6{Roll = 6?}
    CheckStatus -->|IN_TRACK| CalculateNormal[Calculate Normal]
    CheckStatus -->|IN_FINISH| ErrorFinish[Already Finished]
    
    CheckRoll6 -->|No| ErrorHome[Need Roll of 6]
    CheckRoll6 -->|Yes| MoveToHomeIndex[Move to HomeIndex]
    MoveToHomeIndex --> SetTrack[Set Status = IN_TRACK]
    
    CalculateNormal --> CalculatePos[toIndex = fromIndex + roll]
    CalculatePos --> CheckFinish{toIndex > FinishIndex?}
    
    CheckFinish -->|Yes| SetFinishIndex[Set to FinishIndex]
    SetFinishIndex --> SetFinishStatus[Set Status = IN_FINISH_LINE]
    
    CheckFinish -->|No| KeepPosition[Keep Calculated Position]
    
    SetTrack --> BuildPath[Build Path]
    SetFinishStatus --> BuildPath
    KeepPosition --> BuildPath
    
    BuildPath --> End([Position Determined])
    
    style Start fill:#4CAF50,stroke:#2E7D32,color:#FFFFFF
    style End fill:#43A047,stroke:#1B5E20,color:#FFFFFF
    style ErrorHome fill:#E53935,stroke:#B71C1C,color:#FFFFFF
    style ErrorFinish fill:#E53935,stroke:#B71C1C,color:#FFFFFF
```

---

## Interaction Resolution

### Enemy Piece Interaction Flow

```mermaid
flowchart TD
    Start([Resolve Interactions]) --> BuildPath[Build Path Indices]
    BuildPath --> InitVars[Initialize Variables]
    
    InitVars --> LoopPass{Pass-Through Loop}
    
    LoopPass --> CheckEnemy{Enemy at Index?}
    CheckEnemy -->|No| NextPass[Next Index]
    CheckEnemy -->|Yes| CheckPiercing{Piercing Triggered?}
    
    CheckPiercing -->|Yes| Skip[Skip - Piercing Done]
    CheckPiercing -->|No| KickFirst[Kick First Enemy]
    
    KickFirst --> ResetPiece[Reset Piece to Home]
    ResetPiece --> RecordKick[Record Kick History]
    RecordKick --> SetPiercing[Set Piercing = true]
    
    SetPiercing --> NextPass
    Skip --> NextPass
    NextPass --> LoopPass
    
    LoopPass -->|All Indices| CheckLanding[Check Landing Position]
    
    CheckLanding --> LoopLanding{Landing Loop}
    LoopLanding --> CheckEnemyLanding{Enemies at Landing?}
    
    CheckEnemyLanding -->|No| EndLanding[End Landing Loop]
    CheckEnemyLanding -->|Yes| KickAll[Kick All Enemies]
    
    KickAll --> ResetPiece2[Reset Pieces to Home]
    ResetPiece2 --> RecordKick2[Record Kick Histories]
    RecordKick2 --> LoopLanding
    
    EndLanding --> ReturnResults([Return Results])
    
    style Start fill:#4CAF50,stroke:#2E7D32,color:#FFFFFF
    style ReturnResults fill:#43A047,stroke:#1B5E20,color:#FFFFFF
```

### Pass-Through vs Landing Interaction

```mermaid
graph TB
    Start([Piece Moving]) --> CalculatePath[Calculate Path]
    
    CalculatePath --> PathIndices[Path: fromIndex+1 to toIndex-1]
    CalculatePath --> LandingIndex[Landing: toIndex]
    
    PathIndices --> PassThrough[Pass-Through Check]
    LandingIndex --> LandingCheck[Landing Check]
    
    PassThrough --> FindFirst{Find First Enemy}
    FindFirst -->|Found| KickFirst[Kick First Enemy Only]
    FindFirst -->|Not Found| SkipPass[Skip]
    
    KickFirst --> SetPiercing[Set Piercing = true]
    SetPiercing --> ContinuePass{Continue?}
    ContinuePass -->|No| SkipRemaining[Skip Remaining]
    ContinuePass -->|Yes| NextIndex[Check Next Index]
    
    NextIndex --> CheckPiercingFlag{Piercing Triggered?}
    CheckPiercingFlag -->|Yes| SkipPass
    CheckPiercingFlag -->|No| FindFirst
    
    LandingCheck --> FindAll{Find All Enemies}
    FindAll -->|Found| KickAll[Kick All Enemies]
    FindAll -->|Not Found| NoAction[No Action]
    
    KickAll --> RecordAll[Record All Kicks]
    SkipPass --> RecordPass[Record Pass Kicks]
    
    RecordAll --> Combine([Combine Results])
    RecordPass --> Combine
    
    Combine --> End([Interaction Complete])
    
    style Start fill:#4CAF50,stroke:#2E7D32,color:#FFFFFF
    style End fill:#43A047,stroke:#1B5E20,color:#FFFFFF
```

---

## Sequence Diagrams

### Complete Turn Sequence

```mermaid
sequenceDiagram
    participant Client
    participant Handler
    participant RollUC as Roll UseCase
    participant MoveUC as Move UseCase
    participant GameState as GameState Repo
    participant Match as Match Repo
    participant Piece as Piece Repo
    
    Client->>Handler: POST /roll-dice
    Handler->>RollUC: Execute(RollInput)
    RollUC->>Match: GetByID(matchID)
    RollUC->>GameState: GetByMatchID(matchID)
    RollUC->>RollUC: Validate Turn
    RollUC->>RollUC: Calculate Weighted Roll
    RollUC->>RollUC: Save Roll History
    RollUC->>Match: Update(match)
    RollUC->>GameState: Update(gameState)
    RollUC-->>Handler: RollHistoryDTO
    Handler-->>Client: 200 OK + Roll Result
    
    Client->>Handler: POST /move
    Handler->>MoveUC: Execute(MoveInput)
    MoveUC->>Match: GetByID(matchID)
    MoveUC->>MoveUC: Get Roll History
    MoveUC->>Piece: GetByMatchID(matchID)
    MoveUC->>MoveUC: Find Active Piece
    MoveUC->>MoveUC: Calculate Position
    MoveUC->>MoveUC: Resolve Interactions
    MoveUC->>Piece: Update(piece)
    MoveUC->>Piece: Create Race History
    MoveUC->>Piece: Create Kick History
    MoveUC->>Match: Update(match)
    MoveUC-->>Handler: Success
    Handler-->>Client: 200 OK + Move Result
    
    Client->>Handler: GET /state
    Handler->>GameState: GetByMatchID(matchID)
    GameState-->>Handler: GameState
    Handler-->>Client: 200 OK + Game State
```

### Game Initialization Sequence

```mermaid
sequenceDiagram
    participant Client
    participant Handler
    participant InitUC as Init UseCase
    participant Match as Match Repo
    participant GameState as GameState Repo
    participant Piece as Piece Repo
    participant Balance as Balance Repo
    participant Redis as Redis Cache
    
    Client->>Handler: POST /initialize
    Handler->>InitUC: Execute(InitInput)
    InitUC->>Match: GetByID(matchID)
    InitUC->>Match: GetMatchPlayers(matchID)
    InitUC->>InitUC: Validate (4 players)
    InitUC->>InitUC: Randomize Player Order
    InitUC->>InitUC: Initialize Board State
    InitUC->>InitUC: Create Game State
    InitUC->>InitUC: Initialize Pieces
    InitUC->>InitUC: Initialize Balances
    
    InitUC->>GameState: Create(gameState)
    InitUC->>Piece: CreateMany(pieces)
    InitUC->>Balance: Create(balance) x4
    InitUC->>Redis: Create Dice Pool
    
    InitUC-->>Handler: Success
    Handler-->>Client: 200 OK + Initialized
```

### Piece Interaction Sequence

```mermaid
sequenceDiagram
    participant MoveUC as Move UseCase
    participant PieceRepo as Piece Repo
    participant KickRepo as Kick History Repo
    participant RaceRepo as Race History Repo
    
    MoveUC->>MoveUC: Build Path Indices
    MoveUC->>MoveUC: Loop Pass-Through Indices
    
    loop For each pass-through index
        MoveUC->>MoveUC: Find Enemy at Index
        alt Enemy Found & Piercing Not Triggered
            MoveUC->>MoveUC: Reset Piece to Home
            MoveUC->>PieceRepo: Update(enemyPiece)
            MoveUC->>RaceRepo: Create(resetHistory)
            MoveUC->>KickRepo: Create(kickHistory)
            MoveUC->>MoveUC: Set Piercing = true
        end
    end
    
    MoveUC->>MoveUC: Check Landing Position
    MoveUC->>MoveUC: Find All Enemies at Landing
    
    loop For each enemy at landing
        MoveUC->>MoveUC: Reset Piece to Home
        MoveUC->>PieceRepo: Update(enemyPiece)
        MoveUC->>RaceRepo: Create(resetHistory)
        MoveUC->>KickRepo: Create(kickHistory)
    end
    
    MoveUC->>MoveUC: Update Active Piece
    MoveUC->>PieceRepo: Update(activePiece)
    MoveUC->>RaceRepo: Create(moveHistory)
```

---

## Board Layout Diagram

### 56-Cell Board Structure

```mermaid
graph TB
    subgraph "Board Layout - 56 Cells"
        P0[Seat 0: Red<br/>HomeIndex: 0<br/>FinishIndex: 13]
        P1[Seat 1: Blue<br/>HomeIndex: 14<br/>FinishIndex: 27]
        P2[Seat 2: Yellow<br/>HomeIndex: 28<br/>FinishIndex: 41]
        P3[Seat 3: Purple<br/>HomeIndex: 42<br/>FinishIndex: 55]
        
        P0 -->|0-13| Cells0[Cells 0-13]
        P1 -->|14-27| Cells1[Cells 14-27]
        P2 -->|28-41| Cells2[Cells 28-41]
        P3 -->|42-55| Cells3[Cells 42-55]
        
        Cells0 -->|14 cells| Cells1
        Cells1 -->|14 cells| Cells2
        Cells2 -->|14 cells| Cells3
        Cells3 -->|14 cells| Cells0
    end
    
    style P0 fill:#EF5350,stroke:#B71C1C,color:#FFFFFF
    style P1 fill:#29B6F6,stroke:#0277BD,color:#FFFFFF
    style P2 fill:#FFD54F,stroke:#F57F17,color:#111111
    style P3 fill:#AB47BC,stroke:#6A1B9A,color:#FFFFFF
```

---

## Doubles Logic Diagram

### Double Handling Flow

```mermaid
flowchart TD
    RollDice[Roll Dice] --> CheckDouble{Is Double?}
    
    CheckDouble -->|No| ResetDouble[Reset DoubleCount = 0]
    ResetDouble --> NormalTurn[Normal Turn]
    
    CheckDouble -->|Yes| IncrementDouble[Increment DoubleCount]
    IncrementDouble --> CheckCount{DoubleCount < 3?}
    
    CheckCount -->|Yes| ExtraTurn[Extra Turn<br/>Player Continues]
    ExtraTurn --> RollDice
    
    CheckCount -->|No| PenaltyTurn[Penalty<br/>DoubleCount = 3<br/>End Turn]
    PenaltyTurn --> EndTurn[End Turn]
    
    NormalTurn --> EndTurn
    EndTurn --> NextPlayer[Next Player]
    
    style RollDice fill:#4CAF50,stroke:#2E7D32,color:#FFFFFF
    style ExtraTurn fill:#FFB300,stroke:#F57F17,color:#111111
    style PenaltyTurn fill:#E53935,stroke:#B71C1C,color:#FFFFFF
    style NextPlayer fill:#1976D2,stroke:#0D47A1,color:#FFFFFF
```

---

## Timer Flow Diagram

### Turn Timer Management

```mermaid
stateDiagram-v2
    [*] --> TimerStart: Turn Begins
    
    TimerStart: Timer = 300s
    TimerStart: IsOvertime = false
    
    TimerStart --> TimerRunning: Timer Active
    
    TimerRunning: Countdown Active
    TimerRunning: TimerSeconds--
    
    TimerRunning --> TimerExpired: TimerSeconds <= 0
    
    TimerExpired: Timer Expired
    TimerExpired: IsOvertime = true
    
    TimerExpired --> Overtime: Continue in Overtime
    
    Overtime: Overtime Mode
    Overtime: Player Can Still Play
    
    Overtime --> TurnEnd: Turn Completes
    TimerRunning --> TurnEnd: Turn Completes
    
    TurnEnd: Turn Ends
    TurnEnd: Reset Timer
    
    TurnEnd --> [*]: Next Turn
```

---

## Notes

### How to View These Diagrams

1. **GitHub/GitLab**: Diagrams render automatically in markdown files
2. **VS Code**: Install "Markdown Preview Mermaid Support" extension
3. **Online**: Use [Mermaid Live Editor](https://mermaid.live)
4. **Documentation Sites**: Most support Mermaid natively

### Diagram Types Used

- **Flowchart**: Linear and branching processes
- **State Diagram**: State machines and transitions
- **Sequence Diagram**: Interactions between components
- **Graph**: Network structures and relationships

