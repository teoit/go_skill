---
name: goweb-page-sections-option
description: >-
  Hướng dẫn và quy chuẩn quản lý các section tĩnh hoặc linh hoạt của trang web
  (Banners, Intro, Content Blocks, Call-to-actions, Popups...) bằng OptionService
  (GetOptionByKeyCacheService và GetOptionByKeyGroupCacheService).
  Phân biệt rõ: 1 section đơn lẻ (Option Value riêng) và nhiều sections trên 1 trang (Option Group -> map[string]string ra Views).
---

# Page Sections Pattern (Quản Lý Section Bằng OptionService: Option Group & Option Value)

Tài liệu này chuẩn hóa quy trình triển khai các **Section nội dung CMS** (Banners, Khối giới thiệu, Khối đối tác, Khối nội dung HTML tùy chỉnh...) trên mọi trang web trong dự án `sunway_goweb` thông qua [`OptionService`](file:///d:/projects/2026_projects/sunway_goweb/modules/web/services/options_service.go).

---

## 1. Bản Chất Kỹ Thuật (Architecture Concept)

Khi xây dựng một trang web (như *Home, About Us, Contact, News, Order Instructions, Terms, Privacy...*), các khối nội dung quản trị CMS được quản lý bằng 2 bảng:
- **Option Group** (`cms.option_group`): Đại diện cho một trang hoặc một nhóm ngữ cảnh (ví dụ: `home`, `about-us`, `contact`).
- **Option Value** (`cms.options_values`): Đại diện cho từng section cụ thể, lưu khóa định danh (`key`) và mã HTML (`content`).

Mọi truy vấn đều đi qua `services.OptionService` ([`modules/web/services/options_service.go`](file:///d:/projects/2026_projects/sunway_goweb/modules/web/services/options_service.go)):
1. **`GetOptionByKeyCacheService(ctx, key, isCache)`** -> trả về `(*string, error)`: Dùng cho trang chỉ cần lấy **1 section đơn lẻ** theo `key`.
2. **`GetOptionByKeyGroupCacheService(ctx, keyGroup, isCache)`** -> trả về `(map[string]string, error)`: Dùng cho trang có **nhiều sections** thuộc cùng 1 `Option Group`, sử dụng **1 câu truy vấn SQL Subquery duy nhất** (`option_group_id = (SELECT id FROM cms.option_group WHERE key = ? AND status = ? LIMIT 1)`) và chỉ lấy 2 cột `key, content` kết hợp Redis cache.

---

## 2. Quy Chuẩn Đặt Tên Hằng Số (`common_option.go`)

> [!IMPORTANT]
> **Tuyệt đối KHÔNG gán cứng chuỗi (Magic Strings)**. Mọi `key` của Option Group và Option Value bắt buộc phải khai báo hằng số trong file [modules/web/common/consts/common_option.go](file:///d:/projects/2026_projects/sunway_goweb/modules/web/common/consts/common_option.go).

```go
package consts

const (
	KEY_HOME_OPTION            = "home"
	KEY_HOME_BANNER_LOGIN      = "home-banner-login"
	KEY_HOME_BANNER_NO_LOGIN   = "home-banner-no-login"
	KEY_HOME_WHY_PARTNER       = "home-why-partner"
	KEY_HOME_READY_OUR_PARTNER = "home-ready-our-partner"
	POPUP_SALE_NO_LOGIN        = "popup-sale-no-login"
	POPUP_SALE_LOGIN           = "popup-sale-login"
	KEY_BLOG_NEWS_BANNER       = "blog-news-banner"
	KEY_BRAND_BANNER           = "brand-banner"
	KEY_PRODUCT_BANNER         = "product-banner"
	KEY_QUESTION               = "footer-question"
)
```

---

## 3. Mẫu Triển Khai Chuẩn Với `OptionService` (Ví dụ Trang Home)

### Bước 1: Khởi tạo `OptionService` trong Composer (`comp_home.go`)
```go
optionValueRepo := repository.NewOptionValueRepo(dbGorm)
optionService := services.NewOptionService(optionValueRepo, redisClient, logger)
```

### Bước 2: Định nghĩa Response (`home_resp.go`)
Lưu trữ map các section dưới dạng `map[string]string` (`key -> content HTML`):
```go
type HomeResp struct {
	...
	HomeOptions map[string]string
}
```

### Bước 3: Gọi `GetOptionByKeyGroupCacheService` trong UseCase (`home_usc.go`)
Viết đầy đủ comment nghiệp vụ theo chuẩn `goweb-code-comments`:
```go
// 9. Lấy danh sách option values theo group key "home" (banner, đối tác...)
homeOptions, err := usc.optionService.GetOptionByKeyGroupCacheService(ctx, consts.KEY_HOME_OPTION, true)
if err != nil {
	return nil, err
}
```

### Bước 4: Hiển thị ngoài View Templ (`views/homes/index.templ`)
Truy xuất trực tiếp từ `resp.HomeOptions[consts.KEY_...]` và render bằng `@templ.Raw(...)`:

```templ
// 1 - if login và key home-banner-login ngược lại home-banner-no-login
if resp.IsLogin {
	if banner, ok := resp.HomeOptions[consts.KEY_HOME_BANNER_LOGIN]; ok && banner != "" {
		@templ.Raw(banner)
	}
} else {
	if banner, ok := resp.HomeOptions[consts.KEY_HOME_BANNER_NO_LOGIN]; ok && banner != "" {
		@templ.Raw(banner)
	}
}

if !resp.IsLogin {
	// if không có login và key home-why-partner show
	if whyPartner, ok := resp.HomeOptions[consts.KEY_HOME_WHY_PARTNER]; ok && whyPartner != "" {
		@templ.Raw(whyPartner)
	}

	// if không có login và key home-ready-our-partner show
	if readyOurPartner, ok := resp.HomeOptions[consts.KEY_HOME_READY_OUR_PARTNER]; ok && readyOurPartner != "" {
		@templ.Raw(readyOurPartner)
	}
}
```

---

## 4. Xử Lý Dữ Liệu Động Cá Nhân Hóa Trong HTML Tĩnh Được Cache (Client-side Sync)

Khi một section HTML từ CMS được lưu Redis cache dùng chung cho mọi user (ví dụ `home-banner-login`), **không** chèn tên từng khách hàng trực tiếp vào chuỗi HTML trong Redis. Thay vào đó:
- Đặt thẻ đánh dấu cố định trong nội dung HTML của Option Value (ví dụ `<span id="bannerCustomerName">Customer</span>`).
- Dùng script riêng của trang (như [public/js/home.js](file:///d:/projects/2026_projects/sunway_goweb/public/js/home.js)) lấy thông tin tên đã render trên Header (`.login-info .login-info_name`) gán qua `.textContent` cho `#bannerCustomerName`.

---

## 5. Checklist Triển Khai

- [ ] Xác định số lượng section: **1 section** (`GetOptionByKeyCacheService`) hay **nhiều section theo group** (`GetOptionByKeyGroupCacheService`).
- [ ] Khai báo hằng số Group Key và Option Value Keys vào [common_option.go](file:///d:/projects/2026_projects/sunway_goweb/modules/web/common/consts/common_option.go).
- [ ] Tiêm `services.OptionService` vào UseCase thông qua Composer.
- [ ] Khai báo `map[string]string` trong Response struct (cho nhóm section) hoặc `*string` (cho 1 section).
- [ ] Viết đầy đủ docblock và comment các bước nghiệp vụ trong UseCase & Service.
- [ ] Ngoài View `.templ`, kiểm tra `if val, ok := resp.Options[consts.KEY_...]; ok && val != ""` trước khi `@templ.Raw(val)` và chạy `make gen`.
