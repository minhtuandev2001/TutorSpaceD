# 02 — Frontend: React, Next.js & UI

> Mục lục: [React & Component](#1) • [JSX](#2) • [State & Props](#3) • [Hook](#4) • [Next.js & App Router](#5) • [Server vs Client Component](#6) • [Tailwind CSS](#7) • [shadcn/ui](#8) • [TanStack Query](#9) • [Zustand](#10)

## 1. React & Component {#1}

**Component** = mảnh giao diện tái sử dụng, viết bằng hàm JS trả về phần hiển thị. Ứng dụng = ghép các mảnh như lắp lego.

```tsx
function ClassCard({ name, color }) {
  return <div className="card">{name}</div>;
}
// Dùng lại 3 lần cho 3 lớp → chỉ cần gọi <ClassCard> 3 lần
```

**Trong dự án:** thẻ "Toán 9-A" trên dashboard là 1 component — đổi dữ liệu đầu vào là tự render lớp khác.

## 2. JSX {#2}

Cách viết HTML ngay trong code JS: `<div className="card">Chào {user.name}</div>`. Chú ý: dùng `className` (không phải `class`), biểu thức JS bọc trong `{ }`.

## 3. State & Props {#3}

- **Props:** dữ liệu cha truyền xuống con — chỉ đọc. `<ClassCard name="Toán 9-A" />`
- **State:** dữ liệu component **tự giữ và tự thay đổi**, thay đổi xong React tự vẽ lại giao diện.

**Trong dự án:** trạng thái bật/tắt mic trong phòng học là state — bấm nút → state đổi → icon và màu nút đổi ngay không cần tải lại trang.

## 4. Hook {#4}

Hàm đặc biệt bắt đầu bằng `use`, cho component dùng được state + các khả năng của React:

| Hook | Dùng để | Ví dụ trong dự án |
|---|---|---|
| `useState` | Tạo state | `const [micOn, setMicOn] = useState(true)` |
| `useEffect` | Chạy code phụ (kết nối WebSocket, hẹn giờ) sau khi render | Kết nối Socket.io khi vào phòng |
| `useForm` (react-hook-form) | Quản lý form | Form tạo lớp, form giao bài |

Quy tắc: hook chỉ gọi trong component, gọi ở **đầu hàm**, không gọi trong `if`.

## 5. Next.js & App Router {#5}

Next.js là framework xây trên React, thêm: điều hướng trang, gọi server, tối ưu tốc độ. **App Router** = cách tổ chức theo folder:

```
app/
├── layout.tsx            ← khung chung mọi trang (nav, footer)
├── page.tsx              ← trang chủ "/"
├── classes/
│   ├── page.tsx          ← "/classes" — danh sách lớp
│   └── [id]/page.tsx     ← "/classes/12" — [id] là phần động
└── api/classes/route.ts  ← API backend "/api/classes"
```

Quy ước tên file: `page.tsx` = trang, `layout.tsx` = khung, `route.ts` = API, folder `[tên]` = tham số động.

## 6. Server Component vs Client Component {#6}

Điểm đặc biệt của Next.js 15:

- **Server Component** (mặc định): chạy trên server, gửi HTML về — tốt cho phần cần đọc DB (danh sách lớp). Không dùng `useState`.
- **Client Component** (thêm `'use client'` đầu file): chạy trong trình duyệt — cần khi có tương tác (form, nút bật mic, timer).

**Quy tắc trong dự án:** trang lớn (danh sách lớp) = Server Component; phần bấm được (nút "Tạo lớp", modal) = Client Component. Không lạm dụng `'use client'` — càng nhiều client component, trình duyệt càng nặng.

## 7. Tailwind CSS {#7}

Viết style ngay trong class thay vì file CSS riêng:

```tsx
<div className="flex items-center gap-2 rounded-lg border border-neutral-800 bg-neutral-950 p-4">
```

- `flex items-center` = bố cục hàng ngang, căn giữa
- `rounded-lg border` = bo góc + viền (style Vercel: viền mảnh 1px, nền tối)
- Học theo nhu cầu: tra tên class khi cần, đừng học thuộc.

## 8. shadcn/ui {#8}

Bộ component làm sẵn (Button, Dialog, Input, Table...) theo style tối giản — **không phải thư viện cài vào dùng mù mờ, mà là code component copy vào dự án của bạn** (`npx shadcn@latest add button`), muốn sửa gì cũng sửa được.

**Trong dự án:** mọi nút, ô input, dialog đều dùng shadcn để giao diện đồng nhất phong cách Vercel.

## 9. TanStack Query (React Query) {#9}

Thư viện quản lý dữ liệu từ server trong client: tự cache, tự gọi lại khi dữ liệu cũ, quản trạng thái đang tải/lỗi.

```tsx
const { data, isLoading } = useQuery({
  queryKey: ['classes'],
  queryFn: () => fetch('/api/classes').then(r => r.json()),
});
```

**Trong dự án:** sau khi tạo lớp xong, gọi `queryClient.invalidateQueries(['classes'])` → danh sách lớp tự cập nhật, không cần tự viết logic refetch.

## 10. Zustand {#10}

Thư viện state **toàn cục** (dùng chung nhiều component không cần truyền props xuyên suốt). Dùng cho state phòng học: bật/tắt cam mic, danh sách tay giơ, trạng thái khóa phòng — nhiều component cùng đọc/sửa.

```tsx
const micOn = useRoomStore((s) => s.micOn);
const toggleMic = useRoomStore((s) => s.toggleMic);
```

> Phân biệt: dữ liệu **từ server** (lớp, bài tập) → TanStack Query. State **tương tác trong phòng** → Zustand. State nhỏ trong 1 component → `useState`.
