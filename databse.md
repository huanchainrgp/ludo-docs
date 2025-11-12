# 🎮 Ludo Game - Tài Liệu Tích Hợp Backend API & WebSocket

Tài liệu kỹ thuật chi tiết để tích hợp Backend API và WebSocket vào Client Game (Cocos2D Creator).

## 📋 Mục Lục

1. [Tổng Quan Hệ Thống](#tổng-quan-hệ-thống)
2. [Kiến Trúc Database](#kiến-trúc-database)
3. [HTTP API Endpoints](#http-api-endpoints)
4. [WebSocket Integration](#websocket-integration)
5. [Workflow Chi Tiết](#workflow-chi-tiết)
6. [Code Examples (React/TypeScript)](#code-examples-reacttypescript)
7. [Tích Hợp Cocos2D Creator](#tích-hợp-cocos2d-creator)
8. [Error Handling](#error-handling)
9. [State Management](#state-management)
10. [Testing & Debugging](#testing--debugging)

---

## 🎯 Tổng Quan Hệ Thống

### Base URLs
- **API Server**: `http://localhost:3003` (Development)
- **WebSocket**: `ws://localhost:3003/ws`
- **Production**: Thay đổi theo cấu hình server

### Kiến Trúc Tổng Quan

```
┌─────────────────┐
│  Cocos2D Client │
│  (Game Frontend) │
└────────┬────────┘
         │
         ├─── HTTP REST API ────┐
         │                      │
         └─── WebSocket ────────┤
                                │
                    ┌───────────▼───────────┐
                    │   Backend API Server   │
                    │   (Go/Gin Framework)   │
                    └───────────┬───────────┘
                                │
                ┌───────────────┼───────────────┐
                │               │               │
        ┌───────▼──────┐ ┌──────▼──────┐ ┌─────▼─────┐
        │  PostgreSQL   │ │    Redis    │ │   Queue   │
        │   Database    │ │   Cache     │ │  (Asynq)  │
        └───────────────┘ └─────────────┘ └───────────┘
```

### Module Structure

1. **IAM Module**: Authentication, User Management
2. **Wallet Module**: Balance Management, Transactions
3. **Room Module**: Room Creation, Join/Leave
4. **Match Module**: Match Management
5. **Gameplay Module**: Game Logic, Dice Rolling, Piece Movement
6. **System Config Module**: System Configuration

---

## 🗄️ Kiến Trúc Database

### Database Schema Overview

Hệ thống sử dụng PostgreSQL với các bảng chính:

### 1. Users Table (IAM Module)

**Table**: `users`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, DEFAULT gen_random_uuid() | User unique identifier |
| `username` | VARCHAR(255) | UNIQUE, NOT NULL | Username for login |
| `email` | VARCHAR(255) | UNIQUE, NOT NULL | User email address |
| `password_hash` | VARCHAR(255) | NOT NULL | Bcrypt hashed password |
| `name` | VARCHAR(255) | | User display name |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT 'active' | User status (active/inactive) |
| `token_version` | INTEGER | NOT NULL, DEFAULT 0 | Token version for logout |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Account creation time |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Last update time |

**Indexes**:
- `idx_users_username` ON `username`
- `idx_users_email` ON `email`
- `idx_users_status` ON `status`

**Relationships**:
- One-to-Many: `users` → `refresh_tokens`
- One-to-Many: `users` → `user_balances`
- One-to-Many: `users` → `user_rooms`
- One-to-Many: `users` → `match_players`

### 2. Refresh Tokens Table

**Table**: `refresh_tokens`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | Token unique identifier |
| `user_id` | UUID | NOT NULL, FK → users(id) | Reference to user |
| `token` | TEXT | NOT NULL, UNIQUE | Refresh token string |
| `expires_at` | TIMESTAMP | NOT NULL | Token expiration time |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Token creation time |

**Indexes**:
- `idx_refresh_tokens_user_id` ON `user_id`
- `idx_refresh_tokens_token` ON `token`
- `idx_refresh_tokens_expires_at` ON `expires_at`

### 3. User Balances Table (Wallet Module)

**Table**: `user_balances`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | Balance unique identifier |
| `user_id` | UUID | NOT NULL, UNIQUE, FK → users(id) | Reference to user |
| `balance` | BIGINT | NOT NULL, DEFAULT 0 | Current balance amount |
| `currency` | VARCHAR(10) | NOT NULL, DEFAULT 'USD' | Currency code |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Creation time |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Last update time |

**Indexes**:
- `idx_user_balances_user_id` ON `user_id`

### 4. Transactions Table

**Table**: `transactions`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | Transaction unique identifier |
| `user_id` | UUID | NOT NULL, FK → users(id) | Reference to user |
| `type` | VARCHAR(20) | NOT NULL | Transaction type (credit/debit) |
| `amount` | BIGINT | NOT NULL | Transaction amount |
| `balance_before` | BIGINT | NOT NULL | Balance before transaction |
| `balance_after` | BIGINT | NOT NULL | Balance after transaction |
| `reference_id` | VARCHAR(255) | | External reference ID |
| `metadata` | JSONB | | Additional transaction data |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT 'pending' | Transaction status |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Transaction time |

**Indexes**:
- `idx_transactions_user_id` ON `user_id`
- `idx_transactions_type` ON `type`
- `idx_transactions_status` ON `status`
- `idx_transactions_reference_id` ON `reference_id`

### 5. Rooms Table (Room Module)

**Table**: `rooms`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | Room unique identifier |
| `owner_id` | UUID | NOT NULL, FK → users(id) | Room owner user ID |
| `max_players` | INTEGER | NOT NULL, DEFAULT 4 | Maximum players allowed |
| `current_players` | INTEGER | NOT NULL, DEFAULT 1 | Current number of players |
| `is_public` | BOOLEAN | NOT NULL, DEFAULT TRUE | Public or private room |
| `password` | BIGINT | CHECK (1000-9999) | 4-digit password for private rooms |
| `bet_value` | BIGINT | NOT NULL, DEFAULT 0 | Bet amount per player |
| `minimum_amount` | BIGINT | NOT NULL, DEFAULT 0 | Minimum balance required |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT TRUE | Room active status |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Room creation time |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Last update time |

**Indexes**:
- `idx_rooms_current_players` ON `current_players`
- `idx_rooms_created_at` ON `created_at DESC`
- `idx_rooms_is_public` ON `is_public`
- `idx_rooms_bet_value` ON `bet_value`
- `idx_rooms_is_active` ON `is_active`

**Relationships**:
- One-to-Many: `rooms` → `user_rooms`
- One-to-Many: `rooms` → `matches`

### 6. User Rooms Table

**Table**: `user_rooms`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | User-room relationship ID |
| `user_id` | UUID | NOT NULL, FK → users(id) | Reference to user |
| `room_id` | UUID | NOT NULL, FK → rooms(id) | Reference to room |
| `seat_index` | INTEGER | NOT NULL | Player seat position (0-3) |
| `joined_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Join time |

**Indexes**:
- `idx_user_rooms_user_id` ON `user_id`
- `idx_user_rooms_room_id` ON `room_id`
- `UNIQUE(user_id, room_id)` - One user per room

### 7. Matches Table (Match Module)

**Table**: `matches`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | Match unique identifier |
| `room_id` | UUID | NOT NULL, FK → rooms(id) | Reference to room |
| `status` | VARCHAR(20) | NOT NULL | Match status (pending/started/finished/cancelled) |
| `max_players` | INTEGER | NOT NULL, DEFAULT 4 | Maximum players |
| `current_players` | INTEGER | NOT NULL, DEFAULT 0 | Current number of players |
| `total_bet_amount` | BIGINT | NOT NULL, DEFAULT 0 | Total bet amount |
| `current_turn` | INTEGER | NOT NULL, DEFAULT 1 | Current turn number |
| `current_player_seat_index` | INTEGER | NOT NULL, DEFAULT 0 | Current player seat |
| `started_at` | TIMESTAMP | | Match start time |
| `finished_at` | TIMESTAMP | | Match finish time |
| `winner_id` | UUID | FK → users(id) | Winner user ID |
| `has_rolled` | BOOLEAN | NOT NULL, DEFAULT FALSE | Has current turn rolled |
| `has_moved` | BOOLEAN | NOT NULL, DEFAULT FALSE | Has current turn moved |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Match creation time |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Last update time |

**Indexes**:
- `idx_matches_room_id` ON `room_id`
- `idx_matches_status` ON `status`
- `idx_matches_started_at` ON `started_at`

**Relationships**:
- One-to-Many: `matches` → `match_players`
- One-to-One: `matches` → `game_states`
- One-to-Many: `matches` → `roll_histories`

### 8. Match Players Table

**Table**: `match_players`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | Match-player relationship ID |
| `match_id` | UUID | NOT NULL, FK → matches(id) | Reference to match |
| `user_id` | UUID | NOT NULL, FK → users(id) | Reference to user |
| `seat_index` | INTEGER | NOT NULL | Player seat position (0-3) |
| `joined_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Join time |

**Indexes**:
- `idx_match_players_match_id` ON `match_id`
- `idx_match_players_user_id` ON `user_id`
- `UNIQUE(match_id, seat_index)` - One player per seat

### 9. Game States Table (Gameplay Module)

**Table**: `game_states`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | Game state unique identifier |
| `match_id` | UUID | NOT NULL, UNIQUE, FK → matches(id) | Reference to match |
| `room_id` | UUID | NOT NULL, FK → rooms(id) | Reference to room |
| `status` | VARCHAR(20) | NOT NULL | Game status (waiting/starting/in_progress/finished/cancelled) |
| `current_turn` | INTEGER | NOT NULL, DEFAULT 1 | Current turn number |
| `current_player_seat_index` | INTEGER | NOT NULL, DEFAULT 0 | Current player seat (0-3) |
| `timer_seconds` | INTEGER | NOT NULL, DEFAULT 300 | Remaining timer in seconds |
| `is_overtime` | BOOLEAN | NOT NULL, DEFAULT FALSE | Overtime mode flag |
| `double_count` | INTEGER | NOT NULL, DEFAULT 0 | Consecutive double count (max 3) |
| `player_order` | JSONB | | Array of user IDs in turn order |
| `board_state` | JSONB | | Complete board state (see BoardState structure) |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Creation time |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Last update time |

**Indexes**:
- `idx_game_states_match_id` ON `match_id`
- `idx_game_states_room_id` ON `room_id`
- `idx_game_states_status` ON `status`

**BoardState JSONB Structure**:
```json
{
  "total_cells": 56,
  "cells_per_direction": 14,
  "home_cells_per_player": 4,
  "finish_cell_per_player": 1,
  "players": [
    {
      "user_id": "uuid",
      "seat_index": 0,
      "color": "red",
      "home_index": 0,
      "finish_index": 13,
      "pieces": [
        {
          "piece_id": "uuid",
          "position": 0,
          "status": "in_home"
        }
      ]
    }
  ]
}
```

**PlayerOrder JSONB Structure**:
```json
["user-id-1", "user-id-2", "user-id-3", "user-id-4"]
```

### 10. Piece States Table

**Table**: `piece_states`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | Piece unique identifier |
| `match_id` | UUID | NOT NULL, FK → matches(id) | Reference to match |
| `user_id` | UUID | NOT NULL, FK → users(id) | Reference to player |
| `status` | VARCHAR(20) | NOT NULL | Piece status (in_home/in_track/in_finish) |
| `color` | VARCHAR(20) | NOT NULL | Piece color (red/blue/yellow/purple) |
| `position` | INTEGER | NOT NULL, DEFAULT 0 | Current cell position (0-55) |
| `home_index` | INTEGER | NOT NULL, DEFAULT 0 | Home starting position |
| `finish_index` | INTEGER | NOT NULL, DEFAULT 0 | Finish line position |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Creation time |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Last update time |

**Indexes**:
- `idx_piece_states_match_id` ON `match_id`
- `idx_piece_states_user_id` ON `user_id`
- `idx_piece_states_status` ON `status`
- `idx_piece_states_color` ON `color`

**Piece Status Values**:
- `in_home`: Piece is in home base (cannot move until rolled 6)
- `in_track`: Piece is on the outer track (can move and be kicked)
- `in_finish`: Piece is in finish line (cannot be kicked)

**Piece Colors**:
- `red`: Seat index 0 (Bottom-right in UI)
- `blue`: Seat index 1 (Top-right in UI)
- `yellow`: Seat index 2 (Top-left in UI)
- `purple`: Seat index 3 (Bottom-left in UI)

### 11. Player Balances Table

**Table**: `player_balances`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | Player balance unique identifier |
| `match_id` | UUID | NOT NULL, FK → matches(id) | Reference to match |
| `user_id` | UUID | NOT NULL, FK → users(id) | Reference to player |
| `initial_bet` | BIGINT | NOT NULL, DEFAULT 0 | Initial bet amount |
| `current_balance` | BIGINT | NOT NULL, DEFAULT 0 | Current balance in match |
| `looted_amount` | BIGINT | NOT NULL, DEFAULT 0 | Total looted from other players |
| `is_bankrupt` | BOOLEAN | NOT NULL, DEFAULT FALSE | Bankruptcy status |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Creation time |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Last update time |

**Indexes**:
- `idx_player_balances_match_id` ON `match_id`
- `idx_player_balances_user_id` ON `user_id`
- `idx_player_balances_is_bankrupt` ON `is_bankrupt`
- `UNIQUE(match_id, user_id)` - One balance per player per match

### 12. Roll Histories Table

**Table**: `roll_histories`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | Roll history unique identifier |
| `match_id` | UUID | NOT NULL, FK → matches(id) | Reference to match |
| `user_id` | UUID | NOT NULL, FK → users(id) | Reference to player |
| `turn` | INTEGER | NOT NULL | Turn number when rolled |
| `roll_value` | INTEGER[] | NOT NULL | Array of 2 dice values [dice1, dice2] |
| `result` | INTEGER | NOT NULL | Sum of dice values (2-12) |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Roll time |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Last update time |

**Indexes**:
- `idx_roll_histories_match_id` ON `match_id`
- `idx_roll_histories_user_id` ON `user_id`
- `idx_roll_histories_turn` ON `turn`
- `idx_roll_histories_match_id_turn` ON `(match_id, turn)` - Composite index

**Example roll_value**:
- `[3, 3]` → result: 6 (double)
- `[1, 5]` → result: 6 (not double)
- `[6, 6]` → result: 12 (double 6)

### 13. Piece Kick Histories Table

**Table**: `piece_kick_histories`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | Kick history unique identifier |
| `piece_id` | UUID | NOT NULL, FK → piece_states(id) | Reference to kicked piece |
| `kicked_by` | UUID | NOT NULL, FK → piece_states(id) | Reference to piece that kicked |
| `kick_value` | INTEGER | NOT NULL | Cell index where kick occurred |
| `turn` | INTEGER | NOT NULL | Turn number when kicked |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Kick time |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Last update time |

**Indexes**:
- `idx_piece_kick_histories_piece_id` ON `piece_id`
- `idx_piece_kick_histories_kicked_by` ON `kicked_by`
- `idx_piece_kick_histories_turn` ON `turn`

### 14. Piece Race Histories Table

**Table**: `piece_race_histories`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | Race history unique identifier |
| `piece_id` | UUID | NOT NULL, FK → piece_states(id) | Reference to piece |
| `from_index` | INTEGER | NOT NULL | Starting cell index |
| `to_index` | INTEGER | NOT NULL | Destination cell index |
| `turn` | INTEGER | NOT NULL | Turn number when moved |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Move time |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Last update time |

**Indexes**:
- `idx_piece_race_histories_piece_id` ON `piece_id`
- `idx_piece_race_histories_turn` ON `turn`

### 15. System Configs Table

**Table**: `system_configs`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | Config unique identifier |
| `key` | VARCHAR(255) | NOT NULL, UNIQUE | Configuration key |
| `value` | TEXT | NOT NULL | Configuration value (JSON) |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Creation time |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Last update time |

**Indexes**:
- `idx_system_configs_key` ON `key`

**Example Config Keys**:
- `dice_pool.enabled`: `"true"`
- `dice_pool.pool_size`: `"100"`
- `dice_pool.distribution.total_six.percentage`: `"20"`

### Database Relationships Diagram

```
users
  ├── refresh_tokens (1:N)
  ├── user_balances (1:1)
  ├── user_rooms (1:N)
  │     └── rooms (N:1)
  │           └── matches (1:N)
  │                 ├── match_players (1:N) → users
  │                 ├── game_states (1:1)
  │                 ├── roll_histories (1:N)
  │                 ├── piece_states (1:N)
  │                 │     ├── piece_kick_histories (1:N)
  │                 │     └── piece_race_histories (1:N)
  │                 └── player_balances (1:N)
```

---

## 🌐 HTTP API Endpoints

### Base URL
```
http://localhost:3003
```

### Authentication Header
Tất cả protected endpoints yêu cầu:
```
Authorization: Bearer <access_token>
```

### 1. IAM Module - Authentication

#### 1.1. User Registration
```http
POST /iam/register
Content-Type: application/json

{
  "username": "player123",
  "password": "password123",
  "email": "player123@example.com"
}
```

**Response 201**:
```json
{
  "data": {
    "id": "9c72f02d-9035-4991-a234-9c991280d35a",
    "username": "player123",
    "email": "player123@example.com",
    "created_at": 1699123456000,
    "updated_at": 1699123456000
  }
}
```

**Error Responses**:
- `400 BAD_REQUEST`: Username/email already exists, invalid format
- `500 INTERNAL_SERVER_ERROR`: Server error

#### 1.2. User Login
```http
POST /iam/login
Content-Type: application/json

{
  "username": "player123",
  "password": "password123"
}
```

**Response 200**:
```json
{
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expires_in": 900,
    "user": {
      "id": "9c72f02d-9035-4991-a234-9c991280d35a",
      "username": "player123",
      "email": "player123@example.com"
    }
  }
}
```

**Token Expiration**:
- `access_token`: 15 minutes (900 seconds)
- `refresh_token`: 7 days

#### 1.3. Refresh Token
```http
POST /iam/refresh
Content-Type: application/json

{
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response 200**:
```json
{
  "data": {
    "access_token": "new_access_token...",
    "expires_in": 900
  }
}
```

#### 1.4. Get User Profile
```http
GET /iam/profile
Authorization: Bearer <token>
```

**Response 200**:
```json
{
  "data": {
    "id": "9c72f02d-9035-4991-a234-9c991280d35a",
    "username": "player123",
    "email": "player123@example.com",
    "name": "Player Name",
    "status": "active",
    "created_at": 1699123456000,
    "updated_at": 1699123456000
  }
}
```

#### 1.5. Update Profile
```http
PUT /iam/profile
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "New Name",
  "email": "newemail@example.com"
}
```

### 2. Wallet Module

#### 2.1. Create User Balance
```http
POST /wallet/create-user-balance
Authorization: Bearer <token>
Content-Type: application/json

{
  "user_id": "9c72f02d-9035-4991-a234-9c991280d35a"
}
```

**Response 201**:
```json
{
  "data": {
    "id": "balance-uuid",
    "user_id": "9c72f02d-9035-4991-a234-9c991280d35a",
    "balance": 0,
    "currency": "USD",
    "created_at": 1699123456000,
    "updated_at": 1699123456000
  }
}
```

#### 2.2. Credit Balance
```http
POST /wallet/credit
Authorization: Bearer <token>
Content-Type: application/json

{
  "amount": 1000000,
  "reference_id": "initial_deposit",
  "metadata": {
    "source": "initial_deposit",
    "description": "Starting balance"
  }
}
```

**Response 200**:
```json
{
  "data": {
    "transaction_id": "txn-uuid",
    "user_id": "9c72f02d-9035-4991-a234-9c991280d35a",
    "type": "credit",
    "amount": 1000000,
    "balance_before": 0,
    "balance_after": 1000000,
    "status": "completed",
    "created_at": 1699123456000
  }
}
```

**Note**: Credit operations are processed asynchronously via queue. Response may return `status: "pending"` initially.

#### 2.3. Debit Balance
```http
POST /wallet/debit
Authorization: Bearer <token>
Content-Type: application/json

{
  "amount": 100000,
  "reference_id": "room_bet",
  "metadata": {
    "source": "room_bet",
    "room_id": "room-uuid"
  }
}
```

#### 2.4. Get Balance
```http
GET /wallet/balance
Authorization: Bearer <token>
```

**Response 200**:
```json
{
  "data": {
    "id": "balance-uuid",
    "user_id": "9c72f02d-9035-4991-a234-9c991280d35a",
    "balance": 1000000,
    "currency": "USD",
    "created_at": 1699123456000,
    "updated_at": 1699123456000
  }
}
```

#### 2.5. Get Transaction History
```http
GET /wallet/transactions?page=1&limit=20
Authorization: Bearer <token>
```

**Response 200**:
```json
{
  "data": [
    {
      "id": "txn-uuid",
      "type": "credit",
      "amount": 1000000,
      "balance_before": 0,
      "balance_after": 1000000,
      "reference_id": "initial_deposit",
      "status": "completed",
      "created_at": 1699123456000
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 1,
    "total_pages": 1
  }
}
```

### 3. Room Module

#### 3.1. Get Open Rooms
```http
GET /rooms/open?page=1&limit=20
```

**Response 200**:
```json
{
  "data": [
    {
      "id": "room-uuid",
      "owner_id": "owner-uuid",
      "max_players": 4,
      "current_players": 2,
      "is_public": true,
      "bet_value": 100000,
      "minimum_amount": 100000,
      "is_active": true,
      "created_at": 1699123456000,
      "updated_at": 1699123456000
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 10,
    "total_pages": 1
  }
}
```

#### 3.2. Get Room Users
```http
GET /rooms/{room_id}/users
```

**Response 200**:
```json
{
  "data": [
    {
      "user_id": "user-uuid",
      "username": "player123",
      "seat_index": 0,
      "joined_at": 1699123456000
    }
  ]
}
```

**Note**: Room creation, join, and leave are handled via WebSocket (see WebSocket section).

### 4. Gameplay Module

#### 4.1. Initialize Game
```http
POST /gameplay/matches/{match_id}/initialize
Authorization: Bearer <token>
```

**Path Parameters**:
- `match_id` (string, required): Match UUID

**Response 200**:
```json
{
  "message": "Game initialized successfully",
  "match_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Error Responses**:
- `400 BAD_REQUEST`: Match must be started before initialization
- `404 NOT_FOUND`: Match not found
- `400 BAD_REQUEST`: Game requires exactly 4 players

**What This Does**:
1. Creates game state with board configuration (56 cells, 4 players, 2 pieces each)
2. Randomizes player order
3. Initializes all piece states (8 pieces total: 4 players × 2 pieces)
4. Creates player balances for all players
5. Sets game status to `IN_PROGRESS`

#### 4.2. Roll Dice
```http
POST /gameplay/matches/{match_id}/roll-dice
Authorization: Bearer <token>
```

**Response 200**:
```json
{
  "id": "roll-uuid",
  "match_id": "550e8400-e29b-41d4-a716-446655440000",
  "user_id": "9c72f02d-9035-4991-a234-9c991280d35a",
  "turn": 1,
  "roll_value": [3, 3],
  "result": 6,
  "created_at": 1699123456000,
  "updated_at": 1699123456000
}
```

**Error Responses**:
- `400 BAD_REQUEST`: Not player's turn
- `400 BAD_REQUEST`: Match has already rolled (for this turn)
- `400 BAD_REQUEST`: Game is not in progress
- `404 NOT_FOUND`: Match not found

**Dice Roll Logic**:
- Uses weighted dice pool (100 entries, cached in Redis)
- Distribution: 20% total=6, 10% double 6, 70% other
- Auto-reshuffles when pool exhausted
- Detects doubles and tracks consecutive doubles (max 3)

#### 4.3. Get Game State
```http
GET /gameplay/matches/{match_id}/state
Authorization: Bearer <token>
```

**Response 200**:
```json
{
  "id": "state-uuid",
  "match_id": "550e8400-e29b-41d4-a716-446655440000",
  "room_id": "room-uuid",
  "status": "IN_PROGRESS",
  "current_turn": 1,
  "current_player_seat_index": 0,
  "timer_seconds": 300,
  "is_overtime": false,
  "double_count": 0,
  "player_order": [
    "user-id-1",
    "user-id-2",
    "user-id-3",
    "user-id-4"
  ],
  "board_state": {
    "total_cells": 56,
    "cells_per_direction": 14,
    "home_cells_per_player": 4,
    "finish_cell_per_player": 1,
    "players": [
      {
        "user_id": "user-id-1",
        "seat_index": 0,
        "color": "red",
        "home_index": 0,
        "finish_index": 13,
        "pieces": [
          {
            "piece_id": "piece-uuid-1",
            "position": 0,
            "status": "in_home"
          },
          {
            "piece_id": "piece-uuid-2",
            "position": 0,
            "status": "in_home"
          }
        ]
      }
    ]
  },
  "created_at": 1699123456000,
  "updated_at": 1699123456000
}
```

**Game Status Values**:
- `WAITING`: Game is waiting to start
- `STARTING`: Game is initializing
- `IN_PROGRESS`: Game is actively being played
- `FINISHED`: Game has ended
- `CANCELLED`: Game was cancelled

#### 4.4. Get Player Balances
```http
GET /gameplay/matches/{match_id}/balances
Authorization: Bearer <token>
```

**Response 200**:
```json
[
  {
    "id": "balance-uuid",
    "match_id": "550e8400-e29b-41d4-a716-446655440000",
    "user_id": "user-id-1",
    "initial_bet": 100000,
    "current_balance": 150000,
    "looted_amount": 50000,
    "is_bankrupt": false,
    "created_at": 1699123456000,
    "updated_at": 1699123456000
  }
]
```

#### 4.5. Get Piece States
```http
GET /gameplay/matches/{match_id}/pieces
Authorization: Bearer <token>
```

**Response 200**:
```json
[
  {
    "id": "piece-uuid",
    "match_id": "550e8400-e29b-41d4-a716-446655440000",
    "user_id": "user-id-1",
    "status": "in_track",
    "color": "red",
    "position": 5,
    "home_index": 0,
    "finish_index": 13,
    "created_at": 1699123456000,
    "updated_at": 1699123456000
  }
]
```

#### 4.6. Get Roll History
```http
GET /gameplay/matches/{match_id}/rolls?limit=10
Authorization: Bearer <token>
```

**Response 200**:
```json
[
  {
    "id": "roll-uuid",
    "match_id": "550e8400-e29b-41d4-a716-446655440000",
    "user_id": "user-id-1",
    "turn": 1,
    "roll_value": [3, 3],
    "result": 6,
    "created_at": 1699123456000,
    "updated_at": 1699123456000
  }
]
```

#### 4.7. Get Dice Pool Config
```http
GET /gameplay/dice-pool/config
Authorization: Bearer <token>
```

**Response 200**:
```json
{
  "enabled": true,
  "pool_size": 100,
  "distribution": {
    "total_six": {
      "percentage": 20,
      "count": 20
    },
    "double_six": {
      "percentage": 10,
      "count": 10
    }
  }
}
```

---

## 🔌 WebSocket Integration

### Connection Setup

#### 1. WebSocket URL
```
ws://localhost:3003/ws?token=<access_token>
```

**Authentication**:
- Token có thể truyền qua query parameter `token` (recommended cho browser)
- Hoặc qua header `Authorization: Bearer <token>` (nếu client hỗ trợ)

#### 2. Message Format

**Outgoing Message (Client → Server)**:
```typescript
interface OutgoingMessage {
  namespace: string;  // "room", "match", "gameplay"
  event: string;      // "create_room", "join_room", etc.
  data: any;          // Event-specific data
}
```

**Incoming Message (Server → Client)**:
```typescript
interface IncomingMessage {
  namespace?: string;  // Optional, echo of request
  event: string;       // Event name
  type: "success" | "error" | "data";
  data?: any;          // Response data (if success/data)
  error?: {            // Error info (if error)
    message: string;
  };
}
```

### WebSocket Events

### Room Namespace (`namespace: "room"`)

#### 1. Create Room

**Client → Server**:
```json
{
  "namespace": "room",
  "event": "create_room",
  "data": {
    "max_players": 4,
    "is_public": true,
    "bet_value": 100000
  }
}
```

**Server → Client (Success)**:
```json
{
  "namespace": "room",
  "event": "create_room",
  "type": "success",
  "data": {
    "id": "room-uuid",
    "owner_id": "user-uuid",
    "max_players": 4,
    "current_players": 1,
    "is_public": true,
    "bet_value": 100000,
    "minimum_amount": 100000,
    "is_active": true,
    "created_at": 1699123456000,
    "updated_at": 1699123456000
  }
}
```

**Server → Client (Error)**:
```json
{
  "namespace": "room",
  "event": "create_room",
  "type": "error",
  "error": {
    "message": "Insufficient balance to create room"
  }
}
```

**Create Private Room**:
```json
{
  "namespace": "room",
  "event": "create_room",
  "data": {
    "max_players": 4,
    "is_public": false,
    "password": 1234,
    "bet_value": 100000
  }
}
```

**Validation Rules**:
- `max_players`: 2-8 (default: 4)
- `bet_value`: Must be >= 0
- `password`: Required if `is_public: false`, must be 4 digits (1000-9999)
- User must have balance >= `bet_value`

#### 2. Join Room

**Client → Server**:
```json
{
  "namespace": "room",
  "event": "join_room",
  "data": {
    "room_id": "room-uuid",
    "password": 1234  // Optional, required for private rooms
  }
}
```

**Server → Client (Success)**:
```json
{
  "namespace": "room",
  "event": "join_room",
  "type": "success",
  "data": {
    "id": "room-uuid",
    "owner_id": "owner-uuid",
    "max_players": 4,
    "current_players": 2,
    "is_public": true,
    "bet_value": 100000,
    "created_at": 1699123456000,
    "updated_at": 1699123456000
  }
}
```

**Broadcast to Room (user_joined)**:
```json
{
  "event": "user_joined",
  "data": {
    "user_id": "user-uuid",
    "room_id": "room-uuid",
    "current_players": 2,
    "max_players": 4
  }
}
```

**Error Cases**:
- `400`: Room is full
- `400`: Invalid password (for private rooms)
- `400`: Insufficient balance
- `404`: Room not found
- `409`: Already in room

#### 3. Leave Room

**Client → Server**:
```json
{
  "namespace": "room",
  "event": "leave_room",
  "data": {
    "room_id": "room-uuid"
  }
}
```

**Server → Client (Success)**:
```json
{
  "namespace": "room",
  "event": "leave_room",
  "type": "success",
  "data": {
    "message": "left room successfully",
    "room_id": "room-uuid"
  }
}
```

**Broadcast to Room (user_left)**:
```json
{
  "event": "user_left",
  "data": {
    "user_id": "user-uuid",
    "room_id": "room-uuid",
    "current_players": 1,
    "max_players": 4,
    "is_owner": false,
    "new_owner_id": "new-owner-uuid"  // If owner left
  }
}
```

**Note**: If room owner leaves, ownership transfers to next player.

### WebSocket Connection Management

#### Connection Lifecycle

```typescript
// 1. Connect
const ws = new WebSocket(`ws://localhost:3003/ws?token=${token}`);

// 2. Handle connection open
ws.onopen = () => {
  console.log('WebSocket connected');
  // Connection is ready, can send messages
};

// 3. Handle incoming messages
ws.onmessage = (event) => {
  const message: IncomingMessage = JSON.parse(event.data);
  handleMessage(message);
};

// 4. Handle errors
ws.onerror = (error) => {
  console.error('WebSocket error:', error);
  // Implement reconnection logic
};

// 5. Handle connection close
ws.onclose = (event) => {
  console.log('WebSocket closed:', event.code, event.reason);
  // Implement reconnection logic
};
```

#### Reconnection Strategy

```typescript
class WebSocketManager {
  private ws: WebSocket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  private reconnectDelay = 1000;

  connect(token: string) {
    try {
      this.ws = new WebSocket(`ws://localhost:3003/ws?token=${token}`);
      this.setupHandlers();
    } catch (error) {
      console.error('Failed to connect:', error);
      this.scheduleReconnect(token);
    }
  }

  private setupHandlers() {
    if (!this.ws) return;

    this.ws.onopen = () => {
      console.log('WebSocket connected');
      this.reconnectAttempts = 0;
    };

    this.ws.onclose = (event) => {
      console.log('WebSocket closed:', event.code);
      if (event.code !== 1000) { // Not normal closure
        this.scheduleReconnect();
      }
    };

    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };

    this.ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      this.handleMessage(message);
    };
  }

  private scheduleReconnect(token?: string) {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error('Max reconnection attempts reached');
      return;
    }

    this.reconnectAttempts++;
    const delay = this.reconnectDelay * Math.pow(2, this.reconnectAttempts - 1); // Exponential backoff

    setTimeout(() => {
      console.log(`Reconnecting... (attempt ${this.reconnectAttempts})`);
      if (token) {
        this.connect(token);
      }
    }, delay);
  }

  send(message: OutgoingMessage) {
    if (this.ws && this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(message));
    } else {
      console.error('WebSocket is not open');
    }
  }

  close() {
    if (this.ws) {
      this.ws.close(1000, 'Client closing');
      this.ws = null;
    }
  }
}
```

---

## 🔄 Workflow Chi Tiết

### Complete Game Flow Sequence

```
┌─────────┐     ┌──────────┐     ┌─────────┐     ┌──────────┐
│ Client  │     │   API    │     │ WebSocket│    │ Database │
└────┬────┘     └────┬─────┘     └────┬────┘     └────┬─────┘
     │               │                │                │
     │──1. Register──>│                │                │
     │<──User ID─────│                │                │──Create User──>│
     │               │                │                │<──User Created─│
     │               │                │                │
     │──2. Login────>│                │                │
     │<──Tokens──────│                │                │
     │               │                │                │
     │──3. Create Balance───────────>│                │
     │<──Balance Created──────────────│                │──Create Balance>│
     │               │                │                │
     │──4. Credit Balance────────────>│                │
     │<──Transaction Pending──────────│                │──Queue Credit──>│
     │               │                │                │
     │──5. Connect WS─────────────────>│                │
     │<──WS Connected───────────────────│                │
     │               │                │                │
     │──6. Create Room (WS)─────────────>│                │
     │               │                │                │──Create Room──>│
     │<──Room Created───────────────────│                │<──Room Created─│
     │               │                │                │
     │──7. Broadcast: user_joined<───────│                │
     │               │                │                │
     │──8. Join Room (WS)───────────────>│                │
     │               │                │                │──Update Room───>│
     │<──Room Joined──────────────────────│                │<──Room Updated─│
     │               │                │                │
     │──9. Broadcast: user_joined<───────│                │
     │               │                │                │
     │──10. Create Match (when room full)│                │
     │               │                │                │──Create Match──>│
     │               │                │                │
     │──11. Start Match──────────────────>│                │
     │               │                │                │──Update Match──>│
     │               │                │                │
     │──12. Initialize Game──────────────>│                │
     │               │                │                │──Create Game State>│
     │               │                │                │──Create Pieces──>│
     │               │                │                │──Create Balances>│
     │<──Game Initialized─────────────────│                │
     │               │                │                │
     │──13. Get Game State───────────────>│                │
     │<──Game State───────────────────────│                │
     │               │                │                │
     │──14. Roll Dice─────────────────────>│                │
     │               │                │                │──Get Dice Pool──>│
     │               │                │                │──Create Roll───>│
     │<──Roll Result──────────────────────│                │
     │               │                │                │
     │──15. Get Pieces───────────────────>│                │
     │<──Piece States──────────────────────│                │
     │               │                │                │
     │──16. Move Piece (Future)───────────>│                │
     │               │                │                │──Update Piece──>│
     │               │                │                │──Create History>│
     │<──Piece Moved───────────────────────│                │
     │               │                │                │
     │──17. Next Turn─────────────────────>│                │
     │               │                │                │──Update Match───>│
     │               │                │                │
     │──18. Game Finished─────────────────>│                │
     │               │                │                │──Update Match───>│
     │<──Game Finished─────────────────────│                │
```

### Phase-by-Phase Workflow

#### Phase 1: User Setup & Authentication

**Step 1.1: User Registration**
```typescript
async function registerUser(username: string, password: string, email: string) {
  const response = await fetch('http://localhost:3003/iam/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ username, password, email })
  });
  
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error?.message || 'Registration failed');
  }
  
  return await response.json();
}
```

**Step 1.2: User Login**
```typescript
async function loginUser(username: string, password: string) {
  const response = await fetch('http://localhost:3003/iam/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ username, password })
  });
  
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error?.message || 'Login failed');
  }
  
  const data = await response.json();
  
  // Store tokens
  localStorage.setItem('access_token', data.data.access_token);
  localStorage.setItem('refresh_token', data.data.refresh_token);
  localStorage.setItem('user_id', data.data.user.id);
  
  return data.data;
}
```

**Step 1.3: Create User Balance**
```typescript
async function createUserBalance(userId: string, token: string) {
  const response = await fetch('http://localhost:3003/wallet/create-user-balance', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ user_id: userId })
  });
  
  return await response.json();
}
```

**Step 1.4: Credit Initial Balance**
```typescript
async function creditBalance(amount: number, token: string) {
  const response = await fetch('http://localhost:3003/wallet/credit', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      amount,
      reference_id: 'initial_deposit',
      metadata: {
        source: 'initial_deposit',
        description: 'Starting balance'
      }
    })
  });
  
  return await response.json();
}
```

#### Phase 2: Room Management (WebSocket)

**Step 2.1: Connect WebSocket**
```typescript
function connectWebSocket(token: string): Promise<WebSocket> {
  return new Promise((resolve, reject) => {
    const ws = new WebSocket(`ws://localhost:3003/ws?token=${token}`);
    
    ws.onopen = () => {
      console.log('WebSocket connected');
      resolve(ws);
    };
    
    ws.onerror = (error) => {
      console.error('WebSocket connection error:', error);
      reject(error);
    };
    
    ws.onclose = (event) => {
      console.log('WebSocket closed:', event.code, event.reason);
    };
  });
}
```

**Step 2.2: Create Room**
```typescript
function createRoom(ws: WebSocket, maxPlayers: number, betValue: number, isPublic: boolean = true, password?: number): Promise<RoomData> {
  return new Promise((resolve, reject) => {
    const message: OutgoingMessage = {
      namespace: 'room',
      event: 'create_room',
      data: {
        max_players: maxPlayers,
        is_public: isPublic,
        bet_value: betValue,
        ...(password && { password })
      }
    };
    
    const messageHandler = (event: MessageEvent) => {
      const response: IncomingMessage = JSON.parse(event.data);
      
      if (response.event === 'create_room') {
        ws.removeEventListener('message', messageHandler);
        
        if (response.type === 'success') {
          resolve(response.data as RoomData);
        } else {
          reject(new Error(response.error?.message || 'Failed to create room'));
        }
      }
    };
    
    ws.addEventListener('message', messageHandler);
    ws.send(JSON.stringify(message));
  });
}
```

**Step 2.3: Join Room**
```typescript
function joinRoom(ws: WebSocket, roomId: string, password?: number): Promise<RoomData> {
  return new Promise((resolve, reject) => {
    const message: OutgoingMessage = {
      namespace: 'room',
      event: 'join_room',
      data: {
        room_id: roomId,
        ...(password && { password })
      }
    };
    
    const messageHandler = (event: MessageEvent) => {
      const response: IncomingMessage = JSON.parse(event.data);
      
      if (response.event === 'join_room') {
        ws.removeEventListener('message', messageHandler);
        
        if (response.type === 'success') {
          resolve(response.data as RoomData);
        } else {
          reject(new Error(response.error?.message || 'Failed to join room'));
        }
      }
    };
    
    ws.addEventListener('message', messageHandler);
    ws.send(JSON.stringify(message));
  });
}
```

**Step 2.4: Handle Room Events**
```typescript
function setupRoomEventHandlers(ws: WebSocket, callbacks: {
  onUserJoined?: (data: any) => void;
  onUserLeft?: (data: any) => void;
  onRoomFull?: (data: any) => void;
}) {
  ws.addEventListener('message', (event) => {
    const message: IncomingMessage = JSON.parse(event.data);
    
    switch (message.event) {
      case 'user_joined':
        console.log('User joined:', message.data);
        callbacks.onUserJoined?.(message.data);
        break;
        
      case 'user_left':
        console.log('User left:', message.data);
        callbacks.onUserLeft?.(message.data);
        break;
        
      default:
        console.log('Unknown room event:', message.event);
    }
  });
}
```

#### Phase 3: Match & Game Initialization

**Step 3.1: Wait for Match Creation**
```typescript
// Match is created automatically when room is full (4 players)
// Or can be triggered manually by room owner
// Listen for match_created event via WebSocket (if implemented)
```

**Step 3.2: Initialize Game**
```typescript
async function initializeGame(matchId: string, token: string) {
  const response = await fetch(`http://localhost:3003/gameplay/matches/${matchId}/initialize`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error?.message || 'Failed to initialize game');
  }
  
  return await response.json();
}
```

**Step 3.3: Load Initial Game State**
```typescript
async function loadGameState(matchId: string, token: string): Promise<GameStateDTO> {
  const response = await fetch(`http://localhost:3003/gameplay/matches/${matchId}/state`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error?.message || 'Failed to load game state');
  }
  
  const data = await response.json();
  return data.data || data;
}
```

**Step 3.4: Load Pieces & Balances**
```typescript
async function loadGameData(matchId: string, token: string) {
  const [piecesResponse, balancesResponse] = await Promise.all([
    fetch(`http://localhost:3003/gameplay/matches/${matchId}/pieces`, {
      headers: { 'Authorization': `Bearer ${token}` }
    }),
    fetch(`http://localhost:3003/gameplay/matches/${matchId}/balances`, {
      headers: { 'Authorization': `Bearer ${token}` }
    })
  ]);
  
  const pieces = await piecesResponse.json();
  const balances = await balancesResponse.json();
  
  return {
    pieces: pieces.data || pieces,
    balances: balances.data || balances
  };
}
```

#### Phase 4: Gameplay Loop

**Step 4.1: Check Turn & Roll Dice**
```typescript
async function gameLoop(matchId: string, userId: string, token: string) {
  let gameRunning = true;
  
  while (gameRunning) {
    // Get current game state
    const gameState = await loadGameState(matchId, token);
    
    // Check if game finished
    if (gameState.status === 'FINISHED') {
      console.log('Game finished!');
      gameRunning = false;
      break;
    }
    
    // Check if it's my turn
    const currentPlayerId = gameState.player_order[gameState.current_player_seat_index];
    
    if (currentPlayerId === userId && gameState.status === 'IN_PROGRESS') {
      // It's my turn - roll dice
      try {
        const rollResult = await rollDice(matchId, token);
        console.log('Rolled:', rollResult.roll_value, 'Result:', rollResult.result);
        
        // After rolling, player can move pieces
        // Move logic will be implemented separately
      } catch (error) {
        console.error('Roll dice error:', error);
      }
    }
    
    // Wait before next check (polling interval)
    await new Promise(resolve => setTimeout(resolve, 2000));
  }
}
```

**Step 4.2: Roll Dice**
```typescript
async function rollDice(matchId: string, token: string): Promise<RollHistoryDTO> {
  const response = await fetch(`http://localhost:3003/gameplay/matches/${matchId}/roll-dice`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error?.message || 'Failed to roll dice');
  }
  
  const data = await response.json();
  return data.data || data;
}
```

**Step 4.3: Get Roll History**
```typescript
async function getRollHistory(matchId: string, token: string, limit: number = 10): Promise<RollHistoryDTO[]> {
  const response = await fetch(`http://localhost:3003/gameplay/matches/${matchId}/rolls?limit=${limit}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error?.message || 'Failed to get roll history');
  }
  
  const data = await response.json();
  return data.data || data;
}
```

---

## 💻 Code Examples (React/TypeScript)

### Project Structure

```
game-play/
├── package.json
├── tsconfig.json
├── yarn.lock
├── src/
│   ├── types/
│   │   ├── api.ts          # API response types
│   │   ├── websocket.ts    # WebSocket message types
│   │   └── game.ts         # Game state types
│   ├── services/
│   │   ├── api.ts          # HTTP API client
│   │   ├── websocket.ts    # WebSocket client
│   │   └── auth.ts     
