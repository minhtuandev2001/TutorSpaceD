# 01 — Tổng quan dự án TutorSpaceD

## 1. Dự án là gì?

**TutorSpaceD** là web giúp **gia sư** (cá nhân dạy kèm) làm được trọn vẹn công việc dạy học online:

1. **Quản lý lớp học** — tạo lớp, mời học sinh, quản lý danh sách
2. **Lịch giảng dạy** — lịch từng lớp + lịch tổng tất cả các lớp
3. **Phòng học trực tuyến** — video, bảng trắng, chat, share màn hình
4. **Hệ thống bài tập** — giao bài → học sinh nộp → chấm → học sinh xem kết quả

## 2. Ai dùng hệ thống? (chỉ 2 vai trò chính)

| Vai trò | Làm gì | Tần suất |
|---|---|---|
| **Gia sư** | Tạo lớp, tạo lịch, dạy học, giao & chấm bài | Hàng ngày — "người dùng nặng" |
| **Học sinh** | Vào lớp qua code, học online, nộp bài, xem điểm | Vài lần/tuần — "người dùng nhẹ" |

> ❗ Dự án **không có vai trò phụ huynh** — mọi báo cáo, thông báo gửi thẳng cho học sinh.
> Giai đoạn sau có thể thêm **trợ giảng** (giúp gia sư chấm bài, quản lớp).

## 3. Luồng nghiệp vụ chính

```
GIA SƯ                                    HỌC SINH
  │                                          │
  │ 1. Tạo lớp "Toán 9-A"                    │
  │ 2. Nhận mã tham gia: X7K2PQ              │
  │ 3. Gửi mã cho học sinh ────────────────► │ 4. Đăng ký tài khoản
  │                                          │ 5. Nhập mã → vào lớp
  │ 6. Tạo lịch: T3–T5 19h00 (lặp hàng tuần) │
  │                                          │ ◄── nhận nhắc lịch qua email
  │ 7. Buổi học: vào phòng học               │
  │    • Bật cam/mic                         │    • Vào phòng, giơ tay hỏi
  │    • Dạy trên bảng trắng                 │    • Viết lên bảng trắng
  │    • Chat với lớp                        │    • Chat
  │ 8. Kết thúc buổi → giao bài tập          │
  │                                          │ ◄── thông báo "có bài tập mới"
  │                                          │ 9. Làm bài, chụp ảnh, nộp
  │ 10. Chấm bài: điểm + nhận xét            │
  │                                          │ ◄── thông báo "bài đã chấm"
  │                                          │ 11. Xem điểm + nhận xét
  │ Lặp lại từ bước 7 hàng tuần...           │
```

## 4. Triết lý sản phẩm

- **Video là phương tiện, không phải sản phẩm.** Phần khó kỹ thuật (truyền video) dùng thư viện có sẵn (LiveKit). Sức mạnh thật nằm ở: lịch thông minh, vòng đời bài tập, quản lý lớp.
- **Màu lớp học là "ngôn ngữ" xuyên suốt** — lịch, dashboard, bài tập đều dùng cùng màu của lớp để người dùng định vị tức thì.
- **Gia sư = quản trị viên của hệ thống của chính mình.** Học sinh chỉ tham gia vào lớp được mời.
- **Trạng thái là vua:** mọi buổi học, bài nộp đều có trạng thái rõ ràng (sắp tới / đang dạy / đã dạy; chưa nộp / nộp / trễ / đã chấm). Giao diện phải luôn trả lời được câu hỏi: *"Bây giờ tôi phải làm gì tiếp?"*

## 5. Phạm vi: dự án HOÀN CHỈNH (không phải MVP)

Dự án xây đầy đủ toàn bộ chức năng ở [`02-danh-sach-chuc-nang.md`](02-danh-sach-chuc-nang.md) — bao gồm cả ghi hình buổi học, trắc nghiệm tự chấm, sổ điểm, trang admin. Việc xây đầy đủ nhưng **vẫn làm theo thứ tự có phụ thuộc** (xem [`04-lo-trinh-toan-du-an.md`](04-lo-trinh-toan-du-an.md)): không thể làm phòng học trước khi có lớp học và đăng nhập.
