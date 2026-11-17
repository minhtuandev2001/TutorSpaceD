# Bài 16 — Cài LiveKit server (Docker + TURN + token)

> ⚠️ Bài hạ tầng phức tạp nhất về phía server — chuẩn bị VPS từ đầu giai đoạn.

## 🎯 Mục tiêu

LiveKit server chạy trên VPS Singapore; token phòng được cấp từ app; học sinh mạng chặn UDP vẫn vào được (TURN); test 2 thiết bị gọi nhau thành công.

## 🧠 Kiến thức cần nắm

- WebRTC, SFU, TURN/STUN → [glossary/05-realtime-webrtc.md](../../glossary/05-realtime-webrtc.md)
- Docker, VPS, domain/HTTPS → [glossary/06-devops-deploy.md](../../glossary/06-devops-deploy.md)

## 📋 Các bước

### Bước 1: Chuẩn bị VPS

Thuê VPS Singapore (4 vCPU/8GB là đủ phát triển): Hetzner/DigitalOcean/Viettel Cloud. SSH vào, cài Docker + Compose. Mở firewall các cổng: 443 (TURN TLS + wss), 7880 (LiveKit API nội bộ), UDP 50000-60000 (media).

> **Vì sao Singapore:** độ trễ VN→SG ~30-50ms — cảm giác "gần như tại chỗ" cho gọi video. Vùng Âu/Mỹ sẽ thấy lag ~200ms+.
> **Vì sao biết các cổng:** media WebRTC dùng UDP dynamic range; NAT/firewall chặn UDP = học sinh vào được phòng nhưng không có hình/âm — bệnh khó chẩn đoán nếu không hiểu sơ đồ cổng.

### Bước 2: docker-compose LiveKit

```yaml
services:
  livekit:
    image: livekit/livekit-server:latest
    command: --config /etc/livekit.yaml
    restart: unless-stopped
    ports:
      - "7880:7880"
      - "7881:7881"          # TURN UDP
      - "443:443"            # TURN TLS
      - "50000-60000:50000-60000/udp"
    volumes: [./livekit.yaml:/etc/livekit.yaml]
```

`livekit.yaml` tối thiểu:

```yaml
port: 7880
rtc:
  tcp_port: 7881
  port_range_start: 50000
  port_range_end: 60000
  use_external_ip: true          # VPS có IP công khai — để LiveKit tự nhận
keys:
  <API_KEY>: <API_SECRET>        # tự sinh chuỗi ngẫu nhiên dài
turn:
  enabled: true
  domain: live.tenmiendomain.com
  tls_port: 443
  cert_file: /etc/letsencrypt/live/.../fullchain.pem
  key_file: /etc/letsencrypt/live/.../privkey.pem
```

> **Vì sao dùng file config:** cấu hình media nhạy cảm (cổng, TLS cert) — file rõ ràng dễ review hơn cờ command line.

### Bước 3: Domain + TLS cho TURN

Trỏ subdomain `live.xxx` → IP VPS. Cài Let's Encrypt cert (certbot). TURN TLS chạy cổng 443 — cùng cổng HTTPS giúp vượt qua các mạng chặn cổng lạ.

> **Vì sao TURN trên 443:** một số mạng chỉ cho ra ngoài cổng web (80/443) — TURN ở 443 là "lối thoát cuối" cho học sinh trong mạng trường học.

### Bước 4: Token service trong app

```bash
npm i livekit-server-sdk
```

`src/modules/room/token.service.ts`:

```typescript
import { AccessToken } from 'livekit-server-sdk';

export async function createRoomToken(sessionId: number, user: SessionUser) {
  // 1. kiểm tra user có quyền vào buổi này (tutor chủ lớp / HS trong lớp)
  // 2. kiểm tra phòng chưa bị khóa (bài 18) nếu user là student mới
  const at = new AccessToken(process.env.LIVEKIT_API_KEY, process.env.LIVEKIT_API_SECRET, {
    identity: `user-${user.userId}`,
    ttl: 60 * 60 * 4,                        // 4 giờ
  });
  at.addGrant({
    room: `session-${sessionId}`,
    roomJoin: true,
    canPublish: user.role === 'tutor',       // chỉ gia sư được đẩy stream mặc định
    canSubscribe: true,
    canPublishData: true,
  });
  return { token: await at.toJwt(), url: process.env.LIVEKIT_URL };
}
```

API: `POST /api/sessions/[id]/room-token` → trả `{ token, url }` cho client.

> **Vì sao token từ server app:** JWT LiveKit chứa quyền (publish/subscribe) — cấp từ client = học sinh tự cấp quyền publish cho mình. Server app có dữ liệu lớp → có quyền quyết định ai vào, quyền gì.
> **Vì sao `canPublish` khác theo vai trò:** mặc định "học sinh chỉ xem" là thiết kế lõi (glossary SFU); bài 18 sẽ có cơ chế mời bật cam/mic.

### Bước 5: `.env` app

```bash
LIVEKIT_URL=wss://live.xxx.com
LIVEKIT_API_KEY=...
LIVEKIT_API_SECRET=...
```

### Bước 6: Test 2 thiết bị

Trang test tạm `/room-test` dùng token API → join room → 2 thiết bị (laptop + điện thoại 4G) thấy nhau. Checklist:
- Laptop wifi + laptop wifi: OK
- Điện thoại 4G + laptop: OK
- Mạng công ty/trường (nếu có): nếu fail → TURN chưa đúng cert/port — xem `docker logs livekit`

> **Vì sao test theo ma trận mạng:** lỗi WebRTC gần như luôn là lỗi mạng, không phải code — phát hiện sớm từ bài này, không phải bài 17 giữa chừng.

## ✅ Tiêu chí hoàn thành

- [ ] `docker logs livekit` server chạy, không lỗi config
- [ ] API token trả về và client join được room
- [ ] 2 thiết bị khác mạng thấy nhau + nghe nhau
- [ ] Tình huống mạng chặn UDP vẫn kết nối qua TURN
- [ ] Commit: `lesson-16: livekit server + token service`

## 📝 Tự kiểm tra

1. SFU khác gì P2P mesh? Vì sao dự án chọn SFU ngay từ đầu?
2. Token LiveKit chứa quyền gì? Vì sao `canPublish` phải theo vai trò?
3. TURN trên cổng 443 giải quyết tình huống nào?
4. Client tự cấp token được không? Chặn bằng cơ chế nào?
