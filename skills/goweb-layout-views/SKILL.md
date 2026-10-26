---
name: goweb-layout-views
description: >-
  Use this skill when creating, modifying, or refactoring layouts, views, partials,
  components, or integrating templates with Fiber HTTP handlers in the sunway_goweb project.
---

# Quy Trình Xử Lý Layout & Views (sunway_goweb)

Tài liệu này cung cấp hướng dẫn toàn diện và quy chuẩn chuẩn xác cho việc phát triển giao diện (layouts, views, components, partials) bằng **a-h/templ** kết hợp với **GoFiber** trong dự án `sunway_goweb`.

---

## 1. Tổng Quan Kiến Trúc (Architecture Overview)

Dự án sử dụng bộ công nghệ kết hợp:
* **Template Engine**: `github.com/a-h/templ` (phiên bản v0.3.x).
* **Template Helper**: `github.com/networld-solution/gos/templates` (`Render`, `AssetURL`, `SafeURL`).
* **Web Framework**: GoFiber v2 (`github.com/gofiber/fiber/v2`).
* **Service Framework**: `github.com/networld-solution/sctx` (Service Context & Components).
* **Live Reload & Code Generation**: `templ generate` & `air` (`.air.server.toml`).

### Cấu Trúc Thư Mục `views/`

```text
views/
├── layouts/                     # Layouts dùng chung toàn bộ ứng dụng
│   ├── master.templ             # Layout chính (HTML5 skeleton, head, header, children, footer, scripts)
│   ├── svg.templ                # Sprite SVG icons toàn cục (@SvgIcon())
│   ├── headers/                 # Header components
│   │   ├── header.templ         # Header wrapper (@Header())
│   │   ├── header_top_login.templ # Thanh topbar người dùng / đăng nhập
│   │   ├── menu.templ           # Desktop navigation & megamenu
│   │   └── menus_mobile.templ   # Mobile navigation drawer
│   └── footers/
│       └── footer.templ         # Footer chính (@Footer())
├── partials/                    # Reusable UI widgets dùng ở nhiều module
│   ├── breadcrumb.templ         # Breadcrumb navigation
│   ├── pagination.templ         # Pagination controls
│   ├── product.templ            # Card sản phẩm tiêu chuẩn (@Product())
│   ├── catalogue.templ          # Card catalogue (@Catalogue())
│   └── category.templ           # Card category (@Category())
└── <feature>/                   # Các trang theo tính năng (Feature Pages)
    ├── homes/                   # Trang chủ
    │   ├── index.templ          # PageHome()
    │   └── components/          # Components riêng của trang chủ (slide, flash sale, faqs...)
    ├── products/                # Trang danh sách sản phẩm
    │   ├── index.templ          # PageProducts()
    │   └── components/          # Components lọc sản phẩm (product_filter.templ)
    ├── catalogues/              # Trang danh mục catalogue
    │   └── index.templ          # PageCatalogues()
    └── auths/                   # Trang xác thực (login, register)
        ├── login.templ          # LoginPage()
        └── register.templ       # RegisterPage()
```

### Cấu Trúc Transport Backend (`modules/web/transport/`)

Dự án phân định rạch ròi 2 tầng xử lý:

```text
modules/web/transport/
├── handlers/                    # Page Handlers (SSR Full Pages)
│   │                            # - Chuyên phục vụ route GET hiển thị trang người dùng
│   │                            # - Render toàn bộ trang HTML (@layouts.Master(...))
│   │                            # - KHÔNG BAO GIỜ mang tiền tố route /api/
│   │                            # - Ví dụ: /news, /products, /brands, /:slug.html
│   ├── news_hdl.go              # ListNewsHdl(), NewsDetailHdl()
│   ├── products_hdl.go          # ProductsPageHdl()
│   └── ...
└── apis/                        # API Handlers (HTMX Fragments & JSON APIs)
    │                            # - Chuyên phục vụ HTMX AJAX (HTML fragments) & JSON APIs
    │                            # - BẮT BUỘC route phải bắt đầu bằng /api/
    │                            # - Ví dụ: /api/news/results, /api/products/partial, /api/promotions/active-sale
    │                            # - Không chia nhỏ API sort/pagination, gom chung vào 1 handler Results
    ├── news_api.go              # NewsResultsApi() (POST /api/news/results), ListNewsApi() (POST /api/news)
    ├── popup_sales_api.go       # ActiveSaleApi() (POST /api/promotions/active-sale)
    └── ...
```

### Quy Tắc Tổ Chức `routes_web.go` (Feature-Grouped & Auth-Separated)

Trong file `modules/web/routes_web.go`, toàn bộ route phải được tổ chức theo 2 nguyên tắc cốt lõi:

1. **Phân Định Tuyệt Đối 2 Khối Lớn (Public vs Authenticated)**:
   * **Khối 1: PUBLIC ROUTES (`app.Get`, `app.Post`)**: Phục vụ tất cả người dùng không yêu cầu đăng nhập.
   * **Khối 2: AUTHENTICATED ROUTES (`appLogin := swmidd.NewAuthRouter(app, midds...)`)**:
     - Chỉ chứa các route bắt buộc đăng nhập (bảo vệ bởi Auth Middleware).
     - Mọi route trong này đều dùng `appLogin.Get(...)`, `appLogin.Post(...)`...
     - Tuyệt đối **không khai báo `app.` bên trong khối `appLogin`**.

2. **Gom Nhóm Các Route Liên Quan Gần Nhau Theo Tính Năng (Feature Grouping)**:
   * Trong từng khối (Public hoặc Authenticated), **tất cả các route thuộc cùng một module/tính năng phải được đặt liền kề nhau** thành từng cụm có comment tiêu đề rõ ràng (ví dụ: `--- Products ---`, `--- News ---`, `--- Brands ---`, `--- Checkout & Payment ---`).
   * **Lợi ích**: Giúp dễ nhìn, dễ tìm; khi phát triển hoặc debug tính năng nào thì toàn bộ route liên quan (trang GET, HTMX AJAX fragment, JSON API) đều nằm tập trung một chỗ.
   * **Ví dụ mẫu cụm News**:
     ```go
     // --- News ---
     app.Get("/news", newsComp.ListNewsHdl())
     app.Get("/news/:pagepath<regex(^([a-z0-9-]+)-([0-9]+)$)>.html", newsComp.NewsDetailHdl())
     app.Post("/api/news/results", apiNewsComp.NewsResultsApi())
     app.Post("/api/news", apiNewsComp.ListNewsApi())
     ```

---

## 2. Các Nguyên Tắc Bất Di Bất Dịch (Golden Rules)

1. **TUYỆT ĐỐI KHÔNG SỬA FILE `*_templ.go`**:
   - Mọi file `*_templ.go` đều do lệnh `templ generate` tự động sinh ra.
   - Luôn luôn chỉnh sửa file nguồn `.templ`.
   - Sau khi sửa, chạy `make gen` hoặc `templ generate`.

2. **Xử Lý Static Assets**:
   - Tất cả đường dẫn CSS, JS, hình ảnh, favicon trong file `.templ` **BẮT BUỘC** đi qua `templates.AssetURL("/static/...")`.
   - URL an toàn cho thẻ `<a>` cần đi qua `templates.SafeURL("/duong-dan")` khi sinh động từ dữ liệu hoặc cấu hình.

3. **SVG Icons**:
   - Mọi icon dùng chung nằm trong `views/layouts/svg.templ`.
   - Gọi biểu tượng bằng thẻ `<svg><use xlink:href="#<icon-id>"></use></svg>`.

4. **Package Naming trong Views**:
   - `views/layouts/master.templ` -> `package layouts`
   - `views/layouts/headers/*.templ` -> `package headers`
   - `views/layouts/footers/*.templ` -> `package footers`
   - `views/partials/*.templ` -> `package partials`
   - `views/<feature>/index.templ` -> `package <feature>` (ví dụ `package homes`, `package products`)
   - `views/<feature>/components/*.templ` -> `package components`

5. **Quy Chuẩn CSS Responsive (`rem`, `em`, `font-size`)**:
   - Gốc tỷ lệ: `html { font-size: 62.5%; }` => `1rem = 10px` ở desktop.
   - Thẻ bao ngoài component đặt `font-size: 1rem;` để neo theo root scale (tự động co về 8px ở 992-1199px và 7px ở 768-991px).
   - Mọi thuộc tính nội bộ component (`padding`, `gap`, `border-radius`, `width/height`, cỡ chữ con) **bắt buộc dùng `em`** để co giãn đồng bộ.
   - Tiêu đề mobile dùng `clamp(1.7em, 5.2vw, 2.2em)`.
   - Xem chi tiết tại [Hướng Dẫn CSS Responsive](./references/responsive-css-guide.md).

6. **Quy Tắc Bọc `<span>` Cho Button / Thẻ Để Bảo Toàn Chiều Cao (`height`) Khi Đặt `font-size`**:
   - **Bản chất kỹ thuật**: Trong CSS, đơn vị `em` trên một phần tử sẽ tính toán kích thước (`height`, `width`, `padding`, `border-radius`) dựa trên `font-size` của **chính phần tử đó**. Nếu đặt `height: 4.4em` (kỳ vọng 44px) cùng lúc với `font-size: 1.5em` (15px) trên cùng một thẻ `.alert_btn`, chiều cao sẽ bị phóng đại thành `4.4 * 1.5em = 6.6em = 66px`.
   - **Quy chuẩn bắt buộc**:
     - Thẻ cha (`.alert_btn`, `.login_submit`, `.register_submit`) giữ `height: 4.4em;` hoặc `4.8em;` (tính theo font-size kế thừa 10px).
     - Luôn bọc chữ bên trong bằng thẻ `<span>`: `<a href="/login" class="alert_btn"><span>Login</span></a>` hoặc `<button class="register_submit"><span>Submit</span></button>`.
     - Đặt `font-size: 1.5em;` trên thẻ con `.alert_btn > span`.

7. **Quy Tắc Bắt Buộc Đặt `font-size: 1rem;` Trên Thẻ Form (`<button>`, `<input>`, `<select>`, `<textarea>`)**:
   - **Bản chất kỹ thuật**: Mặc định trình duyệt (User Agent Stylesheet) không cho phép các thẻ form tự kế thừa font-size thẻ cha, mà luôn ép cỡ chữ `13.333px`.
   - **Hậu quả nếu thiếu**: Toàn bộ thuộc tính `em` trên nút (`height: 4.4em`, `padding: 0 2.4em`, `border-radius: 0.8em`) sẽ bị nhân với `13.333px` thay vì `10px`, khiến nút phình to ~133% (cao `58.6px` thay vì `44px`).
   - **Quy chuẩn bắt buộc**: Luôn khai báo `font-size: 1rem;` trên các class form elements:
     ```css
     .btn_faq_load_more {
         font-size: 1rem;      /* BẮT BUỘC: Neo về root 10px thay vì 13.333px của browser */
         height: 4.4em;        /* Đúng chuẩn 44px (4.4 * 10px) */
         padding: 0 2.4em;     /* Đúng chuẩn 24px */
         border-radius: 0.8em; /* Đúng chuẩn 8px */
     }
     .btn_faq_load_more > span {
         font-size: 1.4em;     /* 14px */
     }
     ```

8. **Quy Chuẩn Đặt Tên Phương Thức UseCase & Handler (`*Usc`, `*Hdl`)**:
   - **Phương thức UseCase** (trong `modules/web/usecase/`) phục vụ cho Web Handler **BẮT BUỘC** kết thúc bằng hậu tố `Usc`:
     - Ví dụ: `ListBrandUsc`, `ListNewsUsc`, `GetProductDetailUsc`...
     - Tuyệt đối không đặt tên thiếu hậu tố (như `ListNews`, `ListBrand`).
   - **Interface UseCase** khai báo cục bộ tại Handler (`modules/web/transport/handlers/<feature>_hdl.go`) phải khớp tên phương thức có hậu tố `*Usc`:
     ```go
     type newsUsc interface {
         ListNewsUsc(ctx context.Context, req *requests.NewsReq) (*responses.NewsListResp, error)
     }
     ```
   - **Phương thức Handler** trả về `fiber.Handler` **BẮT BUỘC** kết thúc bằng hậu tố `Hdl`:
     - Ví dụ: `ListNewsHdl()`, `IndexHdl()`, `PageOrdersHdl()`.

9. **Middleware Bảo Mật (`ServerInternal`), Helmet, CSRF & Tự Động Hóa Với HTMX**:
   - **Helmet**: Sử dụng cấu hình chuẩn tại `ServerInternal`: `XSSProtection: "0"` (chuẩn OWASP), `ContentTypeNosniff: "nosniff"`, `XFrameOptions: "SAMEORIGIN"`, `ReferrerPolicy: "strict-origin-when-cross-origin"`, `CrossOriginResourcePolicy: "cross-origin"` (đảm bảo static assets/webp không bị chặn cross-origin).
   - **CSRF Token tự động cho HTMX**: HTMX đã được nạp toàn cục tại `master.templ` trước `app.js`. `app.js` có listener `htmx:configRequest` tự động lấy cookie `csrf_` gán vào header `X-CSRF-Token` cho toàn bộ request HTMX. Lập trình viên **không cần cấu hình thủ công header CSRF** khi viết code HTMX.
   - **Lưu ý CSP**: Nếu bật `ContentSecurityPolicy` trong Helmet, **bắt buộc** có chỉ thị `connect-src 'self'` (cho AJAX HTMX) và `script-src 'self' 'unsafe-inline'`.

10. **Quy Tắc Phân Định Tuyệt Đối: Route GET (Pages) $\rightarrow$ `handlers/`, Route `/api/...` $\rightarrow$ `apis/`**:
    - **Route GET hiển thị trang cho người dùng** (không có tiền tố `/api/`, ví dụ: `/news`, `/products`, `/brands`, `/about-us`, `/contact`): **Bắt buộc nằm trong `modules/web/transport/handlers/`**, đảm nhận render toàn bộ trang HTML (`layouts.Master(...)`).
    - **Mọi route bắt đầu bằng `/api/...`** (bao gồm cả HTMX AJAX fragments như `POST /api/news/results`, `GET /api/products/partial`, `POST /api/promotions/active-sale` và JSON APIs): **Bắt buộc nằm trong `modules/web/transport/apis/`**.
    - **Nguyên tắc phân chia rõ ràng**: Tuyệt đối không để route của `handlers/` mang tiền tố `/api/`, và ngược lại handler của `apis/` luôn luôn phải được đăng ký ở nhóm route có tiền tố `/api/`.

11. **Quy Chuẩn Active Menu & Navigation (Desktop, Mobile & Sub-items)**:
    - **Không hardcode class `active` tĩnh trong file `.templ`**: Tránh tình trạng một menu luôn luôn bị gạch chân active dù người dùng đang ở trang khác.
    - **Cơ chế kích hoạt tự động qua `app.js` (`activeNavigation`)**:
      - Tự động so khớp URL hiện tại (`pathname`, query string `?category=...`, `?country=...`, `?brand=...`).
      - Hỗ trợ nhận diện các trang con / chi tiết (Regex Product Detail `/:slug-:id.html`, News Detail `/news/:slug.html`...).
      - **Đồng bộ đa cấp (Parent + Child)**: Khi bất kỳ sub-item nào trong `.menu_sub` hoặc `.m_sub` được active, dropdown cha tương ứng bắt buộc cũng phải được active (`.menu_item.active`, `.m_nav_item.active`).
      - **Phản ứng với Lịch sử & HTMX**: Lắng nghe sự kiện `popstate` và `htmx:pushedIntoHistory` để tự động cập nhật active menu khi người dùng duyệt lịch sử hoặc lọc dữ liệu qua HTMX.

12. **Quy Chuẩn Comment Docblock Cho Handlers & APIs (`handlers/` & `apis/`)**:
    - Mọi function handler và API bắt buộc có docblock comment chi tiết phía trước theo mẫu chuẩn:
      ```go
      /**
       * Method: POST | GET | PUT | DELETE
       * Route: /api/brands/results hoặc /brands
       * Description: Mô tả ngắn gọn chức năng của endpoint
       * Request: requests.BrandReq (BodyParser | QueryParser | Params)
       * Validate: true | false (Có gọi req.Validate() hay không)
       * Response: HTML Fragment (templates.Render) | Full Page HTML | JSON core.ResponseData(...)
       */
      ```

13. **Quy Chuẩn Viết UseCase (`usecase/`) & Giới Hạn Lồng Gọi Hàm Tối Đa 2 Cấp (Max 2 Levels Call Depth)**:
    - **Liệt kê tuần tự các bước 1, 2, 3...**: Trong UseCase, bắt buộc phải viết comment các bước thực hiện rõ ràng (`// Bước 1: ...`, `// Bước 2: ...`) để thể hiện mạch nghiệp vụ rõ ràng, dễ bảo trì, dễ review.
    - **Quy tắc lồng gọi hàm tối đa 2 cấp (Max 2 Levels Call Depth)**:
      - **Cấp 1**: Hàm UseCase chính (ví dụ `ListBrandUsc`).
      - **Cấp 2**: Các hàm helper, repository, hoặc mapper được gọi trực tiếp từ UseCase (ví dụ `usc.brandRepo.CountBrandRepo(...)`, `mapping.MapperHomeBrands(...)`).
      - **TUYỆT ĐỐI KHÔNG** lồng gọi sang cấp thứ 3 (không để hàm cấp 2 gọi tiếp hàm helper cấp 3, rồi cấp 3 gọi cấp 4). Toàn bộ luồng xử lý chính phải phẳng (flat workflow), tường minh ngay tại UseCase.

14. **Quy Chuẩn Cửa Ngõ Đầu Vào (Request Validation) & Error-Returning Validate Pattern**:
    - **Triết lý Sống Còn (Security & Data Integrity Gateway)**:
      - Tầng Request Validation là cửa ngõ sống còn bảo vệ toàn bộ hệ thống khỏi tấn công, input rác, bot crawler spam, SQLi và sai lệch dữ liệu.
      - Hàm `Validate(ctx context.Context) error` **bắt buộc phải có chữ ký trả về `error`**. Tuyệt đối **không viết hàm `Validate()` trả về `void`** rồi âm thầm nuốt lỗi hoặc tự ý ép kiểu fallback dễ dãi khi gặp input bất thường!
    - **Nguyên tắc Kiểm Tra Nghiêm Khắc (Strict Validation & Whitelist)**:
      - **Phân trang (`Page`)**: Nếu không truyền (`Page == 0`), gán mặc định `1`. Nếu truyền số âm (`Page < 0`), số 0 chủ đích, hoặc vượt trần chống Deep Pagination DDoS (`Page > 200`), **bắt buộc trả về lỗi** `errors.New("page exceeds maximum limit of 200")`.
      - **Sắp xếp (`OrderBy`, `Sort`) & Bộ lọc**: Bắt buộc kiểm tra qua Whitelist hằng số (ví dụ chỉ chấp nhận `"newest"`, `"oldest"`). Nếu client truyền chuỗi lạ, **bắt buộc trả về lỗi `Bad Request`**, không tự ý fallback mà giấu lỗi.
    - **Xử lý lỗi tại tầng Transport (Handler | API)**:
      - Khi `err := req.Validate(ctx); err != nil`:
        - **JSON API**: Trả về HTTP 400: `c.Status(fiber.StatusBadRequest).JSON(fiber.Map{ "code": 400, "error": err.Error(), "message": err.Error() })`.
        - **HTMX Fragment**: Trả về HTTP 400: `c.Status(fiber.StatusBadRequest).SendString(err.Error())`.
        - **Full Page SSR**: Trả về lỗi `fiber.NewError(fiber.StatusBadRequest, err.Error())` để render trang lỗi 400 có layout.
      - Tầng UseCase (`usc`) **chỉ nhận Request Struct đã vượt qua Validate thành công 100%**.
    - **Single Request Constraint**: Mỗi Handler hoặc API chỉ sử dụng DUY NHẤT 1 Request struct, không khai báo 2 request structs trong cùng một hàm.

15. **Quy Chuẩn Thiết Kế & Xử Lý Các Trang Lỗi (Error Pages: 400, 404 vs 500)**:
    - **Trang 400 (Bad Request) & 404 (Not Found) $\rightarrow$ DÙNG MASTER LAYOUT**:
      - Đây là lỗi do phía Client (người dùng bấm nhầm query string, sai page, hoặc link cũ).
      - Server và Database vẫn hoàn toàn khỏe mạnh. Do đó, **bắt buộc phải bọc trong `layouts.Master(...)`** để giữ nguyên Header, Menu, Giỏ hàng và Footer, giúp người dùng dễ dàng chuyển hướng tiếp tục duyệt web.
    - **Trang 500 (Internal Server Error) $\rightarrow$ ĐỘC LẬP (STANDALONE - KHÔNG DÙNG MASTER LAYOUT)**:
      - Đây là lỗi từ Server, Database hoặc mã nguồn nội bộ.
      - Nếu trang 500 dùng Master layout, khi Database bị sập, việc Master cố tải dữ liệu menu/giỏ hàng từ DB sẽ khiến trang 500 bị crash theo (gây màn hình trắng xóa). Do đó, trang 500 bắt buộc là file HTML độc lập, tối giản, tự chứa toàn bộ tài nguyên.
    - **Tách Biệt Web SSR vs API Trong `error_midd.go`**:
      - Nếu request bắt đầu bằng `/api/`: Trả về JSON `{ "code": code, "error": msg, "message": msg }`.
      - Nếu là request Web thông thường: Render trang HTML tương ứng (`errpages.BadRequest`, `errpages.NotFound`, `errpages.ServerError`).

16. **Quy Chuẩn Định Vị Lỗi & Báo Đúng Nguồn Gốc (Error Tracing & Origin Identification Pattern)**:
    - **Nguyên tắc Sống Còn (Báo Đúng Vị Trí & Ngữ Cảnh)**:
      - Khi lỗi xảy ra, hệ thống bắt buộc phải xác định chính xác 3 cấp độ nguồn gốc phát sinh lỗi:
        1. **Module**: Ví dụ `web`, `admin`, `api`...
        2. **Layer**: `Request` (Validate/Parse), `Transport` (Handler/API), `UseCase` (Nghiệp vụ), `Repository` (Cơ sở dữ liệu).
        3. **Action / Handler / Method Name**: Ví dụ `brandsHdl.IndexHdl`, `newsAPI.NewsResultsApi`, `newsReq.Validate`.
    - **Ghi Log Nội Bộ Bằng `sctx.Logger` (Server-side Trace Logging)**:
      - Tuyệt đối **không dùng package `log` chuẩn của Go**, bắt buộc inject và sử dụng `sctx.Logger` (`sctx.Logger.Errorf`):
        ```go
        h.logger.Errorf("[Module: web] [Action: brandsHdl.IndexHdl] usc.ListBrandUsc error: %v", err)
        ```
      - Giúp log được ghi nhận chuẩn xác theo ngày (`AppLoggerDaily`), đồng bộ với ELK/Sentry và truy vết file/action tức thì.

17. **Quy Chuẩn Cấm Nuốt Lỗi Ngầm & Bắt Buộc Kiểm Tra Lỗi Đầy Đủ (Zero Error-Swallowing & Explicit Error Propagation Pattern)**:
    - **Triết lý Sống Còn (Không Bao Giờ Giấu Lỗi Database / I/O)**:
      - Mọi hàm có thao tác I/O, truy vấn Database, Redis, gọi API hoặc Repository **BẮT BUỘC PHẢI TRẢ VỀ `error`**:
        ```go
        func (usc *xxxUsc) getXxxHelper(ctx context.Context) (DataType, error)
        ```
      - **TUYỆT ĐỐI CẤM** viết hàm chỉ trả về giá trị đơn `func (...) DataType` rồi tự ý nuốt lỗi ngầm.
    - **Rule Mặc Định Bắt Buộc Check Not Found Là `core.ErrRecordNotFound`**:
      - Phân định rõ ràng:
        1. **`errors.Is(err, core.ErrRecordNotFound)`**: Không có bản ghi trong DB $\rightarrow$ Là trường hợp nghiệp vụ bình thường, **được phép** fallback: `return defaultVal, nil`.
        2. **`err != nil` (khác ErrRecordNotFound)**: DB sập, timeout, lỗi mạng, cú pháp SQL sai $\rightarrow$ Là lỗi hệ thống nghiêm trọng, **BẮT BUỘC PHẢI RETURN `err`** ra ngoài để kích hoạt mã lỗi HTTP 500!
    - **Mọi Bước Gọi Trong UseCase Bắt Buộc Phải Kiểm Tra `err != nil`**:
      - Tuyệt đối không được bỏ qua `err` bằng dấu gạch dưới `_`. Bất kỳ hàm con nào trả về `error` đều phải được kiểm tra ngay.

18. **Quy Chuẩn Xử Lý Not Found Cho Query Phức Tạp & Dữ Liệu Phụ Trợ (Auxiliary Data)**:
    - **Query phức tạp (Join, Subquery UNION, Preload)**: Driver ORM có thể trả về lỗi Not Found ở các dạng khác nhau (`gorm.ErrRecordNotFound`, `core.ErrRecordNotFound`, `core.ErrNotFound`, string `"record not found"`). Khi đó cần dùng hàm helper kiểm tra đa tầng (`isRecordNotFound(err)`).
    - **Dữ Liệu Phụ Trợ (Related Items, Sidebar, Recommendations) Không Tìm Thấy $\rightarrow$ KHÔNG PHẢI LÀ LỖI**:
      - Các khối dữ liệu phụ trợ nếu không tìm thấy (Not Found) thì **tuyệt đối không throw lỗi làm chết trang chính của người dùng**. Trang chính vẫn hiển thị bình thường với slice rỗng `[]responses.NewsResp{}`.
      - Chỉ ném lỗi 500 khi DB lỗi thực sự (mất kết nối, timeout).

---

## 3. Cấu Trúc & Sử Dụng `Master` Layout

File: `views/layouts/master.templ`

### Khai Báo Của `Master`

```templ
package layouts

import (
    "github.com/networld-solution/gos/templates"
    "goweb/views/layouts/headers"
    "goweb/views/layouts/footers"
)

templ Master(head *[]templ.Component, fos ...templ.Component) {
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="UTF-8"/>
        <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
        <title>Sunway Home</title>
        <link rel="shortcut icon" href={ templates.AssetURL("/static/img/favicon.ico") } type="image/x-icon"/>
        <link rel="stylesheet" href={ templates.AssetURL("/static/libs/swiper/swiper-bundle.min.css") }/>
        <link rel="stylesheet" href={ templates.AssetURL("/static/css/default.css") }/>
        <link rel="stylesheet" href={ templates.AssetURL("/static/css/style.css") }/>
        if head != nil && len(*head) > 0 {
            for _, item := range *head {
                @item
            }
        }
        <link rel="stylesheet" href={ templates.AssetURL("/static/css/responsive.css") }/>
    </head>
    <body>
        @SvgIcon()
        @headers.Header()
        { children... }
        @footers.Footer()
        <script type="text/javascript" src={ templates.AssetURL("/static/js/countdown-timer.js") }></script>
        <script type="text/javascript" src={ templates.AssetURL("/static/libs/swiper/swiper-bundle.min.js") }></script>
        <script type="text/javascript" src={ templates.AssetURL("/static/js/app.js") }></script>
        if len(fos) > 0 {
            for _, fo := range fos {
                @fo
            }
        }
    </body>
</html>
}
```

### Các Cách Kế Thừa Layout `Master`

#### Cách 1: Trang Không Cần CSS/JS Riêng
```templ
package catalogues

import "goweb/views/layouts"

templ PageCatalogues() {
    @layouts.Master(nil) {
        <main class="catalogues-page">
            <h1>Nội dung trang</h1>
        </main>
    }
}
```

#### Cách 2: Trang Có CSS Riêng Trong `<head>`
```templ
package products

import (
    "github.com/networld-solution/gos/templates"
    "goweb/views/layouts"
)

templ PageProducts() {
    @layouts.Master(&[]templ.Component{pageHead()}) {
        <section class="container">
            <!-- Nội dung trang -->
        </section>
    }
}

templ pageHead() {
    <link rel="stylesheet" href={ templates.AssetURL("/static/css/products.css") }/>
}
```

#### Cách 3: Trang Có Script Hoặc Modal Riêng Ở Cuối Body (`fos`)
```templ
package orders

import (
    "github.com/networld-solution/gos/templates"
    "goweb/views/layouts"
)

templ PageCheckout() {
    @layouts.Master(&[]templ.Component{pageHead()}, pageScripts(), checkoutModal()) {
        <section class="checkout-wrapper">
            <!-- Nội dung thanh toán -->
        </section>
    }
}

templ pageHead() {
    <link rel="stylesheet" href={ templates.AssetURL("/static/css/checkout.css") }/>
}

templ pageScripts() {
    <script src={ templates.AssetURL("/static/js/checkout.js") }></script>
}

templ checkoutModal() {
    <div id="modal-confirm" class="modal hidden">...</div>
}
```

---

## 4. Quy Chuẩn Xây Dựng Component & Partials

### 4.1. Component Cấp Feature (`views/<feature>/components/`)
- Dành cho các khối giao diện chỉ sử dụng tại trang tính năng đó (ví dụ: `SlideBannerHome`, `FlashSaleHome`, `ProductFilterCpn`).
- Khai báo: `package components`.
- Ví dụ:
  ```templ
  package components

  import "github.com/networld-solution/gos/templates"

  templ SlideBannerHome() {
      <section class="slideshow">
          <picture>
              <source media="(max-width: 767px)" srcset={ templates.AssetURL("/static/img/slideshow/banner-m.jpg") }/>
              <img class="slide-home-img" src={ templates.AssetURL("/static/img/slideshow/banner.jpg") } alt="banner"/>
          </picture>
      </section>
  }
  ```

### 4.2. Partials Dùng Chung (`views/partials/`)
- Dành cho các phần tử tái sử dụng trên nhiều trang (Card sản phẩm, Pagination, Breadcrumb, v.v.).
- Khai báo: `package partials`.
- Khi cần truyền dữ liệu (Data/Props):
  ```templ
  package partials

  import (
      "fmt"
      "github.com/networld-solution/gos/templates"
  )

  type ProductProps struct {
      Title       string
      SKU         string
      ThumbURL    string
      Price       float64
      OldPrice    float64
      DiscountTag string
  }

  templ ProductCard(p ProductProps) {
      <article class="product">
          if p.DiscountTag != "" {
              <div class="product_tag"><span>{ p.DiscountTag }</span></div>
          }
          <div class="product_thumb">
              <img class="product_thumb_img" src={ templates.AssetURL(p.ThumbURL) } alt={ p.Title }/>
          </div>
          <div class="product_info">
              <h3 class="product_name">
                  <a href="#" title={ p.Title }>{ p.Title }</a>
              </h3>
              <ul class="product_prop">
                  <li>SKU: { p.SKU }</li>
              </ul>
              <div class="product_prices">
                  <div>
                      <span class="product_price_current">{ fmt.Sprintf("$%.2f", p.Price) }</span>
                  </div>
                  if p.OldPrice > 0 {
                      <del class="product_price_old">{ fmt.Sprintf("$%.2f", p.OldPrice) }</del>
                  }
              </div>
          </div>
      </article>
  }
  ```

---

## 5. Quy Trình Tích Hợp View Vào Handler & Router

Để đưa một trang mới vào hoạt động, luôn tuân thủ 4 bước theo kiến trúc của dự án:

```
[1. Tạo View]          views/<feature>/index.templ
      ↓
[2. Tạo Handler]       modules/web/transport/handlers/<feature>_hdl.go
      ↓
[3. Tạo Composer]      modules/web/composers/comp_<feature>.go
      ↓
[4. Đăng Ký Route]     modules/web/routes_web.go
```

### Bước 1: Tạo View
Tạo `views/orders/index.templ`:
```templ
package orders

import (
    "goweb/views/layouts"
    "goweb/views/partials"
)

templ PageOrders() {
    @layouts.Master(nil) {
        <div class="container">
            @partials.Breadcrumb()
            <h1>Order Management</h1>
        </div>
    }
}
```

### Bước 2: Tạo Handler
Tạo hoặc bổ sung vào `modules/web/transport/handlers/orders_hdl.go`:
```go
package handlers

import (
	"context"
	"goweb/modules/web/transport/requests"
	"goweb/modules/web/transport/responses"
	"goweb/views/orders"

	"github.com/gofiber/fiber/v2"
	"github.com/networld-solution/gos/templates"
	"github.com/networld-solution/sctx"
)

// Khai báo interface UseCase nội bộ cho Handler: phương thức BẮT BUỘC có hậu tố *Usc
type ordersUsc interface {
	ListOrdersUsc(ctx context.Context, req *requests.OrderReq) (*responses.OrderListResp, error)
}

type ordersHdl struct {
	usc    ordersUsc
	logger sctx.Logger
}

func NewOrdersHdl(usc ordersUsc, logger sctx.Logger) *ordersHdl {
	return &ordersHdl{
		usc:    usc,
		logger: logger,
	}
}

/**
 * Method: GET
 * Route: /orders
 * Description: Hiển thị trang danh sách đơn hàng
 * Request: requests.OrderReq (QueryParser)
 * Validate: true (req.Validate)
 * Response: Full Page HTML (templates.Render)
 */
func (h *ordersHdl) PageOrdersHdl() fiber.Handler {
	return func(c *fiber.Ctx) error {
		var req requests.OrderReq
		if err := c.QueryParser(&req); err != nil {
			return fiber.NewError(fiber.StatusBadRequest, err.Error())
		}
		if err := req.Validate(c.Context()); err != nil {
			return fiber.NewError(fiber.StatusBadRequest, err.Error())
		}

		data, err := h.usc.ListOrdersUsc(c.Context(), &req)
		if err != nil {
			h.logger.Errorf("[Module: web] [Action: ordersHdl.PageOrdersHdl] usc.ListOrdersUsc error: %v", err)
			return err
		}

		return templates.Render(c, orders.PageOrders(data))
	}
}
```

### Bước 3: Tạo Composer
Tạo `modules/web/composers/comp_orders.go`:
```go
package composers

import (
	"goweb/modules/web/transport/handlers"
	"goweb/modules/web/usecase"

	"github.com/gofiber/fiber/v2"
	"github.com/networld-solution/sctx"
)

type composerOrders interface {
	PageOrdersHdl() fiber.Handler
}

func ComposerOrdersService(serviceCtx sctx.ServiceContext) composerOrders {
	logger := serviceCtx.Logger()
	usc := usecase.NewOrdersUsc(serviceCtx)
	hdl := handlers.NewOrdersHdl(usc, logger)
	return hdl
}
```

### Bước 4: Đăng Ký Route Trong `routes_web.go`
Mở `modules/web/routes_web.go` và thêm cụm route tập trung:
```go
// --- Orders ---
ordersComp := composers.ComposerOrdersService(serviceCtx)
app.Get("/orders", ordersComp.PageOrdersHdl())
```

---

## 6. Quy Trình Kiểm Thử Chuẩn (Khi Được Yêu Cầu "Test Skill")

Khi lập trình viên yêu cầu **"test skill"** hoặc kiểm tra sau khi code, thực hiện tuần tự theo các bước:

### Bước 1: Kiểm Tra Biên Dịch & Sinh Mã Template
```bash
make gen          # Hoặc: templ generate
go vet ./...      # Đảm bảo không lỗi gõ, không thiếu method trong composer/interface
```
* **Kỳ vọng**: Exit code 0, không có lỗi syntax.

### Bước 2: Kiểm Tra Headers Của Helmet
```bash
curl -I http://localhost:8080/
```
* **Kỳ vọng**: Phải có đầy đủ `X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN`, `Cross-Origin-Resource-Policy: cross-origin`, và cookie `csrf_=...`.

### Bước 3: Kiểm Tra Bảo Mật CSRF
* Chặn request khi thiếu CSRF token: `curl -X POST http://localhost:8080/brands/results -d "page=2"` $\rightarrow$ trả về `403 Forbidden`.
* Chấp nhận request khi có token hợp lệ: đính kèm cookie và header `X-CSRF-Token` $\rightarrow$ trả về `200 OK`.

### Bước 4: Kiểm Tra HTMX Flow & Push URL
* Gửi request POST kèm header `HX-Request: true`.
* **Kỳ vọng**: Response có header `HX-Push-Url: /...` và chỉ trả về HTML Fragment, không có `<html>`, `<head>`, `<footer>`.

### Bước 5: Kiểm Tra Trang Lỗi (400, 404, 500)
* Truy cập URL sai: Kiểm tra trang 404 hiển thị kèm Master layout (Header, Footer nguyên vẹn).
* Mô phỏng mất kết nối DB: Kiểm tra trang 500 hiển thị Standalone an toàn mà không bị crash theo DB.

> Xem chi tiết các kịch bản kiểm thử tại [Hướng Dẫn Kiểm Thử Toàn Diện](./references/testing-and-verification-guide.md).

---

## 7. Build & Lệnh Thực Thi Thường Dùng

Dự án đã định nghĩa sẵn các lệnh trong `Makefile`:

| Lệnh | Ý nghĩa |
| :--- | :--- |
| `make gen` | Chạy `templ generate` để sinh mã Go từ tất cả các file `.templ`. |
| `make air_server` | Chạy dev server với Air, tự động sinh lại template và restart server khi sửa file. |
| `make server` | Chạy `templ generate && go run main.go server`. |
| `make build` | Biên dịch binary production `sunwayweb`. |

> **Lưu ý**: Trước khi chạy `git commit` hoặc kiểm tra lỗi biên dịch, luôn chạy `make gen` hoặc `templ generate` để đảm bảo code Go đồng bộ với `.templ`.

---

## 8. Tài Liệu Tham Khảo Mở Rộng & Hệ Sinh Thái Skills (References & Ecosystem)

### Tài Liệu Tham Khảo Trong Skill Này
* [Hướng Dẫn CSS Responsive (rem, em, font-size)](./references/responsive-css-guide.md): Toàn bộ nguyên lý neo gốc `rem`, mở rộng `em`, fluid typography `clamp()`, bẫy thẻ form `font-size: 1rem` và bảng quy đổi px.
* [Cú Pháp templ & Best Practices](./references/templ-syntax.md): Hướng dẫn chi tiết cú pháp thẻ, vòng lặp, điều kiện, classes động và nhúng component con.
* [Ví Dụ Trang Động Truyền Dữ Liệu](./examples/dynamic-view-with-data.md): Ví dụ mẫu đầy đủ từ Fiber Context, Handler, truyền struct Props vào View đến render ra HTML.
* [Hướng Dẫn Kiểm Thử & Xác Minh (Testing & Verification Guide)](./references/testing-and-verification-guide.md): Bộ lệnh curl, script kiểm tra bảo mật, test HTMX và kiểm tra console.

### Các Skills Đồng Hành Trong Dự Án (Specialized Companion Skills)
* **[goweb-htmx-list-pattern](../goweb-htmx-list-pattern/SKILL.md)**: Chuyên sâu cho các màn hình danh sách, bộ lọc AJAX POST, phân trang Progressive Enhancement (`PaginationHx`), CSRF và SEO URL.
* **[goweb-page-sections-option](../goweb-page-sections-option/SKILL.md)**: Chuyên sâu cho quản lý Banner và các Content Block của trang thông qua Option Group & Option Value kết hợp Redis cache.
