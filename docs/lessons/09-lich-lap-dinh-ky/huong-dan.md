# Bài 09 — Lịch lặp định kỳ (rrule + override buổi lẻ)

> ⚠️ Bài khó nhất giai đoạn B. Làm chậm, làm kỹ — phần lớn app lịch sai vì chỗ này.

## 🎯 Mục tiêu

Gia sư tạo một lần "T3–T5 19:00–20:30, lặp 8 tuần" → hệ sinh đủ chuỗi buổi; dời/nghỉ **một buổi** không phá các buổi còn lại; có thể sửa luật cho chuỗi còn lại.

## 🧠 Kiến thức cần nắm

- Transaction → [glossary/04-database.md](../../glossary/04-database.md#7-transaction)
- Entity sessions (bài 08) + `ruleId`
- Thuật toán rrule — giải thích trong bài

## 📋 Các bước

### Bước 1: Entity recurring_rules

```typescript
@Entity({ name: 'recurring_rules' })
export class RecurringRuleEntity {
  @PrimaryGeneratedColumn() id: number;
  @Column({ name: 'class_id' }) classId: number;

  @Column({ type: 'jsonb' })
  pattern: { weekdays: number[];   // 0=CN..6=T7, [2,4] = T3 & T5
             startTime: string;    // "19:00" theo múi giờ lớp
             durationMin: number;  // 90
             timezone: string; };  // "Asia/Ho_Chi_Minh"

  @Column({ name: 'starts_from', type: 'timestamptz' }) startsFrom: Date; // ngày bắt đầu
  @Column({ name: 'ends_at', type: 'timestamptz', nullable: true }) endsAt?: Date;
  @Column({ name: 'occurrences', type: 'integer', nullable: true }) occurrences?: number; // hoặc N buổi

  @Column({ name: 'is_active', type: 'boolean', default: true }) isActive: boolean;
}
```

> **Vì sao lưu pattern dạng jsonb:** luật lặp "hàng tuần theo thứ" là dạng rrule RRULE chuẩn; sau này mở rộng (cách 2 tuần, lặp tháng) chỉ thêm field, không đổi bảng.
> **Vì sao giữ `timezone`:** buổi "19:00" là khái niệm **giờ địa phương** của người dạy — sinh buổi phải nhân múi giờ lúc tính, không nhân lúc lưu.

### Bước 2: Cài rrule.js + hàm sinh buổi

```bash
npm i rrule
```

```typescript
import { RRule, RRuleSet, rrulestr } from 'rrule';

export function expandRule(rule: RecurringRuleEntity, from: Date, to: Date): Date[] {
  const set = rrulestr(buildRRuleString(rule.pattern), { forceset: true });
  // 1. tính chuỗi các mốc giờ địa phương (19:00 ngày X)
  // 2. quy mỗi mốc về UTC (Date.UTC với offset múi giờ pattern.timezone)
  // 3. trả các mốc nằm trong [from, to]
}
```

> **Vì sao quy 2 bước (mốc địa phương → UTC):** rrule.js làm việc với Date JS theo UTC nội bộ; nếu cứ cộng trừ giờ tay sẽ sai khi đi qua ngày chuyển múi giờ (hạ canh). Cộng trừ dùng thư viện date với timezone (date-fns-tz), không cộng số giây tay.

### Bước 3: Sinh chuỗi buổi trong transaction

```typescript
createRuleAndExpand(dto, user) {
  return this.dataSource.transaction(async (em) => {
    const rule = await em.save(RecurringRuleEntity, ruleRow);

    // sinh các buổi CHƯA TỒN TẠI: buổi có (ruleId, startsAt) trùng thì bỏ qua
    const marks = expandRule(rule, rule.startsFrom, horizon);   // horizon = từ nay + 8 tuần
    const existing = await em.find(SessionEntity, { where: { ruleId: rule.id } });
    const existingKeys = new Set(existing.map(s => s.startsAt.toISOString()));
    const toCreate = marks.filter(m => !existingKeys.has(m.toISOString()))
      .map(m => ({ classId: rule.classId, ruleId: rule.id, startsAt: m,
                   endsAt: addMinutes(m, rule.pattern.durationMin), status: 'upcoming' }));
    await em.insert(SessionEntity, toCreate);
    return rule;
  });
}
```

> **Vì sao skip buổi trùng:** người dùng có thể bấm "sinh lại chuỗi" sau khi sửa vài buổi — logic **idempotent** (chạy 2 lần ra 1 kết quả) tránh nhân đôi buổi.
> **Vì sao chỉ sinh 8 tuần chứ không sinh vô hạn:** hàng nghìn dòng session rác trong DB + lịch UI khó vẽ. Job định kỳ (bài 15) sẽ "đuôi dài" sinh thêm khi chuỗi sắp cạn.

### Bước 4: Override buổi lẻ — mô hình then chốt

Không xóa/đổi buổi gốc. Buổi trong chuỗi **được sửa trực tiếp** (dời giờ, nghỉ với `note`), chuỗi gốc không biết chuyện:

```
Chuỗi: T3 19:00 (upcoming)  T5 19:00 (RESTED, note "HS xin nghỉ")  T3 tuần sau 19:00 (upcoming) ...
                        ↑ buổi này tự do khác chuỗi, ruleId vẫn giữ
```

Quyết định này cho phép: sửa 1 buổi tự do; **tạo lại chuỗi** (bước 3) vẫn giữ mọi chỉnh lẻ vì key `startsAt` không trùng.

> **Vì sao không lưu override riêng:** mô hình "buổi là dòng thật, chuỗi chỉ là máy sinh" đơn giản hơn mô hình "chuỗi + bảng override" nhiều, và mọi query lịch (bài 08) không cần biết luật lặp.

### Bước 5: Nghỉ buổi + bù học

- `PATCH /api/sessions/[id]/status` thêm hành động `rest` (yêu cầu `note` bắt buộc: lý do) và `reschedule` (dời giờ, note ghi "bù vào ngày X").
- Tuần sau thêm trạng thái `made_up` khi có buổi bù trỏ `note: "bù cho buổi #id"`.

> **Vì sao note bắt buộc:** nghiệp vụ dạy kèm, lý do nghỉ là bằng chứng với phụ huynh/học sinh — sau này báo cáo chuyên cần (backlog) đọc note.

### Bước 6: Unit test cho expandRule

Vitest, các case bắt buộc: lặp T3-T5 8 tuần ra đúng 16 buổi đúng giờ UTC; dời DST/múi giờ không lệch; từ 30/11 lặp hàng tuần vẫn đúng qua tháng 31 ngày.

> **Vì sao test trước UI:** đây là hàm thuần logic dễ test nhất toàn dự án — test xanh cho tự tin khi mọi thứ sau dè vào nó.

## ✅ Tiêu chí hoàn thành

- [ ] Tạo luật "T3–T5 19:00, 8 tuần" → 16 buổi đúng giờ
- [ ] Nghỉ 1 buổi + note → chuỗi còn lại nguyên vẹn
- [ ] Dời 1 buổi 30 phút → lịch tổng hiển thị đúng
- [ ] "Sinh lại chuỗi" không nhân đôi buổi (idempotent)
- [ ] Unit test expandRule xanh
- [ ] Commit: `lesson-09: recurring rules + override sessions`

## 📝 Tự kiểm tra

1. Mô hình "chuỗi là máy sinh, buổi là dòng thật" khác gì "chuỗi + bảng override"? Ưu nhược?
2. Vì sao sinh 8 tuần thay vì sinh vô hạn?
3. `idempotent` nghĩa là gì, giúp gì khi người dùng bấm lại nút?
4. Sai chỗ nào nếu sinh buổi bằng "cộng 7 ngày × 8" trên Date JS?
