# Bài 22 — Ghi hình buổi học + trang xem lại

## 🎯 Mục tiêu

Buổi học được ghi thành video MP4 (LiveKit Egress) → lưu R2 → trang xem lại video + bảng trắng + chat của buổi.

## 🧠 Kiến thức cần nắm

- LiveKit Egress → [glossary/05-realtime-webrtc.md](../../glossary/05-realtime-webrtc.md#7-livekit)
- BullMQ job nền, presigned upload (bài 07, 15)

## 📋 Các bước

### Bước 1: Hiểu Egress — các chế độ

| Chế độ | Làm gì | Dùng cho |
|---|---|---|
| **RoomComposite** | Render cả phòng thành 1 video (layout tự động) | **Dự án dùng** — ghi bài giảng |
| TrackEgress | Ghi từng track riêng | Giao hàng nguyên bản từng cam |
| WebEgress | Ghi trang web | Không dùng |

RoomComposite chạy 1 "trình duyệt ảo" trên server render room — tốn CPU server, chỉ bật khi cần.

> **Vì sao RoomComposite:** bài giảng có giá trị ở "gia sư nói + bảng + ai đang hỏi" — layout tổng hợp đúng nhu cầu xem lại, không phải cắt từng track.

### Bước 2: Chọn ghi — bật/tắt theo buổi

Entity `recordings`:

```typescript
@Entity({ name: 'recordings' })
export class RecordingEntity {
  @PrimaryGeneratedColumn() id: number;
  @Column({ name: 'session_id' }) sessionId: number;
  @Column({ type: 'enum', enum: ['recording', 'processing', 'ready', 'failed'] })
  status: 'recording' | 'processing' | 'ready' | 'failed';
  @Column({ type: 'varchar', nullable: true }) egressId?: string;   // id từ LiveKit
  @Column({ type: 'varchar', nullable: true }) fileKey?: string;    // key R2
  @Column({ type: 'integer', nullable: true }) durationSec?: number;
  @Column({ type: 'bigint', nullable: true }) sizeBytes?: number;
  @CreateDateColumn({ name: 'created_at' }) createdAt: Date;
}
```

Toggle trong toolbar phòng (tutor): "Bắt đầu ghi" → API gọi `egress.startRoomCompositeEgress(roomName, { file: { filepath: 'r2://...' } }, layout: 'grid')` → lưu `egressId`. Dừng ghi → `stopEgress`.

> **Vì sao tốn CPU nên bật/tắt:** ghi cả buổi 90 phút liên tục trên VPS 4 vCPU vừa SFU vừa egress là nặng; mặc định tắt, gia sư chủ động bật.
> **Vì sao lưu egressId:** Egress bất đồng bộ (file có sẵn vài phút sau khi dừng) — cần id để webhook/job đối chiếu trạng thái.

### Bước 3: Lưu file lên R2 qua job

Cấu hình egress ghi vào S3-compatible (R2) trực tiếp, hoặc job worker:
- `egressCompleted` webhook từ LiveKit (hoặc poll `egress.getEgress`) khi kết thúc
- Job BullMQ: tải file (nếu chưa ở R2) → upload R2 folder `recordings` → cập nhật `status='ready'` + duration/size

> **Vì sao qua job:** egress hoàn tất mất vài phút — request/API không nên chờ; job có retry khi upload thất bại.

### Bước 4: Trang xem lại buổi

Từ lịch/drawer buổi `done`: nút "Xem lại buổi học" → trang:
- **Video player** (MP4 từ R2, presigned GET URL có hạn — không public link)
- Tabs: Video • Bảng trắng (ảnh snapshot bài 19) • Chat (bài 20)
- Thời lượng + dung lượng hiển thị

> **Vì sao presigned GET:** public MP4 = bất kỳ ai có link xem miễn phí vĩnh viễn; presigned hết hạn kiểm soát quyền theo session đăng nhập.

### Bước 5: Danh sách ghi hình của lớp

Tab trong trang lớp: các buổi có `recordings.status='ready'` + khả năng xóa (tutor chủ) → xóa file R2 + dòng DB (job xóa file async).

> **Vì sao cho xóa:** lưu trữ tốn tiền; gia sư phải có quyền quản lý dữ liệu của mình — không xóa thì chi phí storage tăng vô kiểm soát.

### Bước 6: Kiểm thử

Buổi 5 phút bật ghi → dừng → đợi job → MP4 trong R2, trang xem lại phát được; kiểm tra video có âm thanh gia sư + khung view hợp lý.

## ✅ Tiêu chí hoàn thành

- [ ] Bật ghi giữa buổi → dừng → MP4 tự lưu R2
- [ ] Trang xem lại: video + bảng + chat trong 1 trang
- [ ] Link video là presigned (hết hạn lại xin được)
- [ ] Xóa recording → file R2 cũng xóa
- [ ] Commit: `lesson-22: recording egress + playback page`

## 📝 Tự kiểm tra

1. RoomComposite khác TrackEgress — vì sao chọn RoomComposite cho bài giảng?
2. Vì sao egress tốn CPU? Ghi 3 phòng cùng lúc trên VPS 4 vCPU — ước lượng được gì?
3. Presigned GET URL — khác gì public URL? Ai được xem trong bao lâu?
4. Nếu job upload thất bại 3 lần — trạng thái recording phải là gì, ai được biết?
