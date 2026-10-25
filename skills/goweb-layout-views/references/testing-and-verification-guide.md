# Quy Trình Kiểm Thử & Xác Minh (Testing & Verification Guide)

Tài liệu này cung cấp quy trình và bộ lệnh kiểm thử toàn diện cho các thành phần Layouts, Views, HTMX, CSRF, Helmet và Options trong dự án `sunway_goweb`. Khi được yêu cầu **"test skill"** hoặc **"kiểm tra mã nguồn"**, hãy thực hiện tuần tự theo quy trình dưới đây.

---

## 1. Cấp Độ 1: Kiểm Tra Cú Pháp & Biên Dịch (Compilation & Linting)

Đây là bước bắt buộc đầu tiên trước khi chạy bất kỳ kiểm thử nào khác.

### 1.1. Sinh Mã Template (`templ generate`)
Đảm bảo tất cả file `.templ` cú pháp hợp lệ và đồng bộ với mã nguồn Go:

```bash
templ generate
# Hoặc dùng makefile:
make gen
```
* **Kỳ vọng**: Lệnh kết thúc thành công (exit code 0), không có báo lỗi cú pháp HTML hay Go expressions.

### 1.2. Kiểm Tra Biên Dịch Toàn Dự Án (`go vet` & `go test`)
Kiểm tra tính tương thích giữa handler, usecase, composer interface và template:

```bash
go vet ./...
go test ./... -run=^$ # Dry-run kiểm tra compilation của toàn bộ packages
```
* **Kỳ vọng**: Không có lỗi thiếu method trong interface (ví dụ `composerPages`, `composerBrands`), không có lỗi type mismatch.

---

## 2. Cấp Độ 2: Kiểm Thử Bảo Mật & Headers (Helmet & CSRF)

Kiểm thử xem các middleware trong `ServerInternal` hoạt động chính xác.

### 2.1. Kiểm Tra Headers Của Helmet
Gửi request GET kiểm tra response headers:

```bash
curl -I http://localhost:8080/
```
* **Kỳ vọng xuất hiện các headers sau**:
  ```http
  X-Frame-Options: SAMEORIGIN
  X-Content-Type-Options: nosniff
  Referrer-Policy: strict-origin-when-cross-origin
  Cross-Origin-Resource-Policy: cross-origin
  Set-Cookie: csrf_=...; Path=/; SameSite=Lax
  ```

### 2.2. Kiểm Tra Cơ Chế Chặn CSRF (Khi Thiếu Token)
Thử gửi request POST lên một endpoint yêu cầu CSRF mà **không gửi header `X-CSRF-Token`**:

```bash
curl -X POST http://localhost:8080/brands/results -d "page=2"
```
* **Kỳ vọng**: Server trả về mã **`403 Forbidden`** với JSON:
  ```json
  {
    "code": 403,
    "status": "Forbidden",
    "message": "Invalid or missing CSRF token. Please refresh the page and try again."
  }
  ```

### 2.3. Kiểm Tra Cơ Chế Chấp Nhận CSRF (Khi Có Token Hợp Lệ)
Lấy cookie `csrf_` và đính kèm vào header:

```bash
# 1. Lấy token từ cookie
TOKEN=$(curl -sI http://localhost:8080/ | grep -i 'Set-Cookie: csrf_=' | sed -E 's/.*csrf_=([^;]+).*/\1/')

# 2. Gửi POST kèm token
curl -X POST http://localhost:8080/brands/results \
  -H "X-CSRF-Token: $TOKEN" \
  -H "Cookie: csrf_=$TOKEN" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "page=1"
```
* **Kỳ vọng**: Server trả về mã **`200 OK`** chứa nội dung HTML fragment của danh sách.

---

## 3. Cấp Độ 3: Kiểm Thử Luồng HTMX (Fragment & URL Push)

Áp dụng cho các tính năng danh sách như `news`, `brands`, `products`:

### 3.1. Kiểm Tra Trang Tải Đầu (Full-Page GET)
```bash
curl -s http://localhost:8080/brands | head -n 30
```
* **Kỳ vọng**: Chứa đầy đủ thẻ `<!DOCTYPE html>`, `<html lang="en">`, `<head>`, `@headers.Header`, thẻ `<script src=".../htmx.min.js">`, và footer.

### 3.2. Kiểm Tra Endpoint HTMX AJAX (POST `/.../results`)
Gửi request kèm token CSRF:

```bash
curl -i -X POST http://localhost:8080/brands/results \
  -H "X-CSRF-Token: $TOKEN" \
  -H "Cookie: csrf_=$TOKEN" \
  -H "HX-Request: true" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "page=2"
```
* **Kỳ vọng**:
  1. Header response có: **`HX-Push-Url: /brands?page=2`**.
  2. Nội dung trả về **CHỈ LÀ HTML FRAGMENT** (`BrandResults`), tuyệt đối **KHÔNG** chứa thẻ `<html>`, `<head>`, `<body>`, hay footer.

---

## 4. Cấp Độ 4: Kiểm Thử Banner & Option Group / Option Value

### 4.1. Kiểm Tra Tầng Fallback (Khi CMS Chưa Có Dữ Liệu)
* Xóa hoặc trỏ key banner về một key không tồn tại trong DB.
* Tải lại trang.
* **Kỳ vọng**: Trang vẫn hiển thị banner mặc định với thẻ `<picture>`, `<source media="...">`, text và ảnh `.webp` fallback. Trang không bị trắng hay vỡ khung.

### 4.2. Kiểm Tra Tầng Dynamic (Khi CMS Có Dữ Liệu)
* Đảm bảo CMS trả về chuỗi HTML của banner (vd: `<div class="custom_banner">...</div>`).
* Tải lại trang.
* **Kỳ vọng**: View render chính xác đoạn HTML từ CMS thông qua `@templ.Raw(...)`.

---

## 5. Cấp Độ 5: Kiểm Thử Trên Trình Duyệt (DevTools Verification)

Khi chạy ứng dụng trên trình duyệt:

1. **Kiểm tra Network Tab**:
   - Khi click phân trang hoặc đổi bộ lọc sort: Chỉ có **1 request POST** gửi tới `/results`.
   - Request Header chứa: `X-CSRF-Token: <uuid>`.
   - Response Payload là HTML fragment (size nhỏ < 10KB), không phải toàn bộ trang HTML.
   - Thanh địa chỉ URL của trình duyệt tự động cập nhật (vd: `/news?orderBy=oldest&page=2`) mà không gây chớp/reload trang.
2. **Kiểm tra Console Tab**:
   - Không xuất hiện bất kỳ lỗi màu đỏ nào:
     - Không có lỗi `Refused to connect... because it violates Content Security Policy`.
     - Không có lỗi `403 Forbidden` do thiếu CSRF.
     - Không có lỗi cú pháp JavaScript hay template.
