# Bài 25 — Báo cáo sau buổi học

## 🎯 Mục tiêu

Sau mỗi buổi, gia sư viết feedback ngắn (đã dạy gì, nhận xét HS); học sinh nhận thông báo + email, xem trong trang buổi học.

## 🧠 Kiến thức cần nắm

- Artifact buổi học (bảng trắng bài 19, chat bài 20, ghi hình bài 22)
- Thông báo (bài 14) + email (bài 15)

## 📋 Các bước

### Bước 1: Entity session_reports

```typescript
@Entity({ name: 'session_reports' })
export class SessionReportEntity {
  @PrimaryGeneratedColumn() id: number;
  @Column({ name: 'session_id' }) sessionId: number;
  @Unique(['sessionId'])                                    // 1 báo cáo/buổi

  @Column({ type: 'text' }) content: string;                // nội dung đã dạy + nhận xét chung
  @Column({ type: 'jsonb', default: [] }) studentNotes: { studentId: number; note: string }[];
  // nhận xét riêng từng HS (tùy chọn)

  @ManyToOne(() => UserEntity) @JoinColumn({ name: 'created_by' }) createdBy: UserEntity;
  @CreateDateColumn({ name: 'created_at' }) createdAt: Date;
}
```

> **Vì sao 1-1 với session và unique:** buổi chỉ có 1 báo cáo chính thức; sửa = cập nhật (giữ lịch sử ra khỏi phạm vi — đơn giản, đủ dùng).
> **Vì sao nhận xét riêng từng HS:** lớp nhóm nhỏ — "An nắm nhanh phần hệ, Bình cần thêm bài về đổi dấu" — giá trị lớn với HS mà tốn 1 jsonb.

### Bước 2: Luồng "khuyến khích viết"

Nút "Kết thúc buổi" (bài 21) → sau khi lưu artifact → mở dialog viết báo cáo **ngay** (pre-fill gợi ý: chủ đề buổi từ `session.topic`, list HS). Lưu xong mới quay về lịch.

> **Vì sao nhắc ngay sau buổi:** thống kê hành vi gia sư — viết báo cáo sau 10 phút với dữ liệu còn nhớ nguyên vẹn; sau 1 ngày là quên. Đặt đúng thời điểm = tỷ lệ hoàn thành cao nhất. Đây là quyết định sản phẩm, không phải kỹ thuật.

### Bước 3: API

```
POST /api/sessions/[id]/report     tạo/sửa báo cáo (tutor chủ lớp)
GET  /api/sessions/[id]/report     đọc (thành viên lớp)
```

### Bước 4: Gửi thông báo + email

Khi tạo báo cáo:
- `notifyUser` cho từng HS trong buổi (type `session_report`) → badge + link
- Email (bài 15): template "Báo cáo buổi học Toán 9-A — 20/11" với nội dung rút gọn + nút "Xem chi tiết"

> **Vì sao email cho báo cáo:** đây là "sản phẩm cụ thể" HS/phụ huynh tâm lý mong nhận — đẩy thẳng vào hộp thư tăng cảm nhận giá trị, khác nhắc lịch (chỉ thời gian).

### Bước 5: UI xem báo cáo

Trang buổi học (đã có ở bài 22: video/bảng/chat): thêm tab **Báo cáo** hiển thị content + nhận xét riêng của mình (HS chỉ thấy note của mình, không thấy note HS khác!).

> **Vì sao ẩn note khác HS:** nhận xét cá nhân có thể nhạy cảm ("Bình chậm hơn cả lớp") — lộ cho bạn học gây tổn thương/chê cười. Quyền dữ liệu cá nhân ở đây là luật vô hình nhưng sống còn.

### Bước 6: Thống kê cho gia sư

Dashboard gia sư thêm chỉ số: "x/y buổi gần nhất đã viết báo cáo" — nhắc gia sư củng cố thói quen.

## ✅ Tiêu chí hoàn thành

- [ ] Kết thúc buổi → dialog báo cáo mở đúng, pre-fill chủ đề
- [ ] HS nhận thông báo + email, xem báo cáo + note của mình
- [ ] HS không thấy note của HS khác (test 2 tài khoản)
- [ ] Dashboard hiển thị tỷ lệ báo cáo
- [ ] Commit: `lesson-25: session reports + notifications`

## 📝 Tự kiểm tra

1. Vì sao dialog báo cáo mở ngay sau kết thúc buổi? Quyết định này thuộc loại gì (kỹ thuật/sản phẩm)?
2. `studentNotes` jsonb — học sinh đọc được note của người khác qua API không? Chặn thế nào?
3. Báo cáo 1-1 session — điều gì xảy ra nếu gia sư kết thúc mà không viết? (thiết kế: để trống, sau đó viết được — API tạo nếu chưa có)
4. Kết nối artifact: báo cáo + bảng trắng + chat hợp thành "câu chuyện buổi học" — giá trị giữ chân HS ở điểm nào?
