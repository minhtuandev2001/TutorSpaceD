# 01 — Web, HTTP & API cơ bản

> Mục lục: [Web hoạt động thế nào](#1-web-hoạt-động-thế-nao) • [Frontend vs Backend](#2) • [HTTP](#3) • [API & REST](#4) • [JSON](#5) • [Cookie vs JWT vs Session](#6) • [HTTPS](#7) • [CORS](#8)

## 1. Web hoạt động thế nào? {#1-web-hoạt-động-thế-nao}

Web = **client** (trình duyệt của bạn) xin dữ liệu, **server** (máy tính ở đâu đó) trả dữ liệu.

```
Trình duyệt (client)  ── xin: "cho tôi trang chủ" ──►  Server
Trình duyệt           ◄─ trả: HTML/CSS/JS + dữ liệu ──  Server
```

**Trong TutorSpaceD:** khi bạn mở trang lịch giảng dạy, trình duyệt xin server: "cho lịch tuần này của thầy Minh", server đọc trong database rồi trả về danh sách buổi học.

## 2. Frontend vs Backend {#2}

| | Frontend | Backend |
|---|---|---|
| Là gì | Phần người dùng nhìn thấy + bấm được | Phần "đầu bếp": xử lý logic, lưu dữ liệu |
| Chạy ở đâu | Trình duyệt của người dùng | Server (máy chủ) |
| Trong dự án | React/Next.js: giao diện lịch, form giao bài | API Routes: kiểm tra quyền, lưu bài nộp |

## 3. HTTP {#3}

Ngôn ngữ client–server nói chuyện. Mỗi lần nói = 1 **request** gồm:
- **Method** (hành động gì): `GET` (lấy), `POST` (tạo), `PATCH` (sửa), `DELETE` (xóa)
- **URL** (đối tượng nào): `/api/classes/12`
- **Headers** (thông tin thêm): đăng nhập ai, định dạng gì
- **Body** (nội dung gửi kèm, với POST/PATCH)

**Trong dự án:** bấm "Tạo lớp" → trình duyệt gửi `POST /api/classes` với body `{"name":"Toán 9-A","color":"blue"}`. Server trả `201 Created` + dữ liệu lớp vừa tạo.

Mã trạng thái hay gặp:
| Mã | Nghĩa |
|---|---|
| 200 / 201 | OK / tạo thành công |
| 400 | Client gửi sai (thiếu tên lớp...) |
| 401 | Chưa đăng nhập |
| 403 | Đã đăng nhập nhưng không có quyền (học sinh cố tạo lớp) |
| 404 | Không tồn tại |
| 500 | Lỗi của server |

## 4. API & REST {#4}

**API** = cửa sổ mà server mở ra cho client gọi. **REST** = quy ước đặt tên API theo tài nguyên + method HTTP:

| Hành động | Method + URL |
|---|---|
| Xem danh sách lớp | `GET /api/classes` |
| Tạo lớp | `POST /api/classes` |
| Sửa lớp 12 | `PATCH /api/classes/12` |
| Xóa lớp 12 | `DELETE /api/classes/12` |
| Xem bài nộp của HS 5 trong bài 8 | `GET /api/assignments/8/submissions?studentId=5` |

## 5. JSON {#5}

Định dạng dữ liệu chữ mà cả JS client và server đều hiểu:

```json
{
  "id": 12,
  "name": "Toán 9-A",
  "color": "blue",
  "students": [{"id": 1, "fullName": "An"}]
}
```

## 6. Cookie vs Session vs JWT {#6}

Vấn đề: HTTP không nhớ ai đang gọi (mỗi request là người lạ). Cần "thẻ ăn sáng":

- **Cookie:** server gửi 1 thẻ, trình duyệt tự mang theo ở mọi request sau đó.
- **Session:** thẻ chỉ chứa mã số, dữ liệu thật nằm ở server. Bảo mật cao, server phải nhớ.
- **JWT (JSON Web Token):** thẻ là 1 chuỗi tự chứa thông tin (ai, vai trò gì, hết hạn khi nào), có chữ ký chống giả mạo. Server không cần nhớ gì.

**Trong dự án:** Auth.js dùng **session JWT**. Khi học sinh đăng nhập xong, mỗi lần gọi API có kèm token → server đọc token biết "đây là An, vai trò student, vào được lớp 12".

## 7. HTTPS {#7}

HTTP có mã hóa (tiếng Việt: HTTP có "S" = Secure). Bắt buộc khi có đăng nhập — mật khẩu không được đi dạng văn thường. Tự động có khi deploy trên Vercel; tự tạo bằng Let's Encrypt trên VPS.

## 8. CORS {#8}

Trình duyệt mặc định **chặn** request từ trang A gọi API của server B (chống hack). Server phải nói "cho phép A gọi" qua header. Trong dự án ta chủ yếu dùng cùng domain (Next.js) nên hiếm khi gặp; sẽ gặp khi frontend gọi thẳng LiveKit server → cần cấu hình CORS/allowed origins ở đó.
