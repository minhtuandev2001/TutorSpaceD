# Bài 04 — Phân quyền: vai trò gia sư / học sinh

## 🎯 Mục tiêu

Hệ thống phân biệt 2 vai trò ở **tất cả các tầng**: giao diện khác nhau, API chặn sai quyền, guard tái sử dụng được cho các bài sau.

## 🧠 Kiến thức cần nắm

- Middleware & Guard → [glossary/03-backend-orm.md](../../glossary/03-backend-orm.md)
- 401 vs 403 → [glossary/01-web-api-co-ban.md](../../glossary/01-web-api-co-ban.md#3-http)

## 📋 Các bước

### Bước 1: Chuẩn hóa hàm đọc session

`src/modules/auth/get-session.ts`:

```typescript
import { auth } from '@/modules/auth/auth-options';

export async function requireUser() {
  const session = await auth();
  if (!session?.userId) throw new UnauthorizedError();
  return { userId: Number(session.userId), role: session.role as 'tutor' | 'student' };
}

export async function requireRole(role: 'tutor' | 'student') {
  const user = await requireUser();
  if (user.role !== role) throw new ForbiddenError();
  return user;
}
```

> **Vì sao:** chốt **một cổng duy nhất** đọc session. Bài sau mỗi API chỉ gọi `requireRole('tutor')` — không lặp code auth, không chỗ nào quên kiểm tra.

### Bước 2: Lớp lỗi chuẩn + bọc handler

`src/lib/errors.ts` định nghĩa `UnauthorizedError` (401) và `ForbiddenError` (403); thêm hàm bọc `withApi()` tự chuyển lỗi thành JSON đúng mã:

```typescript
export function withApi(handler: (req: Request, ctx?: any) => Promise<Response>) {
  return async (req: Request, ctx?: any) => {
    try {
      return await handler(req, ctx);
    } catch (e) {
      if (e instanceof UnauthorizedError) return Response.json({ error: { message: 'Chưa đăng nhập' } }, { status: 401 });
      if (e instanceof ForbiddenError) return Response.json({ error: { message: 'Không có quyền' } }, { status: 403 });
      if (e instanceof ZodError) return Response.json({ error: { message: 'Dữ liệu sai', details: e.issues } }, { status: 400 });
      console.error(e);
      return Response.json({ error: { message: 'Lỗi hệ thống' } }, { status: 500 });
    }
  };
}
```

> **Vì sao:** quy ước lỗi thống nhất từ đầu — client luôn biết cách xử lý, Sentry/S3 log được phân loại (bài 28 gắn thêm).

### Bước 3: Menu theo vai trò

Layout chia 2 vùng:
- `/tutor/*` — dashboard gia sư (chỉ tutor thấy)
- `/student/*` — trang học sinh (chỉ student thấy)

Trang chủ sau đăng nhập redirect theo role: `router.push(role === 'tutor' ? '/tutor' : '/student')`.

> **Vì sao tách khu vực:** gia sư và học sinh làm việc khác nhau hoàn toàn; tách route giúp guard đơn giản (check role 1 lần ở middleware) và UI mỗi bên tập trung đúng việc của mình.

### Bước 4: Guard API đầu tiên — thử với health

Đổi `/api/health` thành yêu cầu đăng nhập:

```typescript
export const GET = withApi(async () => {
  await requireUser();                    // 401 nếu chưa đăng nhập
  return Response.json({ ok: true });
});
```

> **Vì sao dùng route cũ:** thử cơ chế guard ở chỗ không sợ hỏng gì, trước khi gắn vào API thật ở bài 05.

### Bước 5: Test bằng tay

- Không đăng nhập → gọi `/api/health` → 401
- Đăng nhập student → gọi API có `requireRole('tutor')` → 403
- Đăng nhập đúng role → 200

Viết script test gọn trong `scripts/test-guard.mjs` (fetch tuần tự, in kết quả).

> **Vì sao test tay trước khi có bộ test tự động:** chứng minh cơ chế hoạt động + hiểu rõ luồng; unit test thật sẽ thêm dần khi có repository (bài 05).

## ✅ Tiêu chí hoàn thành

- [ ] `requireRole` chặn đúng 401/403/200 ở mọi trường hợp
- [ ] Student không truy cập được `/tutor/*`, tutor không vào được `/student/*`
- [ ] Mọi lỗi API trả JSON theo format chuẩn của dự án
- [ ] Commit: `lesson-04: roles + guards + error envelope`

## 📝 Tự kiểm tra

1. 401 khác 403? Cho ví dụ từ dự án.
2. Vì sao guard nằm ở hàm dùng chung thay vì viết trong từng API?
3. Role nằm trong JWT — điều này có rủi ro gì? (Gợi ý: role "nâng" sau khi token đã ký thì nhận ra khi nào?)
