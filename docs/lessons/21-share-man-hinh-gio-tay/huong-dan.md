# Bài 21 — Share màn hình & giơ tay

## 🎯 Mục tiêu

Gia sư share màn hình/cửa sổ; học sinh giơ tay có hàng đợi, gia sư cho phép nói hoặc bỏ qua từng người.

## 🧠 Kiến thức cần nắm

- LiveKit track screen share, DataChannel → [glossary/05-realtime-webrtc.md](../../glossary/05-realtime-webrtc.md)
- Socket.io (bài 14, 20)

## 📋 Các bước

### Bước 1: Share màn hình — LiveKit built-in

```tsx
const { sharing, toggle } = useScreenShare();  // hoặc room.localParticipant.setScreenShareEnabled(true)
```

LiveKit mặc định chỉ cho publish share màn hình nếu `canPublish` — từ bài 16-18 đã cấu hình: tutor được share luôn; HS chỉ share khi được mời (mức `canPublishSources`).

Layout khi share: track screen share của ai đó → replace video strip bằng vùng share lớn (như bảng trắng) + giữ thumbnail người share.

> **Vì sao layout riêng cho share:** share màn hình nội dung chi tiết (code, slide) — phải to; grid video thu nhỏ lại. Đây là quy ước chuẩn mọi app họp.

### Bước 2: Audio share màn hình (chú ý Safari)

`setScreenShareEnabled(true, { audio: true })` — trình duyệt Chrome hỗ trợ share kèm âm thanh hệ thống; **Safari/iOS không hỗ trợ** → ẩn tùy chọn audio trên Safari (detect bằng feature check, không detect theo user agent).

> **Vì sao feature check:** user agent đổi liên tục; `navigator.mediaDevices.getDisplayMedia` có/không audio là thuộc tính có thể thử. Code theo khả năng thật, không theo tên trình duyệt.

### Bước 3: Giơ tay — sự kiện + hàng đợi

Socket.io events:
- `hand:raise` / `hand:lower` (HS) → server giữ mảng hands theo thứ tự thời gian, emit `hands:update` cho cả room
- Gia sư: nút cho phép (→ mời nói, bài 18) hoặc bỏ qua (xóa khỏi hàng đợi)

State giơ tay nằm trong Zustand store phòng (bài 17) — **không lưu DB** (trạng thái phiên, buổi xong là hết).

> **Vì sao hàng đợi theo thời gian:** 10 HS giơ tay cùng lúc — gia sư trả lời theo thứ tự công bằng; mảng push + timestamp tự nhiên giữ thứ tự.
> **Vì sao không lưu DB:** tay giơ là tín hiệu ngắn (< vài phút); dư DB làm bảng trạng thái phòng phải dọn dẹp. Điểm khác chat (bài 20) — chat là lịch sử.

### Bước 4: Badge UI giơ tay

- Tile HS giơ tay: viền vàng + badge "✋"
- Toolbar gia sư: nút "✋ x3" mở dropdown hàng đợi, từng người 2 nút (Mời nói / Bỏ qua)
- HS: nút giơ tay toggle, trạng thái "bạn đang giơ tay" rõ ràng

### Bước 5: Kết thúc buổi — tổng hợp artifact

Nút "Kết thúc buổi" (tutor): gọi API:
1. Đổi trạng thái session → `done`
2. Lưu ảnh bảng trắng (bài 19)
3. Đóng socket room, ngắt LiveKit room (RoomService `deleteRoom`)
4. Thêm hàng đợi job: ghi hình nếu bật (bài 22)

> **Vì sao một nút làm nhiều việc:** gia sư chỉ nhớ 1 hành động "kết thúc"; mọi dọn dẹp phải tự chạy — backend là người gom, UI chỉ là nút.

## ✅ Tiêu chí hoàn thành

- [ ] Share màn hình trên Chrome + Windows/Mac; audio share tùy trình duyệt
- [ ] Layout chuyển đúng khi có share
- [ ] Giơ tay 3 HS → thứ tự đúng, mời/bỏ qua từng người
- [ ] Kết thúc buổi → mọi participant ngắt, trạng thái session = done
- [ ] Commit: `lesson-21: screen share + hands + end session`

## 📝 Tự kiểm tra

1. Vì sao giơ tay không lưu DB mà chat lại lưu?
2. Feature check khác user-agent sniffing? Vì sao quan trọng?
3. Kết thúc buổi — thứ tự các bước backend có quan trọng không? Nếu lưu snapshot sau khi deleteRoom thì sao?
4. HS không được mời nhưng cố share màn hình — chặn ở đâu, tầng nào?
