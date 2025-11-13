# Initialize Game Use Case Documentation

## Overview

The `InitializeGameUseCase` is responsible for initializing a new game of Ludo (cá ngựa). It sets up the game board, pieces, player balances, and game state before gameplay begins.

## Purpose

This use case prepares all necessary game entities and data structures when a match transitions to the started state, ensuring the game is ready for players to begin playing.

## Location

**File:** `internal/modules/gameplay/application/use-cases/initialize_game.go`

**Layer:** Application Layer (Use Cases)

## Dependencies

### Repositories Required

- `GameStateRepository` - Manages game state persistence
- `PlayerBalanceRepository` - Manages player balance data
- `PieceStateRepository` - Manages piece state data
- `MatchRepository` - Retrieves match information
- `MatchPlayerRepository` - Retrieves match player information

## Input

```go
type InitializeGameInput struct {
    MatchID string // ID of the match to initialize
}
```

## Execution Flow

### Step 1: Input Validation
- Validates that `MatchID` is not empty

### Step 2: Retrieve Match
- Fetches match from database using `MatchID`
- Validates match exists
- Verifies match status is `MatchStatusStarted`

### Step 3: Validate Players
- Retrieves all players in the match
- Validates exactly 4 players are present (required)

### Step 4: Randomize Player Order
- Shuffles player order using Fisher-Yates algorithm
- Ensures fair and random turn order
- Returns shuffled list of UserIDs

### Step 5: Initialize Board State
- Creates board with 56 cells total
- 4 directions, 14 cells each
- Each player assigned:
  - HomeIndex (starting position: 0, 14, 28, 42)
  - FinishIndex (end position: 13, 27, 41, 55)
  - Color (Red, Blue, Yellow, Purple)
  - 2 pieces starting in home

### Step 6: Create Game State
- Initializes game state with:
  - `CurrentTurn = 1`
  - `CurrentPlayerSeatIndex = 0`
  - `Status = GameStatusInProgress`
  - `TimerSeconds = 300` (5 minutes)
  - `DoubleCount = 0`
  - `IsOvertime = false`
  - `PlayerOrder` (shuffled)
  - `BoardState`

### Step 7: Initialize Pieces
- Creates 8 pieces total (2 per player)
- All pieces start:
  - Status: `PieceStatusInHome`
  - Position: HomeIndex
  - Assigned to respective player

### Step 8: Initialize Player Balances
- Calculates bet per player: `totalBetAmount / 4`
- Each player receives:
  - `InitialBet = betPerPlayer`
  - `CurrentBalance = betPerPlayer`
  - `LootedAmount = 0`
  - `IsBankrupt = false`

### Step 9: Persist to Database
- Saves GameState
- Saves all 8 pieces (CreateMany)
- Saves all 4 player balances

## Board Layout

### Cell Structure
- **Total Cells:** 56
- **Cells Per Direction:** 14
- **Home Cells Per Player:** 4
- **Finish Cell Per Player:** 1

### Player Positions

| Seat | Color  | HomeIndex | FinishIndex | Range     |
|------|--------|-----------|-------------|-----------|
| 0    | Red    | 0         | 13          | 0-13      |
| 1    | Blue   | 14        | 27          | 14-27     |
| 2    | Yellow | 28        | 41          | 28-41     |
| 3    | Purple | 42        | 55          | 42-55     |

### Board Visualization

```
        [42-55] Purple Finish
           ↓
[28-41] Yellow ← → [0-13] Red
           ↓
        [14-27] Blue Finish
```

### Position Calculation

```go
homeIndex := seatIndex * 14
finishIndex := ((seatIndex + 1) * 14) - 1
```

## Helper Functions

### `randomizePlayerOrder`

**Purpose:** Randomly shuffles the order of players to ensure fair turn order.

**Algorithm:** Fisher-Yates shuffle

**Input:** `[]*MatchPlayer`

**Output:** `[]string` (shuffled UserIDs)

**Implementation:**
- Creates array of UserIDs from match players
- Uses `rand.New(rand.NewSource(time.Now().UnixNano()))` for randomization
- Shuffles using Fisher-Yates algorithm
- Returns shuffled UserIDs

### `initializeBoardState`

**Purpose:** Creates the game board structure with 56 cells and assigns positions to players.

**Input:**
- `matchPlayers []*MatchPlayer`
- `playerOrder []string`
- `totalBetAmount int64`

**Output:** `BoardState`

**Logic:**
1. Creates `BoardState` with 56 total cells
2. Defines 4 colors (Red, Blue, Yellow, Purple)
3. For each player in order:
   - Calculates HomeIndex and FinishIndex
   - Assigns color based on seat index
   - Creates 2 pieces starting in home position

### `initializePieces`

**Purpose:** Creates `PieceState` entities for all 8 pieces in the game.

**Input:**
- `matchID string`
- `matchPlayers []*MatchPlayer`
- `boardState BoardState`

**Output:** `[]*PieceState` (8 pieces)

**Logic:**
1. Iterates through all players in board state
2. For each player's pieces:
   - Creates `PieceState` entity
   - Sets initial status to `PieceStatusInHome`
   - Sets position to HomeIndex
   - Assigns color and indices

### `initializePlayerBalances`

**Purpose:** Creates balance records for all players with equal initial bets.

**Input:**
- `matchID string`
- `matchPlayers []*MatchPlayer`
- `totalBetAmount int64`

**Output:** `[]*PlayerBalance` (4 balances)

**Logic:**
1. Calculates `betPerPlayer = totalBetAmount / 4`
2. For each player:
   - Creates `PlayerBalance` entity
   - Sets `InitialBet = betPerPlayer`
   - Sets `CurrentBalance = betPerPlayer`
   - Initializes `LootedAmount = 0`
   - Sets `IsBankrupt = false`

## Data Created

After successful execution, the following data is created:

1. **1 GameState** record
   - Match information
   - Turn and player tracking
   - Board state snapshot

2. **8 PieceState** records
   - 2 pieces per player
   - All in home position initially

3. **4 PlayerBalance** records
   - One balance per player
   - Equal initial bets

## Error Cases

### Input Validation Errors
- `ErrInvalidMatchID` - MatchID is empty

### Match Errors
- `ErrMatchNotFound` - Match does not exist
- `ErrMatchNotStarted` - Match is not in started status

### Player Errors
- "Game requires exactly 4 players" - Invalid player count

### State Validation Errors
- "Invalid game state" - GameState validation fails

### Database Errors
- "Failed to create game state" - GameState creation fails
- "Failed to create piece states" - Piece creation fails
- "Failed to create player balance" - Balance creation fails

## Important Notes

1. **Dice Pool:** Dice pool is managed in Redis cache, not created in database during initialization.

2. **Fixed Player Count:** Game requires exactly 4 players. This is a hard requirement.

3. **Random Order:** Player order is randomized each game to ensure fairness.

4. **Initial State:** All pieces start in home position (`PieceStatusInHome`).

5. **Equal Distribution:** Initial bets are divided equally among all 4 players.

6. **Timer:** Default game timer is set to 300 seconds (5 minutes) per turn.

7. **Turn Start:** Game starts with turn 1, current player at seat index 0.

## Usage Example

```go
// Initialize use case with dependencies
useCase := NewInitializeGameUseCase(InitializeGameConfig{
    GameStateRepository:     gameStateRepo,
    PlayerBalanceRepository: balanceRepo,
    PieceStateRepository:    pieceRepo,
    MatchRepository:         matchRepo,
    MatchPlayerRepository:   playerRepo,
})

// Execute initialization
input := InitializeGameInput{
    MatchID: "match-123",
}

err := useCase.Execute(ctx, input)
if err != nil {
    // Handle error
}
```

## Related Entities

- `GameState` - Main game state entity
- `BoardState` - Board structure and layout
- `PieceState` - Individual piece states
- `PlayerBalance` - Player financial state
- `Match` - Match entity (from match module)
- `MatchPlayer` - Match player entity (from match module)

## Architecture Notes

This use case follows the DDD (Domain-Driven Design) pattern:

- **Application Layer:** Contains business logic and orchestration
- **Domain Layer:** Uses domain entities and repositories
- **Infrastructure Layer:** Implements repository interfaces
- **Dependency Injection:** Uses constructor injection pattern

The use case coordinates between multiple repositories to create a complete game setup, ensuring data consistency across all game entities.

