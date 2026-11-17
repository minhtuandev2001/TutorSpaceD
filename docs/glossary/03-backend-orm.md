# 03 — Backend: Server, TypeORM & kiến trúc code

> Mục lục: [API Route](#1) • [Middleware & Guard](#2) • [ORM](#3) • [Entity](#4) • [Repository](#5) • [QueryBuilder](#6) • [Migration](#7) • [Service Layer](#8) • [DTO & Validation (zod)](#9) • [BullMQ & Redis](#10)

## 1. API Route {#1}

Trong Next.js, backend nằm ngay trong dự án qua file `route.ts`:

```
app/api/classes/route.ts       → GET (danh sách), POST (tạo)
app/api/classes/[id]/route.ts  → PATCH (sửa), DELETE (xóa)
```

```typescript
export async function GET(request: Request) {
  const classes = await classRepository.findAll();
  return Response.json({ data: classes });
}
```

## 2. Middleware & Guard {#2}

**Middleware** = cổng chặn chạy **trước** API/trang, dùng để kiểm tra đăng nhập, redirect. **Guard** = kiểm tra quyền chi tiết hơn: "request này vai trò gì? có thuộc lớp này không?".

**Trong dự án:** middleware chặn toàn bộ `/tutor/*` chưa đăng nhập; API "tạo lớp" guard kiểm tra `role === 'tutor'`; API "xem bài tập" guard kiểm tra học sinh **có trong lớp đó** — không phải học sinh nào đăng nhập cũng xem được.

## 3. ORM {#3}

**ORM (Object-Relational Mapping)** = viết code JS để làm việc với database thay vì viết SQL tay. "Bảng users" thành "class UserEntity". Lợi ích: type-safe, không lặp SQL, dễ chuyển đổi DB.

**Dự án dùng TypeORM** — chốt từ đầu dự án, mọi truy vấn qua QueryBuilder (xem mục 6).

## 4. Entity {#4}

Class mô tả **1 bảng** trong database. Mỗi thuộc tính = 1 cột:

```typescript
@Entity({ name: 'classes' })
export class ClassEntity {
  @PrimaryGeneratedColumn() id: number;

  @Column({ type: 'varchar', length: 120 })
  name: string;

  @Column({ type: 'varchar', length: 20 })  // token màu: blue, cyan...
  color: string;

  @ManyToOne(() => UserEntity, (u) => u.classes)
  @JoinColumn({ name: 'teacher_id' })
  teacher: UserEntity;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;
}
```

**Trong dự án:** `ClassEntity` mô tả bảng `classes` — một lớp học với tên, màu, gia sư chủ nhiệm.

## 5. Repository {#5}

Cửa sổ làm việc với 1 entity: chứa mọi cách truy vấn của bảng đó. Quy ước dự án: **Repository là nơi duy nhất chứa QueryBuilder**.

```typescript
export class ClassRepository extends Repository<ClassEntity> {
  findMyClasses(teacherId: number) {
    return this.createQueryBuilder('c')
      .where('c.teacherId = :id', { id: teacherId })
      .orderBy('c.createdAt', 'DESC')
      .getMany();
  }
}
```

Vì sao tách riêng? Service chỉ nói *cần gì* ("lấy các lớp của thầy Minh"), Repository quyết định *lấy thế nào* (SQL nào, join gì). Đổi cách lấy không phải sửa service.

## 6. QueryBuilder {#6}

Cách TypeORM cho phép **xây câu SQL từng mảnh** — dùng cho truy vấn phức tạp mà API thường của ORM không làm nổi:

```typescript
this.createQueryBuilder('s')                    // 's' là bí danh bảng sessions
  .leftJoinAndMapOne('s.class', ClassEntity, 'c', 'c.id = s.classId')
  .where('s.startsAt BETWEEN :from AND :to', { from, to })   // :from là tham số — chống SQL injection
  .orderBy('s.startsAt', 'ASC')
  .getMany();
```

Khi nào dùng:
| Tình huống | Dùng |
|---|---|
| CRUD đơn giản | `repository.find()`, `findOne()` |
| JOIN nhiều bảng + điều kiện động + hàm SQL (COUNT, CASE WHEN) | **QueryBuilder** |

⚠️ Luôn truyền giá trị qua tham số `:param` — không nối chuỗi, tránh SQL injection.

## 7. Migration {#7}

**File mô tả thay đổi cấu trúc DB** (tạo bảng, thêm cột...), chạy có kiểm soát:

```bash
# sinh migration sau khi sửa entity
npx typeorm migration:generate -d src/db/data-source.ts src/db/migrations/AddClassTable

# áp dụng vào DB
npx typeorm migration:run
```

**Quy tắc dự án:** `synchronize: false` luôn. Thay đổi schema = sửa entity → generate migration → review → run. Migration giống "git cho database": biết được ai sửa gì, rollback được khi sai.

## 8. Service Layer {#8}

Lớp chứa **logic nghiệp vụ** giữa API route và repository: kiểm tra nghiệp vụ ("học sinh có thuộc lớp này không?", "hạn chót có ở tương lai không?"), gọi repository, phối hợp nhiều bước.

```
route.ts (nhận request) → service.ts (logic) → repository.ts (SQL)
```

Vì sao? API route chỉ lo HTTP; service test được logic không cần HTTP; repository chỉ lo SQL. Mỗi lớp một nhiệm vụ — sửa chỗ này không vỡ chỗ kia.

## 9. DTO & Validation (zod) {#9}

**DTO** = hình dạng dữ liệu nhận vào API. **zod** định nghĩa schema + tự kiểm tra:

```typescript
const createClassSchema = z.object({
  name: z.string().min(1).max(120),
  color: z.enum(['blue','cyan','amber','red','green','purple','pink','gray']),
});

const dto = createClassSchema.parse(await request.json());
// Sai → ném lỗi 400 với thông báo rõ ràng. Đúng → dto có type chuẩn cho TS.
```

**Vì sao:** client gửi gì cũng không tin — validation ở server là lớp phòng thủ duy nhất thực sự.

## 10. BullMQ & Redis {#10}

**Redis** = database trong RAM, nhanh, dùng làm cache và hàng đợi. **BullMQ** = thư viện hàng đợi trên Redis: gửi việc "chạy sau" và chạy định giờ.

**Trong dự án:**
- Job **lặp định giờ**: 30 phút trước mỗi buổi → gửi email nhắc lịch.
- Job **một lần**: khi gia sư kết thúc buổi → xếp việc export bảng trắng thành ảnh + upload R2.

Vì sao cần hàng đợi? Gửi email mất 1-2 giây — nếu gửi ngay trong request, người dùng phải chờ. Xếp vào queue, phản hồi ngay, việc chạy ngầm.
