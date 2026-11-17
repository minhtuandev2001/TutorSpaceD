# Bài 27 — Trang quản trị (admin)

## 🎯 Mục tiêu

Vai trò thứ 3 (admin) xem được tổng quan hệ thống: số user/lớp/buổi, quản lý user, thống kê hoạt động — bảo vệ bằng role guard.

## 🧠 Kiến thức cần nắm

- Role guard (bài 04) — thêm role 'admin'
- Aggregate query (bài 05, 24)

## 📋 Các bước

### Bước 1: Thêm role admin

- Migration: `enum` users.role thêm `'admin'`
- Seed 1 tài khoản admin (email cố định cấu hình .env, chỉ seed chạy production 1 lần)
- `requireRole` mở rộng chấp nhận `roles: UserRole[]`

> **Vì sao seed admin thay vì đăng ký:** không ai tự tạo tài khoản admin qua form công khai — tài khoản admin sinh từ dữ liệu hệ thống, email phải khớp biến môi trường.

### Bước 2: API thống kê

```
GET /api/admin/stats     { users, tutors, students, classes, sessionsThisWeek, uploadsGB, ... }
```

Các aggregate QueryBuilder: `COUNT`, nhóm theo tuần, sum dung lượng file (bảng `storage_objects` — khuyến khích: từ bài 07 mọi upload nên ghi 1 dòng metadata key+size để thống kê dễ).

> **Vì sao bảng metadata storage:** tính dung lượng từ R2 API chậm và tốn call; ghi 1 dòng mỗi khi xin presign (hoặc callback upload thành công) = tổng hợp bằng 1 GROUP BY.

### Bước 3: Trang admin

- `/admin` — dashboard: 4 stat cards (user, lớp, buổi tuần này, dung lượng) + chart hoạt động theo tuần (Recharts)
- `/admin/users` — bảng user: tìm kiếm, xem lớp của từng người, **deactivate** (khóa tài khoản vi phạm — không xóa vì dữ liệu lớp còn ràng buộc)
- `/admin/classes` — danh sách mọi lớp + gia sư chủ + số buổi

> **Vì sao deactivate thay vì delete:** user có lớp/bài/bài nộp ràng buộc — xóa cascade phá dữ liệu lịch sử lớp; khóa = không đăng nhập được nhưng dữ liệu nguyên vẹn.

### Bước 4: Audit log (nền tảng)

Bảng `audit_logs`: ai làm gì khi nào (`action`, `userId`, `metadata` jsonb). Ghi tại các hành động nhạy: khóa user, xóa lớp, đổi role.

> **Vì sao audit log:** hệ thống nhiều vai trò — khi có tranh chấp ("ai xóa lớp của tôi?") cần bằng chứng. Ghi từ bây giờ rẻ hơn làm lại sau nhiều.

### Bước 5: Bảo vệ

- Middleware `/admin/*` — requireRole('admin')
- Không có link admin trong UI thường (truy cập trực tiếp bằng URL) — an toàn nhờ guard, không nhờ "ẩn link"

> **Vì sao nói rõ điểm này:** "ẩn link" không phải bảo mật — guard ở server là lớp duy nhất có giá trị. UI ẩn chỉ để khỏi rối mắt.

## ✅ Tiêu chí hoàn thành

- [ ] Đăng nhập admin → dashboard đúng số liệu
- [ ] Deactivate user → user đó không đăng nhập được, dữ liệu còn
- [ ] Audit log ghi đủ các hành động nhạy
- [ ] Tutor/student truy cập /admin → 403
- [ ] Commit: `lesson-27: admin panel + audit logs`

## 📝 Tự kiểm tra

1. Vì sao seed admin thay vì cho tự đăng ký role admin?
2. Deactivate khác delete — hậu quả dữ liệu của delete cascade?
3. Audit log lưu ở đâu, các action nào bắt buộc ghi?
4. "Ẩn link ≠ bảo mật" — giải thích cơ chế hai lớp (UI vs guard)?
