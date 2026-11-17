# Bài 17 — Video + audio trong phòng học

## 🎯 Mục tiêu

Trang phòng học theo buổi: vào được từ lịch, grid video 1–16 người, điều khiển mic/cam, reconnect tự động, tile hiển thị đúng trạng thái mỗi người.

## 🧠 Kiến thức cần nắm

- LiveKit client SDK, simulcast → [glossary/05-realtime-webrtc.md](../../glossary/05-realtime-webrtc.md)
- Zustand (state phòng) → [glossary/02-frontend.md](../../glossary/02-frontend.md#10-zustand)

## 📋 Các bước

### Bước 1: Cài SDK + cấu trúc trang phòng

```bash
npm i @livekit/components-react livekit-client
```

```
app/tutor/room/[sessionId]/page.tsx   (và student tương tự)
  → server component kiểm tra quyền + session tồn tại
  → RoomClient (client component):
      nhận token qua fetch POST /api/sessions/[id]/room-token
      <LiveKitRoom token={token} serverUrl={url} ...>
```

> **Vì sao tách RoomClient:** trang server lo quyền (403 nếu không thuộc lớp), client component lo kết nối — ranh giới trách nhiệm rõ.

### Bước 2: LiveKitRoom + Layout

```tsx
<LiveKitRoom
  serverUrl={url}
  token={token}
  connect={true}
  audio={true}
  video={false}            // học sinh KHÔNG bật cam mặc định
  onDisconnected={...}
>
  <VideoConference />
</LiveKitRoom>
```

> **Vì sao `video: false` mặc định:** thiết kế lõi dự án — học sinh vào là để **nghe và học**; bật cam là đặc quyền gia sư mời (bài 18). Nhân tiện tiết kiệm băng thông khủng khiếp khi lớp đông.
> **Vì sao `audio: true`:** nghe giảng là điều kiện tối thiểu; mic học sinh vẫn phải bật tay (push-to-talk) — `audio: true` chỉ là quyền thiết bị, LiveKit track mic sẽ mute mặc định qua control.

### Bước 3: Zustand store cho trạng thái phòng

```typescript
type RoomState = {
  micOn: boolean; camOn: boolean; sharing: boolean;
  locked: boolean; raisedHands: { userId: number; name: string; at: number }[];
  participants: { userId: number; name: string; role: string; micOn: boolean; camOn: boolean }[];
};
```

Mọi nút điều khiển ghi vào store → UI 1 nguồn sự thật, không "nút bật nhưng icon chưa đổi".

### Bước 4: Grid video + tile

Dựng `VideoGrid`: tile = video (nếu cam on) hoặc avatar initials (cam off). Dùng component của LiveKit (`useParticipants`, `VideoTrack`) hoặc tự dựng — tự dựng để kiểm soát style Vercel hoàn toàn.

> **Vì sao tự dựng tile:** component mặc định của LiveKit có style riêng; dự án cần đồng bộ phong cách (góc nhỏ, viền mảnh) + badge mic/cam/tay giơ theo chuẩn riêng.

### Bước 5: Reconnect + đồng bộ trạng thái

- `RoomEvent.Reconnecting/Reconnected` → banner "Đang kết nối lại..." + sau khi về, refresh participants.
- 4G chuyển wifi, mất mạng 10s → phải tự hồi mà không cần reload.

> **Vì sao test reconnect sớm:** học sinh dùng 4G — mất mạng là chuyện hàng ngày. UI chết sau mất mạng = học sinh bỏ học giữa buổi và không quay lại.

### Bước 6: Gắn nút "Vào phòng học" từ lịch

Drawer lịch (bài 10): nút gọi `POST room-token` → redirect `/room/[sessionId]`. Chỉ hiện khi trạng thái buổi = `upcoming`/`live`.

## ✅ Tiêu chí hoàn thành

- [ ] Vào phòng từ lịch được, token hết hạn → xin lại tự động
- [ ] 2+ thiết bị thấy video đúng tile tên người
- [ ] Mic/cam toggle mượt, tile phản ánh đúng trạng thái
- [ ] Mất mạng 10s → tự reconnect, không reload
- [ ] Commit: `lesson-17: room page + video grid + reconnect`

## 📝 Tự kiểm tra

1. Vì sao `video: false` mặc định cho học sinh? Đổi ý thiết kế này tốn gì?
2. Zustand vs useState trong phòng học — điểm khác quyết định?
3. Reconnect là sự kiện nào của RoomEvent? Phải làm gì sau khi reconnect?
4. Token hết hạn giữa buổi — ai xin lại, bằng API nào?
