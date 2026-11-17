# 04 — Database: PostgreSQL

> Mục lục: [Database & bảng](#1) • [Kiểu dữ liệu](#2) • [Khóa chính & khóa ngoại](#3) • [Quan hệ](#4) • [JOIN](#5) • [Index](#6) • [Transaction](#7) • [UTC & múi giờ](#8) • [Sơ đồ DB của dự án](#9)

## 1. Database & bảng {#1}

**Database** = nơi lưu dữ liệu lâu dài (server tắt, dữ liệu vẫn còn). **PostgreSQL** = loại database quan hệ: dữ liệu nằm trong **bảng** — giống Excel với hàng (record) và cột (column):

```
Bảng classes
| id | name      | color | teacher_id | created_at          |
|----|-----------|-------|------------|---------------------|
| 1  | Toán 9-A  | blue  | 7          | 2026-11-16 10:00:00 |
| 2  | Lý 10-B   | cyan  | 7          | 2026-11-17 09:30:00 |
```

SQL = ngôn ngữ hỏi DB: `SELECT * FROM classes WHERE teacher_id = 7;`

## 2. Kiểu dữ liệu hay dùng {#2}

| Kiểu | Dùng cho | Trong dự án |
|---|---|---|
| `serial` / `integer` | số tự tăng / số | `id` |
| `varchar(n)` / `text` | chữ ngắn / chữ dài | `name` / mô tả bài tập |
| `timestamptz` | thời gian có múi giờ | `starts_at`, `due_at` |
| `boolean` | đúng/sai | `is_locked` (phòng đã khóa) |
| `enum` | giá trị trong danh sách cố định | `status` buổi học |
| `jsonb` | dữ liệu cấu trúc linh hoạt | dữ liệu bảng trắng, đề trắc nghiệm |

## 3. Khóa chính & khóa ngoại {#3}

- **Khóa chính (primary key):** số định danh duy nhất mỗi hàng — `id`.
- **Khóa ngoại (foreign key):** cột trỏ tới `id` của bảng khác — nối bảng với nhau.

**Trong dự án:** `sessions.class_id` trỏ tới `classes.id` → biết mỗi buổi thuộc lớp nào.

## 4. Quan hệ {#4}

| Quan hệ | Nghĩa | Ví dụ dự án |
|---|---|---|
| **1-N (một-nhiều)** | 1 gia sư có nhiều lớp | `users` → `classes` |
| **N-N (nhiều-nhiều)** | 1 lớp có nhiều HS, 1 HS học nhiều lớp | bảng trung gian `class_students` |
| **1-1** | 1 buổi có 1 phòng học riêng | `sessions` → `rooms` |

**Bảng trung gian** cho N-N trông thế này:

```
class_students
| class_id | student_id | joined_at  |
|----------|------------|------------|
| 1        | 3          | 2026-11-16 |
| 1        | 4          | 2026-11-16 |
| 2        | 3          | 2026-11-17 |  ← An (id=3) học cả 2 lớp
```

## 5. JOIN {#5}

Ghép dữ liệu nhiều bảng khi hỏi:

```sql
SELECT c.name, u.full_name
FROM classes c
INNER JOIN users u ON u.id = c.teacher_id;
-- INNER JOIN: chỉ lấy hàng khớp cả 2 bảng
-- LEFT JOIN:  lấy tất cả bên trái, thiếu bên phải thì để NULL
```

**Vì sao LEFT JOIN quan trọng:** bảng "chưa nộp bài" phải lấy **tất cả học sinh** lớp (LEFT) và bài nộp nếu có (thiếu = NULL = chưa nộp). Nếu INNER JOIN sẽ mất luôn học sinh chưa nộp — mất đúng người cần nhắc!

## 6. Index {#6}

**Mục lục của bảng** — giúp tìm nhanh thay vì quét cả bảng. Tự tạo cho khóa chính/khóa ngoại; tạo tay cho cột hay tra:

```sql
CREATE INDEX idx_sessions_class_starts ON sessions (class_id, starts_at);
```

**Trong dự án:** lịch tổng luôn hỏi "các buổi từ ngày X đến ngày Y của các lớp Z" → index `(class_id, starts_at)` là bắt buộc khi dữ liệu buổi lên hàng nghìn.

## 7. Transaction {#7}

Gom nhiều câu lệnh thành **1 khối cả hoặc không**:

```sql
BEGIN;
INSERT INTO sessions ...;        -- sinh chuỗi buổi
INSERT INTO assignments ...;
COMMIT;                           -- thành công cả 2 mới lưu
-- nếu lỗi giữa chừng → ROLLBACK, không lưu gì cả
```

**Ví dụ dự án:** sinh chuỗi 16 buổi lặp — nếu lỗi ở buổi 13 mà không transaction sẽ có 12 buổi "ma" rác trong DB.

## 8. UTC & múi giờ {#8}

**UTC** = giờ chuẩn quốc tế (VN = UTC+7). **Quy tắc dự án: lưu `timestamptz` dạng UTC, chỉ hiển thị theo giờ người dùng ở frontend.** Nếu lưu giờ VN dạng "19:00" không kèm múi giờ, khi app có user ở múi giờ khác sẽ loạn toàn bộ lịch.

## 9. Sơ đồ database của TutorSpaceD {#9}

```
users ──< classes ──< sessions ──< rooms
           │  │          │
           │  │          └──< session_attendance (điểm danh)
           │  │
           │  └──< class_students >── users (học sinh)
           │
           ├──< assignments ──< submissions >── users
           │         │
           │         └──< quiz_questions / quiz_options (bài 23)
           │
           ├──< recurring_rules (chuỗi lặp, bài 09)
           └──< notifications (bài 14)

sessions ── whiteboard_snapshots (ảnh bảng trắng, bài 19)
sessions ── recordings (ghi hình, bài 22)
```

Sẽ chi tiết hóa từng bảng đúng lúc làm bài tương ứng — đọc sơ đồ này để định hướng thôi.
