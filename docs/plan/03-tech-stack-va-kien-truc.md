# 03 — Tech stack & kiến trúc

## 1. Sơ đồ kiến trúc tổng thể

```
┌─────────────────────────────────────────────────────────┐
│                    NGƯỜI DÙNG (Trình duyệt)               │
└──────────────┬──────────────────────────────┬───────────┘
               │ HTTPS                        │ WebRTC + WebSocket
┌──────────────▼──────────────┐   ┌───────────▼───────────┐
│   APP LAYER (Next.js)       │   │   MEDIA LAYER          │
│   • UI (React components)   │   │   • LiveKit server     │
│   • API Routes              │◄──┼──►  (SFU - video/audio)│
│   • Auth.js (đăng nhập)     │   │   • Socket.io          │
│   • TypeORM + Repository    │   │     (chat, bảng trắng, │
│   • BullMQ (hàng đợi)       │   │      trạng thái phòng) │
└──────┬───────────┬──────────┘   └───────────┬───────────┘
       │           │                          │
┌──────▼─────┐ ┌───▼────────┐        ┌────────▼──────────┐
│ PostgreSQL │ │ Cloudflare │        │ LiveKit Egress    │
│ (dữ liệu)  │ │ R2 (file)  │        │ (ghi hình → MP4   │
│ + Redis    │ │            │        │  → lưu lên R2)    │
└────────────┘ └────────────┘        └───────────────────┘
```

**Vì sao tách 2 lớp?** Phần "app" (lớp, lịch, bài tập) là logic nghiệp vụ thông thường — chạy tốt trên Next.js. Phần "media" (video, bảng trắng) cần server chuyên biệt giữ kết nối realtime liên tục — tách riêng để app không bị nặng và có thể scale độc lập.

## 2. Bảng công nghệ đầy đủ

### Frontend
| Công nghệ | Dùng để | Lý do chọn |
|---|---|---|
| **Next.js 15 (App Router) + TypeScript** | Framework chính | Fullstack một ngôn ngữ, chuẩn ngành, tài liệu nhiều |
| **Tailwind CSS v4** | Style | Viết CSS nhanh, không phải đặt tên class |
| **shadcn/ui** | Bộ component (button, dialog, form...) | Đúng phong cách Vercel — tối giản, nền tối, viền mảnh |
| **Geist + Geist Mono** | Font chữ | Đẹp, hỗ trợ tiếng Việt, đúng style Vercel |
| **Lucide React** | Icon | Miễn phí, đầy đủ |
| **Zustand** | State client (phòng học: mic/cam/tay giơ) | Nhẹ hơn Redux |
| **TanStack Query** | Lấy dữ liệu server (lớp, lịch, bài tập) | Cache + refetch tự động |
| **react-hook-form + zod** | Form (tạo lớp, giao bài) | Validation dùng chung client + server |
| **@schedule-x** hoặc FullCalendar | UI lịch tuần/tháng | Đừng tự vẽ calendar |
| **date-fns + rrule.js** | Xử lý ngày giờ + chuỗi lặp | rrule.js là chuẩn xử lý lịch lặp |
| **tldraw** | Bảng trắng | Open source, sync multiplayer có sẵn |

### Backend
| Công nghệ | Dùng để | Lý do chọn |
|---|---|---|
| **Node.js + TypeScript** | Runtime | Đồng bộ với frontend |
| **Next.js Route Handlers** | API (bắt đầu) | MVP không cần server riêng |
| **NestJS** | API (tách sau khi logic phình to) | Module hóa gọn gàng |
| **TypeORM 0.3 + QueryBuilder** | ORM & truy vấn | **Đã chốt**: QueryBuilder kiểm soát SQL chi tiết, Repository pattern |
| **PostgreSQL 16** | Database | Dữ liệu rất quan hệ (lớp–buổi–bài tập) |
| **Redis + BullMQ** | Cache + hàng đợi | Gửi email nhắc lịch, sinh chuỗi buổi, xử lý ghi hình |
| **Auth.js (NextAuth v5)** | Đăng nhập | Google OAuth + email, session JWT |
| **Socket.io** | Realtime app-layer | Chat, thông báo, trạng thái phòng |

### Media (phòng học)
| Công nghệ | Dùng để | Lý do chọn |
|---|---|---|
| **LiveKit** | Truyền video/audio (SFU) | Open source, WebRTC chuẩn, có SDK React, tự host được |
| **LiveKit Egress** | Ghi hình phòng → MP4 | Tự động, upload thẳng lên R2 |

### Hạ tầng
| Công nghệ | Dùng để | Lý do chọn |
|---|---|---|
| **Cloudflare R2** | Lưu file (ảnh bài, PDF, MP4) | Miễn phí chi phí băng thông đọc (egress) |
| **Vercel** | Host Next.js | Deploy Next.js tự nhiên nhất |
| **VPS (Singapore) + Docker** | Host LiveKit + Redis | Đặt gần VN để giảm độ trễ |
| **GitHub Actions** | CI/CD | Tự động test + deploy |
| **Sentry** | Bắt lỗi production | Biết app lỗi ở đâu trước khi người dùng báo |

## 3. Quy ước code với TypeORM (quan trọng — áp dụng từ bài 02)

```
Controller (route)  →  Service (logic nghiệp vụ)  →  Repository (mọi QueryBuilder)
```

- **100% QueryBuilder nằm trong Repository.** Service không bao giờ gọi `dataSource.query()` trực tiếp.
- Dùng `synchronize: false` — mọi thay đổi schema làm bằng **migration**, không bao giờ để TypeORM tự sửa bảng.
- Tên bảng: `snake_case` số nhiều (`users`, `classes`, `class_students`, `sessions`...).
- Tên cột: `snake_case` (`created_at`, `full_name`...).

Ví dụ chuẩn một truy vấn QueryBuilder (bảng trạng thái bài tập cả lớp):

```typescript
// assignment.repository.ts
getSubmissionBoard(classId: number, assignmentId: number) {
  return this.dataSource
    .createQueryBuilder()
    .select('u.fullName', 'studentName')
    .addSelect(
      `CASE
        WHEN sub.id IS NULL THEN 'not_submitted'
        WHEN sub.submittedAt > a.dueAt THEN 'late'
        ELSE 'submitted' END`,
      'status',
    )
    .from(ClassStudentEntity, 'cs')
    .innerJoin(UserEntity, 'u', 'u.id = cs.studentId')
    .leftJoin(AssignmentEntity, 'a', 'a.classId = cs.classId AND a.id = :assignmentId', { assignmentId })
    .leftJoin(SubmissionEntity, 'sub', 'sub.assignmentId = a.id AND sub.studentId = cs.studentId')
    .where('cs.classId = :classId', { classId })
    .getRawMany();
}
```

## 4. Quy ước chung khác

- Màu lớp học: palette 8 màu cố định, chọn khi tạo lớp, lưu dạng token (`blue`, `cyan`, `amber`...) — không lưu mã màu tùy ý.
- Thời gian lưu DB dạng **UTC** (`timestamptz`), hiển thị theo múi giờ người dùng.
- Mọi API trả về JSON theo format: `{ data: ... }` hoặc `{ error: { message, code } }`.
- File trong dự án: `kebab-case` (`class-student.entity.ts`).
