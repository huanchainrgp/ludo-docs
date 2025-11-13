# 🎲 Giải Thích Chi Tiết Bảng `piece_states`

## 📋 Tổng Quan

Bảng `piece_states` là **bảng quan trọng nhất** trong hệ thống gameplay, lưu trữ **trạng thái hiện tại của từng quân cờ** trong game Ludo. 

**Cấu hình hiện tại:** Mỗi match có **8 records** (4 players × 2 pieces per player)

---

## 🗄️ Database Schema

### **Table Structure**

```sql
CREATE TABLE piece_states (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    match_id UUID NOT NULL,           -- Match mà piece thuộc về
    user_id UUID NOT NULL,             -- Player sở hữu piece
    status VARCHAR(20) NOT NULL,       -- Trạng thái: in_home, in_track, in_finish
    color VARCHAR(20) NOT NULL,        -- Màu: red, blue, yellow, purple
    position INTEGER NOT NULL DEFAULT 0, -- Vị trí hiện tại (0-55)
    home_index INTEGER NOT NULL DEFAULT 0, -- Vị trí xuất phát
    finish_index INTEGER NOT NULL DEFAULT 0, -- Vị trí đích
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_piece_states_match_id ON piece_states(match_id);
CREATE INDEX idx_piece_states_user_id ON piece_states(user_id);
CREATE INDEX idx_piece_states_status ON piece_states(status);
CREATE INDEX idx_piece_states_color ON piece_states(color);
```

---

## 📊 Fields Explanation

### **1. `id` (UUID)**
- **Primary Key** - Unique identifier cho mỗi quân cờ
- Auto-generated khi tạo record
- Không bao giờ thay đổi trong suốt game

### **2. `match_id` (UUID)**
- **Foreign Key** tới bảng `matches`
- Xác định quân cờ thuộc match nào
- Indexed để query nhanh tất cả pieces của 1 match

### **3. `user_id` (UUID)**
- **Foreign Key** tới bảng `users`
- Xác định player sở hữu quân cờ
- Indexed để query pieces của 1 player

### **4. `status` (VARCHAR(20)) - ⭐ QUAN TRỌNG**

**3 trạng thái chính:**

#### **a) `in_home`** - Quân ở nhà
```
Điều kiện:
- Quân chưa xuất phát
- Quân bị kick về nhà (bị quân địch đá)
- Position = HomeIndex

Hành động có thể:
- ❌ Không thể di chuyển (trừ khi roll = 6)
- ✅ Roll = 6 → Có thể ra khỏi nhà → Status = in_track

Ví dụ:
{
  "id": "piece-uuid-1",
  "user_id": "player-red",
  "status": "in_home",
  "position": 0,      // = home_index
  "home_index": 0,
  "finish_index": 13
}
```

#### **b) `in_track`** - Quân đang trên đường đua
```
Điều kiện:
- Quân đã xuất phát từ nhà
- Đang di chuyển trên board (56 cells)
- Chưa về đích
- HomeIndex < Position < FinishIndex

Hành động có thể:
- ✅ Di chuyển theo kết quả roll dice
- ✅ Có thể kick quân địch
- ⚠️ Có thể bị kick về home (nếu không ở safe zone)

Ví dụ:
{
  "id": "piece-uuid-1",
  "user_id": "player-red",
  "status": "in_track",
  "position": 8,      // Đang ở cell 8
  "home_index": 0,
  "finish_index": 13
}
```

#### **c) `in_finish`** - Quân đã về đích
```
Điều kiện:
- Quân đã về finish area
- Position >= FinishIndex
- Không thể di chuyển nữa

Hành động có thể:
- ❌ Không thể di chuyển (đã finish)
- ❌ Không thể bị kick
- ✅ Đóng góp vào điều kiện thắng

Ví dụ:
{
  "id": "piece-uuid-1",
  "user_id": "player-red",
  "status": "in_finish",
  "position": 13,     // = finish_index
  "home_index": 0,
  "finish_index": 13
}
```

### **5. `color` (VARCHAR(20))**

**4 màu theo seat index:**
```
Seat 0 → Red (Đỏ)
Seat 1 → Blue (Xanh Dương)
Seat 2 → Yellow (Vàng)
Seat 3 → Purple (Tím)
```

**Mục đích:**
- Phân biệt quân cờ trên UI
- Xác định path di chuyển
- Map với player area (4 góc bàn cờ)

### **6. `position` (INTEGER) - ⭐ QUAN TRỌNG**

**Vị trí hiện tại của quân trên board (0-55)**

#### **Position Mapping:**

```
Board Layout (56 cells):
┌────────────────────────────────────────┐
│  0 -  13: Red path    (Player 0)       │
│ 14 - 27: Blue path   (Player 1)       │
│ 28 - 41: Yellow path (Player 2)       │
│ 42 - 55: Purple path (Player 3)       │
└────────────────────────────────────────┘

Visual Board:
        [Yellow Path: 28-41]
              │
    ┌─────────┼─────────┐
    │         ↓         │
[Purple] ←  FINISH  → [Blue]
    │       AREA       │
    │         ↑         │
    └─────────┼─────────┘
              │
        [Red Path: 0-13]
```

#### **Position States:**

| Position Value | Meaning | Status |
|---------------|---------|---------|
| `position = home_index` | Quân ở nhà | `in_home` |
| `home_index < position < finish_index` | Quân đang chạy | `in_track` |
| `position = finish_index` | Quân về đích | `in_finish` |

#### **Position Changes:**

```typescript
// Roll dice = 5, piece tại position 3
beforePosition = 3
rollResult = 5
afterPosition = 3 + 5 = 8

// Update piece_states
UPDATE piece_states 
SET position = 8, status = 'in_track'
WHERE id = 'piece-uuid-1';

// Save history
INSERT INTO piece_race_histories (piece_id, from_index, to_index, turn)
VALUES ('piece-uuid-1', 3, 8, 1);
```

### **7. `home_index` (INTEGER) - IMMUTABLE**

**Vị trí xuất phát của quân (không bao giờ thay đổi)**

#### **Home Index theo Player:**

```
Player Red (Seat 0):    home_index = 0
Player Blue (Seat 1):   home_index = 14
Player Yellow (Seat 2): home_index = 28
Player Purple (Seat 3): home_index = 42
```

#### **Công thức:**
```go
homeIndex := seatIndex * 14  // 0, 14, 28, 42
```

#### **Sử dụng:**
1. **Khởi tạo:** Position ban đầu = home_index
2. **Reset khi bị kick:**
   ```go
   piece.Position = piece.HomeIndex
   piece.Status = PieceStatusInHome
   ```

### **8. `finish_index` (INTEGER) - IMMUTABLE**

**Vị trí đích của quân (không bao giờ thay đổi)**

#### **Finish Index theo Player:**

```
Player Red (Seat 0):    finish_index = 13
Player Blue (Seat 1):   finish_index = 27
Player Yellow (Seat 2): finish_index = 41
Player Purple (Seat 3): finish_index = 55
```

#### **Công thức:**
```go
finishIndex := ((seatIndex + 1) * 14) - 1  // 13, 27, 41, 55
```

#### **Sử dụng:**
1. **Kiểm tra về đích:**
   ```go
   if toIndex >= piece.FinishIndex {
       toIndex = piece.FinishIndex
       piece.Status = PieceStatusInFinishLine
   }
   ```

2. **Điều kiện thắng:**
   ```go
   // Player thắng khi tất cả 2 pieces có status = in_finish
   finishedCount := 0
   for _, piece := range playerPieces {
       if piece.Status == PieceStatusInFinishLine {
           finishedCount++
       }
   }
   
   // Player wins if all 2 pieces are finished
   if finishedCount == 2 {
       match.Finish(userID)  // Player wins!
   }
   ```

---

## 🔄 Lifecycle của Piece State

### **1. Initialization (Khi Initialize Game)**

```go
// File: internal/modules/gameplay/application/use-cases/initialize_game.go

func initializePieces(matchID, playerOrder) []*PieceState {
    pieces := make([]*PieceState, 0, 8)  // 4 players × 2 pieces
    
    for seatIndex, userID := range playerOrder {
        homeIndex := seatIndex * 14
        finishIndex := ((seatIndex + 1) * 14) - 1
        color := colors[seatIndex]  // red, blue, yellow, purple
        
        // Tạo 2 pieces cho mỗi player
        for i := 0; i < 2; i++ {
            piece := &PieceState{
                MatchID:     matchID,
                UserID:      userID,
                Status:      PieceStatusInHome,     // Bắt đầu ở nhà
                Color:       color,
                Position:    homeIndex,              // = home_index
                HomeIndex:   homeIndex,              // Không đổi
                FinishIndex: finishIndex,            // Không đổi
            }
            pieces = append(pieces, piece)
        }
    }
    
    // Save to database
    pieceStateRepository.CreateBatch(ctx, pieces)
    
    return pieces
}
```

**Kết quả:** 8 records trong bảng `piece_states`

```json
[
  // Player Red (Seat 0) - 2 pieces
  {
    "id": "piece-red-1",
    "match_id": "match-123",
    "user_id": "player-red",
    "status": "in_home",
    "color": "red",
    "position": 0,
    "home_index": 0,
    "finish_index": 13
  },
  {
    "id": "piece-red-2",
    "match_id": "match-123",
    "user_id": "player-red",
    "status": "in_home",
    "color": "red",
    "position": 0,
    "home_index": 0,
    "finish_index": 13
  },
  
  // Player Blue (Seat 1) - 2 pieces
  {
    "id": "piece-blue-1",
    "match_id": "match-123",
    "user_id": "player-blue",
    "status": "in_home",
    "color": "blue",
    "position": 14,
    "home_index": 14,
    "finish_index": 27
  },
  {
    "id": "piece-blue-2",
    "match_id": "match-123",
    "user_id": "player-blue",
    "status": "in_home",
    "color": "blue",
    "position": 14,
    "home_index": 14,
    "finish_index": 27
  },
  
  // Player Yellow (Seat 2) - 2 pieces
  {
    "id": "piece-yellow-1",
    "match_id": "match-123",
    "user_id": "player-yellow",
    "status": "in_home",
    "color": "yellow",
    "position": 28,
    "home_index": 28,
    "finish_index": 41
  },
  {
    "id": "piece-yellow-2",
    "match_id": "match-123",
    "user_id": "player-yellow",
    "status": "in_home",
    "color": "yellow",
    "position": 28,
    "home_index": 28,
    "finish_index": 41
  },
  
  // Player Purple (Seat 3) - 2 pieces
  {
    "id": "piece-purple-1",
    "match_id": "match-123",
    "user_id": "player-purple",
    "status": "in_home",
    "color": "purple",
    "position": 42,
    "home_index": 42,
    "finish_index": 55
  },
  {
    "id": "piece-purple-2",
    "match_id": "match-123",
    "user_id": "player-purple",
    "status": "in_home",
    "color": "purple",
    "position": 42,
    "home_index": 42,
    "finish_index": 55
  }
]
```

---

### **2. Move from Home (Roll = 6)**

**Scenario:** Player Red roll được 6, muốn đưa quân ra khỏi nhà

```go
// File: internal/modules/gameplay/application/services/move.go

func (uc *MovePieceUseCase) Execute(ctx, input MovePieceInput) error {
    // 1. Get piece
    piece := uc.getPieceById(input.PieceID)
    
    // 2. Check if piece in home
    if piece.Status == PieceStatusInHome {
        // Require roll = 6 to leave home
        if rollResult < 6 {
            return ErrPieceInHomeRequiresRoll
        }
        
        // 3. Update piece state
        piece.Position = piece.HomeIndex  // Vẫn ở home_index
        piece.Status = PieceStatusInTrack  // Thay đổi status!
        
        // 4. Save
        uc.pieceStateRepository.Update(ctx, piece)
    }
    
    return nil
}
```

**Before:**
```json
{
  "id": "piece-red-1",
  "status": "in_home",
  "position": 0,
  "home_index": 0,
  "finish_index": 13
}
```

**After:**
```json
{
  "id": "piece-red-1",
  "status": "in_track",  // ✅ Changed
  "position": 0,          // Still at home_index
  "home_index": 0,
  "finish_index": 13
}
```

---

### **3. Normal Move (In Track)**

**Scenario:** Player Red roll được 5, di chuyển quân từ position 3 → 8

```go
func (uc *MovePieceUseCase) Execute(ctx, input) error {
    piece := uc.getActivePiece(userID)
    
    // 1. Calculate new position
    fromIndex := piece.Position  // 3
    rollResult := 5
    toIndex := fromIndex + rollResult  // 8
    
    // 2. Check không vượt finish
    if toIndex > piece.FinishIndex {
        toIndex = piece.FinishIndex
        piece.Status = PieceStatusInFinishLine
    }
    
    // 3. Update piece
    piece.Position = toIndex
    
    // 4. Save
    uc.pieceStateRepository.Update(ctx, piece)
    
    // 5. Save history
    uc.pieceRaceHistoryRepository.Create(ctx, &PieceRaceHistory{
        PieceID:   piece.ID,
        FromIndex: fromIndex,  // 3
        ToIndex:   toIndex,     // 8
        Turn:      currentTurn,
    })
    
    return nil
}
```

**Before:**
```json
{
  "id": "piece-red-1",
  "status": "in_track",
  "position": 3,
  "home_index": 0,
  "finish_index": 13
}
```

**After:**
```json
{
  "id": "piece-red-1",
  "status": "in_track",
  "position": 8,  // ✅ Changed: 3 → 8
  "home_index": 0,
  "finish_index": 13
}
```

**History Record:**
```json
{
  "id": "history-uuid",
  "piece_id": "piece-red-1",
  "from_index": 3,
  "to_index": 8,
  "turn": 1
}
```

---

### **4. Kick Enemy Piece**

**Scenario:** Player Red di chuyển tới position 8, có quân Blue ở đó → Kick!

```go
func (uc *MovePieceUseCase) Execute(ctx, input) error {
    // 1. Move piece
    myPiece.Position = toIndex  // 8
    
    // 2. Check for enemies at landing position
    enemies := findEnemyPiecesAtIndex(allPieces, myUserID, toIndex)
    
    // 3. Reset enemies to home
    for _, enemy := range enemies {
        uc.resetPieceToHome(ctx, enemy, turn)
    }
    
    return nil
}

func (uc *MovePieceUseCase) resetPieceToHome(ctx, piece, turn) error {
    fromIndex := piece.Position  // 8
    
    // Reset position and status
    piece.Position = piece.HomeIndex   // 14
    piece.Status = PieceStatusInHome   // in_track → in_home
    
    // Save
    uc.pieceStateRepository.Update(ctx, piece)
    
    // Save kick history
    uc.pieceKickHistoryRepository.Create(ctx, &PieceKickHistory{
        PieceID:   piece.ID,
        KickedBy:  myPiece.ID,
        KickValue: fromIndex,
        Turn:      turn,
    })
    
    return nil
}
```

**Enemy Piece Before (Blue):**
```json
{
  "id": "piece-blue-1",
  "user_id": "player-blue",
  "status": "in_track",
  "position": 8,
  "home_index": 14,
  "finish_index": 27
}
```

**Enemy Piece After (Blue - KICKED):**
```json
{
  "id": "piece-blue-1",
  "user_id": "player-blue",
  "status": "in_home",     // ✅ Changed: in_track → in_home
  "position": 14,          // ✅ Changed: 8 → 14 (home_index)
  "home_index": 14,
  "finish_index": 27
}
```

**Kick History:**
```json
{
  "id": "kick-uuid",
  "piece_id": "piece-blue-1",
  "kicked_by": "piece-red-1",
  "kick_value": 8,
  "turn": 1
}
```

---

### **5. Reach Finish**

**Scenario:** Player Red roll được 4, di chuyển từ position 10 → 14 (nhưng finish = 13)

```go
func (uc *MovePieceUseCase) Execute(ctx, input) error {
    piece := uc.getActivePiece(userID)
    
    fromIndex := piece.Position  // 10
    rollResult := 4
    toIndex := fromIndex + rollResult  // 14
    
    // ⚠️ Check overshoot finish
    if toIndex > piece.FinishIndex {
        toIndex = piece.FinishIndex  // 13
        piece.Status = PieceStatusInFinishLine
    }
    
    piece.Position = toIndex
    
    uc.pieceStateRepository.Update(ctx, piece)
    
    // Check if player wins
    allPiecesFinished := uc.checkAllPiecesFinished(userID)
    if allPiecesFinished {
        match.Finish(userID)  // Player wins!
    }
    
    return nil
}
```

**Before:**
```json
{
  "id": "piece-red-1",
  "status": "in_track",
  "position": 10,
  "home_index": 0,
  "finish_index": 13
}
```

**After:**
```json
{
  "id": "piece-red-1",
  "status": "in_finish",  // ✅ Changed: in_track → in_finish
  "position": 13,         // ✅ Changed: 10 → 13 (finish_index)
  "home_index": 0,
  "finish_index": 13
}
```

---

## 🔍 Query Examples

### **1. Get All Pieces in a Match**

```sql
SELECT * FROM piece_states
WHERE match_id = 'match-uuid-123'
ORDER BY user_id, color;
```

**Use Case:** Load tất cả pieces khi render board

---

### **2. Get Player's Pieces**

```sql
SELECT * FROM piece_states
WHERE match_id = 'match-uuid-123'
  AND user_id = 'player-red-uuid'
ORDER BY created_at;
```

**Use Case:** Hiển thị pieces của 1 player cụ thể

---

### **3. Get Active Pieces (In Track)**

```sql
SELECT * FROM piece_states
WHERE match_id = 'match-uuid-123'
  AND status = 'in_track'
ORDER BY user_id;
```

**Use Case:** Tìm pieces có thể di chuyển

---

### **4. Find Pieces at Position**

```sql
SELECT * FROM piece_states
WHERE match_id = 'match-uuid-123'
  AND position = 8
  AND status = 'in_track';
```

**Use Case:** Kiểm tra collision khi di chuyển

---

### **5. Check Win Condition**

```sql
SELECT COUNT(*) as finished_count
FROM piece_states
WHERE match_id = 'match-uuid-123'
  AND user_id = 'player-red-uuid'
  AND status = 'in_finish';
  
-- If finished_count = 2 → Player wins!
```

**Use Case:** Kiểm tra player có thắng chưa

---

### **6. Get Pieces by Color**

```sql
SELECT * FROM piece_states
WHERE match_id = 'match-uuid-123'
  AND color = 'red'
ORDER BY position;
```

**Use Case:** Render pieces theo màu trên UI

---

## 🎮 Integration với UI

### **Frontend Rendering**

```typescript
// Load pieces từ API
async function loadPieces(matchId: string): Promise<PieceStateDTO[]> {
  const response = await fetch(`/api/gameplay/matches/${matchId}/pieces`);
  return response.json();
}

// Render pieces trên board
function renderPieces(pieces: PieceStateDTO[]) {
  pieces.forEach(piece => {
    const screenPosition = calculateIsometricPosition(piece.position);
    
    const pieceElement = createPieceElement({
      color: piece.color,
      position: screenPosition,
      status: piece.status,
      isSelectable: isPieceSelectable(piece),
    });
    
    // Render status indicators
    if (piece.status === 'in_home') {
      pieceElement.classList.add('in-home');
    } else if (piece.status === 'in_finish') {
      pieceElement.classList.add('finished');
      pieceElement.innerHTML += '⭐'; // Star icon
    }
    
    boardContainer.appendChild(pieceElement);
  });
}

// Check if piece can be selected
function isPieceSelectable(piece: PieceStateDTO, rollResult: number): boolean {
  // Piece in home: only if rolled 6
  if (piece.status === 'in_home') {
    return rollResult === 6;
  }
  
  // Piece finished: cannot move
  if (piece.status === 'in_finish') {
    return false;
  }
  
  // Piece in track: always selectable
  return piece.status === 'in_track';
}
```

---

## 📊 Statistics & Analytics

### **Game Progress Tracking**

```sql
-- Pieces distribution by status
SELECT 
    status,
    COUNT(*) as count,
    COUNT(*) * 100.0 / (SELECT COUNT(*) FROM piece_states WHERE match_id = 'match-123') as percentage
FROM piece_states
WHERE match_id = 'match-123'
GROUP BY status;

-- Example Result (8 pieces total):
-- in_home: 4 pieces (50%)
-- in_track: 3 pieces (37.5%)
-- in_finish: 1 piece (12.5%)
```

### **Player Progress**

```sql
-- Player's progress
SELECT 
    user_id,
    SUM(CASE WHEN status = 'in_finish' THEN 1 ELSE 0 END) as pieces_finished,
    AVG(position) as avg_position
FROM piece_states
WHERE match_id = 'match-123'
GROUP BY user_id
ORDER BY pieces_finished DESC, avg_position DESC;
```

---

## ⚠️ Important Notes

### **1. Immutable Fields**
```
✅ Có thể thay đổi:
- position (mỗi lần move)
- status (in_home ↔ in_track ↔ in_finish)
- updated_at (auto)

❌ Không bao giờ thay đổi:
- id
- match_id
- user_id
- color
- home_index
- finish_index
- created_at
```

### **2. Status Transitions**

```
Allowed transitions:
in_home → in_track (roll 6)
in_track → in_home (kicked)
in_track → in_finish (reach finish)

NOT allowed:
in_home → in_finish (must go through in_track)
in_finish → anything (cannot move from finish)
```

### **3. Position Constraints**

```
in_home:   position = home_index
in_track:  home_index < position < finish_index
in_finish: position = finish_index
```

### **4. Concurrent Updates**

```sql
-- Use transactions khi update nhiều pieces
BEGIN;

-- Update my piece
UPDATE piece_states SET position = 8 WHERE id = 'piece-red-1';

-- Reset enemy piece
UPDATE piece_states SET position = 14, status = 'in_home' WHERE id = 'piece-blue-1';

-- Insert histories
INSERT INTO piece_race_histories ...
INSERT INTO piece_kick_histories ...

COMMIT;
```

---

## 🔗 Related Tables

### **Relationships:**

```
piece_states
    ├── FK → matches (match_id)
    ├── FK → users (user_id)
    └── Referenced by:
            ├── piece_race_histories (piece_id)
            └── piece_kick_histories (piece_id, kicked_by)
```

### **Data Flow:**

```
1. Initialize Game:
   game_states (created)
   ↓
   piece_states (16 records created, all in_home)

2. Player Move:
   piece_states (position updated)
   ↓
   piece_race_histories (history created)
   ↓
   piece_kick_histories (if kick occurred)

3. Check Win:
   piece_states (check all in_finish)
   ↓
   matches (status = finished)
```

---

## ✅ Summary

### **Key Points:**

1. **8 records per match** (4 players × 2 pieces per player)
2. **3 statuses:** in_home, in_track, in_finish
3. **Position range:** 0-55 (56 cells)
4. **Immutable:** home_index, finish_index, color
5. **Mutable:** position, status
6. **Tracked by:** piece_race_histories, piece_kick_histories
7. **Win condition:** All 2 pieces of a player have status = in_finish

### **Common Operations:**

- ✅ Create 8 pieces when initialize game (2 per player)
- ✅ Update position when move
- ✅ Update status when state changes
- ✅ Reset to home when kicked
- ✅ Query by match_id, user_id, status, position
- ✅ Check win condition (all 2 pieces finished)

---

**Bảng `piece_states` là trái tim của gameplay - mọi movement, kick, và win đều được reflect qua bảng này!** 🎯
