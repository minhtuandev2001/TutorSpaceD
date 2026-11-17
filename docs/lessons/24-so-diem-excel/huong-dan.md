# Bài 24 — Sổ điểm, xuất Excel & ghi chú học sinh

## 🎯 Mục tiêu

Sổ điểm tổng hợp theo lớp/học sinh; xuất file Excel; biểu đồ tiến độ điểm theo thời gian; ghi chú riêng gia sư cho từng học sinh.

## 🧠 Kiến thức cần nắm

- QueryBuilder aggregate → [glossary/03-backend-orm.md](../../glossary/03-backend-orm.md#6-querybuilder)
- ExcelJS (thư viện xuất xlsx)

## 📋 Các bước

### Bước 1: API sổ điểm

```
GET /api/classes/[id]/gradebook      bảng điểm cả lớp
GET /api/students/[id]/grades        điểm 1 HS (gộp các lớp HS đó học)
```

Sổ điểm = query lớn: rows = học sinh, columns = các assignment của lớp, cell = điểm:

```typescript
const grades = await this.dataSource.createQueryBuilder()
  .select('cs.student_id', 'studentId')
  .addSelect('a.id', 'assignmentId')
  .addSelect('COALESCE(sub.score, qa.score)', 'score')      // gộp cả upload lẫn quiz
  .from(ClassStudentEntity, 'cs')
  .leftJoin(AssignmentEntity, 'a', 'a.class_id = cs.class_id')
  .leftJoin(SubmissionEntity, 'sub', 'sub.assignment_id = a.id AND sub.student_id = cs.student_id')
  .leftJoin(QuizAttemptEntity, 'qa', 'qa.assignment_id = a.id AND qa.student_id = cs.student_id')
  .where('cs.class_id = :cid', { cid })
  .getRawMany();
// chuyển sang ma trận HS × bài ở service
```

> **Vì sao COALESCE(sub.score, qa.score):** điểm có thể đến từ bài nộp upload (bài 13) hoặc quiz tự chấm (bài 23) — sổ điểm phải gộp được cả 2 nguồn, ưu tiên bản nào có điểm.

### Bước 2: Xuất Excel (ExcelJS)

```bash
npm i exceljs
```

```typescript
const wb = new ExcelJS.Workbook();
const ws = wb.addWorksheet('Sổ điểm');
ws.columns = [
  { header: 'Học sinh', key: 'name', width: 24 },
  ...assignments.map(a => ({ header: a.title, key: `a${a.id}`, width: 16 })),
];
rows.forEach(r => ws.addRow(r));
// format: header bold + nền tối, điểm đỏ nếu < 5
const buffer = await wb.xlsx.writeBuffer();
```

API trả file: `Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` + `Content-Disposition: attachment; filename="so-diem-toan9a.xlsx"`.

> **Vì sao ExcelJS:** quản lý cha/mẹ/family phụ huynh hay hỏi file điểm; ExcelJS chạy server-side không cần browser. Dùng số chuẩn + header tiếng Việt đúng.

### Bước 3: Biểu đồ tiến độ (client)

Trang học sinh + trang lớp: chart line — trục X = thời gian nộp/chấm, trục Y = điểm. Dùng Recharts (nhẹ, dễ style):

```tsx
<LineChart data={points}>
  <Line dataKey="score" stroke="var(--blue)" dot />
</LineChart>
```

> **Vì sao Recharts:** component React thuần, style theo Tailwind var; không cần D3 phức tạp cho biểu đồ đơn.

### Bước 4: Ghi chú riêng gia sư

Cột `tutorNotes` text trên `class_students` (migration): chỉ gia sư chủ lớp đọc/sửa (`PATCH /api/classes/[id]/students/[studentId]/notes`).

> **Vì sao nằm trên bảng trung gian:** ghi chú là **quan hệ giữa gia sư và học sinh trong lớp đó** — An lớp Toán và lớp Anh có thể có note khác nhau.

### Bước 5: UI

- Tab mới "Sổ điểm" trong trang lớp: bảng điểm + nút "Xuất Excel"
- Trang HS: biểu đồ tiến độ + danh sách điểm theo lớp
- Trang thành viên: nút mở dialog ghi chú từng HS

## ✅ Tiêu chí hoàn thành

- [ ] Sổ điểm đúng ma trận HS × bài, gộp quiz + upload
- [ ] Xuất Excel mở được, format đẹp, tên file đúng
- [ ] Biểu đồ tiến độ hiển thị theo thời gian
- [ ] Ghi chú riêng chỉ tutor chủ lớp đọc được
- [ ] Commit: `lesson-24: gradebook + excel export + progress chart`

## 📝 Tự kiểm tra

1. COALESCE gộp 2 nguồn điểm — nếu HS có cả 2, ưu tiên gì? Nên chọn thế nào trong service?
2. Xuất file qua API — Content-Type/Content-Disposition nhiệm vụ gì?
3. Ghi chú HS nằm ở bảng nào? Vì sao không phải bảng users?
4. Ma trận điểm với lớp 30 HS × 40 bài — query trên có nguy cơ hiệu năng gì? Index nào cần?
