# Bài 05 — Quản lý lớp học (CRUD + quan hệ N-N)

## 🎯 Mục tiêu

Gia sư tạo/sửa/xóa lớp; học sinh thấy các lớp mình tham gia; trang chi tiết lớp có 3 tab (Học sinh • Lịch • Bài tập).

## 🧠 Kiến thức cần nắm

- Quan hệ 1-N, N-N, bảng trung gian → [glossary/04-database.md](../../glossary/04-database.md#4-quan-hệ)
- Repository + QueryBuilder → [glossary/03-backend-orm.md](../../glossary/03-backend-orm.md)
- TanStack Query → [glossary/02-frontend.md](../../glossary/02-frontend.md#9-tanstack-query-react-query)

## 📋 Các bước

### Bước 1: Thiết kế entity

`class.entity.ts`:

```typescript
@Entity({ name: 'classes' })
export class ClassEntity {
  @PrimaryGeneratedColumn() id: number;
  @Column({ type: 'varchar', length: 120 }) name: string;
  @Column({ type: 'varchar', length: 80, nullable: true }) subject?: string;   // môn
  @Column({ type: 'varchar', length: 40, nullable: true }) grade?: string;     // khối: "9", "10"
  @Column({ type: 'varchar', length: 500, nullable: true }) description?: string;
  @Column({ type: 'varchar', length: 20, default: 'blue' }) color: string;     // token màu
  @Column({ type: 'enum', enum: ['active', 'paused', 'ended'], default: 'active' })
  status: 'active' | 'paused' | 'ended';

  @ManyToOne(() => UserEntity, { nullable: false })
  @JoinColumn({ name: 'teacher_id' })
  teacher: UserEntity;

  @CreateDateColumn({ name: 'created_at' }) createdAt: Date;
}
```

`class-student.entity.ts` — **bảng trung gian** của quan hệ N-N:

```typescript
@Entity({ name: 'class_students' })
@Unique(['classId', 'studentId'])                       // không join trùng 2 lần
export class ClassStudentEntity {
  @PrimaryGeneratedColumn() id: number;
  @Column({ name: 'class_id' }) classId: number;
  @Column({ name: 'student_id' }) studentId: number;
  @CreateDateColumn({ name: 'joined_at' }) joinedAt: Date;
}
```

> **Vì sao bảng trung gian tách riêng:** quan hệ lớp–học sinh là N-N (1 lớp nhiều HS, 1 HS nhiều lớp) — SQL chuẩn phải có bảng nối. Tách entity riêng còn cho phép **thêm cột sau** (ngày join — đã làm, trạng thái, nhóm...).
> **Vì sao `color` là token chứ không mã hex:** giới hạn 8 màu → giao diện đồng nhất, tránh màu khó đọc; token dễ đổi theme sau.

Generate + run migration như bài 02.

### Bước 2: Repository — nơi duy nhất chứa QueryBuilder

`src/modules/classes/class.repository.ts`:

```typescript
export class ClassRepository extends Repository<ClassEntity> {
  // Lịch sử lớp cho gia sư: kèm số học sinh — ví dụ QueryBuilder + GROUP BY
  findTutorClasses(teacherId: number) {
    return this.createQueryBuilder('c')
      .leftJoin(ClassStudentEntity, 'cs', 'cs.classId = c.id')
      .where('c.teacherId = :tid', { tid: teacherId })
      .andWhere('c.status != :ended', { ended: 'ended' })
      .select(['c.id', 'c.name', 'c.color', 'c.subject', 'c.status'])
      .addSelect('COUNT(cs.id)', 'studentCount')
      .groupBy('c.id')
      .orderBy('c.createdAt', 'DESC')
      .getRawAndEntities();
  }

  // Các lớp học sinh đang học
  findStudentClasses(studentId: number) {
    return this.createQueryBuilder('c')
      .innerJoin(ClassStudentEntity, 'cs', 'cs.classId = c.id')
      .where('cs.studentId = :sid', { sid: studentId })
      .andWhere('c.status = :active', { active: 'active' })
      .orderBy('c.createdAt', 'DESC')
      .getMany();
  }
}
```

> **Vì sao `COUNT(cs.id)` + GROUP BY:** dashboard gia sư cần "12 học sinh" ngay trên thẻ lớp — tính trong 1 câu query thay vì hỏi từng lớp một (N+1 query — lỗi hiệu năng kinh điển).

### Bước 3: Service — logic nghiệp vụ

`class.service.ts`:

```typescript
create(dto: CreateClassDto, teacherId: number) { /* repo.create + save */ }

async update(id: number, dto: UpdateClassDto, user: SessionUser) {
  const cls = await this.repo.findOneByOrFail({ id });
  if (cls.teacher.id !== user.userId) throw new ForbiddenError();  // chỉ gia sư chủ lớp
  Object.assign(cls, dto);
  return this.repo.save(cls);
}

async endClass(id: number, user: SessionUser) { /* set status='ended' */ }
```

> **Vì sao kiểm tra `teacher.id` trong service:** "chỉ gia sư chủ lớp được sửa" là luật nghiệp vụ, không phải luật vai trò chung — guard role chỉ trả lời "gia sư hay học sinh", service trả lời "gia sư **của lớp này** không". Phân biệt này quan trọng, lặp lại ở mọi phân hệ sau.

### Bước 4: API routes

```
POST   /api/classes              tạo lớp          (tutor)
GET    /api/classes              danh sách (tự nhận role trả đúng tập)
PATCH  /api/classes/[id]         sửa             (tutor chủ lớp)
PATCH  /api/classes/[id]/status  tạm dừng/kết thúc (tutor chủ lớp)
GET    /api/classes/[id]         chi tiết (thành viên)
```

Tất cả bọc `withApi()`, validate zod (`name` bắt buộc, `color` thuộc 8 token).

> **Vì sao `GET /api/classes` tự nhận role:** một endpoint trả đúng dữ liệu cho 2 vai trò — client không phải biết gọi endpoint nào cho role nào.

### Bước 5: UI

- `/tutor/classes` — grid thẻ lớp (shadcn `Card`), mỗi thẻ: chấm màu, tên, số HS, trạng thái; nút "Tạo lớp" mở `Dialog`.
- `/tutor/classes/[id]` — trang chi tiết với 3 tab (shadcn `Tabs`): Học sinh (bảng danh sách), Lịch (placeholder — bài 10), Bài tập (placeholder — bài 11).
- `/student` — grid lớp của học sinh.

Dữ liệu client dùng TanStack Query; sau khi tạo lớp → `invalidateQueries(['classes'])`.

> **Vì sao placeholder 2 tab:** dựng khung điều hướng ngay để các bài sau chỉ cần "đổ nội dung" vào tab có sẵn — cấu trúc UI không phải sửa lại.

## ✅ Tiêu chí hoàn thành

- [ ] Gia sư tạo/sửa/tạm dừng/kết thúc lớp được
- [ ] Thẻ lớp hiện đúng số học sinh (tính bằng GROUP BY, không N+1)
- [ ] Học sinh sửa lớp của gia sư → 403
- [ ] Trang chi tiết lớp có 3 tab điều hướng
- [ ] Migration `class_students` có unique constraint
- [ ] Commit: `lesson-05: class management + N-N relations`

## 📝 Tự kiểm tra

1. Vì sao cần bảng `class_students`? Nếu không có, thiết kế sai chỗ nào?
2. N+1 query là gì? Truy vấn trên tránh nó bằng cách nào?
3. `withApi` bắt được lỗi nào thành 400? 403 sinh từ đâu?
4. Lớp "ended" hiển thị gì với gia sư? Với học sinh? (Thiết kế: ẩn, nhưng dữ liệu vẫn giữ — vì sao?)
