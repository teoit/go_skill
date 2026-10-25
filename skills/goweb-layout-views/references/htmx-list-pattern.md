# Quy Chuẩn HTMX Danh Sách, Bộ Lọc & Phân Trang (HTMX List Pattern)

Tài liệu này đúc kết kiến trúc và quy chuẩn thực thi HTMX cho các trang danh sách có lọc, sắp xếp và phân trang trong `sunway_goweb` (dựa trên chuẩn thực tế của `news` và `brands`).

---

## 1. Triết Lý & Luồng Xử Lý (Architecture Flow)

Trang sử dụng mô hình **Hybrid Server-Side Rendering**:
1. **Initial Full-Page Load (GET `/news` hoặc `/brands`)**:
   - Trình duyệt yêu cầu URL chuẩn.
   - Server render toàn bộ trang (`layouts.Master`, header, banner, container, list, footer).
   - Tải thư viện HTMX chỉ trên trang đó qua helper `htmxJs()`.
2. **AJAX Fragment Update (POST `/news/results` hoặc `/brands/results`)**:
   - Khi người dùng đổi bộ lọc (sort, filter) hoặc bấm trang tiếp theo (pagination), HTMX gửi request **POST** lên endpoint results.
   - Server chỉ render đoạn HTML kết quả (`NewsResults` hoặc `BrandResults`), không render lại layout hay banner.
   - Server trả header `HX-Push-Url` để cập nhật thanh địa chỉ trình duyệt thành URL GET crawlable / shareable (ví dụ `/news?orderBy=oldest&page=2`).
   - HTMX hoán đổi (`hx-swap`) nội dung danh sách và cuộn mượt lên đầu danh sách (`show:#...-results:top`).
   - Cập nhật các khối thông tin ngoài container (ví dụ: số lượng bài viết / sản phẩm) bằng kỹ thuật **Out-Of-Band Swap (`hx-swap-oob="true"`)**.

---

## 2. Cấu Trúc File & Thành Phần Trong `views/`

Ví dụ chuẩn từ `views/news/index.templ`:

```text
views/news/
├── index.templ        # PageNews (Full page), NewsList (Grid + Pagination), NewsResults (Fragment HTMX)
└── ...
```

### 2.1. Template Trang Chính (Full Page)
Khởi tạo container chứa `hx-target`, `hx-swap`, `hx-indicator` và form điều khiển:

```templ
templ PageNews(data *responses.NewsListResp) {
    if data != nil {
        @layouts.Master(data.IsLogin, &[]templ.Component{newsCss()}, htmxJs()) {
            <main class="news_page">
                <!-- Banner Section (Dynamic / Fallback) -->
                @bannerSection(data.BannerHTML)

                <div class="container">
                    @partials.Breadcrumb(data.Breadcrumbs...)
                    <section class="news_container">
                        <header class="news_head">
                            <!-- Bộ đếm: oob = false cho lần tải đầu -->
                            @newsCount(data, false)

                            <!-- Form Sort / Filter kích hoạt HTMX khi change -->
                            <form
                                class="news_sort"
                                method="get"
                                action="/news"
                                hx-post="/news/results"
                                hx-trigger="change"
                                hx-target="#news-results"
                                hx-swap="innerHTML"
                                hx-indicator="#news-results"
                            >
                                <select id="news-sort" name="orderBy" class="news_sort_select">
                                    <option value="newest" selected?={ data.OrderBy == "newest" }>Newest First</option>
                                    <option value="oldest" selected?={ data.OrderBy == "oldest" }>Oldest First</option>
                                </select>
                            </form>
                        </header>

                        <!-- Vùng chứa danh sách và phân trang -->
                        <div
                            id="news-results"
                            class="news_results"
                            hx-target="this"
                            hx-swap="innerHTML show:#news-results:top"
                            hx-indicator="#news-results"
                            hx-include="#news-sort"
                        >
                            @NewsList(data)
                        </div>
                    </section>
                </div>
            </main>
        }
    }
}
```

### 2.2. Component Danh Sách & Phân Trang (`NewsList` / `BrandList`)
Tách riêng để tái sử dụng ở cả Full-Page và AJAX Fragment:

```templ
templ NewsList(data *responses.NewsListResp) {
    <div class="news_grid">
        if len(data.News) > 0 {
            for _, item := range data.News {
                @partials.ArticleCard(item)
            }
        } else {
            <p class="news_empty">The list of news articles is being updated.</p>
        }
    </div>
    <div>
        <!-- Phân trang HTMX chuẩn của dự án -->
        @partials.PaginationHx(data.CurrentPage, data.TotalPages, "/news/results", func(p int) string {
            return newsPageURL(data.OrderBy, p)
        })
    </div>
}
```

### 2.3. Component Fragment Trả Về Cho HTMX (`NewsResults` / `BrandResults`)
Chỉ render danh sách và OOB swap các phần tử liên quan:

```templ
templ NewsResults(data *responses.NewsListResp) {
    <!-- Cập nhật bộ đếm nằm ngoài container kết quả -->
    @newsCount(data, true)
    <!-- Cập nhật danh sách + phân trang mới -->
    @NewsList(data)
}

// Hàm render bộ đếm hỗ trợ Out-Of-Band swap
func newsCountAttrs(oob bool) templ.Attributes {
    if oob {
        return templ.Attributes{"hx-swap-oob": "true"}
    }
    return templ.Attributes{}
}

templ newsCount(data *responses.NewsListResp, oob bool) {
    <p id="news-count" class="news_count" { newsCountAttrs(oob)... }>
        Showing { fmt.Sprintf("%d", len(data.News)) } articles
    </p>
}
```

### 2.4. Thư Viện HTMX Toàn Cục
HTMX đã được nhúng toàn cục tại `views/layouts/master.templ`:
```html
<script type="text/javascript" src={ templates.AssetURL("/static/libs/htmx/htmx.min.js") }></script>
```
Do đó, tất cả các trang con không cần phải import hoặc định nghĩa lại hàm `htmxJs()` riêng lẻ nữa.

---

## 3. Quy Chuẩn Phân Tách Kiến Trúc: `handlers/` vs `apis/`

Để codebase rõ ràng, dễ bảo trì và không bị lẫn lộn giữa luồng tải trang (SSR Full Page) và luồng API/AJAX:

* **Thư mục `modules/web/transport/handlers/` (Page Handlers)**:
  - Chỉ chứa các handler phục vụ tải toàn bộ trang HTML (`@layouts.Master(...)`).
  - Ví dụ: `ListNewsHdl()` (GET `/news`), `NewsDetailHdl()` (GET `/news/:slug.html`), `ProductsPageHdl()` (GET `/products`).
  - Tuyệt đối không chứa các method trả về JSON hoặc HTML fragment của AJAX.

* **Thư mục `modules/web/transport/apis/` (API Handlers)**:
  - **BẮT BUỘC**: Mọi endpoint do thư mục `apis/` xử lý **bắt buộc phải có tiền tố route `/api/`** (ví dụ: `/api/news/results`, `/api/products/partial`, `/api/promotions/active-sale`).
  - Chứa tất cả các endpoint phục vụ request AJAX / HTMX:
    - **HTMX Fragment Handlers**: `NewsResultsApi()` (POST `/api/news/results`), `ProductsPartialHdl()` (GET `/api/products/partial`), `ActiveSaleApi()` (POST `/api/promotions/active-sale`).
    - **JSON APIs**: `ListNewsApi()` (POST `/api/news`), login, checkout...
  - Xử lý gộp: Khi dùng HTMX, không chia nhỏ thành các API rời rạc như `sort`, `pagination` riêng, mà gom chung vào 1 handler HTMX Results nhận struct `requests.NewsReq` (chứa cả filter, sort và page).

### 3.1. Handler GET Toàn Trang (Trong `handlers/news_hdl.go`)
```go
func (h *newsHdl) ListNewsHdl() fiber.Handler {
    return func(c *fiber.Ctx) error {
        var req requests.NewsReq
        if err := c.QueryParser(&req); err != nil {
            return err
        }
        req.Validate(c.Context())

        data, err := h.usc.ListNewsUsc(c.Context(), &req)
        if err != nil {
            return err
        }

        return templates.Render(c, news.PageNews(data))
    }
}
```

### 3.2. Handler POST HTMX Fragment (Trong `apis/news_api.go`)
```go
// POST /api/news/results
func (a *newsAPI) NewsResultsApi() fiber.Handler {
    return func(c *fiber.Ctx) error {
        var req requests.NewsReq
        if err := c.BodyParser(&req); err != nil {
            return c.Status(fiber.StatusBadRequest).SendString("Invalid request")
        }
        req.Validate(c.Context())

        data, err := a.usc.ListNewsUsc(c.Context(), &req)
        if err != nil {
            return err
        }

        // Logic tạo URL đã được đóng gói trong req.PushURL()
        c.Set("HX-Push-Url", req.PushURL())

        return templates.Render(c, news.NewsResults(data))
    }
}
```

---

## 4. Đăng Ký Route Chuẩn Trong `routes_web.go` (Feature Grouping)

Tất cả các route liên quan đến cùng một tính năng được gom lại và đặt liền kề nhau trong nhóm tương ứng (Public hoặc Authenticated):

```go
// --- News ---
app.Get("/news", newsComp.ListNewsHdl())
app.Get("/news/:pagepath<regex(^([a-z0-9-]+)-([0-9]+)$)>.html", newsComp.NewsDetailHdl())
app.Post("/api/news/results", apiNewsComp.NewsResultsApi())
app.Post("/api/news", apiNewsComp.ListNewsApi())
```

---

## 5. Checklist Khi Chuyển Một Trang Sang HTMX (Ví Dụ: `ProductsPageHdl`)

1. **View Templ**:
   - [ ] Chia `index.templ` thành 3 phần rõ ràng: `Page` (vỏ bọc), `List` (grid + pagination), `Results` (fragment kết quả).
   - [ ] `htmx.min.js` đã được nạp toàn cục trong `layouts/master.templ`, không cần chèn thủ công vào trang.
   - [ ] Container kết quả phải có `id`, `hx-target="this"`, `hx-swap="innerHTML show:#...:top"`, `hx-indicator`.
   - [ ] Đổi bộ lọc (filter sidebar, sort dropdown) sang `hx-post="/api/.../results"`, `hx-trigger="change"`.
   - [ ] Chuyển `@partials.Pagination` sang `@partials.PaginationHx(..., "/api/.../results", ...)`.
   - [ ] Thêm `hx-swap-oob="true"` cho bộ đếm số lượng (products count / news count).

2. **Backend Handlers & APIs**:
   - [ ] Giữ nguyên Page Handler (GET) trong `handlers/` để phục vụ direct URL, search bot và tải trang đầu (route không có `/api/`, ví dụ: `/products`).
   - [ ] Tạo API Handler mới `ProductResultsApi` (POST) trong `apis/products_api.go` nhận `BodyParser`, gọi usecase lấy data.
   - [ ] Thêm `c.Set("HX-Push-Url", pushURL)` trả về URL thân thiện của trang (ví dụ `/products?page=2`).
   - [ ] Render riêng fragment `products.ProductResults(data)`.

3. **Composer & Routes**:
   - [ ] Khai báo `ProductResultsApi() fiber.Handler` trong `comp_api_product.go`.
   - [ ] Đăng ký route bắt buộc có tiền tố `/api/` trong nhóm `// APIs (Public / Auth)` của `routes_web.go`:
     `app.Post("/api/products/results", apiProdComp.ProductResultsApi())`.

