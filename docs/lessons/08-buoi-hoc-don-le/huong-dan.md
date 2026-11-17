# Bài 08 — Buổi học đơn lẻ (sessions)

## 🎯 Mục tiêu

Gia sư tạo buổi học cho lớp (ngày, giờ, chủ đề), buổi hiển thị theo API đúng, trạng thái buổi tự cập nhật.

## 🧠 Kiến thức cần nắm

- timestamptz & UTC → [glossary/04-database.md](../../glossary/04-database.md#8-utc--múi-giờ)
- Enum, migration → [glossary/03-backend-orm.md](../../glossary/03-backend-orm.md)

## 📋 Các bước

### Bước 1: Entity sessions

```typescript
@Entity({ name: 'sessions' })
export class SessionEntity {
  @PrimaryGeneratedColumn() id: number;

  @Column({ name: 'class_id' }) classId: number;
  @ManyToOne(() => ClassEntity) @JoinColumn({ name: 'class_id' }) class: ClassEntity;

  @Column({ type: 'timestamptz' }) startsAt: Date;
  @Column({ type: 'timestamptz' }) endsAt: Date;
  @Column({ type: 'varchar', length: 200, nullable: true }) topic?: string;  // chủ đề buổi

  @Column({ type: 'enum', enum: ['upcoming', 'live', 'done', 'rested'], default: 'upcoming' })
  status: 'upcoming' | 'live' | 'done' | 'rested';

  @Column({ type: 'text', nullable: true }) note?: string;   // ghi chú nghỉ/bù

  @Column({ name: 'rule_id', type: 'integer', nullable: true }) ruleId?: number;
  // ruleId: nếu buổi sinh từ chuỗi lặp (bài 09) — null = tạo thủ công

  @CreateDateColumn({ name: 'created_at' }) createdAt: Date;
}
```

> **Vì sao từng điểm:**
> - `timestamptz` + **mọi đầu vào quy về UTC** ở service trước khi lưu — lưu "19:00 giờ VN" mà không kèm múi giờ là nguồn bug lịch kinh điển.
> - `status` 4 giá trị: `live` cần thiết để hiển thị banner "Đang diễn ra" (bài 17).
> - `ruleId` có ngay từ bây giờ dù bài 09 mới dùng — tránh migration thêm sau.
> - `endsAt` lưu thẳng thay vì tính từ duration — lịch thực tế cho phép dời giờ mà không đổi thời lượng, và ngược lại.

### Bước 2: Validate giờ ở service

```typescript
createSession(dto: { classId: number; startsAt: string; endsAt: string; topic?: string }, user) {
  if (dto.endsAt <= dto.startsAt) throw new BadRequestError('Giờ kết thúc phải sau giờ bắt đầu');
  if (new Date(dto.endsAt).getTime() - new Date(dto.startsAt).getTime() > 6 * 3600_000)
    throw new BadRequestError('Buổi học tối đa 6 giờ');
  // kiểm tra user là gia sư chủ lớp (như bài 05)
  // kiểm tra xung đột: lớp này có buổi khác chồng giờ?
}
```

> **Vì sao kiểm tra chồng giờ ngay từ đầu:** cặp buổi chồng nhau là lỗi dữ liệu đau đầu khi vẽ calendar (bài 10). Chặn ở nguồn rẻ hơn sửa về sau.

### Bước 3: API + query lịch theo khoảng

```
POST   /api/sessions                    tạo buổi (tutor chủ lớp)
PATCH  /api/sessions/[id]               sửa chủ đề/giờ
PATCH  /api/sessions/[id]/status        nghỉ buổi / đổi trạng thái
GET    /api/sessions?from=...&to=...    lịch của user trong khoảng (tutor: các lớp mình dạy; student: các lớp mình học)
```

Truy vấn lịch tổng — QueryBuilder lõi của phân hệ lịch:

```typescript
findScheduleForUser(userId: number, role: 'tutor'|'student', from: Date, to: Date) {
  const qb = this.createQueryBuilder('s')
    .innerJoin(ClassEntity, 'c', 'c.id = s.classId')
    .where('s.startsAt < :to', { to })
    .andWhere('s.endsAt > :from', { from })            // dùng khoảng chồng (overlap), không phải BETWEEN
    .orderBy('s.startsAt', 'ASC');
  if (role === 'tutor') qb.andWhere('c.teacherId = :uid', { uid: userId });
  else qb.andWhere('EXISTS (SELECT 1 FROM class_students cs WHERE cs.class_id = c.id AND cs.student_id = :uid)', { uid: userId });
  return qb.getMany();
}
```

> **Vì sao `startsAt < to AND endsAt > from`:** điều kiện "2 khoảng thời gian chồng nhau" chuẩn — buổi bắt đầu 18:30 kết thúc 20:00 vẫn phải hiện khi xem lịch "19:00–20:00". BETWEEN chỉ kiểm điểm bắt đầu là sai.

> **Vì sao EXISTS thay vì JOIN:** mỗi buổi chỉ cần xuất hiện 1 lần cho học sinh; JOIN vào `class_students` sẽ nhân bản buổi nếu có truy vấn đếm bên dưới (tương tự bài 05).

### Bước 4: Job cập nhật trạng thái tự động

Cron BullMQ mỗi 1 phút: buổi `upcoming` mà `startsAt <= now` → `live`; `live` mà `endsAt < now` → `done`. (Cần Redis + BullMQ — cài nền ở bài này, dùng thật ở bài 15.)

> **Vì sao job tự động thay vì tính "live" lúc đọc:** trạng thái trong DB là nguồn sự thật chung (báo cáo chuyên cần, thống kê sau này đọc trạng thái); tính tạm thời mỗi nơi một kiểu sẽ không khớp nhau.

### Bước 5: UI tạo buổi (form đơn giản)

Trong tab Lịch của trang lớp (bài 05 đã để placeholder): dialog "Tạo buổi" — ngày, giờ bắt đầu/kết thúc (input `datetime-local`), chủ đề. Lưu ý frontend gửi ISO string có múi giờ.

> **Vì sao `datetime-local` + ISO:** input trả giờ không kèm múi giờ → client quy đổi `new Date(value).toISOString()` trước khi gửi, server lưu nguyên UTC.

## ✅ Tiêu chí hoàn thành

- [ ] Tạo/sửa/xóa buổi; sai giờ → 400 với thông báo rõ
- [ ] Tạo buổi chồng giờ buổi khác của cùng lớp → chặn
- [ ] `GET /api/sessions` trả đúng lịch theo vai trò + khoảng thời gian
- [ ] Trạng thái buổi tự chuyển upcoming → live → done
- [ ] Commit: `lesson-08: sessions crud + status job`

## 📝 Tự kiểm tra

1. Vì sao `ruleId` có ngay trong bài này dù chưa dùng?
2. Điều kiện overlap đúng là gì? BETWEEN sai chỗ nào?
3. Buổi 18:30–20:00 có hiện khi hỏi lịch 19:00–19:30 không? Vì sao?
4. Còn lại: nếu bỏ job cron, còn cách nào để `status` đúng? (tham khảo bài sau)
