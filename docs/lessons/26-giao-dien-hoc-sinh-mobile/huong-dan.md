# Bài 26 — Giao diện học sinh mobile-first

## 🎯 Mục tiêu

Toàn bộ trải nghiệm học sinh dùng được tốt trên điện thoại: trang chủ "hôm nay có gì", luồng nộp ảnh bài ≤ 3 chạm, xem lịch, vào phòng học trên mobile.

## 🧠 Kiến thức cần nắm

- Tailwind responsive, LiveKit trên mobile browser → [glossary/02-frontend.md](../../glossary/02-frontend.md), [05-realtime-webrtc.md](../../glossary/05-realtime-webrtc.md)

## 📋 Các bước

### Bước 1: Nguyên tắc mobile-first cho HS

Học sinh ≠ gia sư: màn hình nhỏ, thao tác 1 tay, mạng 4G. Quy tắc:
- Font lớn hơn (base 15-16px), nút ≥ 44px, khoảng cách thoáng
- Mỗi trang trả lời 1 câu: "Bây giờ tôi phải làm gì?"
- Điều hướng bottom tab (không sidebar): **Trang chủ • Bài tập • Lịch • Tài khoản**

> **Vì sao bottom tab:** ngón tay comfortable zone ở đáy màn hình; sidebar web trên mobile thu về hamburger = ẩn tính năng, HS không tìm ra.

### Bước 2: Trang chủ HS — "hôm nay có gì"

Trang chủ hiển thị theo độ ưu tiên thực tế:
1. **Buổi học hôm nay** (nếu có): thẻ lớn + đếm ngược + nút "Vào phòng học" (hoạt động 5 phút trước giờ học)
2. **Bài sắp đến hạn** (đỏ nếu <24h): nút "Nộp bài" trực tiếp
3. **Có gì mới** (thông báo chưa đọc gộp: điểm mới, báo cáo)
4. Lịch tuần thu gọn

> **Vì sao thứ tự này:** đúng nhịp sống HS — 19h phải vào lớp; trước đó nộp bài; mở app bất kỳ lúc nào cũng thấy "việc tiếp theo" đầu trang.

### Bước 3: Luồng nộp ảnh ≤ 3 chạm

```
Chạm 1: "Nộp bài" trên thẻ bài tập (từ trang chủ!)
Chạm 2: chọn "Chụp ảnh" → camera mở (input capture="environment")
Chạm 3: chụp + "Nộp" (tự upload, progress + toast thành công)
```

Chi tiết kỹ thuật:
- `<input type="file" accept="image/*" capture="environment">` — mở camera trực tiếp trên mobile
- Nén ảnh trước upload: canvas resize về max 1600px + JPEG quality 0.8 (dùng browser-image-compression) — ảnh vở 12MB → ~400KB, upload nhanh trên 4G
- Preview + xoay trước khi nộp

> **Vì sao nén client:** ảnh gốc 12MP nặng 5-15MB — upload 3G/4G mất 30-60s dễ timeout; nén còn dưới 500KB = vài giây, chất lượng vẫn đủ đọc chữ vở.
> **Vì sao giữ "chụp" là chạm 2:** không qua chọn "từ thư viện" mặc định — chụp vở là hành động chiếm 80% lượng nộp.

### Bước 4: Phòng học trên mobile

- Grid video: 2 cột portrait / 4 cột landscape; bảng trắng scroll được, pinch-zoom
- Lưu ý: tab/mobile browser bị **suspend** khi ẩn → LiveKit auto-reconnect (đã có bài 17); cảnh báo "giữ màn hình mở"
- iOS Safari: fullscreen API hạn chế — bảng trắng dùng chiều cao viewport động (`100dvh` thay `100vh` để không bị che thanh địa chỉ)

> **Vì sao `100dvh`:** iOS Safari có thanh trình duyệt động — `100vh` cao hơn vùng nhìn thấy, nút cuối trang bị che; `dvh` là đơn vị chuẩn cho vấn đề này.

### Bước 5: Kiểm thử mobile thật

Test ma trận: Android Chrome + iOS Safari; wifi + 4G; light + dark mode. Đặc biệt luồng: vào lớp từ email → đăng nhập → nộp ảnh → chấm xong xem điểm.

> **Vì sao thiết bị thật:** trình mô phỏng DevTools không tái hiện: bàn phím che input, camera permission, cảm ứng cuộn, tốc độ mạng thật.

## ✅ Tiêu chí hoàn thành

- [ ] Trang chủ HS trả lời được "bây giờ làm gì" trong 2 giây đầu nhìn
- [ ] Nộp ảnh bài: chụp → nộp, tổng ≤ 3 chạm, ảnh < 600KB
- [ ] Bottom tab điều hướng đủ 4 mục
- [ ] Phòng học mobile dùng được (video + bảng + chat)
- [ ] Test 2 thiết bị thật (Android + iOS nếu có)
- [ ] Commit: `lesson-26: student mobile experience`

## 📝 Tự kiểm tra

1. Mobile-first khác responsive desktop-first thế nào? Vẽ trước ở size nào?
2. Nén ảnh client — mất gì? Giới hạn nào là hợp lý cho ảnh vở?
3. `100dvh` giải quyết vấn đề iOS cụ thể nào?
4. Thứ tự nội dung trang chủ HS — quyết định sản phẩm nào đằng sau?
