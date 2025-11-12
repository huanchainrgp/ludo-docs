# 🎮 Ludo Game API Integration Guide

Tài liệu hướng dẫn hoàn chỉnh cách gọi API để chơi game Ludo và sử dụng WebSocket.

## 📋 Table of Contents

1. [Overview](#overview)
2. [Authentication](#authentication)
3. [HTTP API Endpoints](#http-api-endpoints)
4. [WebSocket Integration](#websocket-integration)
5. [Complete Game Flow](#complete-game-flow)
6. [Error Handling](#error-handling)
7. [Testing](#testing)

---

## 🎯 Overview

### Base URLs
- **API Server**: `http://localhost:3003`
- **WebSocket**: `ws://localhost:3003/ws`

### Architecture
- **HTTP API**: RESTful endpoints cho các operations
- **WebSocket**: Real-time communication cho room và game events
- **Authentication**: JWT Bearer token required

---

## 🔐 Authentication

### 1. User Registration
```http
POST /iam/register
Content-Type: application/json

{
  "username": "player123",
  "password": "password123",
  "email": "player123@example.com"
}
```

**Response 201:**
```json
{
  "data": {
    "id": "9c72f02d-9035-4991-a234-9c991280d35a",
    "username": "player123",
    "email": "player123@example.com",
    "created_at": 1699123456,
    "updated_at": 1699123456
  }
}
```

### 2. User Login
```http
POST /iam/login
Content-Type: application/json

{
  "username": "player123",
  "password": "password123"
}
```

**Response 200:**
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

### 3. Token Usage
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

---

## 🌐 HTTP API Endpoints

### 💰 Wallet Management

#### Create User Balance
```http
POST /wallet/create-user-balance
Authorization: Bearer <token>
Content-Type: application/json

{
  "user_id": "9c72f02d-9035-4991-a234-9c991280d35a"
}
```

#### Credit Balance
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

#### Get Balance
```http
GET /wallet/balance
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "data": {
    "id": "balance-uuid",
    "user_id": "9c72f02d-9035-4991-a234-9c991280d35a",
    "balance": 1000000,
    "currency": "USD",
    "created_at": 1699123456,
    "updated_at": 1699123456
  }
}
```

### 🏠 Room Management

#### Get Open Rooms
```http
GET /rooms/open
```

**Response 200:**
```json
{
  "data": [
    {
      "id": "room-uuid",
      "max_players": 4,
      "current_players": 2,
      "status": "waiting",
      "is_public": true,
      "bet_value": 100000,
      "created_at": 1699123456
    }
  ]
}
```

#### Create Room (via WebSocket)
**Use WebSocket instead of HTTP API**

#### Join Room (via WebSocket)
**Use WebSocket instead of HTTP API**

### 🎮 Gameplay Management

#### Initialize Game
```http
POST /gameplay/matches/{match_id}/initialize
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "message": "Game initialized successfully",
  "match_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

#### Roll Dice
```http
POST /gameplay/matches/{match_id}/roll-dice
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "data": {
    "id": "roll-uuid",
    "match_id": "550e8400-e29b-41d4-a716-446655440000",
    "user_id": "9c72f02d-9035-4991-a234-9c991280d35a",
    "turn": 1,
    "roll_value": [3, 3],
    "result": 6,
    "created_at": 1699123456,
    "updated_at": 1699123456
  }
}
```

#### Get Game State
```http
GET /gameplay/matches/{match_id}/state
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "data": {
    "id": "state-uuid",
    "match_id": "550e8400-e29b-41d4-a716-446655440000",
    "room_id": "room-uuid",
    "board_state": {
      "players": [
        {
          "user_id": "player1-uuid",
          "seat_index": 0,
          "color": "red",
          "pieces": [
            {
              "id": "piece-uuid",
              "status": "in_home",
              "position": 0,
              "home_index": 0
            }
          ]
        }
      ]
    },
    "current_turn": 1,
    "current_player_seat": 0,
    "timer_seconds": 30,
    "created_at": 1699123456
  }
}
```

#### Get Player Balances
```http
GET /gameplay/matches/{match_id}/balances
Authorization: Bearer <token>
```

#### Get Piece States
```http
GET /gameplay/matches/{match_id}/pieces
Authorization: Bearer <token>
```

---

## 🔌 WebSocket Integration

### Connection Setup

#### 1. Connect to WebSocket
```javascript
const token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...";
const ws = new WebSocket(`ws://localhost:3003/ws?token=${token}`);

ws.onopen = function() {
    console.log('WebSocket connected');
};

ws.onmessage = function(event) {
    const message = JSON.parse(event.data);
    handleMessage(message);
};

ws.onerror = function(error) {
    console.error('WebSocket error:', error);
};
```

#### 2. Message Format
**Outgoing Message (Client → Server):**
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

**Incoming Message (Server → Client):**
```json
{
  "namespace": "room",
  "event": "create_room",
  "type": "success",
  "data": {
    "id": "room-uuid",
    "max_players": 4,
    "current_players": 1,
    "status": "waiting",
    "is_public": true,
    "bet_value": 100000
  }
}
```

### Room Namespace Events

#### Create Room
```javascript
function createRoom() {
    const message = {
        namespace: 'room',
        event: 'create_room',
        data: {
            max_players: 4,
            is_public: true,
            bet_value: 100000
        }
    };
    ws.send(JSON.stringify(message));
}
```

#### Create Private Room
```javascript
function createPrivateRoom() {
    const message = {
        namespace: 'room',
        event: 'create_room',
        data: {
            max_players: 4,
            is_public: false,
            password: 1234,  // 4-digit password
            bet_value: 100000
        }
    };
    ws.send(JSON.stringify(message));
}
```

#### Join Room
```javascript
function joinRoom(roomId, password = null) {
    const message = {
        namespace: 'room',
        event: 'join_room',
        data: {
            room_id: roomId
        }
    };
    
    // Add password for private rooms
    if (password) {
        message.data.password = password;
    }
    
    ws.send(JSON.stringify(message));
}
```

#### Leave Room
```javascript
function leaveRoom(roomId) {
    const message = {
        namespace: 'room',
        event: 'leave_room',
        data: {
            room_id: roomId
        }
    };
    ws.send(JSON.stringify(message));
}
```

### WebSocket Event Handling

```javascript
function handleMessage(message) {
    console.log('Received:', message);
    
    switch (message.namespace) {
        case 'room':
            handleRoomEvent(message);
            break;
        // Add other namespaces as needed
        default:
            console.log('Unknown namespace:', message.namespace);
    }
}

function handleRoomEvent(message) {
    switch (message.event) {
        case 'create_room':
            if (message.type === 'success') {
                console.log('Room created:', message.data);
                // Update UI with room info
                updateRoomInfo(message.data);
            } else {
                console.error('Create room failed:', message.error);
            }
            break;
            
        case 'join_room':
            if (message.type === 'success') {
                console.log('Joined room:', message.data);
                updateRoomInfo(message.data);
            } else {
                console.error('Join room failed:', message.error);
            }
            break;
            
        case 'user_joined':
            console.log('User joined room:', message.data);
            // Update player list
            updatePlayerList(message.data);
            break;
            
        case 'user_left':
            console.log('User left room:', message.data);
            // Update player list
            updatePlayerList(message.data);
            break;
            
        case 'leave_room':
            if (message.type === 'success') {
                console.log('Left room successfully');
                clearRoomInfo();
            }
            break;
            
        default:
            console.log('Unknown room event:', message.event);
    }
}
```

---

## 🎲 Complete Game Flow

### Phase 1: User Setup
```javascript
// 1. Register/Login
async function setupUser() {
    // Login
    const loginResponse = await fetch('/iam/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
            username: 'player123',
            password: 'password123'
        })
    });
    const { data } = await loginResponse.json();
    const token = data.access_token;
    
    // Create balance if needed
    await fetch('/wallet/create-user-balance', {
        method: 'POST',
        headers: {
            'Authorization': `Bearer ${token}`,
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            user_id: data.user.id
        })
    });
    
    // Credit initial balance
    await fetch('/wallet/credit', {
        method: 'POST',
        headers: {
            'Authorization': `Bearer ${token}`,
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            amount: 1000000,
            reference_id: 'initial_deposit'
        })
    });
    
    return token;
}
```

### Phase 2: Room Management
```javascript
// 2. Connect WebSocket and create/join room
async function joinGameRoom(token) {
    const ws = new WebSocket(`ws://localhost:3003/ws?token=${token}`);
    
    await new Promise(resolve => {
        ws.onopen = resolve;
    });
    
    // Create room
    ws.send(JSON.stringify({
        namespace: 'room',
        event: 'create_room',
        data: {
            max_players: 4,
            is_public: true,
            bet_value: 100000
        }
    }));
    
    // Wait for room creation
    return new Promise(resolve => {
        ws.onmessage = function(event) {
            const message = JSON.parse(event.data);
            if (message.event === 'create_room' && message.type === 'success') {
                resolve({ ws, roomId: message.data.id });
            }
        };
    });
}
```

### Phase 3: Match Management
```javascript
// 3. Start match (when room is full or ready)
async function startMatch(token, matchId) {
    // Initialize game
    const initResponse = await fetch(`/gameplay/matches/${matchId}/initialize`, {
        method: 'POST',
        headers: {
            'Authorization': `Bearer ${token}`
        }
    });
    
    if (initResponse.ok) {
        console.log('Game initialized successfully');
        return true;
    }
    return false;
}
```

### Phase 4: Gameplay Loop
```javascript
// 4. Game loop
async function gameLoop(token, matchId) {
    let gameRunning = true;
    
    while (gameRunning) {
        // Get current game state
        const stateResponse = await fetch(`/gameplay/matches/${matchId}/state`, {
            headers: { 'Authorization': `Bearer ${token}` }
        });
        const gameState = await stateResponse.json();
        
        // Check if it's player's turn
        if (isMyTurn(gameState.data)) {
            // Roll dice
            const rollResponse = await fetch(`/gameplay/matches/${matchId}/roll-dice`, {
                method: 'POST',
                headers: { 'Authorization': `Bearer ${token}` }
            });
            const rollResult = await rollResponse.json();
            
            console.log('Rolled:', rollResult.data.result);
            
            // Get piece states
            const piecesResponse = await fetch(`/gameplay/matches/${matchId}/pieces`, {
                headers: { 'Authorization': `Bearer ${token}` }
            });
            const pieces = await piecesResponse.json();
            
            // Move pieces (logic to be implemented)
            // await movePiece(token, matchId, pieceId, newPosition);
        }
        
        // Wait before next check
        await new Promise(resolve => setTimeout(resolve, 2000));
        
        // Check if game finished
        if (gameState.data.status === 'finished') {
            gameRunning = false;
        }
    }
}

function isMyTurn(gameState) {
    // Implement logic to check if it's current player's turn
    return gameState.current_player_seat === myPlayerSeat;
}
```

### Complete Integration Example
```javascript
async function playGame() {
    try {
        // Phase 1: Setup user
        const token = await setupUser();
        console.log('User setup complete');
        
        // Phase 2: Join room
        const { ws, roomId } = await joinGameRoom(token);
        console.log('Joined room:', roomId);
        
        // Wait for match to start (this would be triggered by room owner or automatically)
        // For demo, assume we have matchId
        const matchId = 'some-match-uuid';
        
        // Phase 3: Initialize game
        const gameReady = await startMatch(token, matchId);
        if (!gameReady) {
            throw new Error('Failed to initialize game');
        }
        
        // Phase 4: Play game
        await gameLoop(token, matchId);
        
        console.log('Game completed');
        
    } catch (error) {
        console.error('Game error:', error);
    }
}

// Start the game
playGame();
```

---

## ❌ Error Handling

### HTTP Error Responses
```json
{
  "error": {
    "code": "BAD_REQUEST",
    "message": "Invalid request data",
    "details": "Amount must be greater than 0"
  }
}
```

### WebSocket Error Responses
```json
{
  "namespace": "room",
  "event": "create_room", 
  "type": "error",
  "error": "Insufficient balance to create room"
}
```

### Common Error Codes
- `400 BAD_REQUEST`: Invalid input data
- `401 UNAUTHORIZED`: Invalid or expired token
- `403 FORBIDDEN`: Access denied
- `404 NOT_FOUND`: Resource not found
- `409 CONFLICT`: Resource conflict (e.g., already joined)
- `500 INTERNAL_SERVER_ERROR`: Server error

### Error Handling Best Practices
```javascript
// HTTP API error handling
async function makeApiCall(url, options) {
    try {
        const response = await fetch(url, options);
        const data = await response.json();
        
        if (!response.ok) {
            throw new Error(data.error?.message || 'API call failed');
        }
        
        return data;
    } catch (error) {
        console.error('API Error:', error);
        // Handle error appropriately
        showErrorToUser(error.message);
        throw error;
    }
}

// WebSocket error handling
ws.onmessage = function(event) {
    const message = JSON.parse(event.data);
    
    if (message.type === 'error') {
        console.error('WebSocket Error:', message.error);
        showErrorToUser(message.error);
        return;
    }
    
    // Handle success message
    handleMessage(message);
};
```

---

## 🧪 Testing

### Manual Testing with cURL

#### Test Login
```bash
curl -X POST http://localhost:3003/iam/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "password": "admin123"
  }'
```

#### Test Wallet Credit
```bash
curl -X POST http://localhost:3003/wallet/credit \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 1000000,
    "reference_id": "test_credit"
  }'
```

### WebSocket Testing
Use the provided WebSocket test file:
```
/Users/admin/Documents/ludo-backend/test/websocket_test.html
```

Open in browser and test WebSocket connections with JWT tokens.

### Swagger Documentation
Access API documentation at:
```
http://localhost:3003/swagger/index.html
```

---

## 📈 Performance Considerations

### API Rate Limits
- No specific rate limits implemented currently
- Consider implementing rate limiting for production

### WebSocket Connection Management
- Maintain single WebSocket connection per client
- Implement reconnection logic for dropped connections
- Handle connection cleanup on page unload

### Polling vs Real-time
- **Use WebSocket for**: Room events, player join/leave, real-time notifications
- **Use HTTP API for**: Game state queries, dice rolls, balance checks
- **Recommended polling interval**: 2-5 seconds for game state

---

## 🔧 Development Setup

### Start the Server
```bash
# Setup database and dependencies
make migrate-up
make deps

# Start API server
make dev

# Start worker (for wallet operations)
make worker

# Start monitoring dashboard
make monitor
```

### Environment Variables
Create `.env` file:
```env
PORT=3003
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=password
DB_DATABASE=ludo
JWT_ACCESS_SECRET=dev-access-secret
JWT_REFRESH_SECRET=dev-refresh-secret
REDIS_ADDR=localhost:6379
```

---

## 📞 Support

### Endpoints for Help
- **Swagger UI**: `http://localhost:3003/swagger/index.html`
- **Asynq Monitor**: `http://localhost:9090/monitor`

### Default Admin Credentials
- Username: `admin`
- Password: `admin123`

### Debug Tips
1. Check server logs for detailed error messages
2. Use browser developer tools for WebSocket debugging
3. Verify JWT token is not expired
4. Ensure database is running and migrated
5. Check Redis connection for queue operations

---

**Happy Gaming! 🎮**



# 🎮 Ludo Game API Integration Guide

Tài liệu hướng dẫn hoàn chỉnh cách gọi API để chơi game Ludo và sử dụng WebSocket.

## 📋 Table of Contents

1. [Overview](#overview)
2. [Authentication](#authentication)
3. [HTTP API Endpoints](#http-api-endpoints)
4. [WebSocket Integration](#websocket-integration)
5. [Cocos2D/TypeScript Client Integration](#cocos2dtypescript-client-integration)
6. [Complete Game Flow](#complete-game-flow)
7. [Error Handling](#error-handling)
8. [Testing](#testing)

---

## 🎯 Overview

### Base URLs
- **API Server**: `http://localhost:3003`
- **WebSocket**: `ws://localhost:3003/ws`

### Architecture
- **HTTP API**: RESTful endpoints cho các operations
- **WebSocket**: Real-time communication cho room và game events
- **Authentication**: JWT Bearer token required

---

## 🔐 Authentication

### 1. User Registration
```http
POST /iam/register
Content-Type: application/json

{
  "username": "player123",
  "password": "password123",
  "email": "player123@example.com"
}
```

**Response 201:**
```json
{
  "data": {
    "id": "9c72f02d-9035-4991-a234-9c991280d35a",
    "username": "player123",
    "email": "player123@example.com",
    "created_at": 1699123456,
    "updated_at": 1699123456
  }
}
```

### 2. User Login
```http
POST /iam/login
Content-Type: application/json

{
  "username": "player123",
  "password": "password123"
}
```

**Response 200:**
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

### 3. Token Usage
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

---

## 🌐 HTTP API Endpoints

### 💰 Wallet Management

#### Create User Balance
```http
POST /wallet/create-user-balance
Authorization: Bearer <token>
Content-Type: application/json

{
  "user_id": "9c72f02d-9035-4991-a234-9c991280d35a"
}
```

#### Credit Balance
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

#### Get Balance
```http
GET /wallet/balance
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "data": {
    "id": "balance-uuid",
    "user_id": "9c72f02d-9035-4991-a234-9c991280d35a",
    "balance": 1000000,
    "currency": "USD",
    "created_at": 1699123456,
    "updated_at": 1699123456
  }
}
```

### 🏠 Room Management

#### Get Open Rooms
```http
GET /rooms/open
```

**Response 200:**
```json
{
  "data": [
    {
      "id": "room-uuid",
      "max_players": 4,
      "current_players": 2,
      "status": "waiting",
      "is_public": true,
      "bet_value": 100000,
      "created_at": 1699123456
    }
  ]
}
```

#### Create Room (via WebSocket)
**Use WebSocket instead of HTTP API**

#### Join Room (via WebSocket)
**Use WebSocket instead of HTTP API**

### 🎮 Gameplay Management

#### Initialize Game
```http
POST /gameplay/matches/{match_id}/initialize
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "message": "Game initialized successfully",
  "match_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

#### Roll Dice
```http
POST /gameplay/matches/{match_id}/roll-dice
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "data": {
    "id": "roll-uuid",
    "match_id": "550e8400-e29b-41d4-a716-446655440000",
    "user_id": "9c72f02d-9035-4991-a234-9c991280d35a",
    "turn": 1,
    "roll_value": [3, 3],
    "result": 6,
    "created_at": 1699123456,
    "updated_at": 1699123456
  }
}
```

#### Get Game State
```http
GET /gameplay/matches/{match_id}/state
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "data": {
    "id": "state-uuid",
    "match_id": "550e8400-e29b-41d4-a716-446655440000",
    "room_id": "room-uuid",
    "board_state": {
      "players": [
        {
          "user_id": "player1-uuid",
          "seat_index": 0,
          "color": "red",
          "pieces": [
            {
              "id": "piece-uuid",
              "status": "in_home",
              "position": 0,
              "home_index": 0
            }
          ]
        }
      ]
    },
    "current_turn": 1,
    "current_player_seat": 0,
    "timer_seconds": 30,
    "created_at": 1699123456
  }
}
```

#### Get Player Balances
```http
GET /gameplay/matches/{match_id}/balances
Authorization: Bearer <token>
```

#### Get Piece States
```http
GET /gameplay/matches/{match_id}/pieces
Authorization: Bearer <token>
```

---

## 🔌 WebSocket Integration

### Connection Setup

#### 1. Connect to WebSocket
```javascript
const token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...";
const ws = new WebSocket(`ws://localhost:3003/ws?token=${token}`);

ws.onopen = function() {
    console.log('WebSocket connected');
};

ws.onmessage = function(event) {
    const message = JSON.parse(event.data);
    handleMessage(message);
};

ws.onerror = function(error) {
    console.error('WebSocket error:', error);
};
```

#### 2. Message Format
**Outgoing Message (Client → Server):**
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

**Incoming Message (Server → Client):**
```json
{
  "namespace": "room",
  "event": "create_room",
  "type": "success",
  "data": {
    "id": "room-uuid",
    "max_players": 4,
    "current_players": 1,
    "status": "waiting",
    "is_public": true,
    "bet_value": 100000
  }
}
```

### Room Namespace Events

#### Create Room
```javascript
function createRoom() {
    const message = {
        namespace: 'room',
        event: 'create_room',
        data: {
            max_players: 4,
            is_public: true,
            bet_value: 100000
        }
    };
    ws.send(JSON.stringify(message));
}
```

#### Create Private Room
```javascript
function createPrivateRoom() {
    const message = {
        namespace: 'room',
        event: 'create_room',
        data: {
            max_players: 4,
            is_public: false,
            password: 1234,  // 4-digit password
            bet_value: 100000
        }
    };
    ws.send(JSON.stringify(message));
}
```

#### Join Room
```javascript
function joinRoom(roomId, password = null) {
    const message = {
        namespace: 'room',
        event: 'join_room',
        data: {
            room_id: roomId
        }
    };
    
    // Add password for private rooms
    if (password) {
        message.data.password = password;
    }
    
    ws.send(JSON.stringify(message));
}
```

#### Leave Room
```javascript
function leaveRoom(roomId) {
    const message = {
        namespace: 'room',
        event: 'leave_room',
        data: {
            room_id: roomId
        }
    };
    ws.send(JSON.stringify(message));
}
```

### WebSocket Event Handling

```javascript
function handleMessage(message) {
    console.log('Received:', message);
    
    switch (message.namespace) {
        case 'room':
            handleRoomEvent(message);
            break;
        // Add other namespaces as needed
        default:
            console.log('Unknown namespace:', message.namespace);
    }
}

function handleRoomEvent(message) {
    switch (message.event) {
        case 'create_room':
            if (message.type === 'success') {
                console.log('Room created:', message.data);
                // Update UI with room info
                updateRoomInfo(message.data);
            } else {
                console.error('Create room failed:', message.error);
            }
            break;
            
        case 'join_room':
            if (message.type === 'success') {
                console.log('Joined room:', message.data);
                updateRoomInfo(message.data);
            } else {
                console.error('Join room failed:', message.error);
            }
            break;
            
        case 'user_joined':
            console.log('User joined room:', message.data);
            // Update player list
            updatePlayerList(message.data);
            break;
            
        case 'user_left':
            console.log('User left room:', message.data);
            // Update player list
            updatePlayerList(message.data);
            break;
            
        case 'leave_room':
            if (message.type === 'success') {
                console.log('Left room successfully');
                clearRoomInfo();
            }
            break;
            
        default:
            console.log('Unknown room event:', message.event);
    }
}
```

---

## 🎮 Cocos2D/TypeScript Client Integration

This section provides a complete integration guide for Cocos Creator (Cocos2D) game clients using TypeScript.

### Project Setup

#### 1. Install Dependencies

```bash
# In your Cocos Creator project
npm install axios ws
npm install --save-dev @types/ws
```

#### 2. Project Structure

```
assets/
├── scripts/
│   ├── network/
│   │   ├── ApiService.ts          # HTTP API service
│   │   ├── WebSocketManager.ts    # WebSocket manager
│   │   └── types.ts                # TypeScript type definitions
│   ├── game/
│   │   ├── GameStateManager.ts    # Game state management
│   │   ├── RoomManager.ts         # Room management
│   │   └── GameplayManager.ts     # Gameplay logic
│   └── scenes/
│       ├── LoginScene.ts
│       ├── RoomScene.ts
│       └── GameScene.ts
```

### TypeScript Type Definitions

Create `assets/scripts/network/types.ts`:

```typescript
// API Response Types
export interface ApiResponse<T> {
  data: T;
}

export interface ApiError {
  error: {
    code: string;
    message: string;
    details?: string;
  };
}

// Authentication Types
export interface LoginRequest {
  username: string;
  password: string;
}

export interface LoginResponse {
  access_token: string;
  refresh_token: string;
  expires_in: number;
  user: {
    id: string;
    username: string;
    email: string;
  };
}

export interface RegisterRequest {
  username: string;
  password: string;
  email: string;
}

// Wallet Types
export interface WalletBalance {
  id: string;
  user_id: string;
  balance: number;
  currency: string;
  created_at: number;
  updated_at: number;
}

export interface CreditRequest {
  amount: number;
  reference_id: string;
  metadata?: {
    source?: string;
    description?: string;
  };
}

// Room Types
export interface Room {
  id: string;
  max_players: number;
  current_players: number;
  status: "waiting" | "starting" | "in_progress" | "finished" | "cancelled";
  is_public: boolean;
  bet_value: number;
  created_at: number;
}

export interface CreateRoomRequest {
  max_players: number;
  is_public: boolean;
  bet_value: number;
  password?: number; // 4-digit password for private rooms
}

export interface JoinRoomRequest {
  room_id: string;
  password?: number;
}

// Game State Types
export type GameStatus = "WAITING" | "STARTING" | "IN_PROGRESS" | "FINISHED" | "CANCELLED";
export type PieceStatus = "in_home" | "in_track" | "in_finish";
export type PlayerColor = "red" | "blue" | "yellow" | "purple";

export interface GameStateDTO {
  id: string;
  match_id: string;
  room_id: string;
  status: GameStatus;
  current_turn: number;
  current_player_seat_index: number;
  timer_seconds: number;
  is_overtime: boolean;
  double_count: number;
  player_order: string[];
  board_state: BoardStateDTO;
  created_at: number;
  updated_at: number;
}

export interface BoardStateDTO {
  total_cells: number; // 56
  cells_per_direction: number; // 14
  home_cells_per_player: number; // 4
  finish_cell_per_player: number; // 1
  players: PlayerBoardStateDTO[];
}

export interface PlayerBoardStateDTO {
  user_id: string;
  seat_index: number; // 0-3
  color: PlayerColor;
  home_index: number;
  finish_index: number;
  pieces: PieceBoardStateDTO[];
}

export interface PieceBoardStateDTO {
  piece_id: string;
  position: number; // Cell index (0-55)
  status: PieceStatus;
}

export interface PieceStateDTO {
  id: string;
  match_id: string;
  user_id: string;
  status: PieceStatus;
  color: PlayerColor;
  position: number;
  home_index: number;
  finish_index: number;
  created_at: number;
  updated_at: number;
}

export interface RollHistoryDTO {
  id: string;
  match_id: string;
  user_id: string;
  turn: number;
  roll_value: number[]; // [dice1, dice2]
  result: number; // Sum of dice
  created_at: number;
  updated_at: number;
}

export interface PlayerBalanceDTO {
  id: string;
  match_id: string;
  user_id: string;
  initial_bet: number;
  current_balance: number;
  looted_amount: number;
  is_bankrupt: boolean;
  created_at: number;
  updated_at: number;
}

// WebSocket Message Types
export interface WebSocketMessage {
  namespace: string;
  event: string;
  type?: "success" | "error" | "data";
  data?: any;
  error?: string;
}

export interface RoomWebSocketMessage extends WebSocketMessage {
  namespace: "room";
  event: "create_room" | "join_room" | "leave_room" | "user_joined" | "user_left";
}
```

### HTTP API Service

Create `assets/scripts/network/ApiService.ts`:

```typescript
import axios, { AxiosInstance, AxiosError } from 'axios';
import { ApiResponse, ApiError } from './types';

export class ApiService {
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

    // Add request interceptor to include token
    this.client.interceptors.request.use((config) => {
      if (this.token) {
        config.headers.Authorization = `Bearer ${this.token}`;
      }
      return config;
    });

    // Add response interceptor for error handling
    this.client.interceptors.response.use(
      (response) => response,
      (error: AxiosError<ApiError>) => {
        if (error.response?.data?.error) {
          throw new Error(error.response.data.error.message);
        }
        throw error;
      }
    );
  }

  setToken(token: string) {
    this.token = token;
  }

  clearToken() {
    this.token = null;
  }

  // Authentication
  async login(username: string, password: string): Promise<ApiResponse<any>> {
    const response = await this.client.post('/iam/login', { username, password });
    return response.data;
  }

  async register(username: string, password: string, email: string): Promise<ApiResponse<any>> {
    const response = await this.client.post('/iam/register', { username, password, email });
    return response.data;
  }

  // Wallet
  async createUserBalance(userId: string): Promise<ApiResponse<any>> {
    const response = await this.client.post('/wallet/create-user-balance', { user_id: userId });
    return response.data;
  }

  async creditBalance(amount: number, referenceId: string, metadata?: any): Promise<ApiResponse<any>> {
    const response = await this.client.post('/wallet/credit', {
      amount,
      reference_id: referenceId,
      metadata,
    });
    return response.data;
  }

  async getBalance(): Promise<ApiResponse<any>> {
    const response = await this.client.get('/wallet/balance');
    return response.data;
  }

  // Rooms
  async getOpenRooms(): Promise<ApiResponse<any[]>> {
    const response = await this.client.get('/rooms/open');
    return response.data;
  }

  // Gameplay
  async initializeGame(matchId: string): Promise<any> {
    const response = await this.client.post(`/gameplay/matches/${matchId}/initialize`);
    return response.data;
  }

  async rollDice(matchId: string): Promise<ApiResponse<any>> {
    const response = await this.client.post(`/gameplay/matches/${matchId}/roll-dice`);
    return response.data;
  }

  async getGameState(matchId: string): Promise<ApiResponse<any>> {
    const response = await this.client.get(`/gameplay/matches/${matchId}/state`);
    return response.data;
  }

  async getPlayerBalances(matchId: string): Promise<ApiResponse<any[]>> {
    const response = await this.client.get(`/gameplay/matches/${matchId}/balances`);
    return response.data;
  }

  async getPieceStates(matchId: string): Promise<ApiResponse<any[]>> {
    const response = await this.client.get(`/gameplay/matches/${matchId}/pieces`);
    return response.data;
  }

  async getRollHistory(matchId: string): Promise<ApiResponse<any[]>> {
    const response = await this.client.get(`/gameplay/matches/${matchId}/rolls`);
    return response.data;
  }

  async getDicePoolConfig(): Promise<ApiResponse<any>> {
    const response = await this.client.get('/gameplay/dice-pool/config');
    return response.data;
  }
}
```

### WebSocket Manager

Create `assets/scripts/network/WebSocketManager.ts`:

```typescript
import { WebSocketMessage, RoomWebSocketMessage } from './types';

export class WebSocketManager {
  private ws: WebSocket | null = null;
  private url: string;
  private token: string | null = null;
  private reconnectAttempts: number = 0;
  private maxReconnectAttempts: number = 5;
  private reconnectDelay: number = 3000;
  private messageHandlers: Map<string, (message: WebSocketMessage) => void> = new Map();
  private isConnecting: boolean = false;

  constructor(baseURL: string = 'ws://localhost:3003') {
    this.url = `${baseURL}/ws`;
  }

  connect(token: string): Promise<void> {
    if (this.isConnecting || (this.ws && this.ws.readyState === WebSocket.OPEN)) {
      return Promise.resolve();
    }

    this.token = token;
    this.isConnecting = true;

    return new Promise((resolve, reject) => {
      try {
        const wsUrl = `${this.url}?token=${token}`;
        this.ws = new WebSocket(wsUrl);

        this.ws.onopen = () => {
          console.log('WebSocket connected');
          this.isConnecting = false;
          this.reconnectAttempts = 0;
          resolve();
        };

        this.ws.onmessage = (event) => {
          try {
            const message: WebSocketMessage = JSON.parse(event.data);
            this.handleMessage(message);
          } catch (error) {
            console.error('Failed to parse WebSocket message:', error);
          }
        };

        this.ws.onerror = (error) => {
          console.error('WebSocket error:', error);
          this.isConnecting = false;
          reject(error);
        };

        this.ws.onclose = () => {
          console.log('WebSocket closed');
          this.isConnecting = false;
          this.attemptReconnect();
        };
      } catch (error) {
        this.isConnecting = false;
        reject(error);
      }
    });
  }

  private attemptReconnect() {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error('Max reconnection attempts reached');
      return;
    }

    this.reconnectAttempts++;
    console.log(`Attempting to reconnect (${this.reconnectAttempts}/${this.maxReconnectAttempts})...`);

    setTimeout(() => {
      if (this.token) {
        this.connect(this.token).catch((error) => {
          console.error('Reconnection failed:', error);
        });
      }
    }, this.reconnectDelay);
  }

  disconnect() {
    if (this.ws) {
      this.ws.close();
      this.ws = null;
    }
    this.token = null;
    this.reconnectAttempts = 0;
  }

  send(message: WebSocketMessage) {
    if (!this.ws || this.ws.readyState !== WebSocket.OPEN) {
      console.error('WebSocket is not connected');
      return;
    }

    this.ws.send(JSON.stringify(message));
  }

  on(event: string, handler: (message: WebSocketMessage) => void) {
    this.messageHandlers.set(event, handler);
  }

  off(event: string) {
    this.messageHandlers.delete(event);
  }

  private handleMessage(message: WebSocketMessage) {
    // Handle by event name
    const handler = this.messageHandlers.get(message.event);
    if (handler) {
      handler(message);
    }

    // Handle by namespace + event
    const namespaceHandler = this.messageHandlers.get(`${message.namespace}:${message.event}`);
    if (namespaceHandler) {
      namespaceHandler(message);
    }
  }

  // Room-specific methods
  createRoom(maxPlayers: number, isPublic: boolean, betValue: number, password?: number) {
    this.send({
      namespace: 'room',
      event: 'create_room',
      data: {
        max_players: maxPlayers,
        is_public: isPublic,
        bet_value: betValue,
        ...(password && { password }),
      },
    });
  }

  joinRoom(roomId: string, password?: number) {
    this.send({
      namespace: 'room',
      event: 'join_room',
      data: {
        room_id: roomId,
        ...(password && { password }),
      },
    });
  }

  leaveRoom(roomId: string) {
    this.send({
      namespace: 'room',
      event: 'leave_room',
      data: {
        room_id: roomId,
      },
    });
  }

  isConnected(): boolean {
    return this.ws !== null && this.ws.readyState === WebSocket.OPEN;
  }
}
```

### Game State Manager

Create `assets/scripts/game/GameStateManager.ts`:

```typescript
import { GameStateDTO, PieceStateDTO, PlayerBalanceDTO, RollHistoryDTO } from '../network/types';
import { ApiService } from '../network/ApiService';

export class GameStateManager {
  private apiService: ApiService;
  private currentMatchId: string | null = null;
  private gameState: GameStateDTO | null = null;
  private pieces: PieceStateDTO[] = [];
  private balances: PlayerBalanceDTO[] = [];
  private rollHistory: RollHistoryDTO[] = [];
  private pollingInterval: number | null = null;
  private pollingDelay: number = 2000; // 2 seconds

  constructor(apiService: ApiService) {
    this.apiService = apiService;
  }

  async loadGameState(matchId: string): Promise<GameStateDTO> {
    this.currentMatchId = matchId;
    const response = await this.apiService.getGameState(matchId);
    this.gameState = response.data;
    return this.gameState;
  }

  async loadPieces(matchId: string): Promise<PieceStateDTO[]> {
    const response = await this.apiService.getPieceStates(matchId);
    this.pieces = response.data;
    return this.pieces;
  }

  async loadBalances(matchId: string): Promise<PlayerBalanceDTO[]> {
    const response = await this.apiService.getPlayerBalances(matchId);
    this.balances = response.data;
    return this.balances;
  }

  async loadRollHistory(matchId: string): Promise<RollHistoryDTO[]> {
    const response = await this.apiService.getRollHistory(matchId);
    this.rollHistory = response.data;
    return this.rollHistory;
  }

  async initializeGame(matchId: string): Promise<void> {
    await this.apiService.initializeGame(matchId);
    await this.loadGameState(matchId);
    await this.loadPieces(matchId);
    await this.loadBalances(matchId);
  }

  async rollDice(matchId: string): Promise<RollHistoryDTO> {
    const response = await this.apiService.rollDice(matchId);
    const roll = response.data;
    this.rollHistory.push(roll);
    return roll;
  }

  startPolling(onUpdate?: (gameState: GameStateDTO) => void) {
    if (this.pollingInterval) {
      this.stopPolling();
    }

    if (!this.currentMatchId) {
      console.error('No match ID set for polling');
      return;
    }

    this.pollingInterval = setInterval(async () => {
      try {
        if (this.currentMatchId) {
          const gameState = await this.loadGameState(this.currentMatchId);
          if (onUpdate) {
            onUpdate(gameState);
          }
        }
      } catch (error) {
        console.error('Polling error:', error);
      }
    }, this.pollingDelay) as any;
  }

  stopPolling() {
    if (this.pollingInterval) {
      clearInterval(this.pollingInterval);
      this.pollingInterval = null;
    }
  }

  getGameState(): GameStateDTO | null {
    return this.gameState;
  }

  getPieces(): PieceStateDTO[] {
    return this.pieces;
  }

  getBalances(): PlayerBalanceDTO[] {
    return this.balances;
  }

  getRollHistory(): RollHistoryDTO[] {
    return this.rollHistory;
  }

  isMyTurn(userId: string): boolean {
    if (!this.gameState) return false;
    return this.gameState.current_player_seat_index === this.getPlayerSeatIndex(userId);
  }

  getPlayerSeatIndex(userId: string): number {
    if (!this.gameState) return -1;
    const player = this.gameState.board_state.players.find((p) => p.user_id === userId);
    return player ? player.seat_index : -1;
  }

  getPlayerPieces(userId: string): PieceStateDTO[] {
    return this.pieces.filter((piece) => piece.user_id === userId);
  }

  getPlayerBalance(userId: string): PlayerBalanceDTO | undefined {
    return this.balances.find((balance) => balance.user_id === userId);
  }

  cleanup() {
    this.stopPolling();
    this.currentMatchId = null;
    this.gameState = null;
    this.pieces = [];
    this.balances = [];
    this.rollHistory = [];
  }
}
```

### Room Manager

Create `assets/scripts/game/RoomManager.ts`:

```typescript
import { WebSocketManager } from '../network/WebSocketManager';
import { Room, RoomWebSocketMessage } from '../network/types';

export class RoomManager {
  private wsManager: WebSocketManager;
  private currentRoom: Room | null = null;
  private onRoomUpdate?: (room: Room) => void;
  private onPlayerJoined?: (data: any) => void;
  private onPlayerLeft?: (data: any) => void;

  constructor(wsManager: WebSocketManager) {
    this.wsManager = wsManager;
    this.setupEventHandlers();
  }

  private setupEventHandlers() {
    // Handle room creation
    this.wsManager.on('room:create_room', (message: RoomWebSocketMessage) => {
      if (message.type === 'success' && message.data) {
        this.currentRoom = message.data as Room;
        if (this.onRoomUpdate) {
          this.onRoomUpdate(this.currentRoom);
        }
      }
    });

    // Handle room join
    this.wsManager.on('room:join_room', (message: RoomWebSocketMessage) => {
      if (message.type === 'success' && message.data) {
        this.currentRoom = message.data as Room;
        if (this.onRoomUpdate) {
          this.onRoomUpdate(this.currentRoom);
        }
      }
    });

    // Handle user joined
    this.wsManager.on('room:user_joined', (message: RoomWebSocketMessage) => {
      if (message.data) {
        if (this.currentRoom) {
          this.currentRoom.current_players++;
        }
        if (this.onPlayerJoined) {
          this.onPlayerJoined(message.data);
        }
        if (this.onRoomUpdate) {
          this.onRoomUpdate(this.currentRoom!);
        }
      }
    });

    // Handle user left
    this.wsManager.on('room:user_left', (message: RoomWebSocketMessage) => {
      if (message.data) {
        if (this.currentRoom) {
          this.currentRoom.current_players--;
        }
        if (this.onPlayerLeft) {
          this.onPlayerLeft(message.data);
        }
        if (this.onRoomUpdate) {
          this.onRoomUpdate(this.currentRoom!);
        }
      }
    });
  }

  createRoom(maxPlayers: number, isPublic: boolean, betValue: number, password?: number) {
    this.wsManager.createRoom(maxPlayers, isPublic, betValue, password);
  }

  joinRoom(roomId: string, password?: number) {
    this.wsManager.joinRoom(roomId, password);
  }

  leaveRoom(roomId: string) {
    if (this.currentRoom) {
      this.wsManager.leaveRoom(this.currentRoom.id);
      this.currentRoom = null;
    }
  }

  getCurrentRoom(): Room | null {
    return this.currentRoom;
  }

  setOnRoomUpdate(handler: (room: Room) => void) {
    this.onRoomUpdate = handler;
  }

  setOnPlayerJoined(handler: (data: any) => void) {
    this.onPlayerJoined = handler;
  }

  setOnPlayerLeft(handler: (data: any) => void) {
    this.onPlayerLeft = handler;
  }
}
```

### Gameplay Manager

Create `assets/scripts/game/GameplayManager.ts`:

```typescript
import { GameStateManager } from './GameStateManager';
import { ApiService } from '../network/ApiService';
import { GameStateDTO, PieceStateDTO } from '../network/types';

export class GameplayManager {
  private gameStateManager: GameStateManager;
  private apiService: ApiService;
  private currentUserId: string | null = null;

  constructor(apiService: ApiService, gameStateManager: GameStateManager) {
    this.apiService = apiService;
    this.gameStateManager = gameStateManager;
  }

  setCurrentUserId(userId: string) {
    this.currentUserId = userId;
  }

  async initializeGame(matchId: string): Promise<void> {
    await this.gameStateManager.initializeGame(matchId);
  }

  async rollDice(matchId: string): Promise<any> {
    if (!this.isMyTurn()) {
      throw new Error('Not your turn');
    }
    return await this.gameStateManager.rollDice(matchId);
  }

  isMyTurn(): boolean {
    if (!this.currentUserId) return false;
    return this.gameStateManager.isMyTurn(this.currentUserId);
  }

  getCurrentGameState(): GameStateDTO | null {
    return this.gameStateManager.getGameState();
  }

  getMyPieces(): PieceStateDTO[] {
    if (!this.currentUserId) return [];
    return this.gameStateManager.getPlayerPieces(this.currentUserId);
  }

  startGameLoop(onStateUpdate?: (gameState: GameStateDTO) => void) {
    this.gameStateManager.startPolling(onStateUpdate);
  }

  stopGameLoop() {
    this.gameStateManager.stopPolling();
  }

  cleanup() {
    this.stopGameLoop();
    this.gameStateManager.cleanup();
  }
}
```

### Cocos Creator Integration Example

#### Login Scene Component

```typescript
import { _decorator, Component, Node, EditBox, Button, Label } from 'cc';
import { ApiService } from '../network/ApiService';
import { WebSocketManager } from '../network/WebSocketManager';

const { ccclass, property } = _decorator;

@ccclass('LoginScene')
export class LoginScene extends Component {
  @property(EditBox)
  usernameInput: EditBox = null!;

  @property(EditBox)
  passwordInput: EditBox = null!;

  @property(Button)
  loginButton: Button = null!;

  @property(Label)
  statusLabel: Label = null!;

  private apiService: ApiService;
  private wsManager: WebSocketManager;

  onLoad() {
    this.apiService = new ApiService('http://localhost:3003');
    this.wsManager = new WebSocketManager('ws://localhost:3003');

    this.loginButton.node.on(Button.EventType.CLICK, this.onLoginClick, this);
  }

  async onLoginClick() {
    const username = this.usernameInput.string;
    const password = this.passwordInput.string;

    if (!username || !password) {
      this.statusLabel.string = 'Please enter username and password';
      return;
    }

    try {
      this.statusLabel.string = 'Logging in...';
      const response = await this.apiService.login(username, password);
      const { access_token, user } = response.data;

      // Store token
      this.apiService.setToken(access_token);

      // Connect WebSocket
      await this.wsManager.connect(access_token);

      // Store user info
      localStorage.setItem('token', access_token);
      localStorage.setItem('userId', user.id);
      localStorage.setItem('username', user.username);

      // Navigate to room scene
      cc.director.loadScene('RoomScene');
    } catch (error: any) {
      this.statusLabel.string = `Login failed: ${error.message}`;
    }
  }
}
```

#### Room Scene Component

```typescript
import { _decorator, Component, Node, Button, Label, EditBox } from 'cc';
import { ApiService } from '../network/ApiService';
import { WebSocketManager } from '../network/WebSocketManager';
import { RoomManager } from '../game/RoomManager';

const { ccclass, property } = _decorator;

@ccclass('RoomScene')
export class RoomScene extends Component {
  @property(Button)
  createRoomButton: Button = null!;

  @property(Button)
  joinRoomButton: Button = null!;

  @property(EditBox)
  roomIdInput: EditBox = null!;

  @property(Label)
  roomInfoLabel: Label = null!;

  @property(Label)
  playersLabel: Label = null!;

  private apiService: ApiService;
  private wsManager: WebSocketManager;
  private roomManager: RoomManager;

  onLoad() {
    const token = localStorage.getItem('token');
    if (!token) {
      cc.director.loadScene('LoginScene');
      return;
    }

    this.apiService = new ApiService('http://localhost:3003');
    this.apiService.setToken(token);

    this.wsManager = new WebSocketManager('ws://localhost:3003');
    this.wsManager.connect(token);

    this.roomManager = new RoomManager(this.wsManager);
    this.roomManager.setOnRoomUpdate((room) => {
      this.updateRoomUI(room);
    });

    this.createRoomButton.node.on(Button.EventType.CLICK, this.onCreateRoom, this);
    this.joinRoomButton.node.on(Button.EventType.CLICK, this.onJoinRoom, this);
  }

  onCreateRoom() {
    this.roomManager.createRoom(4, true, 100000);
  }

  onJoinRoom() {
    const roomId = this.roomIdInput.string;
    if (!roomId) {
      this.roomInfoLabel.string = 'Please enter room ID';
      return;
    }
    this.roomManager.joinRoom(roomId);
  }

  updateRoomUI(room: any) {
    if (room) {
      this.roomInfoLabel.string = `Room: ${room.id}\nPlayers: ${room.current_players}/${room.max_players}`;
      this.playersLabel.string = `Status: ${room.status}`;
    }
  }

  onDestroy() {
    this.wsManager.disconnect();
  }
}
```

#### Game Scene Component

```typescript
import { _decorator, Component, Node, Button, Label, Sprite } from 'cc';
import { ApiService } from '../network/ApiService';
import { GameStateManager } from '../game/GameStateManager';
import { GameplayManager } from '../game/GameplayManager';
import { GameStateDTO } from '../network/types';

const { ccclass, property } = _decorator;

@ccclass('GameScene')
export class GameScene extends Component {
  @property(Button)
  rollDiceButton: Button = null!;

  @property(Label)
  turnLabel: Label = null!;

  @property(Label)
  timerLabel: Label = null!;

  @property(Node)
  boardNode: Node = null!;

  private apiService: ApiService;
  private gameStateManager: GameStateManager;
  private gameplayManager: GameplayManager;
  private matchId: string = '';
  private userId: string = '';

  onLoad() {
    const token = localStorage.getItem('token');
    const userId = localStorage.getItem('userId');
    const matchId = localStorage.getItem('matchId');

    if (!token || !userId || !matchId) {
      cc.director.loadScene('RoomScene');
      return;
    }

    this.userId = userId;
    this.matchId = matchId;

    this.apiService = new ApiService('http://localhost:3003');
    this.apiService.setToken(token);

    this.gameStateManager = new GameStateManager(this.apiService);
    this.gameplayManager = new GameplayManager(this.apiService, this.gameStateManager);
    this.gameplayManager.setCurrentUserId(userId);

    this.rollDiceButton.node.on(Button.EventType.CLICK, this.onRollDice, this);

    this.startGame();
  }

  async startGame() {
    try {
      await this.gameplayManager.initializeGame(this.matchId);
      this.gameplayManager.startGameLoop((gameState) => {
        this.updateGameUI(gameState);
      });
    } catch (error: any) {
      console.error('Failed to start game:', error);
    }
  }

  async onRollDice() {
    if (!this.gameplayManager.isMyTurn()) {
      this.turnLabel.string = 'Not your turn!';
      return;
    }

    try {
      const roll = await this.gameplayManager.rollDice(this.matchId);
      this.turnLabel.string = `Rolled: ${roll.result}`;
    } catch (error: any) {
      this.turnLabel.string = `Error: ${error.message}`;
    }
  }

  updateGameUI(gameState: GameStateDTO) {
    this.turnLabel.string = `Turn: ${gameState.current_turn}`;
    this.timerLabel.string = `Timer: ${gameState.timer_seconds}s`;

    // Update roll button state
    this.rollDiceButton.interactable = this.gameplayManager.isMyTurn();

    // Render board pieces
    this.renderBoard(gameState);
  }

  renderBoard(gameState: GameStateDTO) {
    // Implement board rendering logic
    // Update piece positions based on gameState.board_state
  }

  onDestroy() {
    this.gameplayManager.cleanup();
  }
}
```

### Configuration

Create a configuration file `assets/scripts/config/GameConfig.ts`:

```typescript
export class GameConfig {
  // API Configuration
  static readonly API_BASE_URL = 'http://localhost:3003';
  static readonly WS_BASE_URL = 'ws://localhost:3003';

  // Game Configuration
  static readonly POLLING_INTERVAL = 2000; // 2 seconds
  static readonly MAX_RECONNECT_ATTEMPTS = 5;
  static readonly RECONNECT_DELAY = 3000; // 3 seconds

  // Board Configuration
  static readonly TOTAL_CELLS = 56;
  static readonly CELLS_PER_DIRECTION = 14;
  static readonly HOME_CELLS_PER_PLAYER = 4;
  static readonly FINISH_CELL_PER_PLAYER = 1;

  // Player Colors
  static readonly PLAYER_COLORS = ['red', 'blue', 'yellow', 'purple'] as const;
}
```

### Best Practices

1. **Error Handling**: Always wrap API calls in try-catch blocks
2. **Token Management**: Store tokens securely and refresh when expired
3. **Connection Management**: Reconnect WebSocket on disconnect
4. **State Synchronization**: Use polling for game state, WebSocket for events
5. **Memory Management**: Clean up intervals and connections on scene destroy
6. **UI Updates**: Use Cocos Creator's scheduler for UI updates
7. **Network Status**: Check network connectivity before API calls

### Testing in Cocos Creator

1. **Build and Run**: Build for Web platform to test WebSocket connections
2. **Debug Console**: Use browser console for debugging
3. **Network Tab**: Monitor API calls in browser DevTools
4. **WebSocket Inspector**: Use browser WebSocket inspector for debugging

---

## 🎲 Complete Game Flow

### Phase 1: User Setup
```javascript
// 1. Register/Login
async function setupUser() {
    // Login
    const loginResponse = await fetch('/iam/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
            username: 'player123',
            password: 'password123'
        })
    });
    const { data } = await loginResponse.json();
    const token = data.access_token;
    
    // Create balance if needed
    await fetch('/wallet/create-user-balance', {
        method: 'POST',
        headers: {
            'Authorization': `Bearer ${token}`,
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            user_id: data.user.id
        })
    });
    
    // Credit initial balance
    await fetch('/wallet/credit', {
        method: 'POST',
        headers: {
            'Authorization': `Bearer ${token}`,
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            amount: 1000000,
            reference_id: 'initial_deposit'
        })
    });
    
    return token;
}
```

### Phase 2: Room Management
```javascript
// 2. Connect WebSocket and create/join room
async function joinGameRoom(token) {
    const ws = new WebSocket(`ws://localhost:3003/ws?token=${token}`);
    
    await new Promise(resolve => {
        ws.onopen = resolve;
    });
    
    // Create room
    ws.send(JSON.stringify({
        namespace: 'room',
        event: 'create_room',
        data: {
            max_players: 4,
            is_public: true,
            bet_value: 100000
        }
    }));
    
    // Wait for room creation
    return new Promise(resolve => {
        ws.onmessage = function(event) {
            const message = JSON.parse(event.data);
            if (message.event === 'create_room' && message.type === 'success') {
                resolve({ ws, roomId: message.data.id });
            }
        };
    });
}
```

### Phase 3: Match Management
```javascript
// 3. Start match (when room is full or ready)
async function startMatch(token, matchId) {
    // Initialize game
    const initResponse = await fetch(`/gameplay/matches/${matchId}/initialize`, {
        method: 'POST',
        headers: {
            'Authorization': `Bearer ${token}`
        }
    });
    
    if (initResponse.ok) {
        console.log('Game initialized successfully');
        return true;
    }
    return false;
}
```

### Phase 4: Gameplay Loop
```javascript
// 4. Game loop
async function gameLoop(token, matchId) {
    let gameRunning = true;
    
    while (gameRunning) {
        // Get current game state
        const stateResponse = await fetch(`/gameplay/matches/${matchId}/state`, {
            headers: { 'Authorization': `Bearer ${token}` }
        });
        const gameState = await stateResponse.json();
        
        // Check if it's player's turn
        if (isMyTurn(gameState.data)) {
            // Roll dice
            const rollResponse = await fetch(`/gameplay/matches/${matchId}/roll-dice`, {
                method: 'POST',
                headers: { 'Authorization': `Bearer ${token}` }
            });
            const rollResult = await rollResponse.json();
            
            console.log('Rolled:', rollResult.data.result);
            
            // Get piece states
            const piecesResponse = await fetch(`/gameplay/matches/${matchId}/pieces`, {
                headers: { 'Authorization': `Bearer ${token}` }
            });
            const pieces = await piecesResponse.json();
            
            // Move pieces (logic to be implemented)
            // await movePiece(token, matchId, pieceId, newPosition);
        }
        
        // Wait before next check
        await new Promise(resolve => setTimeout(resolve, 2000));
        
        // Check if game finished
        if (gameState.data.status === 'finished') {
            gameRunning = false;
        }
    }
}

function isMyTurn(gameState) {
    // Implement logic to check if it's current player's turn
    return gameState.current_player_seat === myPlayerSeat;
}
```

### Complete Integration Example
```javascript
async function playGame() {
    try {
        // Phase 1: Setup user
        const token = await setupUser();
        console.log('User setup complete');
        
        // Phase 2: Join room
        const { ws, roomId } = await joinGameRoom(token);
        console.log('Joined room:', roomId);
        
        // Wait for match to start (this would be triggered by room owner or automatically)
        // For demo, assume we have matchId
        const matchId = 'some-match-uuid';
        
        // Phase 3: Initialize game
        const gameReady = await startMatch(token, matchId);
        if (!gameReady) {
            throw new Error('Failed to initialize game');
        }
        
        // Phase 4: Play game
        await gameLoop(token, matchId);
        
        console.log('Game completed');
        
    } catch (error) {
        console.error('Game error:', error);
    }
}

// Start the game
playGame();
```

---

## ❌ Error Handling

### HTTP Error Responses
```json
{
  "error": {
    "code": "BAD_REQUEST",
    "message": "Invalid request data",
    "details": "Amount must be greater than 0"
  }
}
```

### WebSocket Error Responses
```json
{
  "namespace": "room",
  "event": "create_room", 
  "type": "error",
  "error": "Insufficient balance to create room"
}
```

### Common Error Codes
- `400 BAD_REQUEST`: Invalid input data
- `401 UNAUTHORIZED`: Invalid or expired token
- `403 FORBIDDEN`: Access denied
- `404 NOT_FOUND`: Resource not found
- `409 CONFLICT`: Resource conflict (e.g., already joined)
- `500 INTERNAL_SERVER_ERROR`: Server error

### Error Handling Best Practices
```javascript
// HTTP API error handling
async function makeApiCall(url, options) {
    try {
        const response = await fetch(url, options);
        const data = await response.json();
        
        if (!response.ok) {
            throw new Error(data.error?.message || 'API call failed');
        }
        
        return data;
    } catch (error) {
        console.error('API Error:', error);
        // Handle error appropriately
        showErrorToUser(error.message);
        throw error;
    }
}

// WebSocket error handling
ws.onmessage = function(event) {
    const message = JSON.parse(event.data);
    
    if (message.type === 'error') {
        console.error('WebSocket Error:', message.error);
        showErrorToUser(message.error);
        return;
    }
    
    // Handle success message
    handleMessage(message);
};
```

---

## 🧪 Testing

### Manual Testing with cURL

#### Test Login
```bash
curl -X POST http://localhost:3003/iam/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "password": "admin123"
  }'
```

#### Test Wallet Credit
```bash
curl -X POST http://localhost:3003/wallet/credit \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 1000000,
    "reference_id": "test_credit"
  }'
```

### WebSocket Testing
Use the provided WebSocket test file:
```
/Users/admin/Documents/ludo-backend/test/websocket_test.html
```

Open in browser and test WebSocket connections with JWT tokens.

### Swagger Documentation
Access API documentation at:
```
http://localhost:3003/swagger/index.html
```

---

## 📈 Performance Considerations

### API Rate Limits
- No specific rate limits implemented currently
- Consider implementing rate limiting for production

### WebSocket Connection Management
- Maintain single WebSocket connection per client
- Implement reconnection logic for dropped connections
- Handle connection cleanup on page unload

### Polling vs Real-time
- **Use WebSocket for**: Room events, player join/leave, real-time notifications
- **Use HTTP API for**: Game state queries, dice rolls, balance checks
- **Recommended polling interval**: 2-5 seconds for game state

---

## 🔧 Development Setup

### Start the Server
```bash
# Setup database and dependencies
make migrate-up
make deps

# Start API server
make dev

# Start worker (for wallet operations)
make worker

# Start monitoring dashboard
make monitor
```

### Environment Variables
Create `.env` file:
```env
PORT=3003
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=password
DB_DATABASE=ludo
JWT_ACCESS_SECRET=dev-access-secret
JWT_REFRESH_SECRET=dev-refresh-secret
REDIS_ADDR=localhost:6379
```

---

## 📞 Support

### Endpoints for Help
- **Swagger UI**: `http://localhost:3003/swagger/index.html`
- **Asynq Monitor**: `http://localhost:9090/monitor`

### Default Admin Credentials
- Username: `admin`
- Password: `admin123`

### Debug Tips
1. Check server logs for detailed error messages
2. Use browser developer tools for WebSocket debugging
3. Verify JWT token is not expired
4. Ensure database is running and migrated
5. Check Redis connection for queue operations

---

**Happy Gaming! 🎮**

