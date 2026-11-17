# Bài 03 — Đăng ký & đăng nhập (Auth.js)

## 🎯 Mục tiêu

Người dùng đăng ký (email + mật khẩu), đăng nhập bằng Google, session hoạt động, hồ sơ cá nhân xem/sửa được.

## 🧠 Kiến thức cần nắm

- Cookie/Session/JWT → [glossary/01-web-api-co-ban.md](../../glossary/01-web-api-co-ban.md#6-cookie-vs-session-vs-jwt)
- Middleware, zod → [glossary/03-backend-orm.md](../../glossary/03-backend-orm.md)
- Hash mật khẩu (bcrypt) — giải thích trong bài

## 📋 Các bước

### Bước 1: Cài Auth.js

```bash
npm i next-auth@beta @auth/prisma-adapter   # next-auth v5 (Auth.js)
npm i bcryptjs && npm i -D @types/bcryptjs
```

> **Vì sao Auth.js:** đăng nhập tự viết dễ sai ở chỗ nguy hiểm nhất (lưu session, chống giả mạo cookie). Auth.js làm chuẩn, hỗ trợ Google OAuth sẵn. Adapter sẽ cấu hình ở bước 3.

### Bước 2: Cấu hình Auth.js với TypeORM

Auth.js mặc định dùng Prisma — dự án của ta dùng TypeORM nên **tự viết adapter nhỏ**:

`src/modules/auth/auth-options.ts`:

```typescript
import NextAuth from 'next-auth';
import Google from 'next-auth/providers/google';
import Credentials from 'next-auth/providers/credentials';
import { AppDataSource } from '@/db/data-source';
import { UserEntity } from '@/db/entities/user.entity';
import bcrypt from 'bcryptjs';

export const { handlers, auth, signIn, signOut } = NextAuth({
  session: { strategy: 'jwt' },
  providers: [
    Google({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
    Credentials({
      credentials: { email: {}, password: {} },
      async authorize(creds) {
        await AppDataSource.initialize().catch(() => {});
        const user = await AppDataSource.getRepository(UserEntity).findOneBy({
          email: creds?.email as string,
        });
        if (!user?.passwordHash) return null;
        const ok = await bcrypt.compare(creds!.password as string, user.passwordHash);
        return ok ? { id: String(user.id), email: user.email, name: user.fullName, role: user.role } : null;
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) token.role = (user as any).role;   // gắn role vào token
      return token;
    },
    async session({ session, token }) {
      (session as any).userId = token.sub;
      (session as any).role = token.role;
      return session;
    },
  },
});
```

> **Vì sao từng phần:**
> - `strategy: 'jwt'` — session dạng JWT (stateless), server không cần bảng session.
> - `authorize` cho đăng nhập email/mật khẩu: tìm user theo email, so mật khẩu bằng `bcrypt.compare` — **không bao giờ so chuỗi mật khẩu trực tiếp** vì DB chỉ lưu dạng hash.
> - `callbacks`: gắn `role` vào token → mọi API sau đọc được vai trò không cần truy DB lại mỗi request.

### Bước 3: Route handler + middleware

```bash
# app/api/auth/[...nextauth]/route.ts
export { GET, POST } from '@/modules/auth/auth-options';

# src/middleware.ts — chặn chưa đăng nhập
import { auth } from '@/modules/auth/auth-options';
export default auth((req) => {
  const isLoggedIn = !!req.auth;
  if (!isLoggedIn && req.nextUrl.pathname.startsWith('/tutor')) {
    return Response.redirect(new URL('/login', req.url));
  }
});
export const config = { matcher: ['/tutor/:path*'] };
```

> **Vì sao:** middleware là cổng duy nhất — mọi trang gia sư đều được bảo vệ tự động, không phải mỗi trang tự kiểm tra.

### Bước 4: Đăng ký tài khoản (API + form)

API `app/api/auth/register/route.ts`:

```typescript
import { z } from 'zod';
import bcrypt from 'bcryptjs';
import { AppDataSource } from '@/db/data-source';
import { UserEntity } from '@/db/entities/user.entity';

const schema = z.object({
  fullName: z.string().min(2),
  email: z.string().email(),
  password: z.string().min(8),
  role: z.enum(['tutor', 'student']),
});

export async function POST(req: Request) {
  const dto = schema.parse(await req.json());           // sai → 400 tự động
  await AppDataSource.initialize().catch(() => {});
  const repo = AppDataSource.getRepository(UserEntity);
  const exists = await repo.findOneBy({ email: dto.email });
  if (exists) return Response.json({ error: { message: 'Email đã được dùng' } }, { status: 409 });

  const user = repo.create({
    fullName: dto.fullName,
    email: dto.email,
    passwordHash: await bcrypt.hash(dto.password, 10),  // hash trước khi lưu!
    role: dto.role,
  });
  await repo.save(user);
  return Response.json({ data: { id: user.id } }, { status: 201 });
}
```

> **Vì sao hash:** nếu hacker đọc được DB, thấy mật khẩu thô = thảm họa. bcrypt hash 1 chiều — không giải ngược được. `10` là số vòng băm (cân bằng an toàn/tốc độ).
> **Vì sao kiểm tra email tồn tại trước:** tránh trùng tài khoản, trả lỗi 409 rõ ràng thay vì lỗi DB ngầm.

### Bước 5: Trang đăng nhập/đăng ký (UI)

- `app/(auth)/login/page.tsx` + `register/page.tsx` dùng shadcn `Input`/`Button`.
- Đăng ký: form chọn vai trò (radio Gia sư / Học sinh).
- Đăng nhập: 2 nút — "Đăng nhập Google" (gọi `signIn('google')`) và form email/mật khẩu.

> **Vì sao chọn role lúc đăng ký:** mô hình dự án là gia sư tự quản — đơn giản nhất là người dùng tự khai vai trò lúc tạo tài khoản. (Sản phẩm thương mại thật sẽ cần quy trình duyệt gia sư — ghi vào backlog.)

### Bước 6: Trang hồ sơ cá nhân

`app/profile/page.tsx` — Server Component đọc session (`await auth()`), hiển thị tên/email/role, form đổi tên.

> **Vì sao:** kiểm chứng session hoạt động thật, chuẩn bị chỗ để gắn avatar (bài 07).

## ✅ Tiêu chí hoàn thành

- [ ] Đăng ký 2 tài khoản (1 tutor, 1 student) bằng email/mật khẩu
- [ ] Đăng nhập Google thành công (tạo user tự động nếu mới)
- [ ] Vào `/tutor` chưa đăng nhập → bị đá về `/login`
- [ ] Session hiển thị đúng tên + role ở trang hồ sơ
- [ ] Trong DB, cột `password_hash` là chuỗi hash (không đọc được mật khẩu)
- [ ] Commit: `lesson-03: auth register + login + session`

## 📝 Tự kiểm tra

1. Vì sao không lưu mật khẩu thô? bcrypt.hash khác gì bcrypt.compare?
2. `strategy: 'jwt'` khác gì session lưu bảng DB?
3. Token chứa `role` ở đâu, API nào đọc được nó?
4. Nếu client gửi `password` 3 ký tự thì chuyện gì xảy ra và ai chặn?
