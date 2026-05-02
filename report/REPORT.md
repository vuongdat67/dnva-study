# BÁO CÁO ĐỒ ÁN A12 – PENTEST VÀ BẢO MẬT DVNA

> *Lưu ý: Vui lòng bổ sung thông tin cá nhân (họ tên, MSSV, lớp, GVHD) và thông tin triển khai thực tế (domain, IP, VPS) nếu cần nộp bản chính thức.*

---

## 1. Giới thiệu

**DVNA (Damn Vulnerable NodeJS Application)** là một ứng dụng web mẫu có nhiều lỗ hổng OWASP Top 10, phục vụ cho việc học tập pentest và secure coding. Mục tiêu của đồ án A12:

- Chạy được ứng dụng web từ mã nguồn được cung cấp.
- Thực hiện pentest, khai thác lỗ hổng.
- Thực hiện bảo mật (fix) để tạo ra phiên bản an toàn.
- So sánh trước / sau khi fix.
- Triển khai (deploy) ứng dụng lên Internet.
- Viết báo cáo đầy đủ, rõ ràng.

---

## 2. Môi trường và công cụ

**Stack:**
- NodeJS + Express + EJS
- Sequelize + MySQL
- Passport.js (auth)

**Công cụ:**
- Docker / Docker Compose
- Burp Suite, DevTools
- curl, sqlmap (tương lai)

**Cấu hình mô phỏng:**
- App port: 9090
- MySQL: 3306 (nội bộ container)
- Biến môi trường trong `vars.env`

---

## 3. Kiến trúc hệ thống

```
[Browser] -> [Express App] -> [Sequelize ORM] -> [MySQL]
                    |-> [EJS Views]
```

**Thành phần chính:**
- `server.js`: khởi tạo Express, session, middleware
- `routes/`: định tuyến `/login`, `/usersearch`, `/ping`, `/products`, ...
- `core/`: xử lý logic (appHandler, authHandler)
- `models/`: ORM (User, Product, PassReset)
- `views/`: giao diện EJS

---

## 4. Recon và Mapping Endpoint

Đã map endpoint từ source + truy vết request:

**Auth routes:**
- `GET/POST /login`
- `GET/POST /register`
- `GET/POST /forgotpw`
- `GET/POST /resetpw`
- `GET /learn`

**App routes (yêu cầu login):**
- `GET/POST /usersearch` (SQLi)
- `GET/POST /ping` (Command Injection)
- `GET/POST /products` (SQLi)
- `GET/POST /bulkproducts` (XXE)
- `GET/POST /modifyproduct` (IDOR)
- `GET/POST /useredit` (IDOR)
- `GET /admin/usersapi` (Sensitive Data Exposure)
- `GET /redirect` (Open Redirect)

---

## 5. Phân tích lỗ hổng & giải pháp bảo mật (A1 – A6)

### A1 – Injection (SQLi) – `/usersearch`

**Mô tả:**
- Query được ghép chuỗi trực tiếp từ input.
- Có thể bypass auth / lấy dữ liệu bằng payload SQL.

**Vị trí:** `core/appHandler.js` – hàm `userSearch`

**Payload ví dụ:**
```
' OR 1=1 --
```

**Tác động:**
- Truy vấn dữ liệu người dùng, leak thông tin.

**Fix:**
- Dùng ORM/parameterized query (Sequelize `findOne`)

**Kết quả sau fix:**
- Payload SQLi không còn tác dụng.

---

### A1 – Injection (Command Injection) – `/ping`

**Mô tả:**
- Truyền input vào `exec()` để chạy lệnh hệ thống.

**Vị trí:** `core/appHandler.js` – hàm `ping`

**Payload ví dụ:**
```
127.0.0.1; cat /etc/passwd
```

**Tác động:**
- Thực thi lệnh tùy ý, chiếm quyền hệ thống.

**Fix:**
- Đổi sang `execFile()` và truyền tham số an toàn.

**Kết quả sau fix:**
- Chuỗi mở rộng lệnh bị vô hiệu.

---

### A2 – Broken Authentication – `/forgotpw` + `/resetpw`

**Mô tả:**
- Token reset password được tạo bằng `md5(login)` → đoán được.
- Reset password dựa vào `login` và `token` trong query.

**Tác động:**
- Đổi mật khẩu người khác nếu biết login.

**Fix:**
- Tạo token ngẫu nhiên (`crypto.randomBytes`)
- Hash token trong DB (sha256)
- Lưu trong bảng `PassReset` có thời gian hết hạn
- Gửi link qua email

**Kết quả sau fix:**
- Token không đoán được.
- Token hết hạn sau 15 phút.

---

### A3 – Sensitive Data Exposure – `/admin/usersapi`

**Mô tả:**
- API trả về toàn bộ user record (bao gồm password hash).

**Fix:**
- Giới hạn field trả về: `id`, `name`, `email`

---

### A4 – XXE – `/bulkproducts`

**Mô tả:**
- XML parser cho phép external entity (`noent=true`)
- Có thể đọc file nội bộ hoặc SSRF.

**Payload ví dụ:**
```xml
<!DOCTYPE data [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
```

**Fix:**
- Tắt `noent` (không resolve entity) khi parse XML.

---

### A5 – Broken Access Control – `/admin/usersapi`, `/useredit`

**Mô tả:**
- User thường có thể truy cập API admin.
- User có thể sửa thông tin người khác bằng ID.

**Fix:**
- Thêm middleware `requireRole('admin')`
- Kiểm tra `req.user.id` hoặc `role` trước khi sửa.

---

### A6 – Security Misconfiguration

**Mô tả:**
- Session secret hard-code.
- Express để lộ thông tin `X-Powered-By`.
- Logging quá nhiều thông tin.

**Fix:**
- `SESSION_SECRET` lấy từ environment.
- `app.disable('x-powered-by')`
- Sequelize `logging: false`
- `NODE_ENV=production`

---

## 6. So sánh trước / sau

| Nội dung | Phiên bản V1 | Phiên bản V2 |
|---|---|---|
| SQLi /usersearch | Có thể khai thác | Đã fix |
| Command Injection /ping | Có thể thực thi lệnh | Đã fix |
| Reset password | Token đoán được | Token ngẫu nhiên + hết hạn |
| Sensitive data API | Leak password hash | Chỉ trả về field cần thiết |
| XXE | Đọc file / SSRF | Tắt entity |
| Access control | IDOR / admin leak | Check role + ownership |
| Security headers | Lộ thông tin x-powered-by | Đã tắt |

---

## 7. Tự động hóa CI bằng GitHub Actions

**Mục tiêu:** Tự động kiểm tra chất lượng và build image khi có thay đổi mã nguồn.

**Các bước trong pipeline:**
- `npm install`
- `npm run lint --if-present` và `npm test --if-present`
- `docker build`
- `docker push` lên GHCR (hoặc Docker Hub)
- `trivy` quét lỗ hổng image

**Lưu ý cấu hình:**
- Nếu dùng GHCR: dùng `GITHUB_TOKEN` mặc định.
- Nếu dùng Docker Hub: cần `DOCKERHUB_USERNAME` và `DOCKERHUB_TOKEN` trong secrets.

---

## 8. Kết luận và bài học

- Ứng dụng web cần được thiết kế theo secure coding từ đầu.
- Không bao giờ xử lý input trực tiếp vào SQL hoặc OS command.
- Reset password là chức năng nhạy cảm, phải dùng token ngẫu nhiên và hết hạn.
- Phân quyền (role/ownership) là bắt buộc để ngăn IDOR.
- Bảo mật không chỉ là fix code, mà còn là cấu hình hệ thống (ENV, header, logging).

---

## 9. Tài liệu tham khảo

- OWASP Top 10 (2021)
- DVNA (appsecco)
- NodeJS/Express security best practices