# Bài 07 — Upload file lên Cloudflare R2

## 🎯 Mục tiêu

Nền tảng upload dùng chung cho toàn dự án: avatar hồ sơ (bài này), ảnh đề bài (bài 11), bài nộp (bài 12), ảnh bảng trắng (bài 19), video ghi hình (bài 22) — tất cả đi qua một cơ chế duy nhất.

## 🧠 Kiến thức cần nắm

- Presigned URL → giải thích trong bài
- Biến môi trường, Docker → [glossary/06-devops-deploy.md](../../glossary/06-devops-deploy.md)
- R2 tương thích API S3 → dùng SDK chuẩn AWS

## 📋 Các bước

### Bước 1: Hiểu mô hình — vì sao file không đi qua server app?

```
❌ Sai cách:  trình duyệt → server Next.js → R2   (file chạy 2 chặng, server phải nhớ file lớn trong RAM/disk)
✅ Đúng cách: trình duyệt xin "giấy phép" (presigned URL) → tải thẳng lên R2
```

**Presigned URL** = URL do server ký sẵn, cho phép **chỉ 1 hành động cụ thể** (PUT file tên X, hết hạn sau 10 phút). Client tải thẳng lên storage — server app chỉ làm người phát giấy phép.

> **Vì sao quan trọng với dự án này:** bài nộp ảnh, video ghi hình nặng hàng trăm MB — nhờ presigned, server Next.js nhẹ, không tốn băng thông chuyển tiếp.

### Bước 2: Tạo bucket trên Cloudflare R2

1. Dashboard Cloudflare → R2 → Create bucket → tên `tutorspace-dev`.
2. R2 → API Tokens → tạo token có quyền Object Read & Write → lưu `ACCOUNT_ID`, `ACCESS_KEY_ID`, `SECRET_ACCESS_KEY`.
3. `.env`:

```bash
R2_ACCOUNT_ID=...
R2_ACCESS_KEY_ID=...
R2_SECRET_ACCESS_KEY=...
R2_BUCKET=tutorspace-dev
R2_PUBLIC_BASE=https://pub-xxxx.r2.dev    # bật public dev domain hoặc custom domain
```

### Bước 3: Cài SDK + module storage

```bash
npm i @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
```

`src/modules/storage/storage.service.ts`:

```typescript
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { randomUUID } from 'crypto';

const s3 = new S3Client({
  region: 'auto',
  endpoint: `https://${process.env.R2_ACCOUNT_ID}.r2.cloudflarestorage.com`,
  credentials: {
    accessKeyId: process.env.R2_ACCESS_KEY_ID!,
    secretAccessKey: process.env.R2_SECRET_ACCESS_KEY!,
  },
});

export async function createUploadUrl(opts: {
  folder: 'avatars' | 'assignments' | 'submissions' | 'boards' | 'recordings';
  fileName: string;
  contentType: string;
}) {
  const ext = opts.fileName.split('.').pop()?.toLowerCase() ?? 'bin';
  const key = `${opts.folder}/${new Date().toISOString().slice(0, 10)}/${randomUUID()}.${ext}`;

  const url = await getSignedUrl(
    s3,
    new PutObjectCommand({ Bucket: process.env.R2_BUCKET, Key: key, ContentType: opts.contentType }),
    { expiresIn: 600 },                       // 10 phút
  );
  return { uploadUrl: url, key, publicUrl: `${process.env.R2_PUBLIC_BASE}/${key}` };
}
```

> **Vì sao key có cấu trúc `folder/ngày/uuid`:** (1) folder phân loại — biết file nào của phân hệ nào, chính sách xóa khác nhau; (2) ngày — tìm file theo buổi học dễ dàng; (3) UUID — tránh trùng tên (2 học sinh cùng upload `bai-lam.jpg` không ghi đè nhau); giữ extension để trình duyệt biết loại file khi mở.

### Bước 4: API cấp giấy phép + validate loại file

```
POST /api/uploads/presign
Body: { folder, fileName, contentType }
```

Server kiểm tra: user đã đăng nhập, `folder` cho phép đúng vai trò (vd `submissions` chỉ student, `assignments` chỉ tutor), loại file nằm trong whitelist theo folder (`avatars`: image/*; `submissions`: image/* + pdf...).

> **Vì sao validate ở đây:** presigned URL là quyền — ai xin được là upload được. Không kiểm tra = học sinh tự ý upload file 5GB/anything lên bucket của bạn. **Giới hạn dung lượng** R2 không chặn được qua presigned → kiểm tra ở client + sẽ thêm luật ở tầng ứng dụng khi lưu metadata.

### Bước 5: Upload 2 bước ở client

Hook tái sử dụng `src/lib/use-upload.ts`:

```typescript
export async function uploadFile(file: File, folder: Folder) {
  const res = await fetch('/api/uploads/presign', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ folder, fileName: file.name, contentType: file.type }),
  }).then((r) => r.json());

  await fetch(res.data.uploadUrl, {
    method: 'PUT',
    body: file,
    headers: { 'Content-Type': file.type },
  });
  return res.data;   // { key, publicUrl }
}
```

> **Vì sao trả về `key` + `publicUrl`:** lưu **key** vào DB (tương đối, đổi domain/CDN không phải sửa DB), dùng publicUrl để hiển thị.

### Bước 6: Áp dụng đầu tiên — avatar hồ sơ

Trang profile (bài 03) thêm ảnh: input file → `uploadFile(file, 'avatars')` → lưu `avatarKey` vào `users` (migration thêm cột) → hiển thị.

> **Vì sao làm avatar trước:** file nhỏ, quay nhanh vòng "xin URL → upload → lưu → hiển thị" để chứng minh cơ chế, trước khi dùng cho file lớn ở bài sau.

## ✅ Tiêu chí hoàn thành

- [ ] Upload avatar thành công, ảnh hiển thị ở profile
- [ ] Xóa avatar cũ không còn file mồ côi (key đổi → ghi đè hoặc xóa file cũ)
- [ ] Student xin presign folder `assignments` → 403
- [ ] URL hết hạn 10 phút → upload lại phải xin mới
- [ ] Commit: `lesson-07: r2 presigned upload + avatar`

## 📝 Tự kiểm tra

1. Presigned URL là gì? Ai có thể dùng nó, trong thời gian nào?
2. Vì sao key có UUID? Chuyện gì xảy ra nếu 2 người cùng upload "anh-de-bai.jpg" mà không có UUID?
3. Validate loại file làm ở tầng nào? Vì sao không chỉ validate ở client?
4. Lưu `key` vào DB thay vì URL đầy đủ — được gì khi chuyển CDN?
