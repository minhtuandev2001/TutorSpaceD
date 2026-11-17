# Bài 15 — Email nhắc lịch (BullMQ + Resend)

## 🎯 Mục tiêu

Email nhắc tự động trước buổi học 30 phút (cho gia sư + học sinh); email bài tập mới/đã chấm; template email đẹp, hàng đợi không chặn request.

## 🧠 Kiến thức cần nắm

- BullMQ, Redis → [glossary/03-backend-orm.md](../../glossary/03-backend-orm.md#10-bullmq--redis)
- Cron job (nền từ bài 08)

## 📋 Các bước

### Bước 1: Chọn nhà cung cấp email

Dùng **Resend** (miễn phí 3.000 email/tháng, API đơn giản). Tài khoản → tạo API key → thêm domain gửi thật (bắt buộc cho production; dev dùng địa chỉ test).

```bash
npm i resend
```

> **Vì sao không SMTP trần:** gửi mail từ IP thường vào spam ngay; Resend/SES lo SPF/DKIM/reputation. Đây là hạ tầng "không thể tự làm nhanh" như LiveKit với video.

### Bước 2: Mailer service + template

`src/modules/mailer/mailer.service.ts` — enqueue vào BullMQ (không gửi trực tiếp):

```typescript
await mailQueue.add('send-email', { to, subject, template: 'session-reminder', data }, {
  delay: sendAtMs - Date.now(),       // hẹn giờ gửi!
  attempts: 3,                        // thử 3 lần khi lỗi
  backoff: { type: 'exponential', delay: 60_000 },
  jobId: `session-reminder-${sessionId}-${userId}`,   // chống trùng
});
```

Worker (`worker.mjs` chạy process riêng): nhận job → render template → `resend.emails.send()`.

> **Vì sao delay + jobId:** hẹn giờ gửi là tính năng lõi ("trước buổi 30 phút"); jobId deterministic để cron tạo lại job không sinh email đôi (idempotence như bài 14).
> **Vì sao attempts + backoff:** Resend lỗi tạm thời (network) — thử lại tự động thay vì mất email; exponential tránh dập dềnh gọi API.

### Bước 3: Cron sinh job nhắc

Job chạy mỗi 5 phút: query buổi `upcoming` có `startsAt` trong [now+30phút, now+35phút] → với mỗi buổi, với mỗi thành viên → add job `session-reminder` (jobId chặn trùng).

> **Vì sao quét khoảng 5 phút:** cron chậm trễ vài giây không bỏ sót buổi; khoảng hẹp để không gửi sớm hơn dự định.

### Bước 4: Templates email

React Email hoặc HTML template đơn giản — 3 template:
1. **session-reminder:** "Buổi học Toán 9-A bắt đầu lúc 19:00 (còn 30 phút)" + nút "Vào phòng học" (link deep-link có sẵn từ bài 08) + link lịch.
2. **assignment-new:** tên bài, hạn chót, nút xem bài.
3. **assignment-graded:** điểm + nhận xét (bản rút gọn) + nút xem chi tiết.

Gửi **tiếng Việt UTF-8**, subject ≤ 60 ký tự.

> **Vì sao mỗi email có CTA duy nhất:** email nhắc có mục tiêu một việc — mở link. Nhiều CTA chia sự chú ý, tỷ lệ bấm giảm.

### Bước 5: Tùy chọn tắt email

Cột `emailOptOut` jsonb trên users: `{ sessionReminder: true, assignmentNew: true, ... }` — trang profile có toggle; mailer check trước khi enqueue.

> **Vì sao:** email vô chủ → đưa thẳng vào spam, phí tín dụng domain gửi. Người dùng chọn được = giữ chất lượng channel.

### Bước 6: Test + kiểm soát an toàn dev

- BullMQ UI (bull-board) gắn vào route admin xem queue đang chạy
- Biến `MAIL_DISABLED=true` cho dev tránh gửi thật

> **Vì sao nhìn được queue:** hàng đợi là "hộp đen" nếu không có UI — khi email không tới, bull-board cho biết job nằm que, thất bại hay đã gửi.

## ✅ Tiêu chí hoàn thành

- [ ] Tạo buổi trong ~35 phút tới → nhận đúng 1 email nhắc đúng lúc
- [ ] Giao bài + chấm bài → email đúng template
- [ ] Job lỗi retry 3 lần, thất bại hiện trên bull-board
- [ ] Tắt loại email ở profile → không nhận loại đó
- [ ] Commit: `lesson-15: email reminders (bullmq + resend)`

## 📝 Tự kiểm tra

1. Vì sao enqueue thay vì gửi ngay trong service?
2. `delay`, `attempts`, `backoff`, `jobId` — mỗi cái chặn loại sự cố nào?
3. Cron quét "now+30 đến now+35 phút" — vì sao không quét "bắt đầu từ now"?
4. Nếu worker chết giữa chừng, email của job đang chạy thành sao? (tra BullMQ stuck job)
