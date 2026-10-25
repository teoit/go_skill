# Quy Chuẩn Quản Lý Banner & Section Content (Option Group & Option Value)

Tài liệu này quy định kiến trúc quản lý nội dung tĩnh / CMS (Banner, Section Content) trong dự án `sunway_goweb` thông qua bảng `OptionGroup` và `OptionValue`.

---

## 1. Bản Chất Kiến Trúc (Architecture Concept)

Trong hệ thống:
* **OptionGroup (`cms_option_groups`)**: Đại diện cho một trang hoặc một nhóm ngữ cảnh lớn gồm nhiều khối nội dung (ví dụ: Trang chủ `home`, Trang giới thiệu `about-us`, Trang chính sách...).
* **OptionValue (`cms_option_values`)**: Đại diện cho từng khối nội dung HTML / text cụ thể (ví dụ: Banner chính, Khối sứ mệnh, Khối các bước hướng dẫn, Cam kết chất lượng...).
  * Mỗi `OptionValue` có: `key`, `name`, `content` (chứa mã HTML do CMS biên tập), `ordering`, `status`, và liên kết tới `option_group_id`.

---

## 2. Quy Tắc Quyết Định: Option Group vs. Option Value

Khi một trang có các khối content hoặc banner cần quản lý qua CMS:

```text
Trang có bao nhiêu khối content cần quản trị?
 │
 ├── CHỈ CÓ 1 KHỐI (Ví dụ: 1 banner ở đầu trang danh sách sản phẩm / blog / brands)
 │    └── CHỈ DÙNG OPTION VALUE
 │         - Định nghĩa 1 const Option Value Key duy nhất.
 │         - Query trực tiếp qua OptionValueRepo.FirstOptionValueRepo.
 │
 └── CÓ NHIỀU KHỐI SECTION (Ví dụ: Trang chủ, About Us, How Ordering Works, FAQ Landing...)
      └── BẮT BUỘC DÙNG OPTION GROUP
           - Định nghĩa 1 const Option Group Key đại diện cho cả trang.
           - Mỗi section bên trong là 1 Option Value thuộc group đó.
           - Lấy toàn bộ qua OptionShared.GetOptionValuesByGroupKey (tự động cache Redis).
```

---

## 3. Trường Hợp 1: Trang Chỉ Có 1 Banner / 1 Section Đơn Lẻ (Chỉ Dùng Option Value)

Áp dụng cho các trang danh sách động (Products, News, Brands) nơi dữ liệu chính là dynamic list (products, articles, brands), chỉ có **duy nhất 1 banner đầu trang**.

### 3.1. Khai Báo Hằng Số (Constants)
Đặt trong `modules/web/common/consts/keys.go`:

```go
const (
    KEY_BLOG_NEWS_BANNER = "blog-news-banner"
    KEY_BRAND_BANNER     = "brand-banner"
    KEY_PRODUCT_BANNER   = "product-banner"
)
```

### 3.2. Truy Vấn Trong UseCase
Inject `optionValueRepo` vào usecase và query trực tiếp theo key:

```go
func (usc *brandUsc) getBrandBanner(ctx context.Context) (string, error) {
    if usc.optionValueRepo == nil {
        return "", nil
    }

    conds := map[string]interface{}{
        "key = ?":    consts.KEY_BRAND_BANNER,
        "status = ?": enum.STATUS_ACTIVE_NUMBER,
    }
    filter := utils.Filters{
        Conds: &conds,
    }

    optVal, err := usc.optionValueRepo.FirstOptionValueRepo(ctx, &filter)
    if err != nil {
        if errors.Is(err, core.ErrRecordNotFound) {
            return "", nil // Không có banner -> Trả về rỗng an toàn để kích hoạt fallback
        }
        return "", err // 💥 DB sập / mất kết nối -> Bắt buộc trả lỗi để kích hoạt HTTP 500!
    }
    if optVal == nil {
        return "", nil
    }

    return optVal.Content, nil
}
```

### 3.3. Quy Chuẩn Render Banner Trong `.templ`
**NGUYÊN TẮC VÀNG**: Luôn luôn có cấu trúc **2 tầng (Dynamic HTML + Fallback Default)**:
- Nếu database có content (`banner != ""`), render bằng `@templ.Raw(banner)`.
- Nếu chưa có dữ liệu hoặc CMS trống, **BẮT BUỘC** hiển thị HTML mặc định với responsive `<picture>`, `<source media="...">`, class CSS và text chuẩn. Tuyệt đối không để trang bị khuyết banner.

```templ
templ brandBanner(banner string) {
    if banner != "" {
        @templ.Raw(banner)
    } else {
        <section class="banners">
            <div class="banner_content">
                <h1 class="banner_title">Brands</h1>
                <p class="banner_des">
                    Discover our brands
                    <br/>
                    in one complete Sunway.
                </p>
            </div>
            <picture>
                <source media="(max-width: 767px)" srcset={ templates.AssetURL("/static/img/banners/banner-catalogues.webp") }/>
                <img class="banner-img" src={ templates.AssetURL("/static/img/banners/banner-catalogues.webp") } alt="Brands Banner"/>
            </picture>
        </section>
    }
}
```

---

## 4. Trường Hợp 2: Trang Có Nhiều Section Content (Bắt Buộc Dùng Option Group)

Áp dụng cho các trang tĩnh, landing page, hoặc các trang có nhiều section nội dung không phải show list dữ liệu thực thể (ví dụ: Trang chủ `home`, Giới thiệu `about-us`, Hướng dẫn mua hàng `how-ordering-work`, Chính sách `terms-of-use`...).

### 4.1. Khai Báo Hằng Số (Option Group Key)
Đặt 1 const duy nhất cho Option Group trong `conf/const.go` hoặc `consts/keys.go`:

```go
const (
    CategorySlugHome        = "home"
    KEY_OPTION_GROUP_ABOUT  = "about-us"
    KEY_OPTION_GROUP_HOW_IT = "how-ordering-works"
)
```

### 4.2. Truy Vấn Trong UseCase Qua `OptionShared`
Sử dụng `OptionShared.GetOptionValuesByGroupKey(ctx, groupKey)`:
- Tự động kiểm tra **Redis Cache** (key: `enum.PREFIX_CACHE_REDIS_OPTION_GROUP + groupKey`).
- Nếu cache miss: Query DB bảng `cms_option_groups` theo `key` -> Lấy danh sách `cms_option_values` theo `option_group_id` sắp xếp theo `ordering ASC, id ASC`.
- Lưu cache Redis với TTL chuẩn `conf.TtlCacheOptionGroup`.
- Trả về slice `[]responses.OptionValueResp`.

```go
// Ví dụ trong HomeUsc hoặc PageUsc:
options, err := usc.optionShared.GetOptionValuesByGroupKey(ctx, conf.CategorySlugHome)
if err != nil {
    return nil, err
}

// Chuyển danh sách options thành Map để dễ dàng tra cứu theo Key trong template/response:
sectionsMap := make(map[string]string)
for _, opt := range options {
    sectionsMap[opt.Key] = opt.Content
}
```

### 4.3. Render Từng Section Trong Template
Trong template `.templ`, mỗi section sẽ nhận HTML từ CMS theo key riêng, kèm theo HTML fallback:

```templ
templ AboutUsPage(sections map[string]string) {
    @layouts.Master(false, nil) {
        <main class="about_page">
            <!-- Section 1: Hero Banner -->
            @renderSection(sections["about-hero-banner"], defaultAboutHero())

            <!-- Section 2: Mission & Vision -->
            @renderSection(sections["about-mission"], defaultAboutMission())

            <!-- Section 3: Core Values -->
            @renderSection(sections["about-values"], defaultAboutValues())
        </main>
    }
}

templ renderSection(htmlContent string, defaultCpn templ.Component) {
    if htmlContent != "" {
        @templ.Raw(htmlContent)
    } else {
        @defaultCpn
    }
}
```

---

## 5. Checklist Triển Khai Cho Lập Trình Viên

1. **Phân loại nhu cầu**:
   - Nếu là 1 banner lẻ đầu trang danh sách -> Tạo const `KEY_..._BANNER`, dùng `FirstOptionValueRepo`.
   - Nếu là trang gồm nhiều section text/hình ảnh CMS -> Tạo const `KEY_OPTION_GROUP_...`, dùng `optionShared.GetOptionValuesByGroupKey`.
2. **Fallback HTML**:
   - Luôn dựng đầy đủ HTML fallback chuẩn thiết kế trong `.templ` phòng khi DB chưa có dữ liệu.
3. **An toàn bảo mật & CSS**:
   - Nội dung render qua `@templ.Raw()` phải đến từ nguồn CMS đáng tin cậy.
   - Thẻ ảnh trong CMS hoặc fallback phải tối ưu định dạng `.webp` và responsive với `<picture>` / `<source>`.
