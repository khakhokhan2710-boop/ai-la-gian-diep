# 🕵️ AI LÀ GIÁN ĐIỆP — Game Specification

## Tổng quan
Game tập thể realtime (4-20 người), chơi trên điện thoại qua web. Mỗi người có 1 vai trò bí mật, nói miệng để tìm ra gián điệp, vote loại người.

## Tech Stack
- **Frontend:** HTML/CSS/JS thuần (1 file), chạy trên mọi trình duyệt (kể cả iOS Safari cũ)
- **Backend:** Firebase Realtime Database
- **Hosting:** GitHub Pages (free)
- **JS compatibility:** CHỈ dùng `var` + `function` — KHÔNG `const/let/=>/template literals/optional chaining/async-await/spread operator`
- **UI:** Mobile-first, dark theme, 3D card flip animation

## Cấu trúc Firebase
```
/rooms/{roomCode}/
├── host: string (playerId của chủ phòng)
├── hostName: string
├── total: number (số người chơi, default 2)
├── spies: number (số gián điệp, default 1)
├── whiteHats: number (số mũ trắng, default 0)
├── status: "setup" | "drawing" | "phase" | "voting" | "result" | "guess" | "gameover"
├── phase: "draw" | "draw_done" | "vote_round" | "tie"
├── pair: [string, string, string] (từ dân, từ gián điệp, gợi ý mũ trắng)
├── players: { playerId: { name, deviceId } }
├── assignments: { playerId: { role, word, hint } }
├── turnOrder: [playerId, ...]
├── currentTurn: number
├── cardsSeen: { playerId: true }
├── votes: { voterId: targetId }
├── voteTime: number (epoch ms, khi host bấm bắt đầu 3 phút)
├── eliminated: { playerId: true }
├── lastEliminated: playerId
├── lastRole: "civilian" | "spy" | "white_hat"
├── guessPlayer: playerId (khi mũ trắng bị loại)
├── winner: "civilian" | "spy" | "white_hat" | "none"
```

## State Machine
```
setup → [host cài số người, mọi người join realtime]
  ↓ host bấm "Bắt đầu"
drawing → [lần lượt rút thẻ, mỗi người chạm → flip → "Đã xong"]
  ↓ người cuối bấm "Đã xong" → phase='draw_done'
draw_done → [HOST thấy nút "⏱️ Bắt đầu 3 phút"]
            [PLAYER thấy "Chờ chủ phòng bắt đầu 3 phút..."]
  ↓ host bấm → voteTime = Date.now() + 180000
  ↓ CẢ host và player thấy:
    - Timer 3:00 đếm ngược (sync qua Firebase)
    - Name card: hiện tên người đang nói (tự động chuyển mỗi 15s)
    - Nút "⏩ Bỏ qua → Vote" (chỉ host bấm mới chạy)
3 phút sau (hoặc host bấm bỏ qua) ↓
voting → [mỗi người chọn 1 tên → bấm "Bỏ phiếu"]
          [hiện số phiếu realtime + ai vote ai]
  ↓ tất cả đã vote (hoặc hết thời gian)
result → [hiện TÊN + ROLE của người bị vote cao nhất]
          [chỉ hiện role, KHÔNG hiện từ]
  ↓ nếu role = white_hat
guess → [mũ trắng nhập từ đoán vào ô text]
  ↓ nếu đúng
gameover → winner: white_hat
  ↓ nếu sai
gameover → winner: none
  ↓ nếu vote ra gián điệp → kiểm tra win condition
     - hết gián điệp → dân thắng
     - dân còn <= 40% → gián điệp thắng
     - chưa ai thắng → vote vòng tiếp theo
```

## 3 Vai trò

### 1. Người Dân
- **Thấy:** từ chính xác (VD: "con chó")
- **Nhiệm vụ:** tìm ra gián điệp và mũ trắng
- **Thắng khi:** vote hết gián điệp

### 2. Gián Điệp
- **Thấy:** từ gần giống với dân (VD: dân="con chó" → gián điệp="con mèo")
- Có thể là đồng nghĩa, trái nghĩa, cùng chủ đề
- **Nhiệm vụ:** trà trộn, không để bị vote ra
- **Thắng khi:** dân làng còn ≤ 40% số lượng ban đầu

### 3. Mũ Trắng
- **Thấy:** gợi ý mơ hồ (VD: "có lông") — KHÔNG thấy từ chính xác
- **Nhiệm vụ:** đoán đúng từ của dân làng
- **Thắng khi:** bị vote ra → đoán đúng từ → thắng cả ván

## Luật chơi chi tiết

### 1. Tạo phòng
- Host nhập tên → "Tạo Phòng" → được mã phòng 6 ký tự
- Host thấy màn hình cài đặt:
  - Số người (default 2, có thể +/-)
  - Số gián điệp (có thể +/-)
  - Số mũ trắng (có thể +/-)
  - Danh sách người đã vào phòng (realtime)
  - Nút "Bắt đầu"

### 2. Vào phòng
- Người khác nhập tên + mã phòng → "Vào"
- Mỗi thiết bị chỉ được 1 nick (dùng deviceId trong localStorage)
- Host thấy tên người mới join ngay lập tức

### 3. Rút thẻ
- Host bấm "Bắt đầu" → phân vai ngẫu nhiên
- Mỗi người lần lượt chạm thẻ → hiệu ứng 3D flip → xem vai + từ
- Bấm "Đã xong" → ẩn → qua người tiếp theo
- Người cuối bấm "Đã xong" → phase='draw_done'

### 4. Vòng nói (3 phút)
- Host thấy nút "⏱️ Bắt đầu 3 phút"
- Player thấy "Chờ chủ phòng bắt đầu 3 phút..."
- Host bấm → CẢ 2 thấy:
  - Timer 3:00 đếm ngược (trên góc)
  - Name card: tên người đang nói (tự động chuyển mỗi 15s)
  - Nút "⏩ Bỏ qua → Vote" (chỉ host bấm được)
- Nói miệng bên ngoài theo thứ tự

### 5. Vote
- Sau 3 phút (hoặc host bấm bỏ qua) → màn vote
- Mỗi người thấy danh sách tên người còn sống
- Bấm chọn 1 người → "Bỏ phiếu"
- Số phiếu realtime (ai vote ai thấy được)
- Khi tất cả đã vote (hoặc hết giờ) → người cao phiếu nhất bị loại

### 6. Kết quả vote
- Hiện TÊN + ROLE của người bị loại
- ROLE (không từ):
  - 👤 Người Dân
  - 🕵️ Gián Điệp
  - 🎩 Mũ Trắng
- Nếu là Mũ Trắng → chuyển sang phase đoán từ

### 7. Mũ Trắng đoán từ
- Mũ Trắng thấy gợi ý + ô nhập từ
- Nhập 1 lần duy nhất:
  - Đúng → Mũ Trắng thắng cả ván
  - Sai → Mũ Trắng bị loại (không ai thắng thêm)

### 8. Kết thúc game
- **Dân thắng:** vote ra hết gián điệp
- **Gián điệp thắng:** khi dân còn ≤ 40% so với số dân ban đầu
- **Mũ Trắng thắng:** đoán đúng từ sau khi bị vote ra
- Hiện màn hình game over với icon + chữ + nút "Về phòng chờ"

### 9. Các nút khác
- **Rời phòng:** host xóa phòng, player xóa tên mình
- **Reset:** quay lại lobby

## Word Pairs
- **Bộ dữ liệu:** 550+ cặp từ
- **Định dạng JSON:** `[["từ_dân", "từ_gián_điệp", "gợi_ý_mũ_trắng"], ...]`
- **Lưu:** file `word_pairs.json` tĩnh, fetch khi load game
- **Không lưu trong Firebase** (tránh bloat)

## Ghi chú kỹ thuật
1. **100dvh** thay vì 100vh cho iOS Safari
2. **touch-action: manipulation** để tránh delay trên mobile
3. **alert()** thay console — người dùng không mở DevTools
4. **`.catch()` trên mọi Firebase write** — không có uncaught rejection
5. **`roomRef.child('players').off('value')` riêng** — parent .off() không cleanup child listener
6. **Tất cả Firebase `.remove()` cần `.catch(function(){})`** — phòng bị xóa bởi người khác
7. **Timer sync qua Firebase:** host lưu `voteTime` (epoch) → ai cũng đọc và đếm ngược

---

## 🔧 Dành cho Claude (dev)

### GitHub Repo
```
Repo:   khakhokhan2710-boop/ai-la-gian-diep
Clone:  git clone git@github.com:khakhokhan2710-boop/ai-la-gian-diep.git
Push:   git push origin main
Web:    https://khakhokhan2710-boop.github.io/ai-la-gian-diep/
```

### Firebase Config
```js
const firebaseConfig = {
  apiKey: "AIzaSyAFpnzjUdHCL_j7CWsDeJvGVXcr2wOryow",
  authDomain: "ai-la-gian-diep.firebaseapp.com",
  databaseURL: "https://ai-la-gian-diep-default-rtdb.asia-southeast1.firebasedatabase.app",
  projectId: "ai-la-gian-diep",
  storageBucket: "ai-la-gian-diep.firebasestorage.app",
  messagingSenderId: "452354702659",
  appId: "1:452354702659:web:b89fa58aa465ece7d8c3f6"
};
```

### File structure
- `index.html` — toàn bộ game (HTML + CSS + JS)
- `word_pairs.json` — 550+ cặp từ
- `game-spec.md` — file này

### Trạng thái hiện tại
- ✅ Tạo phòng, join phòng, realtime player list
- ✅ Rút thẻ 3D flip, turn-based
- ✅ Timer 3 phút + name card (sync qua Firebase)
- ✅ Vote với realtime phiếu bầu
- ✅ Kết quả vote + hiện role
- ✅ Mũ trắng đoán từ
- ✅ Win conditions: dân/gián điệp/mũ trắng
- ✅ 1 nick 1 thiết bị
- ✅ Rời phòng

### Cần fix
- Player sau khi rút thẻ xong đôi khi không thấy màn "Chờ chủ phòng"
- Kiểm tra flow 4 người có chạy mượt không
