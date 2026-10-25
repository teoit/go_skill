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
   * **Khối 1: PUBLIC ROUTES (`app.Get`, `app.Post`)**:
     - Phục vụ tất cả người dùng không yêu cầu đăng nhập.
   * **Khối 2: AUTHENTICATED ROUTES (`appLogin := swmidd.NewAuthRouter(app, midds...)`)**:
     - Chỉ chứa các route bắt buộc đăng nhập (bảo vệ bởi Auth Middleware).
     - Mọi route trong này đều dùng `appLogin.Get(...)`, `appLogin.Post(...)`...
     - Tuyệt đối **không khai báo `app.` bên trong khối `appLogin`**.

2. **Gom Nhóm Các Route Liên Quan Gần Nhau Theo Tính Năng (Feature Grouping)**:
   * Trong từng khối (Public hoặc Authenticated), **tất cả các route thuộc cùng một module/tính năng phải được đặt liền kề nhau** thành từng cụm có comment tiêu đề rõ ràng (ví dụ: `--- Products ---`, `--- News ---`, `--- Brands ---`, `--- Checkout & Payment ---`).
   * **Tại sao**: Giúp lập trình viên dễ nhìn, dễ tìm, khi phát triển hoặc debug tính năng nào thì toàn bộ route liên quan (trang GET, HTMX AJAX fragment, JSON API) đều nằm tập trung một chỗ, không bị phân tán rải rác khắp file.
   * **Ví dụ mẫu cụm News**:
     ```go
     // --- News ---
     app.Get("/news", newsComp.ListNewsHdl())
     app.Get("/news/:pagepath<regex(^([a-z0-9-]+)-([0-9]+)$)>.html", newsComp.NewsDetailHdl())
     app.Post("/api/news/results", apiNewsComp.NewsResultsApi())
     app.Post("/api/news", apiNewsComp.ListNewsApi())
     ```
   * **Ví dụ mẫu cụm Products**:
     ```go
     // --- Products ---
     app.Get("/products", prodComp.ProductsPageHdl())
     app.Get("/:pagepath<regex(^([a-z0-9-]+)-([0-9]+)$)>.html", prodDetailComp.ProductDetailPageHdl())
     app.Get("/api/products/partial", apiProductsComp.ProductsPartialHdl())
     app.Get("/api/products/best-sellers", apiBestSellersComp.BestSellersAPI())
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
   - **Bản chất kỹ thuật**: Mặc định trong stylesheet trình duyệt (User Agent Stylesheet), các thẻ form như `<button>`, `<input>` **KHÔNG tự động kế thừa `font-size` của thẻ cha**, mà luôn bị áp đặt cỡ chữ mặc định là `13.333px` (hoặc font control hệ thống).
   - **Hậu quả nếu thiếu `font-size: 1rem`**: Toàn bộ thuộc tính viết bằng `em` trên nút (`height: 4.4em`, `padding: 0 2.4em`, `border-radius: 0.8em`, cỡ chữ con `span { font-size: 1.4em }`) sẽ bị nhân với `13.333px` thay vì `10px` (`1rem`), khiến nút bị phình to ~133% (ví dụ: cao thành `58.6px` thay vì `44px`).
   - **Quy chuẩn bắt buộc**: Luôn luôn khai báo `font-size: 1rem;` trên các class của `<button>`, `<input>`, `<select>`, `<textarea>` để neo chính xác về gốc `html { font-size: 62.5%; } => 1rem = 10px`:
     ```css
     .btn_faq_load_more {
         font-size: 1rem;      /* BẮT BUỘC: Neo về root 10px thay vì 13.333px của browser */
         height: 4.4em;        /* Đúng chuẩn 44px (4.4 * 10px) */
         display: inline-flex;
         align-items: center;
         justify-content: center;
         padding: 0 2.4em;
         border-radius: 0.8em;
     }
     ```




8. **Quy Chuẩn Banner & Section Content: Luôn Có 2 Tầng (Dynamic + Fallback)**:
   - Mọi banner hoặc section quản trị qua CMS khi render trong `.templ` **bắt buộc có 2 tầng**:
     - Tầng dynamic: `if banner != "" { @templ.Raw(banner) }`
     - Tầng fallback: `else { <section class="...">...</section> }` chứa đầy đủ cấu trúc HTML, responsive `<picture>`, `<source media="...">`, text và CSS class chuẩn. Tuyệt đối không để trang bị trắng/thiếu banner khi DB chưa có dữ liệu.

9. **Quy Tắc Lựa Chọn Option Group vs Option Value Khi Dùng Options Cho Page Content**:
   - **Chỉ 1 section / 1 banner lẻ**: Chỉ cần định nghĩa **1 const Option Value Key** (ví dụ `KEY_BRAND_BANNER`, `KEY_BLOG_NEWS_BANNER`). Query đơn lẻ qua `optionValueRepo.FirstOptionValueRepo`.
   - **Nhiều section content / Landing Page không show dynamic list data**: Bắt buộc tạo **1 const Option Group Key** (ví dụ `CategorySlugHome = "home"`, `KEY_OPTION_GROUP_ABOUT_US = "about-us"`). Lấy toàn bộ qua `optionShared.GetOptionValuesByGroupKey` để tận dụng Redis caching tự động, sau đó map theo key của từng option value con.

10. **Quy Chuẩn Middleware Bảo Mật (`ServerInternal`), Helmet, CSRF & Tự Động Hóa Với HTMX**:
    - **Helmet**: Sử dụng cấu hình chuẩn tại `ServerInternal`: `XSSProtection: "0"` (chuẩn OWASP), `ContentTypeNosniff: "nosniff"`, `XFrameOptions: "SAMEORIGIN"`, `ReferrerPolicy: "strict-origin-when-cross-origin"`, `CrossOriginResourcePolicy: "cross-origin"` (đảm bảo static assets/webp không bị chặn cross-origin).
    - **CSRF Token tự động cho HTMX**: HTMX đã được nạp toàn cục tại `master.templ` trước `app.js`. `app.js` có listener `htmx:configRequest` tự động lấy cookie `csrf_` gán vào header `X-CSRF-Token` cho toàn bộ request HTMX. Lập trình viên **không cần cấu hình thủ công header CSRF** khi viết code HTMX.
    - **Lưu ý CSP**: Nếu sau này bật `ContentSecurityPolicy` trong Helmet, **bắt buộc** phải có chỉ thị `connect-src 'self'` (cho AJAX HTMX) và `script-src 'self' 'unsafe-inline'` để tránh bị trình duyệt chặn request.

11. **Quy Tắc Phân Định Tuyệt Đối: Route GET (Pages) $\rightarrow$ `handlers/`, Route `/api/...` $\rightarrow$ `apis/`**:
    - **Route GET hiển thị trang cho người dùng** (không có tiền tố `/api/`, ví dụ: `/news`, `/products`, `/brands`, `/about-us`, `/contact`): **Bắt buộc nằm trong `modules/web/transport/handlers/`**, đảm nhận render toàn bộ trang HTML (`layouts.Master(...)`).
    - **Mọi route bắt đầu bằng `/api/...`** (bao gồm cả HTMX AJAX fragments như `POST /api/news/results`, `GET /api/products/partial`, `POST /api/promotions/active-sale` và JSON APIs): **Bắt buộc nằm trong `modules/web/transport/apis/`**.
    - **Nguyên tắc phân chia rõ ràng**: Tuyệt đối không để route của `handlers/` mang tiền tố `/api/`, và ngược lại handler của `apis/` luôn luôn phải được đăng ký ở nhóm route có tiền tố `/api/`.

12. **Quy Chuẩn Active Menu & Navigation (Desktop, Mobile & Sub-items)**:
    - **Không hardcode class `active` tĩnh trong file `.templ`**: Tránh tình trạng một menu luôn luôn bị gạch chân active dù người dùng đang ở trang khác.
    - **Cơ chế kích hoạt tự động qua `app.js` (`activeNavigation`)**:
      - Tự động so khớp URL hiện tại (`pathname`, query string `?category=...`, `?country=...`, `?brand=...`).
      - Hỗ trợ nhận diện các trang con / chi tiết (Regex Product Detail `/:slug-:id.html`, News Detail `/news/:slug.html`...).
      - **Đồng bộ đa cấp (Parent + Child)**: Khi bất kỳ sub-item nào trong `.menu_sub` hoặc `.m_sub` được active, **dropdown cha tương ứng bắt buộc cũng phải được active** (`.menu_item.active`, `.m_nav_item.active`).
      - **Phản ứng với Lịch sử & HTMX**: Lắng nghe sự kiện `popstate` và `htmx:pushedIntoHistory` để tự động cập nhật active menu khi người dùng duyệt lịch sử hoặc lọc dữ liệu qua HTMX.

13. **Quy Chuẩn Comment Docblock Cho Handlers & APIs (`handlers/` & `apis/`)**:
    - Mọi function handler và API bắt buộc có docblock comment chi tiết phía trước theo mẫu chuẩn để bất kỳ Dev nào đọc vào cũng hiểu rõ:
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

14. **Quy Chuẩn Viết UseCase (`usecase/`) & Giới Hạn Lồng Gọi Hàm Tối Đa 2 Cấp (Max 2 Levels Call Depth)**:
    - **Liệt kê tuần tự các bước 1, 2, 3...**: Trong UseCase, bắt buộc phải viết comment các bước thực hiện rõ ràng (`// Bước 1: ...`, `// Bước 2: ...`) để thể hiện mạch nghiệp vụ rõ ràng, dễ bảo trì, dễ review.
    - **Quy tắc lồng gọi hàm tối đa 2 cấp (Max 2 Levels Call Depth)**:
      - **Cấp 1**: Hàm UseCase chính (ví dụ `ListBrandUsc`).
      - **Cấp 2**: Các hàm helper, repository, hoặc mapper được gọi trực tiếp từ UseCase (ví dụ `usc.brandRepo.CountBrandRepo(...)`, `usc.getBrandBanner(...)`, `mapping.MapperHomeBrands(...)`).
      - **TUYỆT ĐỐI KHÔNG** lồng gọi sang cấp thứ 3 (không để hàm cấp 2 gọi tiếp hàm helper cấp 3, rồi cấp 3 gọi cấp 4). Toàn bộ luồng xử lý chính phải phẳng (flat workflow), tường minh ngay tại UseCase, tránh code spaghetti và hiệu ứng lồng nhau khó debug.

15. **Quy Chuẩn Cửa Ngõ Đầu Vào (Request Validation) & Error-Returning Validate Pattern**:
    - **Triết lý Sống Còn (Security & Data Integrity Gateway)**:
      - Tầng Request Validation là cửa ngõ sống còn bảo vệ toàn bộ hệ thống khỏi tấn công, input rác, bot crawler spam, SQLi và sai lệch dữ liệu.
      - Hàm `Validate(ctx context.Context) error` **bắt buộc phải có chữ ký trả về `error`**. Tuyệt đối **không viết hàm `Validate()` trả về `void`** rồi âm thầm nuốt lỗi hoặc tự ý ép kiểu fallback dễ dãi khi gặp input bất thường!
    - **Nguyên tắc Kiểm Tra Nghiêm Khắc (Strict Validation & Whitelist)**:
      - **Phân trang (`Page`)**: Nếu không truyền (`Page == 0`), gán mặc định `1`. Nhưng nếu truyền số âm (`Page < 0`), số 0 truyền chủ đích, hoặc vượt trần chống Deep Pagination / DB Index Scan DDoS (`Page > 200`), **bắt buộc phải trả về lỗi** `errors.New("page exceeds maximum limit of 200")`.
      - **Sắp xếp (`OrderBy`, `Sort`) & Bộ lọc**: Bắt buộc kiểm tra qua Whitelist hằng số (ví dụ chỉ chấp nhận `"newest"`, `"oldest"`). Nếu client truyền chuỗi lạ không nằm trong whitelist, **bắt buộc trả về lỗi `Bad Request`**, tuyệt đối không tự ý fallback về mặc định mà giấu lỗi.
    - **Xử lý lỗi tại tầng Transport (Handler | API)**:
      - Khi `err := req.Validate(ctx); err != nil`:
        - **JSON API**: Trả về HTTP 400 Bad Request kèm JSON chuẩn: `c.Status(fiber.StatusBadRequest).JSON(fiber.Map{ "code": 400, "error": err.Error(), "message": err.Error() })`.
        - **HTMX Fragment**: Trả về HTTP 400 Bad Request kèm message rõ ràng: `c.Status(fiber.StatusBadRequest).SendString(err.Error())` để kích hoạt bắt lỗi trên client hoặc dừng swap rác.
        - **Full Page SSR**: Trả về lỗi `c.Status(fiber.StatusBadRequest)...` hoặc redirect an toàn.
      - Tầng UseCase (`usc`) **chỉ nhận Request Struct đã vượt qua Validate thành công 100%**.
    - **Request-Driven Transport Helpers (Xử lý Header/Cookie/Push-URL)**:
      - Mọi logic tính toán Header (như `HX-Push-Url`), Cookie hay URL **phải đóng gói thành method của chính struct Request** (ví dụ `req.PushURL()`), tuyệt đối không viết logic kiểm tra lộn xộn ở tầng Handler/API. Tại tầng Transport chỉ nhận giá trị và gọi `c.Set("HX-Push-Url", req.PushURL())`.
    - **Single Request Constraint**: Mỗi Handler hoặc API chỉ sử dụng DUY NHẤT 1 Request struct, không khai báo 2 request structs trong cùng một hàm.

16. **Quy Chuẩn Thiết Kế & Xử Lý Các Trang Lỗi (Error Pages: 400, 404 vs 500)**:
    - **Trang 400 (Bad Request) & 404 (Not Found) $\rightarrow$ DÙNG MASTER LAYOUT**:
      - Đây là lỗi do phía Client (người dùng bấm nhầm query string, sai page, hoặc link cũ).
      - Server và Database vẫn hoàn toàn khỏe mạnh. Do đó, **bắt buộc phải bọc trong `layouts.Master(isLogin, ...)`** để giữ nguyên Header, Menu, Giỏ hàng và Footer. Giúp người dùng dễ dàng chuyển hướng sang trang khác tiếp tục mua sắm mà không bị đứt đoạn trải nghiệm.
    - **Trang 500 (Internal Server Error) $\rightarrow$ ĐỘC LẬP (STANDALONE - KHÔNG DÙNG MASTER LAYOUT)**:
      - Đây là lỗi từ Server, Database hoặc mã nguồn nội bộ.
      - Nếu trang 500 dùng Master layout, khi Database bị sập, việc Master cố tải dữ liệu menu/giỏ hàng từ DB sẽ khiến trang 500 bị crash theo (gây màn hình trắng xóa). Do đó, trang 500 bắt buộc phải là một file HTML độc lập, tối giản, tự chứa toàn bộ tài nguyên cần thiết.
    - **Tách Biệt Web SSR vs API Trong `error_midd.go`**:
      - Nếu request bắt đầu bằng `/api/`: Trả về JSON `{ "code": code, "error": msg, "message": msg }`.
      - Nếu là request Web thông thường: Render trang HTML tương ứng (`errpages.BadRequest`, `errpages.NotFound`, `errpages.ServerError`).

17. **Quy Chuẩn Định Vị Lỗi & Báo Đúng Nguồn Gốc (Error Tracing & Origin Identification Pattern)**:
    - **Nguyên tắc Sống Còn (Báo Đúng Vị Trí & Ngữ Cảnh)**:
      - Khi lỗi xảy ra, hệ thống bắt buộc phải xác định chính xác 3 cấp độ nguồn gốc phát sinh lỗi:
        1. **Module**: Ví dụ `web`, `admin`, `api`...
        2. **Layer**: `Request` (Validate/Parse), `Transport` (Handler/API), `UseCase` (Nghiệp vụ), `Repository` (Cơ sở dữ liệu).
        3. **Action / Handler / Method Name**: Ví dụ `brandsHdl.IndexHdl`, `newsAPI.NewsResultsApi`, `newsAPI.ListNewsApi`, `newsReq.Validate`.
    - **Quy Chuẩn Ghi Log Nội Bộ Bằng `sctx.Logger` (Server-side Trace Logging)**:
      - Tuyệt đối **không dùng package `log` chuẩn của Go**, bắt buộc phải inject và sử dụng `sctx.Logger` (`sctx.Logger.Errorf`):
        ```go
        logger.Errorf("[Module: %s] [Action: %s] %v", moduleName, actionName, err)
        ```
        Ví dụ: `h.logger.Errorf("[Module: web] [Action: brandsHdl.IndexHdl] usc.ListBrandUsc error: %v", err)`
      - Giúp log được ghi nhận chuẩn xác theo ngày (`AppLoggerDaily`), đồng bộ với hệ thống Sentry/ELK/CloudWatch và xác định file/action chỉ trong 1 giây.
    - **Quy Chuẩn Báo Lỗi Cho Client Theo Từng Loại Action**:
      - **Web SSR Action (Handler)**:
        - Lỗi 400 (Query/Validate): Trả về `fiber.NewError(fiber.StatusBadRequest, err.Error())` $\rightarrow$ Tự động hiển thị trang HTML 400 kèm Master Layout và thông báo lỗi rõ ràng.
        - Lỗi 404 (Không thấy slug/pagepath): Trả về `fiber.NewError(fiber.StatusNotFound, "Article not found")` $\rightarrow$ Tự động hiển thị trang HTML 404 Not Found kèm Master Layout.
        - Lỗi 500 (DB sập): Log chi tiết `[Module: web] [Action: ...]` và trả về trang HTML 500 Standalone.
      - **HTMX Fragment Action (API Results)**:
        - Đính kèm header `c.Set("HX-Reswap", "innerHTML")` và trả về fragment HTML `.alert_error_box` hiển thị thông báo lỗi trực quan ngay tại container kết quả (`#brands-results`, `#news-results`), tuyệt đối không để UI bị đơ im lặng.
      - **JSON API Action**:
        - Trả về JSON envelope đồng nhất:
          ```json
          {
            "code": 400,
            "error": "Tên hoặc loại lỗi",
            "message": "Nội dung chi tiết lỗi của action"
          }
          ```

18. **Quy Chuẩn Cấm Nuốt Lỗi Ngầm & Bắt Buộc Kiểm Tra Lỗi Đầy Đủ (Zero Error-Swallowing & Explicit Error Propagation Pattern)**:
    - **Triết lý Sống Còn (Không Bao Giờ Giấu Lỗi Database / I/O)**:
      - Mọi hàm có thao tác I/O, truy vấn Database (PostgreSQL, MongoDB), Redis, gọi API hoặc Repository **BẮT BUỘC PHẢI TRẢ VỀ `error`**:
        ```go
        func (usc *xxxUsc) getXxxHelper(ctx context.Context) (DataType, error)
        ```
      - **TUYỆT ĐỐI CẤM** viết hàm chỉ trả về giá trị đơn `func (...) DataType` rồi tự ý nuốt lỗi như sau:
        ```go
        // ❌ CẤM TUYỆT ĐỐI MẪU CODE NÀY:
        optVal, err := usc.repo.First(...)
        if err != nil || optVal == nil {
            return "" // Âm thầm nuốt lỗi, biến sự cố DB sập thành "không có dữ liệu"!
        }
        ```
    - **Rule Mặc Định Bắt Buộc Check Not Found Là `core.ErrRecordNotFound` (Default Not Found Rule)**:
      - Đối với các query đơn bảng, First/Find tiêu chuẩn: Bắt buộc mặc định phải so khớp lỗi Not Found bằng `errors.Is(err, core.ErrRecordNotFound)` (hoặc `errors.Is(err, core.ErrNotFound)`).
      - Khi nhận `err` từ Database / Repository, bắt buộc phải phân định:
        1. **`errors.Is(err, core.ErrRecordNotFound)`**: Không có bản ghi trong DB $\rightarrow$ Là trường hợp nghiệp vụ bình thường, **được phép** fallback giá trị mặc định: `return defaultVal, nil`.
        2. **`err != nil` (khác ErrRecordNotFound)**: Mất kết nối DB, DB sập, timeout, lỗi mạng, cú pháp SQL sai $\rightarrow$ Là lỗi hệ thống nghiêm trọng, **BẮT BUỘC PHẢI RETURN `err`** ra UseCase để kích hoạt mã lỗi HTTP 500!
        ```go
        // ✅ MẪU CODE CHUẨN XÁC:
        optVal, err := usc.optionValueRepo.FirstOptionValueRepo(ctx, &filter)
        if err != nil {
            if errors.Is(err, core.ErrRecordNotFound) {
                return "", nil // Không có banner -> Trả về rỗng an toàn
            }
            return "", err // 💥 DB sập -> Bắt buộc trả lỗi ra ngoài!
        }
        if optVal == nil {
            return "", nil
        }
        return optVal.Content, nil
        ```
    - **Mọi Bước Gọi Trong UseCase Bắt Buộc Phải Kiểm Tra `err != nil` (Every Call Must Be Checked)**:
      - Trong hàm UseCase chính, tuyệt đối không được bỏ qua `err` bằng dấu gạch dưới `_`. Bất kỳ hàm con nào trả về `error` đều phải được kiểm tra ngay:
        ```go
        banner, err := usc.getBrandBanner(ctx)
        if err != nil {
            usc.logger.Errorf("[Module: %s] [Action: %s] getBrandBanner error: %v", module, action, err)
            return nil, err // 💥 Chặn đứng luồng và trả lỗi 500
        }
        ```

19. **Quy Chuẩn Xử Lý Not Found Cho Query Phức Tạp (Join Nhiều Bảng, Subquery Lồng, Preload) & Dữ Liệu Phụ Trợ (Auxiliary Data)**:
    - **Bản Chất Kỹ Thuật (Khi Nào Phải Check Lại Not Found Đa Tầng?)**:
      - Quy tắc mặc định của dự án là dùng `core.ErrRecordNotFound`.
      - **TUY NHIÊN**, khi thực hiện các câu truy vấn phức tạp:
        1. **Kết nối nhiều bảng / JOIN**: `db.Joins(...)`, bảng quan hệ many-to-many qua bảng trung gian.
        2. **Subquery lồng nhau**: Sử dụng `clause.Expr`, `IN (SELECT ...)`, `UNION`.
        3. **Preload quan hệ**: `Preload("Categories")`, `Preload("PostContent")`.
      - Các tầng trừu tượng ORM (GORM / Postgres driver) khi không khớp dữ liệu phức hợp có thể trả về lỗi Not Found ở các dạng khác nhau: `gorm.ErrRecordNotFound`, `core.ErrRecordNotFound`, `core.ErrNotFound`, hoặc chuỗi text `"record not found"`.
      - Nếu chỉ kiểm tra cứng nhắc 1 loại `core.ErrRecordNotFound`, các lỗi Not Found dạng khác sẽ bị lọt lưới $\rightarrow$ bị hệ thống hiểu nhầm là lỗi database sập và ném ra HTTP 500 oan uổng!
    - **Quy Chuẩn Bắt Buộc Dùng Hàm Kiểm Tra Not Found Toàn Diện (`isRecordNotFound`)**:
      - Khi xử lý các query phức tạp, bắt buộc phải sử dụng hàm helper kiểm tra đa tầng:
        ```go
        func isRecordNotFound(err error) bool {
            if err == nil {
                return false
            }
            return errors.Is(err, core.ErrRecordNotFound) ||
                errors.Is(err, core.ErrNotFound) ||
                errors.Is(err, gorm.ErrRecordNotFound) ||
                strings.Contains(err.Error(), "record not found")
        }
        ```
    - **Dữ Liệu Phụ Trợ (Related Items, Sidebar, Recommendations) Không Tìm Thấy $\rightarrow$ KHÔNG PHẢI LÀ LỖI**:
      - **Nguyên tắc Sống Còn**: Các khối dữ liệu phụ trợ (ví dụ bài viết liên quan trong trang chi tiết, sản phẩm gợi ý, danh mục phụ...) nếu không có dữ liệu (Not Found) thì **tuyệt đối không được throw lỗi làm chết trang chính của người dùng**!
      - Trang chi tiết chính vẫn phải hiển thị bình thường với mảng rỗng `[]responses.NewsResp{}`.
      - **Chỉ ném lỗi 500 khi DB lỗi thực sự**: Mất kết nối database, connection pool cạn kiệt, timeout, syntax SQL hỏng...
        ```go
        // ✅ MẪU CODE CHUẨN XÁC KHI LẤY DỮ LIỆU PHỤ TRỢ:
        relatedArticles, err := usc.getRelatedNews(ctx, post, 3)
        if err != nil {
            if isRecordNotFound(err) {
                relatedArticles = []responses.NewsResp{} // Không có bài liên quan -> Tiếp tục render trang chính!
            } else {
                usc.logger.Errorf("[Module: %s] [Action: %s] getRelatedNews error: %v", module, action, err)
                return nil, err // 💥 Chỉ trả lỗi 500 khi DB lỗi kết nối / timeout thật sự!
            }
        }
        ```
    - **Quy Chuẩn Viết Subquery Trong `utils.Filters.Conds` (Tránh Lỗi Syntax SQL)**:
      - Khi truyền subquery vào `Conds` dạng `map[string]interface{}`:
        - **CẤM** đặt key map chứa dấu `?` phức tạp (như `conds["(category_id IN (?) OR id IN (?))"] = clause.Expr{...}`), vì GORM sẽ nhồi `clause.Expr` vào dấu `?` đầu tiên gây lồng ngoặc sai cú pháp SQL.
        - **BẮT BUỘC** dùng subquery gộp chuẩn qua `UNION` với key đơn giản:
          ```go
          subQuery := "SELECT post_id FROM cms.categories_posts WHERE category_id IN (?) UNION SELECT id FROM cms.posts WHERE category_id IN (?)"
          condsCat := map[string]interface{}{
              "status":        enum.POST_STATUS_PUBLISH,
              "id NOT IN (?)": excludeList,
              "id IN (?)":     clause.Expr{SQL: subQuery, Vars: []interface{}{categoryIDs, categoryIDs}},
          }
          ```

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
- Dành cho các phần tử tái sử dụng trên nhiều trang (Card sản phẩm, Pagination, Breadcrumb, Banner header, v.v.).
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
	"goweb/views/orders"

	"github.com/gofiber/fiber/v2"
	"github.com/networld-solution/gos/templates"
)

type ordersUsc interface {
}

type ordersHdl struct {
	ordersUsc ordersUsc
}

func NewOrdersHdl(ordersUsc ordersUsc) *ordersHdl {
	return &ordersHdl{
		ordersUsc: ordersUsc,
	}
}

func (h *ordersHdl) PageOrdersHdl() fiber.Handler {
	return func(c *fiber.Ctx) error {
		return templates.Render(c, orders.PageOrders())
	}
}
```

### Bước 3: Tạo Composer
Tạo `modules/web/composers/comp_orders.go`:
```go
package composers

import (
	"goweb/modules/web/transport/handlers"

	"github.com/gofiber/fiber/v2"
	"github.com/networld-solution/sctx"
)

type composerOrders interface {
	PageOrdersHdl() fiber.Handler
}

func ComposerOrdersService(serviceCtx sctx.ServiceContext) composerOrders {
	hdl := handlers.NewOrdersHdl(nil)
	return hdl
}
```

### Bước 4: Đăng Ký Route Trong `routes_web.go`
Mở `modules/web/routes_web.go` và thêm:
```go
ordersComp := composers.ComposerOrdersService(serviceCtx)
app.Get("/orders", ordersComp.PageOrdersHdl())
```

---

## 6. Build & Lệnh Thực Thi Thường Dùng

Dự án đã định nghĩa sẵn các lệnh trong `Makefile`:

| Lệnh | Ý nghĩa |
| :--- | :--- |
| `make gen` | Chạy `templ generate` để sinh mã Go từ tất cả các file `.templ`. |
| `make air_server` | Chạy dev server với Air, tự động sinh lại template và restart server khi sửa file. |
| `make server` | Chạy `templ generate && go run main.go server`. |
| `make build` | Biên dịch binary production `sunwayweb`. |

> **Lưu ý**: Trước khi chạy `git commit` hoặc kiểm tra lỗi biên dịch, luôn chạy `make gen` hoặc `templ generate` để đảm bảo code Go đồng bộ với `.templ`.

---

## 8. Quy Chuẩn HTMX Cho Trang Danh Sách (News & Brands Pattern)

Áp dụng cho mọi trang danh sách cần lọc (filter), sắp xếp (sort), tìm kiếm và phân trang mượt mà không reload toàn trang.

### 8.1. Cấu Trúc 3 Khối Trong File `.templ`
Mỗi trang danh sách HTMX chia thành 3 phần:
1. **`PageXxx(data)` (Full Page)**: Render toàn trang cho request GET đầu tiên.
   - Bọc trong `@layouts.Master(data.IsLogin, ...)` (thư viện `htmx.min.js` đã được nạp toàn cục tại `master.templ`).
   - Vùng kết quả đặt `id="xxx-results"`, thuộc tính `hx-target="this"`, `hx-swap="innerHTML show:#xxx-results:top"`, `hx-indicator="#xxx-results"`.
   - Form lọc/sort: `hx-post="/xxx/results"`, `hx-trigger="change"`, `hx-target="#xxx-results"`.
2. **`XxxList(data)` (List & Pagination)**: Render thẻ cards grid + phân trang HTMX `@partials.PaginationHx(data.CurrentPage, data.TotalPages, "/xxx/results", urlBuilderFunc)`.
3. **`XxxResults(data)` (AJAX Fragment)**: Component độc lập trả về riêng cho request HTMX POST.
   - Nếu có bộ đếm/tiêu đề ngoài container cần cập nhật, sử dụng **Out-Of-Band Swap**: `hx-swap-oob="true"` (`newsCount(data, true)`).
   - Render `@XxxList(data)`.

### 8.2. Phân Tách Kiến Trúc: `handlers/` (Full Pages) vs `apis/` (HTMX Fragments & APIs)
Mỗi tính năng danh sách HTMX được tách bạch rõ ràng giữa 2 thư mục:
1. **Thư mục `modules/web/transport/handlers/` (Page Handlers)**:
   - Chỉ chứa handler phục vụ request **GET tải toàn trang HTML** (`@layouts.Master(...)`).
   - Ví dụ: `ListNewsHdl()` (GET `/news`), `ProductsPageHdl()` (GET `/products`).
   - Parse query bằng `c.QueryParser(&req)`, gọi usecase, render toàn trang: `templates.Render(c, xxx.PageXxx(data))`.
2. **Thư mục `modules/web/transport/apis/` (API Handlers)**:
   - **QUY TẮC BẮT BUỘC**: Mọi endpoint do thư mục `apis/` xử lý **BẮT BUỘC PHẢI CÓ TIỀN TỐ ROUTE `/api/`** (ví dụ: `POST /api/news/results`, `GET /api/products/partial`, `POST /api/promotions/active-sale`). Tuyệt đối không đăng ký route của `apis/` ở root level không có `/api/`.
   - Chứa handler phục vụ request **POST/GET của HTMX** (và các JSON APIs nếu có).
   - Ví dụ: `NewsResultsApi()` (POST `/api/news/results`), `ProductsPartialHdl()` (GET `/api/products/partial`).
   - Parse body bằng `c.BodyParser(&req)`, gọi usecase, đồng bộ thanh địa chỉ qua `c.Set("HX-Push-Url", pushURL)`.
   - Chỉ render fragment: `templates.Render(c, xxx.XxxResults(data))`.
   - **Lưu ý**: Không chia nhỏ thành các API rời rạc như `sort`, `pagination` riêng, mà gom chung vào 1 handler HTMX Results nhận struct request hoàn chỉnh.

> Xem tài liệu chi tiết và code mẫu tại [Quy Chuẩn HTMX Danh Sách](./references/htmx-list-pattern.md).

---

## 9. Quy Chuẩn Quản Lý Banner & Section Content (Option Group / Option Value)

Dành cho các khối nội dung tĩnh, banner, section giới thiệu, cam kết, hướng dẫn do CMS biên tập qua bảng `cms_option_groups` và `cms_option_values`.

### 9.1. Khi Nào Dùng Option Value (Đơn Lẻ)
* **Đối tượng**: Trang chỉ có **1 banner hoặc 1 section content duy nhất** (ví dụ: Banner đầu trang `/products`, `/news`, `/brands`).
* **Khai báo**: Đặt 1 const key Option Value (ví dụ: `consts.KEY_BRAND_BANNER = "brand-banner"`).
* **Truy vấn**: Dùng trực tiếp `optionValueRepo.FirstOptionValueRepo(ctx, &utils.Filters{Conds: map[string]interface{}{"key = ?": consts.KEY_..., "status = ?": enum.STATUS_ACTIVE_NUMBER}})`. Lấy `optVal.Content`.
* **Render**: Render bằng `@templ.Raw(bannerHTML)` nếu có, hoặc hiển thị markup HTML fallback mặc định nếu trống.

### 9.2. Khi Nào Dùng Option Group (Nhiều Section Content)
* **Đối tượng**: Trang có **nhiều section content / landing page không hiển thị dynamic entity list** (ví dụ: Trang chủ `home`, Giới thiệu `about-us`, Hướng dẫn `how-ordering-works`, Chính sách điều khoản...).
* **Khai báo**: Đặt **1 const key đại diện cho Option Group** của cả trang (ví dụ: `CategorySlugHome = "home"`, `KEY_OPTION_GROUP_ABOUT_US = "about-us"`).
* **Truy vấn**: Gọi qua `optionShared.GetOptionValuesByGroupKey(ctx, groupKey)`:
  - Tự động cache Redis với TTL `conf.TtlCacheOptionGroup`.
  - Tự động lấy tất cả `OptionValue` con thuộc group theo thứ tự `ordering ASC, id ASC`.
  - Trả về slice `[]responses.OptionValueResp`.
* **Map & Render**: Chuyển thành `map[string]string` theo key từng section để render từng khối tương ứng trong `.templ`. Mỗi section đều có fallback HTML độc lập.

> Xem tài liệu chi tiết và checklist tại [Quy Chuẩn Option Group & Banner](./references/cms-options-and-banner-guide.md).

---

## 10. Kiến Trúc Bảo Mật Middleware (`ServerInternal`), Helmet, CSRF & Tự Động Hóa HTMX

Hệ thống bảo mật và middleware cốt lõi được khởi tạo tập trung tại `cmd/server/internal/server_internal.go`. Lập trình viên phát triển View / HTMX cần nắm rõ các quy tắc sau để tránh các lỗi bị chặn bởi trình duyệt (Blocked by CSP, 403 CSRF, Cross-Origin Isolation).

### 10.1. Cấu Hình Chuẩn Của Helmet
Trong `ServerInternal`, middleware Helmet được thiết lập:
```go
app.Use(helmet.New(helmet.Config{
    XSSProtection:             "0",
    ContentTypeNosniff:        "nosniff",
    XFrameOptions:             "SAMEORIGIN",
    ReferrerPolicy:            "strict-origin-when-cross-origin",
    CrossOriginResourcePolicy: "cross-origin",
}))
```
* **`XSSProtection: "0"`**: Tuân thủ chuẩn OWASP/MDN hiện đại, loại bỏ các lỗi rò rỉ dữ liệu của XSS auditor cũ.
* **`ContentTypeNosniff: "nosniff"`**: Chống MIME-sniffing, bảo vệ các file static assets (`/static/...`).
* **`CrossOriginResourcePolicy: "cross-origin"`**: Bắt buộc để hình ảnh `.webp`, file tĩnh, fonts, sprite SVG load bình thường trên mọi ngữ cảnh cross-origin.

### 10.2. Tương Thích Content-Security-Policy (CSP) Với HTMX
Hiện tại `ServerInternal` không bật chặn CSP cứng. **Nếu trong tương lai dự án bổ sung `ContentSecurityPolicy`**, bắt buộc cấu hình tối thiểu các chỉ thị sau để HTMX và giao diện không bị tê liệt:
* `connect-src 'self'`: **Bắt buộc** để trình duyệt cho phép HTMX gửi request AJAX (`hx-post`, `hx-get`).
* `script-src 'self' 'unsafe-inline'`: Cho phép nạp HTMX và các script điều khiển giao diện.
* `img-src 'self' data: https:`: Cho phép hiển thị ảnh từ CMS và hình ảnh base64/CDN.
* `style-src 'self' 'unsafe-inline' https://fonts.googleapis.com`: Cho phép nạp styles và fonts.

### 10.3. Cơ Chế Tự Động Đính Kèm CSRF Token Cho HTMX
1. **Thiết lập Server**: Fiber CSRF middleware lưu token ở cookie `csrf_` với thuộc tính `CookieHTTPOnly: false` để JavaScript có thể đọc được.
2. **Thứ tự script trong [master.templ](file:///Users/mr.nil/Documents/projects/2026/sunway_goweb/views/layouts/master.templ)**:
   ```html
   api-fetch.js -> modal.js -> htmx.min.js -> app.js
   ```
3. **Cơ chế tự động**:
   - Trong `app.js`, hàm `initHtmxCsrf()` tự động lắng nghe sự kiện:
     ```javascript
     document.body.addEventListener('htmx:configRequest', (e) => {
         const token = (typeof ApiFetch !== 'undefined') ? ApiFetch.getCsrfToken() : '';
         if (token) {
             e.detail.headers['X-CSRF-Token'] = token;
         }
     });
     ```
   - **Lợi ích**: Toàn bộ các component hay form sử dụng HTMX (`hx-post="/..."`, `hx-put`, `hx-delete`) trên toàn bộ website đều **tự động có header `X-CSRF-Token` hợp lệ**. Lập trình viên không cần phải truyền thẻ `<input type="hidden" name="csrf" ...>` thủ công trong file `.templ`.
   - **Tự động phục hồi khi token hết hạn**: Sự kiện `htmx:responseError` khi gặp mã 403 sẽ tự động gọi `ApiFetch.refreshCsrfToken()` để cấp phát token mới.

---

## 11. Quy Trình Kiểm Thử Chuẩn (Khi Được Yêu Cầu "Test Skill")

Khi lập trình viên yêu cầu **"test skill"** hoặc kiểm tra sau khi code, thực hiện tuần tự theo 5 bước kiểm tra:

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
* **Chặn request không hợp lệ**:
  ```bash
  curl -X POST http://localhost:8080/brands/results -d "page=2"
  ```
  $\rightarrow$ **Kỳ vọng**: Trả về `403 Forbidden` do thiếu CSRF token.
* **Chấp nhận request hợp lệ**:
  ```bash
  TOKEN=$(curl -sI http://localhost:8080/ | grep -i 'Set-Cookie: csrf_=' | sed -E 's/.*csrf_=([^;]+).*/\1/')
  curl -X POST http://localhost:8080/brands/results -H "X-CSRF-Token: $TOKEN" -H "Cookie: csrf_=$TOKEN" -d "page=1"
  ```
  $\rightarrow$ **Kỳ vọng**: Trả về `200 OK` chứa HTML fragment.

### Bước 4: Kiểm Tra HTMX Flow & Push URL
Gửi request mô phỏng HTMX:
```bash
curl -i -X POST http://localhost:8080/brands/results \
  -H "X-CSRF-Token: $TOKEN" -H "Cookie: csrf_=$TOKEN" -H "HX-Request: true" -d "page=2"
```
* **Kỳ vọng**:
  1. Header response có `HX-Push-Url: /brands?page=2`.
  2. Nội dung trả về chỉ là fragment danh sách, **không có** thẻ `<html>`, `<head>`, `<footer>`.

### Bước 5: Kiểm Tra Banner & Section 2 Tầng (Dynamic + Fallback)
* Kiểm tra khi CMS trống: View phải hiển thị banner/section mặc định với `<picture>`, text và CSS chuẩn.
* Kiểm tra khi CMS có dữ liệu: View render qua `@templ.Raw(...)`.

> Xem chi tiết các kịch bản test tại [Hướng Dẫn Kiểm Thử Toàn Diện](./references/testing-and-verification-guide.md).

---

## 12. Tài Liệu Tham Khảo Mở Rộng (References & Examples)

* [Hướng Dẫn Kiểm Thử & Xác Minh (Testing & Verification Guide)](./references/testing-and-verification-guide.md): Bộ lệnh curl, script kiểm tra bảo mật, test HTMX và kiểm tra console.
* [Quy Chuẩn HTMX Danh Sách, Bộ Lọc & Phân Trang (News & Brands)](./references/htmx-list-pattern.md): Mô hình Full Page + AJAX POST Results, `PaginationHx`, OOB Swap và `HX-Push-Url`.
* [Quy Chuẩn Banner & Option Group / Option Value](./references/cms-options-and-banner-guide.md): Nguyên tắc phân biệt Option Group vs Option Value, render 2 tầng Dynamic + Fallback, tích hợp Redis cache.
* [Hướng Dẫn CSS Responsive (rem, em, font-size)](./references/responsive-css-guide.md): Toàn bộ nguyên lý neo gốc `rem`, mở rộng `em`, fluid typography `clamp()`, và bảng quy đổi px.
* [Cú Pháp templ & Best Practices](./references/templ-syntax.md): Hướng dẫn chi tiết cú pháp thẻ, vòng lặp, điều kiện, classes động và nhúng component con.
* [Ví Dụ Trang Động Truyền Dữ Liệu](./examples/dynamic-view-with-data.md): Ví dụ mẫu đầy đủ từ Fiber Context, Handler, truyền struct Props vào View đến render ra HTML.




