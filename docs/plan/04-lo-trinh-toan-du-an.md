# 04 — Lộ trình toàn dự án & thứ tự làm

> Dự án làm **đầy đủ** (không cắt MVP), nhưng vẫn phải làm theo thứ tự phụ thuộc kỹ thuật. Mỗi giai đoạn kết thúc bằng một "trạm kiểm tra" — sản phẩm phải chạy được cái gì đó thật.

## Bản đồ giai đoạn

```
GIAI ĐOẠN A        GIAI ĐOẠN B         GIAI ĐOẠN C         GIAI ĐOẠN D         GIAI ĐOẠN E
Nền tảng           Lịch giảng dạy      Bài tập             Phòng học           Hoàn thiện
Bài 01–07          Bài 08–10           Bài 11–15           Bài 16–22           Bài 23–28
─────────────      ─────────────       ─────────────       ─────────────       ─────────────
Chạy được          Đặt được lịch       Giao-nộp-chấm       Dạy được            Sản phẩm hoàn chỉnh
đăng nhập +        cho lớp thật        bài trọn vòng       buổi học thật        + deploy production
lớp học            + nhắc lịch         + thông báo         (video+bảng)        + sổ điểm + admin
```

## GIAI ĐOẠN A — Nền tảng (bài 01–07)

**Mục tiêu:** khung dự án chạy được, có đăng nhập, có lớp học và học sinh tham gia được.

| Bài | Việc | Phụ thuộc |
|---|---|---|
| 01 | Khởi tạo dự án (Next.js, TS, Tailwind, shadcn) | — |
| 02 | TypeORM + PostgreSQL + migration đầu tiên | 01 |
| 03 | Đăng ký & đăng nhập (Auth.js) | 01, 02 |
| 04 | Phân quyền gia sư/học sinh | 03 |
| 05 | Quản lý lớp học (CRUD + danh sách HS) | 02, 04 |
| 06 | Mời học sinh bằng mã tham gia | 05 |
| 07 | Upload file lên R2 (avatar, tài liệu) | 03 |

**🏁 Trạm kiểm tra A:** *Tạo tài khoản gia sư → tạo lớp → nhận mã → học sinh đăng ký, nhập mã → vào được lớp.*

## GIAI ĐOẠN B — Lịch giảng dạy (bài 08–10)

**Mục tiêu:** gia sư quản lý được lịch tổng + lịch từng lớp, kể cả lịch lặp định kỳ.

| Bài | Việc | Phụ thuộc |
|---|---|---|
| 08 | Buổi học đơn lẻ (entity, tạo, hiển thị) | 05 |
| 09 | Lịch lặp định kỳ (rrule, override, nghỉ/dời) | 08 |
| 10 | UI lịch tổng & lịch lớp (calendar, màu, lọc) | 08 |

**🏁 Trạm kiểm tra B:** *Tạo lịch lặp T3–T5 19h → dời 1 buổi → nghỉ 1 buổi → lịch tổng hiển thị đúng, màu đúng.*

## GIAI ĐOẠN C — Bài tập (bài 11–15)

**Mục tiêu:** vòng đời bài tập trọn vẹn: giao → nộp → chấm → xem kết quả, kèm thông báo.

| Bài | Việc | Phụ thuộc |
|---|---|---|
| 11 | Giao bài tập (assignments, gán người, file đính kèm) | 05, 07 |
| 12 | Học sinh nộp bài (submissions, trạng thái trễ) | 11 |
| 13 | Chấm bài (điểm, nhận xét, chấm tiếp liên tục) | 12 |
| 14 | Thông báo in-app (Socket.io + badge) | 11 |
| 15 | Email nhắc lịch + bài tập (Redis + BullMQ) | 08, 14 |

**🏁 Trạm kiểm tra C:** *Giao bài có hạn chót → học sinh nộp ảnh → gia sư chấm 3 bài liên tiếp → học sinh nhận thông báo + email.*

## GIAI ĐOẠN D — Phòng học (bài 16–22) — giai đoạn phức tạp nhất

**Mục tiêu:** phòng học thật: video, bảng trắng, chat, share màn hình, ghi hình.

| Bài | Việc | Phụ thuộc |
|---|---|---|
| 16 | Cài LiveKit server (Docker, TURN, token service) | — |
| 17 | Video + audio (join phòng từ buổi học) | 16, 10 |
| 18 | Quyền điều khiển gia sư (tắt mic, khóa phòng) | 17 |
| 19 | Bảng trắng tldraw (sync, upload đề, lưu ảnh) | 17 |
| 20 | Chat thời gian thực (Socket.io, lưu DB) | 17 |
| 21 | Share màn hình + giơ tay | 17 |
| 22 | Ghi hình buổi học (Egress → R2 → trang xem lại) | 17, 07 |

**🏁 Trạm kiểm tra D:** *Lớp 5 người thật (2 laptop + 3 điện thoại) dạy 45 phút ổn định: video, bảng trắng 2 chiều, chat, giơ tay, share, kết thúc lưu vết.*

## GIAI ĐOẠN E — Hoàn thiện (bài 23–28)

**Mục tiêu:** các tính năng nâng cao + đưa sản phẩm lên production thật.

| Bài | Việc | Phụ thuộc |
|---|---|---|
| 23 | Trắc nghiệm tự chấm | 11 |
| 24 | Sổ điểm + xuất Excel + biểu đồ tiến độ | 13 |
| 25 | Báo cáo sau buổi (gia sư viết, HS nhận) | 13, 14 |
| 26 | Giao diện học sinh mobile-first (luồng nộp ảnh 3 chạm) | 12 |
| 27 | Trang admin (quản user, thống kê) | 03 |
| 28 | Deploy production (VPS, Docker, domain, HTTPS, Sentry) | tất cả |

**🏁 Trạm kiểm tra E (đích cuối):** *1–2 gia sư thật dùng liên tục 2 tuần trên production mà không cần bạn can thiệp.*

## Nguyên tắc trong suốt quá trình làm

1. **Mỗi bài xong phải chạy được** — không để dở và sang bài tiếp.
2. **Mỗi thay đổi schema = migration mới**, commit riêng.
3. **Mỗi bài có commit git riêng** với message mô tả bài (`lesson-05: class management`). Đó là "nhật ký học" của bạn.
4. Gặp thuật ngữ lạ → tra `glossary/`. Không hiểu bước → đọc lại phần "Vì sao".
5. Yêu cầu mới phát sinh ngoài danh sách → ghi vào backlog trong `02-danh-sach-chuc-nang.md`, không phá thứ tự bài.

## Ước lượng thời gian (1 người, học đi làm)

| Giai đoạn | Thời gian ước tính |
|---|---|
| A — Nền tảng | 3–4 tuần |
| B — Lịch | 2 tuần |
| C — Bài tập | 2–3 tuần |
| D — Phòng học | 3–4 tuần |
| E — Hoàn thiện | 3–4 tuần |
| **Tổng** | **~13–17 tuần** (kèm thời gian học hiểu — không chỉ làm theo) |
