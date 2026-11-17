# Bài 02 — TypeORM + PostgreSQL: kết nối database & migration đầu tiên

## 🎯 Mục tiêu

Dự án nói chuyện được với PostgreSQL qua TypeORM; có entity đầu tiên (`users`) và migration đầu tiên chạy thành công.

## 🧠 Kiến thức cần nắm

- Database, bảng, khóa, quan hệ → [glossary/04-database.md](../../glossary/04-database.md)
- Entity, Repository, Migration → [glossary/03-backend-orm.md](../../glossary/03-backend-orm.md)
- Docker, .env → [glossary/06-devops-deploy.md](../../glossary/06-devops-deploy.md)

## 📋 Các bước

### Bước 1: Bật PostgreSQL + Redis bằng Docker

Tạo `docker-compose.yml` ở **gốc** TutorSpaceD:

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: tutorspace
      POSTGRES_PASSWORD: devpass
      POSTGRES_DB: tutorspace
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]
  redis:
    image: redis:7
    ports: ["6379:6379"]
volumes:
  pgdata:
```

```bash
docker compose up -d
```

> **Vì sao Docker thay vì cài PostgreSQL thẳng?** Xóa máy này không mất môi trường; mọi máy/bài sau chạy đúng một cấu hình. `volumes` để dữ liệu không mất khi tắt container.

### Bước 2: Cài TypeORM

```bash
cd app
npm i typeorm pg reflect-metadata
npm i -D @types/node
```

> **Vì sao từng gói:** `typeorm` là ORM; `pg` là driver nói chuyện với PostgreSQL; `reflect-metadata` là cơ chế TypeORM đọc decorator (`@Column`...) — thiếu nó mọi entity vô hiệu.

### Bước 3: Bật decorator trong tsconfig

Sửa `tsconfig.json` thêm 2 dòng vào `compilerOptions`:

```json
"experimentalDecorators": true,
"emitDecoratorMetadata": true
```

> **Vì sao:** decorator (`@Entity`, `@Column`) là cú pháp đặc biệt TS phải bật tay. Quên bước này → lỗi "decorator is not valid here".

### Bước 4: Biến môi trường

Tạo `app/.env` (và `app/.env.example` cho bạn team):

```bash
DATABASE_URL=postgresql://tutorspace:devpass@localhost:5432/tutorspace
```

> **Vì sao:** mật khẩu DB không bao giờ nằm trong code. `.env` bỏ vào `.gitignore`, `.env.example` commit để ghi nhớ cần những biến nào.

### Bước 5: Entity đầu tiên — users

`src/db/entities/user.entity.ts`:

```typescript
import { Entity, PrimaryGeneratedColumn, Column, CreateDateColumn } from 'typeorm';

export type UserRole = 'tutor' | 'student';

@Entity({ name: 'users' })
export class UserEntity {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ type: 'varchar', length: 120 })
  fullName: string;

  @Column({ type: 'varchar', length: 160, unique: true })
  email: string;

  @Column({ type: 'varchar', length: 20, default: 'student' })
  role: UserRole;

  @Column({ type: 'varchar', length: 255, nullable: true })
  passwordHash?: string;   // đã hash, KHÔNG lưu mật khẩu thô

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;
}
```

> **Vì sao từng dòng:** `unique` ở email — không cho 2 tài khoản trùng mail; `role` mặc định `student` — ai đăng ký mới là học sinh, gia sư nâng quyền sau; `passwordHash` đặt tên nói rõ là đã băm (bài 03 dùng); `nullable` vì đăng nhập bằng Google không có mật khẩu.

### Bước 6: data-source.ts — điểm kết nối

`src/db/data-source.ts`:

```typescript
import 'reflect-metadata';
import { DataSource } from 'typeorm';
import { UserEntity } from './entities/user.entity';

export const AppDataSource = new DataSource({
  type: 'postgres',
  url: process.env.DATABASE_URL,
  entities: [UserEntity],
  migrations: [__dirname + '/migrations/*{.ts,.js}'],
  synchronize: false,          // QUAN TRỌNG: luôn false
  logging: ['error'],
});
```

> **Vì sao `synchronize: false`:** nếu true, TypeORM tự sửa bảng theo entity mỗi lần chạy — tiện dev nhưng dễ làm mất dữ liệu khi đổi cột. Dự án chuẩn dùng migration (bước 7) — tự tay kiểm soát từng thay đổi.

### Bước 7: Migration đầu tiên

```bash
npm i -D ts-node
npx typeorm migration:generate -d src/db/data-source.ts src/db/migrations/CreateUsers
npx typeorm migration:run -d src/db/data-source.ts
```

Kiểm tra: `docker compose exec postgres psql -U tutorspace -c "\dt"` → thấy bảng `users` + bảng `migrations`.

> **Vì sao generate thay vì viết tay:** generate **đọc sự khác nhau giữa entity và DB hiện tại** rồi sinh migration — bạn review file sinh ra trước khi run. Đây là thói quen an toàn: entity là nguồn sự thật, migration là bước thực thi.

### Bước 8: Kiểm tra kết nối từ Next.js

Tạo `app/api/health/route.ts`:

```typescript
import { AppDataSource } from '@/db/data-source';

export async function GET() {
  const ds = AppDataSource.isInitialized ? AppDataSource : await AppDataSource.initialize();
  const count = await ds.getRepository('users').count();
  return Response.json({ ok: true, users: count });
}
```

Mở `http://localhost:3000/api/health` → `{"ok":true,"users":0}`.

> **Vì sao có route này:** "route sức khỏe" là cách nhanh nhất kiểm tra tầng DB hoạt động, và sau này dùng cho uptime monitor (bài 28).

## ✅ Tiêu chí hoàn thành

- [ ] `docker compose ps` → postgres + redis chạy
- [ ] `migration:run` thành công, bảng `users` tồn tại
- [ ] `/api/health` trả `{"ok":true,"users":0}`
- [ ] Commit: `lesson-02: typeorm + postgres + users migration`

## 📝 Tự kiểm tra

1. `synchronize: true` nguy hiểm thế nào? Vì sao dự án cấm?
2. `@PrimaryGeneratedColumn` sinh cột kiểu gì trong PostgreSQL?
3. Khi nào dùng `migration:generate` và khi nào viết migration tay?
4. `reflect-metadata` làm nhiệm vụ gì?
