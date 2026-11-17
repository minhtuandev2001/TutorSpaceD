# Bài 23 — Trắc nghiệm tự chấm

## 🎯 Mục tiêu

Gia sư tạo bài trắc nghiệm (câu hỏi + đáp án, thời gian làm), học sinh làm online → hệ thống chấm tự động, xem đáp án đúng/sai.

## 🧠 Kiến thức cần nắm

- jsonb lưu đề → [glossary/04-database.md](../../glossary/04-database.md#2-kiểu-dữ-liệu)
- Assignment entity (bài 11) — trắc nghiệm là **một dạng bài tập**

## 📋 Các bước

### Bước 1: Mở rộng assignment cho trắc nghiệm

Thêm cột `kind` trên assignments: `upload | text | quiz` (migration). Quiz lưu đề trong jsonb:

```typescript
// assignments.quizData khi kind = 'quiz'
{
  timeLimitMin: 15,                    // null = không giới hạn
  shuffleQuestions: true,
  questions: [
    {
      id: 'q1',
      text: 'Giải hệ: 2x + 3y = 6 và x − y = 1, x = ?',
      options: ['1/5', '9/5', '7/3', '4/3'],
      correctIndex: 1,
      points: 1,                        // điểm mỗi câu — tổng = 10
    },
    // ...
  ],
}
```

> **Vì sao jsonb cho đề:** cấu trúc câu hỏi linh hoạt (đa chọn, điền khuyết sau này); đề "khai báo một lần, làm nhiều" — không cần 3 bảng (questions/options/answers) cho giai đoạn này. Khi cần ngân hàng đề (backlog) sẽ tách bảng.

### Bước 2: Entity quiz_attempts

```typescript
@Entity({ name: 'quiz_attempts' })
export class QuizAttemptEntity {
  @PrimaryGeneratedColumn() id: number;
  @Column({ name: 'assignment_id' }) assignmentId: number;
  @Column({ name: 'student_id' }) studentId: number;
  @Column({ type: 'jsonb' }) answers: { questionId: string; selectedIndex: number }[];
  @Column({ type: 'numeric', precision: 4, scale: 1 }) score: number;
  @Column({ name: 'submitted_at', type: 'timestamptz' }) submittedAt: Date;
  @Unique(['assignmentId', 'studentId'])     // 1 lần làm (ghi đè nếu làm lại — cấu hình được)
}
```

> **Vì sao bảng riêng, không nhét vào submissions:** vòng đời khác — trắc nghiệm chấm ngay khi nộp, không qua tay gia sư. Dùng `kind='quiz'` để bảng trạng thái (bài 12) hiển thị khác ("Đã làm 8/10").

### Bước 3: API làm trắc nghiệm

```
GET  /api/assignments/[id]/quiz          HS bắt đầu → trả đề (Không kèm correctIndex!)
POST /api/assignments/[id]/quiz-submit   nộp → chấm ngay → trả điểm + đáp án đúng
```

Chấm tự động:

```typescript
const total = quizData.questions.reduce((s, q) => s + q.points, 0);
const got = quizData.questions.reduce((s, q) => {
  const a = answers.find(x => x.questionId === q.id);
  return s + (a && a.selectedIndex === q.correctIndex ? q.points : 0);
}, 0);
const score = Math.round((got / total) * 100) / 10 * 10 / 10;   // quy thang 10
```

> **Vì sao API bắt đầu không trả đáp án:** đáp án phải ở server lúc chấm; gửi trước = học sinh mở DevTools xem đáp án — cheat kinh điển.
> **Vì sao chấm ở server:** cùng lý do — client tự chấm là tự ghi điểm.

### Bước 4: Shuffle ổn định

`shuffleQuestions: true` → trộn câu hỏi **theo seed = studentId + assignmentId** (random có hạt giống): mỗi HS thấy thứ tự khác nhau, nhưng cùng HS reload vẫn thứ tự cũ (không trộn lại mỗi lần refresh).

> **Vì sao seeded shuffle:** trộn "mỗi lần mỗi khác" khiến HS refresh = đổi câu — nhiễu cho việc làm bài; seed cố định giữ trải nghiệm ổn định mà vẫn khác nhau giữa các HS.

### Bước 5: UI làm bài

- Trang làm bài: 1 câu/trang hoặc danh sách, đồng hồ đếm ngược (timeLimitMin) — hết giờ tự nộp
- Bật/tắt timer client + server kiểm `submittedAt - startedAt ≤ timeLimit + 5s` (nới 5s cho độ trễ mạng)
- Kết quả ngay: điểm + từng câu đúng/sai (đáp án đúng mở sau khi nộp)

> **Vì sao server kiểm thời gian nộp:** đồng hồ client chỉnh được; kiểm khoảng cách thời gian là rào cuối chống "nộp sau khi hết giờ 10 phút".

### Bước 6: Gắn với bảng theo dõi

Board bài 12: submission của quiz = quiz_attempts (LEFT JOIN cả 2 bảng, CASE WHEN ưu tiên attempt). Trạng thái HS: "Chưa làm / Đã làm (8.5)".

## ✅ Tiêu chí hoàn thành

- [ ] Tạo quiz 10 câu, HS làm → điểm tự tính đúng
- [ ] DevTools không thấy đáp án trước khi nộp
- [ ] Hết giờ → tự nộp, server từ chối nộp trễ quá 5s
- [ ] Shuffle khác nhau giữa HS, ổn định khi reload
- [ ] Board hiển thị trạng thái quiz đúng
- [ ] Commit: `lesson-23: quiz assignments + auto grading`

## 📝 Tự kiểm tra

1. Vì sao đề quiz để jsonb, bài upload lại dùng files/jsonb khác cấu trúc?
2. Tấn công "xem đáp án qua DevTools" chặn ở tầng nào?
3. Seeded shuffle — chống vấn đề nào của shuffle ngẫu nhiên thuần?
4. Quiz_attempts tách khỏi submissions — rồi board bài 12 gộp thế nào?
