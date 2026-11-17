# Bài 12 — Học sinh nộp bài (submissions + bảng theo dõi)

## 🎯 Mục tiêu

Học sinh xem bài được giao, nộp file/ảnh hoặc viết text, nộp lại trước hạn; gia sư thấy bảng trạng thái cả lớp (chưa nộp / nộp / trễ).

## 🧠 Kiến thức cần nắm

- LEFT JOIN lấy cả người chưa nộp → [glossary/04-database.md](../../glossary/04-database.md#5-join)
- QueryBuilder + CASE WHEN → [glossary/03-backend-orm.md](../../glossary/03-backend-orm.md#6-querybuilder)

## 📋 Các bước

### Bước 1: Entity submissions

```typescript
@Entity({ name: 'submissions' })
export class SubmissionEntity {
  @PrimaryGeneratedColumn() id: number;
  @Column({ name: 'assignment_id' }) assignmentId: number;
  @ManyToOne(() => AssignmentEntity) @JoinColumn({ name: 'assignment_id' }) assignment: AssignmentEntity;
  @Column({ name: 'student_id' }) studentId: number;

  @Column({ type: 'enum', enum: ['file', 'text'] }) kind: 'file' | 'text';
  @Column({ type: 'text', nullable: true }) textAnswer?: string;
  @Column({ type: 'jsonb', default: [] }) files: { key: string; name: string; contentType: string }[];

  @Column({ name: 'submitted_at', type: 'timestamptz' }) submittedAt: Date;
  // điểm/nhận xét ở bài 13 (thêm cột — migration riêng)
  @Index(['assignmentId', 'studentId'])
  // unique: 1 HS 1 bản nộp "hiện tại" cho 1 bài (nộp lại = ghi đè)
  @Unique(['assignmentId', 'studentId'])
}
```

**Quyết định thiết kế: nộp lại = ghi đè bản duy nhất** (không giữ lịch sử nộp).

> **Vì sao ghi đè thay vì nhiều bản:** danh sách nộp của gia sư chỉ quan tâm bản mới nhất; lưu nhiều bản cần thêm logic "bản nào là hiện tại" ở mọi query. Về sau nếu cần lịch sử nộp (backlog) → thêm bảng `submission_history`, không phá hiện tại.
> **Vì sao `submittedAt` lưu tay (không CreateDateColumn):** vì là thời điểm **nộp**, được server ghi khi chấp nhận bản nộp — phân biệt với thời điểm tạo bản ghi nội bộ.

### Bước 2: API nộp bài

```
POST /api/assignments/[id]/submit     nộp / nộp lại (student, phải là assignee)
GET  /api/assignments/mine            danh sách bài của học sinh (gộp mọi lớp)
GET  /api/assignments/[id]/board      bảng trạng thái cả lớp (tutor chủ lớp)
```

Service `submit`:

```typescript
async submit(assignmentId: number, studentId: number, dto: SubmitDto) {
  const a = await this.assignmentRepo.findOneByOrFail({ id: assignmentId });
  if (!(await this.isAssignee(a, studentId))) throw new ForbiddenError('Bạn không được giao bài này');
  if (new Date() > a.dueAt) throw new BadRequestError('Đã quá hạn chót');
  if (dto.kind === 'text' && !dto.textAnswer?.trim()) throw new BadRequestError('Nội dung trống');
  if (dto.kind === 'file' && !(dto.files?.length)) throw new BadRequestError('Chưa đính kèm file');

  await this.subRepo.save({ assignmentId, studentId, ...dto, submittedAt: new Date() }); // upsert
}
```

> **Vì sao server chặn nộp sau hạn** dù client cũng ẩn nút: client là tiện nghi, server là luật. Đồng hồ máy người dùng có thể sai — chặn theo giờ server (UTC) mới tin được.

### Bước 3: Bảng theo dõi — QueryBuilder trọng tâm bài này

```typescript
getSubmissionBoard(assignmentId: number) {
  return this.dataSource.createQueryBuilder()
    .select('u.id', 'studentId')
    .addSelect('u.full_name', 'studentName')
    .addSelect('sub.id', 'submissionId')
    .addSelect('sub.submitted_at', 'submittedAt')
    .addSelect(
      `CASE
         WHEN sub.id IS NULL THEN 'not_submitted'
         WHEN sub.submitted_at > a.due_at THEN 'late'
         ELSE 'submitted' END`,
      'status')
    .from(ClassStudentEntity, 'cs')
    .innerJoin(UserEntity, 'u', 'u.id = cs.student_id')
    .innerJoin(AssignmentEntity, 'a', 'a.class_id = cs.class_id AND a.id = :aid', { aid: assignmentId })
    // mode 'selected': chỉ những HS trong danh sách gán
    .andWhere("a.assignee_mode = 'all' OR (a.selected_student_ids @> :sids)")
    .leftJoin(SubmissionEntity, 'sub',
      'sub.assignment_id = a.id AND sub.student_id = cs.student_id')
    .where('cs.class_id = (SELECT class_id FROM assignments WHERE id = :aid)', { aid: assignmentId })
    .orderBy('status', 'ASC')     // chưa nộp nổi lên đầu
    .getRawMany();
}
```

> **Vì sao LEFT JOIN là trái tim của bảng này:** học sinh **chưa nộp không có dòng** trong `submissions` — chỉ LEFT JOIN giữ họ lại với `status = not_submitted`. INNER JOIN làm bảng "không bao giờ thấy ai chưa nộp" — mất chính người cần nhắc.
> **Vì sao CASE WHEN tính trạng thái lúc đọc:** trạng thái nộp/trễ phụ thuộc **quan hệ giữa submittedAt và dueAt** — lưu cột `status` vào DB sẽ lỗi khi gia sư gia hạn hạn chót (status cũ sai). Tính lúc đọc = luôn nhất quán.

### Bước 4: Danh sách bài của học sinh

`GET /api/assignments/mine` — QueryBuilder: tất cả assignment mà HS là assignee, JOIN bản nộp của mình (LEFT), kèm trạng thái (đã nộp/đến hạn trong x ngày/quá hạn). Sắp: sắp đến hạn trước.

### Bước 5: UI

- **Học sinh** `/student/assignments`: danh sách nhóm "Sắp đến hạn" (đỏ nếu <24h) / "Đã nộp" / "Đã chấm". Chi tiết bài: xem đề + file, nút **Nộp bài** → dialog chọn "Chụp ảnh / Tải ảnh / Viết text" → upload (bài 07) → gửi. Nút đổi thành "Nộp lại" trước hạn.
- **Gia sư**: bảng theo dõi trong trang chi tiết bài: từng HS + badge trạng thái (❌ chưa / ✅ đã / ⏰ trễ) + submittedAt; click HS → màn hình chấm (bài 13).

> **Vì sao "chụp ảnh" nổi bật ở nộp bài:** thói quen thực tế VN là học sinh chụp vở nộp — mobile input `capture="environment"` mở camera trực tiếp, luồng nộp ≤ 3 chạm (chi tiết mobile ở bài 26).

## ✅ Tiêu chí hoàn thành

- [ ] Nộp text / nộp file (nhiều file) thành công
- [ ] Nộp sau hạn → 400; nộp lại trước hạn → ghi đè
- [ ] Bảng theo dõi: đủ cả HS, trạng thái đúng kể cả trễ
- [ ] HS không được giao (selected) không thấy bài
- [ ] Commit: `lesson-12: submissions + status board`

## 📝 Tự kiểm tra

1. Vì sao LEFT JOIN quyết định sống còn cho bảng theo dõi?
2. Trạng thái 'late' tính lúc đọc thay vì lưu cột — lợi ích khi gia hạn?
3. Nộp lại = ghi đè — trade-off chấp nhận là gì? Cách bổ sung lịch sử sau này?
4. Client ẩn nút nộp rồi, vì sao server vẫn phải chặn?
