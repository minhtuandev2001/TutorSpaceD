# Bài 18 — Quyền điều khiển của gia sư trong phòng

## 🎯 Mục tiêu

Gia sư kiểm soát lớp: tắt mic học sinh, mời học sinh bật cam/mic, khóa phòng; học sinh thấy trạng thái khóa ngay khi cố vào.

## 🧠 Kiến thức cần nắm

- LiveKit permissions, RoomParticipant metadata → [glossary/05-realtime-webrtc.md](../../glossary/05-realtime-webrtc.md#7-livekit)
- Socket.io room (bài 14)

## 📋 Các bước

### Bước 1: Thiết kế mô hình quyền động

LiveKit hỗ trợ đổi quyền **tại chỗ** qua `UpdateParticipant` (server API) — không cần rời phòng để cấp lại token:

```typescript
import { RoomServiceClient } from 'livekit-server-sdk';
const roomService = new RoomServiceClient(LIVEKIT_URL, API_KEY, API_SECRET);

// mời học sinh nói: bật quyền publish
await roomService.updateParticipant('session-45', `user-${studentId}`, undefined, {
  canPublish: true, canPublishSources: ['mic', 'camera'], canSubscribe: true,
});

// tắt mic + khóa quyền
await roomService.mutePublishedTrack('session-45', identity, trackSid, true);
await roomService.updateParticipant('session-45', identity, undefined, { canPublish: false });
```

> **Vì sao dùng UpdateParticipant thay vì cấp token lại:** rời phòng để xin token mới = gián đoạn buổi học. Đổi quyền động là chuẩn cho "gọi lên sân khấu".

### Bước 2: Bảng trạng thái phòng (DB + memory)

Bảng `rooms` (1-1 với session): `isLocked`, `allowStudentPublish` (global). Giá trị live đồng bộ Socket.io cho tốc độ + DB cho sự thật (người vào sau đọc DB qua API token).

> **Vì sao 2 lớp (DB + socket):** người vào sau không nghe được event cũ — phải đọc DB; người đang trong phòng nghe socket realtime. Tương tác: socket event kèm event "room-state-changed", client update store.

### Bước 3: API điều khiển (chỉ tutor)

```
POST /api/rooms/[sessionId]/lock         khóa/mở phòng
POST /api/rooms/[sessionId]/mute-all     tắt mic tất cả HS
POST /api/rooms/[sessionId]/invite-speaker   { studentId, mic?, cam? }
DELETE /api/rooms/[sessionId]/kick       { studentId }  — đá khỏi phòng
```

Service: guard tutor chủ lớp → gọi RoomServiceClient → cập nhật DB → emit socket `room-state` cho room.

> **Vì sao "tắt mic tất cả" tồn tại:** 30 HS mở mic ồn środ = buổi học chết. Nút 1 phát của gia sư là UX giáo dục chuẩn (Zoom/Meet đều có).

### Bước 4: UI điều khiển (toolbar gia sư)

- Nút **Tắt mic tất cả** + **Khóa phòng** (toggle, icon lock đổi)
- Danh sách participants: từng HS nút "Mời nói" (mic/cam) + "Đá"
- Khi HS được mời: nhận socket event → tự động `setCameraEnabled(true)`/`setMicrophoneEnabled(true)` + toast "Thầy đã mời bạn phát biểu"

> **Vì sao HS được mời tự bật thay vì phải bấm:** ít thao tác nhất khi được gọi tên; đồng thời nếu HS từ chối (tắt lại), gia sư thấy ngay tile cam off.

### Bước 5: Chặn vào phòng khi khóa

`createRoomToken` (bài 16) kiểm `isLocked` + role student → trả 403 "Phòng đã khóa". Client bắt lỗi → màn hình "Phòng đã khóa, chờ gia sư mở".

> **Vì sao chặn ở token service:** điểm chặn duy nhất — mọi đường vào phòng đều qua đây (trang phòng, deep-link email nhắc lịch).

## ✅ Tiêu chí hoàn thành

- [ ] Tắt mic tất cả HS → mọi HS mất tiếng ngay
- [ ] Mời HS nói → HS bật cam/mic được, từ chối thì tile off
- [ ] Khóa phòng → HS mới nhận màn hình "đã khóa" không token
- [ ] Đá HS → HS tự động bị ngắt kết nối
- [ ] Commit: `lesson-18: room controls + dynamic permissions`

## 📝 Tự kiểm tra

1. UpdateParticipant khác gì cấp token lại? Trải nghiệm người dùng khác nhau thế nào?
2. Vì sao cần DB + socket đồng thời cho trạng thái phòng?
3. Trào lưu tấn công "class-bombing" — cơ chế nào trong bài chống lại?
4. Nếu HS tinh vi gọi API token trước khi khóa rồi vào sau — lỗ hổng nào còn hở? (gợi ý: LiveKit webhook/RoomClosed)
