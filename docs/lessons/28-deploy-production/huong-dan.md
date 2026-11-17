# Bài 28 — Deploy production (VPS + Docker + HTTPS + monitoring)

## 🎯 Mục tiêu

App chạy thật trên domain có HTTPS; LiveKit/Redis production trên VPS; CI/CD tự deploy; Sentry + uptime monitor giám sát. Sau bài này: **mời gia sư thật dùng 2 tuần** (đích cuối dự án).

## 🧠 Kiến thức cần nắm

- Môi trường, Docker Compose, domain/HTTPS, CI/CD, Sentry → [glossary/06-devops-deploy.md](../../glossary/06-devops-deploy.md)

## 📋 Các bước

### Bước 1: Bản đồ triển khai

```
Domain (Cloudflare DNS)
├── app.tutorspace.app      → Vercel (Next.js app + API)
├── live.tutorspace.app     → VPS (LiveKit + TURN TLS:443)
├── sync.tutorspace.app     → VPS (tldraw sync server, wss)
└── worker (VPS, không domain)  → BullMQ worker + cron

VPS Docker Compose: livekit, redis, sync-server, worker
Database: Neon/Supabase PostgreSQL (managed) — hoặc trên VPS
Storage: Cloudflare R2 (production bucket)
```

> **Vì sao DB managed (Neon):** backup tự động, point-in-time restore — dữ liệu học sinh không phải thứ tự thành thử tay backup. Rẻ giai đoạn đầu, đúng khả năng.
> **Vì sao tách Vercel (app) và VPS (media):** app stateless scale dễ trên Vercel; media cần server giữ kết nối UDP/TCP liên tục — VPS quản lý trực tiếp.

### Bước 2: Chuẩn bị cấu hình production

- `.env.production` trên Vercel: DATABASE_URL (Neon), LIVEKIT_*, R2_* (bucket prod), RESEND_API_KEY, SENTRY_DSN, APP_URL
- Kiểm tra **mọi biến có trong `.env.example`** — deploy thiếu biến là lỗi xuất hiện 30 phút sau khi deploy, khó truy
- Migration chạy trong release step: `npm run migration:run` trước khi app mới lên

> **Vì sao migration trước deploy:** app mới code giả định schema mới; chạy migration sau khi app lên = app mới gặp schema cũ trong vài giây → lỗi loạn.

### Bước 3: VPS — docker-compose.production.yml

```yaml
services:
  livekit:    # như bài 16 + cert TLS production
  redis:
    image: redis:7
    volumes: [redisdata:/data]
    command: redis-server --requirepass ${REDIS_PASSWORD}
  sync-server:    # tldraw sync (Dockerfile riêng)
    build: ./sync-server
  worker:
    build: ./worker   # BullMQ worker + cron (bài 15, 22)
volumes:
  redisdata:
```

UFW firewall: mở 443, 7881, UDP 50000-60000; **không** mở 5432/6379 ra ngoài.

> **Vì sao chặn DB/Redis:** 5432/6379 mở public = bot scan toàn internet sẽ thử mật khẩu trong vài giờ. Redis có password + chỉ nội bộ.

### Bước 4: HTTPS

- Vercel: tự động cho app
- VPS: Let's Encrypt cho `live.` + `sync.` (certbot hoặc Caddy reverse proxy), cron gia hạn

### Bước 5: CI/CD hoàn chỉnh

`.github/workflows/deploy.yml`:
- Push tag `v*` → chạy test → deploy Vercel (`vercel --prod`) → SSH vào VPS `docker compose pull && up -d` (sync/worker image build qua registry)

> **Vì sao deploy bằng tag:** "main ổn định, tag = phát hành" — rollback là `git revert` + tag cũ, không phải tìm commit nào là commit tốt.

### Bước 6: Monitoring + backup

- **Sentry:** DSN production; test bằng route `/api/debug-error` (xóa sau)
- **UptimeRobot:** ping `/api/health` + `live.` + `sync.` mỗi 5 phút → email khi sập
- **Backup DB:** Neon có PITR tự động; nếu DB trên VPS → cron `pg_dump` hàng ngày upload R2 giữ 7 ngày
- **Chạy seed admin** (bài 27) 1 lần

> **Vì sao backup trước khi có user:** mất dữ liệu học sinh 1 lần là mất uy tín sản phẩm vĩnh viễn — backup là đầu tư rẻ nhất rủi ro nhất.

### Bước 7: Chạy thật với gia sư thật 🎓

- Mời 1–2 gia sư tạo lớp thật, dạy 2 tuần liên tục
- Bạn trực hàng: thu lỗi Sentry, trả lời sớm
- Mỗi tuần: review Sentry + feedback → sửa → deploy; ghi yêu cầu mới vào backlog (`02-danh-sach-chuc-nang.md`)
- Chuẩn: gia sư dùng **không cần bạn hướng dẫn từng bước**

> **Vì sao đây là đích của dự án:** mọi tính năng trước đó chỉ là giả thuyết; 2 tuần người dùng thật là bằng chứng duy nhất dự án thành công — và là bài học lớn nhất.

## ✅ Tiêu chí hoàn thành

- [ ] `https://app.tutorspace.app` chạy được đầy đủ luồng thật
- [ ] Phòng học + bảng trắng qua domain prod (không localhost)
- [ ] Push tag → CI xanh → tự động deploy cả app + VPS
- [ ] Sentry bắt được lỗi thử; uptime cảnh báo khi tắt service
- [ ] Backup DB chạy hằng ngày, khôi phục thử thành công
- [ ] 2 tuần gia sư thật chạy liên tục
- [ ] Commit/tag cuối: `v1.0.0`

## 📝 Tự kiểm tra

1. Vì sao migration chạy trước khi app mới lên?
2. 5432/6379 mở public — kịch bản tấn công cụ thể?
3. Deploy theo tag khác deploy theo main? Quy trình rollback?
4. "2 tuần gia sư thật" đo được điều gì mà test không đo được? Liệt kê 3 điều.
