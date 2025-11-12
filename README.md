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
