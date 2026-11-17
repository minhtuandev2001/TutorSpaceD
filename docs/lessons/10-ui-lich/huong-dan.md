# Bài 10 — UI lịch tổng & lịch lớp (calendar)

## 🎯 Mục tiêu

Trang lịch cho gia sư: tuần/tháng, màu theo lớp, lọc lớp, click buổi → chi tiết + hành động; tab Lịch trong trang lớp hiển thị lịch riêng của lớp đó.

## 🧠 Kiến thức cần nắm

- API lịch (bài 08), chuỗi lặp (bài 09)
- @schedule-x / FullCalendar → [glossary/02-frontend.md](../../glossary/02-frontend.md)

## 📋 Các bước

### Bước 1: Chọn thư viện calendar

So sánh nhanh:
| | @schedule-x | FullCalendar |
|---|---|---|
| Dung lượng | nhẹ (~15KB) | nặng hơn |
| API | React-first, hooks | Framework-agnostic, cấu hình object |
| Premium features | một số tính phí | nhiều thứ phí |

Dự án dùng **@schedule-x** cho nhẹ + React-first. Cài: `npm i @schedule-x/react @schedule-x/premium?` — chỉ dùng phần free (week/month view).

> **Vì sao không tự vẽ calendar:** vẽ grid tuần đúng (chồng giờ, timezone, DST, scroll) là hàng tuần công việc — dùng thư viện, dành công sức cho logic nghiệp vụ.

### Bước 2: Nối dữ liệu → events

Biến API (bài 08) thành event của calendar:

```typescript
const events = useMemo(() => sessions.map((s) => ({
  id: String(s.id),
  title: `${s.class.name}${s.topic ? ' · ' + s.topic : ''}`,
  start: new Date(s.startsAt).toISOString(),
  end: new Date(s.endsAt).toISOString(),
  calendarId: s.class.colorToken,          // màu = màu lớp!
  status: s.status,
})), [sessions]);
```

> **Vì sao `calendarId` = token màu lớp:** cơ chế "màu lớp xuyên suốt" — calendar library chia event theo calendarId, ta đặt mỗi lớp thành 1 calendar với màu tương ứng. Một chỗ cấu hình = toàn bộ lịch đúng màu.

### Bước 3: Trang lịch tổng `/tutor/calendar`

Layout:
- Toolbar: ◀ ▶ đổi tuần, label "16 – 22 tháng 11", toggle Tuần/Tháng, nút "+ Tạo buổi"
- Chips lọc lớp (toggle từng lớp ẩn/hiện — client-side, lọc trên mảng events)
- Calendar week view

Click event → Drawer (shadcn `Sheet`) chi tiết: lớp, giờ, chủ đề, trạng thái (badge), nút **Vào phòng học** (bài 17 gắn), **Sửa**, **Nghỉ buổi**.

> **Vì sao dùng Drawer cho chi tiết:** giữ ngữ cảnh lịch phía sau — user sửa xong vẫn thấy vị trí buổi trên lịch, tốt hơn mở trang mới.

### Bước 4: Tab Lịch trong trang lớp

Dùng lại **cùng component** `ScheduleCalendar` nhưng truyền `classId` → API gọi thêm filter `?classId=...`. Thêm nút "Tạo buổi lặp" mở dialog của bài 09.

> **Vì sao tái sử dụng component:** lịch tổng = lịch lớp + filter "tất cả". Một component, hai nơi dùng — sửa UI một chỗ.

### Bước 5: Hiển thị trạng thái buổi

Badge trong event + trong drawer:
- `upcoming` → outline; `live` → xanh lá + chấm nhấp nháy; `done` → xám mờ; `rested` → vàng + tooltip ghi chú lý do.

Lấy trạng thái tươi: sau mỗi thao tác `invalidateQueries(['sessions', range])` — TanStack Query tự refetch.

> **Vì sao invalidate theo range:** refetch cả lịch mọi tháng là lãng phí; key cache gồm khoảng thời gian đang xem — chỉ trang đó làm mới.

### Bước 6: Chỉnh buổi lẻ từ lịch

Dialog sửa (giờ, chủ đề) + nút "Nghỉ buổi" gọi API bài 09. Sau thao tác → toast kết quả + invalidate.

> **Vì sao thao tác nghỉ ngay trong drawer:** luồng thật của gia sư là "nhìn lịch thấy buổi nào không dạy được → nghỉ ngay tại đó", không phải vào tab khác.

## ✅ Tiêu chí hoàn thành

- [ ] Lịch tuần/tháng hiển thị đúng sự kiện, giờ đúng múi giờ VN
- [ ] Màu event = màu lớp, lọc lớp hoạt động
- [ ] Click buổi → drawer chi tiết + sửa + nghỉ được
- [ ] Tab lịch trong trang lớp dùng chung component
- [ ] Trạng thái live/done hiển thị đúng sau job cron
- [ ] Commit: `lesson-10: schedule ui (week/month + filters + drawer)`

## 📝 Tự kiểm tra

1. Vì sao chọn thư viện calendar thay vì tự vẽ? Chi phí thật của "tự vẽ" gồm những gì?
2. Màu lớp đi qua các tầng nào để tới được event trên lịch?
3. Cache TanStack Query key gồm những gì? Vì sao phải có range?
4. Drawer chi tiết tốt hơn trang riêng trong trường hợp này vì sao?
