# Bài 19 — Bảng trắng 2 chiều (tldraw)

## 🎯 Mục tiêu

Bảng trắng chiếm ~60% màn hình phòng học; gia sư + học sinh vẽ cùng lúc thấy nhau; upload ảnh đề/PDF lên bảng; kết thúc buổi lưu ảnh bảng.

## 🧠 Kiến thức cần nắm

- tldraw sync, WebSocket → [glossary/05-realtime-webrtc.md](../../glossary/05-realtime-webrtc.md#8-bảng-trắng-sync-tldraw)
- Upload R2 (bài 07)

## 📋 Các bước

### Bước 1: Cài tldraw + sync server

```bash
npm i tldraw @tldraw/sync
```

Sync server: chạy process Node riêng (cùng VPS hoặc chung với Socket.io worker) dùng `@tldraw/sync-core` + WebSocket. Room sync = `session-{id}-board`.

> **Vì sao process riêng:** tldraw sync giữ state trong RAM + chuyển tiếp delta liên tục — không nhét vào Next.js server (request/response), tách để scale độc lập như LiveKit.

### Bước 2: Layout bảng trong phòng

```
┌──────────────────────────────┬────────────┐
│  video strip (trên, nhỏ)      │            │
├──────────────────────────────┤   chat     │
│  BẢNG TRẮNG (flex-1)          │  (bài 20)  │
│  toolbar trái của tldraw      │            │
└──────────────────────────────┴────────────┘
```

Bảng dùng component `<TldrawEditor persistenceKey=... store={syncStore}>`.

> **Vì sao bảng lớn:** dạy kèm là "viết lên bảng" — Meet tối ưu cho họp (video to), phòng học của bạn tối ưu cho **giảng bài trên bảng**. Video chỉ cần thấy mặt nhau.

### Bước 3: Quyền trên bảng

Giai đoạn này: mọi thành viên vẽ được (2 chiều). Cấu hình `getUserPermissions` cho phép resize/select mọi shape.

> **Vì sao cho HS vẽ ngay:** dạy kèm cần HS làm bài trực tiếp (viết phương trình). Bài tập trên bảng kiểu "giao đề từng HS" là tính năng backlog — không cần tách trang ngay.

### Bước 4: Upload ảnh đề / PDF

Nút "Chèn ảnh đề bài" trong toolbar tùy chỉnh tldraw:
1. Chọn file → upload R2 (folder `boards`) như bài 07
2. Chèn shape ảnh vào canvas (`createAssets` + `createShape` kiểu image) — HS và gia sư vẽ đè được

PDF: từng trang render thành ảnh (client `pdf.js` hoặc server poppler) → chèn như ảnh.

> **Vì sao biến PDF thành ảnh:** tldraw xử lý image asset dễ nhất; giữ nguyên PDF cần viewer riêng ngoài canvas — phá trải nghiệm "mọi thứ nằm trên một bảng".

### Bước 5: Lưu ảnh bảng khi kết thúc buổi

Nút "Kết thúc buổi" (bài 21) gọi API → server gọi `store.exportAsBlob` (hoặc client thực hiện trước khi disconnect) → upload PNG folder `boards` → lưu dòng `whiteboard_snapshots` gắn sessionId.

> **Vì sao snapshot ảnh thay vì giữ dữ liệu shapes:** ảnh là artifact ổn định xem mọi nơi không cần tldraw; dữ liệu shapes giữ trong sync server để mở lại bảng sau (tùy chọn backlog).

### Bước 6: Kiểm thử multiplayer

2 trình duyệt profile khác nhau: vẽ cùng lúc, kéo shape, undo — không xung đột (tldraw sync lo CRDT-ish merge). Kiểm mất mạng: vẽ offline → nối lại → shape còn.

## ✅ Tiêu chí hoàn thành

- [ ] 2 người vẽ cùng lúc thấy nhau < 300ms
- [ ] Upload ảnh đề + vẽ đè được
- [ ] PDF nhiều trang chèn từng trang thành ảnh
- [ ] Kết thúc buổi → PNG lưu + xem được từ buổi trong lịch
- [ ] Commit: `lesson-19: tldraw whiteboard + snapshots`

## 📝 Tự kiểm tra

1. Delta sync hoạt động thế nào khi 2 người vẽ cùng một vị trí? (tra CRDT của tldraw)
2. Vì sao sync server tách khỏi Next.js?
3. PDF thành ảnh — mất gì so với PDF gốc? Đánh đổi có đáng không cho MVP phòng học?
4. Artifact ảnh bảng dùng ở đâu trong các bài sau? (báo cáo sau buổi — bài 25)
