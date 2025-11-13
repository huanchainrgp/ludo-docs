# Gameplay API Integration Guide

Tài liệu này mô tả cách tích hợp với Gameplay API của hệ thống Ludo Backend. API này cung cấp các endpoints để quản lý trạng thái game, xúc xắc, quân cờ, và số dư người chơi.

## Table of Contents

1. [Overview](#overview)
2. [Authentication](#authentication)
3. [Base URL](#base-url)
4. [API Endpoints](#api-endpoints)
5. [Data Transfer Objects (DTOs)](#data-transfer-objects-dtos)
6. [Error Handling](#error-handling)
7. [Turn Management](#turn-management)
8. [Integration Examples](#integration-examples)
9. [Best Practices](#best-practices)

## Overview

Gameplay API cung cấp các chức năng:
- **Game Initialization**: Khởi tạo game state, board, pieces, và dice pool
- **Dice Rolling**: Tung xúc xắc với weighted pool system
- **Game State Management**: Lấy trạng thái game, board, và timer
- **Piece Management**: Quản lý vị trí và trạng thái các quân cờ
- **Player Balance**: Theo dõi số dư và cược của người chơi
- **Roll History**: Lịch sử các lần tung xúc xắc

## Authentication

Tất cả các endpoints yêu cầu JWT authentication. Gửi token trong header:

```
Authorization: Bearer <your_jwt_token>
```

## Base URL

```
Production: https://api.ludo-game.com
Development: http://localhost:3003
```

## API Endpoints

### 1. Initialize Game

Khởi tạo game state, board, pieces, và dice pool cho một match.

**Endpoint:**
```
POST /gameplay/matches/{match_id}/initialize
```

**Path Parameters:**
- `match_id` (string, required): ID của match cần khởi tạo

**Headers:**
```
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**
Không có request body (empty body)

**Response:**
```json
{
  "message": "Game initialized successfully",
  "match_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Status Codes:**
- `200 OK`: Game initialized successfully
- `400 Bad Request`: Invalid match_id or game already initialized
- `401 Unauthorized`: Missing or invalid JWT token
- `404 Not Found`: Match not found
- `500 Internal Server Error`: Server error

**Example:**
```bash
curl -X POST \
  http://localhost:3003/gameplay/matches/550e8400-e29b-41d4-a716-446655440000/initialize \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json"
```

---

### 2. Roll Dice

Tung xúc xắc cho lượt chơi hiện tại của người chơi sử dụng weighted pool system.

**Endpoint:**
```
POST /gameplay/matches/{match_id}/roll-dice
```

**Path Parameters:**
- `match_id` (string, required): ID của match

**Headers:**
```
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**
Không có request body (user_id được lấy từ JWT token)

**Response:**
```json
{
  "id": "roll-123",
  "match_id": "550e8400-e29b-41d4-a716-446655440000",
  "user_id": "user-123",
  "turn": 1,
  "roll_value": [3, 4],
  "result": 7,
  "created_at": 1704067200000,
  "updated_at": 1704067200000
}
```

**Status Codes:**
- `200 OK`: Dice rolled successfully
- `400 Bad Request`: Invalid match_id, not player's turn, or already rolled
- `401 Unauthorized`: Missing or invalid JWT token
- `404 Not Found`: Match or game state not found
- `500 Internal Server Error`: Server error

**Example:**
```bash
curl -X POST \
  http://localhost:3003/gameplay/matches/550e8400-e29b-41d4-a716-446655440000/roll-dice \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json"
```

**Notes:**
- Chỉ người chơi có lượt hiện tại mới có thể tung xúc xắc
- Mỗi lượt chỉ có thể tung xúc xắc một lần
- Kết quả được tính từ weighted dice pool system

---

### 3. Get Game State

Lấy trạng thái game hiện tại bao gồm board, timer, và thông tin người chơi.

**Endpoint:**
```
GET /gameplay/matches/{match_id}/state
```

**Path Parameters:**
- `match_id` (string, required): ID của match

**Headers:**
```
Authorization: Bearer <token>
```

**Response:**
```json
{
  "id": "game-state-123",
  "match_id": "550e8400-e29b-41d4-a716-446655440000",
  "room_id": "room-123",
  "status": "IN_PROGRESS",
  "current_turn": 5,
  "current_player_seat_index": 2,
  "timer_seconds": 30,
  "is_overtime": false,
  "double_count": 1,
  "player_order": ["user-1", "user-2", "user-3", "user-4"],
  "board_state": {
    "total_cells": 52,
    "cells_per_direction": 13,
    "home_cells_per_player": 4,
    "finish_cell_per_player": 6,
    "players": [
      {
        "user_id": "user-1",
        "seat_index": 0,
        "color": "red",
        "home_index": 0,
        "finish_index": 0,
        "pieces": [
          {
            "piece_id": "piece-1",
            "position": 5,
            "status": "in_track"
          }
        ]
      }
    ]
  },
  "created_at": 1704067200000,
  "updated_at": 1704067200000
}
```

**Status Codes:**
- `200 OK`: Game state retrieved successfully
- `400 Bad Request`: Invalid match_id
- `401 Unauthorized`: Missing or invalid JWT token
- `404 Not Found`: Game state not found
- `500 Internal Server Error`: Server error

**Example:**
```bash
curl -X GET \
  http://localhost:3003/gameplay/matches/550e8400-e29b-41d4-a716-446655440000/state \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

---

### 4. Get Player Balances

Lấy số dư của tất cả người chơi trong match.

**Endpoint:**
```
GET /gameplay/matches/{match_id}/balances
```

**Path Parameters:**
- `match_id` (string, required): ID của match

**Headers:**
```
Authorization: Bearer <token>
```

**Response:**
```json
[
  {
    "id": "balance-123",
    "match_id": "550e8400-e29b-41d4-a716-446655440000",
    "user_id": "user-1",
    "initial_bet": 10000,
    "current_balance": 8500,
    "looted_amount": 1500,
    "is_bankrupt": false,
    "created_at": 1704067200000,
    "updated_at": 1704067200000
  },
  {
    "id": "balance-124",
    "match_id": "550e8400-e29b-41d4-a716-446655440000",
    "user_id": "user-2",
    "initial_bet": 10000,
    "current_balance": 12000,
    "looted_amount": 2000,
    "is_bankrupt": false,
    "created_at": 1704067200000,
    "updated_at": 1704067200000
  }
]
```

**Status Codes:**
- `200 OK`: Player balances retrieved successfully
- `400 Bad Request`: Invalid match_id
- `401 Unauthorized`: Missing or invalid JWT token
- `404 Not Found`: Match not found
- `500 Internal Server Error`: Server error

**Example:**
```bash
curl -X GET \
  http://localhost:3003/gameplay/matches/550e8400-e29b-41d4-a716-446655440000/balances \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

---

### 5. Get Piece States

Lấy trạng thái (vị trí) của tất cả các quân cờ trong match.

**Endpoint:**
```
GET /gameplay/matches/{match_id}/pieces
```

**Path Parameters:**
- `match_id` (string, required): ID của match

**Headers:**
```
Authorization: Bearer <token>
```

**Response:**
```json
[
  {
    "id": "piece-123",
    "match_id": "550e8400-e29b-41d4-a716-446655440000",
    "user_id": "user-1",
    "status": "in_track",
    "color": "red",
    "position": 15,
    "home_index": 0,
    "finish_index": 0,
    "created_at": 1704067200000,
    "updated_at": 1704067200000
  },
  {
    "id": "piece-124",
    "match_id": "550e8400-e29b-41d4-a716-446655440000",
    "user_id": "user-1",
    "status": "in_home",
    "color": "red",
    "position": 0,
    "home_index": 0,
    "finish_index": 0,
    "created_at": 1704067200000,
    "updated_at": 1704067200000
  }
]
```

**Status Codes:**
- `200 OK`: Piece states retrieved successfully
- `400 Bad Request`: Invalid match_id
- `401 Unauthorized`: Missing or invalid JWT token
- `404 Not Found`: Match not found
- `500 Internal Server Error`: Server error

**Example:**
```bash
curl -X GET \
  http://localhost:3003/gameplay/matches/550e8400-e29b-41d4-a716-446655440000/pieces \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

---

### 6. Get Roll History

Lấy lịch sử tất cả các lần tung xúc xắc trong match.

**Endpoint:**
```
GET /gameplay/matches/{match_id}/rolls
```

**Path Parameters:**
- `match_id` (string, required): ID của match

**Headers:**
```
Authorization: Bearer <token>
```

**Response:**
```json
[
  {
    "id": "roll-123",
    "match_id": "550e8400-e29b-41d4-a716-446655440000",
    "user_id": "user-1",
    "turn": 1,
    "roll_value": [3, 4],
    "result": 7,
    "created_at": 1704067200000,
    "updated_at": 1704067200000
  },
  {
    "id": "roll-124",
    "match_id": "550e8400-e29b-41d4-a716-446655440000",
    "user_id": "user-2",
    "turn": 2,
    "roll_value": [6, 6],
    "result": 12,
    "created_at": 1704067300000,
    "updated_at": 1704067300000
  }
]
```

**Status Codes:**
- `200 OK`: Roll history retrieved successfully
- `400 Bad Request`: Invalid match_id
- `401 Unauthorized`: Missing or invalid JWT token
- `404 Not Found`: Match not found
- `500 Internal Server Error`: Server error

**Example:**
```bash
curl -X GET \
  http://localhost:3003/gameplay/matches/550e8400-e29b-41d4-a716-446655440000/rolls \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

---

### 7. Get Dice Pool Config

Lấy cấu hình weighted dice pool system.

**Endpoint:**
```
GET /gameplay/dice-pool/config
```

**Headers:**
```
Authorization: Bearer <token>
```

**Response:**
```json
{
  "pool_size": 100,
  "weights": {
    "2": 1,
    "3": 2,
    "4": 3,
    "5": 4,
    "6": 5,
    "7": 6,
    "8": 5,
    "9": 4,
    "10": 3,
    "11": 2,
    "12": 1
  },
  "regeneration_threshold": 20
}
```

**Status Codes:**
- `200 OK`: Config retrieved successfully
- `401 Unauthorized`: Missing or invalid JWT token
- `404 Not Found`: Config not found
- `500 Internal Server Error`: Server error

**Example:**
```bash
curl -X GET \
  http://localhost:3003/gameplay/dice-pool/config \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

---

## Data Transfer Objects (DTOs)

### RollHistoryDTO

```typescript
interface RollHistoryDTO {
  id: string;
  match_id: string;
  user_id: string;
  turn: number;
  roll_value: number[]; // Array of dice values, e.g., [3, 4]
  result: number; // Sum of roll_value
  created_at: number; // Unix timestamp in milliseconds
  updated_at: number; // Unix timestamp in milliseconds
}
```

### GameStateDTO

```typescript
interface GameStateDTO {
  id: string;
  match_id: string;
  room_id: string;
  status: "WAITING" | "STARTING" | "IN_PROGRESS" | "FINISHED" | "CANCELLED";
  current_turn: number;
  current_player_seat_index: number;
  timer_seconds: number;
  is_overtime: boolean;
  double_count: number;
  player_order: string[]; // Array of user IDs in turn order
  board_state: BoardStateDTO;
  created_at: number; // Unix timestamp in milliseconds
  updated_at: number; // Unix timestamp in milliseconds
}

interface BoardStateDTO {
  total_cells: number;
  cells_per_direction: number;
  home_cells_per_player: number;
  finish_cell_per_player: number;
  players: PlayerBoardStateDTO[];
}

interface PlayerBoardStateDTO {
  user_id: string;
  seat_index: number;
  color: "red" | "blue" | "yellow" | "purple";
  home_index: number;
  finish_index: number;
  pieces: PieceBoardStateDTO[];
}

interface PieceBoardStateDTO {
  piece_id: string;
  position: number;
  status: "in_home" | "in_track" | "in_finish";
}
```

### PieceStateDTO

```typescript
interface PieceStateDTO {
  id: string;
  match_id: string;
  user_id: string;
  status: "in_home" | "in_track" | "in_finish";
  color: "red" | "blue" | "yellow" | "purple";
  position: number;
  home_index: number;
  finish_index: number;
  created_at: number; // Unix timestamp in milliseconds
  updated_at: number; // Unix timestamp in milliseconds
}
```

### PlayerBalanceDTO

```typescript
interface PlayerBalanceDTO {
  id: string;
  match_id: string;
  user_id: string;
  initial_bet: number; // Initial bet amount in smallest currency unit
  current_balance: number; // Current balance in smallest currency unit
  looted_amount: number; // Total amount looted from other players
  is_bankrupt: boolean;
  created_at: number; // Unix timestamp in milliseconds
  updated_at: number; // Unix timestamp in milliseconds
}
```

---

## Error Handling

Tất cả các lỗi trả về theo format:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable error message",
    "details": "Additional error details (optional)"
  }
}
```

### Common Error Codes

- `BAD_REQUEST`: Invalid request parameters
- `UNAUTHORIZED`: Missing or invalid authentication token
- `FORBIDDEN`: User doesn't have permission
- `NOT_FOUND`: Resource not found
- `INTERNAL_SERVER_ERROR`: Server error

### Example Error Response

```json
{
  "error": {
    "code": "BAD_REQUEST",
    "message": "Not player's turn",
    "details": "Current turn belongs to user-2"
  }
}
```

---

## Turn Management

Turn management là một phần quan trọng của gameplay. Mỗi match có một lượt chơi hiện tại (`current_turn`) và chỉ người chơi có lượt mới có thể thực hiện các hành động. Phần này mô tả chi tiết cách quản lý turn, các quy tắc, edge cases, và best practices.

### Turn Structure

Game state chứa thông tin về turn:
- `current_turn`: Số lượt hiện tại (bắt đầu từ 1, tăng lên sau mỗi vòng)
- `current_player_seat_index`: Chỉ số của người chơi đang có lượt (0-3)
- `player_order`: Mảng thứ tự người chơi (được randomize khi initialize game)
- `timer_seconds`: Thời gian còn lại cho lượt hiện tại (mặc định 300 giây = 5 phút)
- `double_count`: Số lần double liên tiếp (0-3)
- `is_overtime`: Có đang trong thời gian overtime không

### Turn Flow và Lifecycle

#### 1. Turn Initialization
Khi game được initialize:
- `current_turn` = 1
- `current_player_seat_index` = 0 (người chơi đầu tiên)
- `timer_seconds` = 300 (5 phút)
- `double_count` = 0

#### 2. Turn Sequence
Mỗi turn bao gồm các bước:
1. **Roll Dice**: Người chơi tung xúc xắc (chỉ khi đến lượt)
2. **Move Piece**: Người chơi di chuyển quân cờ (sau khi roll)
3. **Turn Completion**: Turn tự động kết thúc sau khi move
4. **Turn Advancement**: Chuyển sang người chơi tiếp theo

#### 3. Turn Advancement

Turn tự động advance theo thứ tự:
- Sau khi người chơi hoàn thành move
- Sau khi timer hết hạn
- Sau 3 lần double liên tiếp

**Công thức advance:**
```typescript
// Pseudo-code
nextSeatIndex = (currentSeatIndex + 1) % 4
if (nextSeatIndex === 0) {
  currentTurn++  // Một vòng mới
}
```

### Checking Current Turn

Để kiểm tra lượt hiện tại, lấy game state:

```typescript
const gameState = await gameplayApi.getGameState(matchId);

// Check if it's your turn
const currentPlayerId = gameState.player_order[gameState.current_player_seat_index];
const isMyTurn = currentPlayerId === userId;

if (isMyTurn) {
  console.log(`It's your turn! Turn ${gameState.current_turn}`);
  console.log(`Time remaining: ${gameState.timer_seconds} seconds`);
} else {
  console.log(`Waiting for player ${currentPlayerId}...`);
  console.log(`Current turn: ${gameState.current_turn}`);
}
```

### Turn Validation

Luôn validate turn trước khi thực hiện action:

```typescript
async function validateTurn(matchId: string, userId: string): Promise<void> {
  const gameState = await gameplayApi.getGameState(matchId);
  
  // Check game status
  if (gameState.status !== 'IN_PROGRESS') {
    throw new Error(`Game is not in progress. Status: ${gameState.status}`);
  }
  
  // Check if it's player's turn
  const currentPlayerId = gameState.player_order[gameState.current_player_seat_index];
  if (currentPlayerId !== userId) {
    throw new Error(`Not your turn. Current player: ${currentPlayerId}`);
  }
  
  // Check timer (optional - server will handle timeout)
  if (gameState.timer_seconds <= 0 && !gameState.is_overtime) {
    throw new Error('Turn timer has expired');
  }
}

// Before rolling dice
await validateTurn(matchId, userId);
await gameplayApi.rollDice(matchId);
```

### Double Dice Rules

Khi người chơi roll được double (6-6):
- `double_count` tăng lên 1
- Người chơi có thể tiếp tục lượt (không advance turn)
- Có thể roll dice lại ngay lập tức
- Tối đa 3 lần double liên tiếp

**Logic:**
```typescript
// Check if can continue turn after double
function canContinueTurn(gameState: GameStateDTO): boolean {
  return gameState.double_count > 0 && gameState.double_count < 3;
}

// After rolling double
const rollResult = await gameplayApi.rollDice(matchId);
const gameState = await gameplayApi.getGameState(matchId);

if (rollResult.roll_value[0] === 6 && rollResult.roll_value[1] === 6) {
  if (canContinueTurn(gameState)) {
    console.log(`Double! You can roll again. Double count: ${gameState.double_count}`);
    // Player can roll again immediately
  } else {
    console.log('Max doubles reached (3). Turn will advance after move.');
  }
}
```

### Timer Management

Mỗi turn có timer 300 giây (5 phút):
- Timer được reset khi turn advance
- Khi timer hết, turn tự động advance
- Overtime mode: Sau khi timer hết, có thể có thời gian overtime

**Polling Timer:**
```typescript
// Poll timer every second
function startTimerPolling(matchId: string) {
  const interval = setInterval(async () => {
    const gameState = await gameplayApi.getGameState(matchId);
    
    if (gameState.status !== 'IN_PROGRESS') {
      clearInterval(interval);
      return;
    }
    
    updateTimerDisplay(gameState.timer_seconds);
    
    // Warn when time is running out
    if (gameState.timer_seconds <= 30 && gameState.timer_seconds > 0) {
      showTimeWarning(gameState.timer_seconds);
    }
    
    // Check if timer expired
    if (gameState.timer_seconds <= 0) {
      console.log('Timer expired, turn will advance');
      // Poll for turn change
      await waitForTurnChange(matchId);
    }
  }, 1000); // Poll every second
}
```

### Complete Turn Manager Class

```typescript
class TurnManager {
  private gameState: GameStateDTO | null = null;
  private userId: string;
  private pollingInterval: NodeJS.Timeout | null = null;

  constructor(userId: string) {
    this.userId = userId;
  }

  // Check if it's current player's turn
  async isMyTurn(matchId: string): Promise<boolean> {
    this.gameState = await gameplayApi.getGameState(matchId);
    const currentPlayerId = this.gameState.player_order[
      this.gameState.current_player_seat_index
    ];
    return currentPlayerId === this.userId;
  }

  // Get current player info
  getCurrentPlayer(): { user_id: string; seat_index: number } | null {
    if (!this.gameState) return null;
    
    const seatIndex = this.gameState.current_player_seat_index;
    const userId = this.gameState.player_order[seatIndex];
    
    return { user_id: userId, seat_index: seatIndex };
  }

  // Get turn information
  getTurnInfo(): {
    turn: number;
    player_index: number;
    player_id: string;
    timer_seconds: number;
    double_count: number;
  } | null {
    if (!this.gameState) return null;
    
    return {
      turn: this.gameState.current_turn,
      player_index: this.gameState.current_player_seat_index,
      player_id: this.gameState.player_order[this.gameState.current_player_seat_index],
      timer_seconds: this.gameState.timer_seconds,
      double_count: this.gameState.double_count
    };
  }

  // Check if can continue turn (double dice)
  canContinueTurn(): boolean {
    if (!this.gameState) return false;
    return this.gameState.double_count > 0 && this.gameState.double_count < 3;
  }

  // Wait for turn (polling)
  async waitForTurn(
    matchId: string,
    maxWaitTime: number = 300000,
    pollInterval: number = 2000
  ): Promise<void> {
    const startTime = Date.now();
    let lastTurn = -1;
    
    while (Date.now() - startTime < maxWaitTime) {
      const gameState = await gameplayApi.getGameState(matchId);
      
      // Check if turn changed
      if (gameState.current_turn !== lastTurn) {
        lastTurn = gameState.current_turn;
        const currentPlayerId = gameState.player_order[gameState.current_player_seat_index];
        
        if (currentPlayerId === this.userId) {
          this.gameState = gameState;
          return; // It's my turn!
        }
      }
      
      await sleep(pollInterval);
    }
    
    throw new Error('Timeout waiting for turn');
  }

  // Start polling for turn changes
  startPolling(matchId: string, onTurnChange: (turnInfo: any) => void) {
    if (this.pollingInterval) {
      this.stopPolling();
    }
    
    this.pollingInterval = setInterval(async () => {
      try {
        const gameState = await gameplayApi.getGameState(matchId);
        
        // Check if turn changed
        if (this.gameState && 
            (this.gameState.current_turn !== gameState.current_turn ||
             this.gameState.current_player_seat_index !== gameState.current_player_seat_index)) {
          
          const turnInfo = {
            turn: gameState.current_turn,
            player_index: gameState.current_player_seat_index,
            player_id: gameState.player_order[gameState.current_player_seat_index],
            timer_seconds: gameState.timer_seconds
          };
          
          onTurnChange(turnInfo);
        }
        
        this.gameState = gameState;
      } catch (error) {
        console.error('Error polling turn:', error);
      }
    }, 2000); // Poll every 2 seconds
  }

  // Stop polling
  stopPolling() {
    if (this.pollingInterval) {
      clearInterval(this.pollingInterval);
      this.pollingInterval = null;
    }
  }

  // Validate turn before action
  async validateTurn(matchId: string): Promise<void> {
    const gameState = await gameplayApi.getGameState(matchId);
    
    if (gameState.status !== 'IN_PROGRESS') {
      throw new Error(`Game is not in progress. Status: ${gameState.status}`);
    }
    
    const currentPlayerId = gameState.player_order[gameState.current_player_seat_index];
    if (currentPlayerId !== this.userId) {
      throw new Error(`Not your turn. Current player: ${currentPlayerId}`);
    }
    
    this.gameState = gameState;
  }
}

// Usage Example
const turnManager = new TurnManager('user-123');

// Start polling for turn changes
turnManager.startPolling(matchId, (turnInfo) => {
  console.log(`Turn changed: ${turnInfo.turn}, Player: ${turnInfo.player_id}`);
  
  if (turnInfo.player_id === 'user-123') {
    enablePlayerActions();
  } else {
    disablePlayerActions();
  }
});

// Before action, validate turn
try {
  await turnManager.validateTurn(matchId);
  await gameplayApi.rollDice(matchId);
} catch (error) {
  console.error('Cannot perform action:', error.message);
}
```

### Turn Polling Strategy

Để theo dõi turn changes, sử dụng polling:

```typescript
// Smart polling - only poll when needed
class TurnPoller {
  private matchId: string;
  private userId: string;
  private interval: NodeJS.Timeout | null = null;
  private lastTurn: number = -1;
  private lastPlayerIndex: number = -1;

  constructor(matchId: string, userId: string) {
    this.matchId = matchId;
    this.userId = userId;
  }

  start(onTurnChange: (isMyTurn: boolean, turnInfo: any) => void) {
    this.interval = setInterval(async () => {
      try {
        const gameState = await gameplayApi.getGameState(this.matchId);
        
        // Check if turn or player changed
        const turnChanged = gameState.current_turn !== this.lastTurn;
        const playerChanged = gameState.current_player_seat_index !== this.lastPlayerIndex;
        
        if (turnChanged || playerChanged) {
          const currentPlayerId = gameState.player_order[gameState.current_player_seat_index];
          const isMyTurn = currentPlayerId === this.userId;
          
          const turnInfo = {
            turn: gameState.current_turn,
            player_id: currentPlayerId,
            player_index: gameState.current_player_seat_index,
            timer_seconds: gameState.timer_seconds,
            double_count: gameState.double_count
          };
          
          onTurnChange(isMyTurn, turnInfo);
          
          this.lastTurn = gameState.current_turn;
          this.lastPlayerIndex = gameState.current_player_seat_index;
        }
      } catch (error) {
        console.error('Error polling turn:', error);
      }
    }, 2000); // Poll every 2 seconds
  }

  stop() {
    if (this.interval) {
      clearInterval(this.interval);
      this.interval = null;
    }
  }
}

// Usage
const poller = new TurnPoller(matchId, userId);

poller.start((isMyTurn, turnInfo) => {
  if (isMyTurn) {
    console.log(`Your turn! Turn ${turnInfo.turn}`);
    enablePlayerActions();
  } else {
    console.log(`Player ${turnInfo.player_id}'s turn`);
    disablePlayerActions();
  }
  
  updateTurnIndicator(turnInfo);
});
```

### Turn State Machine

Turn có các states:
1. **Waiting for Roll**: Đang chờ người chơi roll dice
2. **Rolled**: Đã roll dice, đang chờ move
3. **Moving**: Đang di chuyển quân cờ
4. **Completed**: Đã hoàn thành, chờ advance
5. **Advanced**: Đã chuyển sang người chơi tiếp theo

**Flow:**
```
Waiting for Roll → Roll Dice → Rolled → Move Piece → Completed → Advanced → (Next Player)
                                                      ↓
                                              (Double Dice)
                                                      ↓
                                              Waiting for Roll (same player)
```

### Best Practices for Turn Management

1. **Always Validate Before Action**
   ```typescript
   // ❌ Bad: No validation
   await gameplayApi.rollDice(matchId);
   
   // ✅ Good: Validate first
   await turnManager.validateTurn(matchId);
   await gameplayApi.rollDice(matchId);
   ```

2. **Poll Efficiently**
   ```typescript
   // ❌ Bad: Poll too frequently
   setInterval(() => getGameState(), 100); // Every 100ms
   
   // ✅ Good: Poll every 2 seconds
   setInterval(() => getGameState(), 2000);
   ```

3. **Handle Turn Changes Gracefully**
   ```typescript
   // Show clear UI indication
   if (isMyTurn) {
     highlightCurrentPlayer();
     enableActionButtons();
     startTurnTimer();
   } else {
     dimPlayerArea();
     disableActionButtons();
     showWaitingMessage();
   }
   ```

4. **Cache Game State**
   ```typescript
   // Cache to reduce API calls
   let cachedGameState: GameStateDTO | null = null;
   let lastUpdateTime = 0;
   const CACHE_TTL = 1000; // 1 second
   
   async function getCachedGameState(matchId: string): Promise<GameStateDTO> {
     const now = Date.now();
     if (cachedGameState && (now - lastUpdateTime) < CACHE_TTL) {
       return cachedGameState;
     }
     
     cachedGameState = await gameplayApi.getGameState(matchId);
     lastUpdateTime = now;
     return cachedGameState;
   }
   ```

### Turn Advancement Scenarios

Có nhiều scenarios khác nhau khiến turn advance:

#### Scenario 1: Normal Turn Completion
Sau khi người chơi roll dice và move piece thành công, turn tự động advance:

```typescript
// Normal flow
async function completeTurn(matchId: string, userId: string) {
  // 1. Roll dice
  const rollResult = await gameplayApi.rollDice(matchId);
  
  // 2. Move piece
  await movePiece(matchId, targetIndex);
  
  // 3. Turn automatically advances
  // Server calls gameState.NextPlayer() internally
  // current_player_seat_index increments
  // If back to player 0, current_turn increments
}
```

#### Scenario 2: Timer Expiry
Khi timer hết, turn tự động advance:

```typescript
// Timer expiry handling
async function handleTimerExpiry(matchId: string) {
  const gameState = await gameplayApi.getGameState(matchId);
  
  if (gameState.timer_seconds <= 0 && !gameState.is_overtime) {
    // Server automatically advances turn
    // Poll for updated game state
    await waitForTurnChange(matchId);
    
    const newState = await gameplayApi.getGameState(matchId);
    console.log(`Turn advanced due to timer expiry. New turn: ${newState.current_turn}`);
  }
}
```

#### Scenario 3: Max Doubles Reached
Sau 3 lần double liên tiếp, turn sẽ advance sau khi move:

```typescript
// Double dice limit
async function handleDoubleDice(matchId: string) {
  let rollResult = await gameplayApi.rollDice(matchId);
  let gameState = await gameplayApi.getGameState(matchId);
  
  let doubleCount = 0;
  while (rollResult.roll_value[0] === 6 && rollResult.roll_value[1] === 6) {
    doubleCount++;
    console.log(`Double! Count: ${doubleCount}/3`);
    
    if (doubleCount >= 3) {
      console.log('Max doubles reached. Turn will advance after move.');
      // After move, turn will advance
      break;
    }
    
    // Can roll again immediately
    rollResult = await gameplayApi.rollDice(matchId);
    gameState = await gameplayApi.getGameState(matchId);
  }
  
  // Move piece
  await movePiece(matchId, targetIndex);
  
  // If doubleCount >= 3, turn advances
  // Otherwise, turn continues (if double)
}
```

#### Scenario 4: No Valid Moves
Khi người chơi không có nước đi hợp lệ (tất cả quân cờ đã về đích hoặc không thể di chuyển), turn vẫn advance sau khi roll:

```typescript
// No valid moves scenario
async function handleNoValidMoves(matchId: string, userId: string) {
  try {
    // Try to roll dice
    const rollResult = await gameplayApi.rollDice(matchId);
    
    // Check if any piece can move
    const pieces = await gameplayApi.getPieceStates(matchId);
    const myPieces = pieces.filter(p => p.user_id === userId);
    
    const canMove = myPieces.some(piece => {
      // Check if piece can move with current roll
      if (piece.status === 'in_home' && rollResult.result < 6) {
        return false; // Need 6 to leave home
      }
      if (piece.status === 'in_finish') {
        return false; // Already finished
      }
      // Check if move would exceed finish line
      const newPosition = piece.position + rollResult.result;
      return newPosition <= piece.finish_index;
    });
    
    if (!canMove) {
      console.log('No valid moves available. Turn will advance.');
      // Server will advance turn automatically
      // No move action needed
    } else {
      // Move available piece
      await movePiece(matchId, targetIndex);
    }
  } catch (error) {
    if (error.message.includes('no active piece')) {
      console.log('No active pieces. Turn will advance.');
    }
  }
}
```

### Turn Validation và Error Handling

#### Common Turn Errors

1. **Not Player's Turn**
```typescript
try {
  await gameplayApi.rollDice(matchId);
} catch (error) {
  if (error.response?.status === 400 && 
      error.response?.data?.error?.message?.includes('not player\'s turn')) {
    console.log('It is not your turn yet');
    // Wait for turn
    await turnManager.waitForTurn(matchId);
  }
}
```

2. **Game Not In Progress**
```typescript
try {
  await gameplayApi.rollDice(matchId);
} catch (error) {
  if (error.response?.data?.error?.message?.includes('not in progress')) {
    console.log('Game is not in progress');
    // Check game status
    const gameState = await gameplayApi.getGameState(matchId);
    console.log(`Game status: ${gameState.status}`);
  }
}
```

3. **Already Rolled This Turn**
```typescript
try {
  await gameplayApi.rollDice(matchId);
} catch (error) {
  if (error.response?.data?.error?.message?.includes('already rolled')) {
    console.log('You have already rolled this turn');
    // Get roll result
    const rollHistory = await gameplayApi.getRollHistory(matchId);
    const currentRoll = rollHistory.find(r => r.turn === currentTurn);
    console.log('Current roll:', currentRoll);
  }
}
```

4. **Timer Expired**
```typescript
try {
  await gameplayApi.rollDice(matchId);
} catch (error) {
  if (error.response?.data?.error?.message?.includes('timer expired')) {
    console.log('Turn timer has expired');
    // Turn will advance automatically
    await waitForNextTurn(matchId);
  }
}
```

### Complete Turn Flow với Error Handling

```typescript
class RobustTurnManager {
  private gameState: GameStateDTO | null = null;
  private userId: string;
  private matchId: string;
  private retryCount: number = 0;
  private maxRetries: number = 3;

  constructor(userId: string, matchId: string) {
    this.userId = userId;
    this.matchId = matchId;
  }

  // Complete turn flow with error handling
  async executeTurn(): Promise<boolean> {
    try {
      // Step 1: Validate turn
      await this.validateTurn();
      
      // Step 2: Roll dice
      const rollResult = await this.rollDiceWithRetry();
      
      // Step 3: Check for double
      const isDouble = rollResult.roll_value[0] === 6 && 
                      rollResult.roll_value[1] === 6;
      
      if (isDouble) {
        await this.handleDoubleDice(rollResult);
      } else {
        // Step 4: Move piece
        await this.movePiece(rollResult);
        
        // Step 5: Turn advances automatically
        await this.waitForTurnAdvancement();
      }
      
      return true;
    } catch (error) {
      console.error('Turn execution error:', error);
      return this.handleTurnError(error);
    }
  }

  private async validateTurn(): Promise<void> {
    this.gameState = await gameplayApi.getGameState(this.matchId);
    
    // Check game status
    if (this.gameState.status !== 'IN_PROGRESS') {
      throw new Error(`Game is not in progress. Status: ${this.gameState.status}`);
    }
    
    // Check if it's player's turn
    const currentPlayerId = this.gameState.player_order[
      this.gameState.current_player_seat_index
    ];
    
    if (currentPlayerId !== this.userId) {
      throw new Error(`Not your turn. Current player: ${currentPlayerId}`);
    }
    
    // Check timer
    if (this.gameState.timer_seconds <= 0 && !this.gameState.is_overtime) {
      throw new Error('Turn timer has expired');
    }
  }

  private async rollDiceWithRetry(): Promise<RollHistoryDTO> {
    for (let i = 0; i < this.maxRetries; i++) {
      try {
        return await gameplayApi.rollDice(this.matchId);
      } catch (error) {
        if (i === this.maxRetries - 1) {
          throw error;
        }
        
        // Check if already rolled
        if (error.response?.data?.error?.message?.includes('already rolled')) {
          // Get existing roll
          const rollHistory = await gameplayApi.getRollHistory(this.matchId);
          const currentTurn = this.gameState?.current_turn || 1;
          const existingRoll = rollHistory.find(r => r.turn === currentTurn);
          
          if (existingRoll) {
            return existingRoll;
          }
        }
        
        // Wait and retry
        await sleep(1000);
        await this.validateTurn(); // Re-validate
      }
    }
    
    throw new Error('Failed to roll dice after retries');
  }

  private async handleDoubleDice(rollResult: RollHistoryDTO): Promise<void> {
    this.gameState = await gameplayApi.getGameState(this.matchId);
    
    if (this.gameState.double_count >= 3) {
      console.log('Max doubles reached. Must move and turn will advance.');
      await this.movePiece(rollResult);
      await this.waitForTurnAdvancement();
    } else {
      console.log(`Double! Count: ${this.gameState.double_count}/3. You can roll again.`);
      // Player can roll again immediately
      // Don't advance turn yet
    }
  }

  private async movePiece(rollResult: RollHistoryDTO): Promise<void> {
    // Get available pieces
    const pieces = await gameplayApi.getPieceStates(this.matchId);
    const myPieces = pieces.filter(p => p.user_id === this.userId);
    
    // Find valid move
    const validPiece = myPieces.find(piece => {
      if (piece.status === 'in_home' && rollResult.result < 6) {
        return false;
      }
      if (piece.status === 'in_finish') {
        return false;
      }
      const newPosition = piece.position + rollResult.result;
      return newPosition <= piece.finish_index;
    });
    
    if (!validPiece) {
      console.log('No valid moves. Turn will advance automatically.');
      // Server will advance turn
      return;
    }
    
    // Calculate target index
    let targetIndex = validPiece.position + rollResult.result;
    if (validPiece.status === 'in_home') {
      targetIndex = validPiece.home_index;
    }
    if (targetIndex > validPiece.finish_index) {
      targetIndex = validPiece.finish_index;
    }
    
    // Move piece
    try {
      await movePieceAPI(this.matchId, targetIndex);
    } catch (error) {
      if (error.response?.data?.error?.message?.includes('cannot move')) {
        console.log('Piece cannot move to target position');
        // Try another piece or skip
      } else {
        throw error;
      }
    }
  }

  private async waitForTurnAdvancement(): Promise<void> {
    const initialTurn = this.gameState?.current_turn || 0;
    const initialPlayerIndex = this.gameState?.current_player_seat_index || -1;
    
    // Poll until turn changes
    for (let i = 0; i < 30; i++) { // Max 60 seconds
      await sleep(2000);
      
      const newState = await gameplayApi.getGameState(this.matchId);
      
      if (newState.current_turn !== initialTurn || 
          newState.current_player_seat_index !== initialPlayerIndex) {
        console.log('Turn advanced!');
        this.gameState = newState;
        return;
      }
    }
    
    throw new Error('Timeout waiting for turn advancement');
  }

  private handleTurnError(error: any): boolean {
    const errorMessage = error.response?.data?.error?.message || error.message;
    
    if (errorMessage.includes('not player\'s turn')) {
      console.log('Not your turn. Waiting...');
      // Will be handled by polling
      return false;
    }
    
    if (errorMessage.includes('timer expired')) {
      console.log('Timer expired. Turn will advance.');
      // Turn will advance automatically
      return false;
    }
    
    if (errorMessage.includes('not in progress')) {
      console.log('Game is not in progress');
      return false;
    }
    
    // Other errors
    console.error('Unexpected error:', errorMessage);
    return false;
  }
}

// Usage
const turnManager = new RobustTurnManager('user-123', 'match-456');

// Execute turn
const success = await turnManager.executeTurn();
if (success) {
  console.log('Turn completed successfully');
}
```

### Turn Synchronization với WebSocket

Nếu sử dụng WebSocket, có thể lắng nghe turn change events:

```typescript
// WebSocket turn synchronization
class WebSocketTurnManager {
  private ws: WebSocket;
  private matchId: string;
  private userId: string;
  private onTurnChange: (turnInfo: any) => void;

  constructor(ws: WebSocket, matchId: string, userId: string) {
    this.ws = ws;
    this.matchId = matchId;
    this.userId = userId;
    this.setupWebSocketListeners();
  }

  private setupWebSocketListeners() {
    this.ws.on('message', (data: string) => {
      const message = JSON.parse(data);
      
      // Listen for turn change events
      if (message.type === 'turn_changed' || 
          message.type === 'game_state_updated') {
        const turnInfo = {
          turn: message.data.current_turn,
          player_id: message.data.current_player_id,
          player_index: message.data.current_player_seat_index,
          timer_seconds: message.data.timer_seconds
        };
        
        this.onTurnChange(turnInfo);
      }
      
      // Listen for dice roll events
      if (message.type === 'dice_rolled') {
        const isMyTurn = message.data.user_id === this.userId;
        if (isMyTurn) {
          console.log('You rolled:', message.data.result);
        }
      }
    });
  }

  setOnTurnChange(callback: (turnInfo: any) => void) {
    this.onTurnChange = callback;
  }
}

// Usage with WebSocket
const ws = new WebSocket('ws://localhost:3003/socket?token=...');
const wsTurnManager = new WebSocketTurnManager(ws, matchId, userId);

wsTurnManager.setOnTurnChange((turnInfo) => {
  if (turnInfo.player_id === userId) {
    console.log('Your turn!');
    enablePlayerActions();
  } else {
    console.log(`Player ${turnInfo.player_id}'s turn`);
    disablePlayerActions();
  }
});
```

### Turn Calculation và Prediction

Có thể tính toán turn tiếp theo:

```typescript
// Calculate next turn info
function calculateNextTurn(currentTurn: number, currentPlayerIndex: number, playerOrder: string[]): {
  nextTurn: number;
  nextPlayerIndex: number;
  nextPlayerId: string;
} {
  const nextPlayerIndex = (currentPlayerIndex + 1) % playerOrder.length;
  const nextTurn = nextPlayerIndex === 0 ? currentTurn + 1 : currentTurn;
  const nextPlayerId = playerOrder[nextPlayerIndex];
  
  return {
    nextTurn,
    nextPlayerIndex,
    nextPlayerId
  };
}

// Usage
const gameState = await gameplayApi.getGameState(matchId);
const nextTurnInfo = calculateNextTurn(
  gameState.current_turn,
  gameState.current_player_seat_index,
  gameState.player_order
);

console.log(`Next turn: ${nextTurnInfo.nextTurn}, Player: ${nextTurnInfo.nextPlayerId}`);
```

### Turn Statistics

Theo dõi thống kê về turn:

```typescript
class TurnStatistics {
  private matchId: string;
  private stats: {
    totalTurns: number;
    playerTurns: Map<string, number>;
    averageTurnDuration: number;
    longestTurn: number;
    shortestTurn: number;
  };

  constructor(matchId: string) {
    this.matchId = matchId;
    this.stats = {
      totalTurns: 0,
      playerTurns: new Map(),
      averageTurnDuration: 0,
      longestTurn: 0,
      shortestTurn: Infinity
    };
  }

  async trackTurn(playerId: string, startTime: number) {
    const endTime = Date.now();
    const duration = endTime - startTime;
    
    this.stats.totalTurns++;
    
    const playerTurnCount = this.stats.playerTurns.get(playerId) || 0;
    this.stats.playerTurns.set(playerId, playerTurnCount + 1);
    
    // Update statistics
    this.stats.averageTurnDuration = 
      (this.stats.averageTurnDuration * (this.stats.totalTurns - 1) + duration) / 
      this.stats.totalTurns;
    
    if (duration > this.stats.longestTurn) {
      this.stats.longestTurn = duration;
    }
    
    if (duration < this.stats.shortestTurn) {
      this.stats.shortestTurn = duration;
    }
  }

  getStatistics() {
    return {
      ...this.stats,
      playerTurns: Object.fromEntries(this.stats.playerTurns)
    };
  }
}

// Usage
const turnStats = new TurnStatistics(matchId);

// Track turn start
const turnStartTime = Date.now();

// After turn completes
await turnStats.trackTurn(userId, turnStartTime);

// Get statistics
const stats = turnStats.getStatistics();
console.log('Turn statistics:', stats);
```

### Advanced Turn Management Patterns

#### Pattern 1: Turn Queue
Quản lý queue các actions trong turn:

```typescript
class TurnActionQueue {
  private queue: Array<() => Promise<void>> = [];
  private isProcessing: boolean = false;

  async addAction(action: () => Promise<void>) {
    this.queue.push(action);
    if (!this.isProcessing) {
      await this.processQueue();
    }
  }

  private async processQueue() {
    this.isProcessing = true;
    
    while (this.queue.length > 0) {
      const action = this.queue.shift();
      try {
        await action();
      } catch (error) {
        console.error('Action failed:', error);
        // Handle error
      }
    }
    
    this.isProcessing = false;
  }
}

// Usage
const actionQueue = new TurnActionQueue();

// Add actions to queue
await actionQueue.addAction(async () => {
  await gameplayApi.rollDice(matchId);
});

await actionQueue.addAction(async () => {
  await movePiece(matchId, targetIndex);
});
```

#### Pattern 2: Turn State Machine với Events

```typescript
enum TurnState {
  WAITING = 'waiting',
  ROLLING = 'rolling',
  ROLLED = 'rolled',
  MOVING = 'moving',
  COMPLETED = 'completed'
}

class TurnStateMachine {
  private state: TurnState = TurnState.WAITING;
  private listeners: Map<TurnState, Array<() => void>> = new Map();

  on(state: TurnState, callback: () => void) {
    if (!this.listeners.has(state)) {
      this.listeners.set(state, []);
    }
    this.listeners.get(state)!.push(callback);
  }

  transition(newState: TurnState) {
    const oldState = this.state;
    this.state = newState;
    
    // Trigger listeners
    const callbacks = this.listeners.get(newState) || [];
    callbacks.forEach(cb => cb());
    
    console.log(`Turn state: ${oldState} -> ${newState}`);
  }

  getState(): TurnState {
    return this.state;
  }
}

// Usage
const stateMachine = new TurnStateMachine();

stateMachine.on(TurnState.ROLLED, () => {
  console.log('Dice rolled, ready to move');
  enableMoveButtons();
});

stateMachine.on(TurnState.COMPLETED, () => {
  console.log('Turn completed');
  disableAllButtons();
});

// Transition states
stateMachine.transition(TurnState.ROLLING);
await gameplayApi.rollDice(matchId);
stateMachine.transition(TurnState.ROLLED);
```

### Summary: Turn Management Checklist

Khi implement turn management, đảm bảo:

- [ ] Validate turn trước mọi action
- [ ] Handle double dice logic (max 3)
- [ ] Poll game state định kỳ (2 giây)
- [ ] Handle timer expiry
- [ ] Handle no valid moves scenario
- [ ] Error handling cho tất cả edge cases
- [ ] Cache game state để giảm API calls
- [ ] UI feedback rõ ràng về turn status
- [ ] WebSocket integration (nếu có)
- [ ] Turn statistics tracking (optional)


---

## Integration Examples

### TypeScript/JavaScript Example

```typescript
import axios, { AxiosInstance } from 'axios';

class GameplayApiService {
  private client: AxiosInstance;
  private baseURL: string;
  private token: string | null = null;

  constructor(baseURL: string = 'http://localhost:3003') {
    this.baseURL = baseURL;
    this.client = axios.create({
      baseURL,
      headers: {
        'Content-Type': 'application/json',
      },
    });

    // Add request interceptor for auth token
    this.client.interceptors.request.use((config) => {
      if (this.token) {
        config.headers.Authorization = `Bearer ${this.token}`;
      }
      return config;
    });
  }

  setToken(token: string) {
    this.token = token;
  }

  // Initialize game
  async initializeGame(matchId: string): Promise<void> {
    const response = await this.client.post(
      `/gameplay/matches/${matchId}/initialize`
    );
    return response.data;
  }

  // Roll dice
  async rollDice(matchId: string): Promise<RollHistoryDTO> {
    const response = await this.client.post(
      `/gameplay/matches/${matchId}/roll-dice`
    );
    return response.data;
  }

  // Get game state
  async getGameState(matchId: string): Promise<GameStateDTO> {
    const response = await this.client.get(
      `/gameplay/matches/${matchId}/state`
    );
    return response.data;
  }

  // Get player balances
  async getPlayerBalances(matchId: string): Promise<PlayerBalanceDTO[]> {
    const response = await this.client.get(
      `/gameplay/matches/${matchId}/balances`
    );
    return response.data;
  }

  // Get piece states
  async getPieceStates(matchId: string): Promise<PieceStateDTO[]> {
    const response = await this.client.get(
      `/gameplay/matches/${matchId}/pieces`
    );
    return response.data;
  }

  // Get roll history
  async getRollHistory(matchId: string): Promise<RollHistoryDTO[]> {
    const response = await this.client.get(
      `/gameplay/matches/${matchId}/rolls`
    );
    return response.data;
  }

  // Get dice pool config
  async getDicePoolConfig(): Promise<any> {
    const response = await this.client.get('/gameplay/dice-pool/config');
    return response.data;
  }
}

// Usage
const gameplayApi = new GameplayApiService('http://localhost:3003');
gameplayApi.setToken('your-jwt-token');

// Initialize game
await gameplayApi.initializeGame('match-123');

// Roll dice
const rollResult = await gameplayApi.rollDice('match-123');
console.log(`Rolled: ${rollResult.result}`);

// Get game state
const gameState = await gameplayApi.getGameState('match-123');
console.log(`Current turn: ${gameState.current_turn}`);
```

### Game Flow Example

```typescript
async function playGame(matchId: string, userId: string) {
  const gameplayApi = new GameplayApiService();
  gameplayApi.setToken(getAuthToken());

  try {
    // 1. Initialize game (only once, before match starts)
    await gameplayApi.initializeGame(matchId);
    console.log('Game initialized');

    // 2. Get initial game state
    let gameState = await gameplayApi.getGameState(matchId);
    console.log(`Game status: ${gameState.status}`);

    // 3. Game loop
    while (gameState.status === 'IN_PROGRESS') {
      // Check if it's current player's turn
      const currentPlayerId = gameState.player_order[gameState.current_player_seat_index];
      
      if (currentPlayerId === userId) {
        // It's my turn - roll dice
        const rollResult = await gameplayApi.rollDice(matchId);
        console.log(`You rolled: ${rollResult.result}`);
        
        // Then make move decision...
      } else {
        // Wait for other player's turn
        console.log(`Waiting for player ${currentPlayerId}...`);
      }

      // Poll for game state updates
      await sleep(2000); // Poll every 2 seconds
      gameState = await gameplayApi.getGameState(matchId);
    }

    // Game finished
    console.log(`Game finished with status: ${gameState.status}`);
    
    // Get final balances
    const balances = await gameplayApi.getPlayerBalances(matchId);
    console.log('Final balances:', balances);

  } catch (error) {
    console.error('Game error:', error);
  }
}
```

---

## Best Practices

### 1. Error Handling

Luôn xử lý errors một cách graceful:

```typescript
try {
  const result = await gameplayApi.rollDice(matchId);
} catch (error) {
  if (error.response?.status === 400) {
    // Handle bad request (e.g., not player's turn)
    showError('It is not your turn');
  } else if (error.response?.status === 401) {
    // Handle unauthorized (token expired)
    refreshToken();
  } else {
    // Handle other errors
    showError('An error occurred');
  }
}
```

### 2. Polling Strategy

- **Use HTTP polling** để cập nhật game state định kỳ
- **Poll interval**: Không nên poll quá thường xuyên (recommended: 1-2 seconds)
- Poll game state khi game đang IN_PROGRESS
- Giảm polling frequency khi không có thay đổi

### 3. State Management

- Cache game state locally để giảm số lượng API calls
- Update cache sau mỗi API call thành công
- Sync với server định kỳ để đảm bảo consistency
- Invalidate cache khi có thay đổi quan trọng (turn change, dice roll)

### 4. Turn Management

- Luôn kiểm tra `current_player_seat_index` trước khi roll dice
- Disable UI cho các actions không hợp lệ
- Show clear indication về lượt chơi hiện tại

### 5. Performance

- Batch requests khi có thể
- Poll game state một cách thông minh (chỉ khi cần thiết)
- Cache static data (dice pool config, board configuration)
- Debounce polling requests để tránh spam server

### 6. Security

- Luôn validate JWT token trước khi gọi API
- Không expose token trong client-side code
- Refresh token khi hết hạn

---

## Testing

### Test Scenarios

1. **Initialize Game**
   - Initialize game successfully
   - Initialize game that already exists (should handle gracefully)
   - Initialize game with invalid match_id

2. **Roll Dice**
   - Roll dice on player's turn
   - Roll dice when not player's turn (should fail)
   - Roll dice twice in same turn (should fail)
   - Verify turn advancement after roll

3. **Get Game State**
   - Get game state for active match
   - Get game state for non-existent match
   - Verify all fields in response

4. **Get Player Balances**
   - Get balances for match with players
   - Verify balance calculations

5. **Get Piece States**
   - Get pieces for match
   - Verify piece positions and statuses

6. **Get Roll History**
   - Get history for match with rolls
   - Verify chronological order

---

## Support

Nếu có vấn đề hoặc câu hỏi, vui lòng liên hệ:
- Email: support@ludo-game.com
- Documentation: https://docs.ludo-game.com
- GitHub Issues: https://github.com/ludo-game/backend/issues

