# 05 — Realtime, WebSocket & WebRTC

> Mục lục: [Realtime là gì](#1) • [WebSocket](#2) • [Socket.io](#3) • [WebRTC](#4) • [SFU](#5) • [TURN/STUN](#6) • [LiveKit](#7) • [Bảng trắng sync (tldraw)](#8) • [Simulcast](#9)

## 1. Realtime là gì? {#1}

HTTP thông thường: client hỏi → server trả → dứt. Muốn dữ liệu mới phải hỏi lại (refresh). **Realtime** = server **đẩy dữ liệu tới client ngay** khi có sự kiện, không cần hỏi.

**Trong dự án cần realtime ở:** chat trong phòng, trạng thái ai vào/ra phòng, giơ tay, thông báo "bài đã chấm", bảng trắng 2 người vẽ cùng lúc.

## 2. WebSocket {#2}

Kênh nối **hai chiều, giữ liên tục** giữa client và server: mở 1 lần rồi cả hai gửi cho nhau tự do:

```
HTTP:     client → server (hỏi)     client → server (hỏi)...   chậm, lặp
WebSocket: client ⇄ server  (kênh mở suốt phiên)                 nhanh, tức thời
```

## 3. Socket.io {#3}

Thư viện WebSocket + tự xử lý reconnect, room, fallback. Khái niệm quan trọng nhất là **room**:

```typescript
// server: chỉ gửi tới phòng "session-45" (buổi học 45)
io.to('session-45').emit('hand-raised', { studentId: 3, name: 'An' });

// client: tham gia phòng và nghe
socket.emit('join-session', 'session-45');
socket.on('hand-raised', (data) => { /* hiện badge ✋ */ });
```

**Trong dự án:** mỗi buổi học = 1 room Socket.io. Chat, tay giơ, trạng thái khóa phòng chạy qua đây. LiveKit lo video riêng (mục 7) — Socket.io không truyền video.

## 4. WebRTC {#4}

Công nghệ chuẩn cho phép trình duyệt truyền **video/audio trực tiếp** cho nhau (peer-to-peer), có mã hóa sẵn, không cần plugin. Ba phần: `getUserMedia` (xin cam/mic), `RTCPeerConnection` (kết nối truyền), signaling (trao đổi thông tin kết nối — phải tự xây).

**Vì sao dự án không tự viết WebRTC thuần?** Kết nối P2P chỉ hợp lớp nhỏ; lớp đông + ghi hình + qua NAT đều cần hạ tầng server → dùng LiveKit, đã đóng gói hết phần khó.

## 5. SFU {#5}

**SFU (Selective Forwarding Unit)** = server trung gian nhận video/audio từ mỗi người rồi **chọn và chuyển tiếp** cho những người cần xem.

```
P2P mesh: 5 người = mỗi người gửi 4 luồng → nổ băng thông
SFU:      5 người = mỗi người gửi 1 luồng lên server, server phân phát → ổn
```

**Trong dự án:** LiveKit chính là SFU. Học sinh tắt cam mặc định → không tạo luồng → lớp 50 người vẫn nhẹ. Ai được mời bật cam thì luồng mới xuất hiện.

## 6. TURN / STUN {#6}

Thực tế nhiều mạng (trường học, công ty) chặn kết nối trực tiếp:
- **STUN:** server giúp 2 bên "tìm địa chỉ thật" của nhau → thử nối trực tiếp. Miễn phí.
- **TURN:** khi nối trực tiếp thất bại, dữ liệu **đi vòng qua server TURN**. Tốn băng thông nhưng cứu được ~10-15% người dùng không nối được nếu thiếu.

**Trong dự án:** LiveKit server đã có TURN tích hợp — bài 16 chỉ cần bật cấu hình.

## 7. LiveKit {#7}

Video SDK open source dùng trong dự án:
- **LiveKit server:** SFU + TURN, chạy bằng Docker trên VPS.
- **LiveKit client SDK (React):** hook/component có sẵn — `useRoom`, `VideoTile`, điều khiển mic/cam/share màn hình.
- **Token:** để vào phòng phải có JWT do **server app của mình cấp** (qua LiveKit Access Token API) — chứa room, quyền (publish hay chỉ subscribe), thời hạn. Học sinh không thể tự chế token.
- **Egress:** thành phần ghi hình phòng → MP4, tự upload storage.

## 8. Bảng trắng sync (tldraw) {#8}

**tldraw** = thư viện bảng trắng open source. Phần khó không phải vẽ (tldraw lo), mà là **sync**: người A vẽ đường, người B thấy ngay. Giải pháp: mọi thay đổi (thêm/sửa/xóa shape) thành "deltas" truyền qua WebSocket, server phát cho mọi người + lưu để người vào muộn có bản hiện tại.

**Trong dự án:** dùng `@tldraw/sync` (sync có sẵn của tldraw) qua WebSocket server riêng. Khi kết thúc buổi → export canvas thành PNG lưu R2 làm "artifact".

## 9. Simulcast {#9}

Người bật cam gửi **nhiều chất lượng cùng lúc** (180p/360p/720p); SFU phát chất lượng phù hợp từng người xem — mạng yếu nhận 180p, mạng tốt nhận 720p.

**Trong dự án:** bật simulcast cho gia sư (nguồn quan trọng nhất) để học sinh máy/network yếu vẫn theo được bài.
