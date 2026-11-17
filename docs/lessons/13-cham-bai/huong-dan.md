# Bài 13 — Chấm bài (điểm + nhận xét + chấm tiếp)

## 🎯 Mục tiêu

Gia sư mở bài nộp, chấm điểm + nhận xét (+ trả file đã chấm), luồng "Lưu & chấm tiếp" chạy liên tục không quay lại danh sách.

## 🧠 Kiến thức cần nắm

- Migration thêm cột, Transaction → [glossary/04-database.md](../../glossary/04-database.md)
- State client (Zustand) cho luồng chấm liên tục

## 📋 Các bước

### Bước 1: Migration thêm cột chấm vào submissions

```typescript
score?: number;            // numeric(4,1) — cho phép 8.5
feedback?: string;         // text
gradedAt?: Date;           // timestamptz
gradedBy?: number;         // tutor id
returnedFiles: { key: string; name: string }[];   // jsonb — file đã chấm trả về
```

> **Vì sao `gradedAt`/`gradedBy` riêng:** thống kê "5 bài chưa chấm" (dashboard) = `submitted_at IS NOT NULL AND graded_at IS NULL`; ai chấm sau này cũng tra được (tranh chấp, đối soát).
> **Vì sao `numeric(4,1)`:** điểm thang 10 có lẻ .5 — float dễ lỗi so sánh (0.1+0.2); numeric chính xác.

### Bước 2: API chấm

```
PATCH /api/submissions/[id]/grade
Body: { score, feedback, returnedFiles? }
```

Service: chỉ gia sư chủ lớp; chỉ chấm được submission có file/text (không chấm vào khoảng không); lưu `gradedAt = now`, `gradedBy`. **Gia hạn riêng:** `PATCH /api/assignments/[id]/deadline` nhận `{ studentId? }` — nếu có studentId, viết vào bản "gia hạn riêng" (jsonb `individualDeadlines` trên assignment).

> **Vì sao gia hạn riêng nằm trên assignment:** hạn chót gốc là luật chung; gia hạn cá nhân là ngoại lệ khai báo — bảng trạng thái (bài 12) đọc khi tính 'late' sẽ dùng `individualDeadlines[studentId] ?? dueAt` (cập nhật query CASE WHEN của bài 12).

### Bước 3: Trang chấm `/tutor/assignments/[id]/grade`

Layout 2 cột:
- **Trái:** danh sách HS theo trạng thái (dùng dữ liệu board bài 12), HS đang chấm highlight.
- **Phải:** 
  - Ảnh bài nộp: viewer zoom + xoay (mô-đun nhỏ, dùng CSS transform + wheel zoom)
  - Input điểm (thang 10, 1 chữ số lẻ), textarea nhận xét
  - Upload file trả lại (dùng `use-upload` folder `submissions`)
  - Nút **Lưu & chấm tiếp →** và Lưu

> **Vì sao zoom/xoay ảnh:** ảnh chụp vở ngược, mờ, nghiêng — gia sư chấm 20 bài nếu không xoay/zoom sẽ mất công tải ảnh ra xem ngoài, phá luồng.

### Bước 4: Luồng "chấm tiếp" — state máy

```typescript
type GradeQueue = {
  order: string[];          // thứ tự HS chưa chấm
  currentIndex: number;
};
```

- "Lưu & chấm tiếp": PATCH grade → cập nhật TanStack Query cache (đổi trạng thái bản vừa chấm → `graded`) → `currentIndex++` → form reset về dữ liệu HS kế (trống nếu chưa có).
- **Đếm tiến độ:** `Đã chấm 7/12` đọc trực tiếp từ cache — cập nhật ngay không cần refetch.

> **Vì sao dùng máy trạng thái queue:** chấm bài là luồng lặp dài — gia sư chấm 20 bài không muốn "lưu → về danh sách → bấm tiếp" 20 lần. Một nút, một chiều tiến, giảm 80% thao tác.
> **Vì sao cập nhật cache trực tiếp:** optimistic — UI phản hồi tức thì; refetch xảy ra lặng lẽ sau đó để đồng bộ nếu có gì lệch.

### Bước 5: Chống chấm đè — khóa bản đang chấm (mức đơn giản)

Trước khi lưu grade, service kiểm `gradedAt` chưa có **hoặc** `gradedBy = user` (cho sửa lại bài mình đã chấm). Nếu bài đã chấm bởi khác → cần nút "Sửa điểm" rõ ràng thay vì ghi đè lặng lẽ.

> **Vì sao:** chấm nhầm vào bài đã chấm mất điểm của học sinh. Sửa có chủ ý ≠ ghi đè vô tình — phân biệt bằng hành động UI.

## ✅ Tiêu chí hoàn thành

- [ ] Chấm 3 bài liên tiếp bằng nút "Lưu & chấm tiếp" không reload
- [ ] Điểm 8.5 lưu đúng (numeric), feedback hiển thị cho HS
- [ ] Trả file đã chấm thành công
- [ ] Gia hạn riêng 1 HS → trạng thái trễ của HS đó tính theo hạn mới
- [ ] Tiến độ "x/12" cập nhật tức thì
- [ ] Commit: `lesson-13: grading flow + individual deadlines`

## 📝 Tự kiểm tra

1. Vì sao `gradedAt`/`gradedBy` quan trọng cho thống kê?
2. numeric(4,1) thay vì float — sai số cụ thể tránh được?
3. Optimistic update ở luồng chấm — nếu PATCH thất bại, cache phải làm gì? (tra TanStack Query rollback)
4. Phân biệt "sửa điểm có chủ ý" và "ghi đè vô tình" trong UI?
