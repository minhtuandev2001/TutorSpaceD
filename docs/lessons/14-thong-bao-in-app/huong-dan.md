# Bài 14 — Thông báo in-app (Socket.io)

## 🎯 Mục tiêu

Hệ thống thông báo cho học sinh: bài mới, bài đã chấm, sắp đến hạn — badge realtime, danh sách thông báo, đánh dấu đã đọc.

## 🧠 Kiến thức cần nắm

- WebSocket, Socket.io, room → [glossary/05-realtime-webrtc.md](../../glossary/05-realtime-webrtc.md#3-socket-io)
- BullMQ nền từ bài 08

## 📋 Các bước

### Bước 1: Cài Socket.io + server riêng

```bash
npm i socket.io socket.io-client
```

Server Socket.io chạy trong **node server tùy chỉnh** (`server.mjs` — bọc Next.js):

```javascript
import { createServer } from 'http';
import next from 'next';
import { Server } from 'socket.io';

const app = next({ dev: process.env.NODE_ENV !== 'production' });
await app.prepare();
const httpServer = createServer((req, res) => app.getRequestHandler()(req, res));

const io = new Server(httpServer, { cors: { origin: process.env.APP_URL } });

io.use(async (socket, next) => {
  // xác thực: đọc token session từ cookie socket handshake → gắn socket.data.userId
  // không có token → callback(new Error('unauthorized'))
});

io.on('connection', (socket) => {
  socket.join(`user-${socket.data.userId}`);   // mỗi user 1 room riêng
});

httpServer.listen(3000);
```

Đổi script dev/prod thành `node server.mjs`. Đăng ký `io` vào biến toàn cục để API route truy cập (`globalThis.__io = io`).

> **Vì sao server tùy chỉnh:** Next.js mặc định không expose HTTP server cho Socket.io. Bọc lại là cách chuẩn.
> **Vì sao xác thực ngay khi handshake:** socket không có auth = ai kết nối được đều nghe được thông báo của room nào đó — lỗ hổng thông tin cá nhân.

### Bước 2: Entity notifications

```typescript
@Entity({ name: 'notifications' })
export class NotificationEntity {
  @PrimaryGeneratedColumn() id: number;
  @Column({ name: 'user_id' }) userId: number;            // người nhận
  @Column({ type: 'enum', enum: ['assignment_new', 'assignment_graded', 'deadline_soon', 'session_reminder'] })
  type: string;
  @Column({ type: 'jsonb' }) payload: { title: string; body: string; link?: string };
  @Column({ name: 'read_at', type: 'timestamptz', nullable: true }) readAt?: Date;
  @CreateDateColumn({ name: 'created_at' }) createdAt: Date;
}
```

> **Vì sao lưu DB dù đã push realtime:** người dùng ngoại tuyến phải thấy thông báo khi quay lại. Realtime = nhận tức thì; DB = nhận muộn nhưng chắc chắn. 2 lớp phủ nhau.

### Bước 3: Hàm thông báo dùng chung

`src/modules/notifications/notify.ts`:

```typescript
export async function notifyUser(userId: number, type: NotificationType, payload: Payload) {
  const n = await notifyRepo.save({ userId, type, payload });
  globalThis.__io?.to(`user-${userId}`).emit('notification', n);   // nếu online
}
```

Gọi tại đúng sự kiện nghiệp vụ (trong service):
- `assignmentsService.create` → `assignment_new` cho từng assignee
- `grade()` (bài 13) → `assignment_graded`
- Cron (bước 4) → `deadline_soon`

> **Vì sao gọi trong service chứ không phải API route:** mọi đường dẫn tạo bài (API + job tương lai) đều phải thông báo — đặt ở service = một điểm duy nhất không bỏ sót.

### Bước 4: Cron "sắp đến hạn"

Job BullMQ chạy mỗi giờ: tìm assignment có `dueAt` trong 24h tới, assignee chưa nộp, chưa nhận thông báo deadline (check bảng notification đã gửi cùng assignment) → gửi `deadline_soon`.

> **Vì sao check "đã gửi":** job chạy mỗi giờ — không check sẽ nhắc 24 lần cùng bài. Lưu dấu đã gửi (truy vấn notification tồn tại cùng type+assignment) là idempotence thực tế.

### Bước 5: UI thông báo

- Chuông nav: badge đếm chưa đọc (realtime qua socket + TanStack Query cho lần đầu)
- Dropdown danh sách: loại, tiêu đề, thời gian tương đối ("5 phút trước"), link payload; click → đánh dấu đã đọc (`PATCH /api/notifications/read`) + điều hướng
- Trang `/notifications` đầy đủ + "Đánh dấu tất cả đã đọc"

> **Vì sao 2 nguồn (socket + query):** mở trang mới phải có badge đúng ngay (query); có sự kiện mới phải cập nhật không reload (socket). Xử lý trùng: socket event cũng `invalidateQueries(['notifications'])` — cache tự hợp nhất.

## ✅ Tiêu chí hoàn thành

- [ ] Giao bài → HS online nhận badge tức thì; offline mở lại thấy
- [ ] Chấm bài → thông báo kèm điểm/link
- [ ] Cron sắp đến hạn nhắc đúng 1 lần/bài
- [ ] Đánh dấu đã đọc cập nhật badge
- [ ] Commit: `lesson-14: in-app notifications (socket + db)`

## 📝 Tự kiểm tra

1. Vì sao lưu notification vào DB khi đã emit realtime?
2. Room `user-${id}` vs room lớp — khi nào dùng loại nào?
3. Idempotence của cron deadline — thiếu check này hậu quả?
4. Vì sao xác thực socket ở handshake thay vì ở từng event?
