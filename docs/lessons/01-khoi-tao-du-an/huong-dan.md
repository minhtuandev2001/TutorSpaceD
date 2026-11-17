# Bài 01 — Khởi tạo dự án TutorSpaceD

## 🎯 Mục tiêu

Tạo khung dự án Next.js chạy được: TypeScript, Tailwind, shadcn/ui, cấu trúc folder chuẩn, Git + CI — nền để mọi bài sau xây lên.

## 🧠 Kiến thức cần nắm

- Component, App Router, Server/Client Component → [glossary/02-frontend.md](../../glossary/02-frontend.md)
- Git, biến môi trường, CI → [glossary/06-devops-deploy.md](../../glossary/06-devops-deploy.md)

## 📋 Các bước

### Bước 1: Cài công cụ trên máy

```bash
# Kiểm tra đã có chưa (node >= 20, git)
node -v
git --version
```

> **Vì sao:** Next.js 15 yêu cầu Node >= 18.17 (khuyên 20+). Git để lưu lịch sử học của bạn.

### Bước 2: Tạo dự án Next.js

Chạy **trong folder dự án đã có** (`TutorSpaceD`):

```bash
npx create-next-app@latest app --ts --tailwind --eslint --app --src-dir --import-alias "@/*" --use-npm --no-turbopack
```

> **Vì sao:** `create-next-app` sinh khung chuẩn để không tự dựng từng cấu hình. Các cờ: `--ts` TypeScript (bắt lỗi khi viết code — chất lượng học cao hơn JS thường), `--app` dùng App Router (chuẩn mới), `--src-dir` gom code vào `src/` gọn gàng. Code nằm trong `app/` — folder gốc dành cho `docs/` của bạn.

### Bước 3: Dọn trang mặc định, cấu hình style Vercel

- Đổi background mặc định trong `app/layout.tsx` + `globals.css` sang nền tối: `bg-black text-neutral-100`.
- Cài font Geist (create-next-app đã có sẵn Geist — kiểm tra `app/layout.tsx`).

> **Vì sao:** chốt phong cách ngay từ đầu để mọi component sau làm theo một chuẩn duy nhất, không phải "sơn lại" sau.

### Bước 4: Cài shadcn/ui

```bash
cd app
npx shadcn@latest init      # chọn: Neutral, CSS variables
npx shadcn@latest add button card input label dialog badge
```

> **Vì sao:** shadcn **copy code component vào dự án** (`src/components/ui/`) — bạn sở hữu và sửa được. 6 component đầu là thứ mọi màn hình sau sẽ dùng.

### Bước 5: Tạo cấu trúc folder theo plan

```
src/
├── app/                  (do create-next-app tạo)
├── components/
│   ├── ui/               (shadcn)
│   ├── layout/           (nav, sidebar)
│   ├── calendar/         (bài 10)
│   └── room/             (bài 16+)
├── db/
│   ├── entities/         (TypeORM — bài 02)
│   ├── migrations/
│   └── data-source.ts
├── modules/              (mỗi phân hệ 1 folder: classes/, sessions/, assignments/...)
└── lib/                  (hàm dùng chung)
```

> **Vì sao:** định vị code từ đầu. `modules/` là nguyên tắc "mỗi phân hệ một folder" — khi dự án lớn, tìm và sửa tập trung.

### Bước 6: Git + commit đầu tiên

```bash
cd ..   # về gốc TutorSpaceD
git init
# đảm bảo app/.env*, node_modules đã trong .gitignore
git add . && git commit -m "lesson-01: init Next.js + Tailwind + shadcn"
```

> **Vì sao:** commit là "nút lưu game" — sai bước nào quay lại được bước đó.

### Bước 7: CI cơ bản (GitHub Actions)

Tạo `.github/workflows/ci.yml`:

```yaml
name: CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: cd app && npm ci
      - run: cd app && npm run lint
      - run: cd app && npm run build
```

> **Vì sao:** CI là "kiểm tra tự động" — mỗi lần push, GitHub tự build. Thói quen này từ bài đầu giúp bạn không mang lỗi sang bài sau.

## ✅ Tiêu chí hoàn thành

- [ ] `npm run dev` mở được trang chủ nền tối
- [ ] `<Button>` của shadcn render đúng
- [ ] Có commit `lesson-01` trên GitHub, CI xanh
- [ ] Cấu trúc folder như bước 5

## 📝 Tự kiểm tra

1. `page.tsx` là gì? Khác gì `route.ts`?
2. Khi nào thêm `'use client'` vào đầu file?
3. Vì sao shadcn khác các UI library thông thường (Ant Design...)?
