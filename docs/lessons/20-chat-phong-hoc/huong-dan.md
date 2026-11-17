# Bài 20 — Chat thời gian thực trong phòng học

## 🎯 Mục tiêu

Chat trong phòng qua Socket.io; tin nhắn lưu PostgreSQL; sau buổi xem lại lịch sử chat từ trang buổi học.

## 🧠 Kiến thức cần nắm

- Socket.io room, xác thực handshake (bài 14) → [glossary/05-realtime-webrtc.md](../../glossary/05-realtime-webrtc.md#3-socket-io)
- TanStack Query + optimistic update

## 📋 Các bước

### Bước 1: Entity messages

```typescript
@Entity({ name: 'chat_messages' })
export class ChatMessageEntity {
  @PrimaryGeneratedColumn() id: number;
  @Column({ name: 'session_id' }) sessionId: number;
  @Column({ name: 'user_id' }) userId: number;
  @ManyToOne(() => UserEntity) @JoinColumn({ name: 'user_id' }) user: UserEntity;

  @Column({ type: 'text' }) content: string;
  @Column({ type: 'jsonb', default: [] }) attachments: { key: string; name: string }[];

  @CreateDateColumn({ name: 'created_at' }) createdAt: Date;
}
```

> **Vì sao lưu DB:** chat là lịch sử buổi học (HS xem lại câu hỏi đã hỏi; gia sư thấy hiểu bài qua câu hỏi). Socket.io chỉ là ống truyền, DB là bộ nhớ.

### Bước 2: Socket.io events cho chat

Server (trong `server.mjs` từ bài 14):

```javascript
socket.on('chat:send', async ({ sessionId, content }, ack) => {
  // 1. kiểm tra user thuộc buổi này (tutor chủ lớp / HS trong lớp)
  // 2. kiểm tra độ dài (≤ 2000 ký tự) + rate limit đơn giản (10 msg/10s)
  // 3. lưu DB
  // 4. io.to(`session-${sessionId}`).emit('chat:new', message)
  ack({ ok: true, id: message.id });
});
```

> **Vì sao `ack` (callback):** client biết chắc tin nhắn đã được server chấp nhận (hiện bình thường) hay bị chặn (rate limit → toast). Không ack = client phải đoán.
> **Vì sao rate limit:** một HS spam 1000 tin/giây → cả phòng lag. Chặn ở server là luật, UI chỉ là cảnh báo.

### Bước 3: Client — UI chat cột phải

- Header "Chat", body scroll (auto-scroll xuống khi có tin mới **nếu đang ở đáy** — không giật khi đang đọc tin cũ), input + nút gửi.
- Optimistic: gửi → hiện tin mờ "đang gửi" → ack → đậm + id; lỗi → đỏ + nút gửi lại.

> **Vì sao auto-scroll có điều kiện:** đang đọc tin cũ phía trên mà bị kéo xuống đáy = UX tệ kinh điển. Chỉ scroll khi user đã ở đáy.

### Bước 4: Lịch sử chat khi vào giữa

Khi vào phòng: fetch `GET /api/sessions/[id]/messages` (200 tin cuối) → hiển thị → socket chỉ thêm tin mới. Dùng TanStack Query key `['messages', sessionId]`.

> **Vì sao không gửi cả lịch sử qua socket:** socket là kênh sự kiện, không phải kênh dữ liệu lớn; refetch từ API có cache + phân trang dễ hơn.

### Bước 5: Xem lại chat sau buổi

Trang buổi học (từ lịch): phần "Lịch sử chat" — danh sách tin theo thời gian. Cột được gắn vào artifact buổi học (cùng với ảnh bảng trắng bài 19).

> **Vì sao nằm cùng artifact:** câu chuyện buổi học = bảng + chat (+ video sau này bài 22). Xem riêng lẻ mất ngữ cảnh.

### Bước 6: Dọn dẹp session

Ngắt kết nối: client `socket.disconnect()` khi rời phòng; server cập nhật `participants` của room store (bài 17) → mọi người thấy tile biến mất.

## ✅ Tiêu chí hoàn thành

- [ ] Chat realtime 2 chiều, tin mới < 300ms
- [ ] Vào giữa buổi thấy 200 tin cuối
- [ ] Spam 20 tin/10s → bị chặn, thông báo rõ
- [ ] Sau buổi xem lại toàn bộ chat
- [ ] Commit: `lesson-20: room chat (socket + db + history)`

## 📝 Tự kiểm tra

1. ack trong Socket.io giải quyết vấn đề UX nào?
2. Auto-scroll có điều kiện — logic "ở đáy" tính thế nào?
3. Vào giữa buổi: lịch sử từ API, tin mới từ socket — làm sao không lặp tin? (gợi ý: id tăng dần)
4. Chat lưu DB → nhớ đúng phân hệ nào sẽ dùng lại nó? (báo cáo sau buổi — bài 25)
