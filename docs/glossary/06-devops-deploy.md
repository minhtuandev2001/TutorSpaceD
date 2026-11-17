# 06 — DevOps: Docker, Deploy & vận hành

> Mục lục: [Môi trường](#1) • [Biến môi trường (.env)](#2) • [Git cơ bản](#3) • [Docker](#4) • [Docker Compose](#5) • [VPS](#6) • [Domain & HTTPS](#7) • [CI/CD](#8) • [Monitoring & Sentry](#9)

## 1. Môi trường {#1}

Bản code của bạn chạy ở 3 nơi khác nhau:

| Môi trường | Ở đâu | Dùng để |
|---|---|---|
| **Development (dev)** | Máy bạn | Viết code, chạy thử — DB Docker local, lỗi hiện đầy đủ |
| **Staging** | Server giống production | Thử tính năng mới trước khi cho người dùng |
| **Production** | Server thật | Người dùng thật dùng — lỗi chỉ hiện thông báo chung |

Quy tắc: cấu hình khác nhau giữa các môi trường → đưa vào biến môi trường, không viết cứng trong code.

## 2. Biến môi trường (.env) {#2}

File `.env` chứa cấu hình + **bảo mật** (mật khẩu, key) — **không bao giờ commit lên Git**:

```bash
DATABASE_URL=postgresql://postgres:123456@localhost:5432/tutorspace
LIVEKIT_URL=wss://livekit.mydomain.com
LIVEKIT_API_KEY=devkey
LIVEKIT_API_SECRET=secret-dai-ngau-nhien
```

- `.env.example` (commit lên Git): danh sách tên biến cần có, giá trị để trống hoặc giả.
- Trên Vercel: đặt trong Settings → Environment Variables.

## 3. Git cơ bản {#3}

Hệ quản lý phiên bản code. Lệnh dùng hằng ngày:

```bash
git add .                                  # chọn thay đổi
git commit -m "lesson-05: class management" # lưu phiên bản
git push                                    # đẩy lên GitHub
git log --oneline                          # xem lịch sử
```

**Quy ước dự án:** 1 bài = nhánh riêng (tùy chọn) hoặc commit rõ ràng theo bài. Message dạng `lesson-XX: mô tả`. Commit nhỏ, thường xuyên — là "nút lưu game".

## 4. Docker {#4}

Đóng gói app + mọi thứ nó cần thành **container** — chạy được ở mọi máy giống nhau ("chạy máy tôi mà!" hết cảnh khổ). 

```bash
docker run -e POSTGRES_PASSWORD=123456 -p 5432:5432 postgres:16
# chạy PostgreSQL 16 trên máy bạn, cổng 5432, không cần cài PostgreSQL
```

**Trong dự án:** local dùng Docker chạy PostgreSQL + Redis (bài 02); production dùng Docker chạy LiveKit server (bài 16, 28).

## 5. Docker Compose {#5}

File `docker-compose.yml` mô tả **nhiều container cùng chạy** + mạng + volume:

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: 123456
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]   # dữ liệu không mất khi tắt
  redis:
    image: redis:7
    ports: ["6379:6379"]
volumes:
  pgdata:
```

```bash
docker compose up -d      # bật tất cả
docker compose down       # tắt
```

## 6. VPS {#6}

**VPS (Virtual Private Server)** = máy chủ Linux thuê (máy ảo trong datacenter). Trong dự án dùng 1 VPS Singapore (gần VN) chạy: LiveKit server + TURN, Redis (production), Socket.io nếu cần. Công việc chính trên VPS: SSH vào, cài Docker, chạy compose, mở cổng firewall.

## 7. Domain & HTTPS {#7}

- **Domain:** tên web (`tutorspace.app`) trỏ về server qua DNS (bản ghi A → IP VPS, CNAME → Vercel).
- **HTTPS:** chứng chỉ SSL mã hóa — trên Vercel tự có; trên VPS dùng **Caddy** hoặc **Let's Encrypt** (miễn phí) tự gia hạn.

## 8. CI/CD {#8}

Tự động hóa sau mỗi lần push GitHub:
- **CI (Continuous Integration):** tự chạy lint + test + build — sai thì báo đỏ, không cho merge.
- **CD (Continuous Deployment):** tự deploy lên Vercel/VPS khi code ổn.

File cấu hình: `.github/workflows/ci.yml`. **Trong dự án:** CI chạy từ bài 01 — thói quen không để code lỗi tồn đọng.

## 9. Monitoring & Sentry {#9}

- **Sentry:** bắt lỗi runtime ở production kèm stack trace + người dùng nào gặp → biết app hỏng trước khi người dùng phàn nàn.
- **Uptime monitor (UptimeRobot):** ping web + LiveKit mỗi 5 phút, sập thì email.
- **Log:** Sentry cho lỗi, `docker logs` cho server LiveKit/Redis.

> Nhớ: deploy xong ≠ xong việc. "Có监控" là phần của tính năng — bài 28 sẽ làm tất cả các mục này.
