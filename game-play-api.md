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

Turn management là một phần quan trọng của gameplay. Mỗi match có một lượt chơi hiện tại (`current_turn`) và chỉ người chơi có lượt mới có thể thực hiện các hành động.

### Turn Structure

Game state chứa thông tin về turn:
- `current_turn`: Số lượt hiện tại (bắt đầu từ 1)
- `current_player_seat_index`: Chỉ số của người chơi đang có lượt (0-3)
- `player_order`: Mảng thứ tự người chơi (được randomize khi initialize game)

### Checking Current Turn

Để kiểm tra lượt hiện tại, lấy game state:

```typescript
const gameState = await gameplayApi.getGameState(matchId);

// Check if it's your turn
const currentPlayerId = gameState.player_order[gameState.current_player_seat_index];
const isMyTurn = currentPlayerId === userId;

if (isMyTurn) {
  console.log(`It's your turn! Turn ${gameState.current_turn}`);
} else {
  console.log(`Waiting for player ${currentPlayerId}...`);
}
```

### Turn Flow

1. **Initialize Game**: Turn bắt đầu từ 1, `current_player_seat_index` = 0
2. **Roll Dice**: Chỉ người chơi có lượt mới có thể roll dice
3. **Move Piece**: Sau khi roll dice, người chơi có thể di chuyển quân cờ
4. **Advance Turn**: Turn tự động advance sau khi người chơi hoàn thành lượt (sau khi move hoặc skip)

### Turn Advancement Rules

- Turn tự động advance khi:
  - Người chơi hoàn thành move
  - Người chơi skip lượt (nếu không có nước đi hợp lệ)
  - Timer hết hạn
- Double dice (6-6): Người chơi có thể tiếp tục lượt (không advance turn)
- Max 3 doubles: Sau 3 lần double liên tiếp, turn sẽ advance

### Example: Turn Management

```typescript
class TurnManager {
  private gameState: GameStateDTO | null = null;
  private userId: string;

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

  // Wait for turn (polling)
  async waitForTurn(matchId: string, maxWaitTime: number = 300000): Promise<void> {
    const startTime = Date.now();
    
    while (Date.now() - startTime < maxWaitTime) {
      if (await this.isMyTurn(matchId)) {
        return;
      }
      await sleep(2000); // Poll every 2 seconds
    }
    
    throw new Error('Timeout waiting for turn');
  }

  // Get turn info
  getTurnInfo(): { turn: number; player_index: number; player_id: string } | null {
    if (!this.gameState) return null;
    
    return {
      turn: this.gameState.current_turn,
      player_index: this.gameState.current_player_seat_index,
      player_id: this.gameState.player_order[this.gameState.current_player_seat_index]
    };
  }
}

// Usage
const turnManager = new TurnManager('user-123');

// Check turn before action
if (await turnManager.isMyTurn(matchId)) {
  // Roll dice
  await gameplayApi.rollDice(matchId);
} else {
  const currentPlayer = turnManager.getCurrentPlayer();
  console.log(`Waiting for ${currentPlayer?.user_id}...`);
}
```

### Turn Validation

Trước khi thực hiện bất kỳ action nào, luôn validate turn:

```typescript
async function validateTurn(matchId: string, userId: string): Promise<void> {
  const gameState = await gameplayApi.getGameState(matchId);
  const currentPlayerId = gameState.player_order[gameState.current_player_seat_index];
  
  if (currentPlayerId !== userId) {
    throw new Error(`Not your turn. Current player: ${currentPlayerId}`);
  }
}

// Before rolling dice
await validateTurn(matchId, userId);
await gameplayApi.rollDice(matchId);
```


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

