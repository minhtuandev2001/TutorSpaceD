# Bài 06 — Mời học sinh bằng mã tham gia

## 🎯 Mục tiêu

Gia sư lấy được mã 6 ký tự + link mời; học sinh nhập mã → vào lớp; mã hết hạn được; chống vào lớp 2 lần.

## 🧠 Kiến thức cần nắm

- Transaction → [glossary/04-database.md](../../glossary/04-database.md#7-transaction)
- Service layer → [glossary/03-backend-orm.md](../../glossary/03-backend-orm.md)

## 📋 Các bước

### Bước 1: Thiết kế lưu mã — nơi nào?

2 phương án:
| Phương án | Ưu | Nhược |
|---|---|---|
| Lưu mã trực tiếp trên lớp (`classes.join_code`) | Đơn giản | Không hết hạn được, không thu hồi riêng |
| **Bảng `join_codes` riêng** | Mã có hạn hạn + thu hồi + thống kê lượt dùng | Thêm 1 bảng |

**Chọn bảng riêng** — sản phẩm thật cần "tạo lại mã khi lộ":

```typescript
@Entity({ name: 'join_codes' })
export class JoinCodeEntity {
  @PrimaryGeneratedColumn() id: number;
  @Column({ name: 'class_id' }) classId: number;
  @Column({ type: 'varchar', length: 6, unique: true }) code: string;
  @Column({ type: 'timestamptz', nullable: true }) expiresAt?: Date;   // null = không hết hạn
  @Column({ name: 'is_active', type: 'boolean', default: true }) isActive: boolean;
  @Column({ name: 'max_uses', type: 'integer', nullable: true }) maxUses?: number;
  @Column({ name: 'used_count', type: 'integer', default: 0 }) usedCount: number;
  @CreateDateColumn({ name: 'created_at' }) createdAt: Date;
}
```

> **Vì sao mã 6 ký tự:** đủ dễ đọc/gõ/ghi nhớ, đủ khó đoán. **Ký tự loại bỏ:** bỏ `0/O`, `1/I` — người dùng không phân biệt được khi đọc mã qua điện thoại. Sinh từ bộ 32 chữ số an toàn: `2-9A-Z` trừ ký tự dễ nhầm.

### Bước 2: Sinh mã an toàn

`src/lib/join-code.ts`:

```typescript
const ALPHABET = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789'; // bỏ I, O, 0, 1
export function generateJoinCode(): string {
  const bytes = randomBytes(6);
  return Array.from(bytes, (b) => ALPHABET[b % ALPHABET.length]).join('');
}
```

> **Vì sao `randomBytes` (crypto) thay vì `Math.random`:** Math.random **không an toàn mật mã** — kẻ xấu đoán được chuỗi; randomBytes sinh từ entropy hệ thống.

### Bước 3: API tạo/đọc mã

```
POST /api/classes/[id]/join-code        tạo mã mới (tutor chủ lớp)  → vô hiệu hóa mã cũ
GET  /api/classes/[id]/join-code        đọc mã hiện tại (tutor chủ lớp)
POST /api/join                          học sinh nhập mã → vào lớp (student)
```

### Bước 4: Logic join — chỗ khó nhất, viết cẩn thận

```typescript
async joinWithCode(code: string, studentId: number) {
  return this.dataSource.transaction(async (em) => {          // TRANSACTION!
    const jc = await em.findOne(JoinCodeEntity, {
      where: { code, isActive: true },
    });
    if (!jc) throw new NotFoundError('Mã không hợp lệ');
    if (jc.expiresAt && jc.expiresAt < new Date()) throw new BadRequestError('Mã đã hết hạn');
    if (jc.maxUses && jc.usedCount >= jc.maxUses) throw new BadRequestError('Mã đã hết lượt');

    const already = await em.findOne(ClassStudentEntity, {
      where: { classId: jc.classId, studentId },
    });
    if (already) return { classId: jc.classId, already: true };  // vào lại → không lỗi

    await em.insert(ClassStudentEntity, { classId: jc.classId, studentId });
    await em.update(JoinCodeEntity, jc.id, { usedCount: jc.usedCount + 1 });

    return { classId: jc.classId, already: false };
  });
}
```

> **Vì sao từng điều kiện:**
> - **Transaction:** kiểm tra + insert + tăng đếm phải là 1 khối — 2 học sinh cùng lúc dùng mã hết lượt cuối (race condition) không thể "lọt" cả hai.
> - **`already: true` không phải lỗi:** học sinh join lại (mất link, quên đã vào) phải được dẫn thẳng vào lớp — thông báo lỗi sẽ gây khó chịu vô lý.
> - `usedCount` cập nhật bằng giá trị mới trong transaction — tránh đếm sai khi nhiều request song song.

### Bước 5: UI

- Gia sư: trong trang chi tiết lớp → phần "Mời học sinh": mã hiển thị to + nút copy link (`/join?code=X7K2PQ`) + nút "Tạo mã mới".
- Học sinh: `/join` — input mã (tự in hoa, bỏ khoảng trắng), nút "Vào lớp" → thành công → redirect về lớp.

> **Vì sao tự normalize input:** người dùng gõ "x7k2pq" hoặc " X7K2PQ " — chuẩn hóa ở client + server (uppercase + trim) để mã luôn khớp, không tạo ticket "vào không được".

## ✅ Tiêu chí hoàn thành

- [ ] Tạo lớp → có mã, dùng được ngay
- [ ] Học sinh vào được qua mã VÀ qua link
- [ ] Join lại lần 2 → thành công, không tạo dòng trùng
- [ ] Tạo mã mới → mã cũ thất bại
- [ ] Hết hạn/lượt → lỗi 400 rõ ràng
- [ ] Commit: `lesson-06: join codes + student join flow`

## 📝 Tự kiểm tra

1. Vì sao bảng `join_codes` tách khỏi `classes`?
2. Race condition ở đây là gì? Transaction chặn nó thế nào?
3. Vì sao loại I, O, 0, 1 khỏi bộ ký tự?
4. "Join lần 2 không lỗi" — quyết định UX này cứu trường hợp nào thực tế?
