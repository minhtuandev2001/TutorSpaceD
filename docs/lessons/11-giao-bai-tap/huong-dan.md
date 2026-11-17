# Bài 11 — Giao bài tập (assignments)

## 🎯 Mục tiêu

Gia sư tạo bài tập cho lớp (hoặc cho một số học sinh cụ thể), đính file/ảnh đề, đặt hạn chót; học sinh của danh sách gán thấy bài.

## 🧠 Kiến thức cần nắm

- Upload presigned (bài 07)
- LEFT JOIN lấy cả người chưa có bản ghi → [glossary/04-database.md](../../glossary/04-database.md#5-join)
- TanStack Query, react-hook-form + zod

## 📋 Các bước

### Bước 1: Entity assignments

```typescript
@Entity({ name: 'assignments' })
export class AssignmentEntity {
  @PrimaryGeneratedColumn() id: number;
  @Column({ name: 'class_id' }) classId: number;
  @ManyToOne(() => ClassEntity) @JoinColumn({ name: 'class_id' }) class: ClassEntity;

  @Column({ type: 'varchar', length: 200 }) title: string;
  @Column({ type: 'text', nullable: true }) description?: string;

  @Column({ type: 'timestamptz' }) dueAt: Date;                    // hạn chót (UTC)

  @Column({ type: 'enum', enum: ['all', 'selected'], default: 'all' })
  assigneeMode: 'all' | 'selected';

  @Column({ type: 'jsonb', nullable: true }) selectedStudentIds?: number[];
  // chỉ đọc khi assigneeMode = 'selected' — danh sách id học sinh được giao

  @Column({ type: 'jsonb', default: [] }) attachments: { key: string; name: string; contentType: string }[];
  // đề bài: ảnh/PDF — key R2 từ bài 07

  @ManyToOne(() => UserEntity) @JoinColumn({ name: 'created_by' }) createdBy: UserEntity;
  @CreateDateColumn({ name: 'created_at' }) createdAt: Date;
}
```

> **Vì sao `selectedStudentIds` dùng jsonb thay vì bảng nối:** danh sách gán là dữ liệu "khai báo một lần" ít thay đổi — bảng nối (`assignment_assignees`) hợp khi cần query ngược ("HS có những bài nào") thường xuyên. Cả 2 cách hợp lệ; jsonb đơn giản hơn cho CRUD giao bài. **Ghi chú:** nếu bài 24 cần query ngược mạnh → migration đổi sang bảng nối, đây là ví dụ thương lượng schema.
> **Vì sao `attachments` jsonb mảng:** mô tả file (key + tên + loại) là dữ liệu phẳng nhỏ; không cần bảng riêng cho đề bài.

### Bước 2: API giao bài

```
POST   /api/assignments                tạo (tutor chủ lớp)
GET    /api/classes/[id]/assignments   danh sách bài của lớp (kèm thống kê nộp)
GET    /api/assignments/[id]           chi tiết (kèm quyền: tutor full, student bản của mình)
PATCH  /api/assignments/[id]           sửa/hủy (tutor)
PATCH  /api/assignments/[id]/deadline  gia hạn (cả lớp hoặc 1 HS)
```

Validate zod: `dueAt` phải tương lai; `selected` thì mảng id không rỗng và **mọi id phải thuộc lớp** (query `class_students` kiểm tra — không tin id client gửi).

> **Vì sao kiểm tra id thuộc lớp:** tránh giao bài cho học sinh lớp khác (lỗi dữ liệu + lộ thông tin giữa các lớp).

### Bước 3: Ai được thấy bài nào — hàm dùng chung

```typescript
async isAssignee(assignment, studentId: number): Promise<boolean> {
  if (assignment.assigneeMode === 'all') {
    const member = await this.classStudentRepo.findOneBy({ classId: assignment.classId, studentId });
    return !!member;
  }
  return (assignment.selectedStudentIds ?? []).includes(studentId);
}
```

Dùng ở API chi tiết + danh sách bài học sinh (bài 12) + thông báo (bài 14).

> **Vì sao viết thành hàm dùng chung:** luật "ai được giao" xuất hiện ở 3+ nơi; luật đổi một chỗ, mọi nơi đúng.

### Bước 4: UI giao bài (dialog trong tab Bài tập)

Form react-hook-form + zod:
- Tiêu đề, mô tả (textarea)
- Upload đề (multiple, dùng `use-upload` bài 07, hiện preview ảnh)
- Hạn chót (`datetime-local` → ISO)
- Chế độ giao: "Cả lớp" / "Chọn học sinh" (checkbox danh sách lớp — lấy từ API members)

> **Vì sao dialog trong tab:** luồng gia sư là "vừa dạy xong → vào lớp → giao ngay", giữ ngữ cảnh trang lớp.

### Bước 5: Tab Bài tập của lớp (đổ nội dung placeholder)

Danh sách bài: tiêu đề, hạn chót (đổi màu gần hạn), mode giao, **thống kê nhanh**: `Đã chấm x/y · z chưa nộp` — query đếm theo trạng thái (sẽ chi tiết ở bài 12). Click → trang chi tiết bài (dẫn tới màn hình chấm bài 13).

## ✅ Tiêu chí hoàn thành

- [ ] Giao bài cho cả lớp / cho subset học sinh
- [ ] Upload nhiều file đề thành công, preview hiển thị
- [ ] dueAt quá khứ → 400
- [ ] selectedStudentIds chứa id ngoài lớp → 400
- [ ] Tab Bài tập hiển thị danh sách + thống kê
- [ ] Commit: `lesson-11: assignments create + list`

## 📝 Tự kiểm tra

1. `assigneeMode` + `selectedStudentIds` — tại sao gom vào 1 bảng thay vì 2 bảng riêng?
2. Khi nào nên chuyển jsonb sang bảng nối? Đánh đổi là gì?
3. "Mọi id phải thuộc lớp" — nếu không kiểm tra, hậu quả cụ thể?
4. Hàm `isAssignee` dùng ở đâu trong bài này và bài sau?
