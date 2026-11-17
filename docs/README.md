# 📚 TutorSpaceD — Tài liệu học & xây dựng dự án

> **TutorSpaceD** là web cho phép **gia sư quản lý lớp học, lịch giảng dạy, phòng học trực tuyến và hệ thống bài tập** — dành cho gia sư dạy học sinh (không có vai trò phụ huynh).
>
> Dự án này sinh ra với 2 mục tiêu: (1) có một sản phẩm thật để dùng, (2) **là lộ trình học lập trình web fullstack** — mỗi chức năng là một "bài học" có hướng dẫn từng bước.

---

## 🗺️ Cấu trúc tài liệu

```
docs/
├── README.md                     ← bạn đang ở đây (bản đồ toàn bộ)
├── plan/                         ← KẾ HOẠCH DỰ ÁN (đọc trước khi làm)
│   ├── 01-tong-quan-du-an.md           Dự án là gì, ai dùng, luồng nghiệp vụ
│   ├── 02-danh-sach-chuc-nang.md       TOÀN BỘ chức năng của dự án hoàn chỉnh
│   ├── 03-tech-stack-va-kien-truc.md   Công nghệ chọn và lý do
│   └── 04-lo-trinh-toan-du-an.md       Thứ tự làm, giai đoạn, tiêu chí xong
├── glossary/                     ← TỪ ĐIỂN (đọc khi gặp thuật ngữ lạ)
│   ├── README.md                       Cách dùng từ điển + mục lục
│   ├── 01-web-api-co-ban.md            Web, HTTP, API, JSON, REST...
│   ├── 02-frontend.md                  React, Next.js, component, state...
│   ├── 03-backend-orm.md               Server, TypeORM, Repository, Migration...
│   ├── 04-database.md                  PostgreSQL, bảng, quan hệ, index...
│   ├── 05-realtime-webrtc.md           WebSocket, Video SDK, SFU, TURN...
│   └── 06-devops-deploy.md             Docker, CI/CD, deploy, biến môi trường...
└── lessons/                      ← BÀI LÀM (thứ tự từ 01 → 28)
    ├── 01-khoi-tao-du-an/
    ├── 02-typeorm-postgres/
    ├── ... (mỗi folder = 1 chức năng/việc)
    └── 28-deploy-production/
```

---

## 🎓 Cách học với bộ tài liệu này

1. **Đọc `plan/01` → `plan/04`** để hiểu bạn đang xây cái gì và đi tới đâu.
2. **Làm các bài trong `lessons/` theo đúng thứ tự** — mỗi bài xây trên bài trước.
3. **Khi gặp thuật ngữ lạ** → mở `glossary/` (mục lục ở `glossary/README.md`) để đối chiếu.
4. Mỗi bài làm có cấu trúc thống nhất:

```
🎯 Mục tiêu         — làm xong bài này có được gì
🧠 Kiến thức cần     — khái niệm + link glossary
📋 Các bước         — từng bước làm, mỗi bước có phần "Vì sao làm bước này"
✅ Tiêu chí hoàn thành — cách kiểm tra bài đã xong đúng chưa
📝 Tự kiểm tra      — câu hỏi giúp bạn chắc chắn đã hiểu, không chỉ làm theo
```

> ⚠️ **Quy tắc vàng khi học:** đừng chỉ copy code. Sau mỗi bước, thử tự trả lời "vì sao bước này tồn tại?". Nếu không trả lời được → đọc lại phần giải thích hoặc tìm trong glossary.

---

## 📚 Danh sách 28 bài (theo thứ tự làm)

### Giai đoạn A — Nền tảng (bài 01–07)
| Bài | Tên | Nội dung chính |
|---|---|---|
| 01 | Khởi tạo dự án | Next.js, TypeScript, Tailwind, shadcn/ui, cấu trúc folder |
| 02 | TypeORM + PostgreSQL | Kết nối DB, entity đầu tiên, migration |
| 03 | Đăng ký & đăng nhập | Auth.js, session, hash mật khẩu |
| 04 | Phân quyền gia sư/học sinh | Role, guard, middleware |
| 05 | Quản lý lớp học | CRUD lớp, quan hệ N-N |
| 06 | Mời học sinh bằng code | Sinh mã tham gia, join lớp |
| 07 | Upload file lên R2 | Presigned URL, lưu ảnh/file |

### Giai đoạn B — Lịch giảng dạy (bài 08–10)
| Bài | Tên | Nội dung chính |
|---|---|---|
| 08 | Buổi học đơn lẻ | Entity sessions, tạo/hiển thị buổi |
| 09 | Lịch lặp định kỳ | rrule, sinh chuỗi buổi, override buổi lẻ |
| 10 | UI lịch tổng & lịch lớp | Component calendar, màu lớp, lọc |

### Giai đoạn C — Bài tập (bài 11–15)
| Bài | Tên | Nội dung chính |
|---|---|---|
| 11 | Giao bài tập | Entity assignments, form giao bài |
| 12 | Học sinh nộp bài | Submissions, trạng thái nộp/trễ |
| 13 | Chấm bài | Màn hình chấm, điểm + nhận xét |
| 14 | Thông báo in-app | Realtime notify, badge |
| 15 | Email nhắc lịch | BullMQ queue, job định giờ |

### Giai đoạn D — Phòng học (bài 16–22)
| Bài | Tên | Nội dung chính |
|---|---|---|
| 16 | Cài LiveKit server | SFU, TURN, Docker, token |
| 17 | Video + audio | LiveKit client, join phòng |
| 18 | Quyền điều khiển của gia sư | Tắt mic học sinh, khóa phòng |
| 19 | Bảng trắng (tldraw) | Sync multiplayer, upload đề |
| 20 | Chat thời gian thực | Socket.io, lưu tin nhắn |
| 21 | Share màn hình & giơ tay | Screen share, hàng đợi tay giơ |
| 22 | Ghi hình buổi học | LiveKit Egress, lưu MP4, trang xem lại |

### Giai đoạn E — Hoàn thiện (bài 23–28)
| Bài | Tên | Nội dung chính |
|---|---|---|
| 23 | Trắc nghiệm tự chấm | Đề trắc nghiệm, chấm tự động |
| 24 | Sổ điểm & xuất Excel | Tổng hợp điểm, export file |
| 25 | Báo cáo sau buổi | Gia sư viết feedback, học sinh nhận |
| 26 | Giao diện học sinh (mobile) | Mobile-first, luồng nộp ảnh 3 chạm |
| 27 | Trang quản trị (admin) | Quản lý user, thống kê |
| 28 | Deploy production | VPS, Docker, domain, HTTPS, monitoring |

---

## 🚀 Bắt đầu từ đâu?

→ Mở [`lessons/01-khoi-tao-du-an/huong-dan.md`](../lessons/01-khoi-tao-du-an/huong-dan.md) và bắt tay vào làm.
